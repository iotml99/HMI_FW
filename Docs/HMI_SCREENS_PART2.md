# HMI Screen Design Specification (Part 2)

**Continued from:** [HMI_SCREENS.md](./HMI_SCREENS.md)

---

### Screen 7: System Info

**Purpose:** Device information and configuration status

```
┌──────────────────────────┐
│ SYSTEM INFO (1/2)        │ ← Pagination
│──────────────────────────│
│ Device: RSU-01           │
│ Location: DC-Rack-A42    │
│                          │
│ Hardware:                │
│  Platform: BeagleBone    │
│  CPU: AM335x 1GHz        │
│  RAM: 512MB              │
│  Storage: 16GB eMMC      │
│                          │
│ Software:                │
│  Backend: v2.1.3         │
│  HMI FW: v1.0.1          │
│  Python: 3.11.2          │
│──────────────────────────│
│ ↕=Scroll  ◄=Back         │
└──────────────────────────┘
```

**Page 2 (scroll down):**
```
┌──────────────────────────┐
│ SYSTEM INFO (2/2)        │
│──────────────────────────│
│ Runtime:                 │
│  Uptime: 5d 12h 34m      │
│  Started: 2025-11-20     │
│            07:00:25      │
│                          │
│ Configuration:           │
│  device.yaml      ✓      │
│  peripherals.yaml ✓      │
│  rsu_config.yaml  ✓      │
│  appconfig.yaml   ✓      │
│                          │
│ Mock mode: OFF           │
│ Debug mode: OFF          │
│──────────────────────────│
│ ↕=Scroll  ◄=Back         │
└──────────────────────────┘
```

---

### Screen 8: Network Status

**Purpose:** Connectivity and communication status

```
┌──────────────────────────┐
│ NETWORK STATUS           │
│──────────────────────────│
│ Ethernet:                │
│  IP: 192.168.1.100       │
│  Mask: 255.255.255.0     │
│  Gateway: 192.168.1.1    │
│  DNS: 8.8.8.8            │
│  Link: 1000Mbps FULL     │
│  Status: ✓ CONNECTED     │
│                          │
│ Backend API:             │
│  Status: ✓ RUNNING       │
│  Port: 5000              │
│  Clients: 3              │
│  Req/min: 142            │
│──────────────────────────│
│ ↕=Scroll OK=Ping  ◄      │
└──────────────────────────┘
```

**Page 2:**
```
┌──────────────────────────┐
│ NETWORK STATUS (2/2)     │
│──────────────────────────│
│ Serial Ports:            │
│  Modbus1: ✓ 9600 baud    │
│  Modbus2: ✓ 19200 baud   │
│  Serial1: ✓ 115200 baud  │
│  HMI: ✓ 115200 baud      │
│                          │
│ Time Sync (NTP):         │
│  Status: ✓ SYNCED        │
│  Server: pool.ntp.org    │
│  Last sync: 12:30:15     │
│  Offset: +12ms           │
│                          │
│ Latency: 2ms (BB-HMI)    │
│──────────────────────────│
│ ↕=Scroll OK=Test  ◄      │
└──────────────────────────┘
```

---

### Screen 9: System Logs

**Purpose:** View recent system logs

```
┌──────────────────────────┐
│ SYSTEM LOGS              │
│ Filter: [ALL ▼]          │ ← Can filter by level
│──────────────────────────│
│ 12:45:23 [INFO] API      │
│  Sensor TH1 updated      │
│                          │
│ 12:45:20 [WARN] SENSOR   │
│  TH4 high temp: 26.2°C   │
│                          │
│ 12:45:15 [INFO] RULE     │
│  Rule 'temp_ctrl' exec   │
│                          │
│ 12:45:10 [ERROR] SENSOR  │
│  Modbus slave 3 timeout  │
│──────────────────────────│
│ ↕=Scroll ►=Filter  ◄     │
└──────────────────────────┘
```

**Filter Options:**
```
┌──────────────────────────┐
│ LOG FILTER               │
│──────────────────────────│
│ ► ALL                    │
│   ERROR only             │
│   WARN + ERROR           │
│   INFO + WARN + ERROR    │
│   DEBUG (all)            │
│                          │
│                          │
│──────────────────────────│
│ ↕=Select OK=Apply  ◄     │
└──────────────────────────┘
```

---

### Screen 10: PIN Entry

**Purpose:** Authentication for protected operations

```
┌──────────────────────────┐
│ 🔒 ENTER PIN CODE        │
│──────────────────────────│
│                          │
│      ****                │ ← Masked input
│                          │
│  [1] [2] [3]             │
│  [4] [5] [6]             │ ← Virtual keypad
│  [7] [8] [9]             │   Navigate with buttons
│  [←] [0] [✓]             │
│                          │
│  Action: Restart API     │ ← What operation requires auth
│                          │
│──────────────────────────│
│ ↕←→=Nav OK=Sel  ◄=Cancel │
└──────────────────────────┘
```

**Behavior:**
- UP/DOWN/LEFT/RIGHT: Navigate virtual keypad
- OK: Select current number/action
- [←]: Backspace
- [✓]: Submit PIN
- After 3 failed attempts: 60-second lockout

**Success:**
```
┌──────────────────────────┐
│ ✓ AUTHENTICATED          │
│──────────────────────────│
│                          │
│   Access Granted         │
│                          │
│   Session: 5 minutes     │
│                          │
│                          │
│──────────────────────────│
│ [Auto-continuing...]     │
└──────────────────────────┘
```

**Failure:**
```
┌──────────────────────────┐
│ ✗ ACCESS DENIED          │
│──────────────────────────│
│                          │
│   Invalid PIN            │
│                          │
│   Attempts left: 2       │
│                          │
│                          │
│──────────────────────────│
│ OK=Retry  ◄=Cancel       │
└──────────────────────────┘
```

---

### Screen 11: Alert Overlay (Interrupt)

**Purpose:** Critical alerts interrupt current screen

```
┌══════════════════════════┐ ← Blinking border
│ ⚠⚠⚠ CRITICAL ⚠⚠⚠         │   (inverted colors)
│══════════════════════════│
│                          │
│  TEMPERATURE HIGH        │ ← Large text
│                          │
│  TH4: 28.5°C             │ ← Problem detail
│  Threshold: 26°C         │
│                          │
│  Action: RLY1 ON         │ ← Automated response
│                          │
│══════════════════════════│
│ Auto-clear in 3s...      │ ← Countdown
│ OK=Acknowledge           │
└══════════════════════════┘
```

**Behavior:**
- Appears immediately for CRITICAL/HIGH priority alerts
- Screen inverts (flash effect)
- Buzzer sounds (pattern based on priority)
- LEDs blink
- Shows for 3 seconds, then returns to previous screen
- OK button: Acknowledge immediately and dismiss
- Logs whether manually or auto-acknowledged

---

### Screen 12: Firmware Update

**Purpose:** Display firmware update progress

```
┌──────────────────────────┐
│ FIRMWARE UPDATE          │
│──────────────────────────│
│ Current: v1.0.0          │
│ New:     v1.0.1          │
│ Size:    96 KB           │
│                          │
│ ⚠ Update will take       │
│   approximately 2 min    │
│                          │
│ Device will reboot       │
│                          │
│ Continue?                │
│──────────────────────────│
│ OK=Yes  ◄=Cancel         │
└──────────────────────────┘
```

**During Update (Bootloader Mode):**
```
┌──────────────────────────┐
│ FIRMWARE UPDATE          │
│──────────────────────────│
│                          │
│ Writing firmware...      │
│                          │
│ ▓▓▓▓▓▓▓▓░░░░░░░░  45%   │ ← Progress bar
│                          │
│ Chunk: 110/256           │
│ Speed: 8.2 KB/s          │
│ ETA: 58 seconds          │
│                          │
│ ⚠ DO NOT POWER OFF!      │
│                          │
│──────────────────────────│
│ [Updating...]            │
└──────────────────────────┘
```

**Update Complete:**
```
┌──────────────────────────┐
│ ✓ UPDATE COMPLETE        │
│──────────────────────────│
│                          │
│ Firmware: v1.0.1         │
│                          │
│ ✓ Written: 96 KB         │
│ ✓ Verified: CRC OK       │
│                          │
│ Rebooting in 3s...       │
│                          │
│                          │
│──────────────────────────│
│ [Please wait...]         │
└──────────────────────────┘
```

---

## Icon Legend

### Status Icons

| Icon | Meaning | Usage |
|------|---------|-------|
| ✓ | OK / Success | Normal operation |
| ⚠ | Warning | Non-critical issue |
| ✗ | Error / Failed | Critical failure |
| ? | Unknown / Stale | Data not updated |
| ℹ | Information | Low priority info |
| 🔒 | Locked / Auth Required | Protected operation |
| ► | Selected / Active | Current selection |
| ◄ | Back / Return | Navigation hint |
| ► | Forward / Next | Navigation hint |
| ↕ | Up/Down | Scroll/Navigate |
| ═ | Header bar | Section separator |
| ─ | Separator line | Visual grouping |

### Sensor Icons

| Icon | Sensor Type | Example |
|------|-------------|---------|
| 🌡 | Temperature | TH sensors |
| 💧 | Humidity | TH sensors |
| 🚪 | Door | NONC sensors |
| 🔥 | Smoke | Smoke detectors |
| ⚡ | Power/Relay | Output status |
| 📶 | Network | Connectivity |
| 🔔 | Alert | Notification |

### Drawing Primitives

| Character | Usage |
|-----------|-------|
| █ | Solid block (ON state) |
| ░ | Light block (OFF state) |
| ▓ | Medium block (progress) |
| ┌ ┐ └ ┘ | Box corners |
| │ | Vertical line |
| ─ | Horizontal line |
| ├ ┤ ┬ ┴ ┼ | Line junctions |

---

## Design Guidelines

### Typography

**Font Size Hierarchy:**
1. **Critical Info (16×32):** Alert messages, large numbers
2. **Headers (8×16):** Screen titles, section headers
3. **Body Text (6×8):** Most content, details
4. **Small Text (5×7):** Navigation hints, timestamps

**Text Formatting:**
- **Bold:** Important status, warnings
- **UPPERCASE:** Headers, commands
- **lowercase:** Regular text, details
- **Monospace:** Numbers, technical data

### Layout Principles

1. **Header (Lines 1-2):**
   - Screen title
   - Time/date (if space permits)
   - Status indicators

2. **Content Area (Lines 3-10):**
   - Primary information
   - Scrollable if needed
   - Group related items

3. **Footer (Lines 11-12):**
   - Navigation hints
   - Action prompts
   - Status messages

### Visual Hierarchy

**Importance Order:**
1. Alerts and warnings (largest, center)
2. Current values (medium, prominent)
3. Labels (smaller, left-aligned)
4. Navigation hints (smallest, bottom)

### Spacing

- **Line spacing:** 1-2 pixels between lines
- **Section spacing:** 4-6 pixels between sections
- **Margin:** 2-4 pixels from screen edges
- **Padding:** 1-2 pixels around text boxes

### Readability

**Distance Viewing (1-3 meters):**
- Minimum font: 8×16 for critical info
- High contrast: white on black
- Bold text for emphasis
- Icons supplement text

**Close Viewing (<0.5 meters):**
- Can use 6×8 font for details
- More information density
- Fine-grained controls

### Animation

**Allowed:**
- Blinking alerts (1 Hz)
- Progress bars (smooth fill)
- Cursor movement (instant)
- Screen transitions (fade or instant)

**Prohibited:**
- Excessive scrolling text
- Distracting animations
- Rapid flashing (>2 Hz)
- Complex transitions

### Accessibility

1. **High Contrast:** Pure white on pure black
2. **Large Text:** 8×16 minimum for critical info
3. **Redundancy:** Icons + text, audio + visual
4. **Timeout Warnings:** Show countdown before auto-actions
5. **Clear Navigation:** Always show how to exit/return

### Error Handling

**Display Errors:**
- Show clear error message
- Suggest remediation if possible
- Provide exit path
- Log error details

**Connection Loss:**
```
┌──────────────────────────┐
│ ⚠ CONNECTION LOST        │
│──────────────────────────│
│                          │
│ Backend disconnected     │
│                          │
│ Last contact: 15s ago    │
│                          │
│ Showing cached data      │
│                          │
│ Reconnecting...          │
│                          │
│──────────────────────────│
│ [Automatic retry]        │
└──────────────────────────┘
```

**Stale Data:**
- Gray out text
- Show "?" icon
- Display age: "Last update: 5m ago"
- Don't delete data, just mark stale

---

## Screen State Management

### State Variables

```c
typedef enum {
    SCREEN_MAIN_DASHBOARD = 0,
    SCREEN_MENU = 1,
    SCREEN_SENSORS = 2,
    SCREEN_ALERTS = 3,
    SCREEN_RELAYS = 4,
    SCREEN_SYSTEM_CONTROL = 5,
    SCREEN_DIAGNOSTICS = 6,
    SCREEN_SYSTEM_INFO = 7,
    SCREEN_NETWORK = 8,
    SCREEN_LOGS = 9,
    SCREEN_PIN_ENTRY = 10,
    SCREEN_ALERT_OVERLAY = 11,
    SCREEN_FIRMWARE_UPDATE = 12
} ScreenType;

typedef struct {
    ScreenType current_screen;
    ScreenType previous_screen;  // For returning after overlay
    uint8_t sub_screen;          // For multi-page screens
    uint8_t selected_item;       // Menu selection index
    uint32_t last_input_time;    // For timeout
    bool authenticated;
    uint32_t auth_expiry;
    bool alert_overlay_active;
} ScreenState;
```

### Screen Transition Matrix

| From Screen | Button | To Screen | Notes |
|-------------|--------|-----------|-------|
| Any | LEFT | Main Dashboard | Universal escape |
| Main Dashboard | RIGHT | Menu | Enter menu |
| Menu | OK | Selected screen | Based on cursor |
| Any detail | Timeout | Main Dashboard | 30s inactivity |
| Alert overlay | Auto | Previous screen | After 3s |

---

## Responsive Behavior

### Screen Refresh Strategy

```
Main Dashboard:
  - Auto-refresh every 5s
  - Auto-scroll sub-screens every 5s
  - Immediate refresh on alert

Detail Screens:
  - Refresh when data arrives
  - No auto-scroll unless explicitly stated
  - Manual scroll with buttons

Overlay Screens:
  - Immediate display
  - Block background updates
  - Auto-dismiss or manual dismiss
```

### Priority System

**Screen Priority (highest to lowest):**
1. Firmware Update (blocks all)
2. Critical Alert Overlay (temporary)
3. PIN Entry (modal)
4. User Navigation (normal screens)
5. Idle/Auto-scroll (dashboard)

---

**Next Document:** [HMI_PROTOCOL.md](./HMI_PROTOCOL.md) - Communication protocol specification
