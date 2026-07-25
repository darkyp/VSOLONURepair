# ZTE / V-SOLUTION ONU Hardware Repair & Firmware Backup

A complete teardown, hardware diagnostic, repair guide, and firmware dump analysis for V-SOLUTION / ZTE-derived GPON ONUs experiencing boot loop / brick issues caused by a bootstrap pin soldering defect.

---

## 📌 Project Overview

This repository documents the root cause analysis and successful repair of a batch of bricked GPON Optical Network Units (ONUs). All units exhibited a boot-looping failure mode on startup. 

Through serial debugging, thermal imaging, and microscopic inspection, the issue was traced to a **manufacturing reflow defect on the bootstrap logic circuitry** (LEDs and series resistor network pulling down configuration pins on the main SoC).

---

## 🛠 Hardware Specifications & Notable Components

* **Main SoC:** ZTE ZX279125 series GPON ONU Processor
* **RF / CATV Driver:** **ZD2018C** (GaAs MMIC CATV Amplifier — runs continuously at $\sim 83^\circ\text{C}$)
* **EEPROM / SPI Flash:** 
  * Main SPI NOR Flash (8 MB)
  * **24C08** (I2C EEPROM — likely stores board MAC / PON serial configuration)
  * **25L95** (QFN package SPI Flash/EEPROM)

---

## 🔍 Failure Analysis & Root Cause

### 1. Serial Console Observations
Capturing the UART boot log via PuTTY revealed that the bootloader was failing during early hardware initialization or looping continuously before loading the main Linux kernel. The UART headers are clearly labeled and seen on the board. Working at 115200bps, standard polarity (i.e. idle high). The boot process started with ``` SPI NAND: p_s 512 p_n 32 ``` then hanged while the flash IC is a NOR type. The correct start should be ``` SPI NOR ```. When hanged a scope showed that the SoC was continuously sending the SPI NAND command for read status register to which (of course) the NOR IC was not responding.

### 2. Root Cause: Solder Joint Fracture on Bootstrap Pins
The boot configuration pins of the ZTE SoC (that select the type of flash attached) rely on external pull-down/pull-up networks tied to status LEDs and series resistors. Microscopic inspection revealed cold solder joints and fractured connections across these surface-mount components.

* **Manufacturing Defect:** Heavy copper ground planes (32 mil traces / copper pours) without proper thermal relief spokes created a thermal mass imbalance during factory reflow. 
* **Mechanism:** The ground side of the LED/resistor pads did not reach proper wetting temperature, leading to weak joints. Over time, localized thermal expansion caused the solder joints to shear, leaving the boot pins floating.

---

## 🔧 Repair Procedure

1. **Visual & Microscopic Inspection:** Inspect the series resistors and status LED pads near the SoC for tombstoning or micro-cracks.
2. **Reflow / Resolder:** Apply flux and manually reflow/refresh the solder joints on all bootstrap series resistors and LED network pads.
3. **Optics Inspection:** If the unit reports **LOS (Loss of Signal)** after boot, inspect the internal SC/FC coupler. Ensure the ceramic alignment sleeve and optical ferrule are fully seated and clean.

---

## 📁 Repository Contents

``` text
├── docs/
│   ├── photos/              # Box, PCB layout, and component shots
│   ├── microscope/          # High-magnification shots of cold solder joints & ICs
│   ├── thermal/             # Thermal imaging (showing ZD2018C & SoC thermals)
│   └── logs/                # PuTTY serial boot logs & `help` command outputs
├── firmware/
│   └── D431_full.bin        # 8MB full SPI MTD0 dump (U-Boot + Kernel + RootFS)
└── README.md
```

---

## 💻 Software & Network Access

Once booted, local root access can be obtained over Telnet:

### Firewall Unlock
By default, the device listens on **Port 23 (Telnet)**, but incoming connections may be blocked by internal `iptables` rules. Unlocking the input chain over serial drops straight to a shell:

``` sh
iptables -I INPUT -j ACCEPT
```

### Authentication Details
* **Serial UART:** `root` / `root`
* **Telnet:** `admin` / `stdONUi0i` *(Spawns direct root shell `UID 0`)*

### Firmware Dumping via TFTP
Since `dd`, `nc`, and `curl` are missing from the BusyBox build, and `cat /dev/mtd0` causes an Out-Of-Memory (OOM) panic, dump partitions directly to a local Windows host (running Tftpd32/64) using streaming TFTP:

``` sh
tftp -p -l /dev/mtd0 -r D431_full.bin <YOUR_PC_IP>
```

---

## 🌡️ Thermal Notes

* **ZD2018C (CATV IC):** Operates around **$83^\circ\text{C}$** in normal operation. This is normal behavior for this Class-A GaAs driver chip, but it can be disabled via the Web UI if RF/CATV is not used in your deployment.
* **SoC:** Normal operating temperature sits around **$63^\circ\text{C}$**.

---

## 🤝 Acknowledgments

* Technical documentation, diagnostic synthesis, and markdown structuring assisted by Gemini AI.
