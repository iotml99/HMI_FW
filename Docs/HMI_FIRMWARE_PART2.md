# HMI Firmware Update & Rollback

**Continued from:** [HMI_FIRMWARE.md](./HMI_FIRMWARE.md)

---

## Firmware Update Process

### Overview

```
Total Time: ~2-3 minutes
Phases:
  1. Preparation (5-10s): File validation, auth
  2. Enter Bootloader (2s): App → Bootloader transition
  3. Erase (3-5s): Flash sector erase
  4. Write (60-90s): Transfer firmware in chunks
  5. Verify (2-3s): CRC validation
  6. Launch (2s): Bootloader → New app
```

### Phase 1: Preparation

**Backend Actions:**
1. Firmware file arrives via:
   - Web dashboard upload
   - Automatic download from update server
   - Manual file placement

2. File stored in `/var/lib/rsu-hmi/firmware/updates/`

3. Validate firmware:
   - Check file size (must be ≤ 236 KB)
   - Calculate CRC-32
   - Parse firmware header (version, build date)

4. Backup current firmware info to `/var/lib/rsu-hmi/firmware/backup/`

**HMI Actions:**
1. User navigates to System Control → "Update Firmware"
2. Query backend for available update: `BL CHECK`
3. Display confirmation screen with version info
4. Request admin PIN authentication
5. On confirm: initiate update

### Phase 2: Enter Bootloader

**Sequence:**
```python
# Backend
def initiate_firmware_update(firmware_file):
    logger.info(f"Starting firmware update: {firmware_file}")

    # Read firmware
    with open(firmware_file, 'rb') as f:
        firmware_data = f.read()

    # Calculate expected CRC
    expected_crc = calculate_crc32(firmware_data)

    # Request bootloader entry
    send_command(f"BL ENTER {admin_pin}")
    response = wait_for_response(timeout=5)

    if "OK BL ENTERING" not in response:
        raise Exception("Failed to enter bootloader")

    # Wait for bootloader ready
    time.sleep(2)
    response = wait_for_message(pattern="BL READY", timeout=10)

    if "BL READY" not in response:
        raise Exception("Bootloader did not respond")

    logger.info("Bootloader ready")
    return firmware_data, expected_crc
```

```c
// HMI Application
void enter_bootloader_mode(void) {
    // Disable interrupts
    __disable_irq();

    // Save state if needed
    save_configuration();

    // Clear update flag in RAM
    *(volatile uint32_t*)0x20000000 = 0xDEADBEEF;

    // Reset
    NVIC_SystemReset();
}

// Bootloader checks flag on startup
void bootloader_main(void) {
    // ... init ...

    // Check for update request
    if (*(volatile uint32_t*)0x20000000 == 0xDEADBEEF) {
        // Clear flag
        *(volatile uint32_t*)0x20000000 = 0;

        // Enter update mode
        bootloader_update_mode();
    }

    // Otherwise check and launch app
    // ...
}
```

### Phase 3: Erase Flash

**Duration:** 3-5 seconds (depends on sector count)

```python
# Backend
def erase_application_flash(firmware_size):
    logger.info(f"Erasing {firmware_size} bytes...")

    send_command(f"BL ERASE 0x08004000 {firmware_size}")
    response = wait_for_response(timeout=2)

    if "OK BL ERASING" not in response:
        raise Exception("Erase rejected")

    # Monitor progress
    while True:
        msg = wait_for_message(timeout=10)

        if "BL PROGRESS" in msg:
            percent = extract_progress(msg)
            logger.info(f"Erase progress: {percent}%")
            # Could emit progress event for UI

        elif "OK BL ERASED" in msg:
            logger.info("Erase complete")
            break

        elif "ERR" in msg:
            raise Exception(f"Erase failed: {msg}")
```

### Phase 4: Write Firmware

**Duration:** 60-90 seconds for 230 KB
**Chunk Size:** 256 bytes
**Total Chunks:** ~900 chunks
**Throughput:** ~2-4 KB/s (limited by flash write speed)

```python
# Backend
def write_firmware(firmware_data):
    chunk_size = 256
    total_chunks = (len(firmware_data) + chunk_size - 1) // chunk_size
    addr = 0x08004000

    logger.info(f"Writing {len(firmware_data)} bytes in {total_chunks} chunks...")

    for i in range(total_chunks):
        offset = i * chunk_size
        chunk = firmware_data[offset:offset + chunk_size]

        # Pad last chunk if needed
        if len(chunk) < chunk_size:
            chunk += b'\xFF' * (chunk_size - len(chunk))

        # Convert to hex string
        hex_data = chunk.hex().upper()

        # Send write command
        send_command(f"BL WRITE {addr + offset:08X} {hex_data}")
        response = wait_for_response(timeout=2)

        if "OK BL WRITTEN 256" not in response:
            raise Exception(f"Write failed at chunk {i}")

        # Log progress every 10 chunks
        if i % 10 == 0:
            percent = (i * 100) // total_chunks
            logger.info(f"Write progress: {percent}% ({i}/{total_chunks})")

        # Check for async progress messages
        check_for_progress_updates()

    logger.info("Write complete")
```

**Optimization Strategies:**

1. **Batch Commands:** Could send multiple chunks without waiting (risky)
2. **Compression:** Compress firmware, decompress on STM32 (adds complexity)
3. **Differential Updates:** Only send changed blocks (complex implementation)

**Recommended:** Keep it simple and reliable. 60-90s is acceptable for infrequent updates.

### Phase 5: Verify

**Duration:** 2-3 seconds

```python
# Backend
def verify_firmware(expected_crc):
    logger.info("Verifying firmware...")

    send_command("BL VERIFY")
    response = wait_for_response(timeout=2)

    if "OK BL VERIFYING" not in response:
        raise Exception("Verify rejected")

    # Wait for result
    response = wait_for_message(timeout=10)

    if "OK BL CRC" in response:
        # Extract CRC from response
        calculated_crc = extract_crc(response)

        if "MATCH" in response:
            logger.info(f"CRC verified: 0x{calculated_crc:08X}")
            return True
        else:
            logger.error(f"CRC mismatch: expected 0x{expected_crc:08X}, got 0x{calculated_crc:08X}")
            return False

    elif "ERR" in response:
        raise Exception(f"Verify failed: {response}")
```

**CRC-32 Implementation (both sides must match):**

```python
# Python
def calculate_crc32(data):
    import zlib
    return zlib.crc32(data) & 0xFFFFFFFF
```

```c
// C (STM32)
uint32_t calculate_flash_crc(uint32_t addr, uint32_t length) {
    const uint32_t polynomial = 0xEDB88320;
    uint32_t crc = 0xFFFFFFFF;

    for (uint32_t i = 0; i < length; i++) {
        uint8_t byte = *(volatile uint8_t*)(addr + i);
        crc ^= byte;

        for (int j = 0; j < 8; j++) {
            if (crc & 1)
                crc = (crc >> 1) ^ polynomial;
            else
                crc >>= 1;
        }
    }

    return ~crc;
}
```

### Phase 6: Launch New Application

**Duration:** 2 seconds

```python
# Backend
def launch_new_firmware():
    logger.info("Launching new firmware...")

    send_command("BL LAUNCH")
    response = wait_for_response(timeout=2)

    if "OK BL LAUNCHING" not in response:
        raise Exception("Launch rejected")

    # Wait for application to start
    time.sleep(2)

    # Wait for hello message
    response = wait_for_message(pattern="HMI HELLO", timeout=10)

    if "HMI HELLO" in response:
        version = extract_version(response)
        logger.info(f"New firmware started: version {version}")

        # Send connection acknowledgment
        send_command("OK HMI CONNECTED BB 2.1.3")

        return version
    else:
        raise Exception("New firmware did not start")
```

### Complete Update Function

```python
def update_hmi_firmware(firmware_file, admin_pin):
    """
    Complete firmware update procedure with error handling
    """
    try:
        # Phase 1: Preparation
        logger.info("=== Phase 1: Preparation ===")
        firmware_data, expected_crc = initiate_firmware_update(firmware_file)

        # Phase 2: Enter Bootloader
        logger.info("=== Phase 2: Enter Bootloader ===")
        # (handled in initiate_firmware_update)

        # Phase 3: Erase
        logger.info("=== Phase 3: Erase Flash ===")
        erase_application_flash(len(firmware_data))

        # Phase 4: Write
        logger.info("=== Phase 4: Write Firmware ===")
        write_firmware(firmware_data)

        # Phase 5: Verify
        logger.info("=== Phase 5: Verify ===")
        if not verify_firmware(expected_crc):
            raise Exception("CRC verification failed")

        # Phase 6: Launch
        logger.info("=== Phase 6: Launch ===")
        new_version = launch_new_firmware()

        logger.info(f"✓ Firmware update successful: {new_version}")
        return True

    except Exception as e:
        logger.error(f"✗ Firmware update failed: {e}")

        # Attempt recovery
        logger.info("Attempting rollback...")
        rollback_firmware()

        return False
```

---

## Rollback Strategy

### Automatic Rollback

**Trigger Conditions:**
1. New firmware fails CRC verification
2. New firmware crashes on first boot (watchdog reset)
3. New firmware fails to send HELLO message within 10 seconds
4. Backend detects HMI malfunction after update

### Rollback Mechanisms

#### Method 1: Keep Old Firmware in Flash

**Not Recommended** - Requires 2× flash space (512 KB needed)

```
Flash Layout (512 KB required):
┌────────────────┐ 0x08000000
│ Bootloader     │ 16 KB
├────────────────┤ 0x08004000
│ App Bank A     │ 236 KB (active)
├────────────────┤ 0x08040000
│ App Bank B     │ 236 KB (backup)
├────────────────┤ 0x0807C000
│ Config         │ 4 KB
└────────────────┘ 0x0807D000
```

**Problem:** Our STM32 only has 256 KB flash.

#### Method 2: Backend-Stored Rollback (RECOMMENDED)

**Strategy:** Backend keeps last working firmware, can re-flash on failure.

```
Backend Storage:
/var/lib/rsu-hmi/firmware/
├── current/
│   └── hmi_v1.0.1.bin          ← Currently running
├── backup/
│   └── hmi_v1.0.0.bin          ← Last known good
└── updates/
    └── hmi_v1.0.2.bin          ← Pending update
```

**Rollback Procedure:**
```python
def rollback_firmware():
    logger.warning("Rolling back to previous firmware...")

    backup_file = "/var/lib/rsu-hmi/firmware/backup/hmi_v1.0.0.bin"

    if not os.path.exists(backup_file):
        logger.error("No backup firmware available!")
        return False

    # Re-flash backup firmware
    try:
        update_hmi_firmware(backup_file, admin_pin)
        logger.info("Rollback successful")
        return True
    except Exception as e:
        logger.error(f"Rollback failed: {e}")
        return False
```

#### Method 3: Minimal Safe Mode Application

**Hybrid Approach:** Add 20 KB "safe mode" app between bootloader and main app.

```
Flash Layout (256 KB):
┌────────────────┐ 0x08000000
│ Bootloader     │ 16 KB
├────────────────┤ 0x08004000
│ Safe Mode App  │ 20 KB (minimal UI, network recovery)
├────────────────┤ 0x08009000
│ Main App       │ 216 KB
├────────────────┤ 0x0803F000
│ Config         │ 4 KB
└────────────────┘ 0x08040000
```

**Safe Mode Features:**
- Minimal display: "RECOVERY MODE"
- UART active
- Can receive new firmware from backend
- Cannot be overwritten
- Automatically entered if main app invalid

**Bootloader Logic:**
```c
void bootloader_main(void) {
    // ... init ...

    // Check main app
    if (verify_main_application()) {
        launch_main_application();
    }

    // Main app invalid, try safe mode
    if (verify_safe_mode_application()) {
        launch_safe_mode_application();
    }

    // Both invalid, stay in bootloader
    bootloader_update_mode();
}
```

**Recommended:** Use Method 2 (Backend-Stored) for simplicity. Method 3 adds robustness but complexity.

### Watchdog-Based Rollback

**Detect Crash on First Boot:**

```c
// application/main.c

int main(void) {
    // Check if this is first boot after update
    if (read_backup_register(BKP_REG_FIRST_BOOT) == 0xFIRSTBOOT) {
        // First boot after update

        // Start watchdog (30 second timeout)
        iwdg_init(30000);

        // Run self-test
        if (!self_test_passed()) {
            // Self-test failed, revert to bootloader
            write_backup_register(BKP_REG_UPDATE_FLAG, 0xREVERT);
            NVIC_SystemReset();
        }

        // Self-test passed
        write_backup_register(BKP_REG_FIRST_BOOT, 0);

        // Send hello message to backend
        send_hello_message();

        // Wait for acknowledgment
        if (!wait_for_backend_ack(10000)) {
            // Backend not happy, revert
            write_backup_register(BKP_REG_UPDATE_FLAG, 0xREVERT);
            NVIC_SystemReset();
        }

        // Success! Mark firmware as good
        write_backup_register(BKP_REG_FW_STATUS, 0xGOOD);
    }

    // Normal startup...
}
```

**Backend Verification:**
```python
def verify_new_firmware_healthy():
    """
    Verify new firmware after update
    """
    # Wait for HELLO message
    response = wait_for_message(pattern="HMI HELLO", timeout=10)

    if "HMI HELLO" not in response:
        logger.error("New firmware did not respond")
        return False

    version = extract_version(response)

    # Send ACK
    send_command("OK HMI CONNECTED BB 2.1.3")

    # Test basic functionality
    time.sleep(2)

    # Request status
    send_command("HMI STATUS")
    response = wait_for_message(timeout=5)

    if "HMI STATUS" not in response:
        logger.error("New firmware not responding to commands")
        return False

    # Firmware is healthy
    logger.info(f"New firmware verified healthy: {version}")
    return True
```

### Recovery from Bricked State

**If HMI completely unresponsive:**

1. **Power Cycle with Boot0 Pin:**
   - Hardware BOOT0 pin forced high → enters ST bootloader
   - Use ST-Link or UART bootloader to reflash
   - Requires physical access

2. **JTAG/SWD Recovery:**
   - Connect debugger (ST-Link, J-Link)
   - Erase and reflash entire device
   - Requires physical access

3. **Prevention:**
   - Never overwrite bootloader region
   - Always verify CRC before launch
   - Implement safe mode application
   - Use watchdog for crash detection

---

## Firmware Update Best Practices

### Development

1. **Version Everything:**
   - Embed version in firmware header
   - Include build date/time
   - Git commit hash optional

2. **Test Before Deploy:**
   - Test update procedure on dev board
   - Verify CRC calculation matches
   - Test rollback scenarios

3. **Incremental Updates:**
   - Don't skip versions in production
   - Document changes between versions
   - Keep release notes

### Production

1. **Staged Rollout:**
   - Update one device first
   - Monitor for 24 hours
   - Then update remaining devices

2. **Backup Before Update:**
   - Backend stores previous firmware
   - Keep last 3 versions
   - Document known-good versions

3. **Update Window:**
   - Schedule during low-activity periods
   - Notify operators of downtime
   - Have rollback plan ready

4. **Monitoring:**
   - Log all update attempts
   - Alert on failures
   - Track firmware versions across fleet

### Security

1. **Firmware Signing:**
   - Sign firmware with private key
   - Bootloader verifies signature
   - Prevents unauthorized firmware

2. **Encrypted Transfer:**
   - UART traffic is plain text
   - Consider encrypting firmware payload
   - Use TLS for network downloads

3. **Admin Authorization:**
   - Require admin PIN for updates
   - Log who initiated update
   - Rate-limit update attempts

---

**Next Document:** [HMI_CONFIG.md](./HMI_CONFIG.md) - Configuration specification
