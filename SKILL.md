---
name: flight-search
description: Use when the user gives a rough travel plan and wants the best flight options -- searches multiple nearby airports, compares total trip cost including driving, surfaces extended-layover country visits on request, handles PTO optimization and jet lag planning when PTO hours are mentioned, and outputs a formatted CSV.
---

# Flight Search

## Overview

This skill assumes you live near a **home airport** but are willing to drive to cheaper departure airports. Always search all candidate airports, factor in driving cost, and output a ranked CSV.

**Before running any searches, confirm with the user:**
- Their home airport (IATA code)
- Any nearby airports they are willing to drive to, with approximate drive time and distance
- Their home time zone (for jet lag calculations)

If this information is saved in your project's CLAUDE.md or memory, use that instead of asking.

---

## SerpAPI -- Google Flights

**Base URL:** `https://serpapi.com/search`  
**API key:** `$SERPAPI_API_KEY` environment variable

### Key Parameters

| Parameter | Required | Description |
|---|---|---|
| `engine` | Yes | `google_flights` |
| `api_key` | Yes | `$SERPAPI_API_KEY` |
| `departure_id` | Yes | IATA airport code (e.g. `ORD`) |
| `arrival_id` | Yes | IATA airport code (e.g. `LIM`) |
| `outbound_date` | Yes | `YYYY-MM-DD` |
| `return_date` | Yes (round trip only) | `YYYY-MM-DD` |
| `type` | Yes | `1`=round trip, `2`=one way |
| `adults` | No | Default `1` |
| `travel_class` | No | `1`=economy, `2`=premium economy, `3`=business, `4`=first |
| `stops` | No | `0`=any, `1`=nonstop only, `2`=1 stop max |
| `currency` | No | Always pass `USD` |
| `gl` | No | Always pass `us` |
| `sort_by` | No | `1`=top, `2`=price, `5`=duration -- always use `2` |

### Key Response Fields

```
flights[].departure_airport.id        # IATA code
flights[].departure_airport.time      # "2026-06-13 06:30"
flights[].arrival_airport.id
flights[].arrival_airport.time
flights[].airline
flights[].flight_number               # "UA 123"
flights[].duration                    # minutes
layovers[].name                       # airport name
layovers[].duration                   # minutes
total_duration                        # total minutes including layovers
price                                 # USD
booking_token                         # use for booking link lookup
search_metadata.google_flights_url    # always use as booking link
```

### Example Call

```
GET https://serpapi.com/search?engine=google_flights&api_key=$SERPAPI_API_KEY&departure_id=ORD&arrival_id=LIM&outbound_date=2026-06-13&return_date=2026-06-29&type=1&adults=1&travel_class=1&sort_by=2&currency=USD&gl=us
```

### Rate Limiting & Quota

**SerpAPI free plan: 250 searches/month.** A full 6-airport + multi-city search uses 15-25 calls -- be aware of remaining quota.

**Always make API calls from the main agent via Bash curl -- never delegate to subagents.**

Subagents may not have Bash access, which means they cannot read `$SERPAPI_API_KEY` from the environment. Parallel WebFetch calls also cause 429s because they all fire simultaneously.

**Correct pattern -- sequential Bash curl with sleeps:**

```bash
KEY=$(echo $SERPAPI_API_KEY)
BASE="https://serpapi.com/search?engine=google_flights&api_key=${KEY}&adults=1&travel_class=1&sort_by=2&currency=USD&gl=us"

# Each call separated by sleep to avoid rate limiting
curl -s "${BASE}&departure_id=ORD&arrival_id=LIM&outbound_date=2026-06-13&return_date=2026-06-27&type=1" | python3 -c "..."
sleep 2
curl -s "${BASE}&departure_id=MKE&arrival_id=LIM&outbound_date=2026-06-13&return_date=2026-06-27&type=1" | python3 -c "..."
sleep 2
# ... continue for each airport
```

**Parse each response inline with Python:**
```bash
python3 -c "
import json,sys
d=json.load(sys.stdin)
url=d.get('search_metadata',{}).get('google_flights_url','')
for f in (d.get('best_flights',[]) + d.get('other_flights',[]))[:3]:
    print(f['price'], f.get('airline',''), f['total_duration'], 'stops:', len(f.get('layovers',[])), url)
"
```

**Rules:**
- Get the key once: `KEY=$(echo $SERPAPI_API_KEY)` -- don't re-read per call
- `sleep 2` between every curl call
- Never fire multiple WebFetch or curl calls in parallel for this API
- If 429 appears, `sleep 10` then retry once before stopping

---

## Nearby Airports

Ask the user for their home airport and any nearby airports they are willing to drive to. For each airport, note:
- IATA code
- Drive time from home
- Drive distance in miles

**Drive cost estimate:** $0.20/mile (gas only). Round-trip if returning to same airport. If flying into a different airport, one-way drive only.

**Example table (fill in with user's actual airports):**

| Code | Airport | Drive from Home | Est. Drive Cost |
|------|---------|----------------|-----------------|
| HOME | Home Airport | 0 min | $0 |
| AAA | Nearby Airport 1 | ~1h / 80 mi | ~$16 |
| BBB | Nearby Airport 2 | ~2h / 130 mi | ~$26 |

---

## Workflow

### Step 1 -- Parse the travel plan

Extract from what the user says:
- Destination city / country
- Approximate travel dates (or date range)
- Number of travelers
- Any preferences (nonstop, cabin class, flexible dates)
- **Multi-destination intent** -- trigger multi-city mode (Step 2B) if the user mentions ANY of:
  - "another country," "two countries," "squeeze in X," "stop in X on the way"
  - A country or city that is NOT the primary destination
  - "open jaw," "one-way in / one-way out"

If dates are vague (e.g. "mid-June"), search the cheapest 3-day window in that range by running `sort_by=2` and checking a few date combos.

---

### Step 2 -- Run ALL searches from the main agent via sequential Bash curl

> **STOP. Read before firing any API call.**
>
> **ALL SerpAPI calls must be made by the main agent using Bash curl -- NOT delegated to subagents.**
>
> Subagents lack Bash access and cannot read `$SERPAPI_API_KEY`. Parallel calls (WebFetch or curl) cause 429 rate limit errors. The correct approach is one sequential Bash script from the main agent.
>
> **Do NOT:**
> - Dispatch subagents to make API calls
> - Use WebFetch for SerpAPI (causes parallel 429s)
> - Fire multiple curl calls without `sleep 2` between them
>
> **Do:**
> - Get the key once: `KEY=$(echo $SERPAPI_API_KEY)`
> - Build one Bash script with all calls separated by `sleep 2`
> - Parse each response inline with Python
> - Collect all results into variables, then proceed to Step 3

---

### Step 2A -- Round-trip search (primary destination only)

Search each candidate departure airport to the primary destination, same dates.

Parameters:
- `sort_by=2`, `currency=USD`, `gl=us`
- `type=1` (round trip)

Take the **top 3 results** per airport.

> Round-trip searches to a single destination will NEVER surface multi-country itineraries. They are included for comparison only. Multi-country options come from Step 2B running in the same batch.

---

### Step 2B -- Multi-destination search (dispatched in same batch as 2A when another country is mentioned)

**Do not search only round-trips when the user wants to visit multiple countries.** Multi-city itineraries must be built from separate one-way legs.

#### 2B-1: Identify country combinations to search

Start with any country the user explicitly named. Then consider natural geographic pairings with the primary destination:

| Primary | Natural add-ons to check |
|---------|--------------------------|
| Peru (LIM/CUZ) | Ecuador (UIO/GYE), Bolivia (LPB/VVI), Colombia (BOG/CTG), Chile (SCL) |
| Colombia (BOG) | Ecuador (UIO), Peru (LIM), Panama (PTY) |
| Mexico | Guatemala (GUA), Cuba (HAV), Belize (BZE) |
| Thailand | Vietnam (HAN/SGN), Cambodia (PNH), Laos (VTE) |
| Italy | Croatia (DBV/ZAG), Greece (ATH), Morocco (CMN) |

For each candidate, determine logical ordering:
- **Before primary:** If the country lies on or near the outbound routing (e.g. Ecuador before Peru)
- **After primary:** If it makes more geographic sense on the way home (e.g. Bolivia after Cusco)
- **Both:** Search both orderings when not obvious

#### 2B-2: Search each ordering as three one-way legs

For each combination (e.g. ORD -> UIO -> CUZ -> ORD):

1. **Leg 1** (one-way): each candidate departure airport -> country1 on outbound date  
2. **Leg 2** (one-way): country1 -> primary destination on date = outbound + days in country1  
3. **Leg 3** (one-way): primary destination -> each candidate return airport on return date

Run all legs sequentially with `sleep 2` between calls. Use `type=2` (one-way) for each.

```
GET /search?engine=google_flights&type=2&departure_id=ORD&arrival_id=UIO&outbound_date=2026-06-13&sort_by=2&currency=USD&gl=us
GET /search?engine=google_flights&type=2&departure_id=UIO&arrival_id=CUZ&outbound_date=2026-06-15&sort_by=2&currency=USD&gl=us
GET /search?engine=google_flights&type=2&departure_id=CUZ&arrival_id=ORD&outbound_date=2026-06-28&sort_by=2&currency=USD&gl=us
```

Take cheapest 2 options per leg per airport pairing.

#### 2B-3: Combine and price

```
multi_city_total = leg1_price + leg2_price + leg3_price + drive_cost
```

Drive cost: round-trip if depart/return same airport; one-way only if open jaw (different airports).

Combine the cheapest leg pairings. Add to CSV as multi-city rows alongside standard round-trip rows.

---

### Step 3 -- Compute Value Score and rank within categories

Organize results into two categories: **Round Trip** and **Multi-Country**. Rank within each category by **Value Score** ascending (lower = better):

```
Value Score = Total Cost (USD) + (Total Flying Hours x $20)
```

- `$20/hour` = discomfort cost of time spent on a plane (saving $100 is worth 5 hrs of extra flying)
- **Round Trip:** Total Flying Hours = outbound hours x 1.9 (estimated return leg)
- **Multi-Country:** Total Flying Hours = sum of all individual leg durations
- Do NOT mix categories in the same ranking -- each section has its own Rank 1, 2, 3...

### Step 4 -- Extended layover options (if requested)

If the user asks about squeezing in another country or city during a long layover:

1. Identify layover airports in the raw results that are in interesting countries (e.g. ICN, IST, AMS, DXB, HEL, DOH, NRT).
2. For each interesting layover airport, search a second leg: `departure_id=<layover_code>` -> `arrival_id=<final_destination>` with outbound date = layover date + 1 to 3 days.
3. Present these as **"Extended Layover" rows** in the CSV -- mark the layover city, the extra days spent, and the delta in price vs. direct.

Rule of thumb: layovers > 8 hours at a major hub = flag as "could extend" even if the user didn't ask.

### Step 5 -- PTO optimization (if PTO hours are mentioned)

If the user mentions PTO hours, days off, or a specific number of vacation days available, run this analysis on top of the standard results.

#### Compute PTO efficiency

For each flight option calculate:
- **Days at destination** = arrival date -> departure date (calendar days on the ground)
- **PTO days burned** = workdays in that window that aren't already weekends or US federal holidays
- **PTO efficiency** = days at destination / PTO days burned

Goal: maximize days at destination per PTO day spent.

**Strategies to search and flag:**

| Strategy | Description |
|----------|-------------|
| **Depart Friday night / red-eye** | Arrive Saturday, burn 0 PTO for travel day |
| **Return Sunday night / late** | Land after midnight -> Monday work is possible |
| **Return Monday red-eye** | Burn 1 PTO day but gain a full extra day abroad |
| **Bridge holidays** | If a US holiday falls mid-week, flag it -- turns 2 PTO days into a 5-day trip |
| **Shoulder travel** | Depart/return on a weekend to keep full weekday blocks intact |

For each strategy found in the real flight results, add a row or note in the CSV.

#### Jet lag assessment

For each result, compute the time-zone offset between the user's home time zone and the destination. Then apply:

| Direction | Offset | Jet lag rule of thumb | Return-to-work recommendation |
|-----------|--------|----------------------|-------------------------------|
| East (Europe/Middle East) | +6 to +9h | Hard eastbound, ~1 day/hr to adjust | Return 2+ days before work; red-eye return on Friday or Saturday ideal |
| East (Asia/Oceania) | +13 to +17h | Severe; body clock reverses | Return 3+ days before work if possible; mark options < 2 days as warning |
| West (Hawaii/Pacific) | -4 to -6h | Easier; stay-up strategy works | 1 day buffer usually sufficient |
| Minimal (Mexico/Caribbean) | +-1 to +-3h | Negligible | No buffer needed |

**Flag in the CSV:**
- `Jet Lag Risk`: Low / Moderate / High / Severe
- `Recommended Recovery Days`: integer (days between landing and first workday)
- `First Workday`: date the user would realistically be back at full capacity
- Add "WARNING: tight recovery" in Notes if recovery days < recommended

#### PTO budget check

If the user states a PTO budget (e.g. "I have 5 PTO days"):
1. Filter out any option requiring more PTO than budget
2. Sort remaining options by PTO efficiency descending as a secondary sort (primary is still total cost)
3. Add a `PTO Days Used` column and a `PTO Efficiency` column to the CSV

### Step 6 -- Output CSV

Write results to `flights_<destination>_<date>.csv` in the current directory.

---

## CSV Format

### Structure

Write a **single CSV file** with two clearly separated sections. Each section has its own label row followed by its own header row. Separate sections with one blank row.

```
ROUND TRIP
Rank,Value Score,...headers...
row,row,...
row,row,...

MULTI-COUNTRY
Rank,Value Score,...headers...
row,row,...
```

The label row (`ROUND TRIP` or `MULTI-COUNTRY`) has the label in column 1 and blanks for all other columns -- it is a visual separator, not a data row.

---

### Round Trip columns (in order)

```
Rank,Value Score,Departure Airport,Drive Time,Drive Cost,Airline(s),Flight Numbers,Route,Depart,Arrive,Stops,Total Flying Hrs,Layover Cities,Flight Price,Total Cost,Layover Country,Extra Days,Layover Price Delta,PTO Days Used,PTO Efficiency,Jet Lag Risk,Recovery Days,First Workday,Booking Link,Notes
```

### Multi-Country columns (in order)

```
Rank,Value Score,Itinerary Type,Departure Airport,Drive Time,Drive Cost,Airline(s),Leg Breakdown,Route,Depart,Arrive,Total Flying Hrs,Countries + Days,Flight Price,Total Cost,Layover Country,Extra Days,Layover Price Delta,PTO Days Used,PTO Efficiency,Jet Lag Risk,Recovery Days,First Workday,L1 Booking Link,L2 Booking Link,L3 Booking Link,Notes
```

---

### Column definitions

**Shared columns:**

| Column | Format | Notes |
|--------|--------|-------|
| `Rank` | integer | 1 = best Value Score within this section |
| `Value Score` | integer | `Total Cost + (Total Flying Hrs x $20)` -- lower is better |
| `Departure Airport` | IATA | e.g. `ORD` |
| `Drive Time` | text | e.g. `0 min`, `1h 45m`, `3h` |
| `Drive Cost` | `$N` | Round-trip if same departure/return airport; append `(open jaw)` if different airports |
| `Airline(s)` | text | All airlines used, slash-separated, e.g. `Delta / JetSMART / Volaris` |
| `Flight Numbers` | text | Pipe-separated per leg, e.g. `DL 123 \| DL 456` |
| `Route` | text | Full airport chain with `->`, e.g. `ORD -> ATL -> BOG -> LIM` |
| `Depart` | `YYYY-MM-DD HH:MM` | Local time of first departure |
| `Arrive` | `YYYY-MM-DD HH:MM` | Local time of final arrival |
| `Stops` | integer | Outbound connecting stops (RT) or total connecting stops across all legs (multi-city), not counting country destination airports |
| `Total Flying Hrs` | decimal | RT: outbound hrs x 1.9 estimated; multi-city: sum of all leg durations |
| `Layover Cities` | IATA pipe-separated | Connecting airports only, e.g. `ATL \| MIA` |
| `Flight Price` | `$N` | RT: round-trip fare; multi-city: sum of all legs |
| `Total Cost` | `$N` | Flight Price + Drive Cost |
| `Layover Country` | text | Country name for extended layover visit; blank otherwise |
| `Extra Days` | integer | Days spent at extended layover stop; blank if none |
| `Layover Price Delta` | `+$N` | Extra cost vs. direct for extended layover; blank if none |
| `PTO Days Used` | integer | Blank if PTO not mentioned |
| `PTO Efficiency` | decimal | Destination days / PTO days; blank if PTO not mentioned |
| `Jet Lag Risk` | `Low` / `Moderate` / `High` / `Severe` | Blank if PTO not mentioned |
| `Recovery Days` | integer | Recommended days between landing and first workday; blank if PTO not mentioned |
| `First Workday` | `YYYY-MM-DD` | Blank if PTO not mentioned |
| `Notes` | text | Flag anything notable: ULCC carriers, baggage fees, unusual routings, holiday savings, tight connections |

**Round Trip only:**

| Column | Format | Notes |
|--------|--------|-------|
| `Booking Link` | URL | `google_flights_url` from search_metadata |

**Multi-Country only:**

| Column | Format | Notes |
|--------|--------|-------|
| `Itinerary Type` | text | Short label: `Ecuador First`, `Colombia After Peru`, `Bolivia Stopover`, etc. |
| `Leg Breakdown` | text | One entry per leg, slash-separated: `ORD->BOG $178 / BOG->LIM $225 / LIM->ORD $291` |
| `Countries + Days` | text | Approx days in each country: `Colombia: 3 days / Peru: 12 days` |
| `L1 Booking Link` | URL | `google_flights_url` from leg 1 search |
| `L2 Booking Link` | URL | `google_flights_url` from leg 2 search |
| `L3 Booking Link` | URL | `google_flights_url` from leg 3 search; blank if only 2 legs |

---

### Formatting rules

- Use title case for all column headers
- Wrap any field containing commas in double quotes
- No trailing commas
- Leave cells blank for N/A -- do not write `N/A`, `-`, or `0`
- Dollar amounts: `$618` (no decimal unless cents matter)
- Never fabricate prices -- only write values returned by the API

---

## After Writing the CSV

Tell the user:
1. Where the file was saved
2. Top 3 options in plain English (1-2 sentences each)
3. If extended layover options were found, highlight the best one
4. If PTO was mentioned: call out the highest-efficiency option and flag any options with tight recovery windows

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Only searching the home airport | Always search all candidate airports the user provided |
| Forgetting drive cost | Add it before computing Value Score |
| Ranking all options together in one list | Rank within each section separately (Round Trip: Rank 1...N; Multi-Country: Rank 1...N) |
| Ranking only by price | Always use Value Score = Total Cost + (Total Flying Hrs x $20) -- a cheap flight with 40hrs of flying isn't the best option |
| Putting RT and multi-country rows in the same section | They are separate sections with separate headers and separate rankings |
| Omitting the Leg Breakdown column for multi-city | Always show per-leg costs: `ORD->BOG $178 / BOG->LIM $225 / LIM->ORD $291` |
| Omitting Countries + Days for multi-city | Always show time split: `Colombia: 3 days / Peru: 12 days` |
| Using one Booking Link column for multi-city | Multi-city gets L1 / L2 / L3 Booking Link columns; single Booking Link column is for Round Trip only |
| Searching only round-trips when another country is mentioned | Round-trip searches NEVER surface multi-country itineraries -- always run Step 2B in the same batch |
| Running 2A first, then 2B after | Run all searches in one sequential Bash script -- never in separate batches |
| Only flagging existing layovers for "another country" | Step 2B searches entirely new routings (e.g. ORD->UIO->CUZ->ORD), not just pauses on existing tickets |
| Skipping geographic pairing analysis | Use the Step 2B table to identify natural add-ons; don't wait for the user to name a specific country |
| Skipping extended layover flag | Check layover duration >= 8h at major hubs |
| CSV with inconsistent quoting | Wrap any field containing commas in double quotes |
| Fabricating prices | Only write prices from the API response |
| Leaving PTO columns blank when PTO was mentioned | Fill all 5 PTO/jet lag columns for every row when PTO is in scope |
| Ignoring weekend/holiday boundaries in PTO count | Only count workdays (Mon-Fri, non-holiday) as PTO burned |
| Recommending a tight return for east-Asia travel | Flag any return leaving < 3 days before first workday as tight recovery |
| Delegating API calls to subagents | Subagents often lack Bash access and cannot read `$SERPAPI_API_KEY` -- make ALL calls from main agent |
| Parallel WebFetch or curl calls to SerpAPI | Fires all requests simultaneously -> immediate 429s; always use sequential Bash curl with `sleep 2` |
| Ignoring SerpAPI quota | Free plan = 250 searches/month; a full multi-city search burns 15-25; stop early if 429s appear |
