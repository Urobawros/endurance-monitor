# Feature: Lameness Detection

## Clinical context

Lameness is the **#1 cause of elimination** in equestrian endurance racing (>50% of cases).
It can develop gradually over the course of a race, well before becoming visible
to the naked eye during vet checks.

Detecting emerging asymmetry 20-40 km before it becomes clinical allows:
- Immediately adapting gait and speed
- Anticipating the upcoming vet check
- Preventing escalation to serious injury (tendon, bone)

---

## What is an IMU?

**IMU = Inertial Measurement Unit**

An electronic chip measuring motion in 3 dimensions:

| Sensor | Axes | Measures |
|--------|------|---------|
| Accelerometer | X, Y, Z | Linear accelerations (bounce, impact, vibration) |
| Gyroscope | X, Y, Z | Rotation rates (tilt, pitch, roll) |
| Magnetometer* | X, Y, Z | Orientation / compass |

**6 axes** = accelero + gyro (sufficient for lameness)
**9 axes** = + magneto (absolute orientation)

> The commercial Lameness Locator (~5000€) uses exactly this principle:
> an IMU mounted on the horse's head, motion symmetry algorithm.
> Replicable with an ESP32 + IMU for ~30€.

---

## Detection principle

Lameness manifests as a **motion asymmetry**:

**Forelimbs:**
The horse raises its head when the painful limb contacts the ground (Gamb sign).
→ Vertical peaks alternate asymmetrically between left and right stride.

**Hindlimbs:**
The pelvis/croup drops less on the painful side.
→ Asymmetry of the vertical signal at croup level.

---

## Hardware architecture

The head sensor is the **same enclosure** used for gait detection.
A single ESP32 + IMU on the halter does both: gait (FFT) + lameness (symmetry).

```
┌─────────────────────────────────┐
│  Head sensor (halter / noseband)│
│  ESP32-S3 mini                  │  → gait (FFT @ 50Hz)
│  ICM-42688-P (6-axis IMU)       │  → forelimb lameness (symmetry @ 200Hz)
│  LiPo 200mAh (~6h battery life) │
│  BLE → rider smartphone         │
│  (in pocket)                    │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  Croup sensor (saddle pad)      │
│  ESP32-S3 mini                  │  → hindlimb detection (optional, Phase 3)
│  ICM-42688-P                    │
│  BLE → rider smartphone         │
└─────────────────────────────────┘

Enclosure: 3D printed (Creality K2 Pro), velcro + clip mounting
```

### Why ICM-42688-P instead of MPU-6050?

| Spec | MPU-6050 | ICM-42688-P |
|------|----------|-------------|
| Accelero noise | 400 μg/√Hz | **70 μg/√Hz** |
| Max frequency | 1 kHz | **32 kHz** |
| Price | ~3€ | ~8€ |

For lameness, vertical signal resolution is critical (asymmetries of a few mm).
The MPU-6050 generates too much noise to reliably detect grades 1-2.

### Cost per horse

| Component | Price |
|-----------|-------|
| ESP32-S3 mini | ~15€ |
| ICM-42688-P | ~8€ |
| LiPo 200mAh | ~5€ |
| 3D enclosure | ~2€ |
| **Total** | **~30€** |

---

## Algorithm

```
Acquisition: vertical accelerometer (Z axis) @ 200 Hz

Window: 10-second sliding (2000 samples)

Step 1 — Filtering
  - Low-pass filter 15 Hz (removes ground/terrain vibrations)
  - Mean subtraction (removes gravity)

Step 2 — Stance phase detection
  - Peak detection on filtered signal
  - Left / right alternation via stride counting

Step 3 — Asymmetry calculation (Equinosis method)
  MinDiff = mean(min_left) - mean(min_right)    → stance asymmetry
  MaxDiff = mean(max_right) - mean(max_left)    → lift asymmetry

Step 4 — AAEP grade score (0-5)
  |MinDiff| or |MaxDiff| (mm) → grade:
    < 3 mm  → 0 (symmetric)
    3-6 mm  → 1 (very mild)
    6-12 mm → 2 (mild, visible at trot)
    12-20mm → 3 (moderate)
    > 20 mm → 4-5 (severe)

Step 5 — Trend (last 10 minutes)
  Linear regression of score → positive slope = worsening
```

### Algorithmic limitations

- Grade 1: difficult without individual calibration (noise > signal)
- Very rough terrain: parasitic vibrations → adaptive filter required
- Canter: normal asymmetry → disable alert during canter phase

---

## System integration

### Data transmitted via BLE (ESP32 → phone)

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

`limb`: FL_left / FL_right (forelimb) | HL_left / HL_right (hindlimb)
`trend`: stable / increasing / decreasing

### Coach dashboard display

```
Aramis   🟢 LIVE
❤️ 142 bpm   ⚡ 18.4 km/h   🐴 Trot
📍 On route (+18m)
🦵 Symmetry 96%  ✅

──────────────────────────────

Kahina   🟢 LIVE
❤️ 136 bpm   ⚡ 17.2 km/h   🐴 Trot
📍 On route (+5m)
🦵 ⚠️  Left fore — Grade 2/5, trend ↗
```

### Alert levels

| Grade | Colour | Suggested action |
|-------|--------|-----------------|
| 0-1 | 🟢 green | All clear |
| 2 | 🟡 orange | Instruction "Drop to walk, observe" |
| 3 | 🔴 red | Rider vibration + "Stop at next vet check" |
| 4-5 | 🔴🔴 critical | Vibration + "Immediate stop" |

---

## Perspective: clinical validation

This system produces a **continuous symmetry score** throughout the entire race, whereas the
vet check only evaluates a single point in time every 20-40 km.

Data usable for a study:
- Correlation IMU score ↔ vet check result (AAEP grade)
- Anticipation time: how many km before elimination was the score already rising?
- Optimal decision threshold (sensitivity/specificity)

Potential subject for a veterinary thesis or equine publication (EnvA / Pratique Vétérinaire Equine).

---

## References

- Equinosis Lameness Locator — [equinosis.com](https://equinosis.com)
- Paulussen et al. (2017). *Inertial sensor-based lameness detection.* Equine Vet J.
- Bragança et al. (2017). *Head-mounted IMU for objective lameness assessment.* Vet J.
- Rallet N. (2023). EnvA thesis — connected devices for endurance horses.
