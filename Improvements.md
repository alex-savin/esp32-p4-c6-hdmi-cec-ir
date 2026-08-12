Here is a detailed forensic schematic audit of the project file (`SCH_Schematic1_2026-08-12_2.pdf`).

Overall, the schematic is well-designed and captures the complex dual-MCU architecture, power rails, and HDMI protocol requirements. However, there is one major structural divergence in the **External Infrared (IR) section** regarding the number and type of 3.5mm jacks, along with a few minor component recommendations.

---

### 1. Subsystem-by-Subsystem Audit

#### **Power Infrastructure (3V3 & 1V1 Core Rails)**

* **3.3V Main Buck Regulator (`U1` - SY8089AAC):** Converts USB `+VBUS` (5V) to `+3V3`. The feedback network with $R_6 = 91\text{ k}\Omega$ and $R_7 = 20\text{ k}\Omega$ yields an output of $V_{\text{OUT}} = 0.6\text{ V} \times \left(1 + \frac{91\text{ k}\Omega}{20\text{ k}\Omega}\right) = 3.33\text{ V}$, which accurately matches standard 3.3V rail tolerances.


* **1.1V ESP32-P4 Core Regulator (`U15` - TLV62569DBVR):** The ESP32-P4 requires an external 1.1V core voltage for its high-performance digital core (`VDD-HP`). The feedback network with $R_{39} = 8.2\text{ k}\Omega$ and $R_{40} = 10\text{ k}\Omega$ yields $V_{\text{OUT}} = 0.6\text{ V} \times \left(1 + \frac{8.2\text{ k}\Omega}{10\text{ k}\Omega}\right) = 1.092\text{ V} \approx 1.1\text{ V}$, providing the necessary power to the `VDD-HP` pins.


* **USB Power Entry:** The `USBC1` connector properly ties `CC1` and `CC2` through independent 5.1kΩ pull-down resistors (`R1`, `R2`) to GND for standard 5V USB power negotiation. `USBLC6-2SC6` (`D1`) provides ESD protection on the USB data lines.



#### **Main Processor (`U3` - ESP32-P4NRW32X)**

* **SDIO Interconnect:** The 4-bit high-speed SDIO bus is correctly assigned to GPIOs 14–19 (`DAT0`–`DAT3`, `CLK`, `CMD`). All five mandatory 51kΩ pull-up resistors (`R31`–`R35`) are present on `CMD` and `DAT0`–`DAT3`.


* **C6 Reset Line:** GPIO 54 (`RESET-CTR`) directly controls the ESP32-C6 `EN` pin to manage hardware resets and OTA state transitions.


* **Decoupling:** Decoupling capacitors (`C32`–`C43`, 100nF each) are distributed across the `+3V3` and `VDD-HP` power pins.


* **Clock:** `U14` provides the external 40MHz crystal reference with 10pF load capacitors (`C27`, `C28`).



#### **Wireless Co-Processor (`U5` - ESP32-C6)**

* **SDIO Mapping:** Bus pins map directly to C6 GPIOs 18–23.


* **C6 Recovery Header (`RECOVER-HEADER`):** A 4-pin JST connector (`B4B-XH-A`) exposes `TXD0` (GPIO 16), `RXT0` (GPIO 17), `BOOT` (GPIO 9 with 10kΩ pull-up `R41`), and `GND` for recovering or flashing the C6 independently.



#### **HDMI Interface (CEC 2.0 & E-DDC)**

* **Level Translation (`U7` - PCA9306DCUR):** Converts 5V HDMI DDC lines (`SDA`/`SCL`) down to 3.3V logic (`HDMI-SDA`/`HDMI-SCL`). Low side is powered by `+3V3` with 4.7kΩ pull-ups (`R12`, `R13`), and high side is powered by `HDMI-5V` with 4.7kΩ pull-ups (`R14`, `R15`).


* **CEC Physical Layer:** CEC line includes the required BAT54 Schottky diode (`D2`) in series and a 27kΩ pull-up resistor (`R11`) to `+3V3`.


* **Hot Plug Detect (HPD):** Stepped down from 5V using a voltage divider ($R_{16} = 10\text{ k}\Omega$, $R_{17} = 6.8\text{ k}\Omega$) yielding $\approx 2.02\text{ V}$ for safe input to ESP32-P4 GPIO 25 (`HDMI-HPD`). ESD protection is provided by `D5` (`PESD3V3L1BA`).



#### **nRF24L01+ 2.4GHz Transceiver (`U12`)**

* **Control Lines:** Mapped to GPIOs 30–34 (`MOSI`, `MISO`, `SCK`, `CSN`, `CE`) and GPIO 37 (`IRQ`).


* **RF Matching Network:** Complete 50Ω matching balun network consisting of inductors `L2` (2.7nH), `L3` (8.2nH), `L4` (3.9nH) and capacitors `C20`–`C23` connected to a 16MHz crystal (`X1`).



#### **CP2102N USB-to-UART Bridge (`U6`)**

* **Leakage Prevention:** Includes the Silicon Labs recommended resistor divider ($R_8 = 21\text{ k}\Omega$, $R_9 = 47.5\text{ k}\Omega$) on `VREGIN`/`VBUS` to prevent back-powering when USB is connected while 3.3V power is off.


* **Auto-Download Circuit:** Standard dual NPN transistor setup (`Q1`, `Q2` S8050) driven by `DTR`/`RTS` to control `EN` and `IO0` (`BOOT`).



#### **Switches & User LEDs**

* **Tactile Buttons:** Includes `EN` (Reset), `BOOT`, and the extra `FSW` (User Action button connected to GPIO 38). Each button has a 10kΩ pull-up resistor and a 1µF debounce capacitor.


* **Status LEDs:** Blue LED (`GPIO 20`) and Green LED (`GPIO 21`) with 100Ω current-limiting resistors (`R27`, `R28`).



---

### 2. Discrepancies & Recommendations for the Designer

1. **IR Interface Architecture (Major Structural Difference):**
* **What was specified:** Two 3.5mm TRS jacks configured as **Universal/Flexi IR ports**, where each jack uses a high-side switch on the Tip (for firmware-selectable TX power or RX VCC) and a Ring input with series protection.


* **What is in the schematic:** The designer implemented **four separate 3.5mm jacks** (`U8`, `U9` for Zone 1 and `U10`, `U11` for Zone 2). `U8` and `U10` are dedicated TX Blaster jacks driven by NPN transistors (`Q3`, `Q4`), while `U9` and `U11` are dedicated RX Receiver jacks.


* **Action Item:** Clarify with the designer whether you prefer the current **4 dedicated single-purpose jacks** or want them converted to **2 Universal TRS jacks** to save board space.


2. **Missing Series Protection Resistors on IR RX Lines:**
* In the current RX circuit (`U9`/`U11`), the RX data line connects directly from the jack to the ESP32-P4 GPIO with only a 10kΩ pull-up (`R19`/`R22`) and 100nF filtering capacitor (`C9`/`C10`).


* **Action Item:** Ask the designer to insert a **1kΩ series protection resistor** between the jack's RX node and the ESP32-P4 GPIO pin to protect against static discharges or momentary shorts when plugging/unplugging cables.


3. **Debounce Capacitor Sizing on Buttons:**
* The tactile switches (`EN`, `BOOT`, `FSW`) use 1µF capacitors (`C14`, `C15`, `C16`) alongside 10kΩ pull-up resistors. This produces a time constant of $\tau = 10\text{ k}\Omega \times 1\text{ }\mu\text{F} = 10\text{ ms}$, which can feel sluggish during rapid button presses or auto-flashing.


* **Action Item:** Recommend lowering `C14`, `C15`, and `C16` to **100nF** ($\tau = 1\text{ ms}$) for cleaner response times.
