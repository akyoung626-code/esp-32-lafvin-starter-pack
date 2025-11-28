# 05 - Sensor Basics (DHT11 / LM35 / Photoresistor)

## Overview

This guide covers three common sensors found in starter kits: DHT11 (temperature/humidity), LM35 (temperature), and photoresistor (light level). You'll learn wiring, library usage, and data interpretation.

---

## Prerequisites

- Completed `01-setup.md` through `04-button-input.md`
- Sensors from Lavfin ESP32 Starter Kit
- Understanding of analog vs digital signals

---

## Analog vs Digital Sensors

```
DIGITAL SENSOR (DHT11):
┌─────────────────┐      ┌─────────────────┐
│ Sensor measures │  →   │ Outputs binary  │
│ environment     │      │ data stream     │
└─────────────────┘      └─────────────────┘
                         Example: "Temperature=25.0C"
                         
ANALOG SENSOR (LM35, Photoresistor):
┌─────────────────┐      ┌─────────────────┐
│ Sensor measures │  →   │ Outputs varying │
│ environment     │      │ voltage (0-3.3V)│
└─────────────────┘      └─────────────────┘
                         ESP32 ADC converts to 0-4095
```

---

## ESP32 ADC (Analog-to-Digital Converter)

### ADC Specifications

| Parameter | Value |
|-----------|-------|
| Resolution | 12-bit (0-4095) |
| Voltage range | 0-3.3V (default) |
| ADC1 pins | GPIO32-39 (8 channels) |
| ADC2 pins | GPIO0, 2, 4, 12-15, 25-27 |
| Wi-Fi conflict | ADC2 unavailable when Wi-Fi is active |

### ADC Accuracy Note

ESP32 ADC has non-linear behavior at extremes:

```
Actual Voltage vs ADC Reading:

Voltage:  0V     0.1V    1.5V    3.0V    3.3V
ADC:      0      ~100    ~1850   ~3900   ~4095
                 ↑               ↑
              Non-linear at low  Non-linear at high
```

For better accuracy, use attenuation settings or external ADC.

---

## Project Structure

```
projects/04-sensor-basics/
├── include/
├── lib/
│   └── README            (library dependencies noted here)
├── src/
│   └── main.cpp
├── platformio.ini
└── README.md
```

---

## Sensor 1: DHT11 (Temperature & Humidity)

### Specifications

| Parameter | Value |
|-----------|-------|
| Temperature range | 0-50°C |
| Temperature accuracy | ±2°C |
| Humidity range | 20-80% RH |
| Humidity accuracy | ±5% |
| Operating voltage | 3.3-5.5V |
| Protocol | Single-wire digital |

### Wiring Diagram

```
DHT11 Module (3-pin or 4-pin):

3-pin module (with PCB):
┌─────────────────┐
│      DHT11      │
│   ┌─────────┐   │
│   │  ┌───┐  │   │
│   │  │   │  │   │
│   └──┴───┴──┘   │
│    S   +   -    │
└────┬───┬───┬────┘
     │   │   │
     │   │   └─── GND
     │   └─────── 3.3V (or 5V)
     └─────────── GPIO4 (Data)

4-pin bare sensor:
┌─────────────┐
│   DHT11     │
│  ┌─────┐    │
│  │     │    │
│  └─────┘    │
│ 1  2  3  4  │
└─┬──┬──┬──┬──┘
  │  │  │  │
  │  │  │  └─── GND
  │  │  └────── Not connected
  │  └───────── Data → GPIO4 (add 10kΩ pull-up to 3.3V)
  └──────────── VCC (3.3V)
```

### Library Installation

Add to `platformio.ini`:

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
lib_deps = 
    adafruit/DHT sensor library@^1.4.4
    adafruit/Adafruit Unified Sensor@^1.1.9
```

### DHT11 Code

```cpp
#include <Arduino.h>
#include <DHT.h>

#define DHT_PIN 4
#define DHT_TYPE DHT11

DHT dht(DHT_PIN, DHT_TYPE);

void setup() {
  Serial.begin(115200);
  dht.begin();
  Serial.println("DHT11 Sensor Ready");
}

void loop() {
  // DHT11 needs 2 seconds between readings
  delay(2000);
  
  float humidity = dht.readHumidity();
  float tempC = dht.readTemperature();
  float tempF = dht.readTemperature(true);
  
  // Check for read errors
  if (isnan(humidity) || isnan(tempC)) {
    Serial.println("Failed to read from DHT sensor!");
    return;
  }
  
  // Heat index calculation
  float heatIndexC = dht.computeHeatIndex(tempC, humidity, false);
  
  Serial.printf("Temperature: %.1f°C (%.1f°F)\n", tempC, tempF);
  Serial.printf("Humidity: %.1f%%\n", humidity);
  Serial.printf("Heat Index: %.1f°C\n", heatIndexC);
  Serial.println("---");
}
```

### Expected Output

```
DHT11 Sensor Ready
Temperature: 24.0°C (75.2°F)
Humidity: 55.0%
Heat Index: 23.8°C
---
Temperature: 24.0°C (75.2°F)
Humidity: 56.0%
Heat Index: 23.9°C
---
```

### DHT11 Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Returns NaN | Wiring error or timing | Check connections, add 10kΩ pull-up |
| Constant values | Reading too fast | Minimum 2s between reads |
| Values seem wrong | DHT22 vs DHT11 mismatch | Verify DHT_TYPE matches sensor |

---

## Sensor 2: LM35 (Analog Temperature)

### Specifications

| Parameter | Value |
|-----------|-------|
| Temperature range | -55 to 150°C |
| Accuracy | ±0.5°C at 25°C |
| Output | 10mV per °C |
| Operating voltage | 4-30V |
| Output at 0°C | 0V |
| Output at 25°C | 250mV |

### Wiring Diagram

```
LM35 Pinout (flat side facing you):

         ┌─────┐
         │     │
    VCC ─┤     ├─ GND
         │     │
         └──┬──┘
            │
          VOUT → GPIO34 (ADC)

Full Wiring:
┌─────────────┐
│    LM35     │
│   ┌─────┐   │
│   │     │   │
│   └──┬──┘   │
│  V  OUT GND │
└──┬───┬───┬──┘
   │   │   │
   │   │   └─── GND
   │   └─────── GPIO34 (input-only ADC pin)
   └─────────── 3.3V
```

**Note**: LM35 is designed for 4-30V but works with reduced range at 3.3V.

### LM35 Code

```cpp
#include <Arduino.h>

const int LM35_PIN = 34;  // ADC1 channel (input-only pin)
const float ADC_RESOLUTION = 4095.0;
const float VREF = 3.3;
const float MV_PER_DEGREE = 10.0;

void setup() {
  Serial.begin(115200);
  analogReadResolution(12);  // Ensure 12-bit ADC
  Serial.println("LM35 Temperature Sensor Ready");
}

void loop() {
  // Take multiple readings and average for stability
  int readings = 0;
  const int NUM_SAMPLES = 10;
  
  for (int i = 0; i < NUM_SAMPLES; i++) {
    readings += analogRead(LM35_PIN);
    delay(10);
  }
  
  float avgReading = readings / (float)NUM_SAMPLES;
  
  // Convert ADC reading to voltage
  float voltage = (avgReading / ADC_RESOLUTION) * VREF;
  
  // Convert voltage to temperature (10mV per degree)
  float tempC = (voltage * 1000) / MV_PER_DEGREE;
  float tempF = (tempC * 9.0 / 5.0) + 32.0;
  
  Serial.printf("ADC: %.0f, Voltage: %.3fV, Temp: %.1f°C (%.1f°F)\n", 
                avgReading, voltage, tempC, tempF);
  
  delay(1000);
}
```

### Expected Output

```
LM35 Temperature Sensor Ready
ADC: 310, Voltage: 0.250V, Temp: 25.0°C (77.0°F)
ADC: 312, Voltage: 0.251V, Temp: 25.1°C (77.2°F)
ADC: 309, Voltage: 0.249V, Temp: 24.9°C (76.8°F)
```

### LM35 Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Reading always 0 | Wrong pin or not ADC capable | Use GPIO32-39 for reliable ADC |
| Readings fluctuate wildly | Electrical noise | Average multiple readings, add capacitor |
| Temperature seems high | Self-heating | Ensure adequate airflow |
| Negative temps wrong | Can't read below 0V | Use different circuit for below 0°C |

---

## Sensor 3: Photoresistor (LDR)

### Specifications

| Parameter | Value |
|-----------|-------|
| Type | Light Dependent Resistor |
| Dark resistance | ~1MΩ |
| Light resistance | ~1-10kΩ |
| Response time | ~20-30ms |

### How LDR Works

```
Light Level vs Resistance:

DARK                           BRIGHT
HIGH resistance               LOW resistance
(~1MΩ)                        (~1-10kΩ)

With voltage divider:
DARK → Higher voltage at ADC
BRIGHT → Lower voltage at ADC
```

### Voltage Divider Circuit

```
                3.3V
                 │
            ┌────┴────┐
            │  10kΩ   │
            │ (fixed) │
            └────┬────┘
                 │
GPIO34 ──────────┼────── ADC reads here
                 │
            ┌────┴────┐
            │   LDR   │
            │(variable)│
            └────┬────┘
                 │
                GND

When DARK:  LDR high (1MΩ), voltage divider → ~3.2V at GPIO
When LIGHT: LDR low (1kΩ), voltage divider → ~0.3V at GPIO
```

### Alternative Wiring (Inverted Response)

```
                3.3V
                 │
            ┌────┴────┐
            │   LDR   │
            └────┬────┘
                 │
GPIO34 ──────────┼────── 
                 │
            ┌────┴────┐
            │  10kΩ   │
            └────┬────┘
                 │
                GND

When DARK:  LDR high → ~0.3V at GPIO
When LIGHT: LDR low → ~3.2V at GPIO (higher = brighter)
```

### LDR Code

```cpp
#include <Arduino.h>

const int LDR_PIN = 34;

void setup() {
  Serial.begin(115200);
  analogReadResolution(12);
  Serial.println("Photoresistor Ready");
}

void loop() {
  int rawValue = analogRead(LDR_PIN);
  
  // Convert to percentage (adjust based on your circuit)
  // Assuming LDR on bottom of divider: higher value = darker
  int lightPercent = map(rawValue, 4095, 0, 0, 100);
  
  // Create visual bar
  int barLength = lightPercent / 5;  // 20 chars max
  char bar[21];
  for (int i = 0; i < 20; i++) {
    bar[i] = (i < barLength) ? '#' : '-';
  }
  bar[20] = '\0';
  
  Serial.printf("Raw: %4d | Light: %3d%% [%s]\n", rawValue, lightPercent, bar);
  
  delay(200);
}
```

### Expected Output

```
Photoresistor Ready
Raw: 3800 | Light:   7% [#-------------------]
Raw: 2500 | Light:  39% [#######-------------]
Raw: 1200 | Light:  71% [##############------]
Raw:  400 | Light:  90% [##################--]
```

### Light Level Thresholds

```cpp
enum LightLevel {
  DARK,
  DIM,
  NORMAL,
  BRIGHT
};

LightLevel getLightLevel(int adcValue) {
  // Adjust thresholds based on your environment and circuit
  if (adcValue > 3500) return DARK;
  if (adcValue > 2000) return DIM;
  if (adcValue > 800)  return NORMAL;
  return BRIGHT;
}

const char* lightLevelName(LightLevel level) {
  switch (level) {
    case DARK:   return "DARK";
    case DIM:    return "DIM";
    case NORMAL: return "NORMAL";
    case BRIGHT: return "BRIGHT";
  }
  return "UNKNOWN";
}
```

---

## Combined Sensor Project

### platformio.ini

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
lib_deps = 
    adafruit/DHT sensor library@^1.4.4
    adafruit/Adafruit Unified Sensor@^1.1.9
```

### Main Code Structure

```cpp
#include <Arduino.h>
#include <DHT.h>

// Pin definitions
#define DHT_PIN 4
#define DHT_TYPE DHT11
#define LDR_PIN 34
#define LM35_PIN 35  // Use if you have LM35

// Sensor instances
DHT dht(DHT_PIN, DHT_TYPE);

// Timing
unsigned long lastDHTRead = 0;
unsigned long lastAnalogRead = 0;
const unsigned long DHT_INTERVAL = 2000;
const unsigned long ANALOG_INTERVAL = 500;

void setup() {
  Serial.begin(115200);
  dht.begin();
  analogReadResolution(12);
  Serial.println("Multi-Sensor Station Ready");
  Serial.println("==========================");
}

void loop() {
  unsigned long now = millis();
  
  // Read DHT11 every 2 seconds
  if (now - lastDHTRead >= DHT_INTERVAL) {
    lastDHTRead = now;
    readDHT();
  }
  
  // Read analog sensors every 500ms
  if (now - lastAnalogRead >= ANALOG_INTERVAL) {
    lastAnalogRead = now;
    readAnalogSensors();
  }
}

void readDHT() {
  float temp = dht.readTemperature();
  float humidity = dht.readHumidity();
  
  if (!isnan(temp) && !isnan(humidity)) {
    Serial.printf("[DHT11] Temp: %.1f°C, Humidity: %.1f%%\n", temp, humidity);
  }
}

void readAnalogSensors() {
  int ldrValue = analogRead(LDR_PIN);
  int lightPercent = map(ldrValue, 4095, 0, 0, 100);
  
  Serial.printf("[LDR] Raw: %d, Light: %d%%\n", ldrValue, lightPercent);
}
```

---

## Sensor Data Structures

Organize readings for transmission:

```cpp
struct SensorData {
  float temperature;
  float humidity;
  int lightLevel;
  unsigned long timestamp;
  bool valid;
};

SensorData readAllSensors() {
  SensorData data;
  data.timestamp = millis();
  
  data.temperature = dht.readTemperature();
  data.humidity = dht.readHumidity();
  data.lightLevel = map(analogRead(LDR_PIN), 4095, 0, 0, 100);
  
  data.valid = !isnan(data.temperature) && !isnan(data.humidity);
  
  return data;
}
```

---

## Testing Steps

### DHT11 Testing

1. Wire sensor according to diagram
2. Upload code
3. Open Serial Monitor at 115200 baud
4. Verify readings appear every 2 seconds
5. Breathe on sensor - humidity should increase
6. Place near heat source - temperature should rise

### LM35 Testing

1. Wire sensor with VOUT to GPIO34
2. Upload code
3. Verify temperature readings are reasonable
4. Touch sensor body - temperature should rise slowly
5. Compare to known thermometer if available

### LDR Testing

1. Wire voltage divider circuit
2. Upload code
3. Cover LDR with hand - values should change
4. Use phone flashlight - values should change opposite direction
5. Note your bright/dark thresholds for your environment

---

## Checklist

- [ ] DHT11 wired correctly (data pin with pull-up if needed)
- [ ] DHT library installed via platformio.ini
- [ ] DHT11 returns valid temperature and humidity
- [ ] Understand LM35 voltage-to-temperature conversion
- [ ] LDR voltage divider wired correctly
- [ ] Light level changes detected when covering/illuminating LDR
- [ ] Know which GPIO pins are ADC-capable
- [ ] Understand ADC2 limitation when using Wi-Fi
- [ ] Multiple readings averaged for analog stability
- [ ] Sensor data organized for future transmission

---

## Next Steps

Proceed to `06-wifi-basics.md` to connect your ESP32 to a Wi-Fi network and prepare for data transmission.
