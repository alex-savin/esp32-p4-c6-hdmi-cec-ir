### ESP32-P4 (Main Application Processor)

| Feature | Pin Name / Assignment | Direction | Notes / Requirements |
| --- | --- | --- | --- |
| **SDIO DAT0** | GPIO 14 | In/Out | *Firmware-locked.* Mandatory 51kΩ pull-up. |
| **SDIO DAT1** | GPIO 15 | In/Out | *Firmware-locked.* Mandatory 51kΩ pull-up. |
| **SDIO DAT2** | GPIO 16 | In/Out | *Firmware-locked.* Mandatory 51kΩ pull-up. |
| **SDIO DAT3** | GPIO 17 | In/Out | *Firmware-locked.* Mandatory 51kΩ pull-up. |
| **SDIO CLK** | GPIO 18 | Out | *Firmware-locked.* Length match to data lines. |
| **SDIO CMD** | GPIO 19 | In/Out | *Firmware-locked.* Mandatory 51kΩ pull-up. |
| **C6 Reset Link** | GPIO 54 | Out | *Firmware-locked.* Drives C6 EN high. |
| **Status LED (Blue)** | GPIO 20 | Out | *Flexible.* Current-limiting resistor required. |
| **Status LED (Green)** | GPIO 21 | Out | *Flexible.* Current-limiting resistor required. |
| **HDMI CEC** | GPIO 22 | In/Out | *Flexible.* Route via RMT. 27kΩ pull-up & blocking diode. |
| **HDMI DDC SDA** | GPIO 23 | In/Out | *Flexible.* 3.3V side of the logic level shifter. |
| **HDMI DDC SCL** | GPIO 24 | Out | *Flexible.* 3.3V side of the logic level shifter. |
| **HDMI HPD** | GPIO 25 | In | *Flexible.* 3.3V side of the level shifter / voltage divider. |
| **IR Jack 1 (Power/TX)** | GPIO 26 | Out | *Flexible.* Drives the high-side MOSFET/switch. |
| **IR Jack 1 (RX Data)** | GPIO 27 | In | *Flexible.* Route via RMT. 10kΩ pull-up & 1kΩ series protection. |
| **IR Jack 2 (Power/TX)** | GPIO 28 | Out | *Flexible.* Drives the high-side MOSFET/switch. |
| **IR Jack 2 (RX Data)** | GPIO 29 | In | *Flexible.* Route via RMT. 10kΩ pull-up & 1kΩ series protection. |
| **nRF24 MOSI** | GPIO 30 | Out | *Flexible.* Standard SPI data out. |
| **nRF24 MISO** | GPIO 31 | In | *Flexible.* Standard SPI data in. |
| **nRF24 SCK** | GPIO 32 | Out | *Flexible.* Standard SPI clock. |
| **nRF24 CSN** | GPIO 33 | Out | *Flexible.* SPI Chip Select. |
| **nRF24 CE** | GPIO 34 | Out | *Flexible.* Chip Enable (RX/TX mode control). |
| **nRF24 IRQ** | GPIO 37 | In | *Flexible.* Interrupt pin. Active low. |
| **Console UART TX** | U0TXD (Default) | Out | Route to onboard CP2102N / CH340. |
| **Console UART RX** | U0RXD (Default) | In | Route to onboard CP2102N / CH340. |
| **Reset Button** | CHIP_PU / EN | In | Standard auto-download circuitry. |
| **Boot Button** | BOOT / Strapping | In | Standard auto-download circuitry. |

### ESP32-C6 (Network Co-Processor)

*Note: The C6 pins are strictly defined by the ESP-Hosted firmware and the recovery requirements.*

| Feature | Pin Name / Assignment | Direction | Notes / Requirements |
| --- | --- | --- | --- |
| **SDIO CMD** | GPIO 18 | In/Out | Connects to P4 GPIO 19. |
| **SDIO CLK** | GPIO 19 | In | Connects to P4 GPIO 18. |
| **SDIO DAT0** | GPIO 20 | In/Out | Connects to P4 GPIO 14. |
| **SDIO DAT1** | GPIO 21 | In/Out | Connects to P4 GPIO 15. |
| **SDIO DAT2** | GPIO 22 | In/Out | Connects to P4 GPIO 16. |
| **SDIO DAT3** | GPIO 23 | In/Out | Connects to P4 GPIO 17. |
| **Main Reset** | EN | In | Driven by P4 GPIO 54. |
| **Recovery UART TX** | GPIO 16 (Verify*) | Out | To recovery header. |
| **Recovery UART RX** | GPIO 17 (Verify*) | In | To recovery header. |
| **Recovery Boot** | GPIO 9 | In | To recovery header. 10kΩ pull-up to 3.3V. |
