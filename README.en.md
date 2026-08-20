[🇫🇷 Français](./README.md) | 🇬🇧 English

# GoPool Variable Speed Pool Pump — Home Assistant Integration (localtuya)

Complete documentation to integrate a [GoPiscine](https://gopiscine.ca) variable speed pool pump into Home Assistant via [localtuya (xZetsubou fork)](https://github.com/xZetsubou/hass-localtuya), with **100% local** control (no dependency on the Tuya Cloud API at runtime).

**Compatibility**: tested and confirmed on the **AG1 1.5HP** model (above-ground pool). The **IG1** and **IG2** models (in-ground pool) likely share the same Tuya control board and DP schema (`model ID: e1n8gbg4` on the Tuya IoT platform) — not directly tested, but the configuration below should apply as-is or with minor adjustments. If you confirm it works on IG1/IG2, a contribution/issue is welcome.

<img width="300" height="300" alt="image" src="https://github.com/user-attachments/assets/ae561930-a652-43d3-a7fe-b870a3eb37f9" />
<img width="300" height="300" alt="image" src="https://github.com/user-attachments/assets/f1e0ae00-3fef-482a-9c2b-e50630bfaa17" />
<img width="300" height="300" alt="image" src="https://github.com/user-attachments/assets/b1513770-c577-4a91-85bb-5612ea23c061" />


## Table of Contents

- [Prerequisites](#prerequisites)
- [Getting Device ID, Local Key and IP](#getting-device-id-local-key-and-ip)
- [Adding the device manually in localtuya](#adding-the-device-manually-in-localtuya)
- [Importing entities via template](#importing-entities-via-template)
- [Complete Data Points (DP) table](#complete-data-points-dp-table)
- [Decoding the Fault bitmap](#decoding-the-fault-bitmap)
- [RPM → Power (W) curve](#rpm--power-w-curve)
- [Power and Energy sensors](#power-and-energy-sensors)
- [Replacement sensors (non-functional DPs)](#replacement-sensors-non-functional-dps)
- [Known limitations](#known-limitations)

## Prerequisites

- Home Assistant with [hass-localtuya (xZetsubou fork)](https://github.com/xZetsubou/hass-localtuya) installed via HACS
- Local protocol **3.5** (tested and working — 3.3 and 3.4 do not work).
- The pump must be connected to WiFi, on the same subnet as Home Assistant (e.g. if Home Assistant is on a 192.168.1.x IP, the pump must also be on a 192.168.1.x IP) and paired at least once via the Tuya Smart / Smart Life app

## Getting Device ID, Local Key and IP

This method does not depend on an active Cloud API subscription in Home Assistant — only a one-time visit to the Tuya developer portal is needed:

1. Go to [iot.tuya.com](https://iot.tuya.com) and sign in with the account linked to your Tuya Smart / Smart Life app.
2. Create (or use) a **Cloud Development Project** (Cloud → Development → Create Cloud Project).
3. Under the **Devices** tab, link your app account ("Link Tuya App Account") if not already done — this exposes all devices paired to your account to this project.
4. Find the pump in the list, click it to see its details: the **Device ID** and **Local Key** are shown directly there. **WARNING**: the Local Key changes value every time you remove the pump and re-add it in the Tuya / Smart Life app. It is therefore recommended to never do this, to avoid a Local Key change.
5. For the **local IP address**: check your router's DHCP client table. Set a static IP / DHCP reservation for this device to prevent localtuya from losing connection after an IP change.

## Adding the device **manually** in localtuya

1. Home Assistant → **Settings → Devices & Services → Add Integration → localtuya**
2. Choose **Add new device manually**
3. Fill in:
   - **Device ID**: (obtained above)
   - **Local Key**: (obtained above)
   - **Host / IP**: pump's local IP
   - **Protocol Version**: **3.5**
   - **Device Name**: your choice (e.g. "Pool Pump")
4. Confirm — localtuya should connect successfully.

## Importing entities via template

localtuya (xZetsubou fork) supports bulk-importing entities via a "template" file instead of adding them one by one:

1. Place the [`localtuya_template.yaml`](./localtuya_template.yaml) file (provided in this repo) in `custom_components/localtuya/templates/` on your HA installation.
2. **Fully restart Home Assistant** (a simple integration "Reload" is not enough — template files are only loaded at startup).
3. When adding a **new** device (see previous section), at the "Pick Entity type" step, a template import form appears automatically, listing the files available in the templates folder.
4. Select the template — all entities listed in `localtuya_template.yaml` are created at once.
5. Click "accept" for all entities without making changes.

> The template only includes DPs confirmed to work locally (see table below).

## Complete Data Points (DP) table

| DP ID | Code | Name | Type | Possible values | Works locally? | Notes |
|---|---|---|---|---|---|---|
| 1 | switch | Power | bool | true / false | ✅ Yes | Main power switch |
| 2 | fault | Fault | bitmap (ro) | 0-15 (see decoding) | ❌ No | Never updated locally, frozen since pairing |
| 101 | product_id | Product ID | string (ro) | — | ❌ No | Always empty on this device |
| 102 | schedule_switch | Schedule | bool | true / false | ✅ Yes | Enables the programmed schedule mode |
| 103 | cur_speed | Current Speed | value | 1150–3450, step 50 (rpm) | ✅ Yes | **Actually controls the speed** despite its "current" name |
| 104 | current_time | Current Time | value (rw) | 0–9999, special hex encoding (HHMM) | ❌ No | Complex format, never used/updated |
| 105 | time_sync | Time Sync | enum (rw) | `["read_time"]` | ❌ No | Command DP, never updated locally |
| 106 | noload_protection | No Load Protection | bool | true / false | ✅ Yes | Dry-run protection |
| 107 | set_speed | Set Speed | value | 1150–3450, step 50 (rpm) | ❌ No | Never written by the firmware — **DP 103** is what actually controls speed in practice |
| 108 | motor_operation_state | Motor Running | bool (rw) | true / false | ❌ No | Frozen at `false` since pairing, never reflects the motor's actual state |
| 124 | overall_status | Overall Status | enum (ro, bit-flags) | Start_Stop / Self_suction / completed / quick_clean / Time_out / Alarm / mode | ❌ No | Never updated locally |
| 125 | soft_setup | Soft Setup | enum (rw) | `["read_setup"]` | ❌ No | Command DP, never updated |
| 137–140 | stage1-4_switch | Stage 1–4 Enabled | bool | true / false | ❌ No | Never transmitted locally, even after explicit toggle via HA |
| 141/143/145/147 | start_timeX_hour | Stage 1–4 Start Hour | value | 0–23, step 1 | ✅ Yes | |
| 142/144/146/148 | start_timeX_min | Stage 1–4 Start Minute | value | 0–50, step 10 | ✅ Yes | Only 10-min increments (0,10,20,30,40,50) |
| 149/152/155/158 | speed_1-4 | Stage 1–4 Speed | value | 1000–3450, step 50 (rpm) | ✅ Yes | |
| 150/153/156/159 | start_time1-4 | (raw, duplicate) | value (rw) | 0–9999, hex encoding | ❌ Not used | Always 0, redundant with separate hour/min fields |
| 151/154/157/160 | duration_1-4 | Stage 1–4 Duration | value | 0–24, step 1 (hours) | ✅ Yes | Confirmed to be in hours via the manufacturer's default schedule |
| 173 | schedule_status | Schedule Status | enum (ro, bit-flags) | stageN_operation / stageN_completed | ❌ No | Confirmed frozen even after a genuine observed stage transition (cur_speed changed, schedule_status didn't) |
| 188 | timeout_duration | Timeout Duration | value | 1–600, step 1 (minutes) | ✅ Yes | |
| 189 | quickclean_switch | Quick Clean | bool | true / false | ✅ Yes | |
| 190 | quickclean_speed | Quick Clean Speed | value | 1000–3450, step 10 (rpm) | ✅ Yes | |
| 191 | quickclean_duration | Quick Clean Duration | value | 10–600, step 10 (min) | ✅ Yes | |

**Identified cause of non-functional DPs**: this firmware only updates its local cache for DPs that have been actively written by a client (app, HA, cloud) — read-only DPs (statuses, fault) are never transmitted locally, even when a genuine internal state change occurs (empirically confirmed: `schedule_status` stayed frozen on "stage1_operation" while `cur_speed` had genuinely changed to follow stages 2 and 3 of the schedule). This is not a localtuya bug — independently confirmed via a direct query with the `tinytuya` library, which shows the exact same behavior while completely bypassing the integration.

## Decoding the Fault bitmap

According to the official product spec (Tuya IoT Platform, `abilityId: 2`), the `fault` DP is a 4-bit bitmap (`maxlen: 4`):

| Bit | Value | Official label |
|---|---|---|
| 0 | 1 | high_temp (overheating) |
| 1 | 2 | flow_low (low flow) |
| 2 | 4 | rotating_fault (rotation fault) |
| 3 | 8 | pump_blocked (pump blocked) |

Multiple simultaneous faults add up (e.g. 6 = bits 1+2 active). Since this DP is read-only and not transmitted locally (see table above), this decoding is only usable via the Tuya Cloud API.

## RPM → Power (W) curve

File [`pool_pump_rpm_power_table.csv`](./pool_pump_rpm_power_table.csv) — 50 points from 1150 to 3450 RPM (step 50), obtained by linear interpolation between the following real measurements:

| RPM | Power (W) |
|---|---|
| 1150 (min) | 50 |
| 1500 | 83 |
| 2000 | 160 |
| 2450 | 271 |
| 2850 | 374 |
| 3450 (max) | 637 |


## Power and Energy sensors

File [`pool_pump_power_energy.yaml`](./pool_pump_power_energy.yaml):

- **`sensor.pool_pump_power`** (W): template sensor that interpolates the RPM→Watts table above in real time from `number.pompe_piscine_rpm`, and returns 0 when `switch.pompe_piscine` is off.
- **Energy (kWh)**: via HA's native "Integration - Riemann sum" helper (Settings → Helpers → + Add Helper), source = `sensor.pool_pump_power`, method Trapezoidal, metric prefix k, time unit Hours, **Max sub-interval = 5 minutes** (important: forces a periodic update even when power stays stable for several hours — otherwise the integral underestimates consumption during long constant-speed stages).

## Replacement sensors (non-functional DPs)

File [`pool_pump_status_estimates.yaml`](./pool_pump_status_estimates.yaml) — since `schedule_status` (173) and `motor_operation_state` (108) are never updated locally (see DP table), these two sensors recompute the equivalent directly in HA from the DPs that do work:

- **Active stage (estimated)**: compares the current time against the 4 configured Start Hour/Minute + Duration ranges to determine which schedule stage should currently be active.
- **Motor Running (estimated)**: `true` when `switch.pompe_piscine` is on AND `number.pompe_piscine_rpm` > 0.

There is no reliable equivalent for `fault`, `overall_status`, and the `stage1-4_switch` — these DPs remain unavailable locally with no workaround.
