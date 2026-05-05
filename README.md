# flight-search

An agent skill that turns a rough travel plan into a fully researched, ranked flight comparison -- searching every airport you are willing to drive to, pricing in the cost of the drive, and writing the results to a structured CSV.

## About

Most flight search tools make you search one airport at a time and rank purely by price. This skill searches all of your candidate departure airports in a single run, adds driving cost to every option, and ranks by a Value Score that accounts for both price and total time in the air. It handles round trips and multi-country itineraries, detects extended layover opportunities at major hubs, and -- when you tell it how many PTO days you have -- optimizes for the most destination time per vacation day burned, including jet lag recovery windows.

## What it does

You describe your trip in plain language. The skill:

- **Searches every candidate airport** you are willing to drive to and returns the top options per departure airport, all priced via the SerpAPI Google Flights API
- **Factors in driving cost** at $0.20/mile (gas only) before ranking, so a cheaper fare from a farther airport is correctly compared against a pricier fare from home
- **Ranks by Value Score** -- `Total Cost + (Total Flying Hours x $20)` -- rather than raw price, so a red-eye with two connections does not beat a clean nonstop just because it is $30 cheaper
- **Builds multi-country itineraries** from separate one-way legs when you mention a second country, rather than just surfacing round-trip options that can never include a multi-destination routing
- **Suggests geographic pairings** you may not have considered -- e.g. if you are flying to Peru, it proactively prices Ecuador-first and Bolivia-after routings
- **Flags extended layover opportunities** at major hubs (ICN, IST, AMS, DXB, etc.) when layover duration exceeds 8 hours, with the price delta vs. a direct connection
- **Optimizes for PTO** when you provide a vacation day budget: computes PTO days burned per option (weekdays only, excluding US holidays), ranks by efficiency, and flags departure/return strategies like Friday-night red-eyes that maximize ground time without burning extra days
- **Assesses jet lag** by time-zone offset and flags options with insufficient recovery time before your first workday back
- **Writes a structured CSV** with separate ranked sections for Round Trip and Multi-Country options, booking links from Google Flights, and per-leg breakdown for multi-city itineraries

## Installation

Copy the skill folder into your agent's skills directory:

```bash
cp -r flight-search ~/.claude/skills/
```

Then invoke it with `/flight-search`.

## Usage

Tell the agent your travel plan in plain language:

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

After writing the file, the agent summarizes the top 3 options in plain English and flags any extended layover opportunities or PTO efficiency wins.

## Requirements

- `SERPAPI_API_KEY` environment variable -- get a key at [serpapi.com](https://serpapi.com)
- SerpAPI free plan includes 250 searches/month; a full multi-airport + multi-city search uses 15-25 calls

## Configuration

Before the first search, tell the agent your travel profile (or add it to your CLAUDE.md):

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
