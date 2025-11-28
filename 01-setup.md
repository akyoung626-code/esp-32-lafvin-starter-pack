# 01 - Environment Setup & VS Code Workspace Structure

## Overview

This guide walks you through setting up a complete ESP32 development environment using VS Code and PlatformIO. By the end, you'll have a structured workspace ready for multiple ESP32 projects.

---

## Prerequisites

- Windows 10/11, macOS, or Linux
- VS Code installed (version 1.70+)
- USB cable (micro-USB or USB-C depending on your ESP32 board)
- Lavfin ESP32 Starter Kit

---

## Step 1: Install VS Code Extensions

1. Open VS Code
2. Press `Ctrl+Shift+X` (Windows/Linux) or `Cmd+Shift+X` (macOS)
3. Search and install these extensions:

| Extension | Publisher | Purpose |
|-----------|-----------|---------|
| PlatformIO IDE | PlatformIO | Core development environment |
| C/C++ | Microsoft | Syntax highlighting, IntelliSense |
| Serial Monitor | Microsoft | Alternative serial monitor |

4. Restart VS Code after installation
5. Wait for PlatformIO to complete initialization (bottom status bar shows progress)

---

## Step 2: Verify PlatformIO Installation

1. Click the PlatformIO icon (alien head) in the left sidebar
2. Select **PIO Home** → **Open**
3. Confirm you see the PlatformIO welcome screen
4. Navigate to **Platforms** → **Installed**
5. If Espressif 32 is not listed, install it:
   - Go to **Platforms** → **Embedded**
   - Search "Espressif 32"
   - Click **Install**

---

## Step 3: Create Workspace Structure

Create the following directory structure on your machine:

```
esp32-learning/
├── .vscode/
│   └── settings.json
├── projects/
│   ├── 01-blink/
│   ├── 02-serial-debug/
│   ├── 03-button-input/
│   ├── 04-sensor-basics/
│   ├── 05-wifi-connect/
│   ├── 06-web-server/
│   ├── 07-async-patterns/
│   ├── 08-sensor-stream/
│   └── 09-ota-updates/
├── shared/
│   ├── secrets.h.template
│   └── common_utils.h
└── README.md
```

### Terminal Commands (Run in Your Preferred Location)

```bash
mkdir -p esp32-learning/.vscode
mkdir -p esp32-learning/projects/{01-blink,02-serial-debug,03-button-input,04-sensor-basics,05-wifi-connect,06-web-server,07-async-patterns,08-sensor-stream,09-ota-updates}
mkdir -p esp32-learning/shared
touch esp32-learning/.vscode/settings.json
touch esp32-learning/shared/secrets.h.template
touch esp32-learning/README.md
```

---

## Step 4: Configure VS Code Workspace Settings

Create `.vscode/settings.json` with these contents:

```json
{
  "platformio-ide.customPATH": "",
  "platformio-ide.activateOnlyOnPlatformIOProject": true,
  "files.associations": {
    "*.h": "cpp",
    "*.ino": "cpp"
  },
  "editor.tabSize": 2,
  "C_Cpp.default.configurationProvider": "platformio.platformio-ide"
}
```

---

## Step 5: Create a Secrets Template

Create `shared/secrets.h.template`:

```cpp
// Copy this file to your project's include/ folder as secrets.h
// DO NOT commit secrets.h to version control

#ifndef SECRETS_H
#define SECRETS_H

#define WIFI_SSID "your-network-name"
#define WIFI_PASSWORD "your-password"

#endif
```

Add to your `.gitignore` (create in workspace root):

```
**/secrets.h
.pio/
.vscode/.browse*
```

---

## Step 6: Identify Your ESP32 Board

The Lavfin ESP32 Starter Kit typically includes the **ESP32-WROOM-32** module. Key specifications:

| Specification | Value |
|---------------|-------|
| Chip | ESP32-WROOM-32 |
| CPU | Dual-core Xtensa LX6, 240 MHz |
| Flash | 4 MB |
| SRAM | 520 KB |
| GPIO Pins | 34 (not all exposed) |
| ADC Channels | 18 |
| USB-Serial Chip | CP2102 or CH340 |

---

## Step 7: Install USB Drivers

### Windows
- **CP2102**: Download from [Silicon Labs](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers)
- **CH340**: Download from [WCH](http://www.wch-ic.com/downloads/CH341SER_EXE.html)

### macOS
- Usually works without additional drivers
- For CH340: `brew install --cask wch-ch34x-usb-serial-driver`

### Linux
- Drivers typically included in kernel
- Add user to dialout group: `sudo usermod -a -G dialout $USER`
- Log out and back in

---

## Step 8: Verify Hardware Connection

1. Connect ESP32 to computer via USB
2. Open Device Manager (Windows) or run `ls /dev/tty*` (macOS/Linux)
3. Note the COM port (Windows) or device path (macOS/Linux)

```
Windows: COM3, COM4, etc.
macOS:   /dev/cu.usbserial-*
Linux:   /dev/ttyUSB0 or /dev/ttyACM0
```

---

## Lavfin Starter Kit Component Reference

```
┌─────────────────────────────────────────────────────────┐
│                    STARTER KIT CONTENTS                  │
├─────────────────────────────────────────────────────────┤
│  ESP32 Dev Board          │  1x  │ Main microcontroller │
│  Breadboard (830 pts)     │  1x  │ Prototyping base     │
│  LEDs (various colors)    │  10x │ Output indicators    │
│  Resistors (220Ω, 10kΩ)   │  20x │ Current limiting     │
│  Push Buttons             │  5x  │ Input switches       │
│  DHT11 Sensor             │  1x  │ Temp/humidity        │
│  Photoresistor (LDR)      │  2x  │ Light sensing        │
│  Potentiometer            │  2x  │ Analog input         │
│  Jumper Wires (M-M, M-F)  │  40x │ Connections          │
│  USB Cable                │  1x  │ Power/programming    │
└─────────────────────────────────────────────────────────┘
```

---

## ESP32 Pinout Reference (Common Dev Board)

```
                    ┌─────────────────┐
                    │     USB PORT    │
                    └────────┬────────┘
              EN ──┤ 1      ─┴─     38 ├── GPIO23 (MOSI)
             VP ──┤ 2               37 ├── GPIO22 (SCL)
             VN ──┤ 3               36 ├── GPIO1  (TX0)
          GPIO34 ──┤ 4               35 ├── GPIO3  (RX0)
          GPIO35 ──┤ 5               34 ├── GPIO21 (SDA)
          GPIO32 ──┤ 6               33 ├── GND
          GPIO33 ──┤ 7               32 ├── GPIO19 (MISO)
          GPIO25 ──┤ 8               31 ├── GPIO18 (SCK)
          GPIO26 ──┤ 9               30 ├── GPIO5
          GPIO27 ──┤ 10              29 ├── GPIO17
          GPIO14 ──┤ 11              28 ├── GPIO16
          GPIO12 ──┤ 12              27 ├── GPIO4
             GND ──┤ 13              26 ├── GPIO0  (BOOT)
          GPIO13 ──┤ 14              25 ├── GPIO2  (LED)
           SD2  ──┤ 15              24 ├── GPIO15
           SD3  ──┤ 16              23 ├── SD1
           CMD  ──┤ 17              22 ├── SD0
           5V   ──┤ 18              21 ├── CLK
           3V3  ──┤ 19              20 ├── GND
                    └─────────────────┘

NOTES:
• GPIO34-39: Input only (no internal pull-up)
• GPIO6-11: Connected to flash (avoid using)
• GPIO2: Often connected to onboard LED
• GPIO0: Boot mode selection (hold LOW during reset to enter bootloader)
```

---

## Step 9: Create Your First PlatformIO Project

1. Open PIO Home
2. Click **New Project**
3. Configure:
   - **Name**: `01-blink`
   - **Board**: `Espressif ESP32 Dev Module`
   - **Framework**: `Arduino`
   - **Location**: Browse to `esp32-learning/projects/`
4. Click **Finish**

PlatformIO creates this structure:

```
01-blink/
├── .pio/              (build artifacts - gitignore)
├── .vscode/           (project-specific settings)
├── include/           (header files)
├── lib/               (project-specific libraries)
├── src/
│   └── main.cpp       (your code goes here)
├── test/              (unit tests)
└── platformio.ini     (project configuration)
```

---

## Step 10: Understand platformio.ini

Default configuration:

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
```

Recommended additions:

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino

; Serial monitor settings
monitor_speed = 115200

; Upload settings
upload_speed = 921600

; Build flags (optional)
build_flags = 
    -D DEBUG_MODE=1
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| PlatformIO not loading | Restart VS Code, check extension is enabled |
| "No such file or directory" | Ensure project path has no spaces |
| Platform install fails | Check internet connection, try `pio platform install espressif32` in terminal |
| IntelliSense not working | Run **PlatformIO: Rebuild IntelliSense Index** from command palette |

---

## Checklist

- [ ] VS Code installed and updated
- [ ] PlatformIO extension installed
- [ ] Espressif 32 platform installed
- [ ] Workspace directory structure created
- [ ] USB drivers installed for your OS
- [ ] ESP32 board detected on a COM/serial port
- [ ] First PlatformIO project created
- [ ] `secrets.h.template` created in shared folder
- [ ] `.gitignore` configured

---

## Next Steps

Proceed to `02-blink.md` to write and upload your first ESP32 program.
