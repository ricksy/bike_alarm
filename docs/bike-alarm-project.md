# WisBlock Bike Alarm — Project Context

> **For the agent reading this:** This is the design context for a hardware/firmware
> project built on Meshtastic. Read it before making changes. Sections marked
> **[UNVERIFIED]** are assumptions that have not been physically or empirically
> confirmed — do not build on them without checking first. Sections marked
> **[VERIFIED]** were confirmed by reading the Meshtastic source at the commit noted
> below.

---

## 1. What this is

A DIY electronic anti-theft device for a bicycle. Motivation is personal: repeated
bike theft. Two things have empirically worked in the past — a loud alarm, and
AirTag-style tracking — so the design leans on both.

**Core functions**

| Function | Mechanism |
|---|---|
| Deterrence | High-decibel piezo siren |
| Trigger | Accelerometer motion detection |
| Remote alerting | LoRa / Meshtastic (EU868) |
| Recovery | GNSS position reporting |
| Endurance | Li-ion + solar trickle charge |

**The hard constraint: standby battery life measured in months**, comparable to
commercial bike alarms. This is the design driver and the tiebreaker for every
architectural decision below. If a proposed change costs standby current, it needs
an explicit justification.

Estimated v1 BOM: roughly €60–80.

---

## 2. Hardware

### 2.1 Ordered and in hand

- **WisBlock Kit 2** — LoRa-based GPS Tracker with Solar Panel, EU868 × 1
- **RAK4630** nRF52840 + SX1262 LoRa/BLE module × 1
  — variant: *without IPEX (antenna via RF trace pinout)*, 8xx MHz band
- **RAK12500** — GNSS module, u-blox ZOE-M8Q × 1
- **RAK12032** — 3-axis accelerometer, Analog Devices ADXL313 × 1

### 2.2 Platform rationale

nRF52840 (RAK4631/RAK4630) was chosen over ESP32. Reasons, in priority order:

1. Dramatically lower sleep current — decisive for months-scale standby
2. Built-in solar charge input and Li-ion charger on the WisBlock base
3. Native, first-class Meshtastic support

### 2.3 Open hardware questions — resolve these first

- **[UNVERIFIED] What exactly is in the Kit 2 box?** It is believed to contain a
  WisBlock base board, the RAK4631 core, a GPS module (likely RAK1910), an
  accelerometer (likely RAK1904 / LIS3DH), a solar panel and an enclosure.
  **Physically inventory the kit and record the actual part numbers here** before
  writing any pin-level code. The base board matters: RAK19007 and RAK5005-O have
  different pinouts and different quiescent current.
- **[UNVERIFIED] Is the separately-ordered RAK4630 a bare stamp module or a WisBlock
  Core?** The "without IPEX / antenna via RF trace pinout" option suggests it may need
  a carrier that routes the antenna, rather than accepting a u.FL pigtail directly.
  Check before assuming it is a drop-in spare for the Core in the kit.
- **Which GPS is actually wired?** RAK12500 (ZOE-M8Q, I2C) and RAK1910 are different
  parts on different WisBlock slots.

### 2.4 Siren

Decision: **reuse a self-contained 12 V piezo siren**, switched by a MOSFET and fed
from a boost converter. Rationale: the loudness is already proven, and reusing a
finished module is the lowest-risk path for v1 versus designing a piezo driver from
scratch. A discrete driver can come later if there is a reason.

Not yet designed: the MOSFET/boost circuit itself, gate drive, inrush behaviour, and
what the boost converter's own quiescent current does to the standby budget. **The
boost converter must be fully disabled in standby, not merely unloaded.**

---

## 3. Critical finding: the ADXL313 is not supported by Meshtastic

**[VERIFIED]** against `meshtastic/firmware` @ `5c5cb094e6deb6e4d01d4894b73886f43da84a65`
(2026-07-26).

A grep for `adxl` across `src/` returns **zero hits**. The RAK12032 (ADXL313) has no
driver, no `ScanI2C` detection entry, and no `AccelerometerThread` case.

Supported accelerometers are: LIS3DH, BMA423, BMI270, BMX160, ICM20948, ICM42607P,
LSM6DS3, MPU6050, QMA6100P, STK8BAXX.

**Consequence:** use the LIS3DH (RAK1904, expected in Kit 2) for v1. Treat the
RAK12032 as a spare, or as a later project once the architecture is settled. Writing
an ADXL313 driver is a real piece of work and should not be on the v1 critical path.

The RAK12500 GNSS is fine — `ScanI2CTwoWire.cpp` explicitly recognises it as a
u-blox device on I2C, and the GPS layer speaks UBX generically.

---

## 4. Meshtastic codebase orientation

All **[VERIFIED]** at the commit above.

### 4.1 Layout

```
src/
  main.cpp            entry point
  mesh/               protocol: routers, radio drivers, NodeDB, MeshService
  modules/            all features; new code goes here
  motion/             accelerometer + magnetometer drivers
  gps/                GNSS
  detect/             I2C scanning and sensor auto-detection
  power/, sleep.cpp, PowerFSM.cpp   power management
  concurrency/        cooperative threading
variants/nrf52840/rak4631/    variant.h (pin map), platformio.ini (env:rak4631)
boards/wiscore_rak4631.json   PlatformIO board definition
protobufs/                    submodule: wire format and config schema
```

### 4.2 Threading model

There are no real threads on nRF52. `concurrency::OSThread`
(`src/concurrency/OSThread.h`) is a cooperative scheduler. Every subsystem subclasses
it and implements:

```cpp
int32_t runOnce();   // returns milliseconds until it wants to be called again
```

The main loop polls every thread for its next-run time, sleeps until the soonest via
the interruptible `mainDelay`, and repeats. Each thread also sets a `canSleep` flag
indicating whether the CPU may enter a deeper sleep state.

**This is where battery life lives.** Standby current is largely a function of what
return values threads produce and whether anything is holding `canSleep = false`.

### 4.3 Module system

Three base classes, each removing boilerplate:

| Class | File | Use when |
|---|---|---|
| `MeshModule` | `src/mesh/MeshModule.h` | raw packets needed |
| `SinglePortModule` | `src/mesh/SinglePortModule.h` | one portnum owned |
| `ProtobufModule<T>` | `src/mesh/ProtobufModule.h` | payload is a protobuf |

A module is identified by **portnum** (an app ID in the packet). Override
`wantPacket()` to filter, `handleReceived()` / `handleReceivedProtobuf()` to act.
Return `ProcessMessage::CONTINUE` to let other modules also see the packet, `STOP` to
consume it. Set `myReply` to answer a request. `handleAdminMessageForModule()` makes
a module configurable over the admin channel.

Most modules multiply-inherit
`public SinglePortModule, private concurrency::OSThread` so they both receive packets
and get a periodic tick.

Registration is one function: `setupModules()` in `src/modules/Modules.cpp`, a list of
`new SomeModule()` calls guarded by config checks. Constructing a module registers it
globally. That is the entire plugin system.

### 4.4 The three stock modules this project builds on

- **`DetectionSensorModule`** — reads a GPIO, broadcasts a text message on change.
  Configurable trigger type, name, minimum broadcast interval.
- **`ExternalNotificationModule`** — drives an output pin, PWM buzzer, RTTTL or LED on
  incoming messages. Has a "nag" repeat timer and a mute function. Closest existing
  analogue to the siren driver; its nag-cycle logic is a useful model for alarm
  escalation.
- **`RemoteHardwareModule`** — remote GPIO read/write/watch over protobuf. Good enough
  for arm/disarm in v1, before a real state machine exists.

---

## 5. Critical finding: the stock motion path polls

**[VERIFIED].** This is the main obstacle to the battery target.

- `DetectionSensorModule` polls its GPIO every **100 ms**
  (`GPIO_POLLING_INTERVAL` in `DetectionSensorModule.cpp`). No interrupt.
- `LIS3DHSensor::runOnce()` (`src/motion/LIS3DHSensor.cpp`, ~35 lines) polls the click
  register over I2C every **50 ms** (`MOTION_SENSOR_CHECK_INTERVAL_MS` in
  `MotionSensor.h`). The LIS3DH's own INT1/INT2 pins are never attached to an
  interrupt.

The upstream motion subsystem exists to wake a screen or emulate a button press. It
was never designed as a low-power motion trigger. Polling I2C at 20 Hz forever is
incompatible with months of standby.

**Target architecture for the custom work:**

1. Configure the LIS3DH's own inertial-wakeup / activity-threshold registers and route
   the event to INT1.
2. `attachInterrupt()` on that pin — or, on nRF52, use a GPIOTE port event so the
   trigger can wake the MCU from SYSTEM_OFF.
3. Never poll the accelerometer. The sensor is the timekeeper.

Note this is a **driver-level change to `LIS3DHSensor`**, not just a new module. Plan
accordingly. It may be worth structuring as an opt-in low-power mode so the change is
plausibly upstreamable rather than a permanent fork.

---

## 6. Software plan

**v1 — prove the concept using stock modules.** Detection Sensor + External
Notification + Remote Hardware, wired up via configuration only. Accept bad battery
life. Goal is a loud noise when the bike moves and a LoRa message that arrives.

**v2 — custom firmware module.** Once v1 proves the hardware, a proper alarm state
machine is needed:

- States: disarmed, armed, pre-alarm (warning chirp), full alarm, auto-rearm
- Escalation logic and siren timeout
- Arm/disarm over LoRa, ideally authenticated
- Interrupt-driven wake as described in §5

**Not yet decided — radio duty cycle.** There is a direct trade-off between how
responsive the device is to a remote disarm/locate command and how long it survives on
standby. Continuous RX is out of the question. Needs an explicit decision, ideally
backed by measurement.

---

## 7. Roadmap / open work

1. Physically inventory Kit 2; record actual part numbers (§2.3)
2. Flash stock Meshtastic `env:rak4631`, confirm the mesh works, confirm which sensors
   the I2C scan reports at boot
3. Design the siren driver circuit — MOSFET, boost converter, standby shutdown
4. Baseline current measurement: stock firmware, idle
5. LIS3DH interrupt-driven wake (driver change)
6. Re-measure current; compare against the months-scale target
7. Alarm state machine module
8. Radio duty-cycle strategy, informed by 4 and 6
9. GPS integration and power gating
10. Battery life model including solar input

---

## 8. Build and workflow

```bash
pio run -e rak4631     # build
./bin/run-tests.sh     # native test suite (0 GREEN, 1 RED, 2 AMBER, 3 FILTERED)
trunk fmt              # formatting — run before committing
```

The repo ships `AGENTS.md` and `.github/copilot-instructions.md`; the latter is the
canonical agent-facing document covering layout, conventions, build system, CI and the
test harness. **Read it before any non-trivial change.**

Suggested reading order for orientation:
`main.cpp` → `concurrency/OSThread.h` → `mesh/MeshModule.h` → `modules/Modules.cpp` →
`modules/DetectionSensorModule.cpp` (smallest complete example) →
`modules/ExternalNotificationModule.cpp` → `motion/AccelerometerThread.h` +
`LIS3DHSensor.cpp` → `sleep.cpp` + `PowerFSM.cpp`.

---

## 9. Working principles for this project

- **Standby current is the tiebreaker.** Any change that costs it needs justification.
- **Interrupt-driven, never polled**, for anything in the always-on path.
- **Measure, don't estimate.** Current-draw claims should be backed by a meter reading.
- **Reuse proven components for v1**; optimise and customise once the concept holds.
- **Prefer upstreamable changes** over a divergent fork where the cost is similar.
- Keep the **[UNVERIFIED]** markers honest — promote items to **[VERIFIED]** only after
  actually checking, and note how they were checked.

---

*Codebase findings verified against `meshtastic/firmware` @ `5c5cb094` (2026-07-26).
Re-verify if working against a substantially later commit.*
