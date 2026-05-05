# flight-search

A Claude Code skill that searches Google Flights across multiple nearby airports, factors in driving cost, and outputs a ranked CSV with value-scored options.

## What it does

Given a rough travel plan ("I want to go to Peru in mid-June"), the skill searches every airport you are willing to drive to, computes a Value Score (price + time penalty), and writes a structured CSV with ranked round-trip and multi-country options. It also handles PTO optimization, jet lag assessment, extended layover detection, and multi-destination itineraries built from one-way legs.

## Installation

Copy the skill folder into your Claude Code skills directory:

```bash
cp -r flight-search ~/.claude/skills/
```

Then invoke it in Claude Code with `/flight-search`.

## Usage

Tell Claude your travel plan in plain language:

```
/flight-search I want to go to Japan in late March. I have 8 PTO days.
```

The skill will:
1. Ask for your home airport and nearby airports (or read from memory/CLAUDE.md if configured)
2. Search all candidate airports via SerpAPI Google Flights
3. Rank results by Value Score within Round Trip and Multi-Country sections
4. Write a CSV to the current directory
5. Summarize the top 3 options in plain English

## Example output

`flights_japan_mar2026.csv`

```
ROUND TRIP
Rank,Value Score,Departure Airport,Drive Time,Drive Cost,Airline(s),Flight Numbers,Route,Depart,Arrive,Stops,Total Flying Hrs,Layover Cities,Flight Price,Total Cost,Booking Link,Notes
1,$1418,ORD,3h,$38,United,UA 837,ORD -> NRT,2026-03-27 11:30,2026-03-28 15:45,0,25.1,,,$1380,$1418,https://flights.google.com/...,
2,$1491,ORD,3h,$38,"Delta / Korean Air","DL 159 | KE 702",ORD -> ICN -> NRT,2026-03-27 13:00,2026-03-28 18:20,1,26.6,ICN,$1453,$1491,https://flights.google.com/...,
3,$1544,AAA,1h,$16,ANA,NH 112,AAA -> NRT,2026-03-27 09:15,2026-03-28 13:30,0,24.8,,,$1528,$1544,https://flights.google.com/...,

MULTI-COUNTRY
Rank,Value Score,Itinerary Type,Departure Airport,Drive Time,Drive Cost,Airline(s),Leg Breakdown,Route,Depart,Arrive,Total Flying Hrs,Countries + Days,Flight Price,Total Cost,L1 Booking Link,L2 Booking Link,L3 Booking Link,Notes
1,$1721,South Korea First,ORD,3h,$38,"United / Asiana","UA 837 | OZ 101 | NH 112",ORD -> ICN -> NRT -> ORD,2026-03-27 11:30,2026-04-07 14:00,28.4,"South Korea: 3 days / Japan: 8 days",$1683,$1721,https://flights.google.com/...,https://flights.google.com/...,https://flights.google.com/...,
```

After writing the file, Claude summarizes the top 3 options in plain English and flags any extended layover opportunities or PTO efficiency wins.

## Requirements

- `SERPAPI_API_KEY` environment variable -- get a key at [serpapi.com](https://serpapi.com)
- SerpAPI free plan includes 250 searches/month; a full multi-airport + multi-city search uses 15-25 calls

## Configuration

Before the first search, tell Claude your travel profile (or add it to your CLAUDE.md):

```
My home airport is XYZ (Your City, ST).
Nearby airports I will drive to:
- AAA (City Name): Xh Ym, NNN miles
- BBB (City Name): Xh Ym, NNN miles
My home time zone is US/Eastern.
```

The skill uses a drive cost estimate of $0.20/mile (gas only) and adds this to each option's total cost before ranking.

## License

MIT
