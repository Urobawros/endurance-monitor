# Feature : Superposition carte parcours

## Besoin

Le coach doit pouvoir :
1. Importer le tracé officiel fourni par l'organisation
2. Le superposer sur une vue satellite (Google / Mapbox)
3. Voir la position des chevaux relative au tracé
4. Détecter immédiatement une erreur d'orientation (mauvais chemin, demi-tour raté)

---

## Formats fournis par les organisations

| Format | Fréquence | Traitement |
|--------|-----------|------------|
| **GPX** | Très courant | Parse → GeoJSON → Mapbox layer ✅ facile |
| **KML / KMZ** | Courant | Parse → GeoJSON → Mapbox layer ✅ facile |
| **GeoJSON** | De plus en plus | Mapbox layer direct ✅ trivial |
| **PDF / image scannée** | Anciennes épreuves | Géoréférencement manuel ⚠️ complexe |
| **Image PNG/JPG** | Parfois | Géoréférencement par calage de coins ⚠️ |

---

## Architecture de la fonctionnalité

```
┌─────────────────────────────────────────────────────────┐
│  PWA Coach — Vue carte                                  │
│                                                         │
│  [📂 Importer parcours]  [🛰 Satellite] [🗺 Terrain]   │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Fond : Mapbox Satellite Streets                  │  │
│  │                                                   │  │
│  │  Layer 1 : Tracé officiel (GPX/KML)               │  │
│  │    ── ligne bleue épaisse + waypoints ──          │  │
│  │                                                   │  │
│  │  Layer 2 : Positions chevaux (live)               │  │
│  │    🟢 Aramis  ●──── tracé GPS parcouru            │  │
│  │    🟡 Kahina  ● (offline)                         │  │
│  │                                                   │  │
│  │  Layer 3 : Alertes déviation                      │  │
│  │    🔴 Zone rouge si cheval > Nm du tracé          │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  Barre d'état :                                         │
│  ⚠️  ARAMIS — Déviation 240m du tracé officiel         │
└─────────────────────────────────────────────────────────┘
```

---

## Implémentation

### 1. Import GPX / KML / GeoJSON

```javascript
// Parsing GPX/KML → GeoJSON via librairie toGeoJSON
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

  // Extraire les LineStrings (tracé) et Points (waypoints / contrôles véto)
  const route    = geojson.features.filter(f => f.geometry.type === 'LineString');
  const waypoints = geojson.features.filter(f => f.geometry.type === 'Point');

  addRouteLayer(route);
  addWaypointMarkers(waypoints);
  fitMapBounds(geojson);
  storeCourseForDeviationCheck(route); // pour le calcul offline
}
```

### 2. Affichage Mapbox GL

```javascript
// Basemap satellite
map.setStyle('mapbox://styles/mapbox/satellite-streets-v12');

// Tracé officiel
map.addSource('official-route', { type: 'geojson', data: routeGeojson });
map.addLayer({
  id: 'official-route-line',
  type: 'line',
  source: 'official-route',
  paint: {
    'line-color': '#3B82F6',   // bleu
    'line-width': 3,
    'line-dasharray': [2, 1],  // pointillé léger pour ne pas masquer le sol
    'line-opacity': 0.85
  }
});

// Waypoints / contrôles vétérinaires
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

### 3. Géoréférencement image (fallback PDF/scan)

Pour les organisations qui ne fournissent qu'une image scannée :

```
Interface de calage :
1. Coach importe l'image (PNG/JPG/PDF→PNG)
2. Interface en 2 étapes :
   a. Cliquer un point de référence sur l'image (ex: départ)
      → entrer lat/lon de ce point (depuis le livret de route)
   b. Répéter pour 3 points non colinéaires
3. Transformation affine → ImageSource Mapbox avec 4 coins calculés
4. Image affichée en overlay semi-transparent (opacity 0.5)

map.addSource('scanned-map', {
  type: 'image',
  url: objectURL,
  coordinates: [
    [lon_NW, lat_NW],  // coin NW
    [lon_NE, lat_NE],  // coin NE
    [lon_SE, lat_SE],  // coin SE
    [lon_SW, lat_SW]   // coin SW
  ]
});
map.addLayer({
  id: 'scanned-map-layer',
  type: 'raster',
  source: 'scanned-map',
  paint: { 'raster-opacity': 0.5 }
});
```

### 4. Détection de déviation (off-course alert)

Calcul distance point GPS cheval → segment le plus proche du tracé officiel.

```javascript
import * as turf from '@turf/turf';

// Pré-calculer la LineString du parcours (une fois au chargement)
let officialRoute; // GeoJSON LineString

function checkDeviation(horse) {
  if (!officialRoute) return;
  
  const horsePoint = turf.point([horse.lon, horse.lat]);
  const snapped = turf.nearestPointOnLine(officialRoute, horsePoint, { units: 'meters' });
  const distance = snapped.properties.dist; // mètres
  
  if (distance > DEVIATION_THRESHOLD_M) { // ex: 150m
    triggerAlert(horse, distance);
    // Flash rouge sur le marqueur carte
    // Afficher bannière : "⚠️ ARAMIS — Déviation 240m"
    // Optionnel : envoyer instruction push à la cavalière
  }
  
  return { distance, snappedPoint: snapped };
}

// Appelé à chaque nouveau point reçu par WebSocket
ws.on('update', (data) => {
  updateHorseMarker(data);
  checkDeviation(data);
});
```

**Seuil recommandé :**
- < 50m : dans le parcours (GPS + track width)
- 50–150m : tolérance (GPS drift, végétation)
- > 150m : alerte déviation
- > 500m : alerte critique (mauvaise direction confirmée)

---

## Stockage parcours côté backend

Le tracé est uploadé une fois par course et mis en cache :

```
POST /races/:race_id/course
  Content-Type: multipart/form-data
  Body: fichier GPX/KML/GeoJSON

Backend :
  → Parse et convertit en GeoJSON normalisé
  → Stocke dans courses/{race_id}.geojson
  → Sert via GET /races/:race_id/course

PWA coach :
  → Charge automatiquement le tracé au chargement de la course
```

---

## UX Coach

**Toolbar carte :**
```
[📂 Parcours]  [🛰 Satellite / 🗺 Terrain]  [👁 Tracé ON/OFF]  [📐 Distance]
```

**Panneau cheval (augmenté) :**
```
Aramis   🟢 LIVE
❤️  142 bpm   ⚡ 18.4 km/h   🐴 Trot
📍 Sur le tracé (+23m)                ← vert
```

```
Kahina   🟢 LIVE
❤️  138 bpm   ⚡ 19.1 km/h   🐴 Amble
📍 ⚠️ Déviation 247m du tracé         ← orange/rouge
```

**Alerte déviation :**
- Bannière rouge en haut de l'écran
- Bouton rapide : [📣 Envoyer instruction "Retourne sur le tracé"]
- Option : vibration automatique sur le téléphone cavalière si déviation > 200m
