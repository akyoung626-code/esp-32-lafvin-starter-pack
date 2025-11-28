# 03 - Serial Communication & Debugging

## Overview

Serial communication is your primary debugging tool for ESP32 development. This guide covers setup, usage patterns, and troubleshooting common issues.

---

## Prerequisites

- Completed `01-setup.md` and `02-blink.md`
- ESP32 connected via USB
- Working blink project

---

## Serial Communication Basics

### What is Serial?

Serial communication sends data one bit at a time over a single wire. The ESP32's USB connection provides a virtual serial port (COM port) for bidirectional communication with your computer.

```
┌─────────┐          USB Cable          ┌────────────┐
│  ESP32  │ ◄───────────────────────────► │ Computer   │
│         │   TX (transmit to computer) │            │
│         │   RX (receive from computer)│ Serial     │
│         │                             │ Monitor    │
└─────────┘                             └────────────┘
```

### Baud Rate

Baud rate = bits per second. Both devices must use the same rate.

| Baud Rate | Use Case |
|-----------|----------|
| 9600 | Legacy, very slow |
| 115200 | **Standard for ESP32** |
| 921600 | Fast debugging, may have issues on some systems |

---

## Project Setup

### platformio.ini

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
monitor_filters = 
    direct
    colorize
    time
```

### Monitor Filter Options

| Filter | Purpose |
|--------|---------|
| `direct` | Disable special character handling |
| `colorize` | Add ANSI colors to output |
| `time` | Prefix lines with timestamp |
| `log2file` | Save output to file |
| `esp32_exception_decoder` | Decode crash stack traces |

---

## Serial Monitor Setup

### Opening Serial Monitor

**Method 1: PlatformIO Toolbar**
- Click the plug icon (🔌) in the bottom toolbar

**Method 2: Command Palette**
- Press `Ctrl+Shift+P`
- Type "PlatformIO: Serial Monitor"
- Press Enter

**Method 3: Terminal Command**
```bash
pio device monitor
```

### Serial Monitor Controls

| Key | Action |
|-----|--------|
| `Ctrl+C` | Exit monitor |
| `Ctrl+T` | Enter command mode |
| `Ctrl+T` then `B` | Change baud rate |
| `Ctrl+T` then `H` | Show help |

---

## Basic Serial Output

### main.cpp - Simple Debug Output

```cpp
#include <Arduino.h>

void setup() {
  Serial.begin(115200);
  
  // Wait for serial connection (useful for debugging setup())
  while (!Serial) {
    delay(10);
  }
  
  Serial.println("ESP32 Started!");
  Serial.println("==================");
}

void loop() {
  Serial.println("Loop running...");
  delay(1000);
}
```

### Output Methods

| Method | Description | Example |
|--------|-------------|---------|
| `Serial.print()` | Print without newline | `Serial.print("Value: ");` |
| `Serial.println()` | Print with newline | `Serial.println(42);` |
| `Serial.printf()` | Formatted print (like C) | `Serial.printf("X=%d Y=%d\n", x, y);` |
| `Serial.write()` | Write raw bytes | `Serial.write(0x55);` |

---

## Debugging Patterns

### Pattern 1: State Markers

Track program flow:

```cpp
void setup() {
  Serial.begin(115200);
  Serial.println("[BOOT] Starting setup...");
  
  // Initialize WiFi
  Serial.println("[INIT] Configuring WiFi...");
  // ... wifi code ...
  Serial.println("[INIT] WiFi configured");
  
  Serial.println("[BOOT] Setup complete");
}
```

### Pattern 2: Variable Inspection

Print variable values:

```cpp
void loop() {
  int sensorValue = analogRead(34);
  float voltage = sensorValue * (3.3 / 4095.0);
  
  Serial.printf("[SENSOR] Raw: %d, Voltage: %.2fV\n", sensorValue, voltage);
  
  delay(500);
}
```

### Pattern 3: Conditional Debug Output

Use compile-time flags:

```cpp
#define DEBUG_ENABLED 1

#if DEBUG_ENABLED
  #define DEBUG_PRINT(x) Serial.println(x)
  #define DEBUG_PRINTF(fmt, ...) Serial.printf(fmt, __VA_ARGS__)
#else
  #define DEBUG_PRINT(x)
  #define DEBUG_PRINTF(fmt, ...)
#endif

void loop() {
  DEBUG_PRINTF("[DEBUG] Free heap: %d bytes\n", ESP.getFreeHeap());
  // In production, this line produces no code
}
```

### Pattern 4: Timing Analysis

Measure execution time:

```cpp
void loop() {
  unsigned long startTime = millis();
  
  // Code to measure
  complexOperation();
  
  unsigned long elapsed = millis() - startTime;
  Serial.printf("[TIMING] Operation took %lu ms\n", elapsed);
}
```

### Pattern 5: Memory Monitoring

Track heap usage:

```cpp
void printMemoryStats() {
  Serial.printf("[MEMORY] Free heap: %d bytes\n", ESP.getFreeHeap());
  Serial.printf("[MEMORY] Min free heap: %d bytes\n", ESP.getMinFreeHeap());
  Serial.printf("[MEMORY] Max alloc heap: %d bytes\n", ESP.getMaxAllocHeap());
}
```

---

## Serial Input (Reading from Computer)

### Basic Input Reading

```cpp
void loop() {
  if (Serial.available() > 0) {
    String input = Serial.readStringUntil('\n');
    input.trim();  // Remove whitespace
    
    Serial.printf("Received: '%s'\n", input.c_str());
    
    if (input == "status") {
      printStatus();
    } else if (input == "reboot") {
      ESP.restart();
    }
  }
}
```

### Command Parser Pattern

```cpp
void processCommand(String cmd) {
  cmd.trim();
  
  int spaceIndex = cmd.indexOf(' ');
  String command = (spaceIndex == -1) ? cmd : cmd.substring(0, spaceIndex);
  String args = (spaceIndex == -1) ? "" : cmd.substring(spaceIndex + 1);
  
  if (command == "led") {
    int state = args.toInt();
    digitalWrite(LED_PIN, state);
    Serial.printf("LED set to %d\n", state);
  } else if (command == "delay") {
    int ms = args.toInt();
    Serial.printf("Delaying %d ms...\n", ms);
    delay(ms);
    Serial.println("Done");
  } else {
    Serial.printf("Unknown command: %s\n", command.c_str());
  }
}
```

---

## ESP32-Specific Serial Features

### Multiple Serial Ports

ESP32 has three hardware UART ports:

| UART | Default TX | Default RX | Notes |
|------|------------|------------|-------|
| Serial (UART0) | GPIO1 | GPIO3 | USB connection |
| Serial1 (UART1) | GPIO10 | GPIO9 | Conflicts with flash |
| Serial2 (UART2) | GPIO17 | GPIO16 | Free to use |

### Using Serial2

```cpp
void setup() {
  Serial.begin(115200);   // USB debugging
  Serial2.begin(9600);    // External device
  
  Serial.println("Both serial ports ready");
}

void loop() {
  // Forward data between ports
  if (Serial2.available()) {
    Serial.write(Serial2.read());
  }
  if (Serial.available()) {
    Serial2.write(Serial.read());
  }
}
```

### Custom Pin Assignment

```cpp
// Assign Serial2 to different pins
Serial2.begin(9600, SERIAL_8N1, 26, 27);  // RX=26, TX=27
```

---

## Troubleshooting

### Problem: Serial Monitor Shows Garbage

```
Symptom: Ç×ú¦&ÏÎ...
```

| Cause | Solution |
|-------|----------|
| Baud rate mismatch | Match monitor to `Serial.begin()` value |
| Wrong encoding | Use UTF-8 in terminal |
| Boot messages | Normal at 74880 baud during boot |

### Problem: No Output At All

```
Troubleshooting flow:

1. Check USB Connection
   └─ Is the cable data-capable? (Some cables are charge-only)
   
2. Check COM Port
   └─ Is the correct port selected?
   └─ Run: pio device list
   
3. Check Code
   └─ Is Serial.begin() called?
   └─ Is baud rate correct?
   
4. Check Drivers
   └─ Are CP2102/CH340 drivers installed?
```

### Problem: COM Port Not Detected

**Windows:**
1. Open Device Manager
2. Look under "Ports (COM & LPT)"
3. If not listed, check "Other Devices" for unknown devices
4. Install appropriate driver (CP2102 or CH340)

**macOS:**
```bash
ls /dev/cu.*
# Should show something like /dev/cu.usbserial-0001
```

**Linux:**
```bash
ls /dev/ttyUSB* /dev/ttyACM*
# If nothing appears:
sudo dmesg | tail -20  # Check for USB events
```

### Problem: Permission Denied (Linux)

```bash
# Add user to dialout group
sudo usermod -a -G dialout $USER

# Log out and log back in, then verify:
groups  # Should include 'dialout'
```

### Problem: Upload Works But Monitor Shows Nothing

```
Common causes:

1. Serial.begin() not called
   └─ Add Serial.begin(115200) in setup()

2. Code crashes before Serial starts
   └─ Add delay(1000) after Serial.begin()
   
3. Infinite loop before Serial output
   └─ Check for blocking code in setup()
```

### Problem: "Resource Busy" Error

```
Symptom: Cannot open serial port - resource busy
```

Causes:
- Another program has the port open
- Previous monitor session didn't close cleanly

Solution:
```bash
# Linux/macOS: Find and kill process
lsof /dev/ttyUSB0
kill -9 <PID>

# Or just restart VS Code
```

---

## Boot Messages Reference

During startup, ESP32 outputs boot information at 74880 baud:

```
ets Jun  8 2016 00:22:57
rst:0x1 (POWERON_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
configsip: 0, SPIWP:0xee
clk_drv:0x00,q_drv:0x00,d_drv:0x00,cs0_drv:0x00
mode:DIO, clock div:2
load:0x3fff0030,len:1184
load:0x40078000,len:13132
load:0x40080400,len:3036
entry 0x400805e4
```

### Reset Reason Codes

| Code | Meaning |
|------|---------|
| POWERON_RESET | Normal power-on |
| SW_RESET | Software reset (ESP.restart()) |
| OWDT_RESET | Legacy watchdog reset |
| DEEPSLEEP_RESET | Wake from deep sleep |
| SDIO_RESET | SDIO reset |
| TG0WDT_SYS_RESET | Task watchdog reset (Group 0) |
| TG1WDT_SYS_RESET | Task watchdog reset (Group 1) |
| RTCWDT_SYS_RESET | RTC watchdog reset |
| INTRUSION_RESET | Brownout reset |

### Decoding Reset Reason in Code

```cpp
#include <rom/rtc.h>

void printResetReason() {
  int reason = rtc_get_reset_reason(0);  // Core 0
  
  Serial.print("Reset reason: ");
  switch (reason) {
    case 1:  Serial.println("POWERON_RESET"); break;
    case 3:  Serial.println("SW_RESET"); break;
    case 4:  Serial.println("OWDT_RESET"); break;
    case 5:  Serial.println("DEEPSLEEP_RESET"); break;
    case 6:  Serial.println("SDIO_RESET"); break;
    case 7:  Serial.println("TG0WDT_SYS_RESET"); break;
    case 8:  Serial.println("TG1WDT_SYS_RESET"); break;
    case 9:  Serial.println("RTCWDT_SYS_RESET"); break;
    case 10: Serial.println("INTRUSION_RESET"); break;
    default: Serial.println("Unknown"); break;
  }
}
```

---

## Exception Decoder

When ESP32 crashes, it outputs a stack trace. Configure PlatformIO to decode it:

### platformio.ini Addition

```ini
monitor_filters = esp32_exception_decoder
```

### Sample Crash Output (Decoded)

```
Guru Meditation Error: Core  1 panic'ed (LoadProhibited)
Core 1 register dump:
PC      : 0x400d1234  PS      : 0x00060030
...

Backtrace: 0x400d1234:0x3ffb1f10 0x400d5678:0x3ffb1f30

Decoded:
0x400d1234: myFunction at src/main.cpp:42
0x400d5678: loop at src/main.cpp:28
```

---

## Debug Output Levels Pattern

Implement log levels for production code:

```cpp
enum LogLevel { LOG_ERROR, LOG_WARN, LOG_INFO, LOG_DEBUG };
LogLevel currentLogLevel = LOG_DEBUG;

void logMessage(LogLevel level, const char* format, ...) {
  if (level > currentLogLevel) return;
  
  const char* prefix;
  switch (level) {
    case LOG_ERROR: prefix = "[ERROR]"; break;
    case LOG_WARN:  prefix = "[WARN] "; break;
    case LOG_INFO:  prefix = "[INFO] "; break;
    case LOG_DEBUG: prefix = "[DEBUG]"; break;
  }
  
  Serial.print(prefix);
  Serial.print(" ");
  
  va_list args;
  va_start(args, format);
  char buffer[256];
  vsnprintf(buffer, sizeof(buffer), format, args);
  va_end(args);
  
  Serial.println(buffer);
}

// Usage:
logMessage(LOG_INFO, "Sensor value: %d", sensorValue);
logMessage(LOG_ERROR, "Connection failed after %d attempts", retries);
```

---

## Checklist

- [ ] Serial.begin(115200) added to setup()
- [ ] Serial Monitor opens without errors
- [ ] Correct baud rate configured in platformio.ini
- [ ] Can see Serial.println() output
- [ ] Can send commands from Serial Monitor to ESP32
- [ ] Understand how to decode reset reasons
- [ ] Know how to troubleshoot common COM port issues
- [ ] Tested debug output patterns (markers, variable inspection)
- [ ] USB drivers verified working

---

## Next Steps

Proceed to `04-button-input.md` to learn digital input handling with debouncing.
