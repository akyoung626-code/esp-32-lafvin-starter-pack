# 08 - Intro to Async Patterns & Non-Blocking Code

## Overview

Blocking code prevents responsive applications. This guide covers techniques to write non-blocking firmware: `millis()` timing, state machines, and task management patterns.

---

## Prerequisites

- Completed `01-setup.md` through `07-web-server.md`
- Understanding of basic C++ functions and variables
- Familiarity with `delay()` problems

---

## Why Blocking Code is Problematic

### The Problem with `delay()`

```cpp
void loop() {
  readSensor();        // Takes 2ms
  delay(1000);         // BLOCKED - Nothing else can happen
  updateDisplay();     // Takes 5ms
  delay(500);          // BLOCKED again
  checkButton();       // User pressed button during delay - MISSED
}
```

### Timeline Visualization

```
With delay() - Blocking:

Time:  0ms    1000ms    1002ms    1502ms    1507ms   2507ms
       │        │          │         │         │        │
       ▼        ▼          ▼         ▼         ▼        ▼
     [Sensor]  │        [Display]   │      [Button]    │
               │                    │                   │
               └──── BLOCKED ──────┘                   │
                                    └──── BLOCKED ─────┘

Problem: Button press at 200ms is MISSED (not checked until 1507ms)


Without delay() - Non-Blocking:

Time:  0ms   1ms   2ms   3ms   4ms   5ms   ...  1000ms
       │     │     │     │     │     │           │
       ▼     ▼     ▼     ▼     ▼     ▼           ▼
      [S]   [B]   [B]   [S]   [B]   [B]   ...   [S]
       │     │     │     │     │     │           │
       └─────┴─────┴─────┴─────┴─────┴───────────┘
                    Rapid cycling

S = Sensor check, B = Button check
All tasks get regular attention
```

---

## Pattern 1: millis() Timing

### Basic Concept

```cpp
unsigned long previousMillis = 0;
const unsigned long INTERVAL = 1000;

void loop() {
  unsigned long currentMillis = millis();
  
  if (currentMillis - previousMillis >= INTERVAL) {
    previousMillis = currentMillis;
    // Do periodic task
  }
  
  // Other code runs every loop iteration
}
```

### Why `currentMillis - previousMillis` Works

```
Handles rollover correctly:

millis() overflows at ~49.7 days (unsigned long max: 4,294,967,295)

Example at rollover:
previousMillis = 4,294,967,290
currentMillis  = 5  (rolled over)

4,294,967,290 - 5 = would be negative... BUT
5 - 4,294,967,290 = 10 (with unsigned arithmetic!)

This subtraction pattern handles rollover automatically.
```

### Multiple Independent Timers

```cpp
// Timer structure
struct Timer {
  unsigned long previousMillis;
  unsigned long interval;
  bool active;
};

Timer ledTimer = {0, 1000, true};
Timer sensorTimer = {0, 2000, true};
Timer displayTimer = {0, 500, true};

bool timerReady(Timer& timer) {
  if (!timer.active) return false;
  
  unsigned long currentMillis = millis();
  if (currentMillis - timer.previousMillis >= timer.interval) {
    timer.previousMillis = currentMillis;
    return true;
  }
  return false;
}

void loop() {
  if (timerReady(ledTimer)) {
    toggleLed();
  }
  
  if (timerReady(sensorTimer)) {
    readSensors();
  }
  
  if (timerReady(displayTimer)) {
    updateDisplay();
  }
  
  // Runs every iteration - instant response
  checkButtons();
}
```

---

## Pattern 2: State Machines

### Concept

A state machine represents a system as discrete states with defined transitions.

```
State Machine Diagram (LED Blinker):

          ┌────────────────┐
          │                │
          ▼                │
     ┌─────────┐     ┌─────────┐
     │  OFF    │────►│   ON    │
     │         │     │         │
     └─────────┘◄────└─────────┘
          │                │
          └────────────────┘
          
Transitions occur after 1000ms in each state
```

### Simple State Machine

```cpp
enum LedState {
  LED_OFF,
  LED_ON,
  LED_FADE_UP,
  LED_FADE_DOWN
};

LedState currentState = LED_OFF;
unsigned long stateStartTime = 0;
int brightness = 0;

void updateStateMachine() {
  unsigned long elapsed = millis() - stateStartTime;
  
  switch (currentState) {
    case LED_OFF:
      digitalWrite(LED_PIN, LOW);
      if (elapsed >= 1000) {
        changeState(LED_ON);
      }
      break;
      
    case LED_ON:
      digitalWrite(LED_PIN, HIGH);
      if (elapsed >= 1000) {
        changeState(LED_OFF);
      }
      break;
      
    case LED_FADE_UP:
      brightness = map(elapsed, 0, 1000, 0, 255);
      analogWrite(LED_PIN, brightness);
      if (elapsed >= 1000) {
        changeState(LED_FADE_DOWN);
      }
      break;
      
    case LED_FADE_DOWN:
      brightness = map(elapsed, 0, 1000, 255, 0);
      analogWrite(LED_PIN, brightness);
      if (elapsed >= 1000) {
        changeState(LED_FADE_UP);
      }
      break;
  }
}

void changeState(LedState newState) {
  Serial.printf("State: %d -> %d\n", currentState, newState);
  currentState = newState;
  stateStartTime = millis();
}

void loop() {
  updateStateMachine();
  // Other tasks here
}
```

### Multi-State Example: WiFi Connection Manager

```cpp
enum WiFiManagerState {
  WIFI_IDLE,
  WIFI_CONNECTING,
  WIFI_CONNECTED,
  WIFI_DISCONNECTED,
  WIFI_RECONNECTING
};

WiFiManagerState wifiState = WIFI_IDLE;
unsigned long stateTime = 0;
int connectionAttempts = 0;
const int MAX_ATTEMPTS = 10;
const unsigned long CONNECTION_TIMEOUT = 15000;
const unsigned long RECONNECT_DELAY = 5000;

void updateWiFiManager() {
  unsigned long elapsed = millis() - stateTime;
  
  switch (wifiState) {
    case WIFI_IDLE:
      Serial.println("[WiFi] Starting connection...");
      WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
      changeWiFiState(WIFI_CONNECTING);
      break;
      
    case WIFI_CONNECTING:
      if (WiFi.status() == WL_CONNECTED) {
        connectionAttempts = 0;
        Serial.printf("[WiFi] Connected! IP: %s\n", 
                      WiFi.localIP().toString().c_str());
        changeWiFiState(WIFI_CONNECTED);
      } else if (elapsed > CONNECTION_TIMEOUT) {
        Serial.println("[WiFi] Connection timeout");
        connectionAttempts++;
        if (connectionAttempts < MAX_ATTEMPTS) {
          changeWiFiState(WIFI_RECONNECTING);
        } else {
          Serial.println("[WiFi] Max attempts reached");
          changeWiFiState(WIFI_DISCONNECTED);
        }
      }
      break;
      
    case WIFI_CONNECTED:
      if (WiFi.status() != WL_CONNECTED) {
        Serial.println("[WiFi] Connection lost");
        changeWiFiState(WIFI_RECONNECTING);
      }
      break;
      
    case WIFI_DISCONNECTED:
      // Wait for manual intervention or reset
      break;
      
    case WIFI_RECONNECTING:
      if (elapsed > RECONNECT_DELAY) {
        WiFi.disconnect();
        WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
        changeWiFiState(WIFI_CONNECTING);
      }
      break;
  }
}

void changeWiFiState(WiFiManagerState newState) {
  wifiState = newState;
  stateTime = millis();
}
```

---

## Pattern 3: Task Scheduler

### Simple Cooperative Scheduler

```cpp
typedef void (*TaskCallback)();

struct Task {
  TaskCallback callback;
  unsigned long interval;
  unsigned long lastRun;
  bool enabled;
  const char* name;
};

const int MAX_TASKS = 10;
Task tasks[MAX_TASKS];
int taskCount = 0;

void addTask(const char* name, TaskCallback callback, unsigned long interval) {
  if (taskCount >= MAX_TASKS) return;
  
  tasks[taskCount] = {
    callback,
    interval,
    0,
    true,
    name
  };
  taskCount++;
}

void runScheduler() {
  unsigned long now = millis();
  
  for (int i = 0; i < taskCount; i++) {
    if (!tasks[i].enabled) continue;
    
    if (now - tasks[i].lastRun >= tasks[i].interval) {
      tasks[i].lastRun = now;
      tasks[i].callback();
    }
  }
}

// Task functions
void taskBlink() {
  static bool state = false;
  state = !state;
  digitalWrite(LED_PIN, state);
}

void taskSensor() {
  int value = analogRead(SENSOR_PIN);
  Serial.printf("Sensor: %d\n", value);
}

void taskHeartbeat() {
  Serial.printf("[%lu] Heartbeat - Heap: %d\n", millis(), ESP.getFreeHeap());
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  
  addTask("blink", taskBlink, 500);
  addTask("sensor", taskSensor, 2000);
  addTask("heartbeat", taskHeartbeat, 10000);
}

void loop() {
  runScheduler();
}
```

---

## Pattern 4: Non-Blocking Delays

### One-Shot Delay

```cpp
class NonBlockingDelay {
public:
  void start(unsigned long duration) {
    _startTime = millis();
    _duration = duration;
    _active = true;
  }
  
  bool isComplete() {
    if (!_active) return false;
    if (millis() - _startTime >= _duration) {
      _active = false;
      return true;
    }
    return false;
  }
  
  bool isRunning() {
    return _active && !isComplete();
  }
  
  void cancel() {
    _active = false;
  }
  
private:
  unsigned long _startTime = 0;
  unsigned long _duration = 0;
  bool _active = false;
};

// Usage
NonBlockingDelay sensorDelay;

void loop() {
  if (sensorDelay.isComplete()) {
    readSensor();
    sensorDelay.start(2000);  // Restart for next reading
  }
  
  // Other code runs freely
  handleButtons();
  updateDisplay();
}
```

---

## Pattern 5: Event Queue

For decoupling event detection from handling:

```cpp
enum EventType {
  EVENT_NONE,
  EVENT_BUTTON_PRESS,
  EVENT_BUTTON_RELEASE,
  EVENT_SENSOR_READY,
  EVENT_WIFI_CONNECTED,
  EVENT_WIFI_DISCONNECTED,
  EVENT_TIMER_EXPIRED
};

struct Event {
  EventType type;
  unsigned long timestamp;
  int data;
};

const int QUEUE_SIZE = 16;
Event eventQueue[QUEUE_SIZE];
int queueHead = 0;
int queueTail = 0;

bool queueEvent(EventType type, int data = 0) {
  int nextTail = (queueTail + 1) % QUEUE_SIZE;
  if (nextTail == queueHead) {
    // Queue full
    return false;
  }
  
  eventQueue[queueTail] = {type, millis(), data};
  queueTail = nextTail;
  return true;
}

bool dequeueEvent(Event& event) {
  if (queueHead == queueTail) {
    // Queue empty
    return false;
  }
  
  event = eventQueue[queueHead];
  queueHead = (queueHead + 1) % QUEUE_SIZE;
  return true;
}

void processEvents() {
  Event event;
  while (dequeueEvent(event)) {
    switch (event.type) {
      case EVENT_BUTTON_PRESS:
        Serial.printf("Button %d pressed at %lu\n", event.data, event.timestamp);
        break;
        
      case EVENT_SENSOR_READY:
        Serial.printf("Sensor reading: %d\n", event.data);
        break;
        
      case EVENT_WIFI_CONNECTED:
        Serial.println("WiFi connected!");
        break;
        
      default:
        break;
    }
  }
}

// In ISR or polling code:
void IRAM_ATTR buttonISR() {
  queueEvent(EVENT_BUTTON_PRESS, 1);
}

void loop() {
  // Detect events
  checkSensors();  // May queue EVENT_SENSOR_READY
  
  // Process all queued events
  processEvents();
}
```

---

## Pattern 6: Async Operations with Callbacks

```cpp
typedef void (*CompletionCallback)(bool success, int result);

struct AsyncOperation {
  bool inProgress;
  unsigned long startTime;
  unsigned long timeout;
  CompletionCallback callback;
};

AsyncOperation sensorOp = {false, 0, 100, nullptr};

void startAsyncSensorRead(CompletionCallback cb) {
  if (sensorOp.inProgress) return;  // Already running
  
  sensorOp.inProgress = true;
  sensorOp.startTime = millis();
  sensorOp.timeout = 100;
  sensorOp.callback = cb;
  
  // Trigger sensor read (hardware-specific)
  // ...
}

void updateAsyncSensor() {
  if (!sensorOp.inProgress) return;
  
  // Check if reading is ready
  if (sensorReadyFlag) {  // Hardware-specific check
    int value = getSensorValue();
    sensorOp.inProgress = false;
    if (sensorOp.callback) {
      sensorOp.callback(true, value);
    }
  }
  
  // Check timeout
  if (millis() - sensorOp.startTime > sensorOp.timeout) {
    sensorOp.inProgress = false;
    if (sensorOp.callback) {
      sensorOp.callback(false, 0);
    }
  }
}

// Usage
void onSensorComplete(bool success, int result) {
  if (success) {
    Serial.printf("Sensor: %d\n", result);
  } else {
    Serial.println("Sensor timeout!");
  }
}

void loop() {
  updateAsyncSensor();
  
  static unsigned long lastRequest = 0;
  if (millis() - lastRequest > 1000) {
    lastRequest = millis();
    startAsyncSensorRead(onSensorComplete);
  }
}
```

---

## Combining Patterns: Complete Example

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include "secrets.h"

// Pin definitions
const int LED_PIN = 2;
const int BUTTON_PIN = 4;
const int SENSOR_PIN = 34;

// Timers
struct Timer {
  unsigned long previous;
  unsigned long interval;
};

Timer ledTimer = {0, 500};
Timer sensorTimer = {0, 2000};
Timer wifiCheckTimer = {0, 5000};

// State machine for LED
enum LedMode { LED_OFF, LED_BLINK, LED_ON };
LedMode ledMode = LED_BLINK;
bool ledState = false;

// Button debouncing
bool lastButtonState = HIGH;
bool buttonState = HIGH;
unsigned long lastDebounceTime = 0;
const unsigned long DEBOUNCE_MS = 50;

// Function prototypes
bool timerReady(Timer& t);
void updateLed();
void updateButton();
void updateSensor();
void checkWiFi();

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.println("Starting...");
}

void loop() {
  // Non-blocking timer checks
  if (timerReady(ledTimer)) {
    updateLed();
  }
  
  if (timerReady(sensorTimer)) {
    updateSensor();
  }
  
  if (timerReady(wifiCheckTimer)) {
    checkWiFi();
  }
  
  // Button checked every iteration (fast response)
  updateButton();
}

bool timerReady(Timer& t) {
  unsigned long now = millis();
  if (now - t.previous >= t.interval) {
    t.previous = now;
    return true;
  }
  return false;
}

void updateLed() {
  switch (ledMode) {
    case LED_OFF:
      digitalWrite(LED_PIN, LOW);
      break;
    case LED_ON:
      digitalWrite(LED_PIN, HIGH);
      break;
    case LED_BLINK:
      ledState = !ledState;
      digitalWrite(LED_PIN, ledState);
      break;
  }
}

void updateButton() {
  bool reading = digitalRead(BUTTON_PIN);
  
  if (reading != lastButtonState) {
    lastDebounceTime = millis();
  }
  
  if ((millis() - lastDebounceTime) > DEBOUNCE_MS) {
    if (reading != buttonState) {
      buttonState = reading;
      
      if (buttonState == LOW) {
        // Button pressed - cycle LED mode
        ledMode = (LedMode)((ledMode + 1) % 3);
        Serial.printf("LED Mode: %d\n", ledMode);
      }
    }
  }
  
  lastButtonState = reading;
}

void updateSensor() {
  int value = analogRead(SENSOR_PIN);
  Serial.printf("[Sensor] Value: %d\n", value);
}

void checkWiFi() {
  if (WiFi.status() == WL_CONNECTED) {
    Serial.printf("[WiFi] Connected - RSSI: %d dBm\n", WiFi.RSSI());
  } else {
    Serial.println("[WiFi] Disconnected - reconnecting...");
    WiFi.reconnect();
  }
}
```

---

## Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Using `delay()` in loop | Blocks all other code | Use `millis()` timing |
| Checking `millis() == target` | May miss exact millisecond | Use `>=` comparison |
| Not using `unsigned long` | Integer overflow issues | Always use `unsigned long` for time |
| Long-running ISRs | Crashes, missed interrupts | Set flag only, process in loop |
| Blocking while loops | Unresponsive system | Convert to state machine |

---

## Timing Precision Notes

```
millis() resolution: 1 ms
micros() resolution: 1 µs (use for high-precision timing)

For sub-millisecond timing:
unsigned long previousMicros = 0;
const unsigned long INTERVAL_US = 500;  // 500 microseconds

void loop() {
  if (micros() - previousMicros >= INTERVAL_US) {
    previousMicros = micros();
    // High-speed task
  }
}

Note: micros() overflows every ~71 minutes
```

---

## Checklist

- [ ] Understand why `delay()` blocks execution
- [ ] Can implement basic `millis()` timing pattern
- [ ] Can create multiple independent timers
- [ ] Understand state machine concept
- [ ] Implemented at least one state machine
- [ ] Know how to handle `millis()` rollover
- [ ] Can combine multiple async patterns
- [ ] Button input remains responsive while other tasks run
- [ ] Sensor readings occur at regular intervals without blocking

---

## Next Steps

Proceed to `09-sensor-stream.md` to build a complete Wi-Fi enabled sensor streaming project.
