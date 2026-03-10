# Références & Littérature

## Thèses et publications scientifiques

- Rallet N. (2023). *Utilisation d'objets connectés pour le suivi en course des chevaux d'endurance : étude exploratoire.* Thèse de doctorat vétérinaire, EnvA / Faculté de Médecine de Créteil (UPEC). [dumas-04303626](https://dumas.ccsd.cnrs.fr/dumas-04303626v1) — PDF dans `../endurance/`
- Robert C. (2023). Protocole de pose capteur FreeStyle Libre 2 chez le cheval. *Pratique Vétérinaire Equine*, n°216.

## Paramètres physiologiques endurance (valeurs de référence)

| Paramètre | Valeur de référence | Note |
|-----------|--------------------|-|
| FC repos | 28–44 bpm | |
| FC effort modéré | 80–120 bpm | |
| FC effort intense | 140–200 bpm | |
| V200 (vitesse à FC=200) | ~6–8 m/s | indicateur forme |
| FC récupération (véto) | < 64 bpm | condition d'homologation |
| Hypoglycémie course | après 80–100 km | données Rallet 2023 |

## Matériel de référence

- **Polar H10** : émetteur thoracique, BLE + ANT+, 1000 Hz ECG, waterproof IP68
- **Biothermo ND** (MSD Animal Health) : puce d'identification avec capteur thermique percutané
- **FreeStyle Libre 2** (Abbott) : capteur CGM glucose interstitiel, pose au cou (Robert 2023)
- **Arioneo Equimetre 2.0** : GPS + FC + allure, 4G, dashboard coach (nov. 2024)
- **WAOOK** (Equi-Test) : FC + GPS multi-chevaux, iPad coach, utilisé par EnvA

## Projets open source pertinents

- [openmha](https://github.com/HörTech/openMHA) — traitement signal audio/capteurs
- [heartpy](https://github.com/paulvangentcom/heartrate_analysis_python) — analyse FC Python
