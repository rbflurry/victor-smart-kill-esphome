# 🏠 Home Assistant automations

Ready-to-paste automations for the Local-Kill trap: battery reports, catch alerts,
cleaning reminders, low-battery alarms and missed check-in detection.

← Back to the [main README](../README.md)

| File | Automation | Fires when | Alert style |
|---|---|---|---|
| [`00-battery-on-wake-notification.yaml`](00-battery-on-wake-notification.yaml) | Battery report on wake | Every check-in | Normal; replaces the previous report |
| [`01-mouse-caught.yaml`](01-mouse-caught.yaml) | Mouse caught | *Needs cleaning* goes off → on | High priority / time-sensitive |
| [`02-needs-cleaning-reminder.yaml`](02-needs-cleaning-reminder.yaml) | Still needs cleaning | Each check-in while *Needs cleaning* is on | Normal; replaces the catch alert |
| [`03-low-battery-warning.yaml`](03-low-battery-warning.yaml) | Low battery warning | Check-in with battery 4.80–5.00 V | Normal, repeats each check-in |
| [`04-critical-battery-alarm.yaml`](04-critical-battery-alarm.yaml) | Critical battery alarm | Check-in with battery < 4.80 V | **Alarm** – rings through silent / DND |
| [`05-missed-check-in.yaml`](05-missed-check-in.yaml) | Missed check-in | No check-in for 26 h | High priority |
| [`06-check-in-restored.yaml`](06-check-in-restored.yaml) | Check-in restored | Trap reports again | Clears the missed check-in alert |

You can use any combination. **00** and **03/04** overlap (both report battery):
use **00** if you want a battery report every check-in, or **03 + 04** if you only want to
hear about the battery when it's low.

## Setup

1. Find your trap's entity prefix. The firmware uses `name_add_mac_suffix`, so entities
   look like `binary_sensor.local_kill_a1b2c3_status`. Check **Settings → Devices &
   services → ESPHome → Local-Kill**.
2. In each file, find/replace:
   - `local_kill_XXXXXX` → your prefix (e.g. `local_kill_a1b2c3`)
   - `notify.mobile_app_YOUR_PHONE` → your notify service (Developer tools → Actions,
     search "notify")
3. **Settings → Automations & scenes → Create automation → ⋮ → Edit in YAML**, paste
   the file contents, save. Repeat per file.

`00-battery-on-wake-notification.yaml` is the exception: it finds the battery and
wake-reason sensors from the trap's own device, so you only need to set the *Status*
entity, and you can list several traps in its trigger.

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
Sends `🔋 Local-Kill woke (Scheduled)` with the voltage and %. The title switches to 🪫 and
adds a swap reminder below 5.0 V. It waits 5 s after connect so fresh values have arrived,
and uses a per-trap notification tag so each report replaces the last.

### 01 – Mouse caught
Triggers on *Needs cleaning* going **off → on**. It deliberately ignores `unknown → on`, so an
HA restart doesn't re-alert. The message includes the kill number and whether the trap was
**armed**. If it was disarmed, the HV didn't fire and the mouse may still be alive.

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
A template trigger that becomes true once *Status* has been **off for more than 26 h**: the
24 h report interval plus margin for retries. Likely causes are a dead battery, Wi-Fi out of
range, HA down for a long time, or a crashed trap. Press the trap's button to force a wake.

> After an HA restart, `last_changed` resets to the restart time, so the alert is
> **delayed** (not lost) by however long HA was up before the restart.

### 06 – Check-in restored
Clears the *missed check-in* notification from your phone the next time the trap reports.
Optional, but pairs with 05.

## Multiple traps

Each file is written for one trap. For more traps either:
- duplicate the automation per trap (simplest), or
- list all the traps' entities in the trigger, then use `trigger.entity_id` and
  `device_entities(device_id(trigger.entity_id))` to find the matching sensors, the way
  `00-battery-on-wake-notification.yaml` does.
