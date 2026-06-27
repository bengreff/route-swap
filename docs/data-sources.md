# Data Sources

## Bike Infrastructure

### OpenStreetMap (Primary Road Network)

- **What**: Complete road network with bike-relevant tags (cycleway, bicycle=designated, surface, etc.)
- **Format**: PBF extract, filtered to Austin metro bounding box
- **Update frequency**: Daily diffs available, we rebuild nightly
- **Cost**: Free (open data, ODbL license)
- **Source**: https://download.geofabrik.de/north-america/us/texas.html
- **Key tags we use**: `highway`, `cycleway`, `bicycle`, `surface`, `maxspeed`, `lit`, `smoothness`

### City of Austin Bicycle Facilities (Local Infrastructure)

- **What**: Authoritative dataset of all bike infrastructure in Austin — protected lanes, bike lanes, shared paths, trails. Includes "level of comfort" classification and All Ages and Abilities (AAA) network designation.
- **Format**: GeoJSON, Shapefile, CSV via SODA API
- **Update frequency**: Updated as infrastructure changes (roughly monthly)
- **Cost**: Free (open data)
- **Source**: https://data.austintexas.gov/Transportation-and-Mobility/Austin-Bicycle-Facilities/kfe9-st9c
- **API endpoint**: `https://data.austintexas.gov/resource/kfe9-st9c.geojson`
- **Key fields**: `facility_type`, `comfort_level`, `street_name`, `geometry`

### Austin Bicycle Facilities (Extended Dataset)

- **What**: Export from Comprehensive Transportation Network with Bike Facility IDs
- **Format**: GeoJSON/Shapefile
- **Source**: https://data.austintexas.gov/Transportation-and-Mobility/TRANSPORTATION_bicycle_facilities/uwbq-ycek
- **Notes**: Can join back to full transportation network via Bike Facility ID

### Future Bike Infrastructure (Planning Data)

- **What**: Planned bike paths from Austin Transit Partnership
- **Source**: https://project-connect-data-portal-atptx.hub.arcgis.com/datasets/future-bike-paths
- **Use**: Flag routes where infrastructure is coming soon ("This route will get a protected lane in Q2 2027")

## Traffic Data

### TomTom Traffic Flow API (Real-Time + Predictive)

- **What**: Real-time observed speeds and travel times. Historical patterns for future trip prediction.
- **Free tier**: 2,500 routing requests/day (no credit card required)
- **Paid tier**: $0.50/1,000 requests after free tier
- **Endpoints used**:
  - `calculateRoute` — point-to-point routing with traffic
  - `calculateRoute?departAt=` — future departure time (uses historical patterns)
- **Docs**: https://developer.tomtom.com/routing-api/documentation
- **Why TomTom over Google**: Better free tier, real-time traffic depth from connected vehicle data, no $200/mo minimum. TomTom processes billions of data points from connected vehicles.

### Fallback: Mapbox Directions API

- **What**: Routing with traffic-aware ETAs
- **Free tier**: 100,000 requests/month
- **Cost after free**: $2/1,000 requests
- **Use as**: Fallback if TomTom is down, or for comparison/validation

## Parking Data

### City of Austin Parking Meters (Open Data)

- **What**: Location and transaction data for city-operated parking meters
- **Source**: Austin Open Data Portal (data.austintexas.gov)
- **Use**: Real pricing data, occupancy patterns by time of day

### SpotHero / ParkMe API (Garage Pricing)

- **What**: Real-time pricing and availability for parking garages
- **Cost**: Partnership/API access required
- **Use**: Accurate garage pricing for downtown, UT campus, events
- **Alternative for MVP**: Manual survey of top 20 parking areas, update quarterly

### Crowdsourced Parking Difficulty

- **What**: User-reported parking difficulty after driving to a destination
- **Collection**: In-app prompt after a drive trip ("How long did parking take?")
- **Use**: Calibrate and improve the parking model over time

## Safety and Crash Data

### TxDOT Crash Records Information System (CRIS)

- **What**: All reported traffic crashes in Texas, including cyclist-involved
- **Format**: CSV export, geocoded
- **Source**: https://cris.dot.state.tx.us/public/Query/
- **Update frequency**: Quarterly
- **Use**: Crash density scoring per road segment (500m buffer analysis)
- **Key fields**: `crash_severity`, `manner_of_collision`, `bicycle_flag`, `latitude`, `longitude`

### Austin Vision Zero Data

- **What**: City's traffic fatality and serious injury data
- **Source**: Austin Transportation Department
- **Use**: Identify high-risk corridors and intersections

## Elevation

### USGS 3D Elevation Program (3DEP)

- **What**: High-resolution elevation data (1m or 1/3 arc-second)
- **Format**: GeoTIFF
- **Cost**: Free
- **Source**: https://apps.nationalmap.gov/downloader/
- **Use**: Calculate grade for each road segment, adjust bike speed estimates

### Alternative: Mapbox Terrain-RGB Tiles

- **What**: Elevation data encoded in RGB raster tiles
- **Cost**: Free tier sufficient
- **Use**: Client-side elevation profiles, simpler integration than raw DEM

## Weather

### OpenWeatherMap API

- **What**: Current conditions, hourly forecast, alerts
- **Free tier**: 1,000 calls/day (60/min)
- **Cost**: Free tier sufficient for MVP
- **Use**: Weather gate for notifications (suppress bike suggestion in thunderstorms, extreme heat)
- **Key data**: temperature, heat index, precipitation, wind speed/direction, alerts

## Geocoding

### Nominatim (OpenStreetMap Geocoder)

- **What**: Convert addresses/place names to lat/lng
- **Cost**: Free (self-hosted or public instance with rate limits)
- **Use**: Geocode calendar event locations
- **Limitation**: Public instance has 1 req/sec rate limit; self-host for production

### Mapbox Geocoding API

- **What**: Higher-quality geocoding with fuzzy matching
- **Free tier**: 100,000 requests/month
- **Cost**: $0.75/1,000 after free tier
- **Use**: Fallback for ambiguous locations that Nominatim can't resolve

## Calendar

### Google Calendar API

- **What**: Read-only access to user's calendar events
- **Scope**: `https://www.googleapis.com/auth/calendar.events.readonly`
- **Cost**: Free (Google Cloud project required)
- **Key endpoints**:
  - `events.list` — get events for a date range
  - `events.watch` — webhook for real-time calendar changes
- **Rate limits**: 1,000,000 queries/day per project

## Maps and Tiles

### CartoDB / CARTO Basemap Tiles

- **What**: Light, clean basemap tiles (used in demo)
- **Cost**: Free for non-commercial / low volume
- **URL pattern**: `https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png`

### OpenStreetMap Tiles

- **What**: Standard OSM map tiles
- **Cost**: Free (usage policy applies)
- **Use**: Fallback if CartoDB is unavailable

## Data Pipeline Summary

| Data Source | Update Frequency | Storage | Size (Austin) |
|---|---|---|---|
| OSM road network | Nightly rebuild | OSRM graph files | ~200 MB |
| Austin bike facilities | Weekly sync | PostGIS table | ~5 MB |
| TxDOT crash data | Quarterly import | PostGIS table | ~50 MB |
| Parking zones | Manual + crowdsourced | PostgreSQL | ~1 MB |
| Elevation (3DEP) | One-time import | GeoTIFF on disk | ~500 MB |
| Traffic cache | 15-min TTL | Redis | ~50 MB |
| Weather | Real-time (5 min cache) | Redis | <1 MB |

## API Cost Estimate (MVP, 500 users)

| Service | Monthly Usage | Monthly Cost |
|---|---|---|
| TomTom Routing | ~30,000 requests | Free (under 75K) |
| Mapbox Geocoding | ~10,000 requests | Free (under 100K) |
| OpenWeatherMap | ~5,000 requests | Free (under 30K) |
| Google Calendar API | ~50,000 requests | Free |
| Hosting (Fly.io) | 2 VMs + OSRM | ~$30 |
| PostgreSQL (Supabase) | Free tier | $0 |
| Redis (Upstash) | Free tier | $0 |
| **Total** | | **~$30/month** |
