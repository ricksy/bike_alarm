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

### 2.3 Verified hardware — from the boot log

**[VERIFIED]** 2026-07-27, from the I2C scan and GNSS probe in a stock Meshtastic boot
log (`env:rak4631`), two consecutive boots, identical results.

| Item | Finding | How |
|---|---|---|
| **GNSS** | **RAK12500** (u-blox ZOE-M8Q), on **UART** not I2C | Log reports `U-blox 8 detected`, `FWVER=SPG 3.01`, `PROTVER=18.00`, `GPS;GLO;GAL;BDS`, then `GPS+SBAS+GLONASS+Galileo configured`. A RAK1910 carries a MAX-7Q and would report *u-blox 7*. The scan probed I2C `0x42` and got `0xffff`, so it is serial-attached |
| **Base board** | RAK19007 family — **no MAX17048 fuel gauge**, ADC battery sensing | `Power::max17048Init lipo sensor is not ready yet`, followed by `Power: battery hardware detected`. Reading 4.271–4.292 V, `battery_level=101` (charging or full) |
| **RTC** | **None** | `RTC not found (found address 0x00)` |
| **Screen / I2C keyboard** | None | Scan of 0x1f, 0x34, 0x55, 0x5a, 0x5f found nothing |
| **I2C bus** | **Exactly one device, at 0x1d** | `1 I2C devices found` |
| **Accelerometer** | **No LIS3DH.** See §3 | Nothing answered at 0x18 or 0x19 |
| BLE | Working | Connected and secured to a phone |

### 2.4 Still open

- **[UNVERIFIED] Is the separately-ordered RAK4630 a bare stamp module or a WisBlock
  Core?** The "without IPEX / antenna via RF trace pinout" option suggests it may need
  a carrier that routes the antenna, rather than accepting a u.FL pigtail directly.
  Check before assuming it is a drop-in spare for the Core in the kit.
- **Does the Kit 2 box contain an unfitted RAK1904?** The boot log proves nothing is
  *attached* at 0x18/0x19 — not that one is not owned. Check the box before ordering.
- **[UNVERIFIED]** Exact base board part number. The absence of a MAX17048 narrows it to
  the RAK19007 family but does not pin it down. Read the silkscreen.

### 2.5 Siren

Decision: **reuse a self-contained 12 V piezo siren**, switched by a MOSFET and fed
from a boost converter. Rationale: the loudness is already proven, and reusing a
finished module is the lowest-risk path for v1 versus designing a piezo driver from
scratch. A discrete driver can come later if there is a reason.

Not yet designed: the MOSFET/boost circuit itself, gate drive, inrush behaviour, and
what the boost converter's own quiescent current does to the standby budget. **The
boost converter must be fully disabled in standby, not merely unloaded.**

---

## 3. BLOCKER: no usable accelerometer is attached

**[VERIFIED]** against `meshtastic/firmware` @ `5c5cb094` and the boot log of 2026-07-27.

### 3.1 The ADXL313 is unsupported *and* actively misidentified

A grep for `adxl` across `src/` returns **zero hits**. The RAK12032 (ADXL313) has no
driver, no `ScanI2C` entry and no `AccelerometerThread` case.

Worse, the boot log reports:

```
DFRobot Rain Gauge found at address 0x1d
1 I2C devices found
```

There is no rain gauge. From `src/configuration.h`:

```c
#define DFROBOT_RAIN_ADDR 0x1d
#define LIS3DH_ADDR       0x18
#define LIS3DH_ADDR_ALT   0x19
```

**Positively identified 2026-07-27** by photograph of the module: silkscreen reads
`RAK12032 VA` with `SDA`/`SCL` pads, and the chip carries the Analog Devices logo over
`XL 313` — AD marks the ADXL series as "XL", so this is an ADXL313. Date code `#2142`,
lot `29645`. An `X` arrow marks axis orientation. This is no longer an inference.

0x1D is the ADXL313's default I2C address (0x53 alternate), which matches the scan
exactly. Meshtastic claims a rain gauge because the scanner handles 0x1d with a bare
`SCAN_SIMPLE_CASE` — address match, no verification. Contrast LIS3DH detection, which
reads register `0x0F` and requires `0x3300` or `0x3333` before claiming the part.

This is the same class of defect that has stalled upstream issue #9750 for five months:
naive address-only detection colliding across unrelated sensors.

**Immediate hazard: do not enable environment telemetry.** Only `DeviceTelemetry` is
currently running, so nothing touches the phantom sensor. Enable environment metrics and
`DFRobotGravitySensor` will start issuing rain-gauge reads against the accelerometer.
Simplest mitigation for now: **physically remove the RAK12032.**

### 3.2 There is no LIS3DH either

Nothing answered at 0x18 or 0x19. Supported accelerometers are LIS3DH, BMA423, BMI270,
BMX160, ICM20948, ICM42607P, LSM6DS3, MPU6050, QMA6100P and STK8BAXX — **none of them is
present.**

**v1 cannot proceed until this is resolved.** The accelerometer is the trigger; there is
no project without it.

### 3.3 Reassessment: the ADXL313 may be the *better* part

Initial guidance was to treat the ADXL313 as a spare and use a LIS3DH. On closer
inspection that undersold it.

**Analog Devices' own listed example applications for the ADXL313 begin with "car
alarm"** (then hill start aid, electronic parking brakes, data recorders). The part is
designed for this exact application.

Capabilities relevant here:

- **Activity (motion present) and inactivity (motion absent) detection**, user-defined
  threshold on any axis, **both mappable to interrupt pins** — the dual-interrupt pattern
  of §5, natively supported
- 32-level FIFO, explicitly to reduce host intervention and system power
- Low power modes for "intelligent motion-based power management with threshold sensing"
- ±0.5/1/2/4 g, up to 13-bit — low-g ranges suit a bike far better than the LIS3DH's
  ±2–16 g
- I2C 0x1D default / 0x53 alternate; SPI also available

**And the driver is less work than first estimated.** The open-source
[SparkFun_ADXL313_Arduino_Library](https://github.com/sparkfun/SparkFun_ADXL313_Arduino_Library)
supports I2C and SPI, ships examples for low power modes and interrupts, and already
exposes `setActivityThreshold()`, `setInactivityThreshold()`, `setTimeInactivity()` over
the `THRESH_ACT` / `THRESH_INACT` / `TIME_INACT` registers.

For scale: `LIS3DHSensor.cpp` is 35 lines, because `MotionSensor` does the heavy lifting.
An `ADXL313Sensor` built on the SparkFun library is plausibly comparable. Revised
estimate: **a focused weekend or two, not weeks.**

The fiddly part is not the driver but the **`ScanI2C` entry**, which must disambiguate
0x1d by reading the device-ID registers to tell an ADXL313 from a genuine DFRobot rain
gauge. That is a real upstream bug fix, and the same class of defect blocking #9750 — so
a PR adding address disambiguation *plus* an ADXL313 driver is well-scoped and likely
welcome.

### 3.4 Plan

1. **Order a RAK1904 (LIS3DH) anyway** — roughly €10. Not because the ADXL313 is wrong,
   but because a supported part removes one unknown while everything else in the system
   is unproven. When interrupt handling misbehaves, a known-good reference tells you
   whether the fault is the driver, the ISR, or the state machine.
2. **Check the Kit 2 box first** for an unfitted RAK1904. The RAK12500 occupies a UART
   slot, so an I2C slot is free.
3. **Start the ADXL313 driver while waiting for delivery.** The project is blocked
   regardless; nothing is wasted.
4. **Keep both parts.** A/B comparison is genuinely valuable when tuning false-alarm
   discrimination — the hard problem flagged in `reference-device.md` §8.
5. Until resolved, **remove the RAK12032** so the phantom rain gauge cannot be probed
   (§3.1).

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
plausibly upstreamable rather than a permanent fork. No upstream work exists on
interrupt-driven accelerometer wake — see `prior-art.md`.

### Use both interrupt lines, not one

The LIS3DH exposes INT1 and INT2. Configure them as a pair:

- one line for **significant motion** → wake / trigger
- one line for **no motion sustained** → the bike has been put down again

This gives the state machine "started moving" *and* "stopped moving" as hardware
events, and the latter is exactly what auto-rearm needs. Detach the no-motion interrupt
while asleep so it cannot spuriously re-wake the MCU. Pattern borrowed from the Long
Cricket asset tracker; see `prior-art.md`.

### Constraint: waking from System OFF resets the CPU

On nRF52, `doDeepSleep()` calls `sd_power_system_off()`. **Waking from System OFF
triggers a full CPU reset** — execution restarts at the reset vector, `SystemInit()`
runs again, all globals re-initialise. Every wake is indistinguishable from a cold boot.
The nRF52840 supports RAM retention in System OFF, but the firmware does not currently
use it. (Source: meshtastic/firmware issue #10945.)

**This is structural.** An accelerometer interrupt does not resume the alarm state
machine — it reboots the device. Armed/disarmed state, pre-alarm counters and auto-rearm
timers must survive a reset, via GPREGRET, retained RAM, or flash. Design the state
machine as *restartable from persisted state*, not as a resident object holding state
in RAM.

---

## 5a. Power budget reality check

The Long Cricket asset tracker (see `prior-art.md`) reaches **~2.5 µA sleep current**,
and its designer estimates that an urban bicycle tracker on that hardware would last
**several months**. That independently validates this project's target as achievable.

But it got there largely by *removing* things:

| Change | Saving |
|---|---|
| 20 nA-quiescent LDO instead of a conventional one | ~0.5 µA |
| Battery charger removed entirely | ~2 µA |
| FET-gating the battery-monitor voltage divider | ~3 µA |
| ~800 nA accelerometer instead of a ~6.5 µA one | ~5.7 µA |

**Almost every one of those is a convenience the WisBlock base board provides and this
project cannot delete** — Li-ion charger, solar input, battery-sense divider, LEDs, an
LDO not chosen for nanoamp quiescent current.

Therefore: **the realistic board-level floor here is plausibly tens of µA, not single
digits.** Still compatible with months of standby on a decent cell plus solar, but the
nRF52840 datasheet System OFF figure is not the number to plan against.

### The benchmark is not 2.5 µA — it is 75 µA

The Long Cricket is a useful teardown, not the target. The target is the commercial
alarm this project has to beat, and it is now a known figure:

> **75 µA whole-device average** (manufacturer spec), giving 6–10 months on a 700 mAh
> cell with **no solar input at all**. See `reference-device.md` §3.

That reframes the WisBlock overhead. Tens of µA of unavoidable board overhead is fatal
against a 2.5 µA benchmark and merely expensive against 75 µA. With solar on top there
is real headroom — but the budget is tight, not generous.

### Draft budget

| Item | Estimate |
|---|---|
| nRF52840, System OFF with RAM retention | ~2 µA |
| LIS3DH, low-power mode, interrupt armed | 2–10 µA |
| Siren boost converter in true shutdown | ~1 µA |
| **WisBlock board overhead** (charger, sense divider, LDO, LEDs) | **10–50 µA — unknown** |
| **LoRa receive duty cycling** | **see below** |
| **Total at a 1 s radio poll** | **~35–90 µA** |

The width of that range is almost entirely the WisBlock overhead. **This makes the bare
board deep-sleep measurement the single decisive number in the project** — the
difference between comfortably beating 75 µA and missing it.

### Radio duty cycle — the real constraint

SX1262 receive is roughly 4.6–5.3 mA. **[UNVERIFIED — reasoning from typical figures,
not a datasheet lookup. Verify.]**

- Polling every **200 ms** like the reference device, with a CAD of ~5 ms including wake
  and settle, is ~2.5% duty — about **115 µA for the radio alone**. Over budget before
  anything else is counted.
- Polling every **1 s** is ~0.5%, roughly **20–25 µA**. Affordable, with a worst-case
  latency of one second — imperceptible for arm/disarm.

**LoRa's long preambles, normally a liability, are an asset here.** The reference device
must poll at 200 ms because an OOK packet lasts milliseconds. A LoRa preamble can
comfortably exceed a second, so this project can poll slowly *because* the modulation is
slow.

Higher spreading factors make CAD proportionally more expensive — longer symbols — while
buying range. Range, latency and current form a three-way trade that only measurement
resolves.

**Consequence for the roadmap:** measure the reference device first, the bare WisBlock
second. Those two numbers — the target set by the incumbent and the floor imposed by the
board — bound the entire design space, and both are obtainable in an afternoon.

## 6. Software plan

**v1 — prove the concept using stock modules.** Detection Sensor + External
Notification + Remote Hardware, wired up via configuration only. Accept bad battery
life. Goal is a loud noise when the bike moves and a LoRa message that arrives.

**v2 — custom firmware module.** Once v1 proves the hardware, a proper alarm state
machine is needed:

- States: disarmed, armed, pre-alarm (warning chirp), full alarm, auto-rearm
- **State persisted across resets** — see the System OFF constraint in §5
- Escalation logic and siren timeout
- Arm/disarm over LoRa, ideally authenticated
- Interrupt-driven wake, using both LIS3DH interrupt lines, as described in §5

**Not yet decided — radio duty cycle.** There is a direct trade-off between how
responsive the device is to a remote disarm/locate command and how long it survives on
standby. Continuous RX is out of the question. Needs an explicit decision, ideally
backed by measurement.

---

## 6a. Dependency: the alert path needs a second node

**[VERIFIED]** 2026-07-27 from the boot log: `num_online_nodes=1, num_total_nodes=1`,
zero packets received from any peer across two minutes, noise floor -96 dBm, channel
utilisation ~1%.

**There is no mesh in range.** Indoors and over two minutes that is not conclusive, but
it exposes an assumption the design never stated.

A phone talks to the node over **BLE**, which is a few metres. That is not "alert me
while I am in the supermarket." For remote alerting to mean anything, one of these has to
be true:

1. **A second node at home** acting as receiver/gateway — the realistic answer, and it
   doubles as the LoRa test partner needed for any radio duty-cycle work
2. An existing community mesh with coverage where the bike is parked — needs checking
   locally, not assumable
3. Alerting degrades to *local siren only*, with mesh alerting as a bonus when in range

**Consequence:** budget for a second node. It is required for development regardless —
duty-cycle and range testing need two radios — so this is not extra cost, just earlier
cost than planned. Note also that a months-standby leaf node cannot itself relay for
others (§5), so this project consumes mesh coverage without contributing it.

## 7. Roadmap / open work

### Done

- ~~Flash stock Meshtastic `env:rak4631`, read the I2C scan and GNSS probe~~ —
  **2026-07-27**, results in §2.3. GPS confirmed working; accelerometer situation
  discovered.

### Blocking — nothing else matters until these are cleared

1. **Obtain a working accelerometer.** Check the Kit 2 box for an unfitted RAK1904;
   failing that, order one (~€10). See §3.4. In parallel, begin the ADXL313 driver — the
   part is already owned and is arguably better suited (§3.3).
2. **Remove the RAK12032** so the phantom rain gauge at 0x1d cannot be probed (§3.1).

### Next — two measurements that bound the whole design

3. **Profile the reference device** (`reference-device.md` §6). Gives the 75 µA target
   measured backing, and practice with the rig on known-good hardware.
4. **Baseline the bare WisBlock in deep sleep.** §5a explains why this may cap what any
   firmware work can achieve.

### Then

5. Acquire a **second node** — required for any radio testing, and for alerting to work
   at all (§6a).
6. Read the base board silkscreen to pin down the exact part number (§2.4).
7. Design the siren driver circuit — MOSFET, boost converter, true standby shutdown
   (`siren-driver.md`).
8. LIS3DH interrupt-driven wake, both interrupt lines (driver change, §5).
9. Persist alarm state across resets — GPREGRET, retained RAM, or flash (§5).
10. Re-measure current against the 75 µA target.
11. Alarm state machine module (§6, and `reference-device.md` §4 for behaviour).
12. Radio duty-cycle strategy (§5a) — the one problem with no prior art.
13. GPS power gating, ephemeris-aware duty cycle. **GPS itself already works** (§2.3).
14. Battery life model including solar input.

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

## 10. Prior art

See `prior-art.md`. Headlines:

- **No finished open-source Meshtastic bike alarm exists.**
- **LoRider** (Hackster.io, 2025) reached the same architecture independently but was
  never implemented — the author lacked firmware experience.
- **meshtastic/firmware #9750 ("Claymore module")** is an open request for exactly this
  kind of motion-triggered alarm module. Stalled on an unrelated I2C address collision.
  There is upstream appetite; an argument for building this upstreamably.
- **Long Cricket** (Tlera Corp) is the power-engineering reference — see §5a.
- **Nothing upstream** addresses interrupt-driven accelerometer wake.

---

*Codebase findings verified against `meshtastic/firmware` @ `5c5cb094` (2026-07-26).
Re-verify if working against a substantially later commit.*
