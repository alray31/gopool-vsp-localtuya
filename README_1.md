# GoPool Variable Speed Pool Pump — Intégration Home Assistant (localtuya)

Documentation complète pour intégrer une pompe de piscine à vitesse variable **GoPool** dans Home Assistant via [localtuya (fork xZetsubou)](https://github.com/xZetsubou/hass-localtuya), en contrôle **100% local** (sans dépendre du Cloud API Tuya au runtime).

**Compatibilité** : testé et confirmé sur le modèle **AG1 1.5HP** (hors-terre). Les modèles **IG1** et **IG2** (installation terre) partagent vraisemblablement la même carte de contrôle Tuya et le même schéma de DP (`model ID: e1n8gbg4` sur la plateforme Tuya IoT) — non testés directement, mais la configuration ci-dessous devrait s'appliquer telle quelle ou avec des ajustements mineurs. Si tu confirmes le fonctionnement sur IG1/IG2, une contribution/issue est bienvenue.

## Sommaire

- [Prérequis](#prérequis)
- [Obtenir Device ID, Local Key et IP](#obtenir-device-id-local-key-et-ip)
- [Ajouter l'appareil manuellement dans localtuya](#ajouter-lappareil-manuellement-dans-localtuya)
- [Importer les entités via template](#importer-les-entités-via-template)
- [Tableau complet des Data Points (DP)](#tableau-complet-des-data-points-dp)
- [Décodage du bitmap Fault](#décodage-du-bitmap-fault)
- [Courbe RPM → Puissance (W)](#courbe-rpm--puissance-w)
- [Capteurs Puissance et Énergie](#capteurs-puissance-et-énergie)
- [Capteurs de remplacement (DP non fonctionnels)](#capteurs-de-remplacement-dp-non-fonctionnels)
- [Limitations connues](#limitations-connues)

## Prérequis

- Home Assistant avec [hass-localtuya (fork xZetsubou)](https://github.com/xZetsubou/hass-localtuya) installé via HACS
- Protocole local **3.5** (testé et fonctionnel — 3.3 et 3.4 n'apportent aucune différence pour les DP problématiques, voir [Limitations connues](#limitations-connues))
- La pompe doit être appairée au moins une fois via l'app Tuya Smart / Smart Life

## Obtenir Device ID, Local Key et IP

Cette méthode ne dépend pas d'un abonnement Cloud API actif dans Home Assistant — seule une visite ponctuelle sur le portail développeur Tuya est nécessaire :

1. Va sur [iot.tuya.com](https://iot.tuya.com) et connecte-toi avec le compte lié à ton app Tuya Smart / Smart Life.
2. Crée (ou utilise) un **Cloud Development Project** (Cloud → Development → Create Cloud Project).
3. Sous l'onglet **Devices**, lie ton compte d'application ("Link Tuya App Account") si ce n'est pas déjà fait — ça expose tous tes appareils appairés à ce projet.
4. Trouve ta pompe dans la liste, clique dessus pour voir ses détails : le **Device ID** et la **Local Key** y sont affichés directement.
5. Pour l'**adresse IP locale** : consulte la table des clients DHCP de ton routeur, ou utilise un scan réseau (`arp -a`, `nmap -sn 192.168.1.0/24`), ou vérifie dans l'app Tuya Smart si l'IP locale est visible dans les infos réseau de l'appareil. Fixe une IP statique/réservation DHCP pour cet appareil pour éviter que localtuya perde la connexion après un changement d'IP.

## Ajouter l'appareil manuellement dans localtuya

1. Home Assistant → **Paramètres → Appareils et services → Ajouter une intégration → localtuya**
2. Choisis **Add new device manually**
3. Remplis :
   - **Device ID** : (obtenu ci-dessus)
   - **Local Key** : (obtenu ci-dessus)
   - **Host / IP** : IP locale de la pompe
   - **Protocol Version** : **3.5**
   - **Device Name** : au choix (ex: "Pompe piscine")
4. Valide — localtuya devrait se connecter avec succès.

## Importer les entités via template

localtuya (fork xZetsubou) supporte l'import en bloc d'entités via un fichier "template" plutôt que de les ajouter une par une :

1. Place le fichier [`localtuya_template.yaml`](./localtuya_template.yaml) (fourni dans ce repo) dans `custom_components/localtuya/templates/` sur ton installation HA.
2. **Redémarre complètement Home Assistant** (un simple "Reload" de l'intégration ne suffit pas — les fichiers de templates ne sont chargés qu'au démarrage).
3. Lors de l'ajout d'un **nouveau** device (voir section précédente), à l'étape "Pick Entity type", un formulaire d'import de template apparaît automatiquement, listant les fichiers disponibles dans le dossier templates.
4. Sélectionne le template et soumets — toutes les entités listées dans `localtuya_template.yaml` sont créées d'un coup.

> Le template n'inclut que les DP confirmés fonctionnels en local (voir tableau ci-dessous). Alternativement, tu peux aussi générer ton propre fichier template en exportant la configuration d'un device déjà configuré : **Configure → Edit Device → Export**.

## Tableau complet des Data Points (DP)

| DP ID | Code | Nom | Type | Valeurs possibles | Fonctionnel en local? | Notes |
|---|---|---|---|---|---|---|
| 1 | switch | Power | bool | true / false | ✅ Oui | Interrupteur principal |
| 2 | fault | Fault | bitmap (ro) | 0-15 (voir décodage) | ❌ Non | Jamais mis à jour localement, figé depuis l'appairage |
| 101 | product_id | Product ID | string (ro) | — | ❌ Non | Toujours vide sur ce device |
| 102 | schedule_switch | Schedule | bool | true / false | ✅ Oui | Active le mode horaire programmé |
| 103 | cur_speed | Current Speed | value | 1150–3450, pas 50 (rpm) | ✅ Oui | **Contrôle réellement la vitesse** malgré son nom "current" |
| 104 | current_time | Current Time | value (rw) | 0–9999, encodage hex spécial (HHMM) | ❌ Non | Format complexe, jamais utilisé/mis à jour |
| 105 | time_sync | Time Sync | enum (rw) | `["read_time"]` | ❌ Non | DP commande, jamais mis à jour localement |
| 106 | noload_protection | No Load Protection | bool | true / false | ✅ Oui | Protection marche à vide |
| 107 | set_speed | Set Speed | value | 1150–3450, pas 50 (rpm) | ❌ Non | Jamais écrit par le firmware — c'est **DP 103** qui contrôle la vitesse en pratique |
| 108 | motor_operation_state | Motor Running | bool (rw) | true / false | ❌ Non | Figé à `false` depuis l'appairage, ne reflète jamais l'état réel du moteur |
| 124 | overall_status | Overall Status | enum (ro, bit-flags) | Start_Stop / Self_suction / completed / quick_clean / Time_out / Alarm / mode | ❌ Non | Jamais mis à jour localement |
| 125 | soft_setup | Soft Setup | enum (rw) | `["read_setup"]` | ❌ Non | DP commande, jamais mis à jour |
| 137–140 | stage1-4_switch | Stage 1–4 Enabled | bool | true / false | ❌ Non | Jamais transmis localement, même après toggle explicite via HA |
| 141/143/145/147 | start_timeX_hour | Stage 1–4 Start Hour | value | 0–23, pas 1 | ✅ Oui | |
| 142/144/146/148 | start_timeX_min | Stage 1–4 Start Minute | value | 0–50, pas 10 | ✅ Oui | Incréments de 10 min seulement (0,10,20,30,40,50) |
| 149/152/155/158 | speed_1-4 | Stage 1–4 Speed | value | 1000–3450, pas 50 (rpm) | ✅ Oui | |
| 150/153/156/159 | start_time1-4 | (brut, dupliqué) | value (rw) | 0–9999, encodage hex | ❌ Non utilisé | Toujours à 0, redondant avec hour/min séparés |
| 151/154/157/160 | duration_1-4 | Stage 1–4 Duration | value | 0–24, pas 1 (heures) | ✅ Oui | Confirmé en heures via l'horaire par défaut du fabricant |
| 173 | schedule_status | Schedule Status | enum (ro, bit-flags) | stageN_operation / stageN_completed | ❌ Non | Confirmé figé même après une vraie transition d'étape observée (cur_speed a changé, schedule_status non) |
| 188 | timeout_duration | Timeout Duration | value | 1–600, pas 1 (minutes) | ✅ Oui | |
| 189 | quickclean_switch | Quick Clean | bool | true / false | ✅ Oui | |
| 190 | quickclean_speed | Quick Clean Speed | value | 1000–3450, pas 10 (rpm) | ✅ Oui | |
| 191 | quickclean_duration | Quick Clean Duration | value | 10–600, pas 10 (min) | ✅ Oui | |

**Cause identifiée des DP non fonctionnels** : ce firmware ne met à jour son cache local que pour les DP activement écrits par un client (app, HA, cloud) — les DP purement en lecture (statuts, fault) ne sont jamais transmis localement, même quand un changement d'état réel survient en interne (confirmé empiriquement : `schedule_status` est resté figé sur "stage1_operation" alors que `cur_speed` avait bien changé pour suivre l'étape 3 du programme). Ce n'est pas un bug de localtuya — confirmé indépendamment via une requête directe avec la librairie `tinytuya`, qui montre exactement le même comportement en contournant complètement l'intégration.

## Décodage du bitmap Fault

Selon la spec officielle du produit (Tuya IoT Platform, `abilityId: 2`), le DP `fault` est un bitmap à 4 bits (`maxlen: 4`) :

| Bit | Valeur | Label officiel |
|---|---|---|
| 0 | 1 | high_temp (surchauffe) |
| 1 | 2 | flow_low (débit faible) |
| 2 | 4 | rotating_fault (défaut de rotation) |
| 3 | 8 | pump_blocked (pompe bloquée) |

Plusieurs défauts actifs simultanément s'additionnent (ex: 6 = bits 1+2 actifs). Ce DP étant en lecture seule et non transmis localement (voir tableau ci-dessus), ce décodage n'est utilisable qu'en passant par le Cloud API Tuya.

## Courbe RPM → Puissance (W)

Fichier [`pool_pump_rpm_power_table.csv`](./pool_pump_rpm_power_table.csv) — 50 points de 1000 à 3450 RPM (pas de 50), obtenus par interpolation linéaire entre les mesures réelles suivantes :

| RPM | Puissance (W) |
|---|---|
| 1150 (min calibré) | 50 |
| 1500 | 83 |
| 2000 | 160 |
| 2450 | 271 |
| 2850 | 374 |
| 3450 (max) | 637 |

> Les valeurs entre 1000 et 1150 RPM sont **extrapolées** (pas mesurées) en prolongeant la pente du premier segment — moins fiables que le reste de la table.

## Capteurs Puissance et Énergie

Fichier [`pool_pump_power_energy.yaml`](./pool_pump_power_energy.yaml) :

- **`sensor.pool_pump_power`** (W) : template sensor qui interpole la table RPM→Watts ci-dessus en temps réel à partir de `number.pompe_piscine_rpm`, et retourne 0 quand `switch.pompe_piscine` est éteint.
- **Énergie (kWh)** : via le helper natif HA "Intégrale - somme de Riemann" (Paramètres → Aides → + Ajouter une aide), source = `sensor.pool_pump_power`, méthode Trapézoïdale, préfixe k, unité de temps Heures, **Sous-intervalle maximum = 5 minutes** (important : force une mise à jour périodique même quand la puissance reste stable pendant plusieurs heures, sinon l'intégrale sous-estime la conso pendant les longs paliers de vitesse constante).

## Capteurs de remplacement (DP non fonctionnels)

Fichier [`pool_pump_status_estimates.yaml`](./pool_pump_status_estimates.yaml) — puisque `schedule_status` (173) et `motor_operation_state` (108) ne sont jamais mis à jour localement (voir tableau des DP), ces deux capteurs recalculent l'équivalent directement dans HA à partir des DP qui fonctionnent :

- **Étape active (estimée)** : compare l'heure actuelle aux 4 plages Start Hour/Minute + Duration configurées pour déterminer quelle étape du programme devrait être active.
- **Motor Running (estimé)** : `true` quand `switch.pompe_piscine` est allumé ET `number.pompe_piscine_rpm` > 0.

Il n'existe pas d'équivalent fiable pour `fault`, `overall_status` et les `stage1-4_switch` — ces DP restent indisponibles en local sans solution de contournement.

## Limitations connues

- Testé sur les protocoles **3.3**, **3.4** et **3.5** — aucune différence sur les DP non fonctionnels, confirmant que la cause est le firmware de la pompe, pas la version de protocole.
- Redémarrer/rallumer la pompe ne réinitialise pas la programmation (persistée en mémoire non-volatile), mais ne résout pas non plus le problème des DP figés.
- Le manuel papier du fabricant liste des codes de défaut (1, 2, 4, 6, 8, 16, 128, 256, 300, 301) qui ne correspondent pas à la spec officielle Tuya (max 4 bits, valeurs 1-8) — probablement un système de codes différent affiché sur l'écran physique de la pompe, sans lien avec ce DP Tuya spécifique.
- Pour obtenir `fault`, `overall_status`, `schedule_status` et les autres DP figés de façon fiable, il faudrait passer par le Cloud API Tuya (nécessite un abonnement Cloud IoT Core actif — attention à ne pas laisser expirer la période d'essai du Cloud Development Project, source du problème initial ayant motivé le passage à localtuya).
