# EcoSpeed Advisor

Working prototype of a traffic-aware eco-driving assistant for a Toyota Yaris Cross Hybrid.

The app behaves like a lightweight route planner:

1. Browser asks for location permission.
2. Current GPS location becomes the start point.
3. User enters a destination.
4. Backend calculates a route using TomTom when `TOMTOM_API_KEY` is available.
5. If no key is available, the backend uses `sample_route.json` demo mode.
6. Frontend shows the route, advisory eco-speed, route summary, warnings, and speed profile chart.

## Safety disclaimer

EcoSpeed Advisor is **not autonomous driving** and does not control the vehicle. It only provides advisory speed suggestions. Always obey road signs, traffic, pedestrians, cyclists, weather, road conditions, and local law. If route, legal speed, curve, or traffic data is missing or uncertain, the prototype chooses conservative recommendations.

## What is implemented

- React + TypeScript frontend with Leaflet map and Recharts speed profile.
- Python FastAPI backend.
- Modular routing provider adapter.
- TomTom provider first, demo provider fallback.
- Eco-speed module for Toyota Yaris Cross Hybrid defaults.
- Smooth slowdown for congestion ahead.
- Curve slowdown heuristic using optional `curve_radius_m` or `curve_severity`.
- Conservative missing-data handling.
- Unit tests.

TomTom's Routing API can calculate routes between an origin and destination and account for current traffic and typical road speeds. TomTom's Geocoding/Search APIs convert a typed destination into coordinates. See the official TomTom developer docs for current API details.

## Project structure

```text
ecospeed-advisor/
  backend/
    app/
      main.py
      eco_speed_advisor.py
      routing_provider.py
      tomtom_provider.py
      models.py
    tests/
      test_eco_speed.py
    sample_route.json
    requirements.txt
    pytest.ini
  frontend/
    src/
      main.tsx
      api.ts
      types.ts
      styles.css
    package.json
    tsconfig.json
    index.html
```

## Install and run backend

```bash
cd backend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Health check:

```bash
curl http://localhost:8000/health
```

## Set TomTom API key

Without a key, the app runs in demo mode.

```bash
export TOMTOM_API_KEY="your_key_here"
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

## Install and run frontend

```bash
cd frontend
npm install
npm run dev
```

Open the Vite URL, usually:

```text
http://localhost:5173
```

Optional backend URL override:

```bash
VITE_API_BASE=http://localhost:8000 npm run dev
```

## Example API calls

### POST /route

```bash
curl -X POST http://localhost:8000/route \
  -H "Content-Type: application/json" \
  -d '{
    "start": {"lat": 48.2082, "lon": 16.3738},
    "destination": "Stephansplatz, Vienna"
  }'
```

### POST /eco-speed

```bash
curl -X POST http://localhost:8000/eco-speed \
  -H "Content-Type: application/json" \
  -d '{
    "current_speed_kmh": 70,
    "current_segment_index": 0,
    "segments": [
      {"distance_m": 800, "speed_limit_kmh": 70, "traffic_speed_kmh": 70, "road_name": "A", "start_index": 0, "end_index": 1},
      {"distance_m": 500, "speed_limit_kmh": 70, "traffic_speed_kmh": 30, "road_name": "B", "start_index": 1, "end_index": 2}
    ]
  }'
```

Expected behavior: the recommendation eases down before slower traffic and never exceeds the legal speed limit.

## Run tests

```bash
cd backend
pytest
```

## Eco-speed algorithm summary

For each route segment:

1. Build a safe speed from legal speed, traffic speed, and curve cap.
2. If speed limit is missing, use traffic speed but cap conservatively.
3. If traffic speed is missing, use legal speed and warn.
4. Look ahead up to 3 km.
5. Detect slower traffic or curve-reduced road conditions ahead.
6. Calculate a smooth entry speed using comfortable deceleration:

```text
recommended_speed = sqrt(target_speed^2 + 2 * comfortable_deceleration * distance_to_slow_zone)
```

7. Clamp speed so it never exceeds the legal limit.
8. Avoid sharp upward advice changes.
9. Smooth the profile and round displayed speed to the nearest 5 km/h.

## Car profile defaults

- Vehicle: Toyota Yaris Cross Hybrid
- Drivetrain: full hybrid petrol/electric
- Gearbox: e-CVT
- Mass: 1300 kg
- Baseline fuel consumption: 4.8 L/100 km
- Comfortable deceleration: 0.8 m/s²
- Max suggested acceleration: 1.0 m/s²
- Preferred behavior: smooth acceleration, early coasting, avoid hard braking

## Notes for future modules

- Add a speed-limit enrichment provider, such as TomTom Snap to Roads or another legal-speed source.
- Add road grade/elevation for better hybrid energy estimates.
- Add weather risk and road-surface caution factors.
- Add map matching for real-time current segment detection.
- Add HERE, Google Routes, OSRM, and Valhalla adapters under the same `RoutingProvider` interface.
- Add persistence of recent routes and simulated drive playback.
