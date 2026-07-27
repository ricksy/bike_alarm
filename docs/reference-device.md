# Reference Device — King-Serry KS-SF32R

The commercial bike alarm currently in use. It is the benchmark this project has to
match on standby life and loudness, and — usefully — its behaviour is documented well
enough to serve as a starting specification for the alarm state machine.

Identified 2026-07-27 from the PCB silkscreen. Items marked **[UNVERIFIED]** are
inference, not fact.

---

## 1. Identification

The silkscreen reads `KS-SF32R`, `FR4 1.2mm HC`, `V2.0`, and `金西瑞` (Jīn Xī Ruì).

- **Manufacturer: King-Serry** (`king-serry.com`) — `金西瑞` is the same company name in
  Chinese. Their catalogue lists KS-SF22R, KS-SF32R and KS-SF04R vehicle anti-theft
  alarms, plus wireless switches, doorbells, window/door alarms and brake taillights.
- **Rebranders:** VioRose (the unit in hand), NineLeaf, Mengshen, and others. Wsdcam
  sells a close sibling.

**Search `KS-SF32R`, not the retail brand.** Full user manuals are available at
manuals.plus, device.report and manualspro.

---

## 2. Published specifications

| | |
|---|---|
| RF frequency | 433.92 MHz |
| Remote range | 30–60 m claimed (~30 m observed in use) |
| Loudness | 110–113 dB claimed, "car-level 6-segment alarm sound" |
| Battery | Built-in Li-ion, **700 mAh**, USB-C charging, not user-replaceable |
| Standby | **75 µA** quiescent; "more than 6–10 months" |
| Sensor | Triaxial accelerometer with discrimination algorithm |
| Sensitivity | 7 levels, set via remote |
| Volume | 3 levels |
| Ingress | IP54–IP55 depending on listing |
| Power saving | Marketed as "the 4th version of power-saving technology" |

---

## 3. The power budget — **target: 75 µA**

**Manufacturer standby current: 75 µA.** Cell: **700 mAh** Li-ion. Both confirmed
2026-07-27.

### The spec is internally consistent

700 mAh at 75 µA is 9,333 h — 12.8 months ideal. Applying two real-world derates:

- ~90% usable capacity (Li-ion protection cutoff, never discharged to zero)
- Li-ion self-discharge at 2–3% per month

| Self-discharge assumed | Resulting standby |
|---|---|
| 0% (ideal) | 11.6 months |
| 2%/month | 9.2 months |
| 3%/month | 8.4 months |

That lands inside the claimed 6–10 months. The numbers agree with each other, which is
worth noting — for products in this segment, they frequently do not.

### What it implies about their design

A continuously-listening 433 MHz receiver draws ~4 mA. On 700 mAh that is **7.3 days**.
So duty-cycled receive is not an inference, it is arithmetic.

Assuming a non-radio baseline of 10–20 µA, the receiver's share is 55–65 µA:

| Baseline | Receiver current | Implied duty | Equivalent |
|---|---|---|---|
| 10 µA | 4 mA | 1.6% | ~3.2 ms per 200 ms |
| 15 µA | 4 mA | 1.5% | ~3.0 ms per 200 ms |
| 20 µA | 5 mA | 1.1% | ~2.2 ms per 200 ms |

So roughly a **2–4 ms sniff window every 200 ms**. Field-proven parameters rather than
a guess, and a useful starting point for this project's own radio duty cycling — with
the important difference noted in `bike-alarm-project.md` §5a.

### **75 µA is the number this project has to match.**

Whole-device average, delivering months of standby and a remote that feels instant, with
no solar input at all. Treat it as the benchmark in every power discussion in this
repository.

Worth verifying directly (§6) — it is a manufacturer's figure — but the uncertainty is
now small enough to design against.

## 4. Documented behaviour — a free state-machine specification

Taken from the KS-SF32R user manual. This behaviour has been tuned against real
customers and real false alarms, which makes it a better starting point than a
first-principles design.

### States and transitions

- **Shipping / storage mode.** The unit arrives in a low-power mode in which the remote
  cannot control it at all. It must be charged for at least a minute to activate.
- **Arm.** Press the arm button → "Duo.Re.Mi" prompt tone → **~5 second delay** → a "Bi"
  and it enters armed mode. That gap is an exit delay, letting the user walk away
  without tripping it.
- **Armed + motion detected → 2 seconds of alarm sound.** This is a **pre-alarm warning
  chirp**, not the full siren. Escalation to sustained alarm presumably follows continued
  motion. **[UNVERIFIED — confirm the escalation rule from the full manual or by
  experiment.]**
- **Disarm.** Dedicated remote button.
- **Bell / search.** Makes a locating sound; 3 selectable ringtones. Only works when
  disarmed.
- **SOS.** Triggers a call-for-help sound.

### Configuration

- Setting mode is entered by pressing disarm, then holding disarm until the remote's
  indicator LED goes off and stays on. It times out after 10 s of inactivity, and
  **cannot be entered while armed**.
- 7 sensitivity levels, indicated audibly by the seven tones Duo–Re–Mi–Fa–Suo–La–Xi,
  high to low.
- 3 volume levels.
- Remote pairing procedure exists (manual §4.6.1).
- Low-battery indication is a weak "Du..Du" tone when arming or disarming.

### What to take from this

- **The exit delay and the pre-alarm chirp are both worth copying.** They are the
  difference between a usable alarm and one that shrieks at its owner.
- **Audible feedback for every state change.** Distinct tones for arm, disarm,
  pre-alarm, setting mode, low battery. Cheap to implement, and it is most of the
  perceived quality.
- **Configuration locked out while armed** — a small but sensible safety property.
- **Everything is driven from a two- or three-button fob.** No screen, no phone. Worth
  remembering when the temptation arises to make configuration app-only.

---

## 5. Board observations

From a photograph of the PCB, 2026-07-27. Silkscreen designators: `ANT` (bottom left),
`SW1` (left edge), `BAT` (top right, JST with red/black pair), `BUZ` (bottom right, JST).

| Observation | Confidence |
|---|---|
| Meandering serpentine trace across the upper middle is the 433 MHz PCB antenna | High — and it explains the 30–60 m range rather than hundreds of metres |
| Metal-topped part, top left, is the RF front end (likely a SAW resonator) | Medium |
| Large SOP-16-class IC in the centre is the MCU | High on role, markings not legible |
| Small SOT-23-class part beside the USB-C is the Li-ion charger (TP4056/LTC4054 class) | Medium |
| One of the small square packages near the bottom centre is the accelerometer | Medium |
| **`BUZ` is a two-pin connector and no wound autotransformer is visible on the board** | Medium-high |

**That last point matters.** Two pins and no visible step-up transformer implies the
siren is a **self-contained module with its own oscillator and boost inside**, and the
board merely switches DC to it. That is precisely the architecture chosen for this
project's v1 — the incumbent does the same thing. See `siren-driver.md`.

### Better photographs needed

1. Macro of the SOP-16 markings, lit at a raking angle — laser marking reads far better
   in oblique light. Identifies the MCU.
2. The reverse side of the board.
3. The buzzer module itself: wire count and any markings.
4. What `SW1` actually is — pairing button, or sensor.
5. Any FCC ID on the case. Manual metadata mentions `OVUSSF32R` and `SF32R09C`; if
   either is an FCC ID, fccid.io hosts the internal photographs and RF test reports the
   manufacturer had to file. That is the closest thing to official teardown
   documentation that exists for a product like this.

---

## 6. Measurement plan

Three sessions. Results belong in `measurements.md`. Doing these on a known-good
commercial device is also good practice with the rig before it matters.

### 6.1 Power profile — highest value

Power analyser (PPK2 / Joulescope) inline with the cell.

- [ ] Average current, disarmed
- [ ] Average current, armed
- [ ] **Capture the receiver sniff pulse train**: period and window width. These are
      field-proven design parameters, not guesses
- [ ] Peak and average current with the siren firing
- [ ] Charging current profile

### 6.2 Acoustic

- [ ] SPL at 1 m, all three volume settings, with a meter — a stated distance, not "loud"
- [ ] Note the siren's resonant frequency if measurable

### 6.3 RF protocol

RTL-SDR plus [rtl_433](https://github.com/merbanan/rtl_433).

- [ ] Confirm 433.92 MHz and OOK/ASK modulation
- [ ] Capture a fob press; determine repetition count and packet timing
- [ ] Identify the encoding. **[UNVERIFIED]** Expected to be EV1527 or PT2262-family
      given the market segment, but not confirmed for this device. rtl_433 ships
      `conf/EV1527-4Button-Universal-Remote.conf`

---

## 7. Security note

If the fob does turn out to be EV1527 or PT2262-family, it is a **fixed code** — a
static ID sent identically every time, with no rolling counter. "Learning code" in these
datasheets refers to the *receiver* learning the transmitter's fixed ID during pairing,
not to any rolling scheme.

The consequence is that a cheap SDR can record and replay a disarm command. Worth
knowing about a device currently being relied on, and worth treating as a
differentiator rather than something to reimplement: Meshtastic provides authenticated,
encrypted commands as standard.

---

## 8. What this device proves, and where it can be beaten

**Proves:** ~110 dB, months of standby, and an instant-feeling remote are all achievable
together in a bike-sized package at a €20 retail price.

**Weaknesses to target:**

| Weakness | This project's answer |
|---|---|
| One-way remote, no confirmation | LoRa is inherently bidirectional |
| Fixed-code RF, replayable **[UNVERIFIED]** | Meshtastic authenticated commands |
| ~30–60 m range | LoRa, plus mesh relaying |
| No position reporting after theft | GNSS |
| No notification unless in earshot | Mesh alerting |

**The genuinely hard part it has already solved:** discrimination. Wind, passing lorries,
someone brushing past, a neighbouring bike falling over. Their "special algorithm" is
the residue of a great many angry reviews. Detection is easy; *not* crying wolf is the
part this project will underestimate.

---

## Sources

- [King-Serry product catalogue](http://king-serry.com/index.php/en/1695454/product/welcome_doorbell/kssf20r.html) — the OEM; KS-SF32R appears in the vehicle anti-theft alarm line
- [NineLeaf KS-SF32R user manual (manuals.plus)](https://manuals.plus/nineleaf/ks-sf32r-vehicle-anti-theft-alarm-pro-vibration-intrusion-detector-manual) — the behavioural spec in §4
- [KS-SF32R manual (device.report)](https://device.report/manual/8000808)
- [KS-SF32R manual (manualspro)](https://manualspro.net/180369-nineleaf-ks-sf32r-vehicle-anti-theft-alarm-pro-vibration-intrusion-detector-user-manual)
- [KS-SF22R manual](https://manualsee.com/blog/MFvgEnoyT.html) — sibling model; explicit about the triaxial accelerometer
- [Wsdcam 113dB bike alarm](https://www.wsdcam.com/products/wsdcam-113db-bike-alarm-wireless-vibration-motion-sensor-waterproof-motorcycle-alarm-with-remote) — close sibling, publishes AAA-based standby figures
- [rtl_433](https://github.com/merbanan/rtl_433) — for §6.3
