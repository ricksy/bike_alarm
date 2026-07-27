# Measurement Log

Empirical results only. If a number in this file is not backed by an instrument
reading, it does not belong here — put estimates in the design doc and mark them as
estimates.

## Method

Record the setup once and reference it, so readings are comparable across dates.

- **Instrument:** _(e.g. Nordic PPK2, Joulescope, µCurrent + DMM)_
- **Wiring:** _(where in the circuit, high side / low side, what is bypassed)_
- **Battery / supply:** _(cell, capacity, supply voltage during test)_
- **Firmware:** _(commit hash, build env, config diffs from stock)_
- **Environment:** _(temperature, radio traffic present, GNSS sky view)_

Averaging matters more than peaks for standby figures. Note the integration window.

## Targets

Derived from the months-scale standby goal. Fill in once the battery capacity is
fixed; these are the numbers the design has to hit, not measurements.

| Quantity | Target | Basis |
|---|---|---|
| Standby (armed, idle) | **≤75 µA** | match the reference device (`reference-device.md` §3) |
| Alarm active | _TBD_ | siren current, duty-limited |
| GNSS fix | _TBD_ | acquisition time × draw |
| LoRa TX burst | _TBD_ | SX1262 at configured power |
| Solar contribution | _TBD_ | panel × realistic insolation |
| **Reference device (KS-SF32R)** | **75 µA** | **manufacturer spec; 700 mAh cell, 6–10 months. The benchmark** |

## Readings

Append newest at the bottom. Never edit a past reading — add a correction entry.

### Template

```
#### YYYY-MM-DD — <what was measured>

- Firmware: <commit> / <env>
- Configuration: <relevant settings>
- Result: <value> (<averaging window>)
- Notes: <anomalies, what changed vs. the previous reading>
```

---

#### _(no readings yet)_

**First entry should be the reference device**, not the WisBlock — see
`reference-device.md` §6. Profiling the commercial King-Serry KS-SF32R gives a
field-proven target number and practice with the rig on known-good hardware.

Second entry: the bare WisBlock baseline, stock Meshtastic `env:rak4631`, deep sleep,
unmodified. Everything after is measured against that.
