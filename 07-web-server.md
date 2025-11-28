# 07 - Simple Web Server on ESP32

## Overview

This guide covers creating a web server on ESP32 to serve HTML pages and respond to HTTP requests. This forms the foundation for remote control and future integration with mobile apps and browser extensions.

---

## Prerequisites

- Completed `01-setup.md` through `06-wifi-basics.md`
- ESP32 connected to Wi-Fi network
- Computer/phone on same network for testing

---

## Web Server Concepts

### Request-Response Model

```
┌────────┐        HTTP Request         ┌────────┐
│Browser │ ────────────────────────►   │ ESP32  │
│        │  GET /                      │ Server │
│        │                             │        │
│        │        HTTP Response        │        │
│        │ ◄────────────────────────   │        │
│        │  200 OK + HTML content      │        │
└────────┘                             └────────┘
```

### HTTP Methods

| Method | Purpose | Example |
|--------|---------|---------|
| GET | Retrieve data/page | `GET /status` |
| POST | Send data to server | `POST /config` with form data |
| PUT | Update resource | `PUT /settings` |
| DELETE | Remove resource | `DELETE /log` |

---

## Project Structure

```
projects/06-web-server/
├── include/
│   └── secrets.h
├── lib/
├── src/
│   └── main.cpp
├── data/              ← For SPIFFS files (optional)
│   └── index.html
└── platformio.ini
```

### platformio.ini

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200

; Enable SPIFFS upload
board_build.filesystem = spiffs
```

---

## Step 1: Minimal Web Server

### src/main.cpp

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <WebServer.h>
#include "secrets.h"

WebServer server(80);  // Port 80 (HTTP default)

const int LED_PIN = 2;
bool ledState = false;

void handleRoot() {
  String html = "<!DOCTYPE html><html><head>";
  html += "<meta name='viewport' content='width=device-width, initial-scale=1'>";
  html += "<title>ESP32 Control</title></head><body>";
  html += "<h1>ESP32 Web Server</h1>";
  html += "<p>LED is currently: " + String(ledState ? "ON" : "OFF") + "</p>";
  html += "<p><a href='/led/on'>Turn ON</a> | <a href='/led/off'>Turn OFF</a></p>";
  html += "</body></html>";
  
  server.send(200, "text/html", html);
}

void handleLedOn() {
  ledState = true;
  digitalWrite(LED_PIN, HIGH);
  server.sendHeader("Location", "/");
  server.send(303);  // Redirect to root
}

void handleLedOff() {
  ledState = false;
  digitalWrite(LED_PIN, LOW);
  server.sendHeader("Location", "/");
  server.send(303);
}

void handleNotFound() {
  server.send(404, "text/plain", "404: Not Found");
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  
  // Connect to WiFi
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();
  Serial.printf("Connected! IP: %s\n", WiFi.localIP().toString().c_str());
  
  // Define routes
  server.on("/", handleRoot);
  server.on("/led/on", handleLedOn);
  server.on("/led/off", handleLedOff);
  server.onNotFound(handleNotFound);
  
  // Start server
  server.begin();
  Serial.println("HTTP server started");
}

void loop() {
  server.handleClient();
}
```

### Testing

1. Upload code to ESP32
2. Open Serial Monitor, note IP address
3. Open browser, navigate to `http://<IP_ADDRESS>`
4. Click LED control links

---

## Step 2: Styled HTML Interface

```cpp
void handleRoot() {
  String html = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>ESP32 Control Panel</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      max-width: 600px;
      margin: 0 auto;
      padding: 20px;
      background: #1a1a2e;
      color: #eee;
    }
    h1 { color: #0f9; }
    .card {
      background: #16213e;
      padding: 20px;
      border-radius: 10px;
      margin: 10px 0;
    }
    .btn {
      display: inline-block;
      padding: 15px 30px;
      margin: 5px;
      border-radius: 5px;
      text-decoration: none;
      font-weight: bold;
    }
    .btn-on { background: #0f9; color: #000; }
    .btn-off { background: #f44; color: #fff; }
    .status { font-size: 24px; margin: 10px 0; }
    .indicator {
      display: inline-block;
      width: 20px;
      height: 20px;
      border-radius: 50%;
      margin-right: 10px;
    }
    .indicator.on { background: #0f9; box-shadow: 0 0 10px #0f9; }
    .indicator.off { background: #666; }
  </style>
</head>
<body>
  <h1>ESP32 Control Panel</h1>
  <div class="card">
    <h2>LED Control</h2>
    <div class="status">
      <span class="indicator %INDICATOR%"></span>
      LED is %STATE%
    </div>
    <a href="/led/on" class="btn btn-on">Turn ON</a>
    <a href="/led/off" class="btn btn-off">Turn OFF</a>
  </div>
  <div class="card">
    <h2>System Info</h2>
    <p>IP Address: %IP%</p>
    <p>Uptime: %UPTIME% seconds</p>
    <p>Free Heap: %HEAP% bytes</p>
  </div>
</body>
</html>
)rawliteral";

  // Replace placeholders
  html.replace("%STATE%", ledState ? "ON" : "OFF");
  html.replace("%INDICATOR%", ledState ? "on" : "off");
  html.replace("%IP%", WiFi.localIP().toString());
  html.replace("%UPTIME%", String(millis() / 1000));
  html.replace("%HEAP%", String(ESP.getFreeHeap()));
  
  server.send(200, "text/html", html);
}
```

---

## Step 3: JSON API Endpoints

For mobile app and browser extension integration:

```cpp
#include <ArduinoJson.h>  // Add to lib_deps

void handleApiStatus() {
  StaticJsonDocument<200> doc;
  
  doc["led"] = ledState;
  doc["uptime"] = millis() / 1000;
  doc["heap"] = ESP.getFreeHeap();
  doc["rssi"] = WiFi.RSSI();
  doc["ip"] = WiFi.localIP().toString();
  
  String response;
  serializeJson(doc, response);
  
  server.send(200, "application/json", response);
}

void handleApiLed() {
  if (server.method() == HTTP_GET) {
    // GET /api/led - return current state
    StaticJsonDocument<50> doc;
    doc["state"] = ledState;
    String response;
    serializeJson(doc, response);
    server.send(200, "application/json", response);
    
  } else if (server.method() == HTTP_POST) {
    // POST /api/led - set state
    if (server.hasArg("plain")) {
      StaticJsonDocument<50> doc;
      DeserializationError error = deserializeJson(doc, server.arg("plain"));
      
      if (error) {
        server.send(400, "application/json", "{\"error\":\"Invalid JSON\"}");
        return;
      }
      
      if (doc.containsKey("state")) {
        ledState = doc["state"].as<bool>();
        digitalWrite(LED_PIN, ledState);
        server.send(200, "application/json", "{\"success\":true}");
      } else {
        server.send(400, "application/json", "{\"error\":\"Missing 'state' field\"}");
      }
    }
  }
}

void handleApiToggle() {
  ledState = !ledState;
  digitalWrite(LED_PIN, ledState);
  
  StaticJsonDocument<50> doc;
  doc["state"] = ledState;
  String response;
  serializeJson(doc, response);
  server.send(200, "application/json", response);
}

// In setup(), add routes:
server.on("/api/status", HTTP_GET, handleApiStatus);
server.on("/api/led", handleApiLed);  // Handles GET and POST
server.on("/api/toggle", HTTP_POST, handleApiToggle);
```

### platformio.ini (add ArduinoJson)

```ini
lib_deps = 
    bblanchon/ArduinoJson@^6.21.0
```

### API Reference

| Endpoint | Method | Description | Response |
|----------|--------|-------------|----------|
| `/api/status` | GET | System status | `{"led":true,"uptime":123,...}` |
| `/api/led` | GET | LED state | `{"state":true}` |
| `/api/led` | POST | Set LED | Body: `{"state":true}` |
| `/api/toggle` | POST | Toggle LED | `{"state":false}` |

### Testing with curl

```bash
# Get status
curl http://192.168.1.100/api/status

# Get LED state
curl http://192.168.1.100/api/led

# Set LED on
curl -X POST -H "Content-Type: application/json" \
     -d '{"state":true}' \
     http://192.168.1.100/api/led

# Toggle LED
curl -X POST http://192.168.1.100/api/toggle
```

---

## Step 4: CORS Headers for Browser Extension

Browser extensions need CORS headers to make cross-origin requests:

```cpp
void addCorsHeaders() {
  server.sendHeader("Access-Control-Allow-Origin", "*");
  server.sendHeader("Access-Control-Allow-Methods", "GET, POST, OPTIONS");
  server.sendHeader("Access-Control-Allow-Headers", "Content-Type");
}

void handleCorsOptions() {
  addCorsHeaders();
  server.send(204);
}

void handleApiStatus() {
  addCorsHeaders();
  // ... rest of handler
}

// In setup():
server.on("/api/status", HTTP_GET, handleApiStatus);
server.on("/api/status", HTTP_OPTIONS, handleCorsOptions);
server.on("/api/led", HTTP_OPTIONS, handleCorsOptions);
// ... add OPTIONS handler for each API endpoint
```

---

## Step 5: Serving Files from SPIFFS

For larger HTML/CSS/JS files:

### data/index.html

Create `data/` folder in project root with your HTML file.

### Code Changes

```cpp
#include <SPIFFS.h>

void setup() {
  // ... WiFi setup ...
  
  // Initialize SPIFFS
  if (!SPIFFS.begin(true)) {
    Serial.println("SPIFFS mount failed");
    return;
  }
  
  // Serve files from SPIFFS
  server.serveStatic("/", SPIFFS, "/index.html");
  server.serveStatic("/style.css", SPIFFS, "/style.css");
  server.serveStatic("/script.js", SPIFFS, "/script.js");
  
  // API routes
  server.on("/api/status", HTTP_GET, handleApiStatus);
  
  server.begin();
}
```

### Upload SPIFFS

In VS Code/PlatformIO:
1. Click PlatformIO icon
2. Select project environment
3. Click "Upload Filesystem Image"

Or via terminal:
```bash
pio run --target uploadfs
```

---

## Step 6: Complete Server Template

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <WebServer.h>
#include <ArduinoJson.h>
#include "secrets.h"

WebServer server(80);

const int LED_PIN = 2;
bool ledState = false;

// Forward declarations
void handleRoot();
void handleApiStatus();
void handleApiLed();
void handleApiToggle();
void handleNotFound();
void addCorsHeaders();

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  
  // Connect WiFi
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  
  Serial.print("Connecting");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();
  Serial.printf("IP: %s\n", WiFi.localIP().toString().c_str());
  
  // Routes
  server.on("/", HTTP_GET, handleRoot);
  server.on("/api/status", HTTP_GET, handleApiStatus);
  server.on("/api/led", HTTP_GET, handleApiLed);
  server.on("/api/led", HTTP_POST, handleApiLed);
  server.on("/api/toggle", HTTP_POST, handleApiToggle);
  server.onNotFound(handleNotFound);
  
  server.begin();
  Serial.println("Server started");
}

void loop() {
  server.handleClient();
}

void addCorsHeaders() {
  server.sendHeader("Access-Control-Allow-Origin", "*");
  server.sendHeader("Access-Control-Allow-Methods", "GET, POST");
  server.sendHeader("Access-Control-Allow-Headers", "Content-Type");
}

void handleRoot() {
  String html = R"(
<!DOCTYPE html>
<html><head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>ESP32</title>
<style>
*{box-sizing:border-box}
body{font-family:system-ui;background:#111;color:#eee;padding:20px;max-width:500px;margin:auto}
h1{color:#0f9}
.card{background:#222;padding:20px;border-radius:8px;margin:15px 0}
button{padding:12px 24px;margin:5px;border:none;border-radius:4px;cursor:pointer;font-size:16px}
.on{background:#0f9;color:#000}
.off{background:#f44;color:#fff}
#status{padding:10px;margin:10px 0;border-radius:4px}
</style>
</head><body>
<h1>ESP32 Control</h1>
<div class="card">
<h2>LED</h2>
<div id="status">Loading...</div>
<button class="on" onclick="setLed(true)">ON</button>
<button class="off" onclick="setLed(false)">OFF</button>
<button onclick="toggle()">Toggle</button>
</div>
<div class="card" id="info"></div>
<script>
async function fetchStatus(){
  try{
    const r=await fetch('/api/status');
    const d=await r.json();
    document.getElementById('status').innerHTML=
      `LED: <strong>${d.led?'ON':'OFF'}</strong>`;
    document.getElementById('status').style.background=d.led?'#0f94':'#f444';
    document.getElementById('info').innerHTML=
      `<p>Uptime: ${d.uptime}s</p><p>RSSI: ${d.rssi} dBm</p><p>Heap: ${d.heap}</p>`;
  }catch(e){document.getElementById('status').innerHTML='Error'}
}
async function setLed(state){
  await fetch('/api/led',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({state})});
  fetchStatus();
}
async function toggle(){
  await fetch('/api/toggle',{method:'POST'});
  fetchStatus();
}
fetchStatus();
setInterval(fetchStatus,5000);
</script>
</body></html>
)";
  server.send(200, "text/html", html);
}

void handleApiStatus() {
  addCorsHeaders();
  StaticJsonDocument<200> doc;
  doc["led"] = ledState;
  doc["uptime"] = millis() / 1000;
  doc["heap"] = ESP.getFreeHeap();
  doc["rssi"] = WiFi.RSSI();
  String response;
  serializeJson(doc, response);
  server.send(200, "application/json", response);
}

void handleApiLed() {
  addCorsHeaders();
  if (server.method() == HTTP_GET) {
    StaticJsonDocument<50> doc;
    doc["state"] = ledState;
    String response;
    serializeJson(doc, response);
    server.send(200, "application/json", response);
  } else if (server.method() == HTTP_POST) {
    StaticJsonDocument<50> doc;
    deserializeJson(doc, server.arg("plain"));
    if (doc.containsKey("state")) {
      ledState = doc["state"];
      digitalWrite(LED_PIN, ledState);
      server.send(200, "application/json", "{\"success\":true}");
    } else {
      server.send(400, "application/json", "{\"error\":\"Missing state\"}");
    }
  }
}

void handleApiToggle() {
  addCorsHeaders();
  ledState = !ledState;
  digitalWrite(LED_PIN, ledState);
  StaticJsonDocument<50> doc;
  doc["state"] = ledState;
  String response;
  serializeJson(doc, response);
  server.send(200, "application/json", response);
}

void handleNotFound() {
  server.send(404, "text/plain", "404 Not Found");
}
```

---

## Integration Notes for App/Extension

### Mobile App Integration

```
API Base URL: http://<ESP32_IP>

Endpoints:
- GET  /api/status  → Device status JSON
- GET  /api/led     → LED state
- POST /api/led     → Set LED (body: {"state": true/false})
- POST /api/toggle  → Toggle LED

Response format: JSON
CORS: Enabled for all origins
```

### Browser Extension Integration

```javascript
// In extension background.js or content script
const ESP32_URL = 'http://192.168.1.100';

async function getStatus() {
  const response = await fetch(`${ESP32_URL}/api/status`);
  return response.json();
}

async function toggleLed() {
  const response = await fetch(`${ESP32_URL}/api/toggle`, {
    method: 'POST'
  });
  return response.json();
}
```

---

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Can't reach server | Wrong IP | Check Serial Monitor for IP |
| Connection refused | Server not started | Verify `server.begin()` called |
| CORS error in browser | Missing headers | Add CORS headers to responses |
| Slow response | Long processing in handler | Keep handlers fast, use async patterns |
| "Entity too large" | POST body too big | Increase buffer or stream data |
| Memory issues | Too many clients | Limit concurrent connections |

---

## Security Considerations

For local network use only. For production:

1. **Authentication**: Add basic auth or token verification
2. **HTTPS**: Use ESP32 SSL capabilities (complex)
3. **Input validation**: Always validate incoming data
4. **Rate limiting**: Prevent DoS attacks

Basic auth example:

```cpp
bool isAuthenticated() {
  if (!server.authenticate("admin", "password123")) {
    server.requestAuthentication();
    return false;
  }
  return true;
}

void handleSecureEndpoint() {
  if (!isAuthenticated()) return;
  // Handle request
}
```

---

## Checklist

- [ ] Basic web server serves HTML page
- [ ] LED can be controlled via web interface
- [ ] JSON API endpoints return valid JSON
- [ ] CORS headers added for browser extension compatibility
- [ ] Tested API with curl or Postman
- [ ] Understand HTTP methods (GET, POST)
- [ ] Know how to serve static files from SPIFFS
- [ ] Web interface updates dynamically (JavaScript fetch)
- [ ] API documentation ready for mobile app integration

---

## Next Steps

Proceed to `08-async-patterns.md` to learn non-blocking programming patterns essential for responsive embedded applications.
