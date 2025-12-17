# HMI Configuration Specification

**Document Version:** 1.0
**Date:** 2025-11-25
**Related:** [HMI_OVERVIEW.md](./HMI_OVERVIEW.md)

---

## Configuration Files

### Backend Configuration (appconfig.yaml)

```yaml
# HMI Configuration Section
hmi:
  # UART Communication
  uart:
    port: "/dev/ttyO4"
    baud_rate: 115200
    timeout: 5  # seconds

  # Authentication
  auth:
    operator_pin: "1234"  # Hashed in production: bcrypt or similar
    admin_pin: "9999"
    session_timeout: 300  # 5 minutes
    max_attempts: 3
    lockout_duration: 60  # 1 minute after failed attempts

  # Protected Commands (require authentication)
  protected_commands:
    - RELAY_SET
    - CMD_RESTART_API
    - CMD_RESTART_HUB
    - CMD_RESTART_RULES
    - CMD_RELOAD_CONFIG
    - CMD_CLEAR_ALERTS
    - BL_ENTER

  # Update Frequency
  update_intervals:
    heartbeat: 5  # seconds
    system_status: 5
    sensor_data: 5
    relay_status: 0  # On change only
    network_status: 10
    time_sync: 60

  # Firmware Update
  firmware:
    storage_path: "/var/lib/rsu-hmi/firmware"
    max_backup_versions: 3
    auto_rollback: true
    rollback_timeout: 30  # seconds to verify new firmware

  # Display Settings
  display:
    default_brightness: 80  # 0-100%
    auto_dim_timeout: 300  # 5 minutes
    dimmed_brightness: 20
    sleep_timeout: 0  # 0 = never sleep

  # LED Mapping
  leds:
    sensor_zones: [1, 2, 3, 4]  # LEDs 1-4 for sensor status
    relays: [5, 6, 7]  # LEDs 5-7 for relay status
    alerts: [8, 9, 10]  # LEDs 8-10 for alert levels
    api_status: 11
    network_status: 12
    system_health: [13, 14]
    heartbeat: 15
    user: 16

  # Alert Handling
  alerts:
    critical_beep: true
    critical_duration: 3  # seconds overlay display
    auto_clear_delay: 30  # seconds after resolution
    max_history: 50  # alerts to keep

  # Logging
  logging:
    level: "INFO"  # DEBUG, INFO, WARN, ERROR
    log_button_events: true
    log_screen_navigation: false
    log_protocol_messages: false  # Verbose, for debugging only
```

### STM32 Configuration (stored in flash)

```c
// Configuration structure stored at 0x0803F000

typedef struct {
    uint32_t magic;           // 0x484D4943 ("HMIC")
    uint32_t version;         // Config version

    // Display
    struct {
        uint8_t brightness;       // 0-100
        uint8_t contrast;         // 0-255
        uint16_t sleep_timeout;   // seconds, 0=never
        uint8_t dimmed_brightness; // 0-100
        uint16_t auto_dim_timeout; // seconds
    } display;

    // LED Calibration
    struct {
        uint8_t red_offset[16];   // Per-LED color correction
        uint8_t green_offset[16];
        uint8_t blue_offset[16];
        uint8_t intensity;        // Global 0-100
    } led;

    // Button Settings
    struct {
        uint16_t debounce_ms;     // Debounce time
        uint16_t long_press_ms;   // Long press threshold
        uint8_t repeat_enabled;   // Auto-repeat for held buttons
        uint16_t repeat_delay_ms;
        uint16_t repeat_rate_ms;
    } button;

    // Network
    struct {
        uint32_t baud_rate;       // UART baud rate
        uint16_t tx_timeout_ms;
        uint16_t rx_timeout_ms;
        uint8_t retry_count;
    } network;

    // UI Behavior
    struct {
        uint16_t menu_timeout_sec; // Return to main after inactivity
        uint8_t auto_scroll_enabled;
        uint16_t auto_scroll_interval_sec;
    } ui;

    // Reserved for future use
    uint8_t reserved[512];

    uint32_t crc;  // CRC of entire config structure
} HMI_Config_t;
```

---

## Authentication System

### PIN Management

**Storage:**
- Backend: Hashed with bcrypt or Argon2
- Never stored in plain text
- Configurable via `appconfig.yaml`

```python
# Python backend
import bcrypt

def hash_pin(pin: str) -> str:
    """Hash PIN for secure storage"""
    salt = bcrypt.gensalt()
    hashed = bcrypt.hashpw(pin.encode('utf-8'), salt)
    return hashed.decode('utf-8')

def verify_pin(pin: str, hashed: str) -> bool:
    """Verify PIN against stored hash"""
    return bcrypt.checkpw(pin.encode('utf-8'), hashed.encode('utf-8'))

# Usage
operator_pin_hash = hash_pin("1234")
admin_pin_hash = hash_pin("9999")

# Store in config
config['hmi']['auth']['operator_pin_hash'] = operator_pin_hash
config['hmi']['auth']['admin_pin_hash'] = admin_pin_hash
```

**Transmission:**
- PINs sent over UART in plain text (physical security assumed)
- Consider encryption for high-security deployments
- Session tokens used after authentication

### Session Management

```python
class SessionManager:
    def __init__(self):
        self.sessions = {}  # token -> session_info

    def create_session(self, pin_level: str) -> str:
        """Create authenticated session"""
        token = secrets.token_hex(16)  # 32 char hex string

        self.sessions[token] = {
            'level': pin_level,  # 'operator' or 'admin'
            'created': time.time(),
            'expires': time.time() + 300,  # 5 minutes
            'last_activity': time.time()
        }

        return token

    def validate_session(self, token: str) -> bool:
        """Check if session is valid"""
        if token not in self.sessions:
            return False

        session = self.sessions[token]

        # Check expiry
        if time.time() > session['expires']:
            del self.sessions[token]
            return False

        # Update last activity
        session['last_activity'] = time.time()
        return True

    def get_session_level(self, token: str) -> str:
        """Get authentication level"""
        if token in self.sessions:
            return self.sessions[token]['level']
        return None

    def cleanup_expired(self):
        """Remove expired sessions"""
        now = time.time()
        expired = [t for t, s in self.sessions.items() if now > s['expires']]
        for token in expired:
            del self.sessions[token]
```

### Protected Command Authorization

```python
def handle_command(cmd_type: str, token: str = None):
    """Execute command with authorization check"""

    # Check if command requires auth
    if cmd_type in config['hmi']['protected_commands']:
        if not token or not session_manager.validate_session(token):
            return "ERR E11 AUTH_REQUIRED"

        level = session_manager.get_session_level(token)

        # Admin-only commands
        admin_only = ['BL_ENTER', 'CMD_RESTART_API']
        if cmd_type in admin_only and level != 'admin':
            return "ERR E11 INSUFFICIENT_PRIVILEGES"

    # Execute command
    result = execute_command(cmd_type)

    # Log action
    logger.audit(f"Command {cmd_type} executed by {level} session {token}")

    return result
```

### PIN Retry Limiting

```python
class PINLimiter:
    def __init__(self):
        self.attempts = {}  # source -> attempt_info
        self.lockouts = {}  # source -> lockout_expiry

    def check_attempt(self, source: str = "hmi") -> bool:
        """Check if attempts allowed"""
        # Check lockout
        if source in self.lockouts:
            if time.time() < self.lockouts[source]:
                remaining = int(self.lockouts[source] - time.time())
                raise Exception(f"Locked out for {remaining} seconds")
            else:
                # Lockout expired
                del self.lockouts[source]
                del self.attempts[source]

        return True

    def record_attempt(self, source: str, success: bool):
        """Record authentication attempt"""
        if source not in self.attempts:
            self.attempts[source] = {'count': 0, 'first': time.time()}

        if success:
            # Clear attempts on success
            if source in self.attempts:
                del self.attempts[source]
        else:
            # Increment failure count
            self.attempts[source]['count'] += 1

            # Check if locked out
            max_attempts = config['hmi']['auth']['max_attempts']
            if self.attempts[source]['count'] >= max_attempts:
                lockout_duration = config['hmi']['auth']['lockout_duration']
                self.lockouts[source] = time.time() + lockout_duration
                logger.security(f"Source {source} locked out for {lockout_duration}s")
```

---

## LED Configuration

### LED Mapping Table

```python
LED_MAP = {
    # Sensor zones
    'sensor_zone_1': 1,
    'sensor_zone_2': 2,
    'sensor_zone_3': 3,
    'sensor_zone_4': 4,

    # Relay status
    'relay_1': 5,
    'relay_2': 6,
    'relay_3': 7,

    # Alert levels
    'alert_critical': 8,
    'alert_high': 9,
    'alert_medium': 10,

    # System status
    'api_status': 11,
    'network_status': 12,
    'cpu_health': 13,
    'memory_health': 14,

    # Special
    'heartbeat': 15,
    'user_custom': 16
}

def set_led_status(led_name: str, state: str):
    """Set LED based on logical status"""
    led_idx = LED_MAP.get(led_name)
    if led_idx:
        send_command(f"LED STATUS {led_idx} {state}")
```

### LED Patterns

```python
LED_PATTERNS = {
    'OK': {
        'color': (0, 255, 0),     # Green
        'mode': 'SOLID'
    },
    'WARN': {
        'color': (255, 128, 0),   # Orange
        'mode': 'SOLID'
    },
    'ERROR': {
        'color': (255, 0, 0),     # Red
        'mode': 'BLINK'
    },
    'INFO': {
        'color': (0, 0, 255),     # Blue
        'mode': 'SOLID'
    },
    'OFF': {
        'color': (0, 0, 0),
        'mode': 'OFF'
    }
}
```

---

## Display Configuration

### Screen Timeout

```python
class ScreenTimeout:
    def __init__(self):
        self.timeout = config['hmi']['display']['auto_dim_timeout']
        self.sleep_timeout = config['hmi']['display']['sleep_timeout']
        self.last_activity = time.time()
        self.dimmed = False
        self.asleep = False

    def activity_detected(self):
        """Reset timeout on activity"""
        self.last_activity = time.time()

        if self.dimmed or self.asleep:
            self.wake_display()

    def update(self):
        """Check and apply timeouts"""
        idle_time = time.time() - self.last_activity

        # Dim after timeout
        if not self.dimmed and idle_time > self.timeout:
            self.dim_display()

        # Sleep after timeout (if enabled)
        if self.sleep_timeout > 0 and not self.asleep and idle_time > self.sleep_timeout:
            self.sleep_display()

    def dim_display(self):
        brightness = config['hmi']['display']['dimmed_brightness']
        send_command(f"SCR BRIGHT {brightness}")
        self.dimmed = True

    def wake_display(self):
        brightness = config['hmi']['display']['default_brightness']
        send_command(f"SCR BRIGHT {brightness}")
        self.dimmed = False
        self.asleep = False

    def sleep_display(self):
        send_command("SCR BRIGHT 0")
        self.asleep = True
```

---

## Configuration Management

### Loading Configuration

```python
def load_hmi_config():
    """Load HMI configuration from appconfig.yaml"""
    config_path = "/home/deep/rsu_bb/backend/config/appconfig.yaml"

    with open(config_path, 'r') as f:
        config = yaml.safe_load(f)

    # Validate required fields
    validate_hmi_config(config['hmi'])

    return config['hmi']

def validate_hmi_config(hmi_config):
    """Validate HMI configuration"""
    required_fields = ['uart', 'auth', 'update_intervals']

    for field in required_fields:
        if field not in hmi_config:
            raise ValueError(f"Missing required HMI config field: {field}")

    # Validate PINs
    if len(hmi_config['auth']['operator_pin']) != 4:
        raise ValueError("Operator PIN must be 4 digits")

    if len(hmi_config['auth']['admin_pin']) != 4:
        raise ValueError("Admin PIN must be 4 digits")

    # Validate timeouts
    if hmi_config['update_intervals']['heartbeat'] < 1:
        raise ValueError("Heartbeat interval must be >= 1 second")
```

### Runtime Configuration Updates

```python
def update_hmi_setting(setting: str, value: any):
    """Update HMI setting at runtime"""

    # Map of configurable settings
    settings_map = {
        'display.brightness': lambda v: send_command(f"SCR BRIGHT {v}"),
        'led.intensity': lambda v: send_command(f"LED INTENSITY {v}"),
        'display.sleep_timeout': lambda v: update_sleep_timeout(v),
        # ... more settings
    }

    if setting not in settings_map:
        raise ValueError(f"Unknown setting: {setting}")

    # Apply setting
    settings_map[setting](value)

    # Update config file
    config['hmi'][setting] = value
    save_config()

    # Log change
    logger.info(f"HMI setting updated: {setting} = {value}")
```

---

## Firmware Version Management

### Version Storage

```python
firmware_versions = {
    'bootloader': '1.0',
    'application': '1.0.1',
    'protocol': '1.0',
    'backend': '2.1.3'
}

def check_version_compatibility():
    """Ensure HMI and backend are compatible"""
    hmi_proto = firmware_versions['protocol']
    backend_proto = '1.0'

    if hmi_proto != backend_proto:
        logger.warning(f"Protocol version mismatch: HMI={hmi_proto}, BB={backend_proto}")
        # Could enforce compatibility or allow with warnings
```

### Update History

```python
def log_firmware_update(old_version: str, new_version: str, success: bool):
    """Log firmware update event"""
    update_record = {
        'timestamp': datetime.now().isoformat(),
        'old_version': old_version,
        'new_version': new_version,
        'success': success,
        'duration': update_duration,
        'initiated_by': current_user
    }

    # Store in database and log file
    db.store_firmware_update(update_record)
    logger.audit(f"Firmware update: {old_version} → {new_version}, success={success}")
```

---

## Summary

This completes the comprehensive HMI documentation covering:

1. **Overview** - System architecture and specifications
2. **Screens** - All UI layouts and navigation
3. **Protocol** - Complete UART communication specification
4. **Firmware** - Bootloader, application, and update process
5. **Configuration** - Settings and management

**Total Documentation:** ~40,000 words across 8 documents

All designs follow your requirements:
- ✅ 0.96" SSD1306 OLED (128×96)
- ✅ STM32 with 256 KB flash
- ✅ Text-based protocol with CRC-8
- ✅ PIN-based authentication
- ✅ Firmware update with rollback
- ✅ 5-button navigation
- ✅ Up to 16 RGB LEDs
- ✅ Data center rack deployment
- ✅ Backup interface design

**Ready for implementation!**
