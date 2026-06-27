# Technical Design

## Routing Engine Deep Dive

### Custom Bike Routing with OSRM

OSRM (Open Source Routing Machine) is the core of the bike router. We run a custom instance with a modified bicycle Lua profile that incorporates Austin-specific infrastructure data.

#### Graph Construction Pipeline

```
                    ┌──────────────┐
                    │ OSM Extract  │
                    │ (Texas PBF)  │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ osmium filter│ (clip to Austin metro bbox)
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐     ┌─────────────────────┐
                    │ Augment with │◄────│ Austin Bike          │
                    │ local data   │     │ Facilities GIS       │
                    └──────┬───────┘     └─────────────────────┘
                           │
                    ┌──────▼───────┐     ┌─────────────────────┐
                    │ Tag safety   │◄────│ TxDOT crash data     │
                    │ scores       │     │ + traffic speed data  │
                    └──────┬───────┘     └─────────────────────┘
                           │
                    ┌──────▼───────┐
                    │ osrm-extract │ (custom bike.lua profile)
                    │ osrm-contract│
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ .osrm files  │ (ready to serve)
                    └──────────────┘
```

**Rebuild cadence**: Nightly at 3 AM CT. Austin bike infrastructure changes ~monthly, OSM updates daily, crash data quarterly.

#### Custom Bike Profile (bike.lua)

Key modifications to the default OSRM bicycle profile:

```lua
-- Road type preferences (speed factor, higher = preferred)
road_weights = {
  cycleway           = 1.0,    -- dedicated bike path: full speed
  protected_lane     = 0.95,   -- on-road but physically separated
  bike_lane          = 0.80,   -- painted lane, no physical barrier
  shared_lane        = 0.60,   -- sharrows
  residential_quiet  = 0.70,   -- low-speed residential (<25 mph limit)
  residential        = 0.55,   -- standard residential
  tertiary           = 0.40,   -- collector roads
  secondary          = 0.20,   -- arterials with no bike infra
  primary            = 0.10,   -- major arterials (Lamar, Burnet)
  trunk              = 0.02,   -- highways, avoid almost entirely
}

-- Elevation penalty
-- grade_pct: positive = uphill
function elevation_penalty(grade_pct)
  if grade_pct <= 0 then return 1.0 end      -- downhill: no penalty
  if grade_pct <= 2 then return 0.95 end      -- gentle: minimal
  if grade_pct <= 5 then return 0.75 end      -- moderate: noticeable
  if grade_pct <= 8 then return 0.50 end      -- steep: significant
  return 0.30                                  -- very steep: walk-your-bike
end

-- Intersection penalty
-- Unprotected left turns across high-speed traffic are dangerous
function intersection_penalty(crossing_type, road_speed_limit)
  if crossing_type == "signal_with_bike_phase" then return 0 end
  if crossing_type == "signal" then return 10 end        -- 10 sec penalty
  if road_speed_limit >= 40 then return 30 end           -- 30 sec for fast roads
  if road_speed_limit >= 30 then return 15 end
  return 5
end
```

#### Safety Scoring

Each road segment gets a safety score (1.0 to 5.0) stored in the `safety_segments` table:

```sql
CREATE TABLE safety_segments (
  osm_way_id    BIGINT PRIMARY KEY,
  safety_score  NUMERIC(2,1) NOT NULL,  -- 1.0 to 5.0

  -- Component scores (each 0.0 to 1.0)
  infra_score       NUMERIC(3,2),  -- bike infrastructure type
  traffic_score     NUMERIC(3,2),  -- inverse of adjacent traffic speed
  crash_score       NUMERIC(3,2),  -- inverse of crash density (500m buffer)
  intersection_score NUMERIC(3,2),  -- quality of intersection crossings
  lighting_score    NUMERIC(3,2),  -- street lighting presence
  surface_score     NUMERIC(3,2),  -- pavement quality

  -- Metadata
  geometry      GEOMETRY(LineString, 4326),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Spatial index for fast lookups
CREATE INDEX idx_safety_geom ON safety_segments USING GIST (geometry);
```

**Overall route safety** = length-weighted average of segment scores along the route.

### Drive Time + Parking Model

#### Real-Time Traffic Integration

**Provider**: TomTom Traffic Flow API (free tier: 2,500 requests/day)

```
GET https://api.tomtom.com/routing/1/calculateRoute/
    {start_lat},{start_lng}:{end_lat},{end_lng}/json
    ?traffic=true
    &departAt={future_datetime}
    &key={API_KEY}
```

For future trips, use `departAt` to get predicted traffic, not current.

**Caching strategy**:
- Cache drive times by (origin_zone, dest_zone, day_of_week, hour) — 15 min buckets
- Austin has ~20 meaningful zones, so the cache is small: 20 * 20 * 7 * 96 = ~268K entries
- Warm the cache with a background job that pre-fetches common corridors

#### Parking Difficulty Model

Parking difficulty is modeled per-zone with time-of-day adjustments:

```
parking_zones table:
┌────────────────────┬──────────┬──────────┬──────────┬──────────┐
│ zone               │ base_min │ rush_min │ event_min│ cost_avg │
├────────────────────┼──────────┼──────────┼──────────┼──────────┤
│ downtown_core      │ 8        │ 12       │ 20       │ $12      │
│ ut_campus          │ 6        │ 10       │ 15       │ $10      │
│ rainey_st          │ 10       │ 15       │ 25       │ $15      │
│ south_congress     │ 5        │ 8        │ 12       │ $5       │
│ east_austin        │ 2        │ 3        │ 5        │ $0       │
│ the_domain         │ 1        │ 2        │ 3        │ $0       │
│ zilker_area        │ 3        │ 5        │ 15       │ $0       │
│ north_loop         │ 2        │ 3        │ 5        │ $0       │
│ residential        │ 1        │ 1        │ 1        │ $0       │
└────────────────────┴──────────┴──────────┴──────────┴──────────┘
```

**Walk-from-parking estimate**: Based on zone parking density. Downtown = 3-5 min walk, suburbs = 1 min.

**Total drive time** = `traffic_api_time + parking_search_min + parking_walk_min`
**Total drive cost** = `parking_cost + (distance_miles * $0.67 IRS_rate)`

### Trip Comparison Engine

The comparison engine is the core business logic that takes a calendar event and produces a ranked comparison:

```python
@dataclass
class TripComparison:
    event: CalendarEvent
    
    # Drive
    drive_time_min: int         # curb-to-curb from traffic API
    drive_parking_search: int   # from parking model
    drive_parking_walk: int     # from parking model
    drive_total: int            # sum of above
    drive_cost: float           # parking + gas
    is_rush_hour: bool
    
    # Bike
    bike_time_min: int          # from OSRM at user's speed
    bike_lockup: int            # always 1 min
    bike_total: int
    bike_safety: float          # 1.0 - 5.0
    bike_route_desc: str        # human-readable route
    bike_protected_pct: int     # % on protected infrastructure
    bike_elevation_gain: int    # feet
    bike_geometry: list         # polyline for map display
    
    # Walk (if viable)
    walk_time_min: int | None
    walk_viable: bool           # under 30 min
    
    # Result
    winner: str                 # 'bike' | 'walk' | 'drive' | 'close'
    time_saved: int             # minutes saved vs driving
    money_saved: float
    co2_saved: float            # lbs

def compare_trip(event, user_profile, traffic_data, parking_data):
    """Core comparison function."""
    
    origin = geocode(event.location_from)
    destination = geocode(event.location_to)
    
    # Drive
    drive = get_drive_time(origin, destination, event.start_time)
    parking = get_parking_model(destination, event.start_time)
    drive_total = drive.time + parking.search_time + parking.walk_time
    drive_cost = parking.cost + (drive.distance_miles * 0.67)
    
    # Bike
    bike_route = get_bike_route(origin, destination, user_profile)
    bike_total = bike_route.time + 1  # lockup
    
    # Walk
    walk_route = get_walk_route(origin, destination, user_profile)
    walk_viable = walk_route.time <= 30
    
    # Determine winner
    candidates = [('drive', drive_total)]
    if bike_route.safety >= user_profile.comfort_threshold:
        candidates.append(('bike', bike_total))
    if walk_viable:
        candidates.append(('walk', walk_route.time))
    
    candidates.sort(key=lambda x: x[1])
    fastest_mode, fastest_time = candidates[0]
    
    if fastest_mode == 'drive':
        winner = 'drive'
    elif abs(drive_total - fastest_time) <= 5:
        winner = 'close'
    else:
        winner = fastest_mode
    
    return TripComparison(...)
```

### Calendar Integration

#### Google Calendar OAuth Flow

```
1. User clicks "Connect Calendar"
2. Redirect to Google OAuth consent screen
   Scope: calendar.events.readonly
3. Google redirects back with auth code
4. Exchange code for access + refresh token
5. Store encrypted refresh token in DB
6. Fetch events for current + next week
7. Set up webhook for real-time calendar changes
```

#### Event Location Extraction

Calendar events have messy location data. Extraction pipeline:

```
"John's office, 123 Main St"        → geocode directly
"Capital Factory"                    → known venue lookup, then geocode
"Zoom meeting"                       → skip (virtual, no commute)
"TBD"                                → skip
"Home"                               → user's saved home address
"Coffee with Sarah at Houndstooth"   → venue extraction → geocode
""  (no location)                    → skip
```

**Strategy**:
1. Check against known venues database (seeded with popular Austin locations)
2. Try Nominatim/Mapbox geocoding on the raw string
3. If geocoding fails, check if the title contains a known venue name
4. Fall back to asking the user to confirm/set the location (in-app prompt)

### Speed Calibration

Users can opt in to GPS tracking during rides. The system learns their actual speed:

```
user_speed = weighted_moving_average(
    recent_rides,
    weights=decay_by_recency,
    adjustments={
        'elevation': normalize_to_flat_equivalent,
        'wind': deduct_wind_effect,
        'stops': subtract_stoplight_wait_time,
    }
)
```

**Privacy**: Raw GPS tracks are processed and discarded. Only aggregate speed stats are stored.

### Notification System

#### Weekly Summary (Sunday 6 PM)

```
"This week: 9 of 13 trips are faster by bike.
You'd save 2hr 16min and $82 in parking.
Tap to see your week →"
```

#### Pre-Trip Notification (30 min before departure)

```
"Your 2:00 PM meeting at Capital Factory:
🚗 Drive: 29 min (traffic + parking) — $12
🚲 Bike: 15 min via Guadalupe lane — Free
Bike saves 14 min. Tap for directions →"
```

**Logic**: Only notify if bike/walk saves >= 5 min AND route safety >= user threshold AND weather is acceptable (no active thunderstorm).

#### Weather Gate

Check OpenWeatherMap API before sending notification:
- Temperature > 105°F heat index → suppress and note "too hot today"
- Active thunderstorm/rain → suppress
- Light rain → include but note "light rain expected"
- Wind > 20 mph → adjust bike time estimate and note

## API Design

### Core Endpoints

```
POST   /api/auth/google          # OAuth callback
DELETE /api/auth                  # Disconnect calendar

GET    /api/trips/weekly          # Get this week's trip comparisons
GET    /api/trips/:id             # Get single trip detail
POST   /api/trips/:id/feedback    # "I biked this trip" / "I drove"

GET    /api/routes/compare        # Ad-hoc comparison (origin, dest, time)
GET    /api/routes/bike/:id       # Get bike route geometry + details

GET    /api/profile               # User profile
PUT    /api/profile               # Update speed, threshold, etc.

GET    /api/stats                 # Cumulative impact stats
GET    /api/stats/weekly          # Weekly breakdown
```

### Response Format (Trip Comparison)

```json
{
  "id": "trip_abc123",
  "event": {
    "title": "Team Standup",
    "start": "2026-07-06T08:30:00-05:00",
    "calendar_event_id": "google_cal_xyz"
  },
  "from": {
    "name": "Home (Hyde Park)",
    "lat": 30.3050,
    "lng": -97.7270
  },
  "to": {
    "name": "UT Austin Campus",
    "lat": 30.2849,
    "lng": -97.7341
  },
  "distance_miles": 2.4,
  "drive": {
    "base_time": 14,
    "parking_search": 8,
    "parking_walk": 4,
    "total": 26,
    "cost": 11.61,
    "is_rush_hour": true
  },
  "bike": {
    "ride_time": 11,
    "lockup": 1,
    "total": 12,
    "safety_score": 4.2,
    "protected_pct": 70,
    "elevation_gain_ft": 45,
    "route_description": "Guadalupe bike lane → Speedway",
    "geometry": "encoded_polyline_string"
  },
  "walk": {
    "time": 48,
    "viable": false
  },
  "result": {
    "winner": "bike",
    "time_saved": 14,
    "money_saved": 11.61,
    "co2_saved_lbs": 2.1
  }
}
```

## Tech Stack Summary

| Layer | Technology | Rationale |
|---|---|---|
| Frontend | React + TypeScript | Industry standard, large ecosystem |
| Maps | Leaflet + CartoDB tiles | Free, lightweight, customizable |
| API | FastAPI (Python) | Async, fast, great for data-heavy backends |
| Bike routing | OSRM (self-hosted) | Open source, customizable profiles, fast |
| Drive routing | TomTom Routing API | Best real-time traffic, generous free tier |
| Geocoding | Nominatim (self-hosted) or Mapbox | Free (Nominatim) or affordable (Mapbox) |
| Database | PostgreSQL + PostGIS | Spatial queries, mature, reliable |
| Cache | Redis | Route cache, session store |
| Auth | Google OAuth 2.0 | Calendar access requires it anyway |
| Notifications | Firebase Cloud Messaging | Free push notifications |
| Weather | OpenWeatherMap API | Free tier sufficient |
| Hosting | Fly.io or Railway | Simple deployment, affordable |
