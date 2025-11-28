# 04 - Button Input Project

## Overview

This guide covers reading digital inputs from push buttons, including debouncing strategies and interrupt-based approaches.

---

## Prerequisites

- Completed `01-setup.md` through `03-serial-debug.md`
- Push button from starter kit
- 10kΩ resistor (optional, can use internal pull-up)

---

## Button Basics

### How Buttons Work

A push button is a momentary switch that connects two points when pressed.

```
Button States:

NOT PRESSED:                PRESSED:
    │                           │
    ○ ← contact open           ═○═ ← contact closed
    │                           │
    
Circuit is OPEN             Circuit is CLOSED
```

### The Floating Pin Problem

An unconnected GPIO pin "floats" between HIGH and LOW, reading random values.

```
Floating pin behavior:

     3.3V ─────────────────────
             ╱╲    ╱╲    ╱╲
Voltage     ╱  ╲  ╱  ╲  ╱  ╲    ← Random noise
           ╱    ╲╱    ╲╱    ╲
     0V ─────────────────────────

GPIO reads: 1, 0, 1, 1, 0, 1, 0, 0... (unpredictable)
```

Solution: Use a pull-up or pull-down resistor.

---

## Wiring Options

### Option A: External Pull-Up Resistor

```
       3.3V
         │
        ┌┴┐
        │ │ 10kΩ Pull-up
        └┬┘
         │
GPIO ────┼──────┤ │
         │     BUTTON
         │      │ │
        GND ────┴─┘

Button NOT pressed: GPIO reads HIGH (3.3V through resistor)
Button pressed: GPIO reads LOW (connected to GND)
```

### Option B: Internal Pull-Up (Recommended)

No external resistor needed - ESP32 has built-in pull-ups.

```
GPIO ──────────┤ │
              BUTTON
               │ │
GND ───────────┴─┘

Configure in code: pinMode(BUTTON_PIN, INPUT_PULLUP);
```

### Option C: External Pull-Down Resistor

```
GPIO ────┬──────┤ │
         │     BUTTON
        ┌┴┐     │ │
        │ │    3.3V
        └┬┘
         │
        GND

Button NOT pressed: GPIO reads LOW (0V through resistor)
Button pressed: GPIO reads HIGH (connected to 3.3V)
```

---

## Project Structure

```
projects/03-button-input/
├── include/
│   └── button.h
├── lib/
├── src/
│   └── main.cpp
└── platformio.ini
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

## The Bounce Problem

Mechanical buttons don't transition cleanly:

```
Ideal button press:

HIGH ────────┐
             │
             │
LOW          └────────────────────

Actual button press (bouncing):

HIGH ────┐ ┌┐ ┌┐
         │ ││ ││
         └─┘└─┘└──────────────────
LOW                    

Duration: 1-50 milliseconds of bouncing
```

Without debouncing, a single press might register as 5-10 presses.

---

## Debouncing Strategies

### Strategy 1: Delay-Based (Simple, Blocking)

```cpp
#include <Arduino.h>

const int BUTTON_PIN = 4;
const int LED_PIN = 2;
const unsigned long DEBOUNCE_MS = 50;

bool ledState = false;

void setup() {
  Serial.begin(115200);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  if (digitalRead(BUTTON_PIN) == LOW) {  // Pressed (active LOW)
    delay(DEBOUNCE_MS);                   // Wait for bounce to settle
    
    if (digitalRead(BUTTON_PIN) == LOW) { // Still pressed?
      ledState = !ledState;
      digitalWrite(LED_PIN, ledState);
      Serial.printf("Button pressed, LED: %s\n", ledState ? "ON" : "OFF");
      
      // Wait for release
      while (digitalRead(BUTTON_PIN) == LOW) {
        delay(10);
      }
      delay(DEBOUNCE_MS);  // Debounce release
    }
  }
}
```

**Pros**: Simple to understand  
**Cons**: Blocks execution, can't do other tasks

---

### Strategy 2: Timestamp-Based (Non-Blocking)

```cpp
#include <Arduino.h>

const int BUTTON_PIN = 4;
const int LED_PIN = 2;
const unsigned long DEBOUNCE_MS = 50;

bool ledState = false;
bool lastButtonState = HIGH;
bool buttonState = HIGH;
unsigned long lastDebounceTime = 0;

void setup() {
  Serial.begin(115200);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  bool reading = digitalRead(BUTTON_PIN);
  
  // If reading changed, reset debounce timer
  if (reading != lastButtonState) {
    lastDebounceTime = millis();
  }
  
  // If reading stable for DEBOUNCE_MS
  if ((millis() - lastDebounceTime) > DEBOUNCE_MS) {
    // If button state actually changed
    if (reading != buttonState) {
      buttonState = reading;
      
      // Only act on press (HIGH to LOW transition)
      if (buttonState == LOW) {
        ledState = !ledState;
        digitalWrite(LED_PIN, ledState);
        Serial.printf("Button pressed, LED: %s\n", ledState ? "ON" : "OFF");
      }
    }
  }
  
  lastButtonState = reading;
  
  // Other tasks can run here without being blocked
}
```

**Pros**: Non-blocking, can multitask  
**Cons**: More variables to track

---

### Strategy 3: Interrupt-Based

Interrupts respond immediately to pin changes, useful for time-critical applications.

```cpp
#include <Arduino.h>

const int BUTTON_PIN = 4;
const int LED_PIN = 2;
const unsigned long DEBOUNCE_MS = 200;

volatile bool buttonPressed = false;
volatile unsigned long lastInterruptTime = 0;

// Interrupt Service Routine - must be IRAM_ATTR on ESP32
void IRAM_ATTR handleButtonPress() {
  unsigned long currentTime = millis();
  
  // Debounce in ISR
  if (currentTime - lastInterruptTime > DEBOUNCE_MS) {
    buttonPressed = true;
    lastInterruptTime = currentTime;
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  
  attachInterrupt(digitalPinToInterrupt(BUTTON_PIN), handleButtonPress, FALLING);
  
  Serial.println("Ready - press button");
}

void loop() {
  if (buttonPressed) {
    buttonPressed = false;
    
    static bool ledState = false;
    ledState = !ledState;
    digitalWrite(LED_PIN, ledState);
    
    Serial.printf("Interrupt triggered, LED: %s\n", ledState ? "ON" : "OFF");
  }
  
  // Other non-blocking tasks here
}
```

**Key Points:**
- `IRAM_ATTR` - places ISR in IRAM for fast access
- `volatile` - tells compiler variable can change outside normal flow
- Keep ISR short - set flag, process in loop()

---

## Level Detection vs Edge Detection

### Level Detection (Polling)

Continuously checks if button is HIGH or LOW.

```cpp
void loop() {
  if (digitalRead(BUTTON_PIN) == LOW) {
    // Runs EVERY loop iteration while pressed
    Serial.println("Button is held down");
  }
  delay(100);
}
```

Use case: Detecting if button is held (e.g., brightness control)

### Edge Detection (State Change)

Detects the moment of press or release.

```
Edge Types:

FALLING edge (HIGH → LOW): Button press detected
     ─────┐
          │
          └─────

RISING edge (LOW → HIGH): Button release detected
          ┌─────
          │
     ─────┘

CHANGE: Either transition
```

```cpp
attachInterrupt(pin, isr, FALLING);  // Press only
attachInterrupt(pin, isr, RISING);   // Release only
attachInterrupt(pin, isr, CHANGE);   // Both
```

Use case: Toggle actions, counting presses

---

## Button Class (Reusable Module)

### include/button.h

```cpp
#ifndef BUTTON_H
#define BUTTON_H

#include <Arduino.h>

class Button {
public:
  Button(uint8_t pin, unsigned long debounceMs = 50);
  
  void begin();
  bool update();        // Call every loop, returns true if state changed
  bool isPressed();     // Current debounced state
  bool wasPressed();    // True once per press
  bool wasReleased();   // True once per release
  
private:
  uint8_t _pin;
  unsigned long _debounceMs;
  bool _state;
  bool _lastReading;
  bool _pressedFlag;
  bool _releasedFlag;
  unsigned long _lastChangeTime;
};

#endif
```

### Usage Example

```cpp
#include <Arduino.h>
#include "button.h"

Button button1(4);   // GPIO4
Button button2(16);  // GPIO16

void setup() {
  Serial.begin(115200);
  button1.begin();
  button2.begin();
  pinMode(2, OUTPUT);
}

void loop() {
  button1.update();
  button2.update();
  
  if (button1.wasPressed()) {
    Serial.println("Button 1 pressed");
    digitalWrite(2, HIGH);
  }
  
  if (button1.wasReleased()) {
    Serial.println("Button 1 released");
    digitalWrite(2, LOW);
  }
  
  if (button2.wasPressed()) {
    Serial.println("Button 2 pressed");
  }
}
```

---

## Common GPIO Pins for Buttons

| GPIO | Recommendation |
|------|----------------|
| 4, 16, 17, 18, 19 | Good choices - no boot conflicts |
| 21, 22, 23, 25, 26, 27 | Good choices |
| 32, 33 | Good, also ADC capable |
| 34, 35, 36, 39 | Input only - no internal pull-up, need external resistor |
| 0 | Avoid - affects boot mode |
| 2 | Use carefully - may affect boot on some boards |
| 12, 15 | Avoid - strapping pins |

---

## Multiple Buttons Pattern

```cpp
const int NUM_BUTTONS = 4;
const int BUTTON_PINS[NUM_BUTTONS] = {4, 16, 17, 18};

bool buttonStates[NUM_BUTTONS];
bool lastStates[NUM_BUTTONS];
unsigned long lastDebounce[NUM_BUTTONS];
const unsigned long DEBOUNCE_MS = 50;

void setup() {
  Serial.begin(115200);
  for (int i = 0; i < NUM_BUTTONS; i++) {
    pinMode(BUTTON_PINS[i], INPUT_PULLUP);
    buttonStates[i] = HIGH;
    lastStates[i] = HIGH;
    lastDebounce[i] = 0;
  }
}

void loop() {
  for (int i = 0; i < NUM_BUTTONS; i++) {
    bool reading = digitalRead(BUTTON_PINS[i]);
    
    if (reading != lastStates[i]) {
      lastDebounce[i] = millis();
    }
    
    if ((millis() - lastDebounce[i]) > DEBOUNCE_MS) {
      if (reading != buttonStates[i]) {
        buttonStates[i] = reading;
        if (buttonStates[i] == LOW) {
          Serial.printf("Button %d pressed\n", i);
        }
      }
    }
    
    lastStates[i] = reading;
  }
}
```

---

## Long Press Detection

```cpp
const int BUTTON_PIN = 4;
const unsigned long LONG_PRESS_MS = 1000;
const unsigned long DEBOUNCE_MS = 50;

bool buttonState = HIGH;
bool lastReading = HIGH;
unsigned long lastDebounceTime = 0;
unsigned long pressStartTime = 0;
bool longPressHandled = false;

void setup() {
  Serial.begin(115200);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
}

void loop() {
  bool reading = digitalRead(BUTTON_PIN);
  
  if (reading != lastReading) {
    lastDebounceTime = millis();
  }
  
  if ((millis() - lastDebounceTime) > DEBOUNCE_MS) {
    if (reading != buttonState) {
      buttonState = reading;
      
      if (buttonState == LOW) {
        // Button just pressed
        pressStartTime = millis();
        longPressHandled = false;
      } else {
        // Button just released
        if (!longPressHandled) {
          Serial.println("Short press detected");
        }
      }
    }
    
    // Check for long press while held
    if (buttonState == LOW && !longPressHandled) {
      if ((millis() - pressStartTime) > LONG_PRESS_MS) {
        Serial.println("Long press detected!");
        longPressHandled = true;
      }
    }
  }
  
  lastReading = reading;
}
```

---

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Button always reads LOW | Wiring reversed or short circuit | Check connections, verify with multimeter |
| Button always reads HIGH | Button not connected, or pull-down missing | Use INPUT_PULLUP mode |
| Multiple triggers per press | No debouncing | Implement debounce strategy |
| Random triggers without pressing | Floating pin or noise | Add pull-up/pull-down resistor |
| Interrupt not firing | Wrong pin or mode | Verify pin supports interrupts, check FALLING/RISING |
| Crash in ISR | ISR too long or not IRAM_ATTR | Add IRAM_ATTR, minimize ISR code |

---

## Debounce Timing Reference

| Button Type | Typical Bounce Time | Recommended Debounce |
|-------------|---------------------|----------------------|
| Tactile (clicky) | 5-25 ms | 50 ms |
| Membrane | 10-50 ms | 75 ms |
| Toggle switch | 1-5 ms | 25 ms |
| Capacitive touch | N/A (no bounce) | 0 ms |

---

## Checklist

- [ ] Button wired correctly (with pull-up or pull-down)
- [ ] Understand difference between INPUT, INPUT_PULLUP, INPUT_PULLDOWN
- [ ] Implemented basic button reading
- [ ] Added debouncing (choose: delay, timestamp, or interrupt)
- [ ] Understand level detection vs edge detection
- [ ] Tested button triggers LED toggle
- [ ] Serial output shows clean single triggers
- [ ] Understand when to use interrupts vs polling
- [ ] Know which GPIOs are safe for button input

---

## Next Steps

Proceed to `05-sensor-basics.md` to learn analog and digital sensor integration.
