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
