# Prior Art

What already exists, what it achieved, and what this project can take from it.

Surveyed 2026-07-27. Sources are linked; re-check before relying on anything
time-sensitive.

---

## Summary

| Project | Overlap | Status | Value here |
|---|---|---|---|
| [LoRider](https://www.hackster.io/Milliam/lorider-the-meshatastic-0cffeb) | Meshtastic bike anti-theft tracker | **Not built** | Validates the architecture |
| [Claymore module #9750](https://github.com/meshtastic/firmware/issues/9750) | Meshtastic tripwire/alarm module | **Open, stalled** | Upstream appetite exists |
| [Long Cricket](https://www.tindie.com/products/tleracorp/long-cricket-loralorawangnss-asset-tracker/) | Ultra-low-power LoRa+GNSS tracker | **Shipped product** | Power engineering reference |

**No finished open-source Meshtastic bike alarm exists.** A GitHub repository search
for "meshtastic bike" returns zero results. The only Meshtastic Discourse thread about
bicycles dates from 2020 and concerns tracking riders during a charity relay, not theft.

---

## LoRider — the same idea, unfinished

Hackster.io, October 2025. Built for the Meshtastic Device Design Challenge by a
developer in Seoul whose electric bike kept getting stolen.

Their state machine, arrived at independently, is essentially this project's:

1. **Parked** — GPS off, accelerometer in low-power wake-on-motion mode, "guard mode"
2. **Motion detected** — accelerometer interrupt fires, GPS powers on, alarm alert sent
   over Meshtastic
3. **Tracking** — GPS logs live position, owner follows in the app
4. **Return to park** — GPS off (manual command or timeout), accelerometer resumes
   standby

**It was never implemented.** The author states plainly that they were unable to do
much function-wise, citing lack of support for the Wio Tracker L1's additional inputs
and outputs, and that this would require writing firmware — something they were not
familiar with. They got some LEDs lit.

**Takeaway:** the concept is sound and has been independently converged on. The blocker
is firmware, which is exactly where this project starts. Different hardware
(Wio Tracker L1 vs RAK4631), so no code to reuse.

---

## Claymore module (meshtastic/firmware #9750) — upstream wants this

Open feature request, February 2026, 19 comments.

The proposal: a Meshtastic node acting as a tripwire or house alarm. A VL53L… ToF
sensor detects a distance change and sends a message to a preconfigured channel. The
author notes the alarm could equally be triggered by a BMA400 accelerometer, for a node
placed on a door.

That is this project, generalised.

**Current state: stalled on an unrelated problem.** The thread has been consumed
entirely by an I2C address collision — VL53L0X and VL53L1X share address 0x29 but need
incompatible driver libraries. Debate ranged over abstraction layers, modifying vendor
headers, and switching to Pololu or ST libraries. The accelerometer branch was raised
again in May 2026 ("would also be interesting to include an accelerometer trigger, to
secure doors or crates") and nobody picked it up.

**Takeaway:** there is a constituency for a motion-triggered alarm module upstream. A
clean accelerometer-triggered implementation would land in receptive territory, and
that is an argument for building this as an upstreamable module rather than a private
fork. Worth commenting on the issue once there is something to show.

---

## No upstream work on interrupt-driven accelerometer wake

Searched meshtastic/firmware issues and PRs for interrupt-driven accelerometer wake.
Nothing exists.

The nearest recent work is PR #11025 (July 2026), "Stop accelerometer thread when
double-tap/wake-on-motion disabled at runtime" — a power fix that *removes* the polling
thread when unused, rather than replacing polling with interrupts.

**Takeaway:** the LIS3DH INT1 change described in the main design doc appears to be
genuinely novel upstream.

---

## Related open issues worth tracking

| Issue | Why it matters here |
|---|---|
| [#10945](https://github.com/meshtastic/firmware/issues/10945) | nRF52 System OFF loses the clock every wake — see the reset constraint below |
| [#9699](https://github.com/meshtastic/firmware/issues/9699) | nRF52 solar nodes: inconsistent wake behaviour and brownout risks |
| [#11211](https://github.com/meshtastic/firmware/pull/11211) | nRF52 opt-in low-voltage boot protection |

The last two are directly relevant to the solar-charging plan.

---

## Long Cricket Asset Tracker — the power engineering reference

Tlera Corp (Kris Winer). A shipped product, ~$80, with the full Arduino source
published at
[kriswiner/CMWX1ZZABZ](https://github.com/kriswiner/CMWX1ZZABZ/tree/master/longCricketAssetTracker).

Hardware: Murata CMWX1ZZABZ-078 (STM32L082 + SX1276), u-blox CAM M8Q GNSS, **BMA400**
accelerometer, BME280, 8 MB SPI NOR flash.

Different MCU, different radio stack (LoRaWAN, not Meshtastic), different sleep model.
Take it as a power-engineering reference, not a code source.

### The headline numbers

- **Sleep current as low as ~2.5 µA** (~12 µA with GNSS backup voltage on)
- ~250 µA average with a GNSS fix every 2 hours, sensor reads every minute and a
  LoRaWAN update every 10 minutes
- Over a year on a 3.6 V 2400 mAh LiSOCl₂ AA cell in that configuration

Winer names this project's exact use case: tracking bicycles in an urban environment
would be unlikely to last a year given the frequent motion expected, but **could last
several months**, with logging, enabling recovery if the bicycle were stolen.

**This is independent expert validation that the months-scale target is achievable.**

### Where the microamps went

Documented savings, from the product description:

| Change | Saving |
|---|---|
| MCP1810 LDO (20 nA quiescent) replacing the NCV8170 | ~0.5 µA |
| **Removing the STBC08 battery charger entirely** | ~2 µA |
| **FET-gating the battery-monitor voltage divider** so it only draws when sampling | ~3 µA |
| BMA400 (~800 nA continuous wake/sleep monitoring) instead of BMA280 (~6.5 µA) | ~5.7 µA |
| GNSS VDD on a dedicated 100 nA standby LDO; backup voltage on a GPIO so it can reach zero | — |

### The uncomfortable implication for this project

**Almost every saving above came from removing or gating exactly the conveniences a
WisBlock base board provides.** The WisBlock has a Li-ion charger, a solar input, a
battery-sense divider, LEDs, and an LDO that was not selected for 20 nA quiescent
current. None of that can be deleted.

The realistic board-level floor for this project is therefore plausibly **tens of µA,
not single digits**. That is still compatible with months of standby on a reasonable
cell plus solar input — but the nRF52840 datasheet System OFF figure is not the number
to plan against.

**Action:** measure the bare WisBlock in deep sleep *before* optimising firmware. If
the board floor turns out to be, say, 40 µA, then no amount of accelerometer interrupt
work materially changes the outcome, and that needs to be known in week one rather than
month three. See `measurements.md`.

### The dual-interrupt pattern — worth copying

The single most transferable idea in the sketch.

The BMA400 drives **two** interrupt lines into the MCU:

- **INT1** — no-motion detected → "go to sleep"
- **INT2** — significant motion detected → "wake"

Both are attached as rising-edge interrupts. On a wake event the firmware re-attaches
the sleep interrupt. On a sleep event it **detaches** the sleep interrupt, specifically
so that it cannot spuriously re-wake the MCU. That detail is the kind of thing learned
by being bitten.

The LIS3DH also has INT1 and INT2. Adopting this pattern gives the alarm state machine
both "started moving" and "stopped moving" as hardware events — and "stopped moving" is
exactly what auto-rearm needs.

Winer's framing of the intent: the tracked asset is stationary most of the time so
updates are infrequent, but when it moves, the device should follow the trajectory and
report promptly. Substitute "sound the siren" for "report" and that is this project.

### Other practices worth adopting

- **Tear down peripherals before sleeping.** The sketch calls `SPI.end()` on the way
  into STOP mode and `SPI.begin()` on the way out, with the flash chip explicitly
  powered down in between. Peripherals left initialised keep clocks and pads alive.
- **GNSS duty cycling with ephemeris awareness.** Fixes every two hours because
  ephemeris goes stale after four. Backup voltage keeps warm/hot starts cheap at
  ~10 µA, or is zeroed entirely when the duty cycle exceeds the ephemeris window.
  Directly applicable to the RAK12500.
- **Accelerometer configured for minimum power**: ±2 g range, low-power mode, lowest
  oversampling. Accuracy is irrelevant for a motion trigger.
- **Log to flash, transmit summaries.** For a bike alarm, a local motion-event history
  could have forensic value after a theft.

### What does not transfer

**The sleep model.** `STM32L0.stop()` *returns* — STOP mode preserves RAM and execution
resumes on the following line. The whole sketch is structured around resumption. The
nRF52 equivalent (System OFF) resets the CPU. See the reset constraint below.

**Radio duty cycle.** LoRaWAN is transmit-mostly; Long Cricket talks to a gateway and
never needs to listen for commands. It therefore has nothing to say about the RX-window
trade-off that remote arm/disarm requires. That problem is unsolved here and Long
Cricket offers no help with it.

**The siren.** It is a tracker, not an alarm. Nothing about pulsed high-current loads,
boost converter inrush, or keeping a 12 V piezo supply genuinely off during standby.

---

## Constraint: nRF52 System OFF resets the CPU

From meshtastic/firmware issue #10945, and important enough to record here as well as
in the main design doc.

On nRF52, `doDeepSleep()` calls `sd_power_system_off()`. **Waking from System OFF
triggers a full CPU reset** — execution restarts at the reset vector, `SystemInit()`
runs again, and all globals are re-initialised. From the firmware's point of view every
wake is indistinguishable from a cold boot. The nRF52840 *supports* RAM retention in
System OFF, but the firmware does not currently exploit it.

**Consequence for the alarm state machine:** if standby uses System OFF, an
accelerometer interrupt does not resume the state machine — it reboots the device.
Armed/disarmed state, pre-alarm counters and auto-rearm timers must survive a reset,
via GPREGRET, retained RAM, or flash.

This is structural, not a detail. Design the state machine as restartable from
persisted state rather than as a resident object holding state in RAM.
