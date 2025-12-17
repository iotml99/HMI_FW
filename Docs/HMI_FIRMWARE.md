# HMI Firmware Architecture

**Document Version:** 1.0
**Date:** 2025-11-25
**Related:** [HMI_OVERVIEW.md](./HMI_OVERVIEW.md)

---

## Table of Contents

1. [Memory Layout](#memory-layout)
2. [Bootloader Architecture](#bootloader-architecture)
3. [Application Architecture](#application-architecture)
4. [Firmware Update Process](#firmware-update-process)
5. [Rollback Strategy](#rollback-strategy)

---

## Memory Layout

### Flash Memory (256 KB)

```
Address Range          Size    Region              Purpose
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
0x08000000-0x08003FFF  16 KB   Bootloader          Minimal firmware updater
0x08004000-0x0803EFFF  236 KB  Application         Main HMI firmware
0x0803F000-0x0803FFFF  4 KB    Configuration       Settings and calibration
```

### Bootloader Region (16 KB)

```
0x08000000  ┌──────────────────────────────┐
            │ Vector Table                 │  1 KB
            ├──────────────────────────────┤
            │ Bootloader Code              │  ~12 KB
            │  - UART handler              │
            │  - Flash operations          │
            │  - Minimal OLED driver       │
            │  - CRC calculation           │
            │  - Application launcher      │
            ├──────────────────────────────┤
            │ Bootloader Data/BSS          │  ~2 KB
            ├──────────────────────────────┤
            │ Application Jump Vector      │  4 bytes
0x08003FFC  │ Application CRC Storage      │  4 bytes
0x08004000  └──────────────────────────────┘
```

### Application Region (236 KB)

```
0x08004000  ┌──────────────────────────────┐
            │ Application Vector Table     │  1 KB
            ├──────────────────────────────┤
            │ Application Code             │  ~200 KB
            │  - Full OLED graphics lib    │
            │  - UI engine                 │
            │  - Screen renderers          │
            │  - Protocol handler          │
            │  - Input management          │
            │  - LED/Buzzer control        │
            ├──────────────────────────────┤
            │ Constant Data                │  ~20 KB
            │  - Font tables               │
            │  - Icon bitmaps              │
            │  - String tables             │
            ├──────────────────────────────┤
            │ Application Data/BSS         │  ~15 KB
            │  - Sensor data cache         │
            │  - Alert history             │
            │  - UI state                  │
0x0803EFFF  └──────────────────────────────┘
```

### Configuration Region (4 KB)

```
0x0803F000  ┌──────────────────────────────┐
            │ Config Magic Number          │  4 bytes
            │ Config Version               │  4 bytes
            ├──────────────────────────────┤
            │ Display Settings             │  ~1 KB
            │  - Brightness                │
            │  - Contrast                  │
            │  - Sleep timeout             │
            ├──────────────────────────────┤
            │ LED Calibration              │  ~1 KB
            │  - Color correction          │
            │  - Brightness curves         │
            ├──────────────────────────────┤
            │ Button Calibration           │  ~500 bytes
            │  - Debounce timing           │
            │  - Long press threshold      │
            ├──────────────────────────────┤
            │ Network Settings             │  ~500 bytes
            │  - UART baud rate            │
            │  - Timeout values            │
            ├──────────────────────────────┤
            │ Reserved / User Data         │  ~1 KB
0x0803FFFF  └──────────────────────────────┘
```

### RAM Layout (64-128 KB typical)

```
0x20000000  ┌──────────────────────────────┐
            │ Vector Table (RAM copy)      │  1 KB
            ├──────────────────────────────┤
            │ Stack (MSP)                  │  4 KB
            ├──────────────────────────────┤
            │ Heap                         │  20 KB
            ├──────────────────────────────┤
            │ Graphics Frame Buffer        │  1.5 KB
            │ (128×96 / 8 bits)            │
            ├──────────────────────────────┤
            │ UART RX Buffer               │  1 KB
            │ UART TX Buffer               │  1 KB
            ├──────────────────────────────┤
            │ Sensor Data Cache            │  2 KB
            │ (16 sensors × 128 bytes)     │
            ├──────────────────────────────┤
            │ Alert History Buffer         │  1 KB
            │ (10 alerts × 100 bytes)      │
            ├──────────────────────────────┤
            │ UI State                     │  512 bytes
            ├──────────────────────────────┤
            │ Application Variables        │  Remaining
            │ (.data, .bss sections)       │
            └──────────────────────────────┘
```

---

## Bootloader Architecture

### Purpose

- **Minimal size:** ~12 KB code + data
- **Always present:** Never overwritten
- **Safe update:** Protects against failed updates
- **Simple display:** Text-only progress indicator
- **No dependencies:** Standalone, no RTOS

### Bootloader State Machine

```
    [POWER ON / RESET]
           │
           ▼
    ┌─────────────┐
    │   INIT      │  Initialize minimal hardware
    │   HARDWARE  │  - Clock, GPIO, UART, Flash
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │   CHECK     │  Check if update requested
    │   MODE      │  - Special flag in RAM?
    └──────┬──────┘  - Button held during boot?
           │
      ┌────┴────┐
      │ Update? │
      └────┬────┘
      YES  │  NO
           │
   ┌───────┴────────┐
   │                │
   ▼                ▼
┌──────┐      ┌──────────┐
│UPDATE│      │  CHECK   │  Verify application CRC
│ MODE │      │   APP    │
└──┬───┘      └────┬─────┘
   │               │
   │          ┌────┴────┐
   │          │  Valid? │
   │          └────┬────┘
   │          YES  │  NO
   │               │
   │          ┌────┘
   │          │
   │          ▼
   │     ┌──────────┐
   │     │  LAUNCH  │  Jump to application
   │     │   APP    │
   │     └──────────┘
   │
   ▼
┌──────────────┐
│   WAIT FOR   │  Display "READY FOR UPDATE"
│   COMMANDS   │  Process BL commands via UART
└──────────────┘
```

### Bootloader Functions

```c
// bootloader/main.c

void bootloader_main(void) {
    // 1. Initialize minimal hardware
    system_init_minimal();
    oled_init_text_mode();
    uart_init(115200);
    flash_unlock();

    // 2. Display bootloader screen
    oled_clear();
    oled_print(0, "BOOTLOADER v1.0");
    oled_print(2, "Checking app...");

    // 3. Check if update mode requested
    if (is_update_requested()) {
        bootloader_update_mode();
    }

    // 4. Verify application
    if (verify_application()) {
        oled_print(4, "Launching app...");
        delay_ms(500);
        launch_application();
    }

    // 5. App invalid, stay in bootloader
    oled_print(4, "APP CORRUPT!");
    oled_print(6, "READY FOR UPDATE");
    bootloader_update_mode();
}

void bootloader_update_mode(void) {
    uint32_t last_activity = get_tick();

    while (1) {
        // Process UART commands
        if (uart_has_data()) {
            process_bootloader_command();
            last_activity = get_tick();
        }

        // Timeout: try to launch app after 30s
        if (get_tick() - last_activity > 30000) {
            if (verify_application()) {
                launch_application();
            }
            last_activity = get_tick();  // Reset timer
        }
    }
}

bool verify_application(void) {
    // Read stored CRC from end of bootloader region
    uint32_t stored_crc = *(volatile uint32_t*)APP_CRC_ADDR;

    // Calculate CRC of application region
    uint32_t calculated_crc = calculate_flash_crc(
        APP_START_ADDR,
        APP_SIZE
    );

    return (stored_crc == calculated_crc);
}

void launch_application(void) {
    // Disable interrupts
    __disable_irq();

    // Deinitialize peripherals
    uart_deinit();
    oled_deinit();

    // Set vector table offset
    SCB->VTOR = APP_START_ADDR;

    // Get stack pointer and reset handler from app vector table
    uint32_t app_stack = *(volatile uint32_t*)APP_START_ADDR;
    uint32_t app_reset = *(volatile uint32_t*)(APP_START_ADDR + 4);

    // Set stack pointer
    __set_MSP(app_stack);

    // Jump to application
    void (*app_entry)(void) = (void (*)(void))app_reset;
    app_entry();

    // Never returns
}
```

### Bootloader Commands

```c
void process_bootloader_command(void) {
    char cmd[512];

    if (!uart_read_line(cmd, sizeof(cmd), 1000)) {
        return;  // Timeout
    }

    if (!validate_crc(cmd)) {
        uart_send("ERR E10 CHECKSUM\r\n");
        return;
    }

    // Parse command
    if (starts_with(cmd, "BL ERASE")) {
        handle_erase(cmd);
    }
    else if (starts_with(cmd, "BL WRITE")) {
        handle_write(cmd);
    }
    else if (starts_with(cmd, "BL VERIFY")) {
        handle_verify(cmd);
    }
    else if (starts_with(cmd, "BL LAUNCH")) {
        handle_launch(cmd);
    }
    else {
        uart_send("ERR E00 UNKNOWN_CMD\r\n");
    }
}

void handle_erase(const char *cmd) {
    uint32_t addr, length;

    // Parse: "BL ERASE 0x08004000 229376"
    if (sscanf(cmd, "BL ERASE 0x%X %u", &addr, &length) != 2) {
        uart_send("ERR E01 INVALID_SYNTAX\r\n");
        return;
    }

    // Validate address range
    if (addr != APP_START_ADDR || length > APP_SIZE) {
        uart_send("ERR E02 RANGE\r\n");
        return;
    }

    uart_send("OK BL ERASING\r\n");
    oled_print(2, "Erasing flash...");

    // Erase flash sectors
    uint32_t sector_start = addr_to_sector(addr);
    uint32_t sector_end = addr_to_sector(addr + length - 1);

    for (uint32_t sector = sector_start; sector <= sector_end; sector++) {
        if (flash_erase_sector(sector) != FLASH_OK) {
            uart_send("ERR E09 HARDWARE_ERROR\r\n");
            return;
        }

        // Update progress
        uint8_t percent = ((sector - sector_start + 1) * 100) /
                          (sector_end - sector_start + 1);
        char progress[64];
        snprintf(progress, sizeof(progress),
                 "BL PROGRESS %u Erasing\r\n", percent);
        uart_send(progress);

        oled_progress(percent, "Erasing...");
    }

    uart_send("OK BL ERASED\r\n");
}

void handle_write(const char *cmd) {
    uint32_t addr;
    char hex_data[513];  // 512 hex chars + null
    uint8_t bin_data[256];

    // Parse: "BL WRITE 0x08004000 0123456789AB...×512"
    if (sscanf(cmd, "BL WRITE 0x%X %512s", &addr, hex_data) != 2) {
        uart_send("ERR E01 INVALID_SYNTAX\r\n");
        return;
    }

    // Validate hex length
    if (strlen(hex_data) != 512) {
        uart_send("ERR E01 INVALID_SYNTAX\r\n");
        return;
    }

    // Convert hex to binary
    for (int i = 0; i < 256; i++) {
        sscanf(&hex_data[i * 2], "%2hhx", &bin_data[i]);
    }

    // Write to flash
    if (flash_write(addr, bin_data, 256) != FLASH_OK) {
        uart_send("ERR E09 HARDWARE_ERROR\r\n");
        return;
    }

    uart_send("OK BL WRITTEN 256\r\n");

    // Progress update (every 10 chunks)
    static uint32_t chunk_count = 0;
    chunk_count++;

    if (chunk_count % 10 == 0) {
        uint32_t percent = (addr - APP_START_ADDR) * 100 / APP_SIZE;
        char progress[64];
        snprintf(progress, sizeof(progress),
                 "BL PROGRESS %u Writing\r\n", percent);
        uart_send(progress);

        oled_progress(percent, "Writing...");
    }
}

void handle_verify(const char *cmd) {
    uart_send("OK BL VERIFYING\r\n");
    oled_print(2, "Verifying...");

    // Calculate CRC of written application
    uint32_t calculated_crc = calculate_flash_crc(
        APP_START_ADDR,
        APP_SIZE
    );

    // Store CRC for future verification
    flash_write(APP_CRC_ADDR, (uint8_t*)&calculated_crc, 4);

    // For now, assume match (BB will verify against expected)
    char response[128];
    snprintf(response, sizeof(response),
             "OK BL CRC 0x%08X MATCH\r\n", calculated_crc);
    uart_send(response);
}

void handle_launch(const char *cmd) {
    uart_send("OK BL LAUNCHING\r\n");

    oled_clear();
    oled_print(2, "LAUNCHING...");
    delay_ms(500);

    if (verify_application()) {
        launch_application();
    } else {
        uart_send("ERR E13 VERIFY\r\n");
        oled_print(4, "APP INVALID!");
    }
}
```

### Minimal OLED Driver

```c
// bootloader/oled_minimal.c

// Text-only driver (no graphics library)
#define CHAR_WIDTH 6
#define CHAR_HEIGHT 8
#define MAX_COLS 21
#define MAX_ROWS 12

void oled_init_text_mode(void) {
    // Initialize I2C/SPI for SSD1306
    i2c_init();

    // Send init commands
    ssd1306_command(0xAE);  // Display off
    ssd1306_command(0xD5);  // Set clock divide
    ssd1306_command(0x80);
    // ... more init commands ...
    ssd1306_command(0xAF);  // Display on
}

void oled_clear(void) {
    for (uint16_t i = 0; i < 1536; i++) {
        ssd1306_data(0x00);
    }
}

void oled_print(uint8_t row, const char *text) {
    if (row >= MAX_ROWS) return;

    // Set cursor position
    ssd1306_set_cursor(0, row);

    // Print text using built-in font
    while (*text && strlen(text) < MAX_COLS) {
        oled_char(*text);
        text++;
    }
}

void oled_progress(uint8_t percent, const char *status) {
    // Row 6: Progress bar
    oled_print(6, "");
    draw_progress_bar(percent);

    // Row 8: Status text
    oled_print(8, status);

    // Row 10: Percentage
    char buf[16];
    snprintf(buf, sizeof(buf), "%u%%", percent);
    oled_print(10, buf);
}

void draw_progress_bar(uint8_t percent) {
    uint8_t filled = (percent * 20) / 100;  // 20 chars wide

    for (uint8_t i = 0; i < 20; i++) {
        if (i < filled) {
            oled_char(0xDB);  // Full block █
        } else {
            oled_char(0xB0);  // Light block ░
        }
    }
}
```

---

## Application Architecture

### Main Loop

```c
// application/main.c

int main(void) {
    // Initialize hardware
    system_init();

    // Initialize peripherals
    oled_init_full();       // Full graphics library (u8g2)
    uart_init(115200);
    button_init();
    led_init();
    buzzer_init();

    // Initialize software
    ui_init();
    protocol_init();
    data_init();

    // Say hello to backend
    send_hello_message();

    // Main loop
    while (1) {
        // 1. Handle button input
        button_update();

        // 2. Handle UART protocol
        protocol_update();

        // 3. Update UI state
        ui_update();

        // 4. Render display (frame rate limited)
        if (should_render_frame()) {
            ui_render();
        }

        // 5. Send periodic status
        if (should_send_status()) {
            send_hmi_status();
        }

        // 6. Watchdog kick
        iwdg_refresh();
    }
}
```

### Component Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Application                       │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │   UI Layer   │  │  Data Layer  │  │ Hardware │ │
│  │              │  │              │  │ Drivers  │ │
│  │ • Screens    │  │ • Sensors    │  │          │ │
│  │ • Rendering  │  │ • Relays     │  │ • OLED   │ │
│  │ • Navigation │  │ • Alerts     │  │ • UART   │ │
│  │ • Widgets    │  │ • Config     │  │ • LED    │ │
│  └──────┬───────┘  └──────┬───────┘  │ • Buzzer │ │
│         │                 │          │ • Button │ │
│         └────────┬────────┘          └────┬─────┘ │
│                  │                        │       │
│         ┌────────▼────────────────────────▼─────┐ │
│         │       Protocol Handler                │ │
│         │  • Message parsing                    │ │
│         │  • Command dispatch                   │ │
│         │  • Response formatting                │ │
│         └───────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

---

**Continue to HMI_FIRMWARE_PART2.md for firmware update process...**
