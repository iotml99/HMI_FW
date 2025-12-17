# HMI Communication Protocol Specification

**Document Version:** 1.0
**Date:** 2025-11-25
**Related:** [HMI_OVERVIEW.md](./HMI_OVERVIEW.md)

---

## Table of Contents

1. [Protocol Overview](#protocol-overview)
2. [Physical Layer](#physical-layer)
3. [Message Format](#message-format)
4. [Command Reference](#command-reference)
5. [Error Handling](#error-handling)
6. [Examples](#examples)

---

## Protocol Overview

### Design Philosophy

**Text-Based Protocol Choice:**
- Human-readable for debugging with serial terminal
- Simple parsing on embedded systems
- Easy to extend without breaking compatibility
- CRC-8 checksum for data integrity
- Low bandwidth usage (<2% at 115200 baud)

**Key Characteristics:**
- **Type:** Asynchronous request-response + async events
- **Format:** ASCII text with checksums
- **Direction:** Bidirectional
- **Update Rate:** 5-second periodic updates, immediate events
- **Reliability:** CRC-8, sequence numbers, timeouts, retries

---

## Physical Layer

### UART Configuration

| Parameter | Value | Notes |
|-----------|-------|-------|
| **Baud Rate** | 115200 bps | ~11.5 KB/s effective |
| **Data Bits** | 8 | Standard |
| **Parity** | None | CRC provides integrity |
| **Stop Bits** | 1 | Standard |
| **Flow Control** | None | Optional software XON/XOFF |
| **Connector** | 3-wire | TX, RX, GND |

### Wiring

```
BeagleBone Black              STM32 HMI Board
┌─────────────┐              ┌─────────────┐
│             │              │             │
│  UART4 TX ──┼─────────────►│── RX (UART1)│
│             │              │             │
│  UART4 RX ◄─┼──────────────┼── TX (UART1)│
│             │              │             │
│       GND ──┼──────────────┼── GND       │
│             │              │             │
└─────────────┘              └─────────────┘

Port: /dev/ttyO4             USART1 typical
```

### Performance

**Bandwidth Analysis:**

| Data Type | Bytes/Msg | Frequency | Bandwidth | % Used |
|-----------|-----------|-----------|-----------|--------|
| Heartbeat | 40 | 5s | 8 B/s | 0.07% |
| System Status | 80 | 5s | 16 B/s | 0.14% |
| Sensor Update (×16) | 35 | 5s each | 112 B/s | 0.97% |
| Relay Status | 40 | On change | ~2 B/s | 0.02% |
| Time Sync | 60 | 60s | 1 B/s | 0.01% |
| HMI Status | 50 | 10s | 5 B/s | 0.04% |
| **Total Periodic** | | | **~144 B/s** | **1.25%** |

**Remaining:** ~11,376 B/s (98.75%) for alerts, commands, diagnostics, firmware updates

---

## Message Format

### Basic Structure

```
#<SEQ> <COMMAND> [arguments...] *<CRC8>\r\n

Components:
  #           - Command prefix (delimiter)
  <SEQ>       - 3-digit sequence number (000-255, wraps)
  <COMMAND>   - Command string (uppercase recommended)
  [args...]   - Space-separated arguments
  *           - Checksum delimiter
  <CRC8>      - 2-character hexadecimal CRC-8
  \r\n        - Line terminator (CR+LF)
```

### Examples

```bash
#042 SYS PING 1234567890 *A3\r\n
#043 LED SET 1 255 0 0 *F2\r\n
#044 SENSOR 1 TH 24.5 58.0 OK 2 server_room *7B\r\n
```

### Field Specifications

#### Sequence Number

| Property | Value |
|----------|-------|
| **Format** | 3-digit decimal (000-255) |
| **Range** | 0-255 |
| **Wrap** | Wraps to 000 after 255 |
| **Purpose** | Track message order, detect loss |
| **Validation** | Not strictly enforced (UDP-style) |

**Behavior:**
- Each side maintains independent sequence counter
- Increment after each transmitted message
- Receiver tracks last seen sequence
- Gap detection logs warning but doesn't block
- Duplicate detection (seq == last_seq) can ignore

#### CRC-8 Checksum

**Algorithm:** CRC-8-CCITT
**Polynomial:** 0x07
**Initial Value:** 0x00
**Data:** Everything between '#' and '*' (exclusive)

**C Implementation:**
```c
uint8_t crc8_ccitt(const char *data, size_t len) {
    uint8_t crc = 0x00;
    for (size_t i = 0; i < len; i++) {
        crc ^= (uint8_t)data[i];
        for (int j = 0; j < 8; j++) {
            if (crc & 0x80)
                crc = (crc << 1) ^ 0x07;
            else
                crc <<= 1;
        }
    }
    return crc;
}

// Usage
char cmd[] = "042 SYS PING 1234567890";
uint8_t crc = crc8_ccitt(cmd, strlen(cmd));
sprintf(output, "#%s *%02X\r\n", cmd, crc);
```

**Python Implementation:**
```python
def crc8_ccitt(data: str) -> int:
    """Calculate CRC-8-CCITT checksum"""
    crc = 0x00
    for byte in data.encode('ascii'):
        crc ^= byte
        for _ in range(8):
            if crc & 0x80:
                crc = (crc << 1) ^ 0x07
            else:
                crc <<= 1
            crc &= 0xFF
    return crc

# Usage
cmd = "042 SYS PING 1234567890"
crc = crc8_ccitt(cmd)
message = f"#{cmd} *{crc:02X}\r\n"
```

### Response Format

**Success:**
```
OK [data] *<CRC8>\r\n
```

**Error:**
```
ERR <code> <message> *<CRC8>\r\n
```

**Event (Async Push):**
```
EVT <code> <data...> *<CRC8>\r\n
```

**Examples:**
```bash
OK *5A\r\n
OK PONG 1234567890 *B3\r\n
ERR E02 RANGE *8F\r\n
EVT 0x20 1 HIGH SENSOR TH4_TEMP_HIGH *C4\r\n
```

---

## Command Reference

### Message Types

#### BeagleBone → STM32 (Backend to HMI)

| Type | Command | Description | Frequency |
|------|---------|-------------|-----------|
| Heartbeat | `SYS PING` | Keep-alive | Every 5s |
| System | `SYS STATUS` | System health | Every 5s |
| Time | `SYS TIME` | Time sync | Every 60s |
| Network | `SYS NET` | Network status | Every 10s |
| Sensor | `SENSOR` | Sensor update | Every 5s per sensor |
| Relay | `RELAY` | Relay status | On change |
| Alert | `ALERT NEW` | New alert | Immediate |
| Alert | `ALERT CLEAR` | Alert cleared | Immediate |
| LED | `LED` | LED control | On change |
| LED | `LED STATUS` | Status LED | On change |
| Buzzer | `BUZZ BEEP` | Beep sound | On demand |
| Buzzer | `BUZZ PATTERN` | Pattern sound | On demand |
| Buzzer | `BUZZ STOP` | Stop sound | On demand |
| Screen | `SCR GOTO` | Change screen | On demand |
| Screen | `SCR OVERLAY` | Show overlay | On demand |
| Screen | `SCR BRIGHT` | Set brightness | On change |
| Command | `CMD PROGRESS` | Command progress | Async |
| Command | `CMD RESULT` | Command result | Async |
| Diagnostic | `DIAG RESULT` | Test result | Async |

#### STM32 → BeagleBone (HMI to Backend)

| Type | Command | Description | Frequency |
|------|---------|-------------|-----------|
| Heartbeat | `PONG` | Heartbeat response | In response |
| Button | `BTN` | Button event | On press/release |
| Authentication | `AUTH PIN` | Login attempt | On demand |
| Command | `CMD` | Execute command | On demand |
| Relay | `RELAY SET` | Control relay | On demand |
| Alert | `ALERT ACK` | Acknowledge alert | On demand |
| Screen | `SCR ACTIVE` | Screen changed | On navigation |
| Status | `HMI STATUS` | HMI health | Every 10s |
| Bootloader | `BL` commands | Firmware update | On demand |

---

### System Commands

#### SYS PING - Heartbeat

**Direction:** BB → HMI
**Purpose:** Keep-alive, verify connection
**Frequency:** Every 5 seconds

**Format:**
```bash
#<seq> SYS PING <timestamp> *<crc>
```

**Parameters:**
- `timestamp`: Unix timestamp (uint32_t as decimal string)

**Response:**
```bash
#<seq> PONG <timestamp> *<crc>
```

**Example:**
```bash
BB: #042 SYS PING 1732550400 *A3\r\n
HMI: #100 PONG 1732550400 *B7\r\n
```

**Behavior:**
- BB sends every 5s
- HMI responds immediately
- If no PING received for 10s → connection lost
- If no PONG received for 10s → HMI offline

#### SYS STATUS - System Health

**Direction:** BB → HMI
**Purpose:** Overall system status
**Frequency:** Every 5 seconds

**Format:**
```bash
#<seq> SYS STATUS <cpu> <mem> <temp> <uptime> <alerts> *<crc>
```

**Parameters:**
- `cpu`: CPU usage 0-100 (%)
- `mem`: Memory usage 0-100 (%)
- `temp`: CPU temperature 0-127 (°C, signed)
- `uptime`: Uptime in seconds (uint32_t)
- `alerts`: Active alert count 0-255

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
#043 SYS STATUS 45 58 52 453600 2 *F4\r\n
```

**Display on HMI:**
- CPU: 45%, Mem: 58%, Temp: 52°C
- Uptime: 5d 6h 0m (calculated from seconds)
- Alerts: 2 active

#### SYS TIME - Time Synchronization

**Direction:** BB → HMI
**Purpose:** Sync HMI clock with backend
**Frequency:** Every 60 seconds

**Format:**
```bash
#<seq> SYS TIME <YYYY-MM-DD> <HH:MM:SS> <DOW> *<crc>
```

**Parameters:**
- `YYYY-MM-DD`: Date (ISO 8601)
- `HH:MM:SS`: Time (24-hour)
- `DOW`: Day of week (0=Mon, 6=Sun)

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
#044 SYS TIME 2025-11-25 12:34:56 1 *C2\r\n
```

**HMI Action:**
- Update internal RTC
- Display time on all screens
- Log time sync event

#### SYS NET - Network Status

**Direction:** BB → HMI
**Purpose:** Network connectivity info
**Frequency:** Every 10 seconds

**Format:**
```bash
#<seq> SYS NET <ip> <status> <clients> *<crc>
```

**Parameters:**
- `ip`: IP address (e.g., 192.168.1.100)
- `status`: UP, DOWN, LAG
- `clients`: Connected API clients 0-255

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
#045 SYS NET 192.168.1.100 UP 3 *A8\r\n
```

---

### Sensor Commands

#### SENSOR - Sensor Data Update

**Direction:** BB → HMI
**Purpose:** Update sensor reading
**Frequency:** Every 5 seconds per sensor

**Format:**
```bash
#<seq> SENSOR <id> <type> <val1> <val2> <status> <age> <name> *<crc>
```

**Parameters:**
- `id`: Sensor ID 0-255
- `type`: TH (temp/humidity), SM (smoke), NC (NONC/door), EN (energy)
- `val1`: Primary value (temp, smoke level, door state) as float
- `val2`: Secondary value (humidity) as float, or 0
- `status`: OK, WARN, ERROR, STALE, OFFLINE
- `age`: Seconds since last update 0-65535
- `name`: Sensor instance name (truncate to fit line, max ~20 chars)

**Response:**
```bash
OK *<crc>
```

**Examples:**
```bash
#046 SENSOR 1 TH 24.5 58.0 OK 2 server_room *7B\r\n
#047 SENSOR 3 SM 15.0 0 OK 1 smoke_det_1 *C4\r\n
#048 SENSOR 5 NC 0 0 OK 3 front_door *9A\r\n
```

**Value Encoding:**
- **TH sensors:** val1=temperature (°C), val2=humidity (%)
- **SM sensors:** val1=smoke level (0-100), val2=0
- **NC sensors:** val1=0 (closed) or 1 (open), val2=0
- **EN sensors:** val1=power (W), val2=energy (kWh)

**HMI Action:**
- Store sensor data in cache
- Update display if screen visible
- Check thresholds (if local rules)
- Show warning icon if status != OK

---

### Relay Commands

#### RELAY - Relay Status Update

**Direction:** BB → HMI
**Purpose:** Update relay state
**Frequency:** On state change

**Format:**
```bash
#<seq> RELAY <id> <state> <mode> <runtime> <rule> *<crc>
```

**Parameters:**
- `id`: Relay ID 1-255
- `state`: ON, OFF, BLINK, ERROR
- `mode`: AUTO, MANUAL, OVERRIDE, DISABLED
- `runtime`: Seconds in current state (uint32_t)
- `rule`: Active rule name (or "none"), max ~20 chars

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
#049 RELAY 1 ON AUTO 1234 temp_ctrl *E7\r\n
#050 RELAY 2 OFF MANUAL 567 none *B2\r\n
```

**HMI Display:**
- Show state with visual indicator (bar)
- Display mode prominently
- Show rule name if AUTO mode
- Convert runtime to human-readable (e.g., "2h 34m")

#### RELAY SET - Manual Relay Control

**Direction:** HMI → BB
**Purpose:** Request relay state change
**Frequency:** On user action
**Authentication:** Required

**Format:**
```bash
#<seq> RELAY SET <id> <state> <token> *<crc>
```

**Parameters:**
- `id`: Relay ID 1-255
- `state`: ON, OFF
- `token`: Authentication token (if authenticated)

**Response:**
```bash
OK RELAY <id> <state> *<crc>
ERR E11 AUTH_REQUIRED *<crc>
ERR E02 INVALID_STATE *<crc>
```

**Example:**
```bash
HMI: #100 RELAY SET 1 OFF abc123token *F4\r\n
BB:  OK RELAY 1 OFF *9A\r\n
```

**Backend Action:**
- Verify authentication token
- Check if manual control allowed
- Change relay state
- Send RELAY status update
- Log action to audit log

---

### Alert Commands

#### ALERT NEW - New Alert Notification

**Direction:** BB → HMI
**Purpose:** Notify of new alert
**Frequency:** Immediate on alert trigger

**Format:**
```bash
#<seq> ALERT NEW <id> <priority> <category> <message> *<crc>
```

**Parameters:**
- `id`: Alert ID 0-65535 (unique)
- `priority`: CRIT, HIGH, MED, LOW
- `category`: SENSOR, SYSTEM, NETWORK, SECURITY
- `message`: Alert description, max ~100 chars

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
#051 ALERT NEW 1 HIGH SENSOR TH4 temp high 26.2C *A3\r\n
```

**HMI Action:**
- Store alert in history
- If CRIT/HIGH: Show alert overlay screen
- Set LED status (alert indicator LED)
- Sound buzzer (pattern based on priority)
- Update alert count on dashboard

#### ALERT CLEAR - Alert Cleared

**Direction:** BB → HMI
**Purpose:** Notify alert condition resolved
**Frequency:** Immediate on resolution

**Format:**
```bash
#<seq> ALERT CLEAR <id> *<crc>
```

**Parameters:**
- `id`: Alert ID to clear

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
#052 ALERT CLEAR 1 *B7\r\n
```

**HMI Action:**
- Mark alert as [CLEAR] in history
- Remove from active count
- Update LED status
- Stop buzzer if no other active alerts
- Auto-remove from display after 30s

#### ALERT ACK - Acknowledge Alert

**Direction:** HMI → BB
**Purpose:** User acknowledges alert
**Frequency:** On user action

**Format:**
```bash
#<seq> ALERT ACK <id> *<crc>
```

**Parameters:**
- `id`: Alert ID to acknowledge

**Response:**
```bash
OK ALERT <id> ACKED *<crc>
```

**Example:**
```bash
HMI: #101 ALERT ACK 1 *C4\r\n
BB:  OK ALERT 1 ACKED *E2\r\n
```

**Backend Action:**
- Mark alert as acknowledged
- Log acknowledgment event
- Update alert status in database

---

### LED Commands

#### LED - LED Control

**Direction:** BB → HMI
**Purpose:** Set LED color and mode
**Frequency:** On state change

**Format:**
```bash
#<seq> LED <idx> <R> <G> <B> <mode> *<crc>
```

**Parameters:**
- `idx`: LED index 1-16
- `R`: Red value 0-255
- `G`: Green value 0-255
- `B`: Blue value 0-255
- `mode`: SOLID, BLINK, BREATHE, OFF

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
#053 LED 1 255 0 0 BLINK *F9\r\n  # LED 1 = red blinking
#054 LED 8 0 255 0 SOLID *A7\r\n  # LED 8 = green solid
```

#### LED STATUS - Status LED Shorthand

**Direction:** BB → HMI
**Purpose:** Set LED based on status
**Frequency:** On state change

**Format:**
```bash
#<seq> LED STATUS <idx> <state> *<crc>
```

**Parameters:**
- `idx`: LED index 1-16
- `state`: OK, WARN, ERROR, INFO, OFF

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
#055 LED STATUS 8 ERROR *C1\r\n
```

**Color Mapping:**
- OK → Green solid
- WARN → Yellow/Orange solid
- ERROR → Red blinking
- INFO → Blue solid
- OFF → All off

---

### Buzzer Commands

#### BUZZ BEEP - Simple Beep

**Direction:** BB → HMI
**Purpose:** Play short beep
**Frequency:** On demand

**Format:**
```bash
#<seq> BUZZ BEEP <duration_ms> <freq_hz> *<crc>
```

**Parameters:**
- `duration_ms`: Duration 0-5000 (ms)
- `freq_hz`: Frequency 1000-5000 (Hz)

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
#056 BUZZ BEEP 200 3000 *D4\r\n
```

#### BUZZ PATTERN - Pattern Playback

**Direction:** BB → HMI
**Purpose:** Play predefined pattern
**Frequency:** On demand

**Format:**
```bash
#<seq> BUZZ PATTERN <pattern_id> *<crc>
```

**Parameters:**
- `pattern_id`: 0-6 (see pattern table)

**Pattern Table:**
| ID | Name | Use Case | Sequence |
|----|------|----------|----------|
| 0 | Off | Normal | Silent |
| 1 | Single | Button feedback | 200ms beep |
| 2 | Double | Confirmation | 100ms, 50ms gap, 100ms |
| 3 | Alert | Critical alerts | Continuous |
| 4 | Warning | Warnings | 500ms on, 1000ms off |
| 5 | Error | Errors | 200ms on, 200ms off |
| 6 | Success | Completed | Rising tone 300ms |

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
#057 BUZZ PATTERN 5 *A2\r\n  # Error pattern
```

#### BUZZ STOP - Stop Buzzer

**Direction:** BB → HMI
**Purpose:** Stop any playing sound
**Frequency:** On demand

**Format:**
```bash
#<seq> BUZZ STOP *<crc>
```

**Response:**
```bash
OK *<crc>
```

---

**Continue to HMI_PROTOCOL_PART2.md for remaining commands...**
