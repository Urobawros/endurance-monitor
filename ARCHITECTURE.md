# Architecture détaillée

## Flux de données

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CHEVAL (×1 à 3)                                 │
│                                                                         │
│   Polar H10                                                             │
│   (émetteur thoracique)  ──── BLE HRM Profile ────►                    │
│                                                    │                    │
└────────────────────────────────────────────────────┼────────────────────┘
                                                     │
                                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    SMARTPHONE CAVALIÈRE                                  │
│                    (fixé sur guidon ou pommeau)                         │
│                                                                         │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │  App React Native                                                │  │
│   │                                                                  │  │
│   │  BLE Manager ──► FC (bpm)                                        │  │
│   │  GPS (50Hz)   ──► lat, lon, vitesse (km/h), cap                  │  │
│   │  Accéléro     ──► Buffer 150 pts ──► FFT ──► allure              │  │
│   │  Gyro         ──► correction orientation                         │  │
│   │                                                                  │  │
│   │  WebSocket Client ──────────────────────────────────────────►   │  │
│   │                                                                  │  │
│   │  ◄── Push notification (instruction) ──────────────────────────  │  │
│   │       → Vibration pattern                                        │  │
│   │       → Affichage plein écran 3s                                 │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────┬───────────────────────────┘
                                              │ 4G WebSocket
                                              │ ~1 message/seconde
                                              │ payload ~200 bytes
                                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         VPS BACKEND (Node.js)                           │
│                                                                         │
│   WebSocket Server (ws)                                                 │
│   ├── /horse/:id    ◄── stream cavalière                                │
│   └── /coach        ◄── dashboard coach (lecture + instructions)        │
│                                                                         │
│   InfluxDB (time series)                                                │
│   └── measurement: horse_telemetry                                      │
│       tags: horse_id, race_id                                           │
│       fields: fc, vitesse, allure, lat, lon                             │
│                                                                         │
│   REST API                                                              │
│   ├── GET /sessions              → liste des courses                    │
│   ├── GET /sessions/:id/data     → données complètes post-course        │
│   └── POST /instructions/:horse  → envoyer instruction                  │
│                                                                         │
└─────────────────────────────────────────────┬───────────────────────────┘
                                              │ WebSocket
                                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      DASHBOARD COACH (React Web / iPad)                  │
│                                                                         │
│   ┌───────────────────┐  ┌─────────────────┐  ┌─────────────────────┐  │
│   │  Carte Mapbox GL  │  │  Panel cheval 1 │  │  Panel cheval 2/3   │  │
│   │  • Positions live │  │  FC: 142 bpm ❤️  │  │  FC: 128 bpm ❤️     │  │
│   │  • Tracé GPS      │  │  ⚡ 18.4 km/h    │  │  ⚡ 16.2 km/h       │  │
│   │  • Zones couleur  │  │  🐴 Trot         │  │  🐴 Amble           │  │
│   │    selon FC       │  │  [⬆][⬇][↻][⚑]  │  │  [⬆][⬇][↻][⚑]     │  │
│   └───────────────────┘  └─────────────────┘  └─────────────────────┘  │
│                                                                         │
│   Boutons instructions :                                                │
│   [⬆ Accélérer]  [⬇ Ralentir]  [→ Changer allure]  [⚑ Contrôle]     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Format du message WebSocket (cavalière → backend)

```json
{
  "horse_id": "aramis",
  "race_id": "2026-03-15_mont-de-marsan",
  "ts": 1741478400000,
  "fc": 142,
  "lat": 43.8921,
  "lon": -0.5012,
  "speed_kmh": 18.4,
  "heading": 247,
  "gait": "trot",
  "gait_confidence": 0.91,
  "battery": 84
}
```

## Format instruction (backend → cavalière)

```json
{
  "type": "instruction",
  "code": "slow_down",
  "message": "Ralentis — FC trop haute",
  "vibration": [200, 100, 200],
  "ts": 1741478450000
}
```

## Algorithme de détection d'allure

```
1. Buffer circulaire accéléromètre (150 pts @ 50Hz = 3 secondes)
2. Correction orientation via quaternion gyroscope
3. Isolation axe vertical Z (rebond)
4. Application fenêtre Hanning
5. FFT → spectre de puissance
6. Fréquence dominante f0 = argmax(PSD)
7. Ratio énergie latérale R = E(X) / E(Z)

Classification :
  f0 < 1.5                 → "pas"
  1.5 ≤ f0 < 2.5           → "trot"
  f0 ≥ 2.5 AND R > 0.6    → "amble"
  f0 ≥ 2.0 AND R ≤ 0.6    → "galop"
```

## Codes vibration

```
Ralentir      : [300ms ON, 200ms OFF, 300ms ON]
Accélérer     : [100ms ON, 100ms OFF, 100ms ON, 100ms OFF, 100ms ON]
Changer allure: [200ms ON, 100ms OFF, 200ms ON, 100ms OFF, 200ms ON]
Contrôle vét. : [500ms ON]
```

## Déploiement

```bash
# Backend (VPS existant)
npm install
node server.js --port 3001

# App (dev)
cd app/
npx expo start

# Dashboard (dev)
cd dashboard/
npm run dev
```
