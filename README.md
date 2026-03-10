# EnduranceMonitor 🐴

Système de monitoring temps réel pour chevaux d'endurance — coaching à distance, instructions aux cavalières, analytics post-course.

## Concept

Monitorer 1 à 3 chevaux engagés en course d'endurance avec :
- **Fréquence cardiaque** en temps réel (Polar H10 via BLE)
- **Allure** détectée via accéléromètre du téléphone (FFT — pas / trot / amble / galop)
- **Position GPS** + **vitesse** (GPS natif téléphone)
- **Instructions coach → cavalière** (vibration + affichage plein écran)
- **Carte parcours** : import GPX/KML/GeoJSON + superposition satellite, détection de déviation
- **Détection de boiterie** : ESP32 + IMU sur le licol, score de symétrie en continu (grade AAEP 0-5)
- **Offline-first** : buffering SQLite local, flush automatique quand réseau revient
- **Analytics post-course** (courbes FC×vitesse, V200, zones d'effort, replay GPS)

## Architecture

```
Polar H10 ──BLE──► App React Native ──SQLite──► Sync Manager ──4G (quand dispo)──► Backend WebSocket
                    • FC                (buffer offline)                              • InfluxDB
                    • GPS → vitesse                                                   • PWA Coach
                    • Accéléro → allure (FFT)          ◄── Instructions push
                    • Vibration + affichage 3s
```

**Principe clé :** toutes les données sont écrites en SQLite local d'abord, puis transmises au backend dès que le réseau est disponible. Le dashboard coach affiche l'état live ou la dernière position connue (avec indicateur offline).

## Hardware par cheval

| Composant | Rôle | Prix |
|-----------|------|------|
| **Polar H10** | Fréquence cardiaque BLE | ~90€ |
| **Smartphone cavalière** (existant) | Hub : GPS + accéléro + BLE + 4G | — |
| **ESP32-S3 + ICM-42688-P** *(optionnel)* | Détection boiterie (IMU sur licol) | ~30€ |

**Coût total 3 chevaux : ~270€** (sans boiterie) / **~360€** (avec boiterie)

## Stack

| Couche | Technologie |
|--------|-------------|
| App cavalière | React Native (Expo) — iOS + Android |
| BLE | react-native-ble-plx (Polar HRM Profile) |
| GPS | expo-location |
| Accéléro/Gyro | expo-sensors (50 Hz) |
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

\+ Affichage plein écran 3 secondes : "⬇ Ralentis — FC trop haute"

## Détection d'allure (FFT)

Fenêtre glissante 3s @ 50Hz sur accéléromètre du téléphone (fixé guidon/pommeau) :

| Allure | Fréquence dominante | Signature |
|--------|-------------------|-----------|
| Pas | 0.8–1.5 Hz | 4 temps |
| Trot | 1.5–2.5 Hz | Rebond vertical fort |
| Amble | 2.5–4 Hz | Ratio énergie latérale élevé |
| Galop | 2–3.5 Hz | Asymétrique |

## Roadmap

### MVP (3 semaines)
- [ ] App : BLE Polar H10 + GPS + SQLite buffer + WebSocket sync
- [ ] Backend : WebSocket ingest (live + batch) + InfluxDB
- [ ] PWA coach : carte satellite live + FC/vitesse + statut online/offline + boutons instructions
- [ ] Import GPX/KML parcours + overlay carte + détection déviation (alerte > 150m)

### Phase 2 (+2 semaines)
- [ ] Détection allure (FFT on-device)
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
