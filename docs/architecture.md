# System Architecture

## Overview

RouteSwap is a web application (PWA) with a backend routing engine that compares driving, biking, and walking for calendar events. The architecture is designed to be city-specific at launch (Austin) with a clear path to multi-city expansion.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Client (PWA)                       │
│  React + Leaflet Map + Calendar View + Notifications    │
└─────────────────┬───────────────────────────────────────┘
                  │ REST/GraphQL API
┌─────────────────▼───────────────────────────────────────┐
│                    API Gateway                          │
│              (Auth, Rate Limiting, CORS)                │
└──────┬──────────┬──────────┬───────────┬────────────────┘
       │          │          │           │
┌──────▼──┐ ┌────▼────┐ ┌───▼────┐ ┌────▼─────┐
│ Calendar│ │ Routing │ │Parking │ │ User     │
│ Service │ │ Engine  │ │ Model  │ │ Service  │
└─────────┘ └────┬────┘ └────────┘ └──────────┘
                 │
       ┌─────────┼─────────┐
       │         │         │
  ┌────▼───┐ ┌───▼──┐ ┌───▼──────┐
  │ Bike   │ │Drive │ │ Walk     │
  │ Router │ │Router│ │ Router   │
  └────┬───┘ └───┬──┘ └───┬──────┘
       │         │         │
┌──────▼─────────▼─────────▼──────────────────────────────┐
│                    Data Layer                            │
│  ┌─────────┐ ┌──────────┐ ┌────────┐ ┌──────────────┐  │
│  │ OSM +   │ │ Traffic  │ │Parking │ │ Safety       │  │
│  │ Austin  │ │ Provider │ │Database│ │ Scores       │  │
│  │ Bike DB │ │(TomTom)  │ │        │ │ (crash data) │  │
│  └─────────┘ └──────────┘ └────────┘ └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Components

### 1. Client Application (PWA)

**Technology**: React, TypeScript, Leaflet, Workbox (service worker)

**Responsibilities**:
- Weekly calendar view with trip comparison cards
- Interactive map with route visualization
- Push notification handling
- Offline support for cached trip data
- User profile and settings

**Key views**:
- Weekly Summary (dashboard with optimization score)
- Trip Detail (drive vs bike vs walk breakdown)
- Map View (all routes, interactive)
- Settings (bike speed, comfort threshold, notification preferences)
- Progress (cumulative impact stats)

### 2. API Gateway

**Technology**: Node.js (Fastify) or Python (FastAPI)

**Responsibilities**:
- Authentication (Google OAuth for calendar access)
- API rate limiting and caching
- Request routing to microservices
- Response aggregation

### 3. Calendar Service

**Responsibilities**:
- Google Calendar API integration (OAuth 2.0, read-only scope)
- Extract events with locations from calendar
- Geocode location strings to lat/lng (using Nominatim or Mapbox Geocoding)
- Detect recurring trips and cache routes
- Trigger re-analysis when calendar changes (webhook or polling)

**Key challenges**:
- Calendar events often have vague locations ("John's office", "the usual spot")
- Need fuzzy matching and user confirmation for ambiguous locations
- Must handle recurring events efficiently (don't re-route weekly standup every time)

### 4. Routing Engine

The core differentiator. Three sub-routers that produce comparable door-to-door times.

#### 4a. Bike Router

**Technology**: Custom OSRM instance with bike profile, augmented with local data

**Data sources**:
- OpenStreetMap road network (base graph)
- Austin Bicycle Facilities dataset (protected lanes, bike lanes, shared paths)
- Elevation data (USGS 3DEP or Mapbox Terrain)
- Crash/incident data (TxDOT crash database)

**Routing profile** (custom OSRM Lua profile):
- Strongly prefer protected bike lanes and off-road trails
- Penalize high-speed roads without bike infrastructure
- Factor elevation gain (penalize steep climbs, less so for e-bikes)
- Avoid known dangerous intersections
- Output: time, distance, safety score, route geometry, turn-by-turn description

**Safety scoring algorithm**:
```
safety_score = weighted_average(
  protected_lane_pct * 0.35,
  avg_traffic_speed_penalty * 0.25,
  crash_density_score * 0.20,
  intersection_safety * 0.10,
  lighting_score * 0.05,
  surface_quality * 0.05
)
```

**Speed model**:
- Base speed from user profile (default 12 mph, adjustable)
- Adjust for elevation: -2 mph per 3% grade uphill, +1 mph downhill (capped)
- Adjust for surface: -1 mph for gravel/unpaved
- E-bike mode: base 15 mph, less elevation penalty
- Lock-up time: +1 min flat

#### 4b. Drive Router

**Technology**: Third-party routing API (TomTom or Mapbox Directions)

**Key addition — parking model**:
- Drive time from API = curb-to-curb
- Add: parking search time (from parking difficulty database)
- Add: walk from parking to destination (estimated from parking location)
- Add: parking cost (from rate database)

**Traffic model**:
- Real-time traffic for "leave now" queries
- Historical traffic patterns for future trips (TomTom Traffic Stats API)
- Rush hour detection and penalty

#### 4c. Walk Router

**Technology**: OSRM pedestrian profile or Mapbox Walking Directions

**Viability threshold**: Only shown for trips under 1.5 miles (~30 min walk)

**Adjustments**:
- User's walking speed (default 3.2 mph)
- Shade-aware routing as a future enhancement (tree canopy + building shadow data)

### 5. Parking Model

A standalone data service that answers: "How hard is it to park near [location] at [time]?"

**Outputs**:
- Estimated parking search time (minutes)
- Expected parking cost ($)
- Estimated walk from parking to destination (minutes)

**Data sources**:
- City of Austin parking meter data (open data)
- ParkMe / SpotHero API for garage pricing
- Crowdsourced difficulty scores from users
- Time-of-day patterns (lunch rush, event nights, weekend vs weekday)

**Location zones** (Austin-specific for MVP):
- Downtown core: high difficulty (8-15 min search, $8-15)
- UT Campus: high difficulty (5-10 min, $8-12)
- South Congress: moderate (5-8 min, $0-5)
- East Austin: low (2-3 min, $0)
- The Domain: low (2 min, free garage)
- Residential areas: minimal (1 min, free)

### 6. User Service

**Responsibilities**:
- User profile (bike speed, e-bike flag, comfort threshold)
- Trip history and progress tracking
- Notification preferences
- Speed calibration from completed rides (GPS tracking opt-in)

### 7. Data Layer

**Primary database**: PostgreSQL with PostGIS extension

**Tables** (key):
- `users` — profile, preferences, OAuth tokens
- `trips` — calendar events with computed comparisons
- `routes` — cached route geometries and timings
- `parking_zones` — parking difficulty/cost by area and time
- `safety_segments` — per-road-segment safety scores
- `ride_history` — GPS tracks for speed calibration (opt-in)

**Caching**: Redis for frequently-accessed routes and traffic data

**Routing graph**: OSRM `.osrm` files, rebuilt nightly from OSM + Austin bike data

## Infrastructure

### MVP Deployment

- **Hosting**: Railway, Render, or Fly.io (simple, affordable)
- **Database**: Managed PostgreSQL with PostGIS (e.g., Supabase or Neon)
- **OSRM**: Self-hosted Docker container (single instance for Austin)
- **CDN**: Cloudflare for static assets and map tiles
- **Background jobs**: BullMQ or similar for calendar sync, route pre-computation

### Scaling Path

- OSRM per-city instances behind a router
- Read replicas for PostgreSQL
- Route cache warming for recurring trips
- Edge-deployed API for latency

## Security and Privacy

- Calendar data is read-only (OAuth scope: `calendar.events.readonly`)
- GPS ride data is opt-in and can be deleted
- No location data shared with third parties
- All API communication over HTTPS
- OAuth tokens encrypted at rest
- GDPR/CCPA compliant data deletion on request
