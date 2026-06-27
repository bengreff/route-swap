# RouteSwap

**Your commute is lying to you.**

RouteSwap analyzes your weekly calendar and finds trips where biking or walking is faster than driving — when you factor in traffic, parking search time, and parking cost. The things Google Maps ignores.

## The Problem

Google Maps says "Drive: 8 min." But in downtown Austin, the real door-to-door time is 28 min + $12 in parking. Meanwhile, a 12-minute bike ride on a protected lane gets you there faster, for free. Nobody shows you this comparison across your entire week.

## What RouteSwap Does

- Connects to your calendar and analyzes every trip
- Compares drive (with real parking data) vs. bike (on safe routes) vs. walk
- Shows you which trips are faster without a car
- Learns your actual cycling speed, not Google's flat 8 mph assumption
- Scores bike routes for safety (protected lanes, traffic speed, crash data)
- Models time-of-day traffic patterns, not just current conditions

## Why Not Google Maps?

1. **Bad bike routing** — routes on dangerous arterials, misses trails and protected lanes
2. **No parking model** — drive time is curb-to-curb, not door-to-door
3. **No schedule integration** — can't analyze your whole week at once
4. **No proactive comparison** — you manually switch between modes
5. **One speed fits nobody** — flat 8 mph assumption for all cyclists
6. **No route comfort rating** — doesn't distinguish protected lane from 45 mph arterial

## Project Structure

```
route-swap/
├── demo/                  # Interactive prototype (single HTML file)
│   └── index.html         # Demo with mock Austin data + Leaflet map
├── docs/                  # Planning and design documents
│   ├── product-requirements.md
│   ├── architecture.md
│   ├── technical-design.md
│   ├── data-sources.md
│   └── roadmap.md
└── README.md
```

## Demo

Open `demo/index.html` in any browser. No build tools or server required. Shows a mock week for "Alex," a startup founder in Austin — 9 of 13 trips would have been faster by bike or foot.

## Status

Pre-seed. Prototype complete. Building toward MVP focused on Austin, TX.

## Docs

- [Product Requirements](docs/product-requirements.md) — what we're building and for whom
- [Architecture](docs/architecture.md) — system design and component overview
- [Technical Design](docs/technical-design.md) — routing engine, traffic integration, scoring
- [Data Sources](docs/data-sources.md) — APIs, datasets, and infrastructure data
- [Roadmap](docs/roadmap.md) — development phases and milestones
