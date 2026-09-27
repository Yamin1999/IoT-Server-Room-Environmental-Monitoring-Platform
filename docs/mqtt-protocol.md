
# MQTT Protocol

## 1. Overview

MQTT is used as the messaging layer between the ESP32 gateway and the backend.

```text
STM32
  │
 UART
  │
ESP32
  │
 MQTT
  │
Mosquitto
  │
ASP.NET Core
```

The ESP32 acts as the MQTT client responsible for transporting device data between the STM32 and the MQTT broker.

---

# 2. Broker

The development system uses an MQTT broker such as Mosquitto.

The broker is responsible for:

- Receiving published messages
- Delivering messages to subscribers
- Managing MQTT clients
- Maintaining MQTT topic routing

The backend and ESP32 do not communicate directly with each other over a custom TCP connection.

MQTT provides the messaging layer between them.

---

# 3. Topic Structure

The initial topic namespace is:

```text
iot/stm32/device01/
```

Topics:

```text
iot/stm32/device01/telemetry
iot/stm32/device01/command
iot/stm32/device01/status
iot/stm32/device01/alarm
```

The `device01` portion identifies the monitoring device.

The topic structure should support additional devices later.

For example:

```text
iot/stm32/device02/telemetry
iot/stm32/device03/telemetry
```

---

# 4. Telemetry Topic

Topic:

```text
iot/stm32/device01/telemetry
```

Direction:

```text
STM32
  ↓
ESP32
  ↓
MQTT
  ↓
Backend
```

The ESP32 publishes environmental measurements received from the STM32.

Example payload:

```json
{
    "device_id": "stm32-01",
    "temperature": 30.43,
    "humidity": 70.6,
    "pressure": 1009
}
```

### Fields

| Field | Type | Description |
|---|---|---|
| `device_id` | string | Device identifier |
| `temperature` | number | Temperature in °C |
| `humidity` | number | Relative humidity in % |
| `pressure` | number | Atmospheric pressure in hPa |

Server-side timestamps may be added by the backend when the message is received.

---

# 5. Command Topic

Topic:

```text
iot/stm32/device01/command
```

Direction:

```text
Backend
   ↓
MQTT
   ↓
ESP32
   ↓
UART
   ↓
STM32
```

Commands are intended for remote device configuration and control.

Example:

```text
CMD:GET_STATUS
```

Another example:

```text
CMD:SET_TEMP_HIGH:30.0
```

The ESP32 receives the MQTT message and forwards the applicable command to the STM32 over UART.

---

# 6. Status Topic

Topic:

```text
iot/stm32/device01/status
```

The status topic is intended to communicate device state.

Possible information includes:

- Online/offline state
- Firmware version
- Monitoring state
- Sensor state
- Gateway state

Example payload:

```json
{
    "device_id": "stm32-01",
    "status": "online"
}
```

The exact status schema will be defined when device-status functionality is implemented.

---

# 7. Alarm Topic

Topic:

```text
iot/stm32/device01/alarm
```

The alarm topic is intended for environmental alarm events.

Example:

```json
{
    "device_id": "stm32-01",
    "type": "temperature",
    "severity": "CRITICAL",
    "value": 32.1
}
```

Possible severity levels:

```text
NORMAL
WARNING
CRITICAL
```

The final alarm schema will be defined alongside the alarm implementation.

---

# 8. Direction Summary

| Topic | Publisher | Subscriber |
|---|---|---|
| `telemetry` | ESP32 | Backend |
| `command` | Backend | ESP32 |
| `status` | ESP32 | Backend |
| `alarm` | ESP32 / device path | Backend |

The exact publisher/subscriber behavior may evolve as the system develops.

---

# 9. MQTT QoS

The project will initially use MQTT QoS according to the reliability requirements of each message type.

A possible initial configuration is:

| Message | Initial QoS |
|---|---:|
| Telemetry | 0 |
| Commands | 1 |
| Status | 1 |
| Alarm events | 1 |

### Telemetry

Telemetry is periodic, so losing one measurement is generally less significant because another measurement will arrive shortly.

Therefore QoS 0 is suitable for the initial implementation.

### Commands

Commands may change device configuration and therefore should have stronger delivery guarantees.

QoS 1 is appropriate for the initial implementation.

### Alarm events

Alarm events are more important than periodic telemetry and should use a more reliable delivery level.

QoS 1 will be used initially.

The final QoS strategy may change after testing.

---

# 10. Retained Messages

Retained MQTT messages may be used for device state/configuration where receiving the latest value immediately after subscription is useful.

This will be evaluated when device status and configuration are implemented.

Telemetry will not initially depend on retained messages.

---

# 11. Last Will and Testament

The ESP32 may use an MQTT Last Will and Testament message to indicate unexpected gateway disconnection.

For example:

```text
iot/stm32/device01/status
```

with:

```json
{
    "device_id": "stm32-01",
    "status": "offline"
}
```

This will allow the backend to detect unexpected gateway disconnections.

The exact implementation will be added when device heartbeat/status monitoring is implemented.

---

# 12. Backend Processing

When telemetry is received, the backend will:

```text
MQTT Message
     │
     ▼
Validate Payload
     │
     ▼
Identify Device
     │
     ├──────────────► Store Telemetry
     │
     ├──────────────► Evaluate Alarm
     │
     └──────────────► Publish Real-Time Update
                              │
                              ▼
                           SignalR
                              │
                              ▼
                         Web Dashboard
```

---

# 13. Security

The initial local development environment may use an unsecured MQTT broker.

For production deployment, the MQTT system should support:

- Username/password authentication
- TLS encryption
- Per-device credentials
- Restricted topic permissions
- Secure credential storage

Security will be implemented after the basic communication system is working.

---

# 14. Protocol Evolution

The MQTT protocol is expected to evolve as the system gains functionality.

Changes to:

- Topic names
- Payload formats
- QoS
- Retained messages
- Authentication
- Device addressing

should be documented in this file when introduced.
