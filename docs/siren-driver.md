# Siren Driver

Design reference for the alarm's audible output. Covers how piezo sirens are driven,
why they get loud, and what that means for this project's power budget and — if these
are ever produced in quantity — for manufacturing.

Sources at the end. Compiled 2026-07-27.

---

## 1. Current decision

**v1 reuses a self-contained 12 V piezo siren**, switched by a MOSFET and fed from a
boost converter. Rationale unchanged: the loudness is already proven, and reusing a
finished module is the lowest-risk path. Everything below is context for that choice
and input for v2.

---

## 2. How piezo sirens get loud

Two independent factors, and they multiply.

### Voltage

SPL rises with applied voltage — roughly 40 dB at low voltage up to ~100 dB at 16 Vpp
for a typical part (Diodes AN1176, Fig. 4). This is why every loud piezo circuit has
some form of step-up.

**Free 6 dB:** differential drive (square waves on both terminals, out of phase) puts
twice the peak-to-peak voltage across the element compared with single-ended drive.

### Resonance

SPL is greatest at the element's resonant frequency, and a given buzzer may have
several resonant peaks (AN1176, Fig. 5). Off resonance, output falls off sharply.

**Practical trap:** driving a two-terminal element with a PWM at a guessed frequency
lands off-resonance and is dramatically quieter than the datasheet number. Meshtastic's
`ExternalNotificationModule` PWM/RTTTL path is exactly this trap.

### The lifetime trade-off

Expected lifetime is **inversely proportional to applied voltage** (AN1176, Fig. 6).
The guidance is to pick an element with high SPL at the target frequency and then apply
the *minimum* voltage that achieves it, rather than overdriving a mediocre element.

For an alarm that fires rarely this is probably academic. It matters if the design ever
uses long test tones or frequent pre-alarm chirps.

---

## 3. Drive topologies

### 3.1 Three-pin inductor (autotransformer)

Found inside cheap alarm modules. A tapped inductor with no galvanic isolation — the
primary winding is part of the secondary. Its job is to step up the drive voltage.

Rough comparison: a CMOS push-pull stage on 9 V gives about 18 Vpp across the element.
An autotransformer circuit gives 9 V one way and flyback spikes of perhaps 50 V the
other. Hence much louder.

**Caveats:**
- There is a maximum pulse duration before the inductor saturates; past that it loses
  inductance and behaves like a short. This sets a *minimum* drive frequency.
- It is not an amplifier. Square waves only. It is a loud-noise generator.

### 3.2 Self-oscillating single-transistor circuit

The classic three-terminal-piezo buzzer. One transistor, one ferrite buzzer coil, one
three-terminal piezo. No RC network.

The loop:

1. Voltage applied → transistor conducts → piezo driven across the buzzer coil
2. That grounds the transistor base **through the piezo's centre tap** → transistor off
3. Piezo switches off → base released → transistor recovers
4. Repeat

The centre tap is what sustains oscillation, which is why a three-terminal element is
required. Oscillations dumped into the coil saturate it magnetically; the coil kicks the
stored energy back, magnifying the AC, and that stepped-up AC drives the element.

The topology resembles a crystal oscillator, with the piezo acting as the resonator —
its internal Ls and Cs set the frequency. **It therefore self-tunes to mechanical
resonance**, achieving with one transistor the same benefit as a driver IC's feedback
pin.

**The inductor must be ferrite-wound. Air core does not work.**

### 3.3 Integrated driver ICs

Diodes' PAM89xx family. Relevant comparison:

| | PAM8904E | PAM8906 | PAM8907 |
|---|---|---|---|
| Boost method | Charge pump | Inductive | Inductive |
| Supply | 1.5–5.5 V | 2.1–5.5 V | 1.8–5.5 V |
| Output | 1×/2×/3× supply | 10 / 12 / 18 V variants | 11 or 15.6 V selectable |
| Buzzer type | 2-terminal | 2-terminal **or self-exciting 3-terminal** | 2-terminal |
| Externals | 4 caps | 6 caps, 2 res, 1 inductor, opt. diode | 5 caps, 1 inductor, opt. diode |

Three points matter for this project:

**Shutdown current is under 1 µA**, with thermal shutdown, overcurrent, overvoltage and
undervoltage lockout built in, plus automatic shutdown and wakeup. That solves the
standby problem for the siren branch outright — compare with engineering a true
shutdown path around a generic boost converter.

**Avoid charge-pump drivers.** The PAM8906 and PAM8907 hold a fixed output voltage as
the battery becomes more resistive. The PAM8904E's output is proportional to its input,
so on a battery it gets quieter as the cell drains. **The alarm must be as loud at 20%
charge as at 100%.**

**Self-exciting mode needs only an enable signal** — no PWM, and the element runs at its
own resonant peak. That maps cleanly onto `ExternalNotificationModule`, which already
drives a plain output pin.

Optional extra: a switching diode cuts power consumption by roughly 20%; a 20 V Schottky
rated 1 A is recommended.

---

## 4. Acoustics — the part that is easy to underestimate

**Mounting determines output as much as the drive circuit does.** Incorrectly mounted,
an element is "very insignificant and low in volume" regardless of how well it is
driven.

Requirements from the reference designs:

- Plastic housing with a **centre hole, 6–8 mm diameter** (7 mm typical)
- The element **must not be glued directly to the base of the housing**
- It sits on either:
  - a **soft, pure rubber ring** with a diameter ~30% smaller than the disc, or
  - a **raised platform ~2 mm wide, a couple of mm above the base**, barely supporting
    the circumference edge of the disc

The element needs to be free to flex. Constraining its face or its centre kills the
output.

**Cheap experiment worth doing:** a buzzer coil connected in parallel across an existing
piezo buzzer's leads reportedly increases output substantially. Low effort, testable on
the existing siren.

---

## 5. Implications for this project

### Power

The self-contained siren is internally a switching, inductive circuit. Consequences:

- It draws **spiky pulse current**, not smooth DC, and has inrush at turn-on
- **Size the boost converter for peak current, not average**, and decouple properly
- A loud siren at 12 V may draw several hundred mA. From a 3.7 V cell that is upwards
  of an amp on the input side — relevant to cell selection and wiring
- Feed the siren **directly from the cell via its own boost**, never through the
  WisBlock's 3V3 rail

### Standby

The module has its own quiescent draw whenever powered. Therefore:

- The MOSFET must cut supply **upstream of the boost converter**
- The boost converter must have a **true shutdown pin**, not merely be left unloaded

An unloaded boost converter still burns its own quiescent current. This is the same
class of mistake catalogued in the Long Cricket teardown (`prior-art.md` §5a) — the
converter is exactly the sort of component that quietly sets the floor.

### Firmware

`ExternalNotificationModule` drives an output pin and has nag-cycle repeat logic, which
suits an enable-only siren well. Do **not** use its PWM/RTTTL path to drive a bare
element — see the resonance trap in §2.

---

## 6. If these are produced in quantity

- **The enclosure is the acoustic design.** Hole diameter, standoff geometry and the
  compliance of the mounting ring need prototyping and measuring with an SPL meter. This
  is where a homemade siren loses 20 dB to a commercial one.
- **Commercial bike alarms hit 110–113 dB.** That comes from a high-SPL element in a
  properly designed cavity, not from any generic buzzer driven hard.
- **The PAM8906/8907 route gets more attractive at volume** — one IC replaces a boost
  converter, a MOSFET and a bought-in siren module, with sub-µA shutdown and a smaller
  footprint. The cost is doing the acoustic design yourself.
- **Measure, don't assume.** SPL claims need a meter, at a stated distance, same as
  every current figure in `measurements.md`.

---

## 7. Open work

1. Measure the existing siren module: SPL at 1 m, current draw at 12 V, inrush
2. Test the parallel-buzzer-coil trick on it
3. Select a boost converter with a genuine shutdown pin; measure its quiescent current
   in shutdown
4. Design the MOSFET switch upstream of the boost
5. (v2) Evaluate PAM8906 with a self-exciting three-terminal element
6. (v2, production) Acoustic housing prototyping

---

## Sources

- [Diodes AN1176 — Design Considerations for Driving Piezoelectric Buzzers](https://www.diodes.com/assets/App-Note-Files/Design-Considerations-for-Driving-Piezoelectric-Buzzers.pdf?v=7) — SPL vs voltage and frequency, lifetime vs voltage, driver IC comparison
- [How does the "alarm boost three pin inductor" make a piezo buzzer louder?](https://electronics.stackexchange.com/questions/539843/how-does-the-component-alarm-boost-three-pin-inductor-make-a-piezo-buzzer-loud) — Stack Exchange; the autotransformer explanation
- [Make this Simple Buzzer Circuit with Transistor and Piezo](https://www.homemade-circuits.com/how-to-make-simple-piezo-buzzer-circuit/) — the self-oscillating single-transistor circuit
- [Understanding and Using a Piezo Transducer](https://www.homemade-circuits.com/understanding-and-using-piezo/) — mounting geometry
- [Simplest Piezo Driver Circuit Explained](https://www.homemade-circuits.com/simplest-piezo-driver-circuit-explained/) — the parallel buzzer-coil trick; ferrite requirement
