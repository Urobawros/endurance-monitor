# References & Literature

## Theses and scientific publications

- Rallet N. (2023). *Utilisation d'objets connectés pour le suivi en course des chevaux d'endurance : étude exploratoire.* Doctoral thesis in veterinary medicine, EnvA / Faculté de Médecine de Créteil (UPEC). [dumas-04303626](https://dumas.ccsd.cnrs.fr/dumas-04303626v1) — PDF in `../endurance/`
- Robert C. (2023). FreeStyle Libre 2 sensor placement protocol in horses. *Pratique Vétérinaire Equine*, n°216.

## Endurance physiological parameters (reference values)

| Parameter | Reference value | Note |
|-----------|----------------|------|
| Resting HR | 28–44 bpm | |
| Moderate effort HR | 80–120 bpm | |
| High effort HR | 140–200 bpm | |
| V200 (speed at HR=200) | ~6–8 m/s | fitness indicator |
| Recovery HR (vet check) | < 64 bpm | homologation condition |
| Race hypoglycaemia | after 80–100 km | Rallet 2023 data |

## Reference equipment

- **Polar Equine Heart Rate Monitor for Riding**: HR sensor designed for horses, chest strap adapted to equine morphology, standard BLE (HRM Profile), compatible with custom apps. [polar.com/fr/horse-heart-rate-sensors](https://www.polar.com/fr/horse-heart-rate-sensors)
- **Polar H10** (human): chest transmitter, BLE + ANT+, 1000 Hz ECG, waterproof IP68 — usable on horses with extended strap but not ideal
- **Biothermo ND** (MSD Animal Health): identification chip with percutaneous thermal sensor
- **FreeStyle Libre 2** (Abbott): interstitial CGM glucose sensor, neck placement (Robert 2023)
- **Arioneo Equimetre 2.0**: GPS + HR + gait, 4G, coach dashboard (Nov. 2024)
- **WAOOK** (Equi-Test): HR + GPS multi-horse, iPad coach, used by EnvA

## Relevant open-source projects

- [openmha](https://github.com/HörTech/openMHA) — audio/sensor signal processing
- [heartpy](https://github.com/paulvangentcom/heartrate_analysis_python) — Python HR analysis
