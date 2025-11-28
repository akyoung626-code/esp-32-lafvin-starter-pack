# 10 - Project: OTA Updates (Over-The-Air)

## Overview

OTA updates let you upload new firmware over Wi-Fi, eliminating the need for USB connections during development and enabling remote updates in deployed devices.

---

## Prerequisites

- Completed `01-setup.md` through `09-sensor-stream.md`
- ESP32 connected to Wi-Fi
- Understanding of PlatformIO build process

---

## OTA Methods Comparison

| Method | Use Case | Complexity | Security |
|--------|----------|------------|----------|
| ArduinoOTA | Development | Low | Basic (password) |
| Web OTA | User updates | Medium | Password/HTTPS |
| HTTP Server Pull | Fleet management | High | TLS + Signing |

This guide covers ArduinoOTA and Web OTA.

---

## Project Structure

```
projects/09-ota-updates/
├── include/
│   ├── secrets.h
│   ├── config.h
│   └── ota_manager.h
├── lib/
├── src/
│   ├── main.cpp
│   └── ota_manager.cpp
├── data/
│   └── update.html
└── platformio.ini
```

---

## Method 1: ArduinoOTA (IDE Integration)

### platformio.ini

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200

; OTA settings
upload_protocol = espota
upload_port = 192.168.1.100  ; Replace with your ESP32's IP
upload_flags = 
    --port=3232
    --auth=your_ota_password

; For initial USB upload, comment above and use:
; upload_protocol = esptool
```

### include/config.h

```cpp
#ifndef CONFIG_H
#define CONFIG_H

#define DEVICE_HOSTNAME "esp32-sensor"
#define OTA_PASSWORD "your_ota_password"
#define OTA_PORT 3232

#define FIRMWARE_VERSION "1.0.0"

#endif
```

### Basic ArduinoOTA Implementation

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <ArduinoOTA.h>
#include "secrets.h"
#include "config.h"

void setupOTA() {
  // Set hostname
  ArduinoOTA.setHostname(DEVICE_HOSTNAME);
  
  // Set password
  ArduinoOTA.setPassword(OTA_PASSWORD);
  
  // Set port (default 3232)
  ArduinoOTA.setPort(OTA_PORT);
  
  // Callbacks
  ArduinoOTA.onStart([]() {
    String type = (ArduinoOTA.getCommand() == U_FLASH) ? "sketch" : "filesystem";
    Serial.println("[OTA] Start updating " + type);
  });
  
  ArduinoOTA.onEnd([]() {
    Serial.println("\n[OTA] Update complete!");
  });
  
  ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {
    Serial.printf("[OTA] Progress: %u%%\r", (progress / (total / 100)));
  });
  
  ArduinoOTA.onError([](ota_error_t error) {
    Serial.printf("[OTA] Error[%u]: ", error);
    switch (error) {
      case OTA_AUTH_ERROR:    Serial.println("Auth Failed"); break;
      case OTA_BEGIN_ERROR:   Serial.println("Begin Failed"); break;
      case OTA_CONNECT_ERROR: Serial.println("Connect Failed"); break;
      case OTA_RECEIVE_ERROR: Serial.println("Receive Failed"); break;
      case OTA_END_ERROR:     Serial.println("End Failed"); break;
    }
  });
  
  ArduinoOTA.begin();
  Serial.printf("[OTA] Ready on %s:%d\n", WiFi.localIP().toString().c_str(), OTA_PORT);
}

void setup() {
  Serial.begin(115200);
  
  // Connect WiFi first
  WiFi.mode(WIFI_STA);
  WiFi.setHostname(DEVICE_HOSTNAME);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();
  Serial.printf("Connected! IP: %s\n", WiFi.localIP().toString().c_str());
  
  // Setup OTA after WiFi
  setupOTA();
  
  Serial.printf("Firmware version: %s\n", FIRMWARE_VERSION);
}

void loop() {
  ArduinoOTA.handle();  // Must be called regularly
  
  // Your application code here
  delay(10);
}
```

### Uploading via OTA (PlatformIO)

1. Initial upload via USB (first time only)
2. Note the IP address from Serial Monitor
3. Update `upload_port` in platformio.ini
4. Subsequent uploads: click Upload as normal

Or via command line:
```bash
pio run --target upload --upload-port 192.168.1.100
```

---

## Method 2: Web OTA (Browser Upload)

### platformio.ini

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200

lib_deps = 
    bblanchon/ArduinoJson@^6.21.0

board_build.filesystem = spiffs
board_build.partitions = min_spiffs.csv
```

### Partition Scheme

For OTA, you need space for two firmware copies. Use `min_spiffs.csv` or custom:

```
# Name,   Type, SubType, Offset,   Size,    Flags
nvs,      data, nvs,     0x9000,   0x5000,
otadata,  data, ota,     0xe000,   0x2000,
app0,     app,  ota_0,   0x10000,  0x1E0000,
app1,     app,  ota_1,   0x1F0000, 0x1E0000,
spiffs,   data, spiffs,  0x3D0000, 0x30000,
```

### Web OTA Manager

#### include/ota_manager.h

```cpp
#ifndef OTA_MANAGER_H
#define OTA_MANAGER_H

#include <Arduino.h>
#include <WebServer.h>
#include <Update.h>

class OTAManager {
public:
  OTAManager(WebServer& server);
  
  void begin();
  void setCredentials(const char* username, const char* password);
  
  String getFirmwareVersion();
  void setFirmwareVersion(const char* version);
  
private:
  WebServer& _server;
  const char* _username;
  const char* _password;
  const char* _version;
  
  void handleUpdatePage();
  void handleUpdate();
  void handleUpdateUpload();
  bool authenticate();
};

#endif
```

#### src/ota_manager.cpp

```cpp
#include "ota_manager.h"

OTAManager::OTAManager(WebServer& server) : _server(server) {
  _username = "admin";
  _password = "admin";
  _version = "1.0.0";
}

void OTAManager::setCredentials(const char* username, const char* password) {
  _username = username;
  _password = password;
}

void OTAManager::setFirmwareVersion(const char* version) {
  _version = version;
}

String OTAManager::getFirmwareVersion() {
  return String(_version);
}

bool OTAManager::authenticate() {
  if (!_server.authenticate(_username, _password)) {
    _server.requestAuthentication();
    return false;
  }
  return true;
}

void OTAManager::begin() {
  _server.on("/update", HTTP_GET, [this]() { handleUpdatePage(); });
  _server.on("/update", HTTP_POST, 
    [this]() { handleUpdate(); },
    [this]() { handleUpdateUpload(); }
  );
  
  Serial.println("[OTA] Web update endpoint ready at /update");
}

void OTAManager::handleUpdatePage() {
  if (!authenticate()) return;
  
  String html = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Firmware Update</title>
  <style>
    * { box-sizing: border-box; }
    body { 
      font-family: system-ui, sans-serif; 
      background: #1a1a2e; 
      color: #eee; 
      padding: 40px 20px;
      max-width: 500px;
      margin: 0 auto;
    }
    h1 { color: #00ff99; }
    .card {
      background: #16213e;
      padding: 30px;
      border-radius: 12px;
      margin: 20px 0;
    }
    .info { 
      background: #0d1b2a; 
      padding: 15px; 
      border-radius: 8px; 
      margin-bottom: 20px;
      font-family: monospace;
    }
    .info span { color: #00ff99; }
    input[type="file"] {
      display: block;
      width: 100%;
      padding: 15px;
      margin: 15px 0;
      background: #0d1b2a;
      border: 2px dashed #333;
      border-radius: 8px;
      color: #eee;
      cursor: pointer;
    }
    input[type="file"]:hover {
      border-color: #00ff99;
    }
    button {
      width: 100%;
      padding: 15px;
      background: #00ff99;
      color: #000;
      border: none;
      border-radius: 8px;
      font-size: 16px;
      font-weight: bold;
      cursor: pointer;
    }
    button:hover { background: #00cc7a; }
    button:disabled { background: #333; color: #666; cursor: not-allowed; }
    .progress {
      width: 100%;
      height: 30px;
      background: #0d1b2a;
      border-radius: 8px;
      overflow: hidden;
      margin: 15px 0;
      display: none;
    }
    .progress-bar {
      height: 100%;
      background: #00ff99;
      width: 0%;
      transition: width 0.3s;
      display: flex;
      align-items: center;
      justify-content: center;
      color: #000;
      font-weight: bold;
    }
    .status { margin-top: 15px; text-align: center; }
    .warning {
      background: #442200;
      padding: 10px;
      border-radius: 8px;
      margin-bottom: 20px;
      border-left: 4px solid #ff9900;
    }
  </style>
</head>
<body>
  <h1>Firmware Update</h1>
  
  <div class="card">
    <div class="info">
      Current Version: <span>%VERSION%</span><br>
      Free Space: <span>%FREESPACE%</span> bytes
    </div>
    
    <div class="warning">
      ⚠️ Do not disconnect power during update
    </div>
    
    <form id="uploadForm" enctype="multipart/form-data">
      <input type="file" name="firmware" id="firmware" accept=".bin" required>
      <div class="progress" id="progress">
        <div class="progress-bar" id="progressBar">0%</div>
      </div>
      <button type="submit" id="submitBtn">Upload Firmware</button>
    </form>
    
    <div class="status" id="status"></div>
  </div>

  <script>
    const form = document.getElementById('uploadForm');
    const progress = document.getElementById('progress');
    const progressBar = document.getElementById('progressBar');
    const status = document.getElementById('status');
    const submitBtn = document.getElementById('submitBtn');
    
    form.addEventListener('submit', async (e) => {
      e.preventDefault();
      
      const fileInput = document.getElementById('firmware');
      if (!fileInput.files.length) {
        status.textContent = 'Please select a file';
        return;
      }
      
      const file = fileInput.files[0];
      if (!file.name.endsWith('.bin')) {
        status.textContent = 'Please select a .bin file';
        return;
      }
      
      submitBtn.disabled = true;
      progress.style.display = 'block';
      status.textContent = 'Uploading...';
      
      const formData = new FormData();
      formData.append('firmware', file);
      
      const xhr = new XMLHttpRequest();
      
      xhr.upload.addEventListener('progress', (e) => {
        if (e.lengthComputable) {
          const percent = Math.round((e.loaded / e.total) * 100);
          progressBar.style.width = percent + '%';
          progressBar.textContent = percent + '%';
        }
      });
      
      xhr.addEventListener('load', () => {
        if (xhr.status === 200) {
          status.innerHTML = '✅ Update successful! Rebooting...<br>Page will refresh in 10 seconds.';
          setTimeout(() => location.reload(), 10000);
        } else {
          status.textContent = '❌ Update failed: ' + xhr.responseText;
          submitBtn.disabled = false;
        }
      });
      
      xhr.addEventListener('error', () => {
        status.textContent = '❌ Upload failed. Check connection.';
        submitBtn.disabled = false;
      });
      
      xhr.open('POST', '/update');
      xhr.send(formData);
    });
  </script>
</body>
</html>
)rawliteral";

  html.replace("%VERSION%", _version);
  html.replace("%FREESPACE%", String(ESP.getFreeSketchSpace()));
  
  _server.send(200, "text/html", html);
}

void OTAManager::handleUpdate() {
  if (Update.hasError()) {
    _server.send(500, "text/plain", "Update failed: " + String(Update.errorString()));
  } else {
    _server.send(200, "text/plain", "Update successful! Rebooting...");
    delay(1000);
    ESP.restart();
  }
}

void OTAManager::handleUpdateUpload() {
  HTTPUpload& upload = _server.upload();
  
  if (upload.status == UPLOAD_FILE_START) {
    Serial.printf("[OTA] Update: %s\n", upload.filename.c_str());
    
    if (!Update.begin(UPDATE_SIZE_UNKNOWN)) {
      Update.printError(Serial);
    }
  } else if (upload.status == UPLOAD_FILE_WRITE) {
    if (Update.write(upload.buf, upload.currentSize) != upload.currentSize) {
      Update.printError(Serial);
    }
  } else if (upload.status == UPLOAD_FILE_END) {
    if (Update.end(true)) {
      Serial.printf("[OTA] Update Success: %u bytes\n", upload.totalSize);
    } else {
      Update.printError(Serial);
    }
  }
}
```

### Main Application with Both OTA Methods

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include <WebServer.h>
#include <ArduinoOTA.h>
#include "secrets.h"
#include "config.h"
#include "ota_manager.h"

WebServer server(80);
OTAManager otaManager(server);

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n================================");
  Serial.printf("  ESP32 OTA Demo v%s\n", FIRMWARE_VERSION);
  Serial.println("================================\n");
  
  // Connect WiFi
  WiFi.mode(WIFI_STA);
  WiFi.setHostname(DEVICE_HOSTNAME);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();
  Serial.printf("IP Address: %s\n", WiFi.localIP().toString().c_str());
  
  // Setup ArduinoOTA (IDE uploads)
  ArduinoOTA.setHostname(DEVICE_HOSTNAME);
  ArduinoOTA.setPassword(OTA_PASSWORD);
  ArduinoOTA.onStart([]() { Serial.println("[OTA] IDE update starting..."); });
  ArduinoOTA.onEnd([]() { Serial.println("[OTA] IDE update complete!"); });
  ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {
    Serial.printf("[OTA] Progress: %u%%\r", (progress / (total / 100)));
  });
  ArduinoOTA.begin();
  
  // Setup Web OTA
  otaManager.setCredentials("admin", OTA_PASSWORD);
  otaManager.setFirmwareVersion(FIRMWARE_VERSION);
  otaManager.begin();
  
  // Basic routes
  server.on("/", HTTP_GET, []() {
    String html = "<h1>ESP32 OTA Demo</h1>";
    html += "<p>Firmware: " + String(FIRMWARE_VERSION) + "</p>";
    html += "<p>Uptime: " + String(millis() / 1000) + " seconds</p>";
    html += "<p><a href='/update'>Firmware Update</a></p>";
    server.send(200, "text/html", html);
  });
  
  server.begin();
  
  Serial.println("\n[System] Ready!");
  Serial.printf("[System] Web: http://%s\n", WiFi.localIP().toString().c_str());
  Serial.printf("[System] OTA Update: http://%s/update\n", WiFi.localIP().toString().c_str());
  Serial.printf("[System] ArduinoOTA Port: %d\n", OTA_PORT);
}

void loop() {
  ArduinoOTA.handle();
  server.handleClient();
  
  // Your application code here
  delay(10);
}
```

---

## Security Considerations

### Password Protection

```cpp
// Strong password
#define OTA_PASSWORD "c0mpl3x_P@ssw0rd_2024!"

// Hash-based (more secure)
ArduinoOTA.setPasswordHash("md5_hash_of_password");
```

### Network Security

```
Production Checklist:
┌────────────────────────────────────────┐
│ □ Use strong, unique OTA password      │
│ □ Limit OTA to local network only      │
│ □ Consider IP whitelisting             │
│ □ Use HTTPS for Web OTA if possible    │
│ □ Implement firmware signing           │
│ □ Add rollback capability              │
│ □ Log all OTA attempts                 │
└────────────────────────────────────────┘
```

### IP Whitelisting Example

```cpp
bool isAuthorizedIP() {
  IPAddress clientIP = server.client().remoteIP();
  
  // Allow only specific IPs
  IPAddress allowed[] = {
    IPAddress(192, 168, 1, 10),
    IPAddress(192, 168, 1, 11)
  };
  
  for (auto& ip : allowed) {
    if (clientIP == ip) return true;
  }
  
  return false;
}

void handleUpdatePage() {
  if (!isAuthorizedIP()) {
    server.send(403, "text/plain", "Forbidden");
    return;
  }
  // ... rest of handler
}
```

---

## Firmware Versioning

### Semantic Versioning

```cpp
// config.h
#define FIRMWARE_VERSION_MAJOR 1
#define FIRMWARE_VERSION_MINOR 2
#define FIRMWARE_VERSION_PATCH 3

#define FIRMWARE_VERSION "1.2.3"

// Or use build flags in platformio.ini
// build_flags = -D VERSION=\"1.2.3\"
```

### Build Number from Git

platformio.ini:
```ini
build_flags = 
    !echo "-D BUILD_NUMBER=\\\"$(git rev-parse --short HEAD)\\\""
    !echo "-D BUILD_DATE=\\\"$(date +%Y-%m-%d)\\\""
```

```cpp
// In code
Serial.printf("Version: %s (build %s, %s)\n", 
              FIRMWARE_VERSION, BUILD_NUMBER, BUILD_DATE);
```

---

## Troubleshooting

### ArduinoOTA Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| "No response from device" | Wrong IP | Verify IP in Serial Monitor |
| "Auth Failed" | Wrong password | Check OTA_PASSWORD matches |
| "Not enough space" | Partition too small | Use different partition scheme |
| "Upload timeout" | Firewall blocking | Allow port 3232 UDP |

### Web OTA Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| "Update failed" | File too large | Check partition size |
| Progress stuck at 0% | CORS or network issue | Check browser console |
| "Begin Failed" | Flash busy | Add delay before Update.begin() |
| Reboot loop | Corrupted firmware | Re-upload via USB |

### Recovery from Bad Firmware

```
If OTA update causes boot loop:

1. Hold BOOT button
2. Press EN/RESET
3. Release BOOT
4. Upload working firmware via USB

Prevention:
• Test firmware thoroughly before OTA
• Keep USB-uploaded backup version
• Implement firmware verification
```

---

## Development Workflow

```
Recommended OTA Development Flow:

┌──────────────────────────────────────────────────────────┐
│                                                          │
│  1. Initial Setup (USB)                                  │
│     └─► Upload base firmware with OTA support           │
│                                                          │
│  2. Development Cycle (OTA)                              │
│     ┌────────────────────────────────────────┐          │
│     │  Edit code                              │          │
│     │      ↓                                  │          │
│     │  Build (pio run)                        │          │
│     │      ↓                                  │          │
│     │  Upload via OTA (pio run -t upload)    │◄────┐    │
│     │      ↓                                  │     │    │
│     │  Test on device                         │     │    │
│     │      ↓                                  │     │    │
│     │  Iterate ───────────────────────────────┘     │    │
│     └────────────────────────────────────────────────    │
│                                                          │
│  3. Production Deploy                                    │
│     └─► Web OTA for field updates                       │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Complete platformio.ini Reference

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200

; Partition table for OTA
board_build.partitions = min_spiffs.csv

; SPIFFS for web files
board_build.filesystem = spiffs

; Library dependencies
lib_deps = 
    bblanchon/ArduinoJson@^6.21.0

; OTA Upload Configuration
upload_protocol = espota
upload_port = 192.168.1.100
upload_flags = 
    --port=3232
    --auth=your_ota_password

; Build flags
build_flags = 
    -D FIRMWARE_VERSION=\"1.0.0\"

; USB upload (uncomment for first upload)
; [env:esp32dev-usb]
; extends = env:esp32dev
; upload_protocol = esptool
```

---

## Checklist

- [ ] ArduinoOTA configured and tested
- [ ] Password set for OTA authentication
- [ ] Web OTA page accessible at /update
- [ ] Basic auth protects update page
- [ ] Firmware version displayed correctly
- [ ] Can upload via PlatformIO OTA
- [ ] Can upload via web browser
- [ ] Understand partition requirements
- [ ] Know recovery procedure for bad firmware
- [ ] Security considerations reviewed

---

## Summary

You've now completed the ESP32 learning series:

1. **Environment Setup** - VS Code, PlatformIO, workspace structure
2. **GPIO Basics** - Blink LED, pin assignments
3. **Serial Debug** - Logging, troubleshooting
4. **Button Input** - Debouncing, interrupts
5. **Sensor Basics** - DHT11, analog sensors
6. **Wi-Fi Connection** - Network setup, reconnection
7. **Web Server** - HTTP API, CORS
8. **Async Patterns** - Non-blocking code, state machines
9. **Sensor Stream** - Complete project with WebSocket
10. **OTA Updates** - Wireless firmware updates

### Next Learning Paths

- **MQTT** - Lightweight pub/sub messaging
- **Bluetooth** - BLE communication
- **Deep Sleep** - Battery-powered projects
- **FreeRTOS** - Multi-tasking with tasks
- **ESP-NOW** - Peer-to-peer communication
- **Matter/Thread** - Smart home protocols

Happy building!
