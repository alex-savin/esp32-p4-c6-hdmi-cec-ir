# GPIO Assignment

Reconciled against `board/electronic-schema/SCH_Schematic1_2026-08-15.pdf`
(rev 2026-08-15) on 2026-08-15; first reconciled against rev 2026-08-12 on
2026-08-14. Every row below was read off the drawing pin by pin; the
**Schematic** column is what the board actually does, and disagreements with
the original specification are called out in **Status**.

> **For the board designer:** [Rework.md](Rework.md) consolidates every
> pre-fabrication change.

Legend: **✓** as specified · **✗** differs from the spec, schematic wins ·
**!** wired, but see [Improvements.md](Improvements.md) · **+** addition not in
the original spec

---

### ESP32-P4 (Main Application Processor) — `U3`, ESP32-P4NRW32X

| Feature | Spec | Schematic | Direction | Status | Notes / Requirements |
| --- | --- | --- | --- | --- | --- |
| **SDIO DAT0** | GPIO 14 | GPIO 14 | In/Out | ✓ | *Firmware-locked.* 51 kΩ pull-up `R31`. |
| **SDIO DAT1** | GPIO 15 | GPIO 15 | In/Out | ✓ | *Firmware-locked.* 51 kΩ pull-up `R32`. |
| **SDIO DAT2** | GPIO 16 | GPIO 16 | In/Out | ✓ | *Firmware-locked.* 51 kΩ pull-up `R33`. |
| **SDIO DAT3** | GPIO 17 | GPIO 17 | In/Out | ✓ | *Firmware-locked.* 51 kΩ pull-up `R34`. |
| **SDIO CLK** | GPIO 18 | GPIO 18 | Out | ✓ | *Firmware-locked.* 51 kΩ pull-up `R35` (present, not required). Length match to data lines. |
| **SDIO CMD** | GPIO 19 | GPIO 19 | In/Out | ✓ | *Firmware-locked.* 51 kΩ pull-up `R36`. |
| **Status LED (Blue)** | GPIO 20 | GPIO 20 | Out | ✓ | Active **high**. 2.2 kΩ `R48` → `YLED0805B` → GND. *(Rev 08-15: was 100 Ω `R27` — now ~0.6 mA, dim; flagged in Rework 16.)* |
| **Status LED (Green)** | GPIO 21 | GPIO 21 | Out | ✓ | Active **high**. 2.2 kΩ `R49` → `YLED0805YG` → GND. *(Same change.)* |
| **HDMI CEC** | GPIO 22 | GPIO 22 | In/Out | ! | Open-drain. 27 kΩ pull-up `R11`, `D2` BAT54 series. **Rev 08-15 deleted the `D3` ESD clamp — Improvements §2.11, Rework 19.** The **connector-side** pad moves when the Type A part lands — Improvements §2.1. |
| **HDMI DDC SDA** | GPIO 23 | GPIO 23 | In/Out | ✓ | 3.3 V side of `U7` PCA9306. 4.7 kΩ pull-up `R15`. *(Rev 08-15: `U7` biasing completed with `R42` 200 kΩ; pinout verified correct — Improvements §2.6.)* |
| **HDMI DDC SCL** | GPIO 24 | GPIO 24 | In/Out | ! | 3.3 V side of `U7` PCA9306. 4.7 kΩ pull-up `R14`. **Moves to GPIO 46 per Rework 20** — GPIO 24 becomes USB-Serial-JTAG D−. |
| **HDMI HPD** | GPIO 25 | GPIO 25 | In | ! | **Redesigned in rev 08-15:** net `HDMI-HPD-ESP`, the tap of `R60` 5.1 kΩ / `R59` 10 kΩ from `HDMI-HPD` — 3.31 V at 5 V HPD, sense-only (the board can no longer *assert* HPD; Improvements §2.5, Rework 7). **Moves to GPIO 47 per Rework 20** — GPIO 25 becomes USB-Serial-JTAG D+. |
| **IR Jack 1 — TX** | GPIO 26 | GPIO 26 | Out | ! | Drives a **low-side NPN** (`Q3` S8050) via 470 Ω `R20`; `R44` 10 kΩ base pull-down *(new 08-15 — LED off during boot)*. Emitter LED fed from `+VBUS` through 33 Ω `R18` on jack pin 4 *(was pin 2)*. |
| **IR Jack 1 — RX** | GPIO 27 | GPIO 27 | In | ! | 10 kΩ pull-up `R19`, signal now on jack pin 4 *(was 2)*. **No 1 kΩ series resistor**; receiver still has **no DC supply** — `C9` still in series, now on pin 2. Improvements §2.2, §3.2. |
| **IR Jack 2 — TX** | GPIO 28 | GPIO 28 | Out | ! | As Jack 1: `Q4`, `R23` 470 Ω, `R45` 10 kΩ *(new)*, `R21` 33 Ω. |
| **IR Jack 2 — RX** | GPIO 29 | GPIO 29 | In | ! | As Jack 1: `R22` 10 kΩ, `C10` still in series. |
| **nRF24 MOSI** | GPIO 30 | GPIO 30 | Out | ✓ | `U12` pin 4. |
| **nRF24 MISO** | GPIO 31 | GPIO 31 | In | ✓ | `U12` pin 5. |
| **nRF24 SCK** | GPIO 32 | GPIO 32 | Out | ✓ | `U12` pin 3. |
| **nRF24 CSN** | GPIO 33 | GPIO 33 | Out | ✓ | `U12` pin 2. |
| **nRF24 CE** | GPIO 34 | GPIO 34 | Out | ✓ | `U12` pin 1. |
| **nRF24 IRQ** | ~~GPIO 37~~ | **GPIO 36** | In | ✗ | Active low, `U12` pin 6. GPIO 37 is the console UART TX — the spec collided with it. |
| **Console UART TX** | U0TXD | **GPIO 37** | Out | ✓ | P4 default U0TXD → `CP2102N` pin 25 `RXD`. **Rework 20 deletes the CP2102N** — GPIO 37/38 become free (optional 3-pad ROM-log header). |
| **Console UART RX** | U0RXD | **GPIO 38** | In | ✓ | P4 default U0RXD ← `CP2102N` pin 26 `TXD`. Same. |
| **User Action Button (FSW)** | ~~GPIO 38~~ | **GPIO 41** | In | ✗ | Active low to GND, 10 kΩ pull-up `R26`, 1 µF `C16`. GPIO 38 is the console UART RX — the spec collided with it. |
| **Core DC-DC enable** | — | `EN_DCDC` (pin 79) | Out | ✓ | **Fixed in rev 08-15** — net `EN-DCDC` now reaches `U15`'s enable, verified at both ends. Improvements §2.9, Rework 17. |
| **Core DC-DC feedback** | — | `FB_DCDC` (pin 78) | Analog | + | Not in the original spec. Ties into the `R39`/`R40` divider so the P4 can trim `VDD-HP` at runtime. |
| **IR zone 1 TX jack detect** | — | **GPIO 42** (pin 83) | In | + | **Reserved, still unrouted in rev 08-15** — Improvements §6, Rework 8. |
| **IR zone 1 RX jack detect** | — | **GPIO 43** (pin 84) | In | + | **Reserved, still unrouted.** The one that actually matters — a receiver cannot otherwise be detected at all. |
| **IR zone 2 TX jack detect** | — | **GPIO 44** (pin 86) | In | + | **Reserved, still unrouted.** *(Pin 85 between them is `VDD_IO_5`, hence the skip.)* |
| **IR zone 2 RX jack detect** | — | **GPIO 45** (pin 87) | In | + | **Reserved, still unrouted.** |

#### Strapping, dedicated and unused pins

| Feature | Pin | Notes |
| --- | --- | --- |
| **Reset Button** | `CHIP_PU` (pin 103) | Net `EN`. Button + 10 kΩ `R46` *(rev 08-15: replaces `R24`, relocated)* + 1 µF `C14` **and** new 1 µF `C56` in parallel (≈2 µF — Rework 11); also driven by `Q1` of the auto-download circuit. |
| **Boot Button / strap** | **GPIO 0** (pin 104) | Net `IO0`. Button + 10 kΩ `R25` + 1 µF `C15`; also driven by `Q2`. **Still on the wrong pin in rev 08-15 — Rework 1.** |
| **C6 Reset Link** | **GPIO 54** (pin 98) | Net `RESET-CTR` → C6 `EN`, now with 10 kΩ pull-up `R61` *(new 08-15)*. *Firmware-locked.* |
| **SPI flash** | `FLASH_CS/Q/WP/HOLD/CK/D` (package pins 27–33) | `U13` W25Q32JVSSIQ, 4 MB, on the dedicated flash bank at `VDD-FLASH` (`C26` 100 nF at VCC). Rev 08-15 adds six 10 kΩ pull-ups `R51`–`R54`/`R56`/`R57` — referenced to `+3V3` instead of `VDD-FLASH`, see Rework 16. |
| **Crystal** | `XTAL_P` / `XTAL_N` (100/99) | `U14` 40 MHz, 10 pF loads `C27`/`C28`. |
| **PSRAM** | in-package | `NRW32X` = 32 MB PSRAM; `VDDO_PSRAM` LDO out, caps `C51`–`C55`. |
| **MIPI DSI / CSI** | package pins 34–48 (not GPIO 34–48) | **Unused.** No display or camera on this board. |
| **USB 2.0 PHY** | `USB_DM`/`USB_DP`/`VDD_USBPHY` (package pins 49–51) | **Unused in rev 08-15** — no USB-JTAG, no USB flashing; the CP2102N is the only path. **Rework 20 changes this:** `VDD_USBPHY` → `+3V3` + 100 nF, OTG pins 49/50 → test pads (DFU provision), and USB-Serial-JTAG arrives on GPIO 24/25. |
| **Unrouted GPIO** | 1–13, 39, 40, 46–53 | Free for expansion. **GPIO 42–45 claimed** by IR jack detect (Rework 8); **GPIO 46/47 earmarked** for the relocated SCL/HPD (Rework 20); GPIO 39/40 reserved for nRF24 `IRQ`/`CE` (Rework 5). |
| **⚠ Strapping pins** | **34, 35, 36, 37, 38** | See the warning below — three of these are currently misused. |

### ⚠ Strapping pins — the board currently gets these wrong

On the **ESP32-P4** the strapping pins are **GPIO 34, 35, 36, 37, 38**.
`GPIO35` selects boot mode: held low at reset, the chip enters the serial
bootloader. `GPIO36` must be high for that to work reliably.

**GPIO0 is not a strapping pin on the ESP32-P4.** It is on the ESP32 and
ESP32-S3, and this board's download circuit was drawn to that convention.

| Pin | Board uses it for | Problem |
| --- | --- | --- |
| GPIO 35 | *nothing — unrouted* | **This is the download-mode strap.** Floating, so it always reads high and the chip can never be put into download mode. |
| GPIO 0 | `BOOT` button + auto-download `Q2` | Not a strapping pin on this chip. Both do nothing. |
| GPIO 36 | nRF24 `IRQ` (active low) | Must be **high** to enter the bootloader; the nRF24 pulls it low when it has a pending interrupt. |
| GPIO 34 | nRF24 `CE` | Strapping pin driven as an output. Benign in practice, but worth knowing. |
| GPIO 37/38 | Console UART0 | Their default function. Fine. |

Consequence: **the P4 cannot be put into serial download mode as drawn**, and
with the USB PHY unconnected there is no USB-JTAG fallback. See
[Improvements.md §2.8](Improvements.md) — this is blocking and must be fixed
before fabrication.

Rework 20 (USB-Serial-JTAG on GPIO 24/25) will add a USB flash/debug path
that works from ROM — but the GPIO35 strap fix **stays mandatory**: it is the
only entry that survives firmware wedging the USB block or reassigning its
pins.

---

### ESP32-C6 (Network Co-Processor) — `U5`

*The C6 pins are fixed by the ESP-Hosted SDIO slave peripheral and by the
recovery requirement; none of them are a design choice.*

| Feature | Pin | Direction | Status | Notes / Requirements |
| --- | --- | --- | --- | --- |
| **SDIO CMD** | GPIO 18 (mod. pin 24) | In/Out | ✓ | ← P4 GPIO 19. |
| **SDIO CLK** | GPIO 19 (mod. pin 25) | In | ✓ | ← P4 GPIO 18. |
| **SDIO DAT0** | GPIO 20 (mod. pin 26) | In/Out | ✓ | ↔ P4 GPIO 14. |
| **SDIO DAT1** | GPIO 21 (mod. pin 27) | In/Out | ✓ | ↔ P4 GPIO 15. |
| **SDIO DAT2** | GPIO 22 (mod. pin 28) | In/Out | ✓ | ↔ P4 GPIO 16. |
| **SDIO DAT3** | GPIO 23 (mod. pin 29) | In/Out | ✓ | ↔ P4 GPIO 17. |
| **Main Reset** | `EN` (mod. pin 8) | In | ✓ | Net `RESET-CTR`, driven by P4 GPIO 54. |
| **Recovery UART TX** | GPIO 16 (mod. pin 31) | Out | ✓ | **Connected in rev 08-15** — net `TXD0` end to end. Improvements §2.10. |
| **Recovery UART RX** | GPIO 17 (mod. pin 30) | In | ✓ | **Connected in rev 08-15** — as net `RXT0` (the typo survives as a working net name; Rework 16). |
| **Recovery Boot** | GPIO 9 (mod. pin 23) | In | ✓ | Net `BOOT`, 10 kΩ pull-up `R41` *(new 08-15)*. To header pin 4. |

**Recovery header** (`B4B-XH-A(LF)(SN)`, 4-pin JST-XH): 1 = `TXD0`, 2 = `RXT0`
(functional; rename to `RXD0` pending), 3 = GND, 4 = `BOOT` — **all four
connect as of rev 08-15**. There is still **no `EN` pin on the header**
(Rework 12) — resetting the C6 during a manual recovery flash depends on the
P4 driving GPIO 54, or on power cycling the board.

---

### Changes in schematic rev 2026-08-15

Re-reconciled 2026-08-15. GPIO **numbers** are unchanged; circuits behind six
of them changed:

1. **`EN_DCDC` (pin 79) is fixed** — the board-killing net-label error is gone.
2. **C6 recovery UART (GPIO 16/17) now connects** to the header.
3. **HPD (GPIO 25)** moved to a sense-only divider net `HDMI-HPD-ESP`; the
   board can no longer assert HPD (Improvements §2.5).
4. **CEC (GPIO 22) lost its ESD clamp** — `D3` deleted, Rework 19.
5. **IR RX (GPIO 27/29)** re-wired but still unpowered at the jack
   (Improvements §2.2); TX gained base pull-downs `R44`/`R45`.
6. **LEDs (GPIO 20/21)** now behind 2.2 kΩ instead of 100 Ω.

Still wrong in rev 08-15: the boot strap (`IO0` on GPIO0 instead of GPIO35)
and nRF24 `IRQ`/`CE` on strapping pins GPIO36/34 — the strapping-pin warning
above stands unchanged.

### Changes made in the 2026-08-14 revision of this document

Three of the original assignments were wrong and have been corrected against the
schematic:

1. **nRF24 IRQ: GPIO 37 → GPIO 36.** GPIO 37 is the P4's default `U0TXD` and is
   used by the CP2102N link; the spec had two functions on one pin.
2. **User Action Button (FSW): GPIO 38 → GPIO 41.** Same collision, on `U0RXD`.
3. **Console UART** is now recorded as the concrete GPIO 37 / GPIO 38 rather
   than "default", so a future collision is visible in the table.

Corrected descriptions (the pin numbers were right, the circuits were not):

4. **IR TX** drives a low-side NPN, not a high-side MOSFET.
5. **HDMI HPD** is a bare resistor divider, not a level-shifter output.
6. **IR RX** has no 1 kΩ series resistor; that item is still open.

Added: `EN_DCDC` / `FB_DCDC`, and the strapping / dedicated / unused-pin table.
