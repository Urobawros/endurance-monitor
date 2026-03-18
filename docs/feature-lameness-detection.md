# Feature : Détection de boiterie (Lameness Detection)

## Contexte clinique

Les boiteries représentent la **1ère cause d'élimination** en endurance équestre (>50% des cas).
Elles peuvent se développer progressivement au cours de la course, bien avant d'être visibles
à l'œil nu lors des contrôles vétérinaires.

Détecter une asymétrie émergente 20-40 km avant qu'elle ne devienne clinique permet :
- D'adapter l'allure et la vitesse immédiatement
- D'anticiper le passage au contrôle vétérinaire
- D'éviter l'aggravation en lésion grave (tendon, os)

---

## Qu'est-ce qu'un IMU ?

**IMU = Inertial Measurement Unit** (centrale inertielle)

Puce électronique mesurant le mouvement dans les 3 dimensions :

| Capteur | Axes | Mesure |
|---------|------|--------|
| Accéléromètre | X, Y, Z | Accélérations linéaires (rebond, choc, vibration) |
| Gyroscope | X, Y, Z | Vitesses de rotation (inclinaison, tangage, roulis) |
| Magnétomètre* | X, Y, Z | Orientation / boussole |

**6 axes** = accéléro + gyro (suffisant pour boiterie)
**9 axes** = + magnéto (orientation absolue)

> Le Lameness Locator commercial (~5000€) utilise exactement ce principe :
> un IMU fixé sur la tête du cheval, algorithme de symétrie de mouvement.
> Recréable avec un ESP32 + IMU pour ~30€.

---

## Principe de détection

La boiterie se traduit par une **asymétrie de mouvement** :

**Membres antérieurs :**
Le cheval lève la tête quand le membre douloureux touche le sol (signe de Gamb).
→ Les pics verticaux alternent de façon asymétrique entre foulée gauche et droite.

**Membres postérieurs :**
Le bassin/croupe descend moins du côté douloureux.
→ Asymétrie du signal vertical au niveau de la croupe.

---

## Architecture hardware

Le capteur tête est le **même boîtier** que celui utilisé pour la détection d'allure.
Un seul ESP32 + IMU sur le licol fait les deux : allure (FFT) + boiterie (symétrie).

```
┌─────────────────────────────────┐
│  Capteur tête (licol / noseband)│
│  ESP32-S3 mini                  │  → allure (FFT @ 50Hz)
│  ICM-42688-P (IMU 6 axes)       │  → boiterie antérieure (symétrie @ 200Hz)
│  LiPo 200mAh (~6h autonomie)    │
│  BLE → smartphone cavalière     │
│  (dans la poche)                │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  Capteur croupe (tapis de selle)│
│  ESP32-S3 mini                  │  → détection postérieure (optionnel, Phase 3)
│  ICM-42688-P                    │
│  BLE → smartphone cavalière     │
└─────────────────────────────────┘

Boîtier : impression 3D (Creality K2 Pro), fixation velcro + clip
```

### Pourquoi ICM-42688-P plutôt que MPU-6050 ?

| Spec | MPU-6050 | ICM-42688-P |
|------|----------|-------------|
| Bruit accéléro | 400 μg/√Hz | **70 μg/√Hz** |
| Fréquence max | 1 kHz | **32 kHz** |
| Prix | ~3€ | ~8€ |

Pour la boiterie, la résolution du signal vertical est critique (asymétries de quelques mm).
Le MPU-6050 génère trop de bruit pour détecter les grades 1-2 fiablement.

### Coût par cheval

| Composant | Prix |
|-----------|------|
| ESP32-S3 mini | ~15€ |
| ICM-42688-P | ~8€ |
| LiPo 200mAh | ~5€ |
| Boîtier 3D | ~2€ |
| **Total** | **~30€** |

---

## Algorithme

```
Acquisition : accéléromètre vertical (axe Z) @ 200 Hz

Fenêtre : 10 secondes glissantes (2000 échantillons)

Étape 1 — Filtrage
  - Filtre passe-bas 15 Hz (supprime vibrations sol/terrain)
  - Soustraction moyenne (supprime gravité)

Étape 2 — Détection des phases d'appui (stance phases)
  - Peak detection sur signal filtré
  - Alternance gauche / droite via comptage de foulées

Étape 3 — Calcul asymétrie (méthode Equinosis)
  MinDiff = mean(min_gauche) - mean(min_droite)   → asymétrie appui
  MaxDiff = mean(max_droite) - mean(max_gauche)   → asymétrie soulèvement

Étape 4 — Score grade AAEP (0-5)
  |MinDiff| ou |MaxDiff| (mm) → grade :
    < 3 mm  → 0 (symétrique)
    3-6 mm  → 1 (très légère)
    6-12 mm → 2 (légère, visible au trot)
    12-20mm → 3 (modérée)
    > 20 mm → 4-5 (sévère)

Étape 5 — Tendance (10 dernières minutes)
  Régression linéaire du score → pente positive = aggravation
```

### Limites algorithmiques

- Grade 1 : difficile sans calibration individuelle (bruit > signal)
- Terrain très accidenté : vibrations parasites → filtre adaptatif nécessaire
- Galop : asymétrie normale → désactiver alerte en phase galop

---

## Intégration dans le système

### Données transmises par BLE (ESP32 → téléphone)

```json
{
  "type": "lameness",
  "horse_id": "aramis",
  "ts": 1741478400000,
  "symmetry_pct": 94,
  "min_diff_mm": 4.2,
  "max_diff_mm": 3.8,
  "grade": 1,
  "limb": "FL_left",
  "trend": "stable"
}
```

`limb` : FL_left / FL_right (forelimb) | HL_left / HL_right (hindlimb)
`trend` : stable / increasing / decreasing

### Affichage dashboard coach

```
Aramis   🟢 LIVE
❤️ 142 bpm   ⚡ 18.4 km/h   🐴 Trot
📍 Sur le tracé (+18m)
🦵 Symétrie 96%  ✅

──────────────────────────────

Kahina   🟢 LIVE
❤️ 136 bpm   ⚡ 17.2 km/h   🐴 Trot
📍 Sur le tracé (+5m)
🦵 ⚠️  Ant. gauche — Grade 2/5, tendance ↗
```

### Niveaux d'alerte

| Grade | Couleur | Action suggérée |
|-------|---------|----------------|
| 0-1 | 🟢 vert | RAS |
| 2 | 🟡 orange | Instruction "Passe au pas, observe" |
| 3 | 🔴 rouge | Vibration cavalière + "Stop au prochain contrôle" |
| 4-5 | 🔴🔴 critique | Vibration + "Arrêt immédiat" |

---

## Perspective : validation clinique

Ce système produit un **score de symétrie continu** sur toute la course, alors que le contrôle
vétérinaire n'évalue qu'un instant ponctuel tous les 20-40 km.

Données exploitables pour une étude :
- Corrélation score IMU ↔ résultat contrôle vétérinaire (grade AAEP)
- Temps d'anticipation : combien de km avant l'élimination le score augmentait déjà ?
- Seuil optimal de décision (sensibilité/spécificité)

Sujet de thèse vétérinaire ou publication équine potentielle (EnvA / Pratique Vétérinaire Equine).

---

## Références

- Equinosis Lameness Locator — [equinosis.com](https://equinosis.com)
- Paulussen et al. (2017). *Inertial sensor-based lameness detection.* Equine Vet J.
- Bragança et al. (2017). *Head-mounted IMU for objective lameness assessment.* Vet J.
- Rallet N. (2023). Thèse EnvA — objets connectés cheval d'endurance.
