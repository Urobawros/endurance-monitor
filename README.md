# EnduranceMonitor 🐴

Real-time monitoring system for endurance horses — remote coaching, instructions to riders, post-race analytics.

## Concept

Monitor 1 to 3 horses entered in an endurance race with:
- **Heart rate** in real time (Polar Equine Heart Rate Monitor for Riding via BLE)
- **Gait + lameness** detected via ESP32-S3 + IMU on the halter (FFT — walk / trot / pace / canter + AAEP symmetry score 0-5)
- **GPS position** + **speed** (phone native GPS)
- **Coach → rider instructions** (vibration + notification)
- **Course map**: GPX/KML/GeoJSON import + satellite overlay, deviation detection
- **Offline-first**: local SQLite buffering, automatic flush when network returns
- **Post-race analytics** (HR×speed curves, V200, effort zones, GPS replay)

### Core design principle: phone in the pocket

The rider's smartphone is a **simple relay** — no need to mount it on the saddle pommel.
All motion analysis (gait + lameness) is performed **on the halter's ESP32**.
The phone only: collects BLE data, records GPS, buffers in SQLite, syncs over 4G, and relays coach vibrations.

## Architecture

```
Polar Equine ─BLE─► Rider smartphone ──SQLite──► Sync Manager ──4G (when available)──► Backend WebSocket
                    (in pocket)        (offline buffer)                                 • InfluxDB
ESP32-S3 ──BLE──►  • HR (Polar)                                                       • PWA Coach
 (halter)           • Gait + Lameness (ESP32 IMU)    ◄── Push instructions
 ICM-42688-P        • GPS → speed                    ◄── Rider vibration
 LiPo               • Buffer + Sync
```

**Key principle:** all data is written to local SQLite first, then transmitted to the backend as soon as the network is available. The coach dashboard shows the live state or last known position (with offline indicator).

**Key design:** the phone stays in the rider's pocket. The ESP32 on the halter handles all motion processing (FFT gait + lameness symmetry) and transmits results via BLE. No mounting constraints.

## Hardware per horse

| Component | Role | Price |
|-----------|------|-------|
| **Polar Equine HR Monitor for Riding** | HR BLE — strap designed for horse thorax | ~200-300€ |
| **ESP32-S3 mini + ICM-42688-P + LiPo 200mAh** | Gait (FFT) + lameness (symmetry) — mounted on halter | ~30€ |
| **Rider smartphone** (existing) | Relay: GPS + BLE + SQLite buffer + 4G sync | — |
| **3D enclosure** (printed on K2 Pro) + velcro/clip | Protection + halter mounting | ~2€ |

**Total cost per horse: ~230-330€** — **3 horses: ~700-1000€**

## Stack

| Layer | Technology |
|-------|-----------|
| ESP32 Firmware | ESP-IDF / Arduino — FFT gait + lameness symmetry + BLE broadcast |
| Rider app | React Native (Expo) — iOS + Android |
| BLE | react-native-ble-plx (Polar Equine HRM Profile + ESP32 custom service) |
| GPS | expo-location |
| Offline buffer | expo-sqlite |
| Transport | WebSocket (ws) |
| Backend | Node.js + ws + InfluxDB |
| Coach dashboard | **PWA React** (Mapbox GL + Recharts) — iPad Safari |

## Offline-first (Store & Forward)

In areas without network coverage:
1. App continues collecting HR + GPS + gait
2. Data stored in local SQLite (~720 KB/h/horse)
3. Coach dashboard shows 🟡 "Offline — last update Xm Xs ago"

When network returns:
1. Sync Manager flushes buffered data in batch
2. Backend inserts with original timestamps
3. Dashboard reconstructs the missing GPS track (replay)
4. Indicator switches back to 🟢 LIVE

## Coach → rider instructions

| Vibration code | Meaning |
|----------------|---------|
| 2× long | Slow down |
| 3× short | Speed up |
| 3× with pause | Change gait |
| 1× long continuous | Vet check |

Instructions arrive via push notification + vibration — the phone can stay in the pocket.
The rider takes out the phone to read the detail if needed.

## Gait detection (FFT on ESP32)

Computed **on the halter ESP32** (IMU ICM-42688-P) — the horse's head movement is a more reliable signal than the saddle pommel.

3s sliding window @ 50Hz on the IMU accelerometer:

| Gait | Dominant frequency | Signature |
|------|-------------------|-----------|
| Walk (pas) | 0.8–1.5 Hz | 4-beat |
| Trot (trot) | 1.5–2.5 Hz | Strong vertical bounce |
| Pace (amble) | 2.5–4 Hz | High lateral energy ratio |
| Canter (galop) | 2–3.5 Hz | Asymmetric |

The ESP32 transmits the result (gait + confidence) via BLE to the phone every 3 seconds. No computation on the app side.

## Roadmap

### MVP (3 weeks)
- [ ] ESP32 firmware: IMU read @ 200Hz + FFT gait @ 50Hz + BLE broadcast (gait + raw accel)
- [ ] App: BLE Polar Equine + BLE ESP32 + GPS + SQLite buffer + WebSocket sync
- [ ] Backend: WebSocket ingest (live + batch) + InfluxDB
- [ ] Coach PWA: live satellite map + HR/speed/gait + online/offline status + instruction buttons
- [ ] GPX/KML course import + map overlay + deviation detection (alert > 150m)

### Phase 2 (+2 weeks)
- [ ] Lameness detection on ESP32 (symmetry, AAEP grade, same IMU)
- [ ] Georeferenced scanned image (organisation PDF fallback)
- [ ] Post-race analytics: GPS replay, HR×speed curves, V200
- [ ] CSV / JSON export

### Phase 3 (future)
- [ ] ML gait classifier (trained on real field data)
- [ ] Elimination risk prediction
- [ ] Biothermo ND chip integration (percutaneous temperature)
- [ ] FreeStyle Libre integration (interstitial glucose)

## References

- Rallet N. (2023). *Utilisation d'objets connectés pour le suivi en course des chevaux d'endurance.* Thèse vétérinaire, EnvA/UPEC. [dumas-04303626](https://dumas.ccsd.cnrs.fr/dumas-04303626v1)
- Robert C. (2023). Capteur FreeStyle Libre 2 chez le cheval. *Pratique Vétérinaire Equine*, n°216.
- Arioneo Equimetre 2.0 — [training.arioneo.com](https://training.arioneo.com)

## Context

Project initiated by Dr Antoine Julienne (plastic surgeon, Clinique George V, Bordeaux)  
Development: Urobawros 🐍
