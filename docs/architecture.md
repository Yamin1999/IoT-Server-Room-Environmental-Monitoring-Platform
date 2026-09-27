# System Architecture

## 1. Overview

The Server Room Environmental Monitoring System is an IoT-based monitoring platform designed to continuously measure environmental conditions and make the information available both locally and remotely.

The system is divided into several layers:

1. Embedded sensing and local monitoring
2. Connectivity gateway
3. Message transport
4. Backend services
5. Persistent storage
6. Web interface

The overall architecture is:

```text
                    ┌──────────────────┐
                    │     BME280       │
                    │                  │
                    │ Temperature      │
                    │ Humidity         │
                    │ Pressure         │
                    └────────┬─────────┘
                             │
                            I2C
                             │
                             ▼
                    ┌──────────────────┐
                    │    STM32F446RE   │
                    │                  │
                    │ Sensor Driver    │
                    │ Monitoring Logic │
                    │ Alarm Logic      │
                    │ OLED Display     │
                    └────────┬─────────┘
                             │
                            UART
                             │
                             ▼
                    ┌──────────────────┐
                    │      ESP32       │
                    │                  │
                    │ UART Gateway     │
                    │ Wi-Fi            │
                    │ MQTT Client      │
                    └────────┬─────────┘
                             │
                         Wi-Fi/MQTT
                             │
                             ▼
                    ┌──────────────────┐
                    │    Mosquitto     │
                    │   MQTT Broker    │
                    └────────┬─────────┘
                             │
                            MQTT
                             │
                             ▼
                    ┌──────────────────┐
                    │  ASP.NET Core    │
                    │                  │
                    │ REST API         │
                    │ MQTT Consumer    │
                    │ Alarm Service    │
                    │ Device Service   │
                    │ SignalR          │
                    └───────┬───┬──────┘
                            │   │
                     ┌──────┘   └──────────┐
                     ▼                     ▼
              ┌──────────────┐      ┌──────────────┐
              │ PostgreSQL   │      │ Web Dashboard│
              │              │      │              │
              │ Telemetry    │      │ Current Data │
              │ Devices      │      │ Charts       │
              │ Thresholds   │      │ Alarms       │
              │ Alarm Events │      │ Configuration│
              └──────────────┘      └──────────────┘
```

---

## 2. Design Principles

The system follows several design principles.

### 2.1 Local monitoring must remain functional

The STM32 is responsible for local environmental monitoring.

The system should continue measuring sensor data and displaying it locally even if:

- Wi-Fi is unavailable.
- The ESP32 is disconnected.
- The MQTT broker is unavailable.
- The backend is unavailable.

Network connectivity must not be a requirement for basic local monitoring.

### 2.2 The ESP32 is a connectivity gateway

The ESP32 primarily provides:

- Wi-Fi connectivity
- MQTT communication
- STM32 UART communication

Environmental monitoring logic should remain on the STM32 rather than being unnecessarily moved into the ESP32.

### 2.3 Backend and device responsibilities are separated

The STM32 is responsible for local operation and local safety behavior.

The backend is responsible for:

- Centralized monitoring
- Historical data
- Configuration management
- Alarm history
- Notifications
- Remote clients

This separation allows the device to remain useful even when disconnected from the backend.

---

# 3. Hardware Architecture

## 3.1 STM32

The STM32F446RE is the primary embedded controller.

Planned responsibilities:

- BME280 communication
- Environmental measurement
- Local alarm evaluation
- OLED updates
- UART communication with ESP32
- Local device state management

The firmware will be developed primarily in C using direct peripheral configuration and project-specific drivers.

---

## 3.2 BME280

The BME280 provides:

- Temperature
- Relative humidity
- Atmospheric pressure

The sensor communicates with the STM32 through I2C.

```text
STM32
  │
  │ I2C
  ▼
BME280
```

---

## 3.3 SSD1306 OLED

The OLED provides a local human-readable view of the monitoring system.

Example:

```text
SERVER ROOM

Temp : 30.4 C
Hum  : 70.6 %
Press: 1009 hPa

Status: NORMAL
```

The OLED is intentionally local. It does not depend on the network or backend.

---

## 3.4 ESP32

The ESP32 acts as the network gateway.

```text
STM32
  │
 UART
  │
ESP32
  │
 Wi-Fi
  │
 MQTT
```

The ESP32 does not directly replace the STM32 monitoring controller.

---

# 4. Firmware Architecture

The STM32 firmware will be divided into logical layers.

```text
┌──────────────────────────────┐
│       Application Layer      │
│                              │
│ Monitoring / Alarm / State   │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│      Communication Layer     │
│                              │
│ UART Protocol / Commands     │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│         Device Drivers       │
│                              │
│ BME280 / SSD1306 / I2C /     │
│ SPI / GPIO / UART / SysTick  │
└──────────────┬───────────────┘
               │
┌──────────────▼───────────────┐
│        STM32 Hardware        │
└──────────────────────────────┘
```

The application layer should not directly manipulate hardware registers whenever a suitable driver abstraction exists.

---

# 5. Local Monitoring

The STM32 periodically reads environmental measurements from the BME280.

The measurement flow is:

```text
BME280
   ↓
Sensor Driver
   ↓
Environmental Measurement
   ↓
Monitoring Logic
   ├── OLED
   ├── Local Alarm
   └── Telemetry
```

The local monitoring loop should continue independently of network connectivity.

---

# 6. Alarm Architecture

There are two related alarm mechanisms.

## 6.1 Local alarm

The STM32 evaluates environmental conditions locally.

For example:

```text
Temperature
     │
     ▼
Threshold Evaluation
         │
 ┌───────┼─────────┐
 ▼       ▼         ▼
NORMAL WARNING CRITICAL
```

Local alarm behavior may include:

- LED indication
- OLED status
- Future audible alarm or external output

## 6.2 Backend alarm

The backend independently evaluates incoming telemetry.

This allows the server to:

- Record alarm events
- Provide alarm history
- Notify remote users
- Display alarm state on the dashboard

The backend should not be the only place where safety-related local conditions are evaluated.

---

# 7. Connectivity Architecture

The STM32 and ESP32 communicate through UART.

```text
STM32 USART1
     │
     │ UART
     ▼
ESP32 UART2
```

The ESP32 converts the device-side UART communication into MQTT messages.

```text
STM32
  │ UART
  ▼
ESP32
  │ MQTT
  ▼
Mosquitto
```

The UART protocol is documented in:

```text
docs/uart-protocol.md
```

The MQTT protocol is documented in:

```text
docs/mqtt-protocol.md
```

---

# 8. Backend Architecture

The backend will be implemented using ASP.NET Core.

The planned logical structure is:

```text
┌──────────────────────────────┐
│        ASP.NET Core API      │
├──────────────────────────────┤
│ REST API                     │
│ MQTT Consumer                │
│ Device Service               │
│ Telemetry Service            │
│ Threshold Service            │
│ Alarm Service                │
│ Notification Service         │
│ SignalR                      │
└──────────────┬───────────────┘
               │
               ▼
          PostgreSQL
```

The backend will expose APIs for:

- Device information
- Current telemetry
- Historical telemetry
- Threshold configuration
- Alarm history
- Device status

---

# 9. Real-Time Web Updates

The web dashboard should not need to continuously poll the backend for every new measurement.

The planned architecture is:

```text
STM32
  ↓
ESP32
  ↓
MQTT
  ↓
ASP.NET Core
  ↓
SignalR
  ↓
Web Browser
```

When new telemetry is received, the backend can publish an update to connected dashboard clients.

---

# 10. Data Flow

## 10.1 Telemetry

```text
BME280
  ↓
STM32
  ↓ UART
ESP32
  ↓ MQTT
Mosquitto
  ↓
ASP.NET Core
  ├── PostgreSQL
  └── SignalR
          ↓
      Dashboard
```

## 10.2 Remote Configuration

```text
Dashboard
    ↓
ASP.NET Core
    ↓
MQTT
    ↓
ESP32
    ↓ UART
STM32
    ↓
Configuration
```

## 10.3 Local Display

```text
BME280
   ↓
STM32
   ↓
SSD1306 OLED
```

This path does not depend on the network.

---

# 11. Failure Handling

The system should tolerate failures at individual layers.

### Sensor failure

The STM32 should detect invalid or unavailable sensor readings where possible.

### UART failure

The STM32 should continue local monitoring even if the ESP32 is unavailable.

### Wi-Fi failure

The ESP32 should attempt to reconnect while the STM32 continues local operation.

### MQTT failure

The ESP32 should reconnect to the MQTT broker.

### Backend failure

The device should continue local monitoring.

Historical data transmission and buffering behavior may be added later.

---

# 12. Security

Security will be introduced after the basic system is functional.

Planned improvements include:

- MQTT authentication
- MQTT TLS
- Device authentication
- API authentication
- Secure credential storage
- Network access restrictions
- OTA firmware security

The initial development environment may use an unsecured local MQTT broker for simplicity.

This is intended for development only and is not a production security configuration.

---

# 13. Future Extensions

Possible future improvements include:

- Multiple monitoring devices
- Multiple server rooms
- Additional sensors
- External alarm outputs
- Email notifications
- Mobile push notifications
- Device heartbeat monitoring
- Offline telemetry buffering
- OTA firmware updates
- Role-based access control
- Deployment with Docker
- Hardware-in-the-loop testing

---

# 14. Architecture Evolution

This document describes the intended architecture.

Implementation details may evolve as development progresses.

The architecture should be updated when an actual design decision changes rather than documenting functionality before it exists.
