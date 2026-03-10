# EnduranceMonitor 🐴

Système de monitoring temps réel pour chevaux d'endurance — coaching à distance, instructions aux cavalières, analytics post-course.

## Concept

Monitorer 1 à 3 chevaux engagés en course d'endurance avec :
- **Fréquence cardiaque** en temps réel
- **Allure** détectée via accéléromètre (pas / trot / amble / galop)
- **Position GPS** + **vitesse**
- **Instructions coach → cavalière** (vibration + affichage)
- **Analytics post-course** (courbes FC×vitesse, V200, zones d'effort, replay)

## Architecture

```
Polar H10 ──BLE──► App Cavalière (smartphone) ──4G──► WebSocket VPS ──► Dashboard Coach (iPad)
                    • FC                                                   • Carte 3 chevaux live
                    • GPS → vitesse + position                             • FC / vitesse / allure
                    • Accéléro → allure (FFT)          ◄── Instructions    • Boutons instructions
                    • Vibration + affichage 3s
```

### Hardware par cheval

| Composant | Rôle | Prix |
|-----------|------|------|
| **Polar H10** | Fréquence cardiaque BLE | ~90€ |
| **Smartphone cavalière** | Hub : GPS + accéléro + 4G | existant |

**Coût total 3 chevaux : ~270€**

Le smartphone embarqué (fixé sur guidon/pommeau) gère :
- Connexion BLE → Polar H10 (FC)
- GPS natif → position + vitesse
- Accéléromètre + gyroscope → détection d'allure (FFT)
- Stream 4G → backend
- Réception instructions coach → vibration + affichage

### Stack technique

| Couche | Technologie |
|--------|-------------|
| App cavalière | React Native (iOS + Android) |
| BLE | react-native-ble-plx (Polar H10 HRM Profile) |
| GPS | expo-location |
| Accéléro/Gyro | expo-sensors (50 Hz) |
| Transport | WebSocket (ws) |
| Backend | Node.js + ws + InfluxDB |
| Dashboard coach | React Web (Mapbox GL + Recharts) |
| Hébergement | VPS existant |

## Détection d'allure

Basée sur FFT glissante (fenêtre 3s, 50 Hz) sur l'axe vertical du téléphone :

| Allure | Fréquence dominante | Signature |
|--------|-------------------|-----------|
| Pas | 0.8–1.5 Hz | 4 temps réguliers |
| Trot | 1.5–2.5 Hz | 2 temps, fort rebond vertical |
| Amble / paso | 2.5–4 Hz | Ratio énergie axe X/Z élevé |
| Galop | 2–3.5 Hz | Asymétrique, 3 temps |

Précision attendue (téléphone en position fixe guidon) : **85-92%**

## Instructions coach → cavalière

Codes vibration simples (no-look) :

| Code | Signification |
|------|--------------|
| 1× long | Accélérer |
| 2× court | Ralentir |
| 3× court | Changer d'allure |
| 1× long + 1× court | Ravitaillement / prochain contrôle |

+ Affichage plein écran 3 secondes avec message texte court.

## Analytics post-course

Données loguées par session :
```json
{
  "timestamp": "ISO8601",
  "cheval": "string",
  "lat": "float",
  "lon": "float",
  "vitesse_kmh": "float",
  "allure": "pas|trot|amble|galop",
  "fc_bpm": "int",
  "instruction_envoyee": "string|null"
}
```

Analyses disponibles :
- Courbe FC × vitesse → calcul V200 (seuil aérobie)
- Zones d'effort par boucle
- FC de récupération aux contrôles vétérinaires
- Allure dominante par portion de parcours
- Replay GPS avec overlay FC
- Comparaison inter-courses (progression saison)
- Corrélation instruction coach → réponse FC/vitesse

## Roadmap

### MVP (3 semaines)
- [ ] App React Native : BLE Polar H10 + GPS + stream WebSocket
- [ ] Backend Node.js : WebSocket multi-clients + logging JSON
- [ ] Dashboard coach : carte live + FC + vitesse + boutons instructions
- [ ] Instructions push notification + vibration

### Phase 2 (+2 semaines)
- [ ] Détection allure (FFT on-device)
- [ ] Analytics post-course (courbes, V200)
- [ ] Export CSV / replay GPS

### Phase 3 (futur)
- [ ] ML classifier allure (modèle entraîné sur données réelles)
- [ ] Prédiction risque d'élimination (FC + allure + portion du parcours)
- [ ] Intégration puce thermique Biothermo ND (température)
- [ ] Intégration FreeStyle Libre (glycémie interstitielle)

## Références

- Rallet N. (2023). *Utilisation d'objets connectés pour le suivi en course des chevaux d'endurance : étude exploratoire.* Thèse vétérinaire, EnvA / UPEC. [DUMAS](https://dumas.ccsd.cnrs.fr/dumas-04303626v1)
- Robert C. (2023). Protocole de pose capteur FreeStyle Libre 2 chez le cheval. *Pratique Vétérinaire Equine*, n°216.
- Arioneo Equimetre 2.0 — [training.arioneo.com](https://training.arioneo.com)
- Polar H10 Equine — [polar.com](https://polar.com)

## Contexte

Projet initié par Dr Antoine Julienne (chirurgien plasticien, Clinique George V, Bordeaux)  
Développement : Urobawros 🐍
