# 02 - Basic Blink + Understanding GPIO

## Overview

The blink project is the embedded equivalent of "Hello World." This guide explains GPIO fundamentals while building a reliable LED blink project.

---

## Prerequisites

- Completed `01-setup.md`
- ESP32 connected via USB
- LED and 220Ω resistor from kit

---

## GPIO Fundamentals

### What is GPIO?

GPIO = General Purpose Input/Output. These are programmable pins that can:
- **Output**: Send HIGH (3.3V) or LOW (0V) signals
- **Input**: Read HIGH or LOW states from external circuits

### ESP32 GPIO Numbering

ESP32 uses **logical GPIO numbers**, not physical pin positions on the board.

```
IMPORTANT: The number printed on the board (GPIO2, GPIO4, etc.)
is the logical number you use in code, NOT the physical pin position.
```

### GPIO Categories

| Category | GPIO Numbers | Notes |
|----------|--------------|-------|
| General Use | 2, 4, 5, 12-19, 21-23, 25-27, 32-33 | Safe for most projects |
| Input Only | 34, 35, 36, 39 | No output, no pull-up/down |
| Strapping Pins | 0, 2, 5, 12, 15 | Affect boot - use carefully |
| Flash Pins | 6, 7, 8, 9, 10, 11 | Reserved - do not use |
| UART0 | 1 (TX), 3 (RX) | Serial communication to USB |

### Strapping Pin Details

These pins have special meaning during boot:

| Pin | Boot Behavior |
|-----|---------------|
| GPIO0 | LOW = bootloader mode, HIGH = normal boot |
| GPIO2 | Must be LOW or floating during boot |
| GPIO5 | Affects SDIO timing |
| GPIO12 | Sets flash voltage (dangerous if wrong) |
| GPIO15 | Affects boot log output |

**Rule**: Avoid GPIO0, 2, 12, 15 for external circuits unless you understand boot implications.

---

## Project Setup

### Directory Structure

```
projects/01-blink/
├── include/
├── lib/
├── src/
│   └── main.cpp
├── platformio.ini
└── README.md
```

### platformio.ini

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
```

---

## Wiring Diagram

### Option A: Using Onboard LED (No Wiring)

Most ESP32 dev boards have an LED connected to GPIO2.

```
┌─────────────────────────┐
│      ESP32 Board        │
│                         │
│    [Onboard LED] ← GPIO2│
│                         │
└─────────────────────────┘

No external wiring needed.
```

### Option B: External LED on Breadboard

```
ESP32                    Breadboard
┌─────┐                  ┌──────────────────┐
│     │                  │                  │
│ GPIO4├──────────────────┤─── [220Ω] ──────│
│     │                  │              │   │
│     │                  │          [LED]   │
│     │                  │         (+) (-)  │
│     │                  │              │   │
│ GND ├──────────────────┤──────────────┘   │
│     │                  │                  │
└─────┘                  └──────────────────┘

LED Orientation:
• Longer leg = Anode (+) → connects through resistor to GPIO
• Shorter leg = Cathode (-) → connects to GND
• Flat side of LED casing indicates cathode
```

### Why 220Ω?

```
LED Forward Voltage: ~2V
ESP32 GPIO Output:   3.3V
Desired Current:     ~5mA (safe, visible)

R = (3.3V - 2V) / 0.005A = 260Ω

220Ω is close enough and common in kits.
Safe range: 150Ω to 330Ω
```

---

## Code Explanation

### Minimal Blink (main.cpp)

```cpp
#include <Arduino.h>

const int LED_PIN = 2;  // Change to 4 for external LED

void setup() {
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_PIN, HIGH);
  delay(1000);
  digitalWrite(LED_PIN, LOW);
  delay(1000);
}
```

### Code Breakdown

| Line | Purpose |
|------|---------|
| `#include <Arduino.h>` | Required for PlatformIO (not needed in Arduino IDE) |
| `const int LED_PIN = 2` | Define pin number as constant for easy changes |
| `pinMode(LED_PIN, OUTPUT)` | Configure pin as output (can drive HIGH/LOW) |
| `digitalWrite(LED_PIN, HIGH)` | Set pin to 3.3V (LED on) |
| `digitalWrite(LED_PIN, LOW)` | Set pin to 0V (LED off) |
| `delay(1000)` | Pause execution for 1000 milliseconds |

---

## Pin Mode Options

```cpp
pinMode(pin, MODE);
```

| Mode | Description |
|------|-------------|
| `OUTPUT` | Drive pin HIGH or LOW |
| `INPUT` | Read external voltage (floating) |
| `INPUT_PULLUP` | Read with internal pull-up resistor (~45kΩ to 3.3V) |
| `INPUT_PULLDOWN` | Read with internal pull-down resistor (~45kΩ to GND) |

---

## Modifying Pin Assignments

### Step-by-Step Process

1. **Choose a safe GPIO**: Refer to GPIO Categories table above
2. **Update the constant**: Change `const int LED_PIN = X;`
3. **Rewire physically**: Move the wire to the new GPIO pin
4. **Recompile and upload**: PlatformIO will rebuild

### Example: Moving to GPIO18

```cpp
const int LED_PIN = 18;  // Changed from 2 to 18
```

Then physically connect your LED circuit to the GPIO18 pin on the board.

---

## Build and Upload

### Using VS Code/PlatformIO

1. Open the `01-blink` project folder in VS Code
2. Click the checkmark icon (✓) in the bottom toolbar to **Build**
3. Click the arrow icon (→) to **Upload**
4. Watch the terminal for progress

### Expected Terminal Output

```
Processing esp32dev (platform: espressif32; board: esp32dev)
...
Linking .pio/build/esp32dev/firmware.elf
Building .pio/build/esp32dev/firmware.bin
...
Uploading .pio/build/esp32dev/firmware.bin
...
Hard resetting via RTS pin...
========================= [SUCCESS] =========================
```

---

## Common Beginner Mistakes

### 1. Wrong Pin Number

**Symptom**: LED doesn't blink  
**Cause**: Using physical pin position instead of GPIO number  
**Fix**: Always use the GPIO number printed on the board

### 2. Missing Resistor

**Symptom**: LED is very bright then dims/dies, or GPIO stops working  
**Cause**: Excessive current damages LED or GPIO  
**Fix**: Always use a current-limiting resistor (150-330Ω)

### 3. LED Polarity Reversed

**Symptom**: LED never lights up  
**Cause**: Anode and cathode swapped  
**Fix**: Long leg (or round side) connects to resistor/GPIO

### 4. Using Flash Pins

**Symptom**: Board crashes, won't boot, or behaves erratically  
**Cause**: GPIO 6-11 are used by internal flash memory  
**Fix**: Never use GPIO 6, 7, 8, 9, 10, or 11

### 5. Strapping Pin Conflicts

**Symptom**: Board won't boot normally, enters bootloader  
**Cause**: External circuit holds GPIO0 LOW during boot  
**Fix**: Add pull-up resistor or use different pin

### 6. Forgetting `#include <Arduino.h>`

**Symptom**: PlatformIO shows compile errors for `pinMode`, `digitalWrite`  
**Cause**: Arduino IDE auto-includes this; PlatformIO doesn't  
**Fix**: Add `#include <Arduino.h>` at the top

### 7. Upload Fails Repeatedly

**Symptom**: "Failed to connect" or timeout errors  
**Cause**: Board not in bootloader mode  
**Fix**: Hold BOOT button, press EN/RESET, release BOOT, then upload

---

## Understanding `delay()` Problems

The `delay()` function blocks all execution:

```
Timeline with delay():
─────────────────────────────────────────────────>
│ LED ON │ [BLOCKED 1s] │ LED OFF │ [BLOCKED 1s] │

During blocked time:
• Cannot read sensors
• Cannot respond to buttons
• Cannot process network
• Cannot do ANYTHING
```

This is fine for learning but problematic in real applications. See `08-async-patterns.md` for alternatives.

---

## Experiments to Try

### Experiment 1: Change Blink Speed

Modify delays to create different patterns:

```cpp
// Fast blink
delay(100);

// Slow blink
delay(2000);

// Asymmetric (long on, short off)
digitalWrite(LED_PIN, HIGH);
delay(2000);
digitalWrite(LED_PIN, LOW);
delay(200);
```

### Experiment 2: Multiple LEDs

Wire LEDs to GPIO4, GPIO16, GPIO17:

```cpp
const int LEDS[] = {4, 16, 17};
const int NUM_LEDS = 3;

void setup() {
  for (int i = 0; i < NUM_LEDS; i++) {
    pinMode(LEDS[i], OUTPUT);
  }
}

void loop() {
  for (int i = 0; i < NUM_LEDS; i++) {
    digitalWrite(LEDS[i], HIGH);
    delay(200);
    digitalWrite(LEDS[i], LOW);
  }
}
```

### Experiment 3: PWM Fading

ESP32 supports hardware PWM (LED dimming):

```cpp
const int LED_PIN = 2;
const int PWM_CHANNEL = 0;
const int PWM_FREQ = 5000;
const int PWM_RESOLUTION = 8;  // 0-255

void setup() {
  ledcSetup(PWM_CHANNEL, PWM_FREQ, PWM_RESOLUTION);
  ledcAttachPin(LED_PIN, PWM_CHANNEL);
}

void loop() {
  // Fade in
  for (int duty = 0; duty <= 255; duty++) {
    ledcWrite(PWM_CHANNEL, duty);
    delay(10);
  }
  // Fade out
  for (int duty = 255; duty >= 0; duty--) {
    ledcWrite(PWM_CHANNEL, duty);
    delay(10);
  }
}
```

---

## GPIO Electrical Specifications

| Parameter | Value |
|-----------|-------|
| Logic HIGH output | ~3.3V |
| Logic LOW output | ~0V |
| Maximum current per pin | 12mA (recommended), 40mA (absolute max) |
| Maximum total GPIO current | 1.2A across all pins |
| Input HIGH threshold | >2.475V (0.75 × 3.3V) |
| Input LOW threshold | <0.825V (0.25 × 3.3V) |
| Internal pull-up | ~45kΩ |
| Internal pull-down | ~45kΩ |

---

## Checklist

- [ ] Understand difference between GPIO number and physical pin position
- [ ] Identified onboard LED pin (usually GPIO2)
- [ ] Wired external LED with resistor correctly
- [ ] Built and uploaded blink code successfully
- [ ] LED blinks at expected rate
- [ ] Tested changing the pin assignment
- [ ] Understand why delay() is problematic for complex applications
- [ ] Know which GPIO pins to avoid (6-11, and strapping pins with caution)

---

## Next Steps

Proceed to `03-serial-debug.md` to learn debugging techniques using the Serial Monitor.
