# EnduranceMonitor 🐴

Système de monitoring temps réel pour chevaux d'endurance — coaching à distance, instructions aux cavalières, analytics post-course.

## Concept

Monitorer 1 à 3 chevaux engagés en course d'endurance avec :
- **Fréquence cardiaque** en temps réel (Polar H10 via BLE)
- **Allure + boiterie** détectées via ESP32-S3 + IMU sur le licol (FFT — pas / trot / amble / galop + score symétrie AAEP 0-5)
- **Position GPS** + **vitesse** (GPS natif téléphone)
- **Instructions coach → cavalière** (vibration + notification)
- **Carte parcours** : import GPX/KML/GeoJSON + superposition satellite, détection de déviation
- **Offline-first** : buffering SQLite local, flush automatique quand réseau revient
- **Analytics post-course** (courbes FC×vitesse, V200, zones d'effort, replay GPS)

### Principe de design : téléphone dans la poche

Le smartphone de la cavalière est un **simple relais** — pas besoin de le fixer au pommeau.
Toute l'analyse de mouvement (allure + boiterie) est réalisée **sur l'ESP32 du licol**.
Le téléphone ne fait que : collecter les données BLE, enregistrer le GPS, buffer en SQLite, sync en 4G, et transmettre les vibrations du coach.

## Architecture

```
Polar H10 ──BLE──► Smartphone cavalière ──SQLite──► Sync Manager ──4G (quand dispo)──► Backend WebSocket
                    (dans la poche)       (buffer offline)                              • InfluxDB
ESP32-S3 ──BLE──►  • FC (Polar)                                                       • PWA Coach
 (licol)            • Allure + Boiterie (ESP32 IMU)    ◄── Instructions push
 ICM-42688-P        • GPS → vitesse                    ◄── Vibration cavalière
 LiPo               • Buffer + Sync
```

**Principe clé :** toutes les données sont écrites en SQLite local d'abord, puis transmises au backend dès que le réseau est disponible. Le dashboard coach affiche l'état live ou la dernière position connue (avec indicateur offline).

**Design clé :** le téléphone reste dans la poche de la cavalière. L'ESP32 sur le licol fait tout le traitement de mouvement (allure FFT + symétrie boiterie) et transmet les résultats par BLE. Pas de contrainte de fixation.

## Hardware par cheval

| Composant | Rôle | Prix |
|-----------|------|------|
| **Polar H10** + sangle adaptée | Fréquence cardiaque BLE | ~90€ |
| **ESP32-S3 mini + ICM-42688-P + LiPo 200mAh** | Allure (FFT) + boiterie (symétrie) — fixé sur le licol | ~30€ |
| **Smartphone cavalière** (existant) | Relais : GPS + BLE + buffer SQLite + sync 4G | — |
| **Boîtier 3D** (imprimé K2 Pro) + velcro/clip | Protection + fixation licol | ~2€ |

**Coût total par cheval : ~120€** — **3 chevaux : ~360€**

## Stack

| Couche | Technologie |
|--------|-------------|
| Firmware ESP32 | ESP-IDF / Arduino — FFT allure + symétrie boiterie + BLE broadcast |
| App cavalière | React Native (Expo) — iOS + Android |
| BLE | react-native-ble-plx (Polar HRM Profile + ESP32 custom service) |
| GPS | expo-location |
| Buffer offline | expo-sqlite |
| Transport | WebSocket (ws) |
| Backend | Node.js + ws + InfluxDB |
| Dashboard coach | **PWA React** (Mapbox GL + Recharts) — iPad Safari |

## Offline-first (Store & Forward)

En zone sans réseau :
1. App continue de collecter FC + GPS + allure
2. Données stockées en SQLite local (~720 KB/h/cheval)
3. Dashboard coach affiche 🟡 "Offline — dernière MAJ il y a Xm Xs"

Quand le réseau revient :
1. Sync Manager flush les données buffered en batch
2. Backend les insère avec leurs timestamps originaux
3. Dashboard reconstruit le tracé GPS manquant (replay)
4. Indicateur repasse 🟢 LIVE

## Instructions coach → cavalière

| Code vibration | Signification |
|----------------|--------------|
| 2× longs | Ralentir |
| 3× courts | Accélérer |
| 3× avec pause | Changer d'allure |
| 1× long continu | Contrôle vétérinaire |

Les instructions arrivent par push notification + vibration — le téléphone peut rester dans la poche.
La cavalière sort le téléphone pour lire le détail si besoin.

## Détection d'allure (FFT sur ESP32)

Calculée **sur l'ESP32 du licol** (IMU ICM-42688-P) — le mouvement de la tête du cheval est un signal plus fiable que le pommeau de selle.

Fenêtre glissante 3s @ 50Hz sur accéléromètre de l'IMU :

| Allure | Fréquence dominante | Signature |
|--------|-------------------|-----------|
| Pas | 0.8–1.5 Hz | 4 temps |
| Trot | 1.5–2.5 Hz | Rebond vertical fort |
| Amble | 2.5–4 Hz | Ratio énergie latérale élevé |
| Galop | 2–3.5 Hz | Asymétrique |

L'ESP32 transmet le résultat (allure + confiance) par BLE au téléphone. Pas de calcul côté app.

## Roadmap

### MVP (3 semaines)
- [ ] Firmware ESP32 : lecture IMU @ 200Hz + FFT allure @ 50Hz + broadcast BLE (allure + raw accel)
- [ ] App : BLE Polar H10 + BLE ESP32 + GPS + SQLite buffer + WebSocket sync
- [ ] Backend : WebSocket ingest (live + batch) + InfluxDB
- [ ] PWA coach : carte satellite live + FC/vitesse/allure + statut online/offline + boutons instructions
- [ ] Import GPX/KML parcours + overlay carte + détection déviation (alerte > 150m)

### Phase 2 (+2 semaines)
- [ ] Détection boiterie sur ESP32 (symétrie, grade AAEP, même IMU)
- [ ] Géoréférencement image scannée (fallback PDF organisation)
- [ ] Analytics post-course : replay GPS, courbes FC×vitesse, V200
- [ ] Export CSV / JSON

### Phase 3 (futur)
- [ ] ML classifier allure (entraîné sur données réelles terrain)
- [ ] Prédiction risque d'élimination
- [ ] Intégration puce Biothermo ND (température percutanée)
- [ ] Intégration FreeStyle Libre (glycémie interstitielle)

## Références

- Rallet N. (2023). *Utilisation d'objets connectés pour le suivi en course des chevaux d'endurance.* Thèse vétérinaire, EnvA/UPEC. [dumas-04303626](https://dumas.ccsd.cnrs.fr/dumas-04303626v1)
- Robert C. (2023). Capteur FreeStyle Libre 2 chez le cheval. *Pratique Vétérinaire Equine*, n°216.
- Arioneo Equimetre 2.0 — [training.arioneo.com](https://training.arioneo.com)

## Contexte

Projet initié par Dr Antoine Julienne (chirurgien plasticien, Clinique George V, Bordeaux)  
Développement : Urobawros 🐍
