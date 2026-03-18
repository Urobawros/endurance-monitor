# Detailed Architecture

## Core principles

1. **Offline-first**: the rider app ALWAYS writes locally first. Network connectivity is opportunistic.
2. **Store & Forward**: data accumulated without coverage is transmitted as soon as the network returns.
3. **Coach = PWA**: web interface (iPad/browser), no app store required.

---

## Data flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         HORSE (×1 to 3)                                 │
│   Polar Equine ── BLE HRM Profile ────►                                 │
│   ESP32-S3 + IMU (halter) ── BLE Custom ──►                            │
│     • FFT → gait (walk/trot/pace/canter)                               │
│     • Symmetry → lameness (AAEP grade)                                 │
└────────────────────────────────────────────────┬────────────────────────┘
                                                 │ BLE ×2
                                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    RIDER SMARTPHONE                                      │
│                    (in pocket — no mounting required)                   │
│                                                                         │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │  Collection (always active)                                      │  │
│   │  BLE → HR (Polar Equine) + gait/lameness (ESP32) │ GPS → pos+speed │  │
│   │                        │                                         │  │
│   │                        ▼                                         │  │
│   │            ┌─────────────────────┐                               │  │
│   │            │   Local SQLite      │  ← systematic write           │  │
│   │            │   (offline buffer)  │     ~200 bytes/point/1Hz      │  │
│   │            └─────────┬───────────┘     ~720 KB/hour/horse        │  │
│   │                      │                                           │  │
│   │            Sync Manager (background)                             │  │
│   │            ┌─────────▼───────────┐                               │  │
│   │            │  Network available? │                               │  │
│   │            └──────┬──────┬───────┘                               │  │
│   │                   │ YES  │ NO                                    │  │
│   │                   ▼      ▼                                       │  │
│   │           WebSocket  Continue                                    │  │
│   │           stream     buffering                                   │  │
│   │           live  +    (SQLite)                                    │  │
│   │           flush                                                  │  │
│   │           buffer                                                 │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│   ◄── Coach instruction (push notification → vibration)                 │
│       Rider takes out phone to read detail if needed                    │
└─────────────────────────────────────────────┬───────────────────────────┘
                                              │ 4G (when available)
                                              │ Live stream + batch flush
                                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         BACKEND (Node.js / VPS)                         │
│                                                                         │
│   WebSocket Server                                                      │
│   ├── /horse/:id     ← live messages + offline batches                  │
│   └── /coach         ← dashboard (read + send instructions)             │
│                                                                         │
│   Ingest                                                                │
│   ├── Live message  → INSERT InfluxDB                                   │
│   └── Batch flush   → INSERT InfluxDB (with original timestamps)        │
│                       → ACK with last received seq_id                   │
│                                                                         │
│   InfluxDB (time series)                                                │
│   └── measurement: telemetry                                            │
│       tags: horse_id, race_id, source [live|buffered]                   │
│       fields: fc, speed_kmh, allure, lat, lon, gait_confidence          │
│                                                                         │
│   REST API                                                              │
│   ├── GET  /races                     → list of races                   │
│   ├── GET  /races/:id/data            → full post-race export           │
│   ├── GET  /races/:id/data?horse=x    → data per horse                  │
│   └── POST /instructions/:horse_id   → send instruction                 │
│                                                                         │
└─────────────────────────────────────────────┬───────────────────────────┘
                                              │ WebSocket
                                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              COACH DASHBOARD — PWA (iPad Safari / browser)              │
│                                                                         │
│   ┌─────────────────────────┐  ┌──────────────────────────────────────┐ │
│   │     Mapbox GL Map       │  │  Horse 1 — Aramis       🟢 LIVE     │ │
│   │  🟢 Live position       │  │  ❤️  142 bpm  ⚡ 18.4 km/h  🐴 Trot  │ │
│   │  🟡 Last known pos.     │  │  [⬆ Speed up] [⬇ Slow down]        │ │
│   │     (out of coverage)   │  │  [→ Gait]     [⚑ Vet check]        │ │
│   │  Full GPS track         │  ├──────────────────────────────────────┤ │
│   │    (live + buff replay) │  │  Horse 2 — Kahina        🟡 OFFLINE │ │
│   └─────────────────────────┘  │  ❤️  128 bpm  ⚡ —  🐴 —            │ │
│                                │  Last update: 4 min 32s ago         │ │
│                                └──────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Store & Forward — Technical detail

### On the phone (React Native app)

```
Collection → SQLite (ALWAYS) → Sync Manager → Backend
```

**SQLite `telemetry` table:**
```sql
CREATE TABLE telemetry (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  horse_id   TEXT NOT NULL,
  race_id    TEXT NOT NULL,
  ts         INTEGER NOT NULL,  -- Unix ms (original timestamp)
  fc         INTEGER,
  lat        REAL,
  lon        REAL,
  speed_kmh  REAL,
  heading    REAL,
  gait       TEXT,
  gait_conf  REAL,
  synced     INTEGER DEFAULT 0  -- 0 = pending, 1 = backend confirmed
);
```

**Sync Manager (background task):**
```
Every 5 seconds:
  1. Check network connectivity
  2. If connected:
     a. Open/maintain WebSocket
     b. SELECT * FROM telemetry WHERE synced = 0 ORDER BY ts LIMIT 200
     c. Send JSON batch [{...}, {...}]
     d. Wait for backend ACK (seq_id)
     e. UPDATE telemetry SET synced = 1 WHERE id <= seq_id
  3. If disconnected:
     a. Close WebSocket
     b. Continue writing to SQLite
     c. Log "offline for X seconds"
```

**Buffer capacity:**
- 1 point/second × ~200 bytes = 720 KB/hour/horse
- A smartphone with 1 GB free can store ~1400 hours
- In practice: purge synced data after 7 days

### On the backend

```javascript
// Unified ingest: live point or offline batch
ws.on('message', (raw) => {
  const msg = JSON.parse(raw);
  
  if (msg.type === 'point') {
    // Real-time live point
    influx.writePoint(msg);
    broadcast('coach', { type: 'update', horse: msg.horse_id, data: msg });
  }
  
  if (msg.type === 'batch') {
    // Offline buffer flush (past timestamps)
    influx.writeBatch(msg.points);
    // Broadcast the update to the coach dashboard
    broadcast('coach', { type: 'replay', horse: msg.horse_id, points: msg.points });
    // ACK → phone can mark synced = 1
    ws.send(JSON.stringify({ type: 'ack', last_id: msg.last_id }));
  }
});
```

### On the coach dashboard (PWA)

**Offline horse behaviour:**
- Map marker → 🟡 icon + tooltip "Out of coverage for Xm Xs"
- Last known HR + speed displayed greyed out
- GPS track interrupted with dashed line
- When data arrives (flush) → track completed retroactively + "sync" animation

**Global indicator:**
```
[ 🟢 Aramis — LIVE ]  [ 🟡 Kahina — Offline 4m32s ]  [ 🟢 Elvire — LIVE ]
```

---

## WebSocket message format

### Live point (phone → backend)
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

### Batch flush (phone → backend, after offline)
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

### Instruction (backend → phone)
```json
{
  "type": "instruction",
  "code": "slow_down",
  "message": "Slow down — HR too high",
  "vibration": [300, 200, 300],
  "ts": 1741478450000
}
```

---

## Gait detection algorithm (ESP32)

Executed **on the halter ESP32** (IMU ICM-42688-P). The horse's head movement
is a cleaner and more reliable signal than the saddle pommel — this is the same
principle used by professional systems (Equimetre, Lameness Locator).

```
Buffer: 150 accelerometer samples @ 50 Hz = 3-second window

1. Isolate Z axis (vertical = horse bounce)
   — IMU is fixed on the halter: no orientation correction needed
2. Apply Hanning window (aliasing reduction)
3. FFT → power spectrum (trivial for ESP32-S3 on 150 samples)
4. Dominant frequency f0 = argmax(PSD[1..10 Hz])
5. Lateral energy ratio R = E(X axis) / E(Z axis)

Classification:
  f0 < 1.5 Hz              → "pas"   (walk)
  1.5 ≤ f0 < 2.5 Hz        → "trot"
  f0 ≥ 2.5 Hz  AND R > 0.6 → "amble" (pace)
  f0 ≥ 2.0 Hz  AND R ≤ 0.6 → "galop" (canter)

Confidence = FFT peak amplitude / total spectrum energy

Result transmitted via BLE to phone every 3 seconds:
  { gait: "trot", gait_conf: 0.91 }
```

**Key advantage:** the phone stays in the rider's pocket.
No mounting constraint on the saddle pommel.

---

## Vibration codes

| Instruction | Pattern | Meaning |
|-------------|---------|---------|
| Slow down | 300ms ON, 200ms OFF, 300ms ON | ⬇ Reduce speed |
| Speed up | 100ms×3 with 100ms OFF | ⬆ Increase speed |
| Change gait | 200ms×3 with 100ms OFF | → Change gait |
| Vet check | 500ms continuous | ⚑ Next vet check |

---

## Coach PWA — Features

- **Progressive Web App**: installable on iPad (Add to Home Screen), works offline for browsing
- **Authentication**: simple shared token (no user accounts)
- **Views**:
  - Live map (Mapbox GL)
  - HR × speed timeline per horse (Recharts)
  - Log of sent instructions
- **Post-race**:
  - Full GPS replay (live data + reconstructed buffered data)
  - CSV / JSON export
  - Curves: HR × speed, effort zones, V200, recovery HR at vet checks

---

## Stack & Deployment

```
app/          React Native (Expo) — iOS + Android
backend/      Node.js + ws + InfluxDB (existing VPS)
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
npm run build  # → dist/ to be served via nginx
```
