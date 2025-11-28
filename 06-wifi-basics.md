# 06 - Wi-Fi Basics & Connecting to a Network

## Overview

This guide covers connecting ESP32 to Wi-Fi networks, managing credentials securely, understanding the connection lifecycle, and troubleshooting common issues.

---

## Prerequisites

- Completed `01-setup.md` through `05-sensor-basics.md`
- Working ESP32 setup
- 2.4GHz Wi-Fi network (ESP32 does not support 5GHz)
- Network SSID and password

---

## ESP32 Wi-Fi Capabilities

| Feature | Specification |
|---------|---------------|
| Standards | 802.11 b/g/n |
| Frequency | 2.4GHz only |
| Modes | Station, Access Point, or Both |
| Security | Open, WEP, WPA, WPA2, WPA3 (newer firmware) |
| Simultaneous | Can be AP and Station together |

### Wi-Fi Modes

```
STATION MODE (STA):
ESP32 connects to existing router
┌────────┐        ┌────────┐
│ ESP32  │◄──────►│ Router │◄───► Internet
│ (STA)  │        │        │
└────────┘        └────────┘

ACCESS POINT MODE (AP):
ESP32 creates its own network
┌────────┐        ┌────────┐
│ Phone  │◄──────►│ ESP32  │
│        │        │  (AP)  │
└────────┘        └────────┘

DUAL MODE (AP+STA):
ESP32 does both simultaneously
┌────────┐        ┌────────┐        ┌────────┐
│ Phone  │◄──────►│ ESP32  │◄──────►│ Router │
│        │        │(AP+STA)│        │        │
└────────┘        └────────┘        └────────┘
```

---

## Project Structure

```
projects/05-wifi-connect/
├── include/
│   └── secrets.h        ← Create from template
├── lib/
├── src/
│   └── main.cpp
├── platformio.ini
└── .gitignore
```

---

## Step 1: Create Secrets File

### include/secrets.h

```cpp
#ifndef SECRETS_H
#define SECRETS_H

// Wi-Fi credentials
#define WIFI_SSID "YourNetworkName"
#define WIFI_PASSWORD "YourPassword"

// Optional: Static IP configuration
// #define USE_STATIC_IP
// #define STATIC_IP "192.168.1.100"
// #define GATEWAY_IP "192.168.1.1"
// #define SUBNET_MASK "255.255.255.0"
// #define DNS_IP "8.8.8.8"

#endif
```

### .gitignore (add to project root)

```
secrets.h
.pio/
.vscode/.browse*
```

**Important**: Never commit credentials to version control.

---

## Step 2: Basic Connection

### platformio.ini

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
```

### src/main.cpp - Simple Connection

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include "secrets.h"

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println();
  Serial.println("=========================");
  Serial.println("ESP32 Wi-Fi Connection");
  Serial.println("=========================");
  
  // Set Wi-Fi mode to Station
  WiFi.mode(WIFI_STA);
  
  // Start connection
  Serial.printf("Connecting to: %s\n", WIFI_SSID);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  
  // Wait for connection
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 30) {
    delay(500);
    Serial.print(".");
    attempts++;
  }
  
  Serial.println();
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("Connected!");
    Serial.printf("IP Address: %s\n", WiFi.localIP().toString().c_str());
    Serial.printf("Signal Strength: %d dBm\n", WiFi.RSSI());
    Serial.printf("MAC Address: %s\n", WiFi.macAddress().c_str());
  } else {
    Serial.println("Connection failed!");
  }
}

void loop() {
  // Connection maintained automatically
  delay(10000);
  Serial.printf("Status: %s, RSSI: %d dBm\n", 
                WiFi.status() == WL_CONNECTED ? "Connected" : "Disconnected",
                WiFi.RSSI());
}
```

---

## Wi-Fi Connection Lifecycle

```
                    ┌─────────────────┐
                    │  WiFi.begin()   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
              ┌─────│    IDLE         │
              │     └────────┬────────┘
              │              │
              │              ▼
              │     ┌─────────────────┐
              │     │  CONNECTING     │◄────────┐
              │     └────────┬────────┘         │
              │              │                  │
              │              ▼                  │
              │     ┌─────────────────┐         │
              │     │  AUTH PENDING   │         │
              │     └────────┬────────┘         │
              │              │                  │
              │       Success│    Failure       │
              │              ▼         │        │
              │     ┌─────────────────┐│        │
              │     │   CONNECTED     ││        │
              │     └────────┬────────┘│        │
              │              │         │        │
              │    Disconnect│         └────────┤
              │              ▼                  │
              │     ┌─────────────────┐         │
              └────►│  DISCONNECTED   │─────────┘
                    └─────────────────┘
                           Reconnect
```

### WiFi Status Codes

| Constant | Value | Meaning |
|----------|-------|---------|
| `WL_IDLE_STATUS` | 0 | Idle, not attempting connection |
| `WL_NO_SSID_AVAIL` | 1 | SSID not found in scan |
| `WL_SCAN_COMPLETED` | 2 | Scan complete |
| `WL_CONNECTED` | 3 | Successfully connected |
| `WL_CONNECT_FAILED` | 4 | Connection failed |
| `WL_CONNECTION_LOST` | 5 | Connection lost |
| `WL_DISCONNECTED` | 6 | Disconnected |

---

## Step 3: Robust Connection with Reconnection

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include "secrets.h"

const unsigned long WIFI_TIMEOUT_MS = 20000;
const unsigned long WIFI_RECONNECT_INTERVAL_MS = 5000;

unsigned long lastReconnectAttempt = 0;
bool wasConnected = false;

void connectWiFi() {
  Serial.printf("[WiFi] Connecting to %s...\n", WIFI_SSID);
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  
  unsigned long startTime = millis();
  
  while (WiFi.status() != WL_CONNECTED) {
    if (millis() - startTime > WIFI_TIMEOUT_MS) {
      Serial.println("[WiFi] Connection timeout");
      return;
    }
    delay(100);
    Serial.print(".");
  }
  
  Serial.println();
  Serial.println("[WiFi] Connected!");
  Serial.printf("[WiFi] IP: %s\n", WiFi.localIP().toString().c_str());
  Serial.printf("[WiFi] Gateway: %s\n", WiFi.gatewayIP().toString().c_str());
  Serial.printf("[WiFi] DNS: %s\n", WiFi.dnsIP().toString().c_str());
  Serial.printf("[WiFi] RSSI: %d dBm\n", WiFi.RSSI());
}

void maintainWiFi() {
  if (WiFi.status() == WL_CONNECTED) {
    if (!wasConnected) {
      Serial.println("[WiFi] Connection restored");
      wasConnected = true;
    }
    return;
  }
  
  // Not connected
  if (wasConnected) {
    Serial.println("[WiFi] Connection lost!");
    wasConnected = false;
  }
  
  // Attempt reconnection with interval
  if (millis() - lastReconnectAttempt > WIFI_RECONNECT_INTERVAL_MS) {
    lastReconnectAttempt = millis();
    Serial.println("[WiFi] Attempting reconnection...");
    WiFi.disconnect();
    WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  connectWiFi();
}

void loop() {
  maintainWiFi();
  
  // Your main code here
  delay(1000);
}
```

---

## Step 4: Using Wi-Fi Events

Event-driven approach for responsive applications:

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include "secrets.h"

void onWiFiEvent(WiFiEvent_t event) {
  switch (event) {
    case ARDUINO_EVENT_WIFI_STA_START:
      Serial.println("[Event] WiFi started");
      break;
      
    case ARDUINO_EVENT_WIFI_STA_CONNECTED:
      Serial.println("[Event] Connected to AP");
      break;
      
    case ARDUINO_EVENT_WIFI_STA_GOT_IP:
      Serial.printf("[Event] Got IP: %s\n", WiFi.localIP().toString().c_str());
      break;
      
    case ARDUINO_EVENT_WIFI_STA_DISCONNECTED:
      Serial.println("[Event] Disconnected from AP");
      WiFi.begin(WIFI_SSID, WIFI_PASSWORD);  // Auto-reconnect
      break;
      
    case ARDUINO_EVENT_WIFI_STA_LOST_IP:
      Serial.println("[Event] Lost IP address");
      break;
      
    default:
      Serial.printf("[Event] Unknown event: %d\n", event);
      break;
  }
}

void setup() {
  Serial.begin(115200);
  
  // Register event handler
  WiFi.onEvent(onWiFiEvent);
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
}

void loop() {
  // Main code runs without blocking WiFi handling
  delay(1000);
}
```

---

## Step 5: Static IP Configuration

Faster connection, consistent address:

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include "secrets.h"

// Static IP configuration
IPAddress staticIP(192, 168, 1, 100);
IPAddress gateway(192, 168, 1, 1);
IPAddress subnet(255, 255, 255, 0);
IPAddress dns(8, 8, 8, 8);

void setup() {
  Serial.begin(115200);
  
  WiFi.mode(WIFI_STA);
  
  // Configure static IP BEFORE calling begin()
  if (!WiFi.config(staticIP, gateway, subnet, dns)) {
    Serial.println("Static IP configuration failed");
  }
  
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  Serial.println();
  Serial.printf("IP Address: %s\n", WiFi.localIP().toString().c_str());
}

void loop() {
  delay(1000);
}
```

---

## Step 6: Network Scanning

Find available networks:

```cpp
#include <Arduino.h>
#include <WiFi.h>

void scanNetworks() {
  Serial.println("[Scan] Starting network scan...");
  
  WiFi.mode(WIFI_STA);
  WiFi.disconnect();
  delay(100);
  
  int numNetworks = WiFi.scanNetworks();
  
  if (numNetworks == 0) {
    Serial.println("[Scan] No networks found");
    return;
  }
  
  Serial.printf("[Scan] Found %d networks:\n", numNetworks);
  Serial.println("─────────────────────────────────────────────");
  Serial.printf("%-32s %4s %4s %s\n", "SSID", "RSSI", "CH", "Security");
  Serial.println("─────────────────────────────────────────────");
  
  for (int i = 0; i < numNetworks; i++) {
    String ssid = WiFi.SSID(i);
    int rssi = WiFi.RSSI(i);
    int channel = WiFi.channel(i);
    wifi_auth_mode_t auth = WiFi.encryptionType(i);
    
    const char* authStr;
    switch (auth) {
      case WIFI_AUTH_OPEN:         authStr = "Open"; break;
      case WIFI_AUTH_WEP:          authStr = "WEP"; break;
      case WIFI_AUTH_WPA_PSK:      authStr = "WPA"; break;
      case WIFI_AUTH_WPA2_PSK:     authStr = "WPA2"; break;
      case WIFI_AUTH_WPA_WPA2_PSK: authStr = "WPA/WPA2"; break;
      case WIFI_AUTH_WPA3_PSK:     authStr = "WPA3"; break;
      default:                     authStr = "Unknown"; break;
    }
    
    Serial.printf("%-32s %4d %4d %s\n", ssid.c_str(), rssi, channel, authStr);
  }
  
  WiFi.scanDelete();  // Free memory used by scan
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  scanNetworks();
}

void loop() {
  delay(30000);
  scanNetworks();
}
```

### Expected Output

```
[Scan] Found 4 networks:
─────────────────────────────────────────────
SSID                             RSSI   CH Security
─────────────────────────────────────────────
HomeNetwork                       -45    6 WPA2
Guest_Network                     -52    6 WPA2
Neighbor_5G                       -78   36 WPA2
OpenCafe                          -82    1 Open
```

---

## Signal Strength Reference

| RSSI (dBm) | Quality | Description |
|------------|---------|-------------|
| > -50 | Excellent | Very close to AP |
| -50 to -60 | Good | Reliable connection |
| -60 to -70 | Fair | Acceptable for most uses |
| -70 to -80 | Weak | May have issues |
| < -80 | Poor | Unstable, frequent drops |

---

## Troubleshooting

### Connection Flowchart

```
WiFi won't connect
        │
        ▼
┌───────────────────┐     Yes    ┌────────────────────┐
│ Is SSID visible   │───────────►│ Check password     │
│ in network scan?  │            │ (case-sensitive)   │
└───────────────────┘            └────────────────────┘
        │ No                              │
        ▼                                 │
┌───────────────────┐                     │
│ Check:            │                     │
│ • 2.4GHz enabled? │                     │
│ • SSID broadcast? │                     │
│ • Within range?   │                     │
└───────────────────┘                     │
        │                                 │
        └─────────────────────────────────┘
                      │
                      ▼
             ┌────────────────┐
             │ Check router:  │
             │ • MAC filtering│
             │ • Client limit │
             │ • Band steering│
             └────────────────┘
```

### Common Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| "No SSID available" | 5GHz network | Use 2.4GHz network |
| Connection timeout | Wrong password | Verify password exactly |
| Connects then drops | Weak signal | Move closer to router |
| WL_CONNECT_FAILED | Auth type mismatch | Check WPA2 vs WPA3 |
| Gets IP then loses it | IP conflict | Use static IP or check DHCP |

### Debug Information

```cpp
void printWiFiDiagnostics() {
  Serial.println("─── WiFi Diagnostics ───");
  Serial.printf("Status: %d\n", WiFi.status());
  Serial.printf("SSID: %s\n", WiFi.SSID().c_str());
  Serial.printf("BSSID: %s\n", WiFi.BSSIDstr().c_str());
  Serial.printf("Channel: %d\n", WiFi.channel());
  Serial.printf("RSSI: %d dBm\n", WiFi.RSSI());
  Serial.printf("IP: %s\n", WiFi.localIP().toString().c_str());
  Serial.printf("Subnet: %s\n", WiFi.subnetMask().toString().c_str());
  Serial.printf("Gateway: %s\n", WiFi.gatewayIP().toString().c_str());
  Serial.printf("DNS: %s\n", WiFi.dnsIP().toString().c_str());
  Serial.printf("MAC: %s\n", WiFi.macAddress().c_str());
  Serial.printf("Hostname: %s\n", WiFi.getHostname());
  Serial.println("────────────────────────");
}
```

### Hostname Setting

```cpp
WiFi.setHostname("esp32-sensor");  // Call before WiFi.begin()
```

---

## Power Management

```cpp
// Disable modem sleep for responsive connections
WiFi.setSleep(false);

// Enable modem sleep for power saving
WiFi.setSleep(true);

// More aggressive power saving (may increase latency)
WiFi.setSleep(WIFI_PS_MAX_MODEM);
```

---

## ADC2 and Wi-Fi Conflict

**Important**: When Wi-Fi is active, ADC2 pins cannot be used for analog reading.

| ADC | Pins | Wi-Fi Compatible |
|-----|------|------------------|
| ADC1 | GPIO32-39 | Yes |
| ADC2 | GPIO0, 2, 4, 12-15, 25-27 | No (when Wi-Fi active) |

**Solution**: Use ADC1 pins (GPIO32-39) for analog sensors when using Wi-Fi.

---

## Checklist

- [ ] Created `secrets.h` with Wi-Fi credentials
- [ ] Added `secrets.h` to `.gitignore`
- [ ] Network is 2.4GHz (not 5GHz)
- [ ] Basic connection works
- [ ] Can see IP address in Serial Monitor
- [ ] Implemented reconnection logic
- [ ] Understand Wi-Fi status codes
- [ ] Know how to scan for networks
- [ ] Understand RSSI values
- [ ] Aware of ADC2/Wi-Fi conflict

---

## Next Steps

Proceed to `07-web-server.md` to create a web server on your ESP32 for remote control and monitoring.
