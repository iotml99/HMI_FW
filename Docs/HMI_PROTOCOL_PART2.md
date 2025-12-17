# HMI Communication Protocol (Part 2)

**Continued from:** [HMI_PROTOCOL.md](./HMI_PROTOCOL.md)

---

### Screen Commands

#### SCR GOTO - Change Screen

**Direction:** BB → HMI
**Purpose:** Navigate to specific screen
**Frequency:** On demand

**Format:**
```bash
#<seq> SCR GOTO <screen_id> *<crc>
```

**Parameters:**
- `screen_id`: 0-12 (see screen table)

**Screen IDs:**
| ID | Screen Name |
|----|-------------|
| 0 | Main Dashboard |
| 1 | Menu |
| 2 | Sensors Detail |
| 3 | Alerts History |
| 4 | Relays Control |
| 5 | System Control |
| 6 | Diagnostics |
| 7 | System Info |
| 8 | Network Status |
| 9 | System Logs |
| 10 | PIN Entry |
| 11 | Alert Overlay |
| 12 | Firmware Update |

**Response:**
```bash
OK SCR <screen_id> *<crc>
```

**Example:**
```bash
BB: #058 SCR GOTO 3 *B4\r\n
HMI: OK SCR 3 *A1\r\n
```

#### SCR OVERLAY - Temporary Overlay

**Direction:** BB → HMI
**Purpose:** Show temporary message overlay
**Frequency:** On demand

**Format:**
```bash
#<seq> SCR OVERLAY <duration> <message> *<crc>
```

**Parameters:**
- `duration`: Display duration in seconds (1-10)
- `message`: Message text, max ~60 chars

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
#059 SCR OVERLAY 3 Alert Acknowledged *E7\r\n
```

**HMI Action:**
- Show overlay on current screen
- Auto-dismiss after duration
- Return to previous screen

#### SCR BRIGHT - Set Brightness

**Direction:** BB → HMI
**Purpose:** Adjust display brightness
**Frequency:** On change

**Format:**
```bash
#<seq> SCR BRIGHT <level> *<crc>
```

**Parameters:**
- `level`: 0-100 (%)

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
#060 SCR BRIGHT 80 *C9\r\n
```

#### SCR ACTIVE - Screen Changed Notification

**Direction:** HMI → BB
**Purpose:** Notify backend of screen change
**Frequency:** On user navigation

**Format:**
```bash
#<seq> SCR ACTIVE <screen_id> *<crc>
```

**Parameters:**
- `screen_id`: Current screen ID

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
HMI: #102 SCR ACTIVE 3 *D5\r\n
BB: OK *5A\r\n
```

---

### Button Commands

#### BTN - Button Event

**Direction:** HMI → BB
**Purpose:** Report button press/release
**Frequency:** On button action

**Format:**
```bash
#<seq> BTN <button> <event> <screen> *<crc>
```

**Parameters:**
- `button`: UP, DOWN, LEFT, RIGHT, OK
- `event`: PRESS, RELEASE, LONG
- `screen`: Current screen ID (for context)

**Response:**
```bash
OK *<crc>
```

**Examples:**
```bash
HMI: #103 BTN OK PRESS 5 *F2\r\n
HMI: #104 BTN LEFT PRESS 3 *A9\r\n
HMI: #105 BTN OK LONG 0 *B7\r\n
```

**Backend Action:**
- Log button event
- Execute context-specific action
- May trigger screen change or command

---

### Authentication Commands

#### AUTH PIN - PIN Authentication

**Direction:** HMI → BB
**Purpose:** Authenticate user with PIN
**Frequency:** On login attempt

**Format:**
```bash
#<seq> AUTH PIN <pin> *<crc>
```

**Parameters:**
- `pin`: 4-digit PIN (transmitted in plain text over UART)

**Response (Success):**
```bash
OK AUTH SESSION <token> EXPIRE <seconds> *<crc>
```

**Response (Failure):**
```bash
ERR E11 INVALID_PIN ATTEMPTS_LEFT <count> *<crc>
ERR E11 LOCKED DURATION <seconds> *<crc>
```

**Examples:**
```bash
HMI: #106 AUTH PIN 1234 *C4\r\n
BB:  OK AUTH SESSION a1b2c3d4 EXPIRE 300 *E8\r\n

HMI: #107 AUTH PIN 0000 *A1\r\n
BB:  ERR E11 INVALID_PIN ATTEMPTS_LEFT 2 *F3\r\n
```

**Session Management:**
- Token valid for 300 seconds (5 minutes)
- Max 3 attempts before lockout
- Lockout duration: 60 seconds
- Session auto-expires and requires re-auth

**Event (Session Expired):**
```bash
BB: EVT 0x90 SESSION_EXPIRED *7C\r\n
```

---

### Command Execution

#### CMD - Execute System Command

**Direction:** HMI → BB
**Purpose:** Execute system maintenance command
**Frequency:** On user request
**Authentication:** Required for most commands

**Format:**
```bash
#<seq> CMD <cmd_type> <token> [params...] *<crc>
```

**Command Types:**
- `RESTART_API`: Restart Flask backend
- `RESTART_HUB`: Restart sensor hub
- `RESTART_RULES`: Restart rule engine
- `RELOAD_CONFIG`: Reload configuration files
- `TEST_SENSORS`: Test all sensors
- `TEST_RELAYS`: Test all relays
- `TEST_NETWORK`: Test network connectivity
- `TEST_GPIO`: Test GPIO pins
- `SCAN_MODBUS`: Scan Modbus bus
- `CLEAR_ALERTS`: Clear alert history
- `GET_LOGS <count>`: Get system logs

**Response (Immediate):**
```bash
OK CMD <cmd_type> STARTED EST_TIME <seconds> *<crc>
ERR E07 BUSY <current_cmd> *<crc>
ERR E11 AUTH_REQUIRED *<crc>
```

**Example:**
```bash
HMI: #108 CMD RESTART_API a1b2c3d4 *B9\r\n
BB:  OK CMD RESTART_API STARTED EST_TIME 15 *D2\r\n
```

#### CMD PROGRESS - Command Progress Update

**Direction:** BB → HMI
**Purpose:** Update command execution progress
**Frequency:** Async during execution

**Format:**
```bash
#<seq> CMD PROGRESS <cmd_type> <percent> <step> *<crc>
```

**Parameters:**
- `cmd_type`: Command being executed
- `percent`: Progress 0-100 (%)
- `step`: Current step description, max ~30 chars

**Response:** None (push notification)

**Example:**
```bash
BB: #061 CMD PROGRESS RESTART_API 20 Stopping_services *A7\r\n
BB: #062 CMD PROGRESS RESTART_API 50 Reloading_config *C3\r\n
BB: #063 CMD PROGRESS RESTART_API 80 Starting_services *E9\r\n
```

#### CMD RESULT - Command Result

**Direction:** BB → HMI
**Purpose:** Report command completion
**Frequency:** Async after completion

**Format:**
```bash
#<seq> CMD RESULT <cmd_type> <result> <duration> [data] *<crc>
```

**Parameters:**
- `cmd_type`: Command executed
- `result`: SUCCESS, FAILED, TIMEOUT, PARTIAL
- `duration`: Actual execution time (seconds)
- `data`: Optional result data

**Response:** None (push notification)

**Examples:**
```bash
BB: #064 CMD RESULT RESTART_API SUCCESS 14 *F1\r\n
BB: #065 CMD RESULT TEST_SENSORS PARTIAL 8 15_of_16_OK *A4\r\n
BB: #066 CMD RESULT SCAN_MODBUS FAILED 5 BUS_ERROR *B7\r\n
```

---

### Diagnostic Commands

#### DIAG RESULT - Diagnostic Test Result

**Direction:** BB → HMI
**Purpose:** Report diagnostic test result
**Frequency:** Async after test completion

**Format:**
```bash
#<seq> DIAG RESULT <test_type> <result> <details> *<crc>
```

**Parameters:**
- `test_type`: Test executed (HW, SENSOR, MODBUS, NETWORK, etc.)
- `result`: PASS, FAIL, PARTIAL
- `details`: Summary, max ~50 chars

**Response:** None (push notification)

**Multi-Line Results:**

For detailed test results (e.g., Modbus scan), use multi-line format:

```bash
BB: #067 DIAG RESULT MODBUS1 PARTIAL 4 *D2\r\n
BB: 01 TH_Sensor OK RTT_12ms *E1\r\n
BB: 02 TH_Sensor OK RTT_15ms *F3\r\n
BB: 03 Energy_Meter FAIL TIMEOUT *A7\r\n
BB: 04 TH_Sensor OK RTT_11ms *B9\r\n
BB: END *5A\r\n
```

**HMI Action:**
- Parse multi-line results
- Display in diagnostic screen
- Store for review

---

### HMI Status

#### HMI STATUS - HMI Health Report

**Direction:** HMI → BB
**Purpose:** Report HMI board health
**Frequency:** Every 10 seconds

**Format:**
```bash
#<seq> HMI STATUS <uptime> <screen> <bright> <temp> <voltage> *<crc>
```

**Parameters:**
- `uptime`: HMI uptime (seconds)
- `screen`: Current screen ID
- `bright`: Display brightness 0-100 (%)
- `temp`: STM32 internal temperature (°C)
- `voltage`: Supply voltage (mV)

**Response:**
```bash
OK *<crc>
```

**Example:**
```bash
HMI: #109 HMI STATUS 12345 0 80 45 3300 *D8\r\n
BB:  OK *5A\r\n
```

**Backend Action:**
- Monitor HMI health
- Log voltage issues
- Alert if temperature high
- Track HMI uptime

---

### Bootloader Commands

#### BL CHECK - Check Firmware Availability

**Direction:** HMI → BB
**Purpose:** Query if firmware update available
**Frequency:** On demand

**Format:**
```bash
#<seq> BL CHECK *<crc>
```

**Response:**
```bash
OK BL FILE <filename> SIZE <bytes> CRC <hex> *<crc>
ERR E12 NO_FILE *<crc>
```

**Example:**
```bash
HMI: #110 BL CHECK *A3\r\n
BB:  OK BL FILE hmi_v1.0.1.bin SIZE 98304 CRC 0xA3F2B184 *E9\r\n
```

#### BL ENTER - Enter Bootloader Mode

**Direction:** HMI → BB
**Purpose:** Request bootloader entry
**Frequency:** On firmware update initiation
**Authentication:** Admin PIN required

**Format:**
```bash
#<seq> BL ENTER <admin_pin> *<crc>
```

**Response:**
```bash
OK BL ENTERING *<crc>
ERR E11 AUTH_FAILED *<crc>
```

**Example:**
```bash
HMI: #111 BL ENTER 9999 *F7\r\n
BB:  OK BL ENTERING *B2\r\n

[HMI jumps to bootloader, resets UART]

HMI: #000 BL READY VERSION 1.0 FLASH 0x08004000 SIZE 229376 *C4\r\n
```

**HMI Action:**
- Save application state
- Jump to bootloader address
- Reset UART (sequence resets to 000)
- Send READY message

#### BL ERASE - Erase Flash

**Direction:** BB → HMI
**Purpose:** Erase application flash region
**Frequency:** During firmware update

**Format:**
```bash
#<seq> BL ERASE <start_addr> <length> *<crc>
```

**Parameters:**
- `start_addr`: Flash start address (hex, e.g., 0x08004000)
- `length`: Bytes to erase

**Response (Immediate):**
```bash
OK BL ERASING *<crc>
```

**Progress Updates:**
```bash
HMI: BL PROGRESS <percent> Erasing *<crc>
```

**Completion:**
```bash
HMI: OK BL ERASED *<crc>
```

**Example:**
```bash
BB:  #001 BL ERASE 0x08004000 229376 *A9\r\n
HMI: OK BL ERASING *B3\r\n
[2-5 seconds erase time]
HMI: BL PROGRESS 50 Erasing *C7\r\n
HMI: OK BL ERASED *D1\r\n
```

#### BL WRITE - Write Flash Chunk

**Direction:** BB → HMI
**Purpose:** Write 256-byte firmware chunk
**Frequency:** Many times during update

**Format:**
```bash
#<seq> BL WRITE <addr> <hex_data> *<crc>
```

**Parameters:**
- `addr`: Flash address (hex)
- `hex_data`: 512 hex characters (256 bytes)

**Response:**
```bash
OK BL WRITTEN 256 *<crc>
ERR E09 FLASH_ERROR *<crc>
```

**Progress Updates:**
```bash
HMI: BL PROGRESS <percent> Writing *<crc>
```

**Example:**
```bash
BB: #002 BL WRITE 0x08004000 0123456789ABCDEF...×512 *F4\r\n
HMI: OK BL WRITTEN 256 *A7\r\n
HMI: BL PROGRESS 1 Writing *B9\r\n
```

**Notes:**
- Firmware written in 256-byte chunks
- 896 chunks for 229 KB firmware
- ~60-90 seconds total write time
- Progress updates every 10 chunks

#### BL VERIFY - Verify Firmware

**Direction:** BB → HMI
**Purpose:** Verify written firmware CRC
**Frequency:** After write complete

**Format:**
```bash
#<seq> BL VERIFY *<crc>
```

**Response (Immediate):**
```bash
OK BL VERIFYING *<crc>
```

**Result:**
```bash
OK BL CRC <calculated_crc> MATCH *<crc>
ERR E13 BL CRC <calculated_crc> MISMATCH *<crc>
```

**Example:**
```bash
BB:  #003 BL VERIFY *A1\r\n
HMI: OK BL VERIFYING *B2\r\n
[CRC calculation ~2-3 seconds]
HMI: OK BL CRC 0xA3F2B184 MATCH *D4\r\n
```

#### BL LAUNCH - Launch Application

**Direction:** BB → HMI
**Purpose:** Exit bootloader, start application
**Frequency:** After successful verification

**Format:**
```bash
#<seq> BL LAUNCH *<crc>
```

**Response:**
```bash
OK BL LAUNCHING *<crc>
```

**Example:**
```bash
BB:  #004 BL LAUNCH *C2\r\n
HMI: OK BL LAUNCHING *A8\r\n

[HMI resets, application starts]

HMI: #000 HMI HELLO VERSION 1.0.1 *B7\r\n
BB:  OK HMI CONNECTED BB 2.1.3 *E3\r\n
```

#### BL ABORT - Abort Update

**Direction:** BB → HMI
**Purpose:** Cancel update, try to boot old app
**Frequency:** On user cancel or error

**Format:**
```bash
#<seq> BL ABORT *<crc>
```

**Response:**
```bash
OK BL ABORTED *<crc>
```

**HMI Action:**
- Check if old application still valid (CRC)
- If valid: launch old app
- If invalid: stay in bootloader, show error

---

## Error Handling

### Error Code Reference

| Code | Name | Description | Recovery |
|------|------|-------------|----------|
| E00 | UNKNOWN_CMD | Command not recognized | Check spelling |
| E01 | INVALID_SYNTAX | Malformed command | Check format |
| E02 | RANGE | Parameter out of range | Check limits |
| E03 | INVALID_TYPE | Wrong parameter type | Check data type |
| E04 | NOT_SUPPORTED | Feature not available | Use alternative |
| E05 | OPERATION_FAILED | Operation error | Retry, check logs |
| E06 | TIMEOUT | Operation timeout | Retry, check connection |
| E07 | BUSY | System busy | Wait and retry |
| E08 | NOT_INITIALIZED | System not ready | Wait for init |
| E09 | HARDWARE_ERROR | Hardware fault | Check hardware |
| E10 | CHECKSUM | CRC mismatch | Retransmit |
| E11 | AUTH | Authentication failed | Check PIN/token |
| E12 | UNAVAILABLE | Resource unavailable | Check availability |
| E13 | VERIFY | Verification failed | Re-verify, retry |

### Timeout Values

| Operation | Timeout | Action on Timeout |
|-----------|---------|-------------------|
| Command response | 5 seconds | Retry (max 3 attempts) |
| Single message ACK | 1 second | Retry transmission |
| Heartbeat | 10 seconds | Connection lost |
| Command execution | 30 seconds | Report timeout error |
| Firmware erase | 10 seconds | Abort update |
| Firmware write chunk | 2 seconds | Retry chunk |
| Firmware verify | 5 seconds | Re-verify |

### Retry Strategy

**Exponential Backoff:**
```
Attempt 1: Immediate
Attempt 2: Wait 1 second
Attempt 3: Wait 2 seconds
Failure: Report error, log event
```

### Connection Loss Handling

**Heartbeat Mechanism:**
1. BB sends `SYS PING` every 5 seconds
2. HMI responds with `PONG`
3. If 2 consecutive PINGs missed (10s total) → connection lost

**HMI Behavior on Connection Loss:**
- Display "CONNECTION LOST" overlay
- Show last received data (marked as stale)
- Continue local functions (buttons, display)
- Attempt reconnect: wait for next PING
- LED indicator shows disconnected state
- Buzzer optional alert

**Backend Behavior on Connection Loss:**
- Log warning event
- Continue sensor polling
- Queue messages for HMI
- On reconnect: send latest state snapshot

### CRC Mismatch Handling

**On CRC Failure:**
1. Send `ERR E10 CHECKSUM`
2. Sender retries message (max 3 attempts)
3. After 3 failures: log error, skip message
4. Don't block communication for single failure

---

## Communication Examples

### Startup Sequence

```
[Power on]

HMI: #000 HMI HELLO VERSION 1.0.1 *B7\r\n
BB:  OK HMI CONNECTED BB 2.1.3 *E3\r\n

BB:  #000 SYS TIME 2025-11-25 12:34:56 1 *C2\r\n
HMI: OK *5A\r\n

BB:  #001 SYS STATUS 45 58 52 453600 0 *F4\r\n
HMI: OK *5A\r\n

BB:  #002 SENSOR 1 TH 24.5 58.0 OK 2 server_room *7B\r\n
HMI: OK *5A\r\n
[... more sensors ...]

BB:  #018 RELAY 1 ON AUTO 1234 temp_ctrl *E7\r\n
HMI: OK *5A\r\n

[Normal operation begins]
```

### Periodic Updates

```
[Every 5 seconds]

BB:  #042 SYS PING 1732550400 *A3\r\n
HMI: #100 PONG 1732550400 *B7\r\n

BB:  #043 SYS STATUS 45 58 52 453605 0 *F9\r\n
HMI: OK *5A\r\n

BB:  #044 SENSOR 1 TH 24.6 58.1 OK 1 server_room *8C\r\n
HMI: OK *5A\r\n
[... other sensors ...]
```

### Alert Flow

```
[Sensor threshold exceeded]

BB:  #050 ALERT NEW 1 HIGH SENSOR TH4 temp high 26.2C *A3\r\n
HMI: OK *5A\r\n

BB:  #051 LED STATUS 8 ERROR *C1\r\n
HMI: OK *5A\r\n

BB:  #052 BUZZ PATTERN 5 *A2\r\n
HMI: OK *5A\r\n

[HMI displays alert overlay for 3 seconds]
[User presses OK to acknowledge]

HMI: #101 BTN OK PRESS 11 *D8\r\n
BB:  OK *5A\r\n

HMI: #102 ALERT ACK 1 *C4\r\n
BB:  OK ALERT 1 ACKED *E2\r\n

BB:  #053 BUZZ STOP *B8\r\n
HMI: OK *5A\r\n

[Condition resolves]

BB:  #054 ALERT CLEAR 1 *B7\r\n
HMI: OK *5A\r\n

BB:  #055 LED STATUS 8 OK *D3\r\n
HMI: OK *5A\r\n
```

### Command Execution Flow

```
[User navigates to System Control, selects Restart API]

HMI: #103 SCR ACTIVE 5 *A9\r\n
BB:  OK *5A\r\n

[User confirms action, but not authenticated]

HMI: #104 CMD RESTART_API *B2\r\n
BB:  ERR E11 AUTH_REQUIRED *A7\r\n

[HMI shows PIN entry screen]

HMI: #105 AUTH PIN 1234 *C4\r\n
BB:  OK AUTH SESSION a1b2c3d4 EXPIRE 300 *E8\r\n

[User retries command with token]

HMI: #106 CMD RESTART_API a1b2c3d4 *B9\r\n
BB:  OK CMD RESTART_API STARTED EST_TIME 15 *D2\r\n

[Backend sends progress updates]

BB:  #056 CMD PROGRESS RESTART_API 20 Stopping *A7\r\n
BB:  #057 CMD PROGRESS RESTART_API 50 Reloading *C3\r\n
BB:  #058 CMD PROGRESS RESTART_API 80 Starting *E9\r\n

[Command completes]

BB:  #059 CMD RESULT RESTART_API SUCCESS 14 *F1\r\n

[HMI shows success screen]
```

---

**Next Document:** [HMI_FIRMWARE.md](./HMI_FIRMWARE.md) - STM32 firmware architecture
