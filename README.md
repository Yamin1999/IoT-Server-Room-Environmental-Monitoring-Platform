# IoT Server Room Environmental Monitoring System

An embedded IoT system for monitoring environmental conditions in server rooms using an STM32 microcontroller, ESP32 Wi-Fi gateway, MQTT, FastAPI, PostgreSQL, and a web dashboard.

The system is designed to provide both **local embedded monitoring** and **remote network-based monitoring**, allowing environmental conditions such as temperature, humidity, and atmospheric pressure to be observed in real time and historically.

---

## Project Overview

Server rooms require continuous environmental monitoring because excessive temperature or humidity can affect equipment reliability and availability.

This project explores the design and implementation of an end-to-end embedded IoT monitoring platform.

The system will use an **STM32F446RE** as the primary sensor and monitoring controller. An **ESP32** will act as the network gateway, forwarding telemetry between the STM32 and an MQTT broker over Wi-Fi.

The backend will consume the telemetry, store historical measurements in PostgreSQL, evaluate environmental thresholds, and expose APIs for monitoring and configuration.

A web dashboard will provide a centralized interface for viewing the current environment, historical measurements, and alarm events.

---

## System Architecture

```text
                 ┌──────────────────────┐
                 │      BME280 Sensor   │
                 │ Temperature          │
                 │ Humidity             │
                 │ Pressure             │
                 └──────────┬───────────┘
                            │ I2C
                            ▼
                 ┌──────────────────────┐
                 │     STM32F446RE      │
                 │                      │
                 │ Environmental        │
                 │ Monitoring           │
                 │ Local Alarm Logic    │
                 │ OLED Display         │
                 └──────────┬───────────┘
                            │ UART
                            ▼
                 ┌──────────────────────┐
                 │        ESP32         │
                 │                      │
                 │ UART Gateway         │
                 │ Wi-Fi                │
                 │ MQTT Client          │
                 └──────────┬───────────┘
                            │ Wi-Fi / MQTT
                            ▼
                 ┌──────────────────────┐
                 │   MQTT Broker        │
                 │     Mosquitto        │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │    FastAPI Backend   │
                 │                      │
                 │ REST API             │
                 │ MQTT Consumer        │
                 │ Alarm Evaluation     │
                 │ Device Management    │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │     PostgreSQL       │
                 │                      │
                 │ Telemetry History    │
                 │ Device Data          │
                 │ Thresholds           │
                 │ Alarm Events         │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │    Web Dashboard     │
                 │                      │
                 │ Current Measurements │
                 │ Historical Charts    │
                 │ Alarm Status         │
                 │ Configuration        │
                 └──────────────────────┘
```

---

## Main Components

### 1. STM32 Environmental Monitoring Controller

The STM32F446RE will be responsible for the embedded monitoring layer.

Planned responsibilities include:

- Reading temperature, humidity, and pressure from the BME280.
- Displaying measurements locally on an SSD1306 OLED.
- Maintaining local environmental thresholds.
- Detecting local warning and critical conditions.
- Driving a local LED/alarm indicator.
- Communicating telemetry and commands through UART.
- Continuing local monitoring even when the network or backend is unavailable.

The firmware will be developed using **bare-metal C** with direct peripheral/register-level programming where appropriate.

No RTOS or high-level STM32 HAL will be required for the core firmware.

---

### 2. ESP32 Connectivity Gateway

The ESP32 will provide network connectivity for the STM32.

Its primary responsibilities will be:

- Communicating with the STM32 over UART.
- Connecting to a Wi-Fi network.
- Publishing STM32 telemetry to MQTT.
- Receiving MQTT commands.
- Forwarding applicable commands to the STM32.
- Reporting gateway/device status.

The ESP32 is intended to remain primarily a **connectivity gateway**, while environmental monitoring and local safety logic remain on the STM32.

---

### 3. MQTT Broker

Mosquitto will provide the messaging layer between the ESP32 gateway and the backend.

Initial MQTT topics:

```text
iot/stm32/device01/telemetry
iot/stm32/device01/command
iot/stm32/device01/status
iot/stm32/device01/alarm
```

Example telemetry:

```json
{
    "device_id": "stm32-01",
    "temperature": 30.43,
    "humidity": 70.6,
    "pressure": 1009
}
```

The MQTT protocol will be documented separately as the project develops.

---

### 4. FastAPI Backend

The backend will provide the application and API layer.

Planned responsibilities:

- Consuming MQTT telemetry.
- Validating incoming device data.
- Storing telemetry in PostgreSQL.
- Managing registered devices.
- Managing environmental thresholds.
- Evaluating environmental alarm conditions.
- Recording alarm events.
- Sending notifications.
- Providing REST APIs for the web and mobile applications.
- Providing real-time updates through WebSockets.

---

### 5. PostgreSQL Database

PostgreSQL will store persistent system data.

Planned data includes:

- Device information.
- Environmental telemetry.
- Environmental thresholds.
- Alarm events.
- Device status/heartbeat information.
- Configuration data.

Database design will be documented separately once the backend implementation begins.

---

### 6. Web Dashboard

The web dashboard will provide a centralized monitoring interface.

Planned functionality includes:

- Current temperature.
- Current humidity.
- Current pressure.
- Device connection status.
- Environmental alarm status.
- Historical telemetry charts.
- Alarm history.
- Threshold configuration.
- Real-time updates.

---

### 7. Mobile Application

A mobile application is planned as a later stage of the project.

It will use the same backend APIs and will provide functionality such as:

- Device monitoring.
- Current environmental readings.
- Alarm status.
- Threshold configuration.
- Alarm notifications.

The mobile application will not be part of the initial implementation.

---

## Environmental Alarm Model

The system will support environmental thresholds for conditions such as excessive temperature and humidity.

The initial alarm states are planned as:

```text
NORMAL
   │
   ├── threshold exceeded
   ▼
WARNING
   │
   ├── critical threshold exceeded
   ▼
CRITICAL
```

Hysteresis will be used where appropriate to prevent the alarm from repeatedly switching between states when a measurement fluctuates around a threshold.

For example:

```text
Critical temperature:
    Trigger: >= 30°C
    Clear:   < 29°C
```

The exact thresholds will be configurable rather than permanently hard-coded.

The STM32 will maintain local alarm behavior so that a network failure does not prevent local environmental monitoring.

The backend will independently evaluate alarms for centralized monitoring and notification.

---
