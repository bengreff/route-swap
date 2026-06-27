# Development Roadmap

## Phase 0: Foundation (Weeks 1-2)

**Goal**: Project scaffolding, data pipeline, proof that the routing works with real data.

### Tasks

- [ ] Set up monorepo structure (frontend + API + OSRM config)
- [ ] Provision PostgreSQL with PostGIS (Supabase free tier)
- [ ] Download and clip Texas OSM extract to Austin metro bbox
- [ ] Import Austin Bicycle Facilities GIS data into PostGIS
- [ ] Import TxDOT crash data (cyclist-involved) into PostGIS
- [ ] Build safety scoring pipeline: compute per-segment scores from infra + crash data
- [ ] Set up OSRM Docker container with custom bike.lua profile
- [ ] Augment OSRM graph with Austin bike facility data
- [ ] Validate: route between 10 known Austin origin-destination pairs, compare to manual knowledge
- [ ] Import elevation data, add grade adjustments to routing profile

### Deliverable

Working CLI that takes two lat/lng points and returns a bike route with time, distance, safety score, and geometry. Verified against real Austin routes.

---

## Phase 1: Core API (Weeks 3-5)

**Goal**: Backend API that compares drive vs bike vs walk for a given trip.

### Tasks

- [ ] Set up FastAPI project with project structure
- [ ] Implement bike routing endpoint (wraps OSRM)
- [ ] Integrate TomTom Routing API for drive times with traffic
- [ ] Build parking model (zone-based lookup with time-of-day adjustment)
- [ ] Implement walk routing (OSRM pedestrian profile, viability threshold)
- [ ] Build trip comparison engine (combine all three modes, determine winner)
- [ ] Add route caching (Redis, keyed by origin/dest zone + departure hour)
- [ ] Set up geocoding pipeline (Nominatim with Mapbox fallback)
- [ ] Write API tests for comparison engine with known Austin routes
- [ ] Deploy API to Fly.io

### Deliverable

`GET /api/routes/compare?from=30.305,-97.727&to=30.285,-97.734&depart=2026-07-06T08:30` returns a full trip comparison JSON with drive (including parking), bike (with safety), and walk times.

---

## Phase 2: Calendar Integration (Weeks 6-7)

**Goal**: Connect Google Calendar, extract trips, batch-analyze a week.

### Tasks

- [ ] Set up Google Cloud project, configure OAuth consent screen
- [ ] Implement OAuth flow (auth code → access/refresh tokens)
- [ ] Build calendar sync service (fetch events for current + next week)
- [ ] Implement location extraction pipeline (geocode event locations, venue matching)
- [ ] Build known venues database (seed with top 100 Austin locations)
- [ ] Handle edge cases: virtual meetings (skip), missing locations (prompt user), "Home"
- [ ] Implement weekly batch analysis (compare all trips, compute summary stats)
- [ ] Set up calendar webhook for real-time event changes
- [ ] Store trip comparisons in database with user association
- [ ] API endpoints: `GET /api/trips/weekly`, `GET /api/trips/:id`

### Deliverable

User connects Google Calendar, RouteSwap fetches their week, and returns a list of trip comparisons with an optimization score.

---

## Phase 3: Web App MVP (Weeks 8-11)

**Goal**: User-facing web application with calendar view, map, and trip cards.

### Tasks

- [ ] Scaffold React + TypeScript + Vite project
- [ ] Implement Google OAuth login flow in frontend
- [ ] Build weekly summary dashboard (optimization gauge, stat cards)
- [ ] Build trip comparison cards (port design from demo prototype)
- [ ] Integrate Leaflet map with real bike route geometries
- [ ] Implement day/mode filtering
- [ ] Build trip detail view with expandable breakdowns
- [ ] Add user profile page (bike speed, e-bike toggle, comfort threshold)
- [ ] Build settings page (notification preferences)
- [ ] Implement PWA manifest + service worker for offline cached data
- [ ] Mobile-responsive testing on iOS Safari and Android Chrome
- [ ] Build "Why Google Maps Can't Do This" page (adapted from demo)
- [ ] Feedback mechanism: "I biked this trip" / "I drove" per-trip buttons
- [ ] Deploy frontend to Cloudflare Pages or Vercel

### Deliverable

Fully functional web app where a user can connect their calendar, see their weekly trip comparisons on a map, filter by day/mode, and view detailed breakdowns. Works on mobile.

---

## Phase 4: Notifications + Weather (Weeks 12-13)

**Goal**: Proactive push notifications before trips where biking wins.

### Tasks

- [ ] Integrate Firebase Cloud Messaging for web push notifications
- [ ] Build notification scheduler (30 min before each trip departure)
- [ ] Implement weather gate (OpenWeatherMap: suppress in storms, extreme heat)
- [ ] Build weekly summary notification (Sunday 6 PM)
- [ ] Add notification preferences (per-trip snooze, global on/off, quiet hours)
- [ ] Implement wind speed adjustment to bike time estimates
- [ ] Build notification history view in app
- [ ] Test notification delivery on iOS, Android, desktop browsers

### Deliverable

Users receive a push notification 30 min before trips where bike/walk wins: "Bike saves 14 min to Capital Factory. Tap for directions." Suppressed in bad weather.

---

## Phase 5: Speed Calibration + Progress (Weeks 14-16)

**Goal**: Learn user's actual speed, track cumulative impact.

### Tasks

- [ ] Build opt-in GPS ride tracking (start/stop in app)
- [ ] Implement speed extraction from GPS tracks (remove stop time, normalize for grade)
- [ ] Update user's bike speed model from ride history
- [ ] Build progress dashboard (time saved, money saved, CO2 avoided, calories)
- [ ] Add streaks and milestones (5 bike commute streak, $100 saved, etc.)
- [ ] Implement monthly impact summary (shareable card)
- [ ] Privacy controls: delete GPS data, opt out, data export

### Deliverable

User's bike time estimates improve with each ride. Dashboard shows cumulative impact.

---

## Phase 6: Polish + Beta Launch (Weeks 17-20)

**Goal**: Closed beta with 50-100 Austin users.

### Tasks

- [ ] User onboarding flow (connect calendar → set home → set bike speed → see first week)
- [ ] Error handling and edge case hardening
- [ ] Performance optimization (route cache warming, API response times < 500ms)
- [ ] Analytics integration (activation, engagement, conversion tracking)
- [ ] Feedback collection mechanism (in-app survey, bug reports)
- [ ] Landing page with waitlist
- [ ] Beta invite system
- [ ] Recruit beta testers (Austin cycling groups, incubator network, UT community)
- [ ] Monitor and iterate based on beta feedback

### Deliverable

50-100 active beta users in Austin, validated engagement metrics, list of top feature requests.

---

## Future Phases (Post-Beta)

### Phase 7: Enterprise / B2B

- Employer admin dashboard
- Aggregate commute data for sustainability reporting
- Integration with commuter benefits platforms
- Per-employee licensing

### Phase 8: Multi-City Expansion

- Templatize the data pipeline (OSM + city bike data + crash data)
- Next cities: Portland, Denver, Minneapolis (strong bike infrastructure)
- City-specific parking models
- Partner with local cycling advocacy organizations for data validation

### Phase 9: Advanced Features

- E-bike rental integration (Austin BCycle, Lime, Bird)
- Public transit multimodal comparison (bike + bus/rail)
- Turn-by-turn navigation (in-app, not redirect to Google Maps)
- Shade-aware routing (tree canopy + building shadow data)
- Air quality-aware routing (avoid high-pollution corridors)
- Native iOS and Android apps

### Phase 10: Platform / Data Business

- Anonymized aggregate data product for city planners
- Infrastructure ROI modeling ("if you build a protected lane on X, Y% of car trips convert")
- API for third-party integrations
- Partnership with real estate platforms (walkability/bikeability scores)

---

## Key Milestones

| Date | Milestone |
|---|---|
| Week 2 | Working bike router with Austin data |
| Week 5 | API returns real drive vs bike comparisons |
| Week 7 | Calendar connected, week analyzed |
| Week 11 | Web app live with real data |
| Week 13 | Push notifications working |
| Week 16 | Speed calibration + progress tracking |
| Week 20 | Beta launch, 50+ users |

## Resource Requirements

### MVP (Solo Developer or 2-Person Team)

- 1 full-stack engineer (Python + React + DevOps)
- 1 product/design (can be part-time or the same person)
- $30/month infrastructure
- $0 API costs (free tiers)
- Access to a bike for route validation in Austin

### Post-Beta Scaling

- +1 backend engineer (routing optimization, data pipeline)
- +1 mobile engineer (native apps)
- +1 growth/sales (B2B enterprise)
- Infrastructure scales to ~$200/month at 5,000 users
