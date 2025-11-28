# 09 - Project: Wi-Fi Enabled Sensor Stream

## Overview

This project combines everything learned so far into a complete sensor streaming solution. The ESP32 reads sensor data and streams it as JSON over HTTP, ready for integration with mobile apps and browser extensions.

---

## Prerequisites

- Completed `01-setup.md` through `08-async-patterns.md`
- DHT11 or similar sensor wired and tested
- ESP32 connected to Wi-Fi

---

## Project Goals

1. Read multiple sensors non-blockingly
2. Serve real-time data via JSON API
3. Implement WebSocket for live updates
4. Structure code for maintainability
5. Prepare for mobile/extension integration

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        ESP32                                │
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────────┐  │
│  │ Sensors  │───►│  Data    │───►│    HTTP Server       │  │
│  │ (DHT11,  │    │ Manager  │    │  /api/sensors        │  │
│  │  LDR)    │    │          │    │  /api/status         │  │
│  └──────────┘    └──────────┘    └──────────────────────┘  │
│                       │                     │               │
│                       │          ┌──────────────────────┐  │
│                       └─────────►│  WebSocket Server    │  │
│                                  │  ws://ip:81          │  │
│                                  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                          │                    │
                          ▼                    ▼
                  ┌──────────────┐    ┌──────────────┐
                  │  Mobile App  │    │   Browser    │
                  │  (HTTP API)  │    │  Extension   │
                  └──────────────┘    └──────────────┘
```

---

## Project Structure

```
projects/08-sensor-stream/
├── include/
│   ├── secrets.h
│   ├── config.h
│   └── sensor_manager.h
├── lib/
├── src/
│   ├── main.cpp
│   └── sensor_manager.cpp
├── data/
│   └── index.html
└── platformio.ini
```

---

## Configuration Files

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
    bblanchon/ArduinoJson@^6.21.0
    links2004/WebSockets@^2.4.0

board_build.filesystem = spiffs
```

### include/config.h

```cpp
#ifndef CONFIG_H
#define CONFIG_H

// Pin Assignments
#define DHT_PIN 4
#define DHT_TYPE DHT11
#define LDR_PIN 34

// Timing (milliseconds)
#define DHT_READ_INTERVAL 2000
#define LDR_READ_INTERVAL 500
#define WEBSOCKET_BROADCAST_INTERVAL 1000
#define WIFI_CHECK_INTERVAL 30000

// Server Configuration
#define HTTP_PORT 80
#define WEBSOCKET_PORT 81

// Data Configuration
#define SENSOR_HISTORY_SIZE 60  // Keep last 60 readings
#define JSON_BUFFER_SIZE 512

// Debug
#define DEBUG_ENABLED 1

#if DEBUG_ENABLED
  #define DEBUG_PRINT(x) Serial.println(x)
  #define DEBUG_PRINTF(fmt, ...) Serial.printf(fmt, __VA_ARGS__)
#else
  #define DEBUG_PRINT(x)
  #define DEBUG_PRINTF(fmt, ...)
#endif

#endif
```

### include/secrets.h

```cpp
#ifndef SECRETS_H
#define SECRETS_H

#define WIFI_SSID "YourNetworkName"
#define WIFI_PASSWORD "YourPassword"

#endif
```

---

## Sensor Manager Module

### include/sensor_manager.h

```cpp
#ifndef SENSOR_MANAGER_H
#define SENSOR_MANAGER_H

#include <Arduino.h>
#include <DHT.h>
#include "config.h"

struct SensorReading {
  float temperature;
  float humidity;
  int lightLevel;
  unsigned long timestamp;
  bool valid;
};

class SensorManager {
public:
  SensorManager();
  
  void begin();
  void update();  // Call in loop()
  
  SensorReading getCurrentReading();
  SensorReading getReading(int index);  // 0 = newest
  int getHistoryCount();
  
  float getTemperature();
  float getHumidity();
  int getLightLevel();
  int getLightPercent();
  
  bool isDataValid();
  unsigned long getLastUpdateTime();
  
private:
  DHT _dht;
  
  SensorReading _history[SENSOR_HISTORY_SIZE];
  int _historyIndex;
  int _historyCount;
  
  unsigned long _lastDHTRead;
  unsigned long _lastLDRRead;
  
  float _temperature;
  float _humidity;
  int _lightRaw;
  bool _dataValid;
  
  void readDHT();
  void readLDR();
  void addToHistory();
};

#endif
```

### src/sensor_manager.cpp

```cpp
#include "sensor_manager.h"

SensorManager::SensorManager() : _dht(DHT_PIN, DHT_TYPE) {
  _historyIndex = 0;
  _historyCount = 0;
  _lastDHTRead = 0;
  _lastLDRRead = 0;
  _temperature = 0;
  _humidity = 0;
  _lightRaw = 0;
  _dataValid = false;
}

void SensorManager::begin() {
  _dht.begin();
  analogReadResolution(12);
  DEBUG_PRINT("[Sensors] Initialized");
}

void SensorManager::update() {
  unsigned long now = millis();
  
  if (now - _lastDHTRead >= DHT_READ_INTERVAL) {
    _lastDHTRead = now;
    readDHT();
  }
  
  if (now - _lastLDRRead >= LDR_READ_INTERVAL) {
    _lastLDRRead = now;
    readLDR();
  }
}

void SensorManager::readDHT() {
  float t = _dht.readTemperature();
  float h = _dht.readHumidity();
  
  if (!isnan(t) && !isnan(h)) {
    _temperature = t;
    _humidity = h;
    _dataValid = true;
    addToHistory();
    DEBUG_PRINTF("[DHT] T: %.1f°C, H: %.1f%%\n", t, h);
  } else {
    DEBUG_PRINT("[DHT] Read failed");
  }
}

void SensorManager::readLDR() {
  // Average multiple readings for stability
  int sum = 0;
  for (int i = 0; i < 5; i++) {
    sum += analogRead(LDR_PIN);
    delayMicroseconds(100);
  }
  _lightRaw = sum / 5;
}

void SensorManager::addToHistory() {
  SensorReading reading;
  reading.temperature = _temperature;
  reading.humidity = _humidity;
  reading.lightLevel = getLightPercent();
  reading.timestamp = millis();
  reading.valid = _dataValid;
  
  _history[_historyIndex] = reading;
  _historyIndex = (_historyIndex + 1) % SENSOR_HISTORY_SIZE;
  
  if (_historyCount < SENSOR_HISTORY_SIZE) {
    _historyCount++;
  }
}

SensorReading SensorManager::getCurrentReading() {
  SensorReading reading;
  reading.temperature = _temperature;
  reading.humidity = _humidity;
  reading.lightLevel = getLightPercent();
  reading.timestamp = millis();
  reading.valid = _dataValid;
  return reading;
}

SensorReading SensorManager::getReading(int index) {
  if (index >= _historyCount) {
    return {0, 0, 0, 0, false};
  }
  int actualIndex = (_historyIndex - 1 - index + SENSOR_HISTORY_SIZE) % SENSOR_HISTORY_SIZE;
  return _history[actualIndex];
}

int SensorManager::getHistoryCount() {
  return _historyCount;
}

float SensorManager::getTemperature() { return _temperature; }
float SensorManager::getHumidity() { return _humidity; }
int SensorManager::getLightLevel() { return _lightRaw; }
int SensorManager::getLightPercent() { return map(_lightRaw, 4095, 0, 0, 100); }
bool SensorManager::isDataValid() { return _dataValid; }
unsigned long SensorManager::getLastUpdateTime() { return _lastDHTRead; }
```

---

## Main Application

### src/main.cpp

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <WebServer.h>
#include <WebSocketsServer.h>
#include <ArduinoJson.h>

#include "config.h"
#include "secrets.h"
#include "sensor_manager.h"

// Global objects
WebServer server(HTTP_PORT);
WebSocketsServer webSocket(WEBSOCKET_PORT);
SensorManager sensors;

// Timing
unsigned long lastWiFiCheck = 0;
unsigned long lastWebSocketBroadcast = 0;

// Function prototypes
void setupWiFi();
void setupServer();
void setupWebSocket();
void handleRoot();
void handleApiSensors();
void handleApiStatus();
void handleApiHistory();
void webSocketEvent(uint8_t num, WStype_t type, uint8_t* payload, size_t length);
void broadcastSensorData();
String getSensorJson();
void addCorsHeaders();

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  DEBUG_PRINT("\n=============================");
  DEBUG_PRINT("  ESP32 Sensor Stream");
  DEBUG_PRINT("=============================\n");
  
  sensors.begin();
  setupWiFi();
  setupServer();
  setupWebSocket();
  
  DEBUG_PRINT("\n[System] Ready!");
  DEBUG_PRINTF("[System] HTTP: http://%s\n", WiFi.localIP().toString().c_str());
  DEBUG_PRINTF("[System] WebSocket: ws://%s:%d\n", WiFi.localIP().toString().c_str(), WEBSOCKET_PORT);
}

void loop() {
  unsigned long now = millis();
  
  // Update sensors (non-blocking)
  sensors.update();
  
  // Handle web clients
  server.handleClient();
  webSocket.loop();
  
  // Periodic WebSocket broadcast
  if (now - lastWebSocketBroadcast >= WEBSOCKET_BROADCAST_INTERVAL) {
    lastWebSocketBroadcast = now;
    broadcastSensorData();
  }
  
  // Periodic WiFi check
  if (now - lastWiFiCheck >= WIFI_CHECK_INTERVAL) {
    lastWiFiCheck = now;
    if (WiFi.status() != WL_CONNECTED) {
      DEBUG_PRINT("[WiFi] Reconnecting...");
      WiFi.reconnect();
    }
  }
}

// ============= WiFi Setup =============

void setupWiFi() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  
  DEBUG_PRINTF("[WiFi] Connecting to %s", WIFI_SSID);
  
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 30) {
    delay(500);
    Serial.print(".");
    attempts++;
  }
  
  Serial.println();
  
  if (WiFi.status() == WL_CONNECTED) {
    DEBUG_PRINTF("[WiFi] Connected! IP: %s\n", WiFi.localIP().toString().c_str());
  } else {
    DEBUG_PRINT("[WiFi] Connection failed!");
  }
}

// ============= HTTP Server Setup =============

void setupServer() {
  server.on("/", HTTP_GET, handleRoot);
  server.on("/api/sensors", HTTP_GET, handleApiSensors);
  server.on("/api/status", HTTP_GET, handleApiStatus);
  server.on("/api/history", HTTP_GET, handleApiHistory);
  
  // Handle CORS preflight
  server.on("/api/sensors", HTTP_OPTIONS, []() {
    addCorsHeaders();
    server.send(204);
  });
  server.on("/api/status", HTTP_OPTIONS, []() {
    addCorsHeaders();
    server.send(204);
  });
  server.on("/api/history", HTTP_OPTIONS, []() {
    addCorsHeaders();
    server.send(204);
  });
  
  server.begin();
  DEBUG_PRINT("[HTTP] Server started on port 80");
}

void addCorsHeaders() {
  server.sendHeader("Access-Control-Allow-Origin", "*");
  server.sendHeader("Access-Control-Allow-Methods", "GET, POST, OPTIONS");
  server.sendHeader("Access-Control-Allow-Headers", "Content-Type");
}

// ============= HTTP Handlers =============

void handleRoot() {
  String html = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>ESP32 Sensor Stream</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: system-ui, sans-serif; background: #0a0a0f; color: #e0e0e0; padding: 20px; }
    .container { max-width: 800px; margin: 0 auto; }
    h1 { color: #00ff99; margin-bottom: 20px; }
    .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 15px; }
    .card { background: #1a1a2e; padding: 20px; border-radius: 12px; border: 1px solid #333; }
    .card h2 { color: #888; font-size: 14px; margin-bottom: 10px; text-transform: uppercase; }
    .value { font-size: 36px; font-weight: bold; color: #00ff99; }
    .unit { font-size: 16px; color: #666; }
    .status { display: flex; align-items: center; gap: 10px; margin-top: 20px; }
    .dot { width: 10px; height: 10px; border-radius: 50%; }
    .dot.connected { background: #00ff99; box-shadow: 0 0 10px #00ff99; }
    .dot.disconnected { background: #ff4444; }
    #log { background: #111; padding: 15px; border-radius: 8px; font-family: monospace; font-size: 12px; height: 200px; overflow-y: auto; margin-top: 20px; }
    .log-entry { padding: 2px 0; border-bottom: 1px solid #222; }
  </style>
</head>
<body>
  <div class="container">
    <h1>ESP32 Sensor Stream</h1>
    <div class="grid">
      <div class="card">
        <h2>Temperature</h2>
        <div class="value"><span id="temp">--</span><span class="unit">°C</span></div>
      </div>
      <div class="card">
        <h2>Humidity</h2>
        <div class="value"><span id="humidity">--</span><span class="unit">%</span></div>
      </div>
      <div class="card">
        <h2>Light Level</h2>
        <div class="value"><span id="light">--</span><span class="unit">%</span></div>
      </div>
      <div class="card">
        <h2>Uptime</h2>
        <div class="value"><span id="uptime">--</span><span class="unit">s</span></div>
      </div>
    </div>
    <div class="status">
      <div class="dot" id="statusDot"></div>
      <span id="statusText">Connecting...</span>
    </div>
    <div id="log"></div>
  </div>
  <script>
    const ws = new WebSocket(`ws://${location.hostname}:81`);
    const log = document.getElementById('log');
    
    function addLog(msg) {
      const entry = document.createElement('div');
      entry.className = 'log-entry';
      entry.textContent = `[${new Date().toLocaleTimeString()}] ${msg}`;
      log.insertBefore(entry, log.firstChild);
      if (log.children.length > 50) log.removeChild(log.lastChild);
    }
    
    ws.onopen = () => {
      document.getElementById('statusDot').classList.add('connected');
      document.getElementById('statusDot').classList.remove('disconnected');
      document.getElementById('statusText').textContent = 'Connected via WebSocket';
      addLog('WebSocket connected');
    };
    
    ws.onclose = () => {
      document.getElementById('statusDot').classList.remove('connected');
      document.getElementById('statusDot').classList.add('disconnected');
      document.getElementById('statusText').textContent = 'Disconnected';
      addLog('WebSocket disconnected');
      setTimeout(() => location.reload(), 5000);
    };
    
    ws.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        document.getElementById('temp').textContent = data.temperature.toFixed(1);
        document.getElementById('humidity').textContent = data.humidity.toFixed(1);
        document.getElementById('light').textContent = data.light;
        document.getElementById('uptime').textContent = Math.floor(data.uptime / 1000);
        addLog(`T:${data.temperature.toFixed(1)}°C H:${data.humidity.toFixed(1)}% L:${data.light}%`);
      } catch (e) {
        addLog('Parse error: ' + e.message);
      }
    };
  </script>
</body>
</html>
)rawliteral";
  
  server.send(200, "text/html", html);
}

void handleApiSensors() {
  addCorsHeaders();
  server.send(200, "application/json", getSensorJson());
}

void handleApiStatus() {
  addCorsHeaders();
  
  StaticJsonDocument<JSON_BUFFER_SIZE> doc;
  doc["uptime"] = millis();
  doc["heap"] = ESP.getFreeHeap();
  doc["rssi"] = WiFi.RSSI();
  doc["ip"] = WiFi.localIP().toString();
  doc["websocket_clients"] = webSocket.connectedClients();
  doc["sensor_valid"] = sensors.isDataValid();
  doc["history_count"] = sensors.getHistoryCount();
  
  String response;
  serializeJson(doc, response);
  server.send(200, "application/json", response);
}

void handleApiHistory() {
  addCorsHeaders();
  
  StaticJsonDocument<2048> doc;
  JsonArray history = doc.createNestedArray("history");
  
  int count = min(sensors.getHistoryCount(), 20);  // Limit to 20 entries
  for (int i = 0; i < count; i++) {
    SensorReading r = sensors.getReading(i);
    JsonObject entry = history.createNestedObject();
    entry["temperature"] = r.temperature;
    entry["humidity"] = r.humidity;
    entry["light"] = r.lightLevel;
    entry["timestamp"] = r.timestamp;
  }
  
  String response;
  serializeJson(doc, response);
  server.send(200, "application/json", response);
}

// ============= WebSocket Setup =============

void setupWebSocket() {
  webSocket.begin();
  webSocket.onEvent(webSocketEvent);
  DEBUG_PRINTF("[WS] Server started on port %d\n", WEBSOCKET_PORT);
}

void webSocketEvent(uint8_t num, WStype_t type, uint8_t* payload, size_t length) {
  switch (type) {
    case WStype_CONNECTED:
      DEBUG_PRINTF("[WS] Client %d connected\n", num);
      webSocket.sendTXT(num, getSensorJson());  // Send current data immediately
      break;
      
    case WStype_DISCONNECTED:
      DEBUG_PRINTF("[WS] Client %d disconnected\n", num);
      break;
      
    case WStype_TEXT:
      DEBUG_PRINTF("[WS] Client %d sent: %s\n", num, payload);
      // Handle incoming commands if needed
      break;
      
    default:
      break;
  }
}

void broadcastSensorData() {
  if (webSocket.connectedClients() > 0) {
    String json = getSensorJson();
    webSocket.broadcastTXT(json);
  }
}

// ============= Helpers =============

String getSensorJson() {
  StaticJsonDocument<JSON_BUFFER_SIZE> doc;
  
  SensorReading reading = sensors.getCurrentReading();
  
  doc["temperature"] = reading.temperature;
  doc["humidity"] = reading.humidity;
  doc["light"] = reading.lightLevel;
  doc["uptime"] = millis();
  doc["valid"] = reading.valid;
  
  String json;
  serializeJson(doc, json);
  return json;
}
```

---

## API Documentation

### HTTP Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | Web dashboard |
| `/api/sensors` | GET | Current sensor readings |
| `/api/status` | GET | System status |
| `/api/history` | GET | Last 20 readings |

### Response Formats

#### GET /api/sensors

```json
{
  "temperature": 24.5,
  "humidity": 55.0,
  "light": 72,
  "uptime": 123456,
  "valid": true
}
```

#### GET /api/status

```json
{
  "uptime": 123456,
  "heap": 245000,
  "rssi": -52,
  "ip": "192.168.1.100",
  "websocket_clients": 2,
  "sensor_valid": true,
  "history_count": 45
}
```

#### GET /api/history

```json
{
  "history": [
    {"temperature": 24.5, "humidity": 55.0, "light": 72, "timestamp": 123456},
    {"temperature": 24.3, "humidity": 55.2, "light": 71, "timestamp": 121456}
  ]
}
```

### WebSocket

Connect to: `ws://<ESP32_IP>:81`

Receives JSON messages every second with current sensor data.

---

## Integration Examples

### Browser Extension (JavaScript)

```javascript
// background.js or content script
class ESP32Client {
  constructor(ip) {
    this.baseUrl = `http://${ip}`;
    this.wsUrl = `ws://${ip}:81`;
    this.ws = null;
    this.onData = null;
  }
  
  async getSensors() {
    const response = await fetch(`${this.baseUrl}/api/sensors`);
    return response.json();
  }
  
  async getStatus() {
    const response = await fetch(`${this.baseUrl}/api/status`);
    return response.json();
  }
  
  connectWebSocket(callback) {
    this.onData = callback;
    this.ws = new WebSocket(this.wsUrl);
    
    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      if (this.onData) this.onData(data);
    };
    
    this.ws.onclose = () => {
      setTimeout(() => this.connectWebSocket(callback), 5000);
    };
  }
  
  disconnect() {
    if (this.ws) this.ws.close();
  }
}

// Usage
const esp32 = new ESP32Client('192.168.1.100');

// Poll-based
const data = await esp32.getSensors();
console.log(`Temperature: ${data.temperature}°C`);

// Real-time
esp32.connectWebSocket((data) => {
  console.log('Live data:', data);
});
```

### Mobile App (React Native / Fetch)

```javascript
const ESP32_IP = '192.168.1.100';

async function fetchSensorData() {
  try {
    const response = await fetch(`http://${ESP32_IP}/api/sensors`);
    const data = await response.json();
    return data;
  } catch (error) {
    console.error('Failed to fetch:', error);
    return null;
  }
}

// For React Native WebSocket
const ws = new WebSocket(`ws://${ESP32_IP}:81`);
ws.onmessage = (e) => {
  const data = JSON.parse(e.data);
  updateUI(data);
};
```

---

## Testing Checklist

### Hardware Verification

- [ ] DHT11 returns valid temperature and humidity
- [ ] LDR responds to light changes
- [ ] LED indicator works (if added)

### Network Verification

- [ ] ESP32 connects to Wi-Fi
- [ ] Can access web dashboard in browser
- [ ] `/api/sensors` returns JSON
- [ ] `/api/status` returns JSON
- [ ] `/api/history` returns array of readings

### WebSocket Verification

- [ ] Dashboard shows "Connected via WebSocket"
- [ ] Data updates every second
- [ ] Log shows received values
- [ ] Reconnects after disconnect

### Integration Verification

- [ ] curl commands return expected data
- [ ] Browser extension can fetch data (CORS works)
- [ ] Multiple WebSocket clients can connect

---

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| No sensor data | DHT not initialized | Check wiring, add delay after begin() |
| WebSocket won't connect | Port blocked | Check firewall, use port 81 |
| CORS errors | Missing headers | Verify addCorsHeaders() called |
| Data stops updating | WiFi disconnected | Check reconnection logic |
| Memory leak | JSON buffer too small | Increase JSON_BUFFER_SIZE |
| Slow response | Blocking code | Review async patterns |

---

## Checklist

- [ ] Project structure matches specification
- [ ] All configuration in config.h
- [ ] Credentials in secrets.h (gitignored)
- [ ] SensorManager reads all sensors non-blocking
- [ ] HTTP API serves JSON with CORS headers
- [ ] WebSocket broadcasts live data
- [ ] Web dashboard displays real-time updates
- [ ] Integration examples tested
- [ ] API documentation complete

---

## Next Steps

Proceed to `10-ota-updates.md` to learn how to update firmware wirelessly for easier development and maintenance.
