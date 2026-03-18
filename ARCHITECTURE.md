# Architecture détaillée

## Principes fondamentaux

1. **Offline-first** : l'app cavalière écrit TOUJOURS en local d'abord. La connexion réseau est opportuniste.
2. **Store & Forward** : les données accumulées hors couverture sont transmises dès que le réseau revient.
3. **Coach = PWA** : interface web (iPad/navigateur), pas d'app store.

---

## Flux de données

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CHEVAL (×1 à 3)                                 │
│   Polar Equine ── BLE HRM Profile ────►                                 │
│   ESP32-S3 + IMU (licol) ── BLE Custom ──►                             │
│     • FFT → allure (pas/trot/amble/galop)                              │
│     • Symétrie → boiterie (grade AAEP)                                 │
└────────────────────────────────────────────────┬────────────────────────┘
                                                 │ BLE ×2
                                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    SMARTPHONE CAVALIÈRE                                  │
│                    (dans la poche — pas de fixation requise)            │
│                                                                         │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │  Collecte (toujours actif)                                       │  │
│   │  BLE → FC (Polar Equine) + allure/boiterie (ESP32) │ GPS → pos+vit │  │
│   │                        │                                         │  │
│   │                        ▼                                         │  │
│   │            ┌─────────────────────┐                               │  │
│   │            │   SQLite local      │  ← écriture systématique      │  │
│   │            │   (buffer offline)  │     ~200 bytes/point/1Hz      │  │
│   │            └─────────┬───────────┘     ~720 KB/heure/cheval      │  │
│   │                      │                                           │  │
│   │            Sync Manager (background)                             │  │
│   │            ┌─────────▼───────────┐                               │  │
│   │            │  Réseau disponible? │                               │  │
│   │            └──────┬──────┬───────┘                               │  │
│   │                   │ OUI  │ NON                                   │  │
│   │                   ▼      ▼                                       │  │
│   │           WebSocket  Continue                                    │  │
│   │           stream     buffering                                   │  │
│   │           live  +    (SQLite)                                    │  │
│   │           flush                                                  │  │
│   │           buffer                                                 │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│   ◄── Instruction coach (push notification → vibration)                 │
│       Cavalière sort le téléphone pour lire le détail si besoin         │
└─────────────────────────────────────────────┬───────────────────────────┘
                                              │ 4G (quand dispo)
                                              │ Live stream + batch flush
                                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         BACKEND (Node.js / VPS)                         │
│                                                                         │
│   WebSocket Server                                                      │
│   ├── /horse/:id     ← messages live + batches offline                  │
│   └── /coach         ← dashboard (lecture + envoi instructions)         │
│                                                                         │
│   Ingest                                                                │
│   ├── Message live  → INSERT InfluxDB                                   │
│   └── Batch flush   → INSERT InfluxDB (avec timestamps originaux)       │
│                       → ACK avec dernier seq_id reçu                    │
│                                                                         │
│   InfluxDB (time series)                                                │
│   └── measurement: telemetry                                            │
│       tags: horse_id, race_id, source [live|buffered]                   │
│       fields: fc, speed_kmh, allure, lat, lon, gait_confidence          │
│                                                                         │
│   REST API                                                              │
│   ├── GET  /races                     → liste des courses               │
│   ├── GET  /races/:id/data            → export complet post-course      │
│   ├── GET  /races/:id/data?horse=x    → données par cheval              │
│   └── POST /instructions/:horse_id   → envoyer instruction              │
│                                                                         │
└─────────────────────────────────────────────┬───────────────────────────┘
                                              │ WebSocket
                                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              DASHBOARD COACH — PWA (iPad Safari / navigateur)           │
│                                                                         │
│   ┌─────────────────────────┐  ┌──────────────────────────────────────┐ │
│   │     Carte Mapbox GL     │  │  Cheval 1 — Aramis      🟢 LIVE     │ │
│   │  🟢 Position live       │  │  ❤️  142 bpm  ⚡ 18.4 km/h  🐴 Trot  │ │
│   │  🟡 Dernière pos. connue│  │  [⬆ Accélérer] [⬇ Ralentir]        │ │
│   │     (hors couverture)   │  │  [→ Allure]   [⚑ Contrôle vét.]    │ │
│   │  Tracé GPS complet      │  ├──────────────────────────────────────┤ │
│   │    (live + replay buff) │  │  Cheval 2 — Kahina       🟡 OFFLINE │ │
│   └─────────────────────────┘  │  ❤️  128 bpm  ⚡ —  🐴 —            │ │
│                                │  Dernière MAJ : il y a 4 min 32s    │ │
│                                └──────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Store & Forward — Détail technique

### Sur le téléphone (app React Native)

```
Collecte → SQLite (TOUJOURS) → Sync Manager → Backend
```

**Table SQLite `telemetry` :**
```sql
CREATE TABLE telemetry (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  horse_id   TEXT NOT NULL,
  race_id    TEXT NOT NULL,
  ts         INTEGER NOT NULL,  -- Unix ms (timestamp original)
  fc         INTEGER,
  lat        REAL,
  lon        REAL,
  speed_kmh  REAL,
  heading    REAL,
  gait       TEXT,
  gait_conf  REAL,
  synced     INTEGER DEFAULT 0  -- 0 = en attente, 1 = confirmé backend
);
```

**Sync Manager (background task) :**
```
Toutes les 5 secondes :
  1. Vérifier connectivité réseau
  2. Si connecté :
     a. Ouvrir/maintenir WebSocket
     b. SELECT * FROM telemetry WHERE synced = 0 ORDER BY ts LIMIT 200
     c. Envoyer batch JSON [{...}, {...}]
     d. Attendre ACK backend (seq_id)
     e. UPDATE telemetry SET synced = 1 WHERE id <= seq_id
  3. Si déconnecté :
     a. Fermer WebSocket
     b. Continuer écriture SQLite
     c. Log "offline depuis X secondes"
```

**Capacité buffer :**
- 1 point/seconde × ~200 bytes = 720 KB/heure/cheval
- Un smartphone avec 1 GB libre peut stocker ~1400 heures
- En pratique : purge des données synced après 7 jours

### Sur le backend

```javascript
// Ingest unifié : live point ou batch offline
ws.on('message', (raw) => {
  const msg = JSON.parse(raw);
  
  if (msg.type === 'point') {
    // Point live temps réel
    influx.writePoint(msg);
    broadcast('coach', { type: 'update', horse: msg.horse_id, data: msg });
  }
  
  if (msg.type === 'batch') {
    // Flush buffer offline (timestamps passés)
    influx.writeBatch(msg.points);
    // Broadcaster la mise à jour au dashboard coach
    broadcast('coach', { type: 'replay', horse: msg.horse_id, points: msg.points });
    // ACK → téléphone peut marquer synced = 1
    ws.send(JSON.stringify({ type: 'ack', last_id: msg.last_id }));
  }
});
```

### Sur le dashboard coach (PWA)

**Comportement cheval offline :**
- Marqueur carte → icône 🟡 + tooltip "Hors couverture depuis Xm Xs"
- Dernier FC + vitesse connus affichés en grisé
- Tracé GPS interrompu avec ligne pointillée
- Quand données arrivent (flush) → tracé complété rétroactivement + animation "sync"

**Indicateur global :**
```
[ 🟢 Aramis — LIVE ]  [ 🟡 Kahina — Offline 4m32s ]  [ 🟢 Elvire — LIVE ]
```

---

## Format messages WebSocket

### Point live (téléphone → backend)
```json
{
  "type": "point",
  "horse_id": "aramis",
  "race_id": "2026-03-15_dax",
  "ts": 1741478400123,
  "fc": 142,
  "lat": 43.8921,
  "lon": -0.5012,
  "speed_kmh": 18.4,
  "heading": 247,
  "gait": "trot",
  "gait_conf": 0.91,
  "battery": 84
}
```

### Batch flush (téléphone → backend, après offline)
```json
{
  "type": "batch",
  "horse_id": "kahina",
  "race_id": "2026-03-15_dax",
  "last_id": 4821,
  "points": [
    { "id": 4700, "ts": 1741478200000, "fc": 136, "lat": 43.9012, "lon": -0.5100, "speed_kmh": 19.2, "gait": "amble" },
    { "id": 4701, "ts": 1741478201000, "fc": 138, ... },
    ...
  ]
}
```

### Instruction (backend → téléphone)
```json
{
  "type": "instruction",
  "code": "slow_down",
  "message": "Ralentis — FC trop haute",
  "vibration": [300, 200, 300],
  "ts": 1741478450000
}
```

---

## Algorithme détection d'allure (ESP32)

Exécuté **sur l'ESP32 du licol** (IMU ICM-42688-P). Le mouvement de la tête du cheval
est un signal plus propre et plus fiable que le pommeau de selle — c'est le même
principe utilisé par les systèmes professionnels (Equimetre, Lameness Locator).

```
Buffer : 150 échantillons accéléro @ 50 Hz = fenêtre 3 secondes

1. Isolation axe Z (vertical = rebond cheval)
   — L'IMU est fixe sur le licol : pas de correction d'orientation nécessaire
2. Application fenêtre Hanning (réduction aliasing)
3. FFT → spectre de puissance (trivial pour l'ESP32-S3 sur 150 samples)
4. Fréquence dominante f0 = argmax(PSD[1..10 Hz])
5. Ratio énergie latérale R = E(axe X) / E(axe Z)

Classification :
  f0 < 1.5 Hz              → "pas"
  1.5 ≤ f0 < 2.5 Hz        → "trot"
  f0 ≥ 2.5 Hz  ET R > 0.6  → "amble"
  f0 ≥ 2.0 Hz  ET R ≤ 0.6  → "galop"

Confiance = amplitude pic FFT / énergie totale spectre

Résultat transmis par BLE au téléphone toutes les 3 secondes :
  { gait: "trot", gait_conf: 0.91 }
```

**Avantage clé :** le téléphone reste dans la poche de la cavalière.
Pas de contrainte de fixation au pommeau.

---

## Codes vibration

| Instruction | Pattern | Signification |
|-------------|---------|---------------|
| Ralentir | 300ms ON, 200ms OFF, 300ms ON | ⬇ Réduire vitesse |
| Accélérer | 100ms×3 avec 100ms OFF | ⬆ Augmenter vitesse |
| Changer allure | 200ms×3 avec 100ms OFF | → Changer d'allure |
| Contrôle vétérinaire | 500ms continu | ⚑ Prochain contrôle |

---

## PWA Coach — Fonctionnalités

- **Progressive Web App** : installable sur iPad (Add to Home Screen), fonctionne offline pour la consultation
- **Authentification** : token simple partagé (pas de compte utilisateur)
- **Vues** :
  - Carte live (Mapbox GL)
  - Timeline FC × vitesse par cheval (Recharts)
  - Log des instructions envoyées
- **Post-course** :
  - Replay GPS complet (données live + données buffered reconstituées)
  - Export CSV / JSON
  - Courbes : FC × vitesse, zones effort, V200, FC récupération aux contrôles

---

## Stack & Déploiement

```
app/          React Native (Expo) — iOS + Android
backend/      Node.js + ws + InfluxDB (VPS existant)
dashboard/    React + Vite PWA (Mapbox GL, Recharts, WebSocket)
```

```bash
# Backend
cd backend && npm install && node server.js

# App (dev)
cd app && npx expo start

# Dashboard
cd dashboard && npm run dev
# Build PWA
npm run build  # → dist/ à servir via nginx
```
