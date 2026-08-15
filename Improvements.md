# Schematic Audit — `SCH_Schematic1_2026-08-15.pdf`

Forensic audit of the project schematic. First performed against rev
2026-08-12 (rendered at 400 dpi and read block by block, net labels verified by
instance-counting the PDF text layer); **re-audited 2026-08-15 against the new
revision**, which responded to part of the first audit. Findings below carry a
dated **Status** line where the new revision changed them.

Overall the schematic captures the dual-MCU architecture, both power rails and
the HDMI protocol requirements correctly. The substantive open items are **three
blocking wiring errors** (§2.1, §2.2, §2.8), a decoupling gap that will be
expensive to discover after fabrication (§2.4), a new regression (§2.11), and a
set of smaller items.

> **For the board designer:** [Rework.md](Rework.md) is the consolidated,
> actionable change list. This document is the reasoning behind it.

> **Confidence.** Findings marked **[confirmed]** were read directly off the
> drawing and are not in doubt. Findings marked **[verify]** depend on a
> datasheet or footprint I could not inspect, and state exactly what to check.

---

## 0. Revision 2026-08-15 — what changed

**Fixed:** the pin-79 `EN-DCDC` net (§2.9), the recovery UART (§2.10), the
nRF24 antenna declaration (§2.7 — "A PCB Trace antenna will be used here", full
matching network retained). The PCA9306 question (§2.6) is closed from the TI
datasheet: the symbol was right, and the new `R42` 200 kΩ bias completes TI's
reference circuit.

**Changed but still broken:** the IR receiver supply (§2.2) — the signal moved
from jack pin 2 to pin 4, but the series capacitor moved with the rewire and no
contact carries DC. Still blocking.

**Redesigned:** HPD (§2.5) — the board no longer asserts HPD at all; it senses
it through a new `R60`/`R59` divider into GPIO25. Safe levels, but the design
intent needs confirming.

**Regressed:** `D3`, the CEC ESD clamp, was deleted with no replacement —
**§2.11**, new. Also `C56` 1 µF landed in parallel with `C14` on the reset net
(§3.3), and the status-LED resistors went 100 Ω → 2.2 kΩ (~0.6 mA — confirm
that is a choice).

**Unchanged and still open:** boot strap on GPIO0 (§2.8), Type C connector
wiring (§2.1), core-rail decoupling (§2.4), nRF24 `IRQ` on GPIO36 (§2.8),
jack detect (§6), flash size, 1 kΩ RX series, debounce values, 4-pin header.

**Unsolicited improvements worth keeping:** `R41` C6-BOOT pull-up, `R61` on
`RESET-CTR`, `C58` 10 µF on the C6 rail, `R44`/`R45` IR TX base pull-downs,
`R51`–`R53` flash pull-ups, `R43` on CP2102N `RSTb`, `C59`/`C60` feed-forward
on the SY8089, HDMI shell EPs grounded, the auto-program truth table, and four
M2 mounting holes.

---

## 1. Subsystem-by-Subsystem Audit

### Power Infrastructure (3V3 & 1V1 Core Rails)

* **3.3 V main buck (`U1` — SY8089AAC):** converts USB `+VBUS` (5 V) to `+3V3`.
  Feedback with $R_6 = 91\text{ k}\Omega$ and $R_7 = 20\text{ k}\Omega$ gives
  $V_{\text{OUT}} = 0.6\text{ V} \times \left(1 + \frac{91}{20}\right) = 3.33\text{ V}$.
  2.2 µH inductor `L1`, 22 µF `C3`/`C4` out. Correct. *(Rev 08-15 adds
  `C59`/`C60` 22 pF feed-forward across the divider — good for transient
  response.)*

* **1.1 V P4 core buck (`U15` — TLV62569DBVR):** the ESP32-P4 needs an external
  1.1 V supply for its high-performance digital core (`VDD-HP`). Feedback with
  $R_{39} = 8.2\text{ k}\Omega$ and $R_{40} = 10\text{ k}\Omega$ gives
  $V_{\text{OUT}} = 0.6\text{ V} \times \left(1 + \frac{8.2}{10}\right) = 1.092\text{ V}$.
  2.2 µH `L5`, 22 µF `C31`, 22 pF feed-forward `C30`. Correct.
  The intent is for the P4 to drive both the enable (via `EN-DCDC`) **and** the
  feedback node (via `FB-DCDC`), putting the core voltage under the chip's own
  control — the standard ESP32-P4 arrangement.
  **Status 2026-08-15: the enable is now correctly connected** — `EN-DCDC`
  verified at both pin 79 and `U15` (§2.9). The feedback half was always right.
  *(An earlier revision of this bullet stated the enable was connected when it
  was not; that was read at too low a zoom — §4.)*

* **USB power entry:** `USBC1` ties `CC1`/`CC2` through independent 5.1 kΩ
  pull-downs (`R1`, `R2`) for 5 V sink advertisement. `USBLC6-2SC6` (`D1`)
  protects the data lines.

### Main Processor (`U3` — ESP32-P4NRW32X)

* **SDIO interconnect:** 4-bit bus on GPIO 14–19. **Six** 51 kΩ pull-ups are
  fitted, `R31`–`R36` — one each on `DAT0`–`DAT3`, `CMD` **and `CLK`**. Only
  five are required; the one on `CLK` is harmless.
  *(Previously recorded as "five, `R31`–`R35`" — miscounted.)*

* **C6 reset line:** GPIO 54 (`RESET-CTR`) drives the C6 `EN` pin directly, for
  hardware reset and OTA state transitions.

* **Decoupling:** twelve 100 nF capacitors, `C32`–`C43`. **All twelve sit in one
  bank across `+3V3` and GND** — they are not distributed per power pin, and the
  1.1 V `VDD-HP` rail gets none of them. See §2.4.
  *(Previously recorded as "distributed across the `+3V3` and `VDD-HP` power
  pins" — that is not what the drawing shows.)*
  Rail-local caps that are present: `VDD-PSRAM` `C51`–`C55`, `VDD-FLASH`
  `C49`/`C50`.

* **Clock:** `U14`, 40 MHz, 10 pF loads `C27`/`C28`.

* **Memory:** 32 MB in-package PSRAM (the `NRW32X` suffix) plus `U13`
  W25Q32JVSSIQ, 4 MB, on the dedicated flash bank. *(Rev 08-15 adds
  `R51`–`R53` 10 kΩ pull-ups on CS/WP/HOLD — correct practice. The 8 MB
  upgrade, Rework 9, is still outstanding.)*

* **No native USB.** `USB_DM`/`USB_DP`/`VDD_USBPHY` are unconnected; USB-C data
  reaches the CP2102N only. There is consequently **no USB-JTAG debugging and no
  USB DFU** — serial over the CP2102N is the only flash and debug path. Worth a
  conscious decision rather than an accident, given the P4's USB-Serial-JTAG is
  normally the primary bring-up tool.
  **Status 2026-08-15: DECIDED — adopt USB-Serial-JTAG, delete the CP2102N.**
  The USB-C data pair moves to GPIO24/25 (the P4's USB-Serial-JTAG, in ROM:
  flashing on a blank chip, console, and OpenOCD/gdb debugging over one
  cable); DDC SCL and the HPD tap move to GPIO46/47 to free those pins;
  `VDD_USBPHY` gets powered and the OTG HS pins go to test pads as a DFU
  provision. Full change list in **Rework 20**. This also retires §3.4.

### Wireless Co-Processor (`U5` — ESP32-C6)

* **SDIO mapping:** the C6's fixed slave pins, GPIO 18–23, to P4 GPIO 19, 18,
  14, 15, 16, 17 respectively.

* **Recovery header (`B4B-XH-A`):** exposes `TXD0` (C6 GPIO 16), `RXD0`
  (C6 GPIO 17, net still spelled `RXT0`), `BOOT` (GPIO 9, 10 kΩ pull-up `R41`)
  and GND, for flashing the C6 independently.
  **Status 2026-08-15: the UART is now connected end to end** (§2.10).
  **`EN` is still not on the header**, so a manual recovery flash needs the P4
  to toggle GPIO 54 or a power cycle. Consider a fifth pin. Rev 08-15 also adds
  `R61` 10 kΩ pull-up on `RESET-CTR` and `C58` 10 µF on the module's 3V3 —
  both good.

### HDMI Interface (CEC 2.0 & E-DDC)

* **Role:** all twelve TMDS pins are explicitly no-connect. **Since rev
  08-15 the board no longer asserts HPD — it senses it** (§2.5), which is the
  behaviour of a CEC tap plugged into a TV or AVR rather than of a sink
  emulating a display to a source. The distinction is now a design decision to
  confirm, not a fact of the drawing. *(This bullet previously said "the board
  asserts HPD itself, so this is an HDMI sink" — true of rev 08-12 only.)*

* **Level translation (`U7` — PCA9306DCUR):** `SCL2`/`SDA2` face the 5 V HDMI
  side with 4.7 kΩ pull-ups `R13`/`R12` to `HDMI-5V`. `SCL1`/`SDA1` face the
  3.3 V side with 4.7 kΩ pull-ups `R14`/`R15`, and `VREF1` to `+3V3`.
  **Status 2026-08-15:** `EN`/`VREF2` now biased to `+3V3` through the new
  `R42` 200 kΩ — TI's reference arrangement. Pinout verified against the
  datasheet, §2.6: the symbol is correct.
  *(Previously recorded with the two sides swapped — `R12`/`R13` are the 5 V
  pull-ups, `R14`/`R15` the 3.3 V ones.)*

* **CEC physical layer:** 27 kΩ pull-up `R11` to `+3V3`, `D2` BAT54 Schottky in
  series. Matches what the `hdmi_cec` component documents as required.
  **Status 2026-08-15: the `D3` AZ5123-01F ESD clamp was deleted — see §2.11,
  a regression.** See §2.1 for where CEC lands on the connector.

* **Hot Plug Detect:** redesigned in rev 08-15 — sensing divider `R60` 5.1 kΩ /
  `R59` 10 kΩ from `HDMI-HPD` to GND, tap `HDMI-HPD-ESP` → GPIO25. `D4`
  AZC199-04S array (shared with SDA/SCL/+5 V) still clamps the connector side;
  the old `R16`/`R17` assert-divider and the `D5` PESD clamp are gone. See
  §2.5.

### nRF24L01+ 2.4 GHz Transceiver (`U12`)

* **Control lines:** GPIO 30–34 (`MOSI`, `MISO`, `SCK`, `CSN`, `CE`) and
  **GPIO 36** (`IRQ`).
  *(Previously recorded as GPIO 37 for `IRQ`; GPIO 37 is the console UART TX.)*

* **RF matching network:** `L2` 2.7 nH, `L3` 8.2 nH, `L4` 3.9 nH with `C20`
  2.2 nF, `C21` 4.7 pF, `C22` 1.5 pF, `C23` 1 pF into a 50 Ω port. `R29` 22 kΩ
  on `IREF`, `C19` 33 nF on `DVDD`, 16 MHz crystal `X1` with 22 pF `C24`/`C25`
  and 1 MΩ `R30`. All standard for the bare `NRF24L01P-R` die.

* **The antenna is not specified** — see §2.7 — and the choice of this part at
  all is worth revisiting before layout: see §5.

### CP2102N USB-to-UART Bridge (`U6`)

* **VBUS sensing:** the `R8` 21 kΩ / `R9` 47.5 kΩ divider feeds the **`VBUS`
  sense pin (8)** only, at 3.47 V, limiting leakage per the on-sheet note.
  `VREGIN` (pin 7) is tied straight to `+VBUS`.
  *(Previously described as a divider "on `VREGIN`/`VBUS`" — only pin 8 is
  divided.)*

* **Auto-download circuit:** standard cross-coupled NPN pair (`Q1`, `Q2` S8050,
  10 kΩ `R10`/`R5`) driven by `DTR`/`RTS` onto `EN` and `IO0`, with the truth
  table printed on the sheet. Correct.

### Switches & User LEDs

* **Tactile buttons:** `EN` (reset), `BOOT` (GPIO 0) and `FSW` (**GPIO 41**),
  each with a 10 kΩ pull-up and a 1 µF debounce capacitor. Rev 08-15 renames
  the `EN` pull-up `R24` → `R46` (relocated to the auto-program block) and adds
  `C56` 1 µF in parallel with `C14` on `EN` — see §3.3.
  *(Previously recorded as GPIO 38 for `FSW`; GPIO 38 is the console UART RX.)*

* **Status LEDs:** blue on GPIO 20, green on GPIO 21, both active high.
  Rev 08-15 changes the resistors 100 Ω (`R27`/`R28`) → **2.2 kΩ**
  (`R48`/`R49`), i.e. ~0.6 mA drive — visible on modern 0805 parts but dim;
  flagged in Rework 16 to confirm it is intentional.

---

## 2. Blocking and High-Priority Findings

### 2.1 Connector changed to full-size Type A — pins 13/14/17 must be remapped — **[confirmed, blocking]**

This finding has a history worth keeping, because the conclusion flipped twice.

**First raised** as "the symbol's pin numbers don't match the HDMI standard,
likely blocking". **Then withdrawn**: `HDMI-519` (`C136421`) turned out to be a
**Mini-HDMI Type C** receptacle, and Type C is not Type A renumbered — the spec
reassigns the pins, moving DDC/CEC Ground to 13, CEC to 14 and Reserved to 17,
and swapping each pair's positive signal with its shield. The symbol matched
Type C exactly. The schematic was right; the audit had compared it against the
wrong connector family.

**Now live again, in a different form.** The decision on 2026-08-14 is
**full-size Type A only** (§3.8), and the current wiring is Type C wiring. As
drawn, on a Type A footprint, the CEC signal would land on pad 14
(Utility/HEAC+) and the real CEC pad 13 would be tied to ground — the original
failure, arrived at from the opposite direction.

Required remap when the connector is swapped:

| Net | Now (Type C) | Must become (Type A) |
| --- | --- | --- |
| `CEC` | pin 14 | **pin 13** |
| GND | pin 13 | **pin 17** |
| *(no-connect)* | pin 17 | **pin 14** — Utility/HEAC+ |
| `HDMI-SCL` | pin 15 | pin 15 — unchanged |
| `HDMI-SDA` | pin 16 | pin 16 — unchanged |
| `HDMI-5V` | pin 18 | pin 18 — unchanged |
| `HDMI-HPD` | pin 19 | pin 19 — unchanged |
| TMDS pairs | shields odd, signals even | **signals odd, shields even** — all no-connect either way |

**Action:** replace `AUDIO1` with a Type A symbol **and** footprint, then
re-verify these seven nets. The four signals that carry traffic (SCL, SDA, +5 V,
HPD) keep their numbers, so it is only 13/14/17 plus the connector body that
move — but getting 13/17 backwards is silent and fatal.

**While you are in there:** the TMDS shields are currently all no-connect. On a
Type A part it is worth tying the four shield pins (2, 5, 8, 11) and the shell
to GND for EMI, even though no TMDS signal is used.

**And annotate the sheet with the connector family.** Nothing on the drawing
says Type A or Type C, which is precisely how this finding went wrong twice.

**Status 2026-08-15: still open.** The connector is still `HDMI-519` with Type
C mapping (re-verified pin by pin: 13 = DDC/CEC Ground → GND, 14 = CEC,
17 = Utility NC). One piece of progress: the shell EP pads (20–23) are now
grounded — keep that through the swap.

### 2.2 The IR receiver has no DC supply — **[confirmed, blocking — survived a rework]**

**As of rev 2026-08-12:** the 100 nF capacitor sat in series between `+3V3`
and jack pin 4; pin 2 was the signal (10 kΩ pull-up `R19`/`R22`), pin 3 GND.
Pin 4 — the receiver's supply — floated behind a capacitor.

**Status 2026-08-15: re-wired, and still broken.** The revision moved the
signal (`RX_IR_JACK_x` + pull-up) from pin 2 to **pin 4**, kept pin 3 on GND,
and connected **pin 2 to `+3V3` through the same series 100 nF** (`C9`/`C10`,
verified at 600 dpi on both zones — no junction between the capacitor's bottom
plate and any DC net). The defect moved to the other contact: a three-wire IR
receiver (VCC/GND/OUT) still has **no contact that carries DC power**, in
either zone. No pull-up parasitic can substitute — a TSOP-style receiver draws
0.4–1.5 mA.

**Action:** wire jack pin 2 **directly to `+3V3`** and hang `C9`/`C10` from
that node **to GND**. Three contacts: pin 2 = VCC, pin 3 = GND, pin 4 = signal
with pull-up. Then verify against the jack datasheet (BOM §4) that the plug's
tip/ring/sleeve land on those pins in that order.

*(Kept from 08-15: `R44`/`R45` 10 kΩ base pull-downs on the TX drivers — the
IR LEDs no longer fire while the P4's GPIOs float during boot.)*

### 2.3 Firmware targets the wrong CEC GPIO — **[confirmed]**

`esp32-p4-firmware/main/main.cpp` sets `CEC_GPIO = GPIO_NUM_14`. On this board
**GPIO 14 is SDIO `DAT0`** to the co-processor. Driving it open-drain, as the
`hdmi_cec` component does, would break the WiFi link.

The CEC net is on **GPIO 22**. The source comment already says "Adjust to match
the board", so this is a known placeholder — but nothing in the repository
records the correct number, and the mistake is silent (the radio simply stops
working, which reads as a driver bug).

**Status: FIXED, 2026-08-14.** `main.cpp` now takes the pin from
`board::HDMI_CEC` in the new `esp32-p4-firmware/main/board_pins.h`, which
carries the whole P4 map — including the SDIO pins, labelled so nothing claims
them by accident. Builds clean against ESP-IDF v6.0.2.

### 2.4 No local decoupling on the 1.1 V core rail — **[confirmed]**

All twelve 100 nF capacitors (`C32`–`C43`) are on `+3V3`. The `VDD-HP` rail
reaches the P4's four core pins (26, 54, 76, 91) with nothing but the
regulator's own 22 µF `C31` at the other end of the board.

A 1.1 V core rail on a 400 MHz dual-core part draws fast, large transients.
Without local high-frequency capacitance the rail will sag at each load step,
and the symptom — sporadic resets or PSRAM corruption under load — is
notoriously hard to diagnose after fabrication.

**Action:** move roughly half the bank to `VDD-HP` (100 nF per core pin, placed
at the pin in layout) and add a 10 µF bulk capacitor local to the P4. Espressif's
ESP32-P4 hardware design guidelines give the reference arrangement.

### 2.5 HPD — redesigned in rev 2026-08-15; levels now safe, intent needs confirming — **[confirmed, decision]**

**As of rev 2026-08-12** the board *asserted* HPD from `HDMI-5V` through an
`R16` 10 kΩ / `R17` 6.8 kΩ divider — at 2.02 V, below the 2.4 V a source must
see, and marginal for the GPIO sharing the node.

**Status 2026-08-15: that circuit is gone, replaced by a sensing divider.**
`HDMI-HPD` → `R60` 5.1 kΩ → tap `HDMI-HPD-ESP` (GPIO25, package pin 53) →
`R59` 10 kΩ → GND. A 5 V HPD arrives at the GPIO as
$5\text{ V} \times \frac{10}{15.1} = 3.31\text{ V}$ — inside the P4's
tolerance, so `D5`'s deletion is acceptable here (`D4` still clamps the
connector side; the divider bounds the GPIO node).

**What was lost:** the board can no longer assert HPD at all — the divider is a
15.1 kΩ pull-*down*, and no path from `HDMI-5V` to `HDMI-HPD` remains
(`HDMI-5V`'s instance count dropped accordingly). The two use cases now differ:

* **Plugged into a TV or AVR input** (board acts as a source): the sink asserts
  HPD; sensing it is correct. The new circuit is complete for this — and CEC
  works regardless of HPD.
* **Plugged into a source's output** (board emulates a display): the source
  waits for HPD ≥ 2.4 V that this board can no longer produce; most sources
  will not talk — CEC included — until it appears.

**Action:** confirm which use the product intends and annotate the sheet. If
source-side use is wanted, add a firmware-switchable assert path (transistor
pull to `HDMI-5V`, ≥ 2.4 V under the 15.1 kΩ divider load) alongside the
sensing divider. Rework 7.

### 2.6 PCA9306 pin 1 / pin 2 assignment — **[CLOSED 2026-08-15 — symbol is correct]**

The `U7` symbol places `GND` on pin 1 and `VREF1` on pin 2, and the earlier
audit suspected TI's datasheet had them the other way round.

**Checked against TI's datasheet (SCPS113, Pin Functions table) on
2026-08-15:** GND = 1, VREF1 = 2, SCL1 = 3, SDA1 = 4, SDA2 = 5, SCL2 = 6,
VREF2 = 7, EN = 8 — the symbol matches on every pin. The suspicion was wrong;
recorded in §4.

Rev 08-15 also added `R42` 200 kΩ biasing `EN`/`VREF2` to `+3V3`, which is
exactly TI's reference circuit. `U7` is now correct in full — nothing left to
do.

### 2.7 The nRF24 antenna — **[RESOLVED 2026-08-15 on the schematic]**

As of rev 2026-08-12 the RF chain ended at a net labelled `50Ohm` with nothing
attached and no statement of intent.

**Status 2026-08-15: the sheet now reads "A PCB Trace antenna will be used
here"** at the 50 Ω port, behind the unchanged matching network
(`L2`/`L3`/`L4`, `C20`–`C23`, now with a 1 pF `C23` shunt at the port). That
is exactly what the item asked for — the intent is on the drawing and will
reproduce.

**Hand-off to layout:** the trace must be designed at 50 Ω with a ground
keep-out (Nordic's quarter-wave meander from nAN400-03 is the usual
reference), and its placement should maximise distance from the C6's antenna
(§5 finding 5, BOM §2). Re-open this only if the layout omits that.

### 2.8 The P4 cannot enter serial download mode — **[confirmed, blocking]**

The auto-download circuit and the `BOOT` button both drive **GPIO0**. That is
the boot strap on the ESP32 and ESP32-S3. **It is not a strapping pin on the
ESP32-P4.**

On the P4 the strapping pins are GPIO 34–38, and per
[Espressif's boot mode documentation](https://docs.espressif.com/projects/esptool/en/latest/esp32p4/advanced-topics/boot-mode-selection.html):
*"The ESP32-P4 will enter the serial bootloader when GPIO35 is held low on
reset."* `GPIO36` must additionally be high for that to work reliably.

On this board:

| Pin | Wired to | Effect |
| --- | --- | --- |
| **GPIO35** | *nothing* | The actual download strap is unrouted. Internally pulled high, so the chip always boots to flash. |
| **GPIO0** | `BOOT` button + `Q2` collector | Does nothing on this chip. |
| **GPIO36** | nRF24 `IRQ`, active low | Must be high to enter the bootloader; the radio pulls it low on any pending interrupt. |
| **GPIO34** | nRF24 `CE` | Strapping pin driven as an output. Benign, but not free of risk. |

**Consequence: there is no way to put the P4 into download mode.** Combined
with the unconnected USB PHY (no USB-JTAG, §1), the first board would arrive
with no way to flash the application processor at all.

**Fix, all pre-fab:**

1. **Move the `IO0` net from GPIO0 to GPIO35.** Both the `BOOT` button and `Q2`
   of the auto-download circuit follow it. This is a net rename plus one pin
   move; the transistor circuit itself is correct.
2. **Move nRF24 `IRQ` off GPIO36** to a non-strapping pin — GPIO 39 or 40 are
   free and adjacent. Leave GPIO36 unconnected so its internal pull holds it
   high.
3. **Consider moving nRF24 `CE` off GPIO34** for the same reason. Lower
   priority: `CE` is an output we control, and it is only sampled at reset.
4. Leave GPIO37/38 alone — console UART0 is their default function.

None of this costs a component. All of it is impossible after fabrication.

**Status 2026-08-15: unchanged — still open, still blocking.** Net `IO0` is
still on pin 104 (GPIO0); GPIO35 (pin 66) is still unrouted; nRF24 `IRQ`/`CE`
are still on GPIO36/GPIO34, with GPIO39/40 (pins 80/81) still free. The
revision added a printed DTR/RTS truth table for the auto-download circuit —
correct, and unaffected by the pin move.

### 2.9 The core regulator's enable is on the wrong net — **[FIXED 2026-08-15]**

Found 2026-08-15, by counting net-label instances in the PDF text layer rather
than reading them by eye. A connected net's label appears at two or more
coordinates; `FB-DCDC` does (P4 pin 78 and `U15`). **`EN-DCDC` appears exactly
once** — at `U15`'s EN pin — and P4 pin 79 (`EN_DCDC`) carries the label
**`EN`** instead, verified at 5× zoom: two characters, cleanly terminated,
with `FB-DCDC` rendering in full directly beneath it.

Net labels are global, so the drawing as it stands means:

* **`U15`'s enable floats.** Net `EN-DCDC` has one member and no driver. TI's
  datasheet does not permit a floating EN; whether it settles low (rail off) or
  drifts, the 1.1 V `VDD-HP` core rail is not reliably up — **and without it
  the P4 never boots. The board is dead on arrival.**
* **P4 pin 79 joins the reset net.** `EN` is the reset-button net: `CHIP_PU`
  (pin 103), `R24`, `C14`, and `Q1` of the auto-download circuit. The core
  regulator's enable output would sit on the chip's own reset line, fighting
  the button and `Q1` whenever either pulls it low.

**Fix: rename the label at pin 79 from `EN` to `EN-DCDC`.** One label. Both
failures disappear together.

*(The audit's §1 previously described this connection as correct — the label
was read at 1× zoom, where `EN` and a clipped `EN-DCDC` are indistinguishable.
Recorded in §4.)*

**Status 2026-08-15: FIXED.** `EN-DCDC` now counts two instances — pin 79 and
`U15` pin 1 — and `EN` dropped from nine to eight, exactly the one label that
moved. Net `EN` verified to contain `CHIP_PU` (103), the button, `C14`, `Q1`
and the new pull-up `R46` (relocated replacement for `R24`). Nothing else to
do.

### 2.10 The C6 recovery UART is not connected — **[FIXED 2026-08-15]**

Same instance-counting method. The recovery header's pin 1 carries label
`TXD0` and pin 2 carries `RXT0`; **neither string appears anywhere else on the
sheet as a net label.** The C6 module's `TXD0`/`RXD0` pins (31/30) end in short
dangling stubs with **no labels at all** — the second `TXD0` in the text layer
is the module's pin *name*, not a net.

So the header's UART pins connect to nothing, and the module's UART pins
connect to nothing. Recovery flashing of the C6 — the header's entire purpose —
is impossible as drawn. `BOOT` (via `R41`) and GND are correctly connected.

An earlier revision of this audit noticed only the `RXT0` spelling and filed it
as cosmetic (§3.6). That call was wrong twice over: the typo is real, but even
spelled correctly the net would have joined nothing, because the module side
carries no labels.

**Fix: add net labels `TXD0` and `RXD0` to module pins 31 and 30, and relabel
header pin 2 from `RXT0` to `RXD0`.** Three labels.

**Status 2026-08-15: FIXED functionally.** `TXD0` and `RXT0` each now count
two instances — header and module pin — so both directions connect. The
designer connected the net under the typo'd name rather than renaming it;
`RXT0` → `RXD0` is now purely cosmetic and lives in Rework 16.

### 2.11 CEC lost its ESD clamp — **[confirmed, NEW in rev 2026-08-15]**

`D3` (AZ5123-01F, the dedicated CEC ESD clamp present in rev 2026-08-12) was
**deleted in this revision with no replacement**. `D4`'s four-channel array
clamps `HDMI-SDA`, `HDMI-SCL`, `HDMI-5V` and `HDMI-HPD` — CEC is not among
its channels. What remains on the CEC net is the 27 kΩ pull-up `R11` and the
`D2` BAT54 rail clamp; neither is rated for ESD strikes.

CEC is the line this product exists for: hot-plugged, user-facing, and
electrically shared with every device on the HDMI tree. It is also wired more
or less directly to a P4 GPIO. An 8 kV contact discharge through an HDMI cable
shield-to-CEC fault is a real field event, and the failure it causes (a dead
GPIO on a soldered-down BGA) is not reworkable.

**Action:** restore a low-capacitance TVS on CEC at the connector. The removed
AZ5123-01F was suitable; any ≤ 5 V working-voltage, < 10 pF unidirectional
clamp works. If the deletion was deliberate — e.g. a plan to move CEC behind a
buffer — record that on the sheet instead. Rework 19.

---

## 3. Lower-Priority Recommendations

1. **IR interface architecture (structural difference from the specification).**
   * **Specified:** two 3.5 mm TRS jacks as universal/flexi IR ports, each using
     a high-side switch on the tip (firmware-selectable TX power or RX VCC) and
     a ring input with series protection.
   * **In the schematic:** **four** separate jacks — `U8`/`U10` as dedicated TX
     blasters driven by low-side NPNs (`Q3`, `Q4`), `U9`/`U11` as dedicated RX
     receivers.
   * **Action:** decide whether you want the current four single-purpose jacks
     or two universal TRS jacks to save panel and board space. Note that fixing
     §2.2 is required either way.

2. **Missing series protection on the IR RX lines.** The RX node runs from the
   jack straight to the P4 GPIO with only the 10 kΩ pull-up. Add a **1 kΩ series
   resistor** between jack and GPIO on each zone to survive ESD and momentary
   shorts during hot-plugging.

3. **Debounce capacitor sizing.** `C14`/`C15`/`C16` are 1 µF against 10 kΩ
   pull-ups, giving $\tau = 10\text{ ms}$ — sluggish for rapid presses, and on
   `EN`/`IO0` it fights the auto-download circuit's timing. Drop them to
   **100 nF** ($\tau = 1\text{ ms}$).
   **Rev 2026-08-15 made this slightly worse:** new `C56` 1 µF sits in
   parallel with `C14` on the `EN` net (~2 µF, τ ≈ 20 ms on reset). Keep one
   capacitor per net and make it 100 nF.

4. **CP2102N supply configuration — [SUPERSEDED 2026-08-15].** `VREGIN` (pin
   7) on `+VBUS` while `VDD` (pin 6) is on `+3V3` ran the bridge's internal
   LDO in parallel with the SY8089 buck — not a datasheet configuration.
   Moot: the CP2102N is deleted by the USB-Serial-JTAG decision (§3.9,
   Rework 20).

5. **No `EN` on the C6 recovery header.** A manual recovery flash of the C6
   needs a reset, and the header does not offer one. Add a fifth pin, or accept
   the power-cycle workflow and document it.

6. ~~Cosmetic: rename `RXT0` → `RXD0`.~~ **Superseded by §2.10** — the
   spelling was the least of it; the whole recovery UART is disconnected.

7. **Jack insertion detect — [SPECIFIED 2026-08-14, pre-fab].** Detection is
   now a required feature, and the GPIOs are allocated. Full circuit below in
   §6; pin map in [GPIOs.md](GPIOs.md).

8. **Connector family — [DECIDED 2026-08-14: full-size Type A].**
   `HDMI-519` (`C136421`) is Mini-HDMI Type C, which would have obliged every
   user to source a Mini-HDMI-to-Type-A cable. Superseded: the board uses a
   **full-size Type A female** receptacle. See the BOM for candidate parts and
   §2.1 for the pin remap this forces.

9. **USB-Serial-JTAG — [DECIDED 2026-08-15: adopt, delete the CP2102N].**
   The P4's ROM USB-Serial-JTAG on GPIO24/25 replaces the CP2102N on the
   existing USB-C: flashing (works on a blank chip), console, and full JTAG
   debugging over one cable — the fallback whose absence §1 flagged. DDC SCL
   moves GPIO24 → GPIO46, the HPD tap GPIO25 → GPIO47; `U6`, `Q1`/`Q2` and
   five resistors are deleted; `VDD_USBPHY` gets connected and the OTG HS
   pins (49/50) go to test pads as a DFU provision. The GPIO35 strap fix
   (§2.8) **remains mandatory** as the last-resort download entry. Complete
   change list: **Rework 20**. Verify at layout: D+/D− assignment on
   GPIO25/24, and whether the GPIO24/25 FS PHY is powered from `VDD_USBPHY`.

---

## 4. Corrections to the Previous Revision of This Document

For traceability, the following statements in the earlier audit were wrong and
have been fixed above:

| Was | Now |
| --- | --- |
| "five mandatory 51 kΩ pull-ups (`R31`–`R35`)" | Six are fitted, `R31`–`R36`, including one on `CLK` |
| Decoupling "distributed across the `+3V3` and `VDD-HP` power pins" | All twelve are on `+3V3`; `VDD-HP` has none — §2.4 |
| PCA9306 "low side powered by `+3V3` with pull-ups `R12`, `R13`; high side by `HDMI-5V` with `R14`, `R15`" | Reversed: `R12`/`R13` are the 5 V pull-ups, `R14`/`R15` the 3.3 V ones |
| nRF24 `IRQ` on "GPIO 37" | GPIO 36 |
| `FSW` "connected to GPIO 38" | GPIO 41 |
| HPD divider "yielding ≈2.02 V for safe input" | Correct arithmetic, wrong conclusion — 2.02 V is below the HDMI 2.4 V minimum, §2.5 |
| CP2102N divider "on `VREGIN`/`VBUS`" | Only the `VBUS` sense pin (8) is divided; `VREGIN` is on `+VBUS` directly |
| IR RX "100 nF filtering capacitor (`C9`/`C10`)" | The capacitors are in series in the supply path, not filtering — §2.2 |
| Filename `SCH_Schematic1_2026-08-12_2.pdf` | `SCH_Schematic1_2026-08-12.pdf` |

### Corrections to the audit itself

| Was (2026-08-14/15) | Now |
| --- | --- |
| §1: "the P4 drives both EN (via EN-DCDC) and the feedback node" | The enable label is `EN`, not `EN-DCDC` — the connection does not exist. §2.9. Read at too low a zoom; net-label claims now verified by instance-counting the PDF text layer. |
| §3.6: `RXT0` "cosmetic typo" | The recovery UART is entirely disconnected — §2.10. The typo was real but immaterial. |
| §2.6: "TI's PCA9306 datasheet has these the other way round" | **The datasheet agrees with the symbol** (GND = 1, VREF1 = 2 in the DCU package) — checked against TI SCPS113 on 2026-08-15. The suspicion, not the schematic, was wrong. |
| §1 HDMI role: "the board asserts HPD itself, so this is an HDMI sink" | True only of rev 2026-08-12. Rev 08-15 removed the assert path entirely; the board now senses HPD — §2.5. |

### Correction to the 2026-08-14 audit itself

| Was | Now |
| --- | --- |
| §2.1 "HDMI connector symbol pin numbers do not match the HDMI standard", raised as likely blocking | **Withdrawn.** The connector is Mini-HDMI **Type C**, whose pinout genuinely differs from Type A in exactly the way the symbol shows. The schematic was right; the audit compared it against the wrong connector family. See §2.1. |

That error is worth recording rather than quietly deleting: it was found only
because the JLCPCB part number identified the connector as Type C, which the
schematic sheet never states. **Adding the connector family to the sheet would
have prevented it** — a worthwhile annotation for the next revision.

New findings not present in the previous revision: §2.2, §2.3, §2.4, §2.6, §2.7,
§3.4, §3.5, §3.7, §3.8, §5, and the "no native USB" note in §1. Found on the
2026-08-15 re-verification pass: **§2.9 and §2.10**. Found auditing rev
2026-08-15 itself: **§2.11** (CEC clamp deleted), the §2.2 re-work that
preserved the defect, and the §2.5 redesign.

---

## 5. Part Choice Review — nRF24L01+ (`U12`)

> ## ⚠️ VERDICT REVERSED — **KEEP THE nRF24L01+**
>
> This section originally recommended deleting `U12`. **That recommendation is
> withdrawn.** Logitech Harmony remotes talk to their hub over 2.4 GHz using
> **Enhanced ShockBurst — the nRF24's own protocol**. `U12` is not speculative
> hardware; it is the only part on the board that can receive a Harmony remote,
> and Harmony support is now a product requirement.
>
> Findings 2–5 below still stand as risks to manage. Finding 1 is void and
> finding 6 inverts into an argument *for* buying genuine silicon.
> See [remote-control-plan.md](../../docs/remote-control-plan.md).

**1. ~~Nothing uses it.~~ VOID.** Superseded: Harmony RF support requires it.
When written, no firmware referenced `U12` and no purpose was recorded anywhere
in the repository — which is why the part looked speculative. The lesson is
about the documentation gap, not the part: **an unstated requirement reads as a
mistake.** The intended function is now recorded in the plan document.

**2. Nordic marks the whole nRF24 series "Not recommended for new designs"**
and points new work at the nRF52 series instead. That is verbatim from Nordic's
own product page, not a forum reading. Parts remain purchasable, but a new
board spun in 2026 around an NRND radio starts its life on a countdown.

**3. Counterfeits are endemic, and the schematic specifies the bare die.**
`NRF24L01P-R` is a bare QFN20, which is exactly the format the fakes come in.
Reported fakes are built on a 350 nm process against the genuine 250 nm, with a
larger die, worse sensitivity and higher power draw. There is no register or
marking that identifies them — the practical screens are a die-shot or the
power-down current test (≈500 nA genuine against 3–4 µA fake). Sourcing through
a Nordic-authorised distributor is the only real defence, and it is not where
bare dies at this price usually come from.

**4. A bare radio die forfeits the pre-certified-module advantage.** The C6 next
to it is a module with an integrated antenna and existing certification. Adding
a second, bare, intentional radiator means the finished product needs full
FCC/CE radiated-emissions testing on its own — several thousand USD and weeks of
schedule that the C6 alone would not have cost.

**5. Coexistence.** nRF24 is 2.4 GHz GFSK sharing a band and a small PCB with
the C6's WiFi 6 and Bluetooth LE. The ESP32 arbitrates its own WiFi/BT
internally but knows nothing about an external radio, so there is no mechanism
to stop the two desensing each other. Any duty cycle on the nRF24 costs WiFi
throughput and vice versa.

**6. Clone silicon will break Harmony pairing — INVERTED into a buying rule.**
This was originally an argument against the part. It is now the strongest
argument for buying a **genuine** one. Harmony's pairing handshake uses the
nRF24's **ACK-payload** feature, and the widespread **Si24R1** clone shipped
with an **inverted ACK bit** (following an error in its own datasheet). A clone
will therefore fail to pair even though it looks like a working radio. **XN297**
is not protocol-compatible at all.

Combined with finding 3, this is a hard sourcing constraint: **buy `U12` through
a Nordic-authorised distributor, and screen incoming parts** with the power-down
current test (≈500 nA genuine, 3–4 µA fake) before committing a build.

### Options

| | Option | Consequence |
| --- | --- | --- |
| **A** | ~~Drop `U12`~~ | **Void.** Would delete Harmony support entirely. |
| **B** | ~~Footprint only, DNP~~ | **Void.** Same. |
| **C** | **Keep the bare die, source it properly** | Cheapest path that works. Requires: an antenna (§2.7), authorised sourcing, incoming inspection, and full FCC/CE radiated testing for the finished product. |
| **D** | Swap for a pre-certified **nRF24 module** | Solves antenna + certification + counterfeit in one purchase, at higher unit cost and board area. Strong candidate if volume is low or certification budget is tight. |
| **E** | Swap for an **nRF52 module** running ESB | Nordic's supported replacement for the NRND part; ESB in software keeps Harmony compatibility and adds headroom to decode more of the protocol later. Highest BOM cost, adds a second MCU to maintain. |

**Recommendation: C for the prototype, then decide between C and D before
volume.** Build the first boards with the bare die so the RF layout gets
validated early — that is the risk you cannot defer. If certification cost or
yield on the die turns out badly, D is a drop-in escape that does not change the
firmware at all, because it is the same radio behind the same SPI interface.

Findings 2 (NRND) and 5 (coexistence with the C6) remain live and unfixable by
part choice. Manage them: NRND by keeping D/E as escape routes, coexistence by
antenna placement — see the plan document.

---

## 6. IR jack insertion detect — circuit specification

Added 2026-08-14. Makes `ir::Ir::connected()` return a real answer instead of
`Presence::Unknown`. The firmware side is already written and inert until these
pins exist.

### Why nothing softer works

The RX signal line carries a 10 kΩ pull-up (`R19`/`R22`). An **empty jack** sits
high through it. A **plugged-in receiver with no remote pointed at it** also
idles high. The two are the same voltage on the same net, so no amount of
firmware separates them:

| Attempt | Result |
| --- | --- |
| Read the level | High in both cases |
| Drive low, release, time the rise | 10 kΩ into cable capacitance is ~1 µs; a receiver's push-pull output is faster but not distinguishably, and driving into it is contention |
| Enable the internal pull-down | ~45 kΩ against an external 10 kΩ still reads high |
| Wait for IR traffic | Proves presence, never absence |

Detection has to come from the connector.

### Pin allocation

| Jack | Function | GPIO | Package pin |
| --- | --- | --- | --- |
| `U8` | Zone 1 TX (emitter) detect | **42** | 83 |
| `U9` | Zone 1 RX (receiver) detect | **43** | 84 |
| `U10` | Zone 2 TX (emitter) detect | **44** | 86 |
| `U11` | Zone 2 RX (receiver) detect | **45** | 87 |

All four are currently unrouted, none is a strapping pin (§2.8), and they sit on
the same package edge as the existing IR pins. If only two can be afforded,
**fit the RX pair (43, 45)** — an absent emitter is at least visible as "the
device did not respond", whereas an absent receiver is silent.

### Preferred: a jack with an independent switch contact

Specify a 3.5 mm jack whose insertion switch is a **separate SPST contact**, not
one normalled to the tip.

```
   3V3 ──[10k]──┬───────────────► GPIO 4x
                │
       jack ────┘   switch closes to GND when the jack is EMPTY
       switch ──────────────────► GND
```

* Empty jack → switch closed → GPIO reads **LOW**
* Plug inserted → switch open → pull-up → GPIO reads **HIGH**

This matches the firmware default `ZoneConfig::detect_active_low = true`. The
P4's internal pull-up (~45 kΩ) can carry this on its own, so the 10 kΩ is
optional — fit it for noise immunity on a connector-facing net. Add the same
1 kΩ series resistor recommended for the signal lines in §3.2.

Identical on TX and RX jacks, which is the main reason to prefer it.

### Fallback: a tip-normalled switch (e.g. plain PJ-320A)

If only tip-switch jacks are available, the switch terminal is shorted to the
tip when empty and floats when a plug is inserted. That still works on the **RX
jacks**, with no extra parts at all:

* Switch terminal → GPIO, configured input with the **internal pull-down**.
* Empty: switch closed to tip, which `R19` pulls to 3V3 → divider with the
  ~45 kΩ internal pull-down gives ~2.7 V → reads **HIGH**.
* Inserted: switch open → internal pull-down → reads **LOW**.

Note the polarity is **inverted** versus the preferred circuit, so set
`ZoneConfig::detect_active_low = false` for these zones. The firmware supports
both per zone.

**This does not work on the TX jacks.** Their tip sits at `+VBUS` through the
33 Ω `R18`/`R21`, so a closed switch would put 5 V on a 3.3 V GPIO. Either fit
a divider (2 × 100 kΩ gives ~2.5 V) or leave the TX jacks undetected.

### Alternative: current sense on the receiver's supply

Worth knowing about because it detects **faults**, not just presence: put a
small series resistor in the RX jack's VCC feed and read the drop. A receiver
draws ~0.4–1.5 mA, so 100 Ω gives ~100 mV — enough to separate absent, present
and shorted.

Two caveats. It needs an ADC channel per zone, and **ADC1's eight inputs
(GPIO16–23) are all already used** on this board, so it would have to come from
ADC2 — verify that against the datasheet before committing. It also touches the
same net as the `C9`/`C10` fix (§2.2), so if that rework is happening anyway,
the incremental cost is one resistor.

The switch-contact approach is simpler and is what the firmware expects. Treat
this as the option to reach for only if emitter/receiver *fault* detection turns
out to matter.

### Consequence for part selection

[BOM.md §4](BOM.md) already flagged that the proposed jack `C18167` needs its
pin numbering verified. That is now a harder requirement: **the chosen jack must
have a documented insertion-switch contact**, and the datasheet must say whether
it is independent or normalled to the tip, because the two need opposite
firmware polarity and different external parts.
