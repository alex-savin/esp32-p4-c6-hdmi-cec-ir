# Schematic rework list — for the board designer

Against `SCH_Schematic1_2026-08-15.pdf`. First compiled 2026-08-14 against the
2026-08-12 revision; **re-audited 2026-08-15 against the new revision**, which
resolved five items and introduced one new one. Item numbers are stable across
revisions — 17–20 are numbered out of sequence deliberately so older references
stay valid.

Everything still open must be resolved **before fabrication**. Each item states
the change; the reasoning is in [Improvements.md](Improvements.md), referenced
by section. Pin numbers are ESP32-P4 package pins, taken from the drawing.

**Where rev 2026-08-15 stands: 3 of the original 6 blocking items are fixed or
resolved; 3 remain — and one of those (item 3) was reworked in this revision
but is still broken.** The revision also added a set of unsolicited
improvements that are worth keeping (see the credits list at the bottom).

---

## Summary

| # | Change | Severity | Status in rev 2026-08-15 |
| --- | --- | --- | --- |
| 17 | Rename pin-79 net `EN` → `EN-DCDC` | 🔴 was board-dead | ✅ **Fixed** — verified both endpoints |
| 18 | Connect the recovery-header UART | 🔴 was blocking | ✅ **Fixed** — connected end to end (`RXT0` typo remains, item 16) |
| 4 | Specify the nRF24 antenna | 🔴 was blocking | ✅ **Resolved** — "PCB Trace antenna" declared on the sheet; layout must follow through |
| 13 | Verify PCA9306 pin 1/2 | 🔵 was check | ✅ **Closed** — TI datasheet confirms the symbol (GND = 1, VREF1 = 2); new `R42` 200 kΩ bias matches TI's reference circuit |
| 1 | Move boot strap from GPIO0 to **GPIO35** | 🔴 Blocking | ❌ **Open** — `IO0` still on pin 104, GPIO35 still unrouted |
| 2 | HDMI connector → **Type A**, remap 13/14/17 | 🔴 Blocking | ❌ **Open** — still `HDMI-519` Type C (shell EP pads now grounded ✓) |
| 3 | IR receiver has no DC supply | 🔴 Blocking | ❌ **Re-worked but still broken** — see the rewritten item |
| 19 | **NEW:** restore ESD protection on CEC | 🟠 High | ❌ `D3` (AZ5123) was deleted in this revision with no replacement |
| 5 | Move nRF24 `IRQ` off GPIO36 | 🟠 High | ❌ Open — `IRQ` still pin 68/GPIO36, `CE` still pin 65/GPIO34 |
| 6 | Local decoupling on the 1.1 V core rail | 🟠 High | ❌ Open — no caps moved to `VDD-HP` |
| 7 | HPD level | 🟠 High | 🔁 **Redesigned — confirm the new intent** (see rewritten item) |
| 8 | IR jack insertion detect (4 GPIOs + switched jacks) | 🟠 High | ❌ Open — GPIO 42–45 still unrouted, jacks still `PJ-320A-3P` |
| 9 | Flash 4 MB → 8 MB | 🟡 Cheap now | ❌ Open — still `W25Q32JVSSIQ` (six new pull-ups ✓, wrong rail — item 16) |
| 10 | 1 kΩ series resistors on IR RX | 🟡 Cheap now | ❌ Open |
| 11 | Debounce caps 1 µF → 100 nF | 🟡 Cheap now | ❌ Open — **worse**: `C56` 1 µF added in parallel with `C14` on `EN` |
| 12 | 5th pin (`EN`) on the C6 recovery header | 🟡 Cheap now | ❌ Open — header still 4-pin |
| 14 | Verify the 3.5 mm jack datasheet | 🔵 Check | ❌ Open — and item 8 requires a *switched* jack anyway |
| 15 | Decide the C6 antenna variant | 🔵 Decide | ❌ Open |
| 16 | Annotate + tidy the sheet | ⚪ Housekeeping | 🟡 Partial — see item |
| 20 | **NEW:** replace the CP2102N with native **USB-Serial-JTAG** | 🟢 Decided 2026-08-15 | To implement — adds USB flashing **and JTAG debugging**, deletes 8 parts |

---

## ✅ Resolved in rev 2026-08-15

### 17. Core regulator enable — FIXED

`EN-DCDC` now appears at exactly two coordinates: P4 pin 79 and `U15` pin 1.
Verified by instance-counting the PDF text layer. The reset net `EN` correctly
carries `CHIP_PU` (pin 103), the reset button, `C14`, `Q1`, and a new pull-up
`R46` 10 kΩ (which replaces the old `R24` — same function, relocated to the
auto-program block).

### 18. C6 recovery UART — FIXED (functionally)

Module pin 31 now carries net `TXD0` and pin 30 carries net `RXT0`, matching
the header labels — both directions connect end to end. The net name `RXT0` is
still the old typo; renaming it to `RXD0` is now purely cosmetic and has moved
back to item 16. A pull-up `R41` 10 kΩ was also added on the C6 `BOOT` strap
(IO9) and `R61` 10 kΩ on `RESET-CTR` — both good.

### 4. nRF24 antenna — RESOLVED on the schematic

The sheet now states **"A PCB Trace antenna will be used here"** at the 50 Ω
port, behind the full matching network (`L2`/`L3`/`L4`, `C20`–`C23`).
**Remaining for layout, not schematic:** the trace must be designed at 50 Ω
with a ground keep-out, per Nordic's reference (e.g. the quarter-wave meander
from nAN400-03). This item stays closed unless the layout omits that.

### 13. PCA9306 pin 1/2 — CLOSED, symbol is correct

Checked against TI's datasheet (SCPS113, Pin Functions table) on 2026-08-15:
**GND = pin 1, VREF1 = pin 2, SCL1 = 3, SDA1 = 4, SDA2 = 5, SCL2 = 6,
VREF2 = 7, EN = 8** — exactly what the symbol shows. The audit's suspicion was
wrong; recorded in Improvements §4. Rev 2026-08-15 additionally added `R42`
200 kΩ from `EN`/`VREF2` to `+3V3`, which is TI's reference biasing. `U7` is
now correct in full.

---

## 🟢 Decided design changes

### 20. Replace the CP2102N with the P4's native USB-Serial-JTAG

*Decided 2026-08-15. Fixes the audit's "no USB-JTAG debugging and no USB DFU"
finding (Improvements §1) and deletes the CP2102N along with its §3.4 supply
oddity.*

The ESP32-P4 has a **USB-Serial-JTAG controller in ROM** on **GPIO24 (D−) /
GPIO25 (D+), package pins 52/53** — the same block the ESP32-P4-Function-EV
board uses as its debug port. One cable then provides: flashing on a blank or
bricked chip (it is ROM), the serial console, and **full JTAG debugging**
(OpenOCD + gdb) with no external probe. The CP2102N becomes redundant.

GPIO24/25 currently carry HDMI DDC SCL and the HPD sense tap, so those two
nets move to free pins. All changes:

1. **USB-C data** (through the existing `D1` USBLC6, which stays where it is)
   → **GPIO25 = D+, GPIO24 = D−**. Route as a 90 Ω differential pair, keep it
   short. *Verify the D+/D− assignment against the datasheet pin table before
   layout.*
2. Move `SCL` (PCA9306 `SCL1` side, `R14` pull-up follows the net) from
   GPIO24 → **GPIO46 (pin 88)**. I2C routes through the GPIO matrix; any free
   non-strap pin works.
3. Move `HDMI-HPD-ESP` (the `R60`/`R59` divider tap, item 7) from GPIO25 →
   **GPIO47 (pin 89)**.
4. **Delete:** `U6` CP2102N, `R8`/`R9` (VBUS sense), `R43` (RSTb), `Q1`/`Q2`
   with `R5`/`R10` (auto-program — esptool triggers download over
   USB-Serial-JTAG natively), and `C56` (which also resolves the
   double-capacitor half of item 11). **Keep** `R46`/`C14` on `EN`, both
   buttons, and `R25`/`C15` on the boot-strap net.
5. **Connect `VDD_USBPHY` (pin 51) → `+3V3`** with 100 nF — it floats today.
   Bring `USB_DM`/`USB_DP` (pins 49/50) to **test pads**: that preserves the
   OTG HS port (ROM DFU, future TinyUSB device/host) at zero cost. *(Decision
   2026-08-15: pads only, no second connector.)*
6. Optional but cheap: a 3-pad header on GPIO37/38 (UART0 TX/RX) + GND for
   last-resort ROM logs; otherwise GPIO37/38 become free.

**Interaction with item 1 — the GPIO35 strap fix stays mandatory.**
USB-Serial-JTAG enters download mode by software command, which covers the
normal case; the strap + `BOOT` button is the only door left if firmware
wedges the USB block or reassigns the pins. After both changes: button = last
resort, USB = everything else.

*Firmware follows:* `board_pins.h` (SCL, HPD) and the sdkconfig console
options change in lockstep **when the schematic revision lands**, not before.

---

## 🔴 Blocking — still open

### 1. Boot strap is on the wrong pin — the P4 cannot be flashed

*Improvements §2.8. Unchanged in rev 2026-08-15.*

The `BOOT` button and auto-download `Q2` still drive net `IO0` into **pin 104
(GPIO0)**, which is not a strapping pin on the ESP32-P4. **Pin 66 (GPIO35)** —
the actual download strap — is still unrouted. As drawn the P4 can never enter
serial download mode, and with the USB PHY unconnected there is no USB-JTAG
fallback: the application processor is unflashable.

**Change:** move the `IO0` net from pin 104 to **pin 66 (GPIO35)**. The button,
`R25`, `C15` and `Q2` all follow the net; only the pin moves. Leave GPIO0
unconnected. Consider renaming the net `STRAP_BOOT`. The new auto-program truth
table printed on the sheet is correct and unaffected.

### 2. HDMI connector must become full-size Type A

*Improvements §2.1, BOM §3. Unchanged in rev 2026-08-15 — still `HDMI-519`
(Mini Type C), still Type C pin mapping.*

The decision is full-size **Type A**. Type C is not Type A renumbered — pins
13, 14 and 17 differ:

| Net | Currently (Type C) | Must become (Type A) |
| --- | --- | --- |
| `CEC` | pin 14 | **pin 13** |
| GND | pin 13 | **pin 17** |
| *(no-connect)* | pin 17 | **pin 14** — Utility/HEAC+ |
| `HDMI-SCL` / `HDMI-SDA` / `HDMI-5V` / `HDMI-HPD` | 15 / 16 / 18 / 19 | unchanged |

Getting 13 and 17 the wrong way round shorts CEC to ground; the symptom is that
nothing on the bus ever answers. Candidate parts (19-pin Type A female,
right-angle SMD): `C2858275`, `C2906135`, `C5184845`. Replace symbol **and**
footprint.

**Progress in 08-15:** the shell EP pads (20–23) are now tied to GND — keep
that. On the Type A part also ground the TMDS shield pins (2, 5, 8, 11).

### 3. The IR receiver still has no DC supply — re-worked in 08-15, still broken

*Improvements §2.2 — rewritten for the new topology.*

Rev 2026-08-15 rewired both RX jacks (`U9`, `U11`), but the defect survived the
rework. Now: the signal (`RX_IR_JACK_x`, 10 kΩ pull-up `R19`/`R22`) moved from
jack pin 2 to **pin 4**; pin 3 is GND; and **pin 2 connects to `+3V3` only
through the 100 nF capacitor** (`C9`/`C10`) — a series capacitor, exactly the
topology the original item flagged, moved to the other contact. A three-wire IR
receiver (VCC/GND/OUT) still has **no contact that supplies DC power**; it will
never work in either zone.

**Change:** wire jack pin 2 (the receiver's VCC contact) **directly to
`+3V3`**, and hang `C9`/`C10` from that node **to GND** as decoupling. Three
nodes: pin 2 = 3V3, pin 3 = GND, pin 4 = signal with pull-up. Verify against
the jack datasheet (item 14) that pins 2/3/4 are the contacts the plug's
tip/ring/sleeve actually land on.

*(Good addition kept from 08-15: `R44`/`R45` 10 kΩ base pull-downs on the TX
drivers `Q3`/`Q4`, so the IR LEDs stay off while the P4 boots.)*

---

## 🟠 High priority

### 19. NEW — CEC lost its ESD protection in rev 2026-08-15

`D3` (AZ5123-01F, the CEC ESD clamp) was **deleted** in this revision, with no
replacement. `D4`'s array covers SDA/SCL/HPD/+5 V, **not CEC**. What remains on
CEC is `R11` 27 kΩ and the `D2` BAT54 clamp — neither is ESD protection.

CEC is the one line this product exists to expose: hot-plugged, user-facing,
wired to every device in the AV stack. **Change:** restore a low-capacitance
TVS on the CEC net at the connector (the removed AZ5123-01F was fine; any
≤5 V working-voltage, <10 pF part works). If the deletion was deliberate,
record why on the sheet.

### 5. nRF24 `IRQ` is on a strapping pin

*Improvements §2.8. Unchanged in rev 2026-08-15 — `IRQ` still on pin 68
(GPIO36), `CE` still on pin 65 (GPIO34), GPIO39/40 still free.*

GPIO36 must be high at reset for reliable bootloader entry; `IRQ` is active-low
and the radio owns it. **Change:** move `IRQ` to **pin 80 (GPIO39)**;
optionally move `CE` to **pin 81 (GPIO40)**. Leave GPIO36 unconnected.
*Firmware follows:* `board_pins.h` changes in lockstep.

### 6. The 1.1 V core rail has no local decoupling

*Improvements §2.4. Unchanged in rev 2026-08-15 — all twelve 100 nF still on
`+3V3`; `VDD-HP` still has only the far-end 22 µF `C31`.*

(The new `C59`/`C60` 22 pF on the SY8089's feedback and `C58` 10 µF on the C6
rail are welcome but are different fixes for different rails.)

**Change:** one 100 nF per core pin (26, 54, 76, 91) on `VDD-HP`, placed at the
pin in layout, plus a local 10 µF. Follow Espressif's P4 hardware design guide.

### 7. HPD — redesigned in 08-15; confirm the intent, then it is one resistor from right

*Improvements §2.5 — rewritten.*

The old circuit *asserted* HPD (R16/R17 divider from `HDMI-5V`, at an
out-of-spec 2.02 V). Rev 2026-08-15 deleted it and instead **senses** HPD:
`HDMI-HPD` → `R60` 5.1 kΩ → `HDMI-HPD-ESP` (P4 pin 53, GPIO25) → `R59` 10 kΩ →
GND. A 5 V HPD now reaches the GPIO as 3.31 V — safe, and `D5`'s removal is
fine because the divider bounds the node and `D4` still clamps the connector
side.

**What no longer exists is any way for the board to assert HPD.** That matters
only if the board is meant to plug directly into a *source* (a console, a
streamer) and convince it a display is attached; sources wait for HPD ≥ 2.4 V
before doing anything, CEC included. Plugged into a TV, sensing is the correct
and sufficient behaviour.

**Change:** decide the use case and record it on the sheet. If source-side
operation is wanted, add one option: a GPIO-controlled pull (e.g. a P-FET or
even a 1 kΩ from `HDMI-5V` behind a transistor) that firmware can enable to
assert HPD ≥ 2.4 V while the divider still senses it. If the board is
TV/AVR-side only, the new circuit is complete as drawn — close this item with
a sheet annotation.

### 8. IR jack insertion detect

*Improvements §6. Unchanged in rev 2026-08-15 — GPIO 42/43/44/45 (pins
83/84/86/87) still unrouted, jacks still switchless `PJ-320A-3P`.*

Bring each jack's insertion-switch contact to its GPIO per the circuit in
Improvements §6; requires the switched-jack part choice (item 14). If only two
detect lines can be afforded, fit the RX pair (GPIO 43, 45).

---

## 🟡 Cheap now, impossible later

### 9. Flash 4 MB → 8 MB

*BOM §8. Unchanged — `U13` is still `W25Q32JVSSIQ`.* Swap to **W25Q64JVSSIQ**,
same package and pinout. Two OTA slots plus the feature set make 4 MB
survivable-with-no-headroom; 8 MB removes the question. *(08-15 added six
10 kΩ pull-ups, `R51`–`R54`/`R56`/`R57`, on all six flash pins — keep, they
apply to either part, but re-reference them to `VDD-FLASH`: see item 16.)*

### 10. Series protection on the IR RX lines

*Improvements §3.2. Unchanged.* 1 kΩ between jack and GPIO on each RX zone.
Do it together with the item-3 rewire.

### 11. Debounce capacitors

*Improvements §3.3. Regressed slightly in 08-15:* `C14`/`C15`/`C16` are still
1 µF, and new `C56` 1 µF sits in parallel with `C14` on `EN`, so the reset RC
is now ~20 ms. **Change:** `C14`–`C16` → 100 nF. (`C56` is already deleted by
item 20, which resolves the parallel-capacitor half.)

### 12. No `EN` on the C6 recovery header

*Improvements §3.5. Unchanged — the header is still 4-pin.* Add a fifth pin
carrying the C6 `EN`/`RESET-CTR` net.

---

## 🔵 Check before ordering

### 14. The 3.5 mm jack datasheet

*BOM §4. Unchanged, and item 8 sharpens it:* the chosen jack must have a
documented insertion-switch contact (independent SPST preferred), and the
tip/ring/sleeve-to-pin mapping must be verified — the item-3 rewire depends on
it. `C18167` remains unverified on both counts; `PJ-320A-3P` (fitted) has no
switch at all.

### 15. C6 antenna variant

*BOM §2. Unchanged decision.* `-1U` needs an IPEX-3/MHF3 antenna; the non-`U`
drops into the same footprint. Two 2.4 GHz receivers now run concurrently;
antenna separation is the main defence against mutual desense. Note the nRF24
side is now committed to a PCB trace antenna (item 4), which weighs toward the
`U` variant for the C6 only if the enclosure forces the two apart. **Decide
against the enclosure.**

---

## ⚪ Housekeeping

### 16. Sheet annotations

Progress in 08-15: auto-program truth table printed ✓, antenna note ✓, M2
mounting holes added ✓. Still to do:

* **State the HDMI connector family on the drawing** — the omission that caused
  a finding to be raised, withdrawn and re-raised (Improvements §2.1).
* Rename net `RXT0` → `RXD0` (header pin 2 and C6 module pin 30). Cosmetic
  since the 08-15 fix, but it will mislead the next reader.
* ~~`CP2102N` `VREGIN` configuration~~ — **superseded by item 20**, which
  deletes the part.
* **Check LED brightness:** `R48`/`R49` changed 100 Ω → 2.2 kΩ, so the status
  LEDs now run at ~0.6 mA. Modern 0805s are visible at that, but confirm it is
  a choice, not a typo — 470 Ω–1 kΩ is the usual compromise.
* State the C6 module part number on the sheet (`U5` is a 53-pad MINI-1
  footprint but no variant is named anywhere on the drawing).
* **Re-reference the flash pull-ups to `VDD-FLASH`.** The six new 10 kΩ
  (`R51`–`R54`, `R56`, `R57`) pull the flash's CS/DO/WP/HOLD/CLK/DI to
  **`+3V3`**, but `U13`'s VCC sits on `VDD-FLASH` — the P4's internal LDO
  output, which rises after 3V3. Until it does, the flash I/O pins sit 3.3 V
  above VCC (≈0.3 mA per pin through the 10 kΩ) — outside the usual
  VCC + 0.4 V absolute maximum. Move the pull-up rail to `VDD-FLASH`.

---

## Net and pin changes still outstanding

| Net | From | To | Item |
| --- | --- | --- | --- |
| `IO0` (boot strap) | pin 104 / GPIO0 | **pin 66 / GPIO35** | 1 |
| `IRQ` (nRF24) | pin 68 / GPIO36 | **pin 80 / GPIO39** | 5 |
| `CE` (nRF24, optional) | pin 65 / GPIO34 | **pin 81 / GPIO40** | 5 |
| `CEC` (connector end) | HDMI pin 14 | **HDMI pin 13** | 2 |
| GND (connector end) | HDMI pin 13 | **HDMI pin 17** | 2 |
| RX jack pin 2 (both zones) | series `C9`/`C10` to 3V3 | **direct to `+3V3`**, cap to GND | 3 |
| *new* — CEC TVS | — | clamp at connector | 19 |
| *new* — 4 × jack detect | — | GPIO 42, 43, 44, 45 | 8 |
| USB-C `D+`/`D−` | CP2102N | **GPIO25 / GPIO24** (pins 53/52) | 20 |
| `SCL` (PCA9306 3.3 V side) | GPIO24 / pin 52 | **GPIO46 / pin 88** | 20 |
| `HDMI-HPD-ESP` (divider tap) | GPIO25 / pin 53 | **GPIO47 / pin 89** | 20 |
| `VDD_USBPHY` (pin 51) | floating | **`+3V3`** + 100 nF | 20 |
| `USB_DM`/`USB_DP` (pins 49/50) | floating | **test pads** (OTG/DFU provision) | 20 |

## Part changes still outstanding

| Ref | From | To | Item |
| --- | --- | --- | --- |
| `AUDIO1` | HDMI-519, Mini Type C | **Type A, 19-pin R/A SMD** | 2 |
| `U13` | W25Q32JVSSIQ (4 MB) | **W25Q64JVSSIQ (8 MB)** | 9 |
| `C14`–`C16` | 1 µF | **100 nF** (and drop `C14`-or-`C56`) | 11 |
| `U8`–`U11` | PJ-320A-3P | **jack with switch contact** | 8, 14 |
| `RECOVER-HEADER` | 4-pin JST-XH | **5-pin JST-XH** | 12 |
| *new* | — | CEC TVS (≤10 pF, ~5 V WV) | 19 |
| *new* | — | 4 × 100 nF + 1 × 10 µF on `VDD-HP` | 6 |
| *new* | — | 2 × 1 kΩ (IR RX series) | 10 |
| *new* | — | up to 4 × 10 kΩ + 4 × 1 kΩ (jack detect) | 8 |
| `U12` | — | **genuine Nordic silicon only** — BOM §7 | — |
| `U6`, `Q1`, `Q2`, `R5`, `R8`–`R10`, `R43`, `C56` | CP2102N + auto-program | **deleted** — replaced by USB-Serial-JTAG | 20 |
| *new* | — | 100 nF on `VDD_USBPHY`; 2 test pads (`USB_DM`/`DP`); optional 3-pad UART0 header | 20 |

## Improvements added in rev 2026-08-15 — keep all of these

* `EN-DCDC` net fixed (17) · recovery UART connected (18) · antenna declared (4)
* `R42` 200 kΩ PCA9306 EN bias — TI reference circuit
* `R41` 10 kΩ C6 BOOT pull-up · `R61` 10 kΩ on `RESET-CTR` · `C58` 10 µF on the C6 rail
* `R44`/`R45` 10 kΩ IR TX base pull-downs (LEDs off during boot)
* `R51`–`R54`/`R56`/`R57` 10 kΩ pull-ups on all six flash pins (wrong rail though — item 16)
* `R43` 1 kΩ on CP2102N `RSTb` · `C59`/`C60` 22 pF feed-forward on the SY8089
* HDMI shell EP pads grounded · auto-program truth table · 4 × M2 mounting holes

---

## Sign-off checklist

- [ ] 1 — `IO0` moved to GPIO35, GPIO0 left NC
- [ ] 2 — Type A connector fitted, pins 13/14/17 remapped, TMDS shields grounded
- [ ] 3 — RX jack pin 2 on `+3V3`, `C9`/`C10` shunt to GND
- [x] 4 — nRF24 antenna on the schematic *(2026-08-15 — PCB trace; verify in layout)*
- [ ] 5 — `IRQ` off GPIO36
- [ ] 6 — `VDD-HP` decoupling added
- [ ] 7 — HPD intent decided and annotated (assert option if source-side use is wanted)
- [ ] 8 — Detect on GPIO 42–45, switched jacks specified
- [ ] 9 — `U13` = W25Q64
- [ ] 10 — 1 kΩ series on IR RX
- [ ] 11 — one 100 nF per button net
- [ ] 12 — 5-pin recovery header
- [x] 13 — PCA9306 pinout verified against TI datasheet *(2026-08-15 — symbol correct)*
- [ ] 14 — Jack datasheet verified (numbering **and** switch)
- [ ] 15 — C6 antenna variant chosen
- [ ] 16 — Sheet annotated (HDMI family, `RXD0`, LED current, C6 part number)
- [x] 17 — Pin-79 net renamed `EN-DCDC` *(2026-08-15 — verified both sides)*
- [x] 18 — Recovery UART connected end to end *(2026-08-15 — `RXT0` name still to tidy)*
- [ ] 19 — CEC TVS restored
- [ ] 20 — USB-Serial-JTAG on the USB-C (GPIO24/25), CP2102N + auto-program deleted, SCL → GPIO46, HPD → GPIO47, `VDD_USBPHY` powered, OTG test pads
