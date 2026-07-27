# Bike Alarm — agent instructions

A DIY bicycle anti-theft device: nRF52840 / RAK WisBlock, LoRa via Meshtastic (EU868),
accelerometer trigger, piezo siren, GNSS, solar-assisted Li-ion.

Full design context, hardware inventory and codebase findings:

@docs/bike-alarm-project.md
@docs/firmware-submodule.md

Measurement log:

@docs/measurements.md

## Non-negotiables

- **Standby current is the tiebreaker.** The target is months of standby. Any change
  that costs standby current needs an explicit justification in the PR or commit message.
- **Interrupt-driven, never polled**, for anything in the always-on path. Upstream
  Meshtastic polls the accelerometer at 20 Hz; that is the specific thing this project
  exists to fix. Do not copy that pattern.
- **Measure, don't estimate.** Do not assert a current draw or a battery life figure
  without a meter reading logged in `docs/measurements.md`. "Should be about X µA" is
  not a result.
- **Respect the [UNVERIFIED] markers** in the design doc. Those are assumptions, not
  facts. Do not write pin-level or board-specific code against them — ask first.
  Promote an item to [VERIFIED] only after it has actually been checked, and record
  how it was checked.
- **Prefer upstreamable changes** to a divergent fork where the cost is similar.

## Conventions

- Firmware work happens against Meshtastic. Read `.github/copilot-instructions.md` in
  the firmware tree before any non-trivial change — it is the canonical agent doc there.
- Build: `pio run -e rak4631`
- Test: `./bin/run-tests.sh` (0 GREEN · 1 RED · 2 AMBER · 3 FILTERED)
- Format: `trunk fmt` before committing.
- Meshtastic is GPL-3.0. Keep that in mind for anything vendored or derived.
