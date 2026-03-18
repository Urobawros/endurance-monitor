# Feature: Course Map Overlay

## Need

The coach must be able to:
1. Import the official route provided by the organisation
2. Overlay it on a satellite view (Google / Mapbox)
3. See the horses' positions relative to the route
4. Immediately detect an orientation error (wrong path, missed turn)

---

## Formats provided by organisations

| Format | Frequency | Processing |
|--------|-----------|-----------|
| **GPX** | Very common | Parse → GeoJSON → Mapbox layer ✅ easy |
| **KML / KMZ** | Common | Parse → GeoJSON → Mapbox layer ✅ easy |
| **GeoJSON** | Increasingly common | Mapbox layer direct ✅ trivial |
| **PDF / scanned image** | Older events | Manual georeferencing ⚠️ complex |
| **PNG/JPG image** | Sometimes | Georeferencing by corner alignment ⚠️ |

---

## Feature architecture

```
┌─────────────────────────────────────────────────────────┐
│  Coach PWA — Map view                                   │
│                                                         │
│  [📂 Import route]  [🛰 Satellite] [🗺 Terrain]         │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Basemap: Mapbox Satellite Streets                │  │
│  │                                                   │  │
│  │  Layer 1: Official route (GPX/KML)                │  │
│  │    ── thick blue line + waypoints ──              │  │
│  │                                                   │  │
│  │  Layer 2: Horse positions (live)                  │  │
│  │    🟢 Aramis  ●──── GPS track covered             │  │
│  │    🟡 Kahina  ● (offline)                         │  │
│  │                                                   │  │
│  │  Layer 3: Deviation alerts                        │  │
│  │    🔴 Red zone if horse > Nm from route           │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  Status bar:                                            │
│  ⚠️  ARAMIS — Deviation 240m from official route       │
└─────────────────────────────────────────────────────────┘
```

---

## Implementation

### 1. GPX / KML / GeoJSON import

```javascript
// Parsing GPX/KML → GeoJSON via toGeoJSON library
import { gpx, kml } from '@mapbox/togeojson';

function importCourse(file) {
  const ext = file.name.split('.').pop().toLowerCase();
  const parser = new DOMParser();
  const dom = parser.parseFromString(await file.text(), 'text/xml');

  let geojson;
  if (ext === 'gpx')           geojson = gpx(dom);
  else if (ext === 'kml')      geojson = kml(dom);
  else if (ext === 'kmz')      geojson = await parseKMZ(file); // unzip + kml
  else if (ext === 'geojson')  geojson = JSON.parse(await file.text());

  // Extract LineStrings (route) and Points (waypoints / vet checks)
  const route    = geojson.features.filter(f => f.geometry.type === 'LineString');
  const waypoints = geojson.features.filter(f => f.geometry.type === 'Point');

  addRouteLayer(route);
  addWaypointMarkers(waypoints);
  fitMapBounds(geojson);
  storeCourseForDeviationCheck(route); // for offline calculation
}
```

### 2. Mapbox GL display

```javascript
// Satellite basemap
map.setStyle('mapbox://styles/mapbox/satellite-streets-v12');

// Official route
map.addSource('official-route', { type: 'geojson', data: routeGeojson });
map.addLayer({
  id: 'official-route-line',
  type: 'line',
  source: 'official-route',
  paint: {
    'line-color': '#3B82F6',   // blue
    'line-width': 3,
    'line-dasharray': [2, 1],  // light dash to avoid obscuring the ground
    'line-opacity': 0.85
  }
});

// Waypoints / vet checks
map.addLayer({
  id: 'waypoints',
  type: 'symbol',
  source: 'waypoints-source',
  layout: {
    'icon-image': 'veterinary-15',
    'text-field': ['get', 'name'],
    'text-size': 11,
    'text-offset': [0, 1.2]
  }
});
```

### 3. Image georeferencing (PDF/scan fallback)

For organisations that only provide a scanned image:

```
Alignment interface:
1. Coach imports the image (PNG/JPG/PDF→PNG)
2. Two-step interface:
   a. Click a reference point on the image (e.g. start)
      → enter lat/lon for that point (from the route booklet)
   b. Repeat for 3 non-collinear points
3. Affine transformation → Mapbox ImageSource with 4 calculated corners
4. Image displayed as semi-transparent overlay (opacity 0.5)

map.addSource('scanned-map', {
  type: 'image',
  url: objectURL,
  coordinates: [
    [lon_NW, lat_NW],  // NW corner
    [lon_NE, lat_NE],  // NE corner
    [lon_SE, lat_SE],  // SE corner
    [lon_SW, lat_SW]   // SW corner
  ]
});
map.addLayer({
  id: 'scanned-map-layer',
  type: 'raster',
  source: 'scanned-map',
  paint: { 'raster-opacity': 0.5 }
});
```

### 4. Deviation detection (off-course alert)

Calculates the distance from the horse's GPS point to the nearest segment of the official route.

```javascript
import * as turf from '@turf/turf';

// Pre-compute the route LineString (once on load)
let officialRoute; // GeoJSON LineString

function checkDeviation(horse) {
  if (!officialRoute) return;
  
  const horsePoint = turf.point([horse.lon, horse.lat]);
  const snapped = turf.nearestPointOnLine(officialRoute, horsePoint, { units: 'meters' });
  const distance = snapped.properties.dist; // metres
  
  if (distance > DEVIATION_THRESHOLD_M) { // e.g. 150m
    triggerAlert(horse, distance);
    // Flash red on the map marker
    // Show banner: "⚠️ ARAMIS — Deviation 240m"
    // Optional: auto-send push instruction to rider if deviation > 200m
  }
  
  return { distance, snappedPoint: snapped };
}

// Called on every new point received via WebSocket
ws.on('update', (data) => {
  updateHorseMarker(data);
  checkDeviation(data);
});
```

**Recommended thresholds:**
- < 50m: on route (GPS + track width)
- 50–150m: tolerance (GPS drift, vegetation)
- > 150m: deviation alert
- > 500m: critical alert (confirmed wrong direction)

---

## Course storage on the backend

The route is uploaded once per race and cached:

```
POST /races/:race_id/course
  Content-Type: multipart/form-data
  Body: GPX/KML/GeoJSON file

Backend:
  → Parses and converts to normalised GeoJSON
  → Stores in courses/{race_id}.geojson
  → Served via GET /races/:race_id/course

Coach PWA:
  → Automatically loads the route when the race is opened
```

---

## Coach UX

**Map toolbar:**
```
[📂 Route]  [🛰 Satellite / 🗺 Terrain]  [👁 Route ON/OFF]  [📐 Distance]
```

**Horse panel (enhanced):**
```
Aramis   🟢 LIVE
❤️  142 bpm   ⚡ 18.4 km/h   🐴 Trot
📍 On route (+23m)                ← green
```

```
Kahina   🟢 LIVE
❤️  138 bpm   ⚡ 19.1 km/h   🐴 Pace
📍 ⚠️ Deviation 247m from route   ← orange/red
```

**Deviation alert:**
- Red banner at the top of the screen
- Quick button: [📣 Send instruction "Return to route"]
- Option: automatic vibration on rider's phone if deviation > 200m
