# HMI Screen Design Specification

**Document Version:** 1.0
**Date:** 2025-11-25
**Related:** [HMI_OVERVIEW.md](./HMI_OVERVIEW.md)

---

## Table of Contents

1. [Display Specifications](#display-specifications)
2. [Navigation Model](#navigation-model)
3. [Screen Catalog](#screen-catalog)
4. [Icon Legend](#icon-legend)
5. [Design Guidelines](#design-guidelines)

---

## Display Specifications

### Physical Constraints

| Property | Value | Impact |
|----------|-------|--------|
| Resolution | 128×96 pixels | Limited screen real estate |
| Aspect Ratio | 4:3 (landscape) | Wider than tall |
| Monochrome | White on black | High contrast only |
| Physical Size | 0.96" diagonal | Small text requires careful sizing |
| Viewing Distance | 1-3 meters typical | Large, clear text needed |

### Text Capacity

| Font Size | Characters × Lines | Best For |
|-----------|-------------------|----------|
| 6×8 pixels | ~21 chars × 12 lines | Dense information, detail screens |
| 8×16 pixels | ~16 chars × 6 lines | Primary information, status |
| 12×16 pixels | ~10 chars × 6 lines | Headers, critical alerts |
| 16×32 pixels | ~8 chars × 3 lines | Large numbers, warnings |

**Recommended Strategy:**
- Mix font sizes: Large headers (8×16), body text (6×8)
- Reserve top 2 lines for header/title
- Reserve bottom 1-2 lines for navigation hints
- Use middle 8-9 lines for content

---

## Navigation Model

### Navigation Tree

```
                    ┌─────────────────┐
                    │ Main Dashboard  │ ← Default screen
                    │   (Screen 0)    │   Auto-scrolls 3 views
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │ LEFT (no-op)      │ RIGHT             │
         │                   ▼                   │
         │          ┌──────────────┐             │
         │          │     Menu     │             │
         │          │  (Screen 1)  │             │
         │          └──────┬───────┘             │
         │                 │                     │
         │        UP/DOWN to select              │
         │        OK to enter                    │
         │                 │                     │
         ├─────────────────┼─────────────────────┤
         │                 │                     │
    ┌────▼─────┐  ┌────────▼────────┐  ┌────────▼────────┐
    │ Sensors  │  │     Alerts      │  │     Relays      │
    │ Screen 2 │  │    Screen 3     │  │    Screen 4     │
    └──────────┘  └─────────────────┘  └─────────────────┘
         │                 │                     │
    ┌────▼─────┐  ┌────────▼────────┐  ┌────────▼────────┐
    │  System  │  │  Diagnostics    │  │   System Info   │
    │ Control  │  │    Screen 6     │  │    Screen 7     │
    │ Screen 5 │  └─────────────────┘  └─────────────────┘
    └──────────┘           │                     │
         │            ┌────▼────┐         ┌──────▼──────┐
         │            │ Network │         │   Logs      │
         │            │ Status  │         │  Viewer     │
         │            │Screen 8 │         │  Screen 9   │
         │            └─────────┘         └─────────────┘
         │
    ┌────▼─────┐
    │   PIN    │
    │  Entry   │
    │ Screen 10│
    └──────────┘
```

### Button Functions

| Button | Global Function | Context-Specific |
|--------|----------------|------------------|
| **LEFT** | Always return to Main Dashboard | Cancel, Back |
| **RIGHT** | Enter Menu (from Dashboard) | Next screen, Advance |
| **UP** | Scroll up, Previous item | Increment value |
| **DOWN** | Scroll down, Next item | Decrement value |
| **OK** | Select, Confirm | Acknowledge alert |

### Navigation Rules

1. **LEFT is Universal Escape**: Always returns to Main Dashboard (Screen 0)
2. **Timeout Return**: Inactive screens return to Dashboard after 30 seconds
3. **Alert Priority**: Critical alerts interrupt current screen temporarily
4. **No Dead Ends**: Every screen has clear exit path
5. **Breadcrumbs**: Show screen name and navigation hints

---

## Screen Catalog

### Screen 0: Main Dashboard (Home)

**Purpose:** Quick-glance system overview, always accessible

**Behavior:**
- Default screen on startup
- Auto-scrolls between 3 sub-screens every 5 seconds
- Can manually navigate with UP/DOWN
- Critical alerts interrupt auto-scroll for 3 seconds

#### Sub-Screen 0A: Status Overview

```
┌──────────────────────────┐
│ RSU-01      2025-11-25   │ ← Instance name + date
│             12:34:56 MON │ ← Time + day (8×16 font)
│──────────────────────────│
│🌡24.5°C 💧58% 🚪CLOSED   │ ← Avg sensor values
│🔥OK     ⚡3/3  📶ONLINE   │ ← Status indicators
│──────────────────────────│
│ SYSTEM: HEALTHY ✓        │ ← Overall health (bold)
│ Uptime: 5d 12h 34m       │
│──────────────────────────│
│🔔 ALERTS: 2 ACTIVE       │ ← Alert summary
│  ⚠ TH4: Temp high        │ ← Top priority alert
│  ℹ Network lag 250ms     │ ← Second alert
│──────────────────────────│
│ ◄MENU      DETAIL►       │ ← Navigation hints
└──────────────────────────┘
```

**Key Elements:**
- **Line 1-2:** System identification and time (always visible)
- **Line 3-4:** Critical sensor summary with icons
- **Line 5-6:** System health status
- **Line 7-9:** Active alerts (shows top 2)
- **Line 12:** Navigation hints

**Icons:** 🌡(temp) 💧(humidity) 🚪(door) 🔥(smoke) ⚡(power) 📶(network) 🔔(alert)

#### Sub-Screen 0B: Sensor Summary

```
┌──────────────────────────┐
│ SENSORS     12:34:59     │
│──────────────────────────│
│ TH1  24.5°C  58%  ✓ 2s  │ ← Sensor readings + age
│ TH2  25.1°C  60%  ✓ 1s  │
│ TH3  23.8°C  55%  ✓ 3s  │
│ TH4  26.2°C  62%  ⚠ 2s  │ ← Warning indicator
│──────────────────────────│
│ SMK1 OK  2s  NONC1 ✓ 1s │ ← Binary sensors
│ SMK2 OK  1s  NONC2 ✓ 2s │
│ SMK3 OK  3s  NONC3 ✓ 1s │
│ SMK4 OK  2s  NONC4 ✓ 3s │
│──────────────────────────│
│ ◄MENU      NEXT►         │
└──────────────────────────┘
```

**Data Display Rules:**
- Show last update age (e.g., "2s")
- Warning icon (⚠) if value out of range
- Status: ✓=OK, ⚠=Warning, ✗=Error, ?=Stale
- Stale data (>10s) shows "?" and dims

#### Sub-Screen 0C: Output Status

```
┌──────────────────────────┐
│ OUTPUTS     12:35:04     │
│──────────────────────────│
│ RLY1 ████████ ON  AUTO   │ ← Visual bar + mode
│   Cooling Fan            │ ← Friendly name
│   Rule: temp_ctrl        │ ← Active rule
│                          │
│ RLY2 ████████ ON  AUTO   │
│   Alert Horn             │
│   Rule: alarm_critical   │
│                          │
│ RLY3 ░░░░░░░░ OFF MANUAL │ ← Manual override
│   Emergency Vent         │
│──────────────────────────│
│ ◄MENU      NEXT►         │
└──────────────────────────┘
```

**Relay Display:**
- Visual bar: █ = ON, ░ = OFF
- Mode: AUTO, MANUAL, OVERRIDE, DISABLED
- Shows controlling rule name if AUTO
- Runtime could be added if space permits

---

### Screen 1: Menu Selection

**Purpose:** Navigate to detailed screens

```
┌──────────────────────────┐
│ ═══════ MAIN MENU ══════ │ ← Header bar
│──────────────────────────│
│ ► SENSORS DETAIL         │ ← Selected (►)
│   ALERTS HISTORY         │
│   RELAYS CONTROL         │
│   SYSTEM CONTROL    🔒   │ ← Lock icon = auth needed
│   DIAGNOSTICS            │
│   SYSTEM INFO            │
│   NETWORK STATUS         │
│──────────────────────────│
│ ↕=Navigate  OK=Select    │
│ ◄=Back to Dashboard      │
└──────────────────────────┘
```

**Behavior:**
- UP/DOWN: Move selection cursor (►)
- OK: Enter selected screen
- LEFT: Return to Dashboard
- 30-second timeout → auto-return to Dashboard
- Show lock icon (🔒) for protected screens

---

### Screen 2: Sensors Detail

**Purpose:** Detailed sensor information and history

```
┌──────────────────────────┐
│ SENSOR: TH1     (1/16)   │ ← Name + pagination
│──────────────────────────│
│ Instance Name:           │
│  server_room_temp        │ ← Full instance name
│                          │
│ Temperature: 24.5°C      │
│   Min: 22.1°C (03:15)    │ ← Daily min/max + time
│   Max: 26.8°C (14:32)    │
│                          │
│ Humidity: 58%            │
│   Status: NORMAL ✓       │
│   Last update: 2s ago    │
│──────────────────────────│
│ ↕=Switch  OK=Graph  ◄    │ ← Future: graph view
└──────────────────────────┘
```

**Features:**
- UP/DOWN: Cycle through all sensors
- Shows full instance name
- Daily min/max with timestamps
- Last update age
- Status: NORMAL, WARNING, ERROR, STALE, OFFLINE

**If sensor offline:**
```
│ Status: OFFLINE ✗        │
│ Last seen: 5m 23s ago    │
│ Check connection         │
```

---

### Screen 3: Alerts History

**Purpose:** Review and acknowledge alerts

```
┌──────────────────────────┐
│ ALERTS (3 ACTIVE)        │
│──────────────────────────│
│⚠HIGH 12:34 [ACTIVE]     │ ← Priority + time + state
│ TH4 Temp > 26°C          │ ← Alert description
│ └─ RLY1 activated        │ ← Action taken
│                          │
│⚠MED  12:30 [ACKED]      │ ← Acknowledged
│ NONC2 Door open          │
│                          │
│ℹLOW  12:15 [CLEAR]      │ ← Cleared
│ Network reconnected      │
│──────────────────────────│
│ ↕=Scroll  OK=Ack  ◄      │
└──────────────────────────┘
```

**Alert States:**
- **[ACTIVE]**: Currently active, not acknowledged
- **[ACKED]**: Acknowledged by user, still active
- **[CLEAR]**: Condition resolved, auto-clears after 30s

**Priority Indicators:**
- ⚠ CRIT: Critical (red LED, continuous beep)
- ⚠ HIGH: High priority (red LED, alert beep)
- ⚠ MED: Medium (yellow LED)
- ℹ LOW: Low (blue LED)

**Behavior:**
- Shows last 10 alerts (most recent first)
- UP/DOWN: Scroll through history
- OK on [ACTIVE] alert: Acknowledge
- Auto-scrolls if >3 alerts visible

---

### Screen 4: Relays Control

**Purpose:** Manual relay override (requires authentication)

```
┌──────────────────────────┐
│ RELAY CONTROL       🔒   │ ← Lock if not authenticated
│──────────────────────────│
│ ► RLY1 [ON]              │ ← Selected relay
│   Cooling Fan            │
│   Mode: AUTO             │
│   Rule: temp_ctrl        │
│   Runtime: 2h 34m        │
│                          │
│   RLY2 [ON]              │
│   Alert Horn             │
│   Mode: AUTO             │
│                          │
│   RLY3 [OFF]             │
│   Emergency Vent         │
│   Mode: MANUAL           │
│──────────────────────────│
│ ↕=Select OK=Toggle  ◄    │
└──────────────────────────┘
```

**If not authenticated:**
```
┌──────────────────────────┐
│ RELAY CONTROL       🔒   │
│──────────────────────────│
│                          │
│   Authentication         │
│   Required               │
│                          │
│   Press OK to enter PIN  │
│                          │
│                          │
│──────────────────────────│
│ OK=Login  ◄=Back         │
└──────────────────────────┘
```

**Confirmation Screen (when toggling):**
```
┌──────────────────────────┐
│ ⚠ MANUAL OVERRIDE        │
│══════════════════════════│
│                          │
│ Override RLY1?           │
│                          │
│ Current: ON (AUTO)       │
│ Active rule: temp_ctrl   │
│                          │
│ New state: OFF (MANUAL)  │
│                          │
│ ⚠ Disables auto control! │
│                          │
│══════════════════════════│
│ OK=Confirm  ◄=Cancel     │
└──────────────────────────┘
```

---

### Screen 5: System Control

**Purpose:** System maintenance operations (requires authentication)

```
┌──────────────────────────┐
│ SYSTEM CONTROL      🔒   │
│──────────────────────────│
│ ► Restart Backend API    │
│   Restart Sensor Hub     │
│   Restart Rule Engine    │
│   Reload Configuration   │
│   ─────────────────────  │
│   Test All Sensors       │
│   Test All Relays        │
│   Test Network Conn      │
│   ─────────────────────  │
│   Clear Alert History    │
│──────────────────────────│
│ ↕=Select OK=Execute  ◄   │
└──────────────────────────┘
```

**Confirmation Example:**
```
┌──────────────────────────┐
│ ⚠ CONFIRM ACTION         │
│══════════════════════════│
│                          │
│ Restart Backend API?     │
│                          │
│ This will:               │
│ • Stop Flask server      │
│ • Reload configuration   │
│ • Restart all services   │
│                          │
│ Duration: ~15 seconds    │
│ Dashboard will be DOWN   │
│                          │
│══════════════════════════│
│ OK=Confirm  ◄=Cancel     │
└──────────────────────────┘
```

**Progress Screen:**
```
┌──────────────────────────┐
│ RESTARTING API...        │
│──────────────────────────│
│                          │
│   ████████░░░░░░░░ 60%  │ ← Progress bar
│                          │
│ Step 3/5:                │
│ Reloading config files   │
│                          │
│ Elapsed: 9s              │
│ Please wait...           │
│                          │
│──────────────────────────│
│ [Processing...]          │
└──────────────────────────┘
```

**Result Screen:**
```
┌──────────────────────────┐
│ ✓ RESTART COMPLETE       │
│──────────────────────────│
│                          │
│ Backend API restarted    │
│ successfully!            │
│                          │
│ Status: HEALTHY          │
│ Services: 8/8 running    │
│ Config: Loaded OK        │
│                          │
│ Restart time: 14s        │
│                          │
│──────────────────────────│
│ OK=Continue              │
└──────────────────────────┘
```

---

### Screen 6: Diagnostics

**Purpose:** Hardware and connectivity testing

```
┌──────────────────────────┐
│ DIAGNOSTICS              │
│──────────────────────────│
│ ► Hardware Test          │
│   Sensor Connectivity    │
│   UART/Serial Ports      │
│   Modbus Bus Scan        │
│   GPIO Test              │
│   ─────────────────────  │
│   Network Ping Test      │
│   API Health Check       │
│   Database Status        │
│──────────────────────────│
│ ↕=Select OK=Run  ◄       │
└──────────────────────────┘
```

**Test Result Example (Sensor Connectivity):**
```
┌──────────────────────────┐
│ SENSOR TEST (1/2)        │
│──────────────────────────│
│ Modbus UART 1:           │
│  ✓ Port open             │
│  ✓ Slave 1 responding    │
│  ✓ Slave 2 responding    │
│  ✗ Slave 3 NO RESPONSE   │ ← Problem
│  ✓ Slave 4 responding    │
│                          │
│ Modbus UART 2:           │
│  ✓ Port open             │
│  ✓ All 4 slaves OK       │
│                          │
│ Result: 1 sensor offline │
│──────────────────────────│
│ ↕=Scroll  OK=Retry  ◄    │
└──────────────────────────┘
```

---

**Continue to next file for remaining screens...**
