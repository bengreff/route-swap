# Product Requirements Document

## Vision

RouteSwap is the first navigation app that proactively tells you when biking or walking beats driving — not per-trip, but across your entire weekly schedule.

## Target Users

### Primary: Urban Professionals (Austin launch)

- Age 25-45, commute to multiple locations per week (office, coworking, meetings, gym, social)
- Own or have access to a bike/e-bike
- Frustrated by parking cost and traffic but default to driving out of habit
- Would bike more if they knew which trips were actually faster and which routes were safe

### Secondary: Employers / HR / Sustainability Leads

- Companies with wellness programs, sustainability goals, or commuter benefits
- Want to incentivize employees to reduce car trips
- Need data on commute patterns and potential impact

### Tertiary: City Transportation Planners

- Need data on where bike infrastructure investment has the most impact
- Want to understand latent demand for cycling on specific corridors

## Core User Stories

### Calendar Integration

- As a user, I connect my Google/Outlook calendar so RouteSwap automatically knows where I'm going each week
- As a user, I see my weekly trips with drive vs. bike vs. walk comparisons for each one
- As a user, I get a Sunday evening summary: "This week, 8 of your 12 trips are faster by bike"

### Route Comparison

- As a user, I see true door-to-door time for driving (including parking search + walk from lot + cost), not just curb-to-curb
- As a user, I see bike time based on my actual speed, not a generic assumption
- As a user, I see a safety rating for each bike route (protected lanes %, traffic speed, crash history)
- As a user, I can set my comfort threshold ("only show bike routes rated 3+ stars")

### Smart Notifications

- As a user, I get a push notification before a trip: "Bike is 6 min faster than driving to Capital Factory right now"
- As a user, notifications factor in current weather and real-time traffic
- As a user, I can snooze or disable notifications per-trip or globally

### Progress Tracking

- As a user, I see cumulative stats: time saved, money saved, CO2 avoided, calories burned
- As a user, I can share my monthly impact report
- As a user, I earn streaks for consecutive bike/walk commute days

## Key Differentiators

| Feature | Google Maps | Citymapper | RouteSwap |
|---|---|---|---|
| Calendar integration | No | No | Yes |
| Weekly batch analysis | No | No | Yes |
| Door-to-door with parking | No | Partial | Yes |
| Personalized bike speed | No | No | Yes |
| Route safety scoring | No | No | Yes (1-5 stars) |
| Proactive notifications | No | Limited | Yes |
| Parking cost modeling | No | No | Yes |
| Rush hour prediction | Current only | Current only | Predictive |

## MVP Scope (Austin Only)

### In Scope

- Google Calendar integration (read-only, OAuth)
- Bike route finding using OpenStreetMap + Austin bike facility data
- Drive time with parking model (location-based parking difficulty + cost database)
- Real-time traffic integration for drive time estimates
- Route safety scoring from infrastructure data + crash reports
- Weekly summary view (web app, mobile-responsive)
- Push notifications for upcoming trips where bike wins
- Basic user profile (bike speed, comfort threshold)

### Out of Scope for MVP

- Turn-by-turn navigation (use external app for now)
- Native mobile app (PWA first)
- Multi-city support (Austin only for data quality)
- Social features / leaderboards
- E-bike rental integration
- Public transit comparison
- Outlook/Apple Calendar (Google Calendar first)

## Success Metrics

- **Activation**: User connects calendar and views first weekly summary
- **Engagement**: User checks RouteSwap at least 2x/week
- **Conversion**: User bikes/walks a trip they would have driven (self-reported or GPS-verified)
- **Retention**: 30-day retention after activation
- **Impact**: Aggregate car trips replaced per week across user base

## Pricing Model (Planned)

### Consumer

- Free tier: 5 trips/week analyzed, basic comparison
- Pro ($4.99/mo): Unlimited trips, notifications, detailed safety data, progress tracking

### Enterprise (B2B)

- Per-employee pricing for companies with commuter/wellness programs
- Admin dashboard with aggregate impact data
- Custom integrations (Slack notifications, benefits platforms)
- $3-8/employee/month depending on volume

### Municipal

- Annual license for city transportation departments
- Anonymized aggregate data on cycling demand by corridor
- Infrastructure ROI modeling (if you build a protected lane on X street, Y% of car trips convert)
- Custom pricing based on city size
