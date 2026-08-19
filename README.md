## netgear-wgt424v30-firmware-extraction
Hardware extraction, UART reconnaissance, and flash dump triage on a Netgear WGT624v3 router

**Device Specifications and Overview
* Model: Netgear WGT624v3 
* FCC ID: PY3WGTY624V3
* IC ID: 4054A-WGT-624V3
* Board Revision: U12H040 REV:2 (PCB Date Code ~2005)

![Router Front](photos/router_front.jpg)
![Router Back](photos/router_back.jpg)
![Bottom Label](photos/bottom_label.jpg)

___

## 1. Disassembly

1. Peeled back the four rubber feet on the bottom of the router and removed them to expose four Torx head screws.
2. Removed all 4 screws and shell comes apart in 3 pieces with no clips.
3. Lifted the green motherboard out of the chassis.
4. Carefully unclipped and removed the top metal RF sheild cover. 

![Disassembly Step 1](photos/disassembly_1.jpg)
![PCB Underside](photos/disassembly_2.jpg)
![PCB Component Side](photos/disassembly_3.jpg)


___

## 2. Integrated Circuit Identification

1. System on Chip: Atheros AR2316A-00, BGA Chip, Single Chip MIPS32 Processor with integrated 802.11b/g MAC/baseband.
2. System Memory (RAM): Samsung K4S281632F-TC75, 54-pin TSOP-II, 128Mbit (16MB) SDRAM (133 MHz, CL3)
3. Flash Storage: STMicroelectronics 25P16V6P, 16-pin SOIC, 16Mbit (2MB) SPI NOR Flash memory, Holds the bootloader.
4. Ethernet Switch: Marvell 88E6060-RCJ, 128 pin PQFP, 6 Port Fast Ethernet switch controller.
5. Voltage Regulation: Richtek RT9164A, SOT-223, Linear low dropout (LDO)voltage regulator

![Atheros SoC and Flash Chip](photos/chip_atheros_AR2316A.jpg)
![Samsung RAM and Power Section](photos/chip_samsung_K4S281632F.jpg)


##3. UART Serial Reconnaissance

12-pin (2x6) through-hole header labeled **JP1** positioned between the RF shield and the Ethernet jacks. This multi-function header serves as the board's combined JTAG and UART diagnostic interface.

![JP1 Header Overview](photos/UART_JP1_pins.jpg)

###Step 1: Signal Probing

Using a Klein Tools MM600 digital multimeter referenced against the chassis ground plane (Pin 11), individual pins on the JP1 header were mapped:

1. Pin 1 (3.3V VCC): Steady 3.3V DC logic high (Left disconnected to prevent back-powering the board).
2. Pin 9 (Router TX): Measured ~3.3V at idle, actively fluctuating down to ~2.8V–3.1V during boot as serial logs transmitted.
3. Pin 11 (GND): Measured 0V with direct continuity (0.2 Ω) to board ground shielding.

![VCC Logic Rail Measurement (3.341V)](photos/UART_RX.jpg)
![Probing Pin 1 vs Ground Pin 11](photos/UART_RX_1.jpg)
![Router TX Active Line Measurement (3.267V)](photos/UART_TX.jpg)
![Probing Pin 9 TX vs Ground Pin 11](photos/UART_TX_1.jpg)

###Step 2: Hardware Serial Interfacing

Connected a CP2102 USB-to-UART bridge directly to the JP1 pin headers using Dupont jumper wires:
* **Adapter RXD** $\rightarrow$ **JP1 Pin 9 (Router TX)**
* **Adapter GND** $\rightarrow$ **JP1 Pin 11 (GND)**
* **Adapter TXD** $\rightarrow$ **Router RX**
* **VCC** left disconnected.

![CP2102 Wiring to JP1 Header](photos/UART_USB_jumpers_1.jpg)

---

###Step3: Terminal Capture

Serial capture was performed using 'picocom' on Linux

Connecting at standard 9600 baud (`picocom -b 9600 /dev/ttyUSB0`) and testing other non-standard rates produced severe framing errors and unreadable byte noise across the terminal buffer.

picocom -b 9600 /dev/ttyUSB0 

![Connecting via 15200 Baud](photos/UART_console_15200.jpg)
![Connecting via 15200 Baud Error](photos/UART_console_15200_1.jpg)
![Connecting via 9600 Baud](photos/UART_console_9600.jpg)
![Connecting via 9600 Baud Success](photos/UART_console_9600_1.jpg)



## 4. Hot-Air Chip-Off Extraction

Because serial UART access was limited to a one-way logging channel, physical flash extraction was used to acquire the raw firmware image directly.

### Desoldering & Pad Cleaning Process
1. Target Package: STMicroelectronics 25P16V6P (16-pin SOIC SPI NOR Flash).
2. Technique: Applied flux across both banks of 8 pins. Used a hot air rework station with circular airflow to heat all 16 leads evenly until solder liquified, lifted the chip vertically with tweezers to avoid damaging traces.
3. Footprint Preparation:Cleaned remaining solder from the 16 surface-mount pads using 99% isopropyl alcohol, checking no short circuits or solder bridges remained on the PCB.

![Board and Desoldered Flash Chip](photos/SOIC16_removed_1.jpg)
![Desoldered ST 25P16V6P Flash Chip](photos/SOIC16_removed_2.jpg)
![Cleaned PCB Footprint with Intact Pads](photos/SOIC16_removed.jpg)

---

## 5. Flash Memory Acquisition (XGecu T48 & Xgpro)

The extracted chip was read using an external hardware programmer to pull the complete flash memory buffer.

### Hardware Programmer Interfacing
* **Programmer:** XGecu T48 High-Speed Universal Programmer.
* **Socket Adapter:** SOP16/SOIC-16 to DIP ZIF (Zero Insertion Force) clamshell socket adapter.
* **Alignment:** Aligned Pin 1 (dot indicator on package) with the socket orientation markings and locked the clamshell retaining frame.

![ST 25P16 Seated in T48 Clamshell Adapter](photos/SOIC16_T48_flash_read.jpg)

### Software Readout & Acquisition
* **Software:** Xgpro Programmer Suite.
* **Device Selected:** `ST [ST25P16 @SOIC16]` (Memory Size: `0x200000` bytes / 2,097,152 bytes).
* **Read Execution:** Read the entire SPI NOR flash memory array into the local software buffer and saved the raw binary dump to disk (`wgt624v3_flash_dump.bin`).

![Xgpro Software Flash Readout Buffer](photos/xgpro_firmware.jpg)

---

## 6. Buffer Correction & Cryptographic Verification

### Resolving the 272-Byte Padding Anomaly
The raw programmer dump produced **2,097,424 bytes**, exceeding the physical 2MB hardware boundary by 272 bytes (`0x110` in hex) due to status and OTP register metadata appended to the buffer by the programmer software.

The extra footer bytes were trimmed to match the exact hardware capacity (2,097,152 bytes / 2MB):

```bash
# Trim the trailing 272 bytes to create a 1:1 hardware flash image
head -c 2097152 wgt624v3_flash_dump.bin > wgt624v3_clean.bin

# Verify exact byte count
ls -l wgt624v3_clean.bin
# -rw-r--r-- 1 user user 2097152 Aug 18 20:15 wgt624v3_clean.bin