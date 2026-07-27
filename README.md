# bike_alarm

A DIY electronic anti-theft device for a bicycle, built on the RAK WisBlock platform
and Meshtastic.

Loud piezo siren, accelerometer-triggered, with remote alerting and position reporting
over LoRa (EU868). Solar-assisted Li-ion power. The design constraint that drives
everything else is **months-scale standby battery life**, comparable to a commercial
bike alarm.

## Status

Design and component-selection phase. No firmware written yet.

| | |
|---|---|
| MCU / radio | nRF52840 + SX1262 (RAK4631 WisBlock Core) |
| Motion | LIS3DH — interrupt-driven wake (planned) |
| GNSS | RAK12500 (u-blox ZOE-M8Q) |
| Siren | 12 V self-contained piezo, MOSFET-switched via boost converter |
| Radio | Meshtastic, EU868 |
| Target BOM | ~€60–80 |

## Repository layout

```
docs/
  bike-alarm-project.md   design context, hardware inventory, codebase findings
  measurements.md         current-draw and battery measurements
CLAUDE.md                 agent instructions (imports the above)
```

## Documentation

Start with [`docs/bike-alarm-project.md`](docs/bike-alarm-project.md). It carries the
hardware inventory, the architectural decisions and their rationale, findings from
reading the Meshtastic source, and the open work list.

Items marked **[UNVERIFIED]** in that document are assumptions that have not been
confirmed. Treat them accordingly.

## Notable finding

Upstream Meshtastic's motion subsystem polls: `LIS3DHSensor::runOnce()` reads the
accelerometer over I2C every 50 ms, and `DetectionSensorModule` polls its GPIO every
100 ms. The sensor's own interrupt pins are unused. That design exists to wake a
screen, not to be a low-power motion trigger, and it is incompatible with months of
standby. Replacing it with hardware-interrupt wake is the core firmware work of this
project.

## Licence

Not yet chosen. Note that Meshtastic is GPL-3.0, which constrains anything derived
from it.
