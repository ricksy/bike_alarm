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
  bike-alarm-project.md    design context, hardware, codebase findings, roadmap
  prior-art.md             what already exists and what to take from it
  reference-device.md      the commercial alarm being benchmarked against
  siren-driver.md          piezo drive topologies, acoustics, driver ICs
  measurements.md          current-draw and battery measurements
  firmware-submodule.md    git submodule setup and daily workflow
CLAUDE.md                  agent instructions (imports the above)
firmware/                  submodule → fork of meshtastic/firmware
```

## Documentation

Start with [`docs/bike-alarm-project.md`](docs/bike-alarm-project.md). Items marked
**[UNVERIFIED]** there are assumptions that have not been confirmed — treat them
accordingly.

## Key findings so far

- **BLOCKED: no usable accelerometer attached.** The boot log finds exactly one I2C
  device, at 0x1d, which Meshtastic misidentifies as a "DFRobot Rain Gauge" — it's
  actually the unsupported RAK12032 (ADXL313), which shares that address. Nothing
  answers at 0x18/0x19, so there is no LIS3DH. A RAK1904 is needed before v1 can
  proceed — though the ADXL313 turns out to be well suited to this application (Analog
  Devices lists "car alarm" as its first example use case) and a driver is a tractable
  upstream contribution.
- **GPS verified working** — `U-blox 8 detected` confirms it's the RAK12500 (ZOE-M8Q),
  UART-attached, with GPS/GLONASS/Galileo/SBAS configured.
- **The stock motion path polls.** `LIS3DHSensor::runOnce()` reads the accelerometer
  over I2C every 50 ms and `DetectionSensorModule` polls its GPIO every 100 ms; the
  sensor's own interrupt pins are unused. Incompatible with months of standby, and the
  core firmware work of this project.
- **Waking from nRF52 System OFF resets the CPU.** The alarm state machine must be
  restartable from persisted state, not resident in RAM.
- **The board, not the MCU, sets the power floor.** See the Long Cricket teardown in
  the prior-art doc.
- **Siren loudness is acoustics as much as electronics.** Mounting geometry — hole
  diameter, standoff, a compliant ring — is where a homemade siren loses ~20 dB to a
  commercial one. See [`docs/siren-driver.md`](docs/siren-driver.md).
- **The incumbent proves the spec is reachable.** A €20 commercial alarm achieves
  ~110 dB, months of standby and an instant-feeling remote — by duty-cycling its
  receiver, not by listening continuously. Its documented arm / pre-alarm / disarm
  behaviour is a useful starting spec. See [`docs/reference-device.md`](docs/reference-device.md).
- **Alerting needs a second node.** A phone connects over BLE at a few metres. The boot
  log shows no mesh peers in range, so remote alerting requires a receiver node at home.
- **No finished open-source Meshtastic bike alarm exists.**

---

## References

Everything consulted while designing this project. Verified 2026-07-27.

### Meshtastic — firmware and docs

- [meshtastic/firmware](https://github.com/meshtastic/firmware) — the firmware source (default branch: `develop`)
- [RAK WisBlock base boards](https://meshtastic.org/docs/hardware/devices/rak-wireless/wisblock/base-board/) — Meshtastic's own notes on RAK5005-O vs RAK19007, including the LiPo and solar connector specs
- [Drag-and-drop nRF52 flashing](https://meshtastic.org/docs/getting-started/flashing-firmware/nrf52/drag-n-drop/) — the UF2 method, plus the RAK4631 vs RAK4631-R warning
- [Updating the nRF52 bootloader](https://meshtastic.org/docs/getting-started/flashing-firmware/nrf52/update-nrf52-bootloader/) — bootloader packages and checksums
- [Meshtastic Web Flasher](https://flasher.meshtastic.org) — Chrome/Edge only
- [Bicycle Tracking (Discourse, 2020)](https://meshtastic.discourse.group/t/bicycle-tracking/1238) — rider tracking on a charity relay, not anti-theft; included for completeness

### Meshtastic issues and PRs being tracked

- [#9750 — Claymore module](https://github.com/meshtastic/firmware/issues/9750) — open request for a motion/tripwire alarm module. Closest thing to upstream demand for this project
- [#10945 — Persist time across nRF52 System OFF](https://github.com/meshtastic/firmware/issues/10945) — source of the "wake resets the CPU" constraint
- [#9699 — nRF52 solar nodes: wake behaviour and brownout risks](https://github.com/meshtastic/firmware/issues/9699)
- [#11211 — nRF52 opt-in low-voltage boot protection](https://github.com/meshtastic/firmware/pull/11211)
- [#11025 — Stop accelerometer thread when wake-on-motion disabled](https://github.com/meshtastic/firmware/pull/11025) — nearest existing work to the interrupt change

### Hardware — RAKwireless

- [WisBlock product index](https://docs.rakwireless.com/product-categories/wisblock/) — canonical entry point; datasheets for the other modules follow the same URL pattern
- [RAK19007 base board datasheet](https://docs.rakwireless.com/product-categories/wisblock/rak19007/datasheet/)
- [RAK12500 GNSS datasheet](https://docs.rakwireless.com/product-categories/wisblock/rak12500/datasheet/) — u-blox ZOE-M8Q; slot A (UART/I2C) or slot C (I2C only)
- [RAK12032 accelerometer datasheet](https://docs.rakwireless.com/product-categories/wisblock/rak12032/datasheet/) — ADXL313; note INT1/INT2 pin assignment varies by slot
- [WisBlock Kit 3 (GPS Tracker)](https://store.rakwireless.com/products/wisblock-kit-3-gps-tracker) — documented contents are RAK19007 + RAK4631 + RAK1904 + RAK12500. Useful for cross-checking what Kit 2 actually contains

### Prior art

- [LoRider](https://www.hackster.io/Milliam/lorider-the-meshatastic-0cffeb) — Meshtastic bike anti-theft tracker. Same architecture, never implemented

### Low-power reference design

- [Long Cricket Asset Tracker (Tindie)](https://www.tindie.com/products/tleracorp/long-cricket-loralorawangnss-asset-tracker/) — the power-budget teardown: ~2.5 µA sleep and an itemised account of where every microamp went
- [Source code](https://github.com/kriswiner/CMWX1ZZABZ/tree/master/longCricketAssetTracker) — Arduino sketch; the BMA400 dual-interrupt wake/sleep pattern is the part worth studying
- [Hackable CMWX1ZZABZ LoRa devices](https://hackaday.io/project/35169-hackable-cmwx1zzabz-lora-devices) — Tlera project documentation
- [Asset Tracker (Hackaday.io)](https://hackaday.io/project/25790-asset-tracker) — background on the design problem
- [ArduinoCore-stm32l0](https://github.com/GrumpyOldPizza/ArduinoCore-stm32l0) — the core the Long Cricket sketch is built on

### Reference device — King-Serry KS-SF32R

The commercial bike alarm currently in use, and the benchmark for standby life and
loudness. See [`docs/reference-device.md`](docs/reference-device.md).

- [King-Serry catalogue](http://king-serry.com/index.php/en/1695454/product/welcome_doorbell/kssf20r.html) — the actual OEM behind the VioRose/NineLeaf/Mengshen rebrands
- [KS-SF32R user manual](https://manuals.plus/nineleaf/ks-sf32r-vehicle-anti-theft-alarm-pro-vibration-intrusion-detector-manual) — documents the arm/pre-alarm/disarm state machine
- [KS-SF22R manual](https://manualsee.com/blog/MFvgEnoyT.html) — sibling; explicit about the triaxial accelerometer and false-alarm rejection
- [Wsdcam 113dB bike alarm](https://www.wsdcam.com/products/wsdcam-113db-bike-alarm-wireless-vibration-motion-sensor-waterproof-motorcycle-alarm-with-remote) — close sibling with published AAA standby figures
- [rtl_433](https://github.com/merbanan/rtl_433) — for decoding the 433.92 MHz fob

### Accelerometer

- [SparkFun ADXL313 Arduino Library](https://github.com/sparkfun/SparkFun_ADXL313_Arduino_Library) — I2C and SPI, with interrupt and low-power examples. The basis for any Meshtastic ADXL313 driver
- [SparkFun ADXL313 hookup guide](https://learn.sparkfun.com/tutorials/sparkfun-qwiic-3-axis-accelerometer-adxl313-hookup-guide/arduino-library) — activity/inactivity threshold registers explained

### Siren and piezo drive

- [Diodes AN1176 — Design Considerations for Driving Piezoelectric Buzzers](https://www.diodes.com/assets/App-Note-Files/Design-Considerations-for-Driving-Piezoelectric-Buzzers.pdf?v=7) — SPL vs voltage and frequency, lifetime vs voltage, and the PAM89xx driver comparison. The best single reference here
- [How does the "alarm boost three pin inductor" make a piezo buzzer louder?](https://electronics.stackexchange.com/questions/539843/how-does-the-component-alarm-boost-three-pin-inductor-make-a-piezo-buzzer-loud) — the autotransformer explanation
- [Make this Simple Buzzer Circuit with Transistor and Piezo](https://www.homemade-circuits.com/how-to-make-simple-piezo-buzzer-circuit/) — self-oscillating single-transistor drive using the piezo's centre tap
- [Understanding and Using a Piezo Transducer](https://www.homemade-circuits.com/understanding-and-using-piezo/) — mounting geometry, which matters as much as the drive circuit
- [Simplest Piezo Driver Circuit Explained](https://www.homemade-circuits.com/simplest-piezo-driver-circuit-explained/) — parallel buzzer-coil trick; ferrite core requirement

### Tooling

- [Claude Code memory / CLAUDE.md](https://code.claude.com/docs/en/memory) — `@path` import syntax used by this repo's `CLAUDE.md`

## Licence

Not yet chosen. Note that Meshtastic is GPL-3.0, which constrains anything derived
from it.
