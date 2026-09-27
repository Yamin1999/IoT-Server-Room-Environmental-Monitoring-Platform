
# UART Protocol

## 1. Overview

The STM32 and ESP32 communicate using a UART serial connection.

The STM32 acts as the environmental monitoring controller, while the ESP32 acts as the network gateway.

```text
STM32F446RE
     │
     │ UART
     ▼
   ESP32
```

The UART interface is a device-to-device communication channel. It is independent of MQTT.

---

# 2. UART Configuration

Initial serial configuration:

| Parameter | Value |
|---|---|
| Baud rate | 115200 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Flow control | None |

Therefore:

```text
115200 8N1
```

---

# 3. Physical Connection

The initial connection is:

```text
STM32 PA9 / USART1_TX
        │
        └──────────────► ESP32 GPIO16 / UART2_RX

STM32 PA10 / USART1_RX
        │
        └──────────────► ESP32 GPIO17 / UART2_TX

STM32 GND
        └────────────── ESP32 GND
```

The STM32 and ESP32 must share a common ground.

---

# 4. Message Framing

Messages are terminated using:

```text
\r\n
```

which represents:

```text
Carriage Return + Line Feed
```

Example:

```text
CMD:GET_STATUS\r\n
```

The receiver should process a complete message after detecting the terminating `\r\n`.

---

# 5. Message Types

The protocol contains two primary message directions.

### STM32 → ESP32

Used for:

- Telemetry
- Device status
- Alarm information
- Command responses

### ESP32 → STM32

Used for:

- Configuration commands
- Control commands
- Status requests

---

# 6. Telemetry

Environmental telemetry is sent from the STM32 to the ESP32.

Example:

```json
{"device_id":"stm32-01","temperature":30.43,"humidity":70.6,"pressure":1009}
```

The ESP32 forwards this information to the MQTT broker.

The UART protocol does not require the ESP32 to understand every telemetry field.

Its primary role is to transport the message to the network layer.

---

# 7. Commands

Commands are sent from the ESP32 to the STM32.

The command format is:

```text
CMD:<COMMAND>\r\n
```

Example:

```text
CMD:GET_STATUS\r\n
```

As additional functionality is implemented, commands may include parameters.

Example:

```text
CMD:SET_TEMP_HIGH:30.0\r\n
```

---

# 8. Command Acknowledgement

Commands that require confirmation may generate a response from the STM32.

Example:

```text
CMD:GET_STATUS\r\n
```

Response:

```text
RESP:OK\r\n
```

For commands that fail:

```text
RESP:ERROR:<REASON>\r\n
```

Example:

```text
RESP:ERROR:INVALID_VALUE\r\n
```

The exact response set will be extended only when required by an implemented feature.

---

# 9. Status

The STM32 may provide its current state in response to a status request.

Example:

```text
CMD:GET_STATUS\r\n
```

Possible response:

```text
RESP:STATUS:RUNNING\r\n
```

Status information may later include:

- Monitoring state
- Alarm state
- Sensor state
- Current configuration
- Firmware version

---

# 10. Configuration

Remote configuration will eventually allow the backend to configure parameters on the STM32.

Example:

```text
CMD:SET_TEMP_HIGH:30.0\r\n
```

Possible response:

```text
RESP:OK\r\n
```

The configuration mechanism will be expanded as the monitoring and alarm system is implemented.

---

# 11. Invalid Commands

If the STM32 receives an unsupported command:

```text
CMD:UNKNOWN\r\n
```

it may respond:

```text
RESP:ERROR:UNKNOWN_CMD\r\n
```

Invalid parameters should generate an appropriate error response.

Example:

```text
RESP:ERROR:INVALID_VALUE\r\n
```

---

# 12. Non-Blocking Communication

The STM32 UART receiver must not block the main monitoring loop while waiting for incoming data.

The intended architecture is:

```text
USART1 RX
    │
    ▼
RX Buffer
    │
    ▼
Message Parser
    │
    ▼
Command Processor
```

This allows the STM32 to continue:

- Reading the BME280
- Updating the OLED
- Evaluating alarms
- Sending telemetry

while simultaneously receiving commands.

---

# 13. Protocol Evolution

The UART protocol is intentionally kept small during the initial development stage.

New commands should be added only when a corresponding system feature requires them.

Protocol changes should be documented here before or alongside their implementation.
