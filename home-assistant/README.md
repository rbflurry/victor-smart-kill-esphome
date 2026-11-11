# 🏠 Home Assistant automations & dashboard

Ready-to-paste automations and a dashboard for **one or many** Local-Kill traps:
battery reports, catch alerts, cleaning reminders, low-battery alarms and missed
check-in detection.

← Back to the [main README](../README.md)

| File | Automation | Fires when | Alert style |
|---|---|---|---|
| [`00-battery-on-wake-notification.yaml`](00-battery-on-wake-notification.yaml) | Battery report on wake | Every check-in | Normal; replaces that trap's previous report |
| [`01-mouse-caught.yaml`](01-mouse-caught.yaml) | Mouse caught | *Needs cleaning* goes off → on | High priority / time-sensitive |
| [`02-needs-cleaning-reminder.yaml`](02-needs-cleaning-reminder.yaml) | Still needs cleaning | Each check-in while *Needs cleaning* is on | Normal; replaces the catch alert |
| [`03-low-battery-warning.yaml`](03-low-battery-warning.yaml) | Low battery warning | Check-in with battery 4.80–5.00 V | Normal, repeats each check-in |
| [`04-critical-battery-alarm.yaml`](04-critical-battery-alarm.yaml) | Critical battery alarm | Check-in with battery < 4.80 V | **Alarm**: rings through silent / DND |
| [`05-missed-check-in.yaml`](05-missed-check-in.yaml) | Missed check-in | A trap silent for 26 h (then daily) | High priority |
| [`06-check-in-restored.yaml`](06-check-in-restored.yaml) | Check-in restored | A trap reports again | Clears its missed check-in alert |
| [`dashboard-local-kill.yaml`](dashboard-local-kill.yaml) | Dashboard | — | Overview of all traps + a tab per trap |

You can use any combination. **00** and **03/04** overlap (both report battery):
use **00** for a battery report every check-in, or **03 + 04** to hear about the
battery only when it's low.

## Setup

### 1. Name each trap (once per trap)

The firmware uses `name_add_mac_suffix`, so a new trap shows up as
*Local-Kill a1b2c3* with entity IDs like `sensor.local_kill_a1b2c3_battery_voltage`.
Give each one a real name:

**Settings → Devices & services → ESPHome → Local-Kill a1b2c3 → ✏️** → name it
e.g. **Kitchen trap** → when asked to **rename the entity IDs too, say yes**.

The entities become `sensor.kitchen_trap_battery_voltage` and so on, and that name
appears in every notification title. The examples here use `kitchen_trap` and
`garage_trap`.

### 2. Automations: one copy covers every trap

Each automation file is written once for all traps. You only edit:

- the **`▼ YOUR TRAPS ▼`** list in the trigger, one line per trap:
  ```yaml
  entity_id:                                   # ▼ YOUR TRAPS ▼
    - binary_sensor.kitchen_trap_status
    - binary_sensor.garage_trap_status
  ```
- `notify.mobile_app_YOUR_PHONE` → your notify service (Developer tools → Actions,
  search "notify").

Then **Settings → Automations & scenes → Create automation → ⋮ → Edit in YAML**,
paste, save.

**Adding a trap later:** name it (step 1), then add one line to the list in
00–04 and 06. **05 needs no list**: it finds every trap automatically.

How it works: when any listed trap triggers, the automation looks up that trap's
other sensors from its device (`device_entities(device_id(trigger.entity_id))`),
so the message, battery reading and notification tag are always for the right trap.
Each trap gets its own notification tags, so alerts from different traps don't
replace each other.

### 3. Dashboard

See [Dashboard](#dashboard) below.

## How the trap looks to Home Assistant

The trap spends almost all its time in deep sleep, which shapes how these automations work:

- **Values stay put while it sleeps.** Because the firmware has a `deep_sleep:` block, the
  ESPHome integration keeps the last battery, *Needs cleaning* and *Kill total* values
  instead of marking them unavailable.
- **Only *Status* changes.** It's **on** while the trap is connected and **off** while it
  sleeps. *Status* going off → on means "the trap just woke and reported fresh readings",
  which is why most automations trigger on it.
- **When it reports:**
  - **Scheduled:** about every 24 h. It wakes every 2 h, but only connects on every 12th wake.
  - **Catch, button press, battery insert:** reports immediately and restarts the 24 h cycle.
  - **Retry:** if Wi-Fi or HA is unreachable, it sleeps 10 min and tries again.
- **Triggering on battery voltage changes doesn't work.** The battery is sampled once per
  wake. If the reading matches last time, the sensor never changes and a state trigger
  would miss that wake.

## Automation details

### 00 – Battery report on wake
Sends `🔋 Kitchen trap woke (Scheduled)` with voltage and %. The title switches to 🪫
and adds a swap reminder below 5.0 V. It waits 5 s after connect so fresh values have arrived.

### 01 – Mouse caught
Triggers on *Needs cleaning* going **off → on**. It deliberately ignores `unknown → on`,
so an HA restart doesn't re-alert. The message includes the kill number and whether the
trap was **armed**. If it was disarmed, the HV didn't fire and the mouse may still be alive.

> There's no "Mark cleaned" action button on the notification. The trap goes back to sleep
> ~15 s after reporting, so the button would only work if *Walk-by red indicator* or
> *Stay awake* is keeping it awake.

### 02 – Still needs cleaning
On every check-in where *Needs cleaning* is still on, sends a reminder with how long ago the
catch happened. It skips the check-in that *was* the catch (wake reason `Catch`) so you don't
get two alerts at once. It clears when you press **Mark cleaned** or re-insert the batteries
with the plates clear.

### 03 – Low battery warning / 04 – Critical battery alarm
Both check the battery on every check-in:

| Voltage | Battery % (firmware scale) | Result |
|---|---|---|
| ≥ 5.00 V | ≥ ~22 % | nothing |
| 4.80 – 5.00 V | ~11 – 22 % | **03** warning |
| < 4.80 V | < ~11 % | **04** alarm |

The firmware treats 6.40 V as 100 % and 4.60 V as 0 %. The 5.0 / 4.8 V thresholds are a
starting point. Raise them if you find the HV stops charging properly earlier.

The **04** alarm uses both the Android and iOS settings (each app ignores the other's):
- **Android:** `channel: alarm_stream` plays through the alarm volume, even on silent.
- **iOS:** `critical: 1`. The Home Assistant app must be allowed to send **Critical Alerts**
  (iOS Settings → Notifications → Home Assistant).

### 05 – Missed check-in
Runs hourly and checks **every** Local-Kill trap: any ESPHome device with a *Kill total*
sensor. It alerts when a trap's *Status* has been off for **26 h** (the 24 h report interval
plus margin for retries), then reminds once a day while it stays silent. Likely causes are a
dead battery, Wi-Fi out of range, HA down for a long time, or a crashed trap. Press the
trap's button to force a wake.

> After an HA restart, `last_changed` resets to the restart time, so the alert is
> **delayed** (not lost) by however long HA was up before the restart.

### 06 – Check-in restored
Clears that trap's *missed check-in* notification the next time it reports. Optional,
but pairs with 05.

## Dashboard

[`dashboard-local-kill.yaml`](dashboard-local-kill.yaml) uses only built-in cards
(no HACS) and needs HA 2024.8 or newer.

**All traps tab** (finds every trap automatically, nothing to edit):
- Alert banners for traps that need emptying, have a low battery or missed a check-in,
  or a green "all OK".
- A table with one row per trap: status, battery % and V, kills, armed, last check-in.

**Per-trap tab** (one per trap):
- **Trap status:** a red *Mouse caught* banner, a check-in banner (*online now* /
  *asleep, last check-in X ago, next in ~N h* / *missed check-in*), and tiles for kills,
  armed, last wake reason, plates and connection.
- **Battery:** a gauge, the voltage, a 30-day voltage trend and a low/critical banner.
- **Controls:** arm (asks for confirmation), walk-by blink, stay awake, mark cleaned,
  sample battery, and HV test (asks for confirmation). A banner shows whether the trap is
  awake, because controls only reach it during a wake.
- **History:** 7 days of check-ins and catches, plus an activity log.

**Install:**
1. Settings → Dashboards → Add dashboard → New dashboard from scratch, open it,
   **✏️ → ⋮ → Raw configuration editor**, paste the file, save.
2. The file has one per-trap tab, for `kitchen_trap`. For each other trap, copy the whole
   view block (from `  - title: Kitchen trap` to the end of the file), paste it at the end,
   find/replace `kitchen_trap` with that trap's ID within the copy, and change its `title`
   and `path`.
