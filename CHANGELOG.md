# Changelog

All notable changes to **Local-Kill** are recorded here.

The format follows [Keep a Changelog](https://keepachangelog.com/), and the project
aims to follow [Semantic Versioning](https://semver.org/) once it reaches a tagged
release. Until then everything lives under **Unreleased** — v1.0.0 is gated on the
first confirmed kill on a real mouse in the field.

## [Unreleased]

### Added
- **Catch-listen window** (`catch_window_ms`, default 5000 ms): after every wake the
  firmware watches the plates in the dark for up to the window length, polling every
  ~20 ms, and fires the instant it sees a debounced bridge. Covers a mouse that is in
  the trap but not shorting the plates at the one moment a single one-shot sample would
  have checked. It breaks immediately on contact, so catch speed is unaffected by the
  window length — the window is only patience for a hesitant mouse.
- **Plates-driven catch detection**, independent of the ESP8266 reset reason. A bridged
  plate is treated as a catch even when a wake-from-sleep mislabels an external (catch)
  reset as "Deep-Sleep Wake" — the root cause of the earlier missed catches.
- **Post-shock kill confirmation + `needs_cleaning` lockout**: after the HV cycle the
  firmware re-checks the plates; a body still across them counts the kill and latches the
  trap so it will not re-fire the corpse on the next wake.
- **Chained deep-sleep heartbeat** (2 h × 12 ≈ 24 h) to work within the ESP8266's ~3.5 h
  per-sleep hardware limit, using an RTC-backed chunk counter.
- Diagnostic entities: **Wake reason**, **Plates bridged**, **HV brownout last cycle**.
- Battery reading via the GPIO0 divider-enable pulse, reported as voltage and whole-number %.
- Stock-firmware **backup & restore** instructions in the README (full-flash `esptool`
  dump to a file and write-back), so others can try Local-Kill with a safe path to stock.

### Changed
- **Fire in the dark.** No LED lights before or during the HV shock; the red LED only
  flashes afterward to confirm a body. A mouse bolts at the first light, so the fire path
  now stays silent until the decision is made.
- **HV settle 300 ms → 100 ms.** The longer settle was insurance against a dead-short
  bench brownout; a real mouse is a resistive load, so the shorter settle gives it less
  time to leave. The brownout detector stays in as a safety net.
- **HV is always live** — removed the separate "arm" switch that was fighting flash
  persistence across sleep.
- **Re-arm is now physical**: empty the trap, then press the trap's Wi-Fi button or
  power-cycle it with the plates clear. Removed the Home Assistant "Mark cleaned" button,
  which was unreachable on a mostly-sleeping trap. Re-arm will not clear while the plates
  are still bridged (body present).

### Fixed
- Boot / no-connect: set logger `baud_rate: 0` so GPIO1/3 stay free for the trap hardware.
- Board flash size corrected to `esp01` (512 KB, AT25SF041). No OTA room — flash via serial.
- Battery % now reports a whole number instead of a long float.

### Known / pending
- **First confirmed field kill achieved** (2026-11-15) on the build *before* the catch-listen
  window — the catch → HV → post-shock confirmation → report chain is proven in the field.
- The catch-listen window (`catch_window_ms`) has **not** been field-proven yet; v1.0.0 is now
  gated on a confirmed kill with that version running.
- Flash persistence of `needs_cleaning` across a real sleep cycle is still not independently
  field-confirmed; both the re-fire lockout and physical re-arm depend on it.
- Next planned change: remove the brownout-detector code and confirm there are no HV ramps.

---

*Log started 2026-11-15. Entries above cover notable work up to that point, grouped by
type rather than exact date; later entries will be added as changes are made.*
