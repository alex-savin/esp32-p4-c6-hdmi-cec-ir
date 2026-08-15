# Schematic rework list — for the board designer

Against `SCH_Schematic1_2026-08-12.pdf`. Compiled 2026-08-14; **items 17–18
added 2026-08-15** after a re-verification pass. They are numbered out of
sequence deliberately, so references to items 1–16 stay valid.

Everything here must be resolved **before fabrication**. Each item states the
change; the reasoning is in [Improvements.md](Improvements.md), referenced by
section. Pin numbers are ESP32-P4 package pins, taken from the drawing.

Nothing in this list requires a redesign. Six items are wiring or net-label
errors — **two of them (17, 18) leave the board dead or unrecoverable as
drawn** — several are one-component changes, and the rest are checks.

---

## Summary

| # | Change | Severity | Cost |
| --- | --- | --- | --- |
| 17 | Rename pin-79 net `EN` → **`EN-DCDC`** | 🔴 Blocking — board is dead as drawn | none |
| 18 | Connect the **recovery-header UART** (3 labels) | 🔴 Blocking for recovery | none |
| 1 | Move boot strap from GPIO0 to **GPIO35** | 🔴 Blocking | none |
| 2 | HDMI connector → **Type A**, remap pins 13/14/17 | 🔴 Blocking | connector swap |
| 3 | `C9`/`C10` — shunt to GND, not in series | 🔴 Blocking | none |
| 4 | Specify the **nRF24 antenna** | 🔴 Blocking | 1 part |
| 5 | Move nRF24 `IRQ` off **GPIO36** | 🟠 High | none |
| 6 | Local decoupling on the **1.1 V core rail** | 🟠 High | ~5 parts |
| 7 | HPD divider — raise to ≥2.4 V | 🟠 High | 1 value |
| 8 | **IR jack insertion detect** — 4 GPIOs + switched jacks | 🟠 High | jack choice |
| 9 | Flash **4 MB → 8 MB** | 🟡 Cheap now | same footprint |
| 10 | 1 kΩ series resistors on IR RX | 🟡 Cheap now | 2 parts |
| 11 | Debounce caps 1 µF → 100 nF | 🟡 Cheap now | none |
| 12 | 5th pin (`EN`) on the C6 recovery header | 🟡 Cheap now | header swap |
| 13 | Verify **PCA9306 pin 1/2** | 🔵 Check | — |
| 14 | Verify the **3.5 mm jack** datasheet | 🔵 Check | — |
| 15 | Decide the **C6 antenna variant** | 🔵 Decide | — |
| 16 | Annotate + tidy the sheet | ⚪ Housekeeping | none |

---

## 🔴 Blocking

### 17. The core regulator's enable is on the wrong net — the board is dead as drawn

*Improvements §2.9. Added 2026-08-15.*

P4 pin 79 (`EN_DCDC`) carries net label **`EN`** — the reset-button net — while
`U15`'s enable carries **`EN-DCDC`**, which appears nowhere else. Two failures
at once: the TLV62569's enable **floats** (no 1.1 V core rail → the P4 never
boots), and the P4's regulator-enable output sits on its own reset line.

**Change:** rename the label at **pin 79** from `EN` to `EN-DCDC`. One label
fixes both. Verify afterwards that net `EN` contains only `CHIP_PU` (103),
`R24`, `C14`, `Q1` and the button, and that `EN-DCDC` contains exactly pin 79
and `U15` pin 1.

### 18. The C6 recovery UART is not connected

*Improvements §2.10. Added 2026-08-15.*

The recovery header's `TXD0`/`RXT0` labels match nothing — the C6 module's
UART pins (31/30) end in dangling stubs with no labels at all. The header's
entire purpose is flashing a bricked C6, and as drawn neither direction works.
`BOOT` and GND are fine.

**Change:** add labels **`TXD0`** to module pin 31 and **`RXD0`** to pin 30;
relabel header pin 2 from `RXT0` to **`RXD0`**. (This absorbs the old item 16
spelling fix.)

### 1. Boot strap is on the wrong pin — the P4 cannot be flashed

*Improvements §2.8.*

The `BOOT` button and the auto-download transistor `Q2` both drive **GPIO0**.
GPIO0 is the boot strap on the ESP32 and ESP32-S3. **On the ESP32-P4 it is not a
strapping pin at all** — the download strap is **GPIO35**, which this board
leaves unrouted.

As drawn the chip always boots to flash and can never enter serial download
mode. With the USB PHY unconnected there is no USB-JTAG fallback either, so the
application processor would be unflashable.

**Change:** move the `IO0` net from **pin 104 (GPIO0)** to **pin 66 (GPIO35)**.
The button, `R25`, `C15` and `Q2` all follow it unchanged — only the pin moves.
Leave GPIO0 unconnected. Consider renaming the net `IO35` or `STRAP_BOOT` so the
next reader is not misled.

The transistor circuit itself is correct and needs no change.

### 2. HDMI connector must become full-size Type A

*Improvements §2.1, BOM §3.*

`HDMI-519` (`C136421`) is a **Mini-HDMI Type C** receptacle. The decision is
full-size **Type A**. Type C is not Type A renumbered — the two families assign
pins 13, 14 and 17 differently — so the connector swap forces a remap.

| Net | Currently (Type C) | Must become (Type A) |
| --- | --- | --- |
| `CEC` | pin 14 | **pin 13** |
| GND | pin 13 | **pin 17** |
| *(no-connect)* | pin 17 | **pin 14** — Utility/HEAC+ |
| `HDMI-SCL` | pin 15 | pin 15 — unchanged |
| `HDMI-SDA` | pin 16 | pin 16 — unchanged |
| `HDMI-5V` | pin 18 | pin 18 — unchanged |
| `HDMI-HPD` | pin 19 | pin 19 — unchanged |

Getting 13 and 17 the wrong way round shorts CEC to ground, and the symptom is
simply that nothing on the bus ever answers.

Candidate parts (19-pin Type A female, right-angle SMD): `C2858275`,
`C2906135`, `C5184845`. Replace symbol **and** footprint.

**While in there:** tie the TMDS shield pins (Type A: 2, 5, 8, 11) and the shell
to GND. They are all no-connect today. No TMDS signal is used, but the shield
return still matters for EMI.

### 3. `C9` / `C10` are wired in series instead of shunting to ground

*Improvements §2.2.*

In both IR receive circuits the 100 nF capacitor sits **in series between +3V3
and the jack's pin 4**, with wire segments above and below its plates. Pin 3 is
GND and pin 2 is the signal; pin 4 is meant to be the receiver's supply.

As drawn, pin 4 has no DC path — **an IR receiver plugged into either zone gets
no power and will never work.**

**Change:** connect jack pin 4 directly to `+3V3`, and place `C9`/`C10` from
`+3V3` to GND as the decoupling capacitors they were meant to be.

### 4. The nRF24 antenna is not specified

*Improvements §2.7.*

The RF chain `ANT1`/`ANT2` → `L2`/`L3`/`L4` → `C22` terminates at a net label
`50Ohm` with **nothing attached** — no connector, no chip antenna, no component.

`U12` is required for Harmony remote support (Improvements §5), so this is not
optional.

**Change:** choose one and put it on the schematic —

* **U.FL / IPEX connector** — easiest to tune, needs a pigtail and an antenna.
* **Chip antenna** — one part, needs a matching network and a keep-out.
* **PCB trace antenna** — free, but must be stated on the sheet with a defined
  impedance and keep-out, or the layout will not reproduce.

Note this interacts with item 15: if the C6 also goes external-antenna, both
radios want connectors, and separating them physically is the main defence
against the two 2.4 GHz receivers desensing each other.

---

## 🟠 High priority

### 5. nRF24 `IRQ` is on a strapping pin

*Improvements §2.8.*

`IRQ` is on **GPIO36 (pin 68)**. GPIO36 must be **high** at reset to enter the
serial bootloader reliably, and `IRQ` is active-low — the radio pulls it down
whenever it has a pending interrupt.

**Change:** move `IRQ` to **pin 80 (GPIO39)**. Leave GPIO36 unconnected.

Optionally also move `CE` from **pin 65 (GPIO34)** — another strapping pin — to
**pin 81 (GPIO40)**. Lower priority: `CE` is an output under our control and is
only sampled at reset, so it is a robustness improvement rather than a fault.

*Firmware follows:* `board_pins.h` must be updated in lockstep.

### 6. The 1.1 V core rail has no local decoupling

*Improvements §2.4.*

All twelve 100 nF capacitors (`C32`–`C43`) sit in one bank across **+3V3**. The
`VDD-HP` rail reaches the P4's four core pins (26, 54, 76, 91) with nothing but
the regulator's own 22 µF `C31` at the far end of the board.

A 1.1 V core rail on a dual-core part draws fast, large transients. Without
local high-frequency capacitance the rail sags at each load step, and the
symptom — sporadic resets or PSRAM corruption under load — is very hard to
diagnose after fabrication.

**Change:** move roughly half the bank to `VDD-HP`, one 100 nF per core pin,
placed at the pin in layout. Add a 10 µF bulk capacitor local to the P4. Follow
Espressif's ESP32-P4 hardware design guidelines for the reference arrangement.

### 7. HPD is asserted below the HDMI minimum

*Improvements §2.5.*

`R16` 10 kΩ / `R17` 6.8 kΩ from `HDMI-5V` gives **2.02 V**. HDMI requires a
source to see **≥ 2.4 V** on Hot Plug Detect. It is also below a comfortable
V<sub>IH</sub> for the 3.3 V GPIO that shares the node.

**Change:** `R16` **10 kΩ → 4.7 kΩ**, keeping `R17` at 6.8 kΩ. That gives
2.96 V — above the HDMI minimum, still inside the P4's tolerance with `D5`
clamping.

### 8. IR jack insertion detect

*Improvements §6.*

There is currently no way to tell whether an IR receiver is plugged in. The RX
signal line has a 10 kΩ pull-up, so an empty jack and a plugged-in-but-idle
receiver sit at the same voltage — no firmware probe separates them.

**Change:** bring each jack's insertion-switch contact to a GPIO.

| Jack | Function | GPIO | Package pin |
| --- | --- | --- | --- |
| `U8` | Zone 1 TX detect | 42 | 83 |
| `U9` | Zone 1 RX detect | 43 | 84 |
| `U10` | Zone 2 TX detect | 44 | 86 |
| `U11` | Zone 2 RX detect | 45 | 87 |

Preferred circuit, for a jack with an **independent** SPST switch contact:

```
   3V3 ──[10k]──┬──[1k]──► GPIO 4x
                │
   jack switch ─┴────────► GND      (closed when the jack is EMPTY)
```

Empty reads LOW, inserted reads HIGH. The 10 kΩ is optional — the P4's internal
pull-up suffices — but is worth fitting on a connector-facing net. The 1 kΩ is
ESD series protection, matching item 10.

If only **tip-normalled** switch jacks are available, the RX jacks work with no
extra parts at all (the existing `R19`/`R22` pull-up plus the P4's internal
pull-down form the divider), but the polarity inverts and the **TX jacks need a
2 × 100 kΩ divider** because their tip sits at `+VBUS`. See Improvements §6.

If only two detect lines can be afforded, fit the **RX pair (43, 45)**.

---

## 🟡 Cheap now, impossible later

### 9. Flash 4 MB → 8 MB

*BOM §8.* `U13` W25Q32JVSSIQ → **W25Q64JVSSIQ**. Same SOIC-8 208-mil package,
same pinout, a few cents. If the on-device IR-classifier model is pursued
([datasets.md §3](../../docs/datasets.md)), this stops being optional — a model
needs a flash partition, and repartitioning after fabrication needs a serial
cable. The P4 must hold the hosted stack, a BLE host, CEC, IR,
Harmony RF and a web surface, **plus two OTA slots** — which doubles the app
footprint. 4 MB is survivable with no headroom; 8 MB removes the question.

### 10. Series protection on the IR RX lines

*Improvements §3.2.* The RX node runs from the jack straight to the GPIO with
only the 10 kΩ pull-up. Add a **1 kΩ series resistor** between jack and GPIO on
each zone, to survive ESD and momentary shorts during hot-plugging.

### 11. Debounce capacitors are ten times too large

*Improvements §3.3.* `C14`/`C15`/`C16` are 1 µF against 10 kΩ pull-ups —
τ = 10 ms, sluggish, and on `EN`/the boot strap it fights the auto-download
circuit's timing. **Change to 100 nF** (τ = 1 ms).

### 12. No `EN` on the C6 recovery header

*Improvements §3.5.* A manual recovery flash of the C6 needs a reset, and the
4-pin header does not offer one — it depends on the P4 driving GPIO54, which is
exactly what may be broken when recovery is needed. **Add a fifth pin** carrying
the C6 `EN` net.

---

## 🔵 Check before ordering

### 13. PCA9306 pin 1 / pin 2

*Improvements §2.6.* The `U7` symbol places `GND` on pin 1 and `VREF1` on pin 2.
TI's datasheet has these the other way round. If the symbol is wrong, `VREF1` is
tied to ground and the 3.3 V rail is shorted through the part's ground pin.
**Verify against the TI datasheet for the DCU package.**

### 14. The 3.5 mm jack datasheet

*BOM §4.* Two things, not one:

1. Contact numbering — tip/ring/sleeve must land on 2/3/4 as the `PJ-320A`
   symbol assumes. A swapped tip/sleeve puts `+VBUS` into the receiver's output.
2. **An insertion-switch contact must exist**, and the datasheet must say
   whether it is an independent SPST or normalled to the tip — the two need
   opposite firmware polarity and different external parts (item 8).

`C18167` is unverified on both counts.

### 15. C6 antenna variant

*BOM §2.* `ESP32-C6-MINI-1U-H4` has **no PCB antenna** — it needs an external
antenna on an **IPEX-3 / MHF3** connector (not the common U.FL). The non-`U`
part drops into the same 53-pad footprint with nothing else to buy.

This is a genuine trade-off, not a default: two 2.4 GHz receivers now run
concurrently with no coexistence arbitration between them, and antenna
separation is the main lever against mutual desense. **Decide against the
enclosure.**

---

## ⚪ Housekeeping

### 16. Sheet annotations

* **State the HDMI connector family on the drawing.** Nothing on the sheet says
  Type A or Type C. That omission caused a review to raise, withdraw, and then
  re-raise the same finding — see Improvements §2.1.
* ~~Rename `RXT0` → `RXD0`~~ — absorbed into item 18, which reconnects the
  whole recovery UART.
* `CP2102N`: `VREGIN` is on `+VBUS` while `VDD` is on the externally regulated
  `+3V3`, running the bridge's internal LDO in parallel with the `SY8089` buck.
  Both derive from VBUS and both target 3.3 V so it will most likely work, but it
  is not a configuration the datasheet describes. Consider tying `VREGIN` to
  `VDD`. *(Improvements §3.4.)*

---

## Net and pin changes at a glance

| Net | From | To | Item |
| --- | --- | --- | --- |
| net label at pin 79 | `EN` (collides with reset) | **`EN-DCDC`** | 17 |
| C6 module pins 31/30 | *no labels* | **`TXD0` / `RXD0`** | 18 |
| Recovery header pin 2 | `RXT0` | **`RXD0`** | 18 |
| `IO0` (boot strap) | pin 104 / GPIO0 | **pin 66 / GPIO35** | 1 |
| `IRQ` (nRF24) | pin 68 / GPIO36 | **pin 80 / GPIO39** | 5 |
| `CE` (nRF24, optional) | pin 65 / GPIO34 | **pin 81 / GPIO40** | 5 |
| `CEC` (connector end) | HDMI pin 14 | **HDMI pin 13** | 2 |
| GND (connector end) | HDMI pin 13 | **HDMI pin 17** | 2 |
| IR jack pin 4 (both RX) | via `C9`/`C10` | **direct to +3V3** | 3 |
| *new* — 4 × jack detect | — | **GPIO 42, 43, 44, 45** | 8 |

## Part changes at a glance

| Ref | From | To | Item |
| --- | --- | --- | --- |
| `AUDIO1` | HDMI-519, Mini Type C | **Type A, 19-pin R/A SMD** | 2 |
| `U13` | W25Q32JVSSIQ (4 MB) | **W25Q64JVSSIQ (8 MB)** | 9 |
| `R16` | 10 kΩ | **4.7 kΩ** | 7 |
| `C14`, `C15`, `C16` | 1 µF | **100 nF** | 11 |
| `U8`–`U11` | PJ-320A-3P | **jack with switch contact** | 8, 14 |
| `RECOVER-HEADER` | 4-pin JST-XH | **5-pin JST-XH** | 12 |
| *new* | — | nRF24 antenna | 4 |
| *new* | — | 4 × 100 nF + 1 × 10 µF on `VDD-HP` | 6 |
| *new* | — | 2 × 1 kΩ (IR RX series) | 10 |
| *new* | — | up to 4 × 10 kΩ + 4 × 1 kΩ (jack detect) | 8 |
| `U12` | — | **genuine Nordic silicon only** — BOM §7 | — |

---

## Sign-off checklist

- [ ] 1 — `IO0` moved to GPIO35, GPIO0 left NC
- [ ] 2 — Type A connector fitted, pins 13/14/17 remapped, shields grounded
- [ ] 3 — `C9`/`C10` shunt to GND, jack pin 4 on `+3V3`
- [ ] 4 — nRF24 antenna on the schematic
- [ ] 5 — `IRQ` off GPIO36
- [ ] 6 — `VDD-HP` decoupling added
- [ ] 7 — `R16` = 4.7 kΩ
- [ ] 8 — Detect on GPIO 42–45, switched jacks specified
- [ ] 9 — `U13` = W25Q64
- [ ] 10 — 1 kΩ series on IR RX
- [ ] 11 — `C14`–`C16` = 100 nF
- [ ] 12 — 5-pin recovery header
- [ ] 13 — PCA9306 pinout verified
- [ ] 14 — Jack datasheet verified (numbering **and** switch)
- [ ] 15 — C6 antenna variant chosen
- [ ] 16 — Sheet annotated, nets renamed
- [ ] 17 — Pin-79 net renamed `EN-DCDC`; net membership verified both sides
- [ ] 18 — Recovery UART labelled and connected end to end
