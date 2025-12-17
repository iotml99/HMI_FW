# HMI System Overview

**Document Version:** 1.0
**Date:** 2025-11-25
**Status:** Design Specification

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Hardware Specifications](#hardware-specifications)
3. [Architecture](#architecture)
4. [Related Documentation](#related-documentation)

---

## System Overview

The RSU Monitoring System includes a dedicated HMI (Human-Machine Interface) for local monitoring and control when the web dashboard is unavailable. The HMI provides backup access to critical system functions, diagnostics, and manual override capabilities.

### Purpose

- **Primary Use Case**: Backup interface when web dashboard is unavailable
- **Secondary Use Case**: Local maintenance and diagnostics by on-site personnel
- **Tertiary Use Case**: Quick-glance system status monitoring

### Key Features

- ✅ Local display of sensor data and system status
- ✅ Manual relay control with authentication
- ✅ System diagnostics and testing
- ✅ Service restart capabilities
- ✅ Alert monitoring and acknowledgment
- ✅ Visual (LED) and audio (buzzer) feedback
- ✅ Firmware update capability
- ✅ PIN-based access control

---

## Hardware Specifications

### Display

| Specification | Value |
|---------------|-------|
| **Type** | OLED Monochrome |
| **Controller** | SSD1306 |
| **Resolution** | 128×96 pixels (width × height) |
| **Physical Size** | 0.96 inch diagonal |
| **Interface** | I²C or SPI (TBD) |
| **Colors** | Monochrome (white/blue on black) |
| **Viewing Distance** | Designed for distance viewing (data center rack) |

### Microcontroller

| Specification | Value |
|---------------|-------|
| **Family** | STM32 (ARM Cortex-M) |
| **Flash Memory** | 256 KB |
| **RAM** | TBD (typically 64-128 KB for STM32) |
| **Bootloader** | 16 KB reserved |
| **Application** | ~230 KB available |
| **Configuration** | 4 KB reserved |

### Input Devices

#### Buttons (5-Button Navigation)

| Button | Function |
|--------|----------|
| **UP** | Navigate up, scroll up, increment values |
| **DOWN** | Navigate down, scroll down, decrement values |
| **LEFT** | Go back, return to previous screen, cancel |
| **RIGHT** | Enter menu, next screen, confirm navigation |
| **OK** | Select, confirm, acknowledge alerts |

**Button Features:**
- Hardware debouncing recommended
- Long-press detection (>2 seconds)
- Release detection for timing
- No multi-button combinations required

#### LEDs (Up to 16 RGB LEDs)

**Recommended Allocation:**

| LED Index | Purpose | Default State |
|-----------|---------|---------------|
| 1-4 | Sensor Zone Status | Green = OK, Yellow = Warning, Red = Error |
| 5-7 | Relay Status | On/Off indicators |
| 8-10 | Alert Severity Indicators | Critical, High, Medium |
| 11 | API Connection Status | Green = Connected, Red = Disconnected |
| 12 | Network Status | Green = OK, Yellow = Lag, Red = Down |
| 13-14 | System Health | CPU, Memory warnings |
| 15 | Heartbeat/Activity | Slow blink = Normal operation |
| 16 | User/Custom | Available for custom use |

**LED Modes:**
- SOLID: Constant on
- BLINK: On/off pattern (configurable timing)
- BREATHE: Smooth fade in/out
- PULSE: Quick flash
- OFF: Disabled

#### Buzzer

**Specifications:**
- Piezo buzzer (3-5V)
- Frequency range: 2-5 kHz recommended
- PWM-controlled for tone generation

**Patterns:**
| Pattern ID | Name | Use Case | Sequence |
|------------|------|----------|----------|
| 0 | Off | Normal operation | Silent |
| 1 | Single Beep | Button press feedback | 200ms beep |
| 2 | Double Beep | Confirmation | 100ms, 50ms gap, 100ms |
| 3 | Alert | Critical alerts | Continuous beeping |
| 4 | Warning | Non-critical warnings | 500ms on, 1000ms off |
| 5 | Error | System errors | 200ms on, 200ms off (fast) |
| 6 | Success | Operation complete | Rising tone 300ms |

### Communication

| Specification | Value |
|---------------|-------|
| **Interface** | UART (Serial) |
| **Port** | `/dev/ttyO4` (BeagleBone) |
| **Baud Rate** | 115200 bps |
| **Data Bits** | 8 |
| **Parity** | None |
| **Stop Bits** | 1 |
| **Flow Control** | None (software optional) |
| **Cable** | 3-wire: TX, RX, GND |

---

## Architecture

### System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ BeagleBone Black (Python Backend)                          │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │ Sensor Hub   │  │ Rules Engine │  │  API Server     │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬────────┘  │
│         │                  │                    │           │
│         └──────────┬───────┴────────────────────┘           │
│                    │                                        │
│         ┌──────────▼──────────┐                            │
│         │  HMI Protocol       │                            │
│         │  Handler            │                            │
│         │  - Data formatting  │                            │
│         │  - Command parsing  │                            │
│         │  - Authentication   │                            │
│         └──────────┬──────────┘                            │
│                    │                                        │
└────────────────────┼────────────────────────────────────────┘
                     │ UART (115200 baud)
                     │ Text-based protocol with CRC-8
                     │
┌────────────────────▼────────────────────────────────────────┐
│ STM32 HMI Board (C Firmware)                                │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │ Protocol     │  │ UI Engine    │  │  Input Handler  │  │
│  │ Parser       │  │ - Screens    │  │  - Buttons      │  │
│  │              │  │ - Rendering  │  │  - Debouncing   │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬────────┘  │
│         │                  │                    │           │
│         └──────────┬───────┴────────────────────┘           │
│                    │                                        │
│         ┌──────────▼──────────┐                            │
│         │  Hardware Drivers   │                            │
│         │  - OLED (SSD1306)   │                            │
│         │  - LEDs (PWM)       │                            │
│         │  - Buzzer (PWM)     │                            │
│         │  - UART             │                            │
│         └─────────────────────┘                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

#### Periodic Updates (BeagleBone → HMI)

```
Every 5 seconds:
  - System status (CPU, memory, temperature, uptime)
  - Individual sensor updates (temperature, humidity, door status)
  - Relay status (if changed)

Every 10 seconds:
  - Network status
  - HMI status request

Every 60 seconds:
  - Time synchronization
```

#### Event-Driven Updates (BeagleBone → HMI)

```
Immediate:
  - Alert notifications (critical/high priority)
  - Alert cleared
  - Relay state changes
  - Command execution progress
  - Command results
  - Diagnostic test results
```

#### User Actions (HMI → BeagleBone)

```
On button press:
  - Button event notification
  - Screen navigation updates
  - Command execution requests
  - Relay control requests
  - Alert acknowledgments
  - Authentication attempts
```

### Memory Layout (STM32)

```
Flash Memory (256 KB):
┌─────────────────────────────────┐ 0x08000000
│ Bootloader                      │
│ - Minimal OLED driver           │ 16 KB
│ - UART handler                  │
│ - Flash operations              │
│ - Simple progress display       │
├─────────────────────────────────┤ 0x08004000
│ Application Firmware            │
│ - Full UI engine                │
│ - All screen layouts            │ ~230 KB
│ - Navigation system             │
│ - Protocol handler              │
│ - Graphics library              │
│ - Input handling                │
├─────────────────────────────────┤ 0x0803F000
│ Configuration Storage           │ 4 KB
│ - Settings                      │
│ - Calibration data              │
├─────────────────────────────────┤ 0x08040000
│ Reserved / Future Use           │ 0 KB
└─────────────────────────────────┘

RAM (64-128 KB typical):
┌─────────────────────────────────┐
│ Stack                           │ 4 KB
├─────────────────────────────────┤
│ Heap                            │ 20 KB
├─────────────────────────────────┤
│ Graphics Buffer                 │ 1.5 KB (128×96 / 8)
├─────────────────────────────────┤
│ Sensor Data Cache               │ 2 KB
├─────────────────────────────────┤
│ UART Buffers (TX/RX)            │ 2 KB
├─────────────────────────────────┤
│ Application Variables           │ Remaining
└─────────────────────────────────┘
```

---

## Design Principles

### 1. Autonomous Operation

**Principle:** HMI must function independently when BeagleBone fails.

**Implementation:**
- STM32 owns display completely
- Local rendering of all screens
- Connection loss detection and display
- Graceful degradation with stale data
- Local button handling without BB dependency

### 2. Data-Driven Protocol

**Principle:** Send data, not pixels or frames.

**Implementation:**
- BB sends high-level data: `SENSOR 1 TH 24.5 58.0 OK 2`
- STM32 decides how to render it
- Reduces UART bandwidth
- Eliminates display latency
- Simplifies protocol

### 3. Fail-Safe Design

**Principle:** Critical monitoring system must be robust.

**Implementation:**
- Watchdog timers on STM32
- Heartbeat mechanism
- CRC checksums on all messages
- Timeout and retry logic
- Automatic rollback on firmware failure
- Connection loss indicators

### 4. Security by Design

**Principle:** Protect critical operations from unauthorized access.

**Implementation:**
- PIN-based authentication
- Session tokens with expiry
- Protected command categories
- Audit logging of privileged actions
- Bootloader access requires admin PIN

### 5. Maintenance-Friendly

**Principle:** Enable local troubleshooting without remote access.

**Implementation:**
- Comprehensive diagnostics menu
- System restart capabilities
- Connection testing tools
- Log viewing on display
- Manual relay override

---

## Performance Characteristics

### Display Update Rate

| Screen Type | Update Frequency | Rationale |
|-------------|------------------|-----------|
| Main Dashboard | 5 seconds (auto-scroll) | Balance freshness with readability |
| Sensor Detail | On data change (~5s) | Real-time enough for monitoring |
| Alert Overlay | Immediate | Critical information |
| Menu/Navigation | Instant (local) | Responsive user experience |
| Diagnostics | On demand | Run when requested |

### Communication Bandwidth

**At 115200 baud = 11,520 bytes/second theoretical:**

| Traffic Type | Bytes/Message | Frequency | Bandwidth | % Used |
|--------------|---------------|-----------|-----------|--------|
| System Status | 80 | 5s | 16 B/s | 0.14% |
| Sensor Updates (×16) | 35 | 5s each | 112 B/s | 0.97% |
| Relay Status (×3) | 40 | On change | ~2 B/s | 0.02% |
| Heartbeat | 40 | 5s | 8 B/s | 0.07% |
| Time Sync | 60 | 60s | 1 B/s | 0.01% |
| HMI Status | 50 | 10s | 5 B/s | 0.04% |
| **Total Periodic** | | | **~144 B/s** | **1.25%** |

**Remaining bandwidth:** ~11,376 B/s (98.75%) available for:
- Alert notifications
- Command execution
- Diagnostic results
- Firmware updates (uses ~8 KB/s during update)
- Future expansion

### Latency Targets

| Operation | Target Latency | Notes |
|-----------|----------------|-------|
| Button press feedback | <50ms | Visual/audio confirmation |
| Screen navigation | <100ms | Local rendering, instant |
| Data display update | <5s | Via periodic updates |
| Alert display | <500ms | Immediate protocol message |
| Command execution | 1-30s | Depends on command type |
| Firmware update | 2-3 minutes | 230 KB @ ~1.5 KB/s |

---

## Related Documentation

- **[HMI Screens Design](./HMI_SCREENS.md)** - Detailed screen layouts and navigation
- **[HMI Protocol](./HMI_PROTOCOL.md)** - UART communication protocol specification
- **[HMI Firmware](./HMI_FIRMWARE.md)** - STM32 firmware architecture and update process
- **[HMI Authentication](./HMI_AUTH.md)** - Security and access control
- **[HMI Configuration](./HMI_CONFIG.md)** - Configuration file format and settings

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-11-25 | Initial | Complete system design specification |

---

**Next Document:** [HMI_SCREENS.md](./HMI_SCREENS.md) - Screen layouts and user interface design
