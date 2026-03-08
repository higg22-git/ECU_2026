# STM32duino Migration Report: Teensy 4.0 → STM32G474RE (Custom PCB)

**Generated:** 2026-03-08  
**Target MCU:** STM32G474RET6 (LQFP-64)  
**Target Framework:** STM32duino (Arduino for STM32) via PlatformIO  
**PCB Reference:** `ProjectAtlas.net` (KiCad netlist)  

---

## Table of Contents

1. [Overview](#overview)
2. [Hardware Differences Summary](#hardware-differences-summary)
3. [PCB Pin Mapping (from ProjectAtlas.net)](#pcb-pin-mapping-from-projectatlasnet)
4. [File-by-File Change Breakdown](#file-by-file-change-breakdown)
   - [platformio.ini](#platformioini)
   - [config/ecu.ini](#configecuini)
   - [config/dcf.ini](#configdcfini)
   - [config/ecu_old.ini](#configecu_oldini)
   - [lib/shared/CAN_message_t.hpp](#libsharedcan_message_thpp)
   - [src/ecu.cpp](#srcecu.cpp)
   - [src/dcf.cpp](#srcdcfcpp)
   - [src/ecu_old.cpp](#srcecu_oldcpp)
   - [lib/shared/util.hpp](#libsharedutilhpp)
5. [CAN Library Deep-Dive](#can-library-deep-dive)
6. [Serial / Debug Output](#serial--debug-output)
7. [Analog Inputs (ADC)](#analog-inputs-adc)
8. [GPIO / Digital I/O](#gpio--digital-io)
9. [Clock & Timing](#clock--timing)
10. [Build System](#build-system)
11. [Known Caveats & Risks](#known-caveats--risks)

---

## Overview

The current codebase targets a **Teensy 4.0** (NXP i.MX RT1062, ARM Cortex-M7) and uses the **FlexCAN_T4** Arduino library for CAN bus communication.  
The replacement MCU is an **STM32G474RET6** (ARM Cortex-M4 with FPU, up to 170 MHz) on a custom PCB named **ProjectAtlas**.  

The STM32G474RE has **three FDCAN (CAN-FD capable) peripherals** mapped to the PCB as follows:

| CAN Peripheral | RX Pin | TX Pin | Connected To |
|---|---|---|---|
| FDCAN1 | PA11 | PA12 | TJA1051 CAN transceiver (CAN1 connector) |
| FDCAN2 | PB12 | PB13 | TJA1051 CAN transceiver (CAN2 connector) |
| FDCAN3 | PA8  | PA15 | TJA1051 CAN transceiver (CAN3 connector) |

> **Netlist discrepancy note:** Net names `/CAN_3_RX_pb3` and `/CAN_3_TX_pb4` are misleading; the actual STM32 pins connected per the netlist are **PA8** (pin 42) and **PA15** (pin 51), which are valid FDCAN3 alternate-function pins on the STM32G474RE. Use PA8/PA15 in your code — not PB3/PB4.

---

## Hardware Differences Summary

| Feature | Teensy 4.0 | STM32G474RE (ProjectAtlas) |
|---|---|---|
| CPU | NXP i.MX RT1062, Cortex-M7, 600 MHz | STM32G474RET6, Cortex-M4F, 170 MHz |
| CAN peripheral | FlexCAN (up to 3 on Teensy 4.1; 1 on 4.0) | FDCAN1, FDCAN2, FDCAN3 (CAN-FD capable) |
| CAN library | `FlexCAN_T4` (third-party) | STM32duino built-in `STM32_CAN` or `ACAN_STM32` |
| USB-to-serial | Built-in USB–CDC on Teensy | CP2102N USB-to-UART bridge chip; USART1 on PA9/PA10 |
| LED (debug blink) | Pin 13 (built-in LED) | PC4 (LED 1), PC5 (LED 2), PB0 (LED 3) per PCB schematic |
| ADC resolution | 12-bit (0–4095) | 12-bit (0–4095) — same resolution |
| `INPUT_PULLDOWN` | Supported | **Supported** in STM32duino core |
| `Serial.printf()` | Supported | **Supported** in STM32duino core (routes to USART1 via CP2102N) |
| `millis()` | Supported | **Supported** |
| `delay()` | Supported | **Supported** |
| `setjmp`/`longjmp` | Supported | **Supported** (C standard library) |
| `std::optional` | Supported (C++17) | **Supported** — must keep `-std=gnu++20` flag |
| PlatformIO platform | `teensy` | `ststm32` |
| PlatformIO board | `teensy41` | `nucleo_g474re` (or custom board JSON) |

---

## PCB Pin Mapping (from ProjectAtlas.net)

The following table was extracted directly from `ProjectAtlas.net` by tracing each named net to its U1 (STM32G474RET6) pin. **Use these pin names verbatim in STM32duino code** — they are defined as preprocessor macros in the STM32duino variant headers (e.g., `PA11`, `PB12`, etc.).

| Net Name | STM32 Pin | Function / Notes |
|---|---|---|
| `/CAN_1_RX_pa11` | PA11 | FDCAN1 RX — Motor CAN |
| `/CAN_1_TX_pa12` | PA12 | FDCAN1 TX — Motor CAN |
| `/CAN_2_RX_pb12` | PB12 | FDCAN2 RX — Data CAN |
| `/CAN_2_TX_pb13` | PB13 | FDCAN2 TX — Data CAN |
| `/CAN_3_RX_pb3` *(net name wrong — see [Overview](#overview) note)* | **PA8** | FDCAN3 RX — Third CAN bus |
| `/CAN_3_TX_pb4` *(net name wrong — see [Overview](#overview) note)* | **PA15** | FDCAN3 TX — Third CAN bus |
| `/USB_TX_p9` | PA9 | USART1 TX → CP2102N RXD |
| `/USB_RX_p10` | PA10 | USART1 RX → CP2102N TXD |
| `/LED 1 Pin` | PC4 | LED 1 (debug/status) |
| `/LED 2 Pin` | PC5 | LED 2 (debug/status) |
| `/LED 3 Pin` | PB0 | LED 3 (debug/status) |
| `/5V Data 1` | PA1 | 5V-level sensor data (ADC1_IN2) |
| `/5V Data 2` | PA0 | 5V-level sensor data (ADC1_IN1) |
| `/5V I/O 1 STM32` | PA3 | 5V digital I/O via MOSFET (ADC1_IN4) |
| `/5V I/O 2 STM32` | PA4 | 5V digital I/O via MOSFET (ADC2_IN17 / DAC1_OUT1 — verify channel number against STM32G474RE datasheet Table 13, as ADC channel numbering on STM32G4 is non-sequential) |
| `/12V I/O 1 STM32` | PA7 | 12V digital I/O via MOSFET |
| `/12V I/O 2 STM32` | PA6 | 12V digital I/O via MOSFET |
| `/3V3 Data 1` | PC0 | 3.3V-level sensor data (ADC1_IN6 / ADC2_IN6) |
| `/3V3 Data 2` | PC2 | 3.3V-level sensor data (ADC1_IN8 / ADC2_IN8) |
| `/3V3V on/off Output 1` | PC1 | 3.3V power switch control |
| `/3V3V on/off Output 2` | PC3 | 3.3V power switch control |
| `/IO_extra_1` | PB5 | General-purpose I/O |
| `/IO_extra_2` | PB6 | General-purpose I/O |
| `/IO_extra_3` | PB4 | General-purpose I/O |
| `/IO_extra_4` | PB7 | General-purpose I/O |
| `/SWDIO` | PA13 | SWD debug — do **not** remap |
| `/SWCLK` | PA14 | SWD debug — do **not** remap |
| `/SWO` | PB3 | SWD trace output |
| `/NRST` | PG10-NRST | Reset pin |
| `/Clock_in` | PF0-OSC_IN | External crystal in |
| `/Clock_out` | PF1-OSC_OUT | External crystal out |
| `/USB_D+` / `/USB_D-` | — | Connected to CP2102N, not direct to STM32 |

---

## File-by-File Change Breakdown

---

### `platformio.ini`

**Location:** `/platformio.ini`  
**Status:** MUST CHANGE

The top-level `platformio.ini` includes environment configs from `config/`. The only change needed here is to update `default_envs`:

```ini
; BEFORE:
default_envs = ecu_debug

; AFTER (when switching to STM32):
default_envs = ecu_stm32_debug
```

The actual platform/board configuration lives in the environment-specific `.ini` files described below.

---

### `config/ecu.ini`

**Location:** `/config/ecu.ini`  
**Status:** MUST CHANGE — highest priority

This is the primary build configuration for the ECU firmware.

#### Current content (Teensy):
```ini
[ecu_base]
extends = common
platform = teensy
board = teensy41
framework = arduino
build_flags = ${common.build_flags}
    -isystem .pio/libdeps/ecu_debug/FlexCAN_T4
    -Wno-variadic-macros
    -Wno-gnu-zero-variadic-macro-arguments
build_src_flags = ${common.build_src_flags} -DLOG_FILE_PATH="logs/log_${date}_${time}.log"
build_src_filter = "+<ecu.cpp>"
monitor_filters = time, log2file
lib_deps = https://github.com/tonton81/FlexCAN_T4

[env:ecu_debug]
extends = ecu_base
build_type = debug
build_flags = ${ecu_base.build_flags} -DENABLE_DEBUGGING
```

#### Required changes for STM32duino:
```ini
[ecu_stm32_base]
extends = common
platform = ststm32
; Use the Nucleo-G474RE board definition as the closest match.
; If you have a fully custom board, you will need to provide a custom
; board JSON file (see PlatformIO docs: https://docs.platformio.org/en/latest/platforms/creating_board.html)
board = nucleo_g474re
framework = arduino
build_flags = ${common.build_flags}
    -Wno-variadic-macros
    -Wno-gnu-zero-variadic-macro-arguments
    ; Tell STM32duino to use USART1 (PA9/PA10) as the Serial port.
    ; If using a custom board, you may need to specify this in the board JSON instead.
    -DHAL_CAN_MODULE_ENABLED
    -DHAL_FDCAN_MODULE_ENABLED
build_src_flags = ${common.build_src_flags} -DLOG_FILE_PATH="logs/log_${date}_${time}.log"
build_src_filter = "+<ecu.cpp>"
monitor_filters = time, log2file
; Replace FlexCAN_T4 with an STM32-compatible CAN library.
; Option A (recommended): Use the stm32duino community STM32_CAN library.
lib_deps = stm32duino/STM32_CAN
; Option B: Use the ACAN_STM32 library (supports FDCAN/CAN-FD).
; lib_deps = pierremolinaro/acan-stm32@^2.1.8
; Option C: Use the built-in FDCAN HAL wrapper if no library is available.
; (Requires writing HAL-level CAN code directly; not recommended.)

[env:ecu_stm32_debug]
extends = ecu_stm32_base
build_type = debug
build_flags = ${ecu_stm32_base.build_flags} -DENABLE_DEBUGGING
```

**Why these changes:**
- `platform = ststm32` is the PlatformIO identifier for the ST STM32 platform (which includes STM32duino).
- `board = nucleo_g474re` is the closest off-the-shelf PlatformIO board for the STM32G474RE. If your custom PCB has a different clock source, oscillator frequency, or flash layout than the Nucleo, you **must** supply a custom board JSON.
- The `-isystem` flag for FlexCAN_T4 is removed — it is irrelevant on STM32.
- `HAL_FDCAN_MODULE_ENABLED` enables the FDCAN HAL in the STM32 HAL layer (required for the CAN library to function).
- The `FlexCAN_T4` lib_dep is replaced with an STM32-compatible CAN library.

---

### `config/dcf.ini`

**Location:** `/config/dcf.ini`  
**Status:** MUST CHANGE

#### Current content:
```ini
[env:dcf]
extends = common
platform = teensy
board = teensy41
framework = arduino
build_src_filter = "+<dcf.cpp>"
lib_deps =
    https://github.com/tonton81/FlexCAN_T4
    https://github.com/adafruit/Adafruit_BNO08x_RVC
    https://github.com/adafruit/Adafruit_BNO08x
    adafruit/Adafruit GPS Library@^1.7.4
    mikalhart/TinyGPSPlus@^1.0.3
```

#### Required changes:
```ini
[env:dcf]
extends = common
platform = ststm32
board = nucleo_g474re
framework = arduino
build_flags = ${common.build_flags}
    -DHAL_FDCAN_MODULE_ENABLED
build_src_filter = "+<dcf.cpp>"
lib_deps =
    stm32duino/STM32_CAN
    ; The BNO08x RVC and GPS libraries are pure Arduino and should be
    ; compatible with STM32duino as long as you adjust which hardware
    ; Serial port they connect to (see dcf.cpp notes below).
    https://github.com/adafruit/Adafruit_BNO08x_RVC
    https://github.com/adafruit/Adafruit_BNO08x
    adafruit/Adafruit GPS Library@^1.7.4
    mikalhart/TinyGPSPlus@^1.0.3
```

---

### `config/ecu_old.ini`

**Location:** `/config/ecu_old.ini`  
**Status:** MUST CHANGE (same pattern as ecu.ini)

Apply the same `platform`, `board`, `lib_deps`, and `build_flags` changes as described for `config/ecu.ini` above. Replace:
- `platform = teensy` → `platform = ststm32`
- `board = teensy41` → `board = nucleo_g474re`
- `lib_deps = https://github.com/tonton81/FlexCAN_T4` → `lib_deps = stm32duino/STM32_CAN`
- Add `-DHAL_FDCAN_MODULE_ENABLED` to `build_flags`
- Remove the `-isystem .pio/libdeps/.../FlexCAN_T4` flag

---

### `lib/shared/CAN_message_t.hpp`

**Location:** `/lib/shared/CAN_message_t.hpp`  
**Status:** MUST CHANGE — this is the most impactful structural change

#### Current content (simplified):
```cpp
#ifdef BUILD_MODE_TEST
typedef struct CAN_message_t {
  uint32_t id = 0;
  uint16_t timestamp = 0;
  uint8_t idhit = 0;
  struct { bool extended, remote, overrun, reserved; } flags;
  uint8_t len = 8;
  uint8_t buf[8] = { 0 };
  int8_t mb = 0;
  uint8_t bus = 0;
  bool seq = 0;
} CAN_message_t;
#else
#include "FlexCAN_T4.h"
#endif
```

The `#else` branch currently pulls in `FlexCAN_T4.h` to get the real `CAN_message_t` definition. On STM32duino there is no `FlexCAN_T4.h`, so this will fail to compile.

#### Required change:

You need to add a third compilation branch for the STM32 target. The approach depends on which CAN library you choose:

**Option A — `stm32duino/STM32_CAN` library (classic CAN, recommended starting point):**

The `STM32_CAN` library uses its own `CAN_message_t` that is identical in layout to the `FlexCAN_T4` one. In that case, replace:

```cpp
#else
#include "FlexCAN_T4.h"
#endif
```

with:

```cpp
#elif defined(ARDUINO_ARCH_STM32)
// STM32duino: use the STM32_CAN library's message struct.
// STM32_CAN defines its own CAN_message_t which is compatible with
// the struct defined in the BUILD_MODE_TEST block above.
#include "STM32_CAN.h"
#else
#include "FlexCAN_T4.h"
#endif
```

**Option B — `ACAN_STM32` library (FDCAN/CAN-FD capable):**

The ACAN_STM32 library uses `ACAN_STM32_Message` instead of `CAN_message_t`. If you use this library, you would need to:
1. Rename `CAN_message_t` throughout the codebase to a project-specific name (e.g., `ECU_CAN_message_t`).
2. Provide a typedef or adapter in `CAN_message_t.hpp` that maps between `ACAN_STM32_Message` and the internal struct.

**This is the more invasive option and is NOT recommended unless you need CAN-FD frames.**

#### Summary of the change:
The critical `#else` block that includes `FlexCAN_T4.h` must gain an STM32-specific branch. The `ARDUINO_ARCH_STM32` macro is automatically defined by the STM32duino core for all STM32 targets.

---

### `src/ecu.cpp`

**Location:** `/src/ecu.cpp`  
**Status:** MUST CHANGE

#### 1. Include directives

```cpp
// BEFORE:
#include "Arduino.h"
#include "FlexCAN_T4.h"

// AFTER:
#include "Arduino.h"
#ifdef ARDUINO_ARCH_STM32
#include "STM32_CAN.h"   // or your chosen CAN library header
#endif
```

#### 2. CAN object declaration

```cpp
// BEFORE (FlexCAN_T4):
FlexCAN_T4<CAN1, RX_SIZE_256, TX_SIZE_16> MotorCAN;

// AFTER (STM32_CAN library, FDCAN1 on PA11/PA12):
STM32_CAN MotorCAN(FDCAN1, DEF, RX_SIZE_256, TX_SIZE_16);
// DEF = default pin mapping. FDCAN1's default RX/TX are PA11/PA12,
// which matches the ProjectAtlas PCB netlist.
```

> **Important:** Verify that the `STM32_CAN` library's default pin mapping for FDCAN1 is indeed PA11/PA12. If not, the library may require explicit pin specification — check the library's README.

#### 3. `setup()` — CAN initialization

```cpp
// BEFORE:
MotorCAN.begin();
MotorCAN.setBaudRate(CAN_BAUD_RATE);

// AFTER:
MotorCAN.begin();
MotorCAN.setBaudRate(CAN_BAUD_RATE);
// Both FlexCAN_T4 and STM32_CAN use this same API. No change needed
// IF you use the STM32_CAN library.
// If you use the raw HAL or a different library, the API differs.
```

#### 4. `setup()` — Serial initialization

```cpp
// BEFORE:
Serial.begin(SERIAL_BAUD_RATE);

// AFTER:
Serial.begin(SERIAL_BAUD_RATE);
// No change needed. In STM32duino, Serial maps to USART1 (PA9/PA10)
// by default when the board variant configures it that way.
// HOWEVER: for the nucleo_g474re board variant, `Serial` may map to
// the ST-Link UART, NOT USART1. You may need to declare:
//   HardwareSerial Serial1(PA10, PA9);  // RX=PA10, TX=PA9
// and use Serial1 throughout, or configure the board variant to
// redirect `Serial` to USART1.
// See "Serial / Debug Output" section below for full details.
```

#### 5. LED pin number

```cpp
// BEFORE — Teensy built-in LED:
pinMode(13, OUTPUT);
digitalWrite(13, HIGH);
// ...
digitalWrite(13, LOW);

// AFTER — ProjectAtlas PCB has three LEDs. Use LED 1 (PC4):
#ifdef ARDUINO_ARCH_STM32
  constexpr uint8_t DEBUG_LED_PIN = PC4;  // LED 1 on ProjectAtlas PCB
#else
  constexpr uint8_t DEBUG_LED_PIN = 13;   // Teensy built-in LED
#endif
// Then replace all occurrences of literal `13` with `DEBUG_LED_PIN`.
```

The magic number `13` appears in `safety_assert_failed_handler()`. Define a constant and replace all uses.

#### 6. `Serial.printf()` calls

```cpp
// These calls appear throughout ecu.cpp:
Serial.printf("Safety assertion failed! In file %s:%d ...", file, line, error_code);

// STM32duino's HardwareSerial does support Serial.printf() via the
// STM32duino core. No change required here IF Serial is correctly
// mapped to USART1. See the "Serial / Debug Output" section.
```

#### 7. CAN read/write API

```cpp
// BEFORE (FlexCAN_T4):
CAN_message_t rmsg;
if (MotorCAN.read(rmsg)) { ... }
MotorCAN.write(shutdown_message);

// AFTER (STM32_CAN — same API):
CAN_message_t rmsg;
if (MotorCAN.read(rmsg)) { ... }
MotorCAN.write(shutdown_message);
// The STM32_CAN library intentionally mirrors the FlexCAN_T4 API.
// No changes required here IF using STM32_CAN.
```

---

### `src/dcf.cpp`

**Location:** `/src/dcf.cpp`  
**Status:** MUST CHANGE

#### 1. Include directives

```cpp
// BEFORE:
#include "Arduino.h"
#include "FlexCAN_T4.h"

// AFTER:
#include "Arduino.h"
#ifdef ARDUINO_ARCH_STM32
#include "STM32_CAN.h"
#endif
```

#### 2. CAN object declarations

```cpp
// BEFORE (FlexCAN_T4):
FlexCAN_T4<CAN1, RX_SIZE_256> motorCAN;
FlexCAN_T4<CAN2, RX_SIZE_256> dataCAN;

// AFTER (STM32_CAN):
// From ProjectAtlas.net:
//   Motor CAN → FDCAN1 (PA11 RX, PA12 TX)
//   Data CAN  → FDCAN2 (PB12 RX, PB13 TX)
STM32_CAN motorCAN(FDCAN1, DEF);
STM32_CAN dataCAN(FDCAN2, DEF_2);
// Note: verify that "DEF" maps to PA11/PA12 and "DEF_2" maps to
// PB12/PB13 for the STM32_CAN library. Consult that library's
// documentation for the correct DEF pin-set constants.
```

#### 3. Pin number constants — THIS IS A CRITICAL CHANGE

The current code uses **Teensy pin numbers** that have no meaning on STM32:

```cpp
// BEFORE (Teensy pin numbers — meaningless on STM32):
constexpr uint8_t THROTTLE_1_PIN = 18;
constexpr uint8_t THROTTLE_2_PIN = 19;
constexpr uint8_t BRAKE_PIN      = 20;
constexpr uint8_t SWITCH_PIN     = 21;
```

These must be mapped to **STM32 pin names** based on your actual sensor wiring on the ProjectAtlas PCB. The netlist provides these available analog/digital signal lines connected to the STM32G474RE:

| Net Name | STM32 Pin | Suitable For |
|---|---|---|
| `/5V Data 1` | PA1 | Analog sensor (throttle, brake) — ADC1_IN2 |
| `/5V Data 2` | PA0 | Analog sensor (throttle, brake) — ADC1_IN1 |
| `/3V3 Data 1` | PC0 | Analog sensor — ADC1_IN6 / ADC2_IN6 |
| `/3V3 Data 2` | PC2 | Analog sensor — ADC1_IN8 / ADC2_IN8 |
| `/5V I/O 1 STM32` | PA3 | 5V-level digital I/O (MOSFET-switched) |
| `/5V I/O 2 STM32` | PA4 | 5V-level digital I/O (MOSFET-switched) |
| `/12V I/O 1 STM32` | PA7 | 12V-level digital I/O (MOSFET-switched) |
| `/12V I/O 2 STM32` | PA6 | 12V-level digital I/O (MOSFET-switched) |

You must determine which PCB net corresponds to each sensor connector and update accordingly. A placeholder mapping (based on net names suggesting analog sensor connections) is:

```cpp
// AFTER (STM32 pin names — update to match actual sensor wiring):
// Throttle sensors are typically ratiometric 0–5V, so use 5V data lines.
constexpr uint32_t THROTTLE_1_PIN = PA1;   // /5V Data 1 — ADC1_IN2
constexpr uint32_t THROTTLE_2_PIN = PA0;   // /5V Data 2 — ADC1_IN1
// Brake pressure sensor — adjust to actual wiring:
constexpr uint32_t BRAKE_PIN      = PC0;   // /3V3 Data 1 — ADC1_IN6
// Start switch — digital input. PA3 is 5V I/O (MOSFET level-shifted).
constexpr uint32_t SWITCH_PIN     = PA3;   // /5V I/O 1 STM32
```

**You MUST verify these assignments against the actual schematic/connector labels.** The netlist names are functional labels; the actual throttle/brake/switch sensor connections depend on which PCB connector each sensor plugs into.

#### 4. `INPUT_PULLDOWN` compatibility

```cpp
// BEFORE and AFTER — no change needed:
pinMode(THROTTLE_1_PIN, INPUT_PULLDOWN);
// STM32duino DOES support INPUT_PULLDOWN for digital pins.
// However, PA3 and PA4 (/5V I/O x) have MOSFET circuitry that acts
// as a level translator. The internal STM32 pulldown may interfere
// with the external circuit — check the schematic carefully.
```

#### 5. CAN initialization

```cpp
// BEFORE:
motorCAN.begin();
motorCAN.setBaudRate(CAN_BAUD_RATE);
dataCAN.begin();
dataCAN.setBaudRate(CAN_BAUD_RATE);

// AFTER (STM32_CAN — same API):
motorCAN.begin();
motorCAN.setBaudRate(CAN_BAUD_RATE);
dataCAN.begin();
dataCAN.setBaudRate(CAN_BAUD_RATE);
// No change needed if using STM32_CAN.
```

#### 6. ADC resolution note

```cpp
// The STM32G474RE ADC is 12-bit by default (0–4095), same as Teensy.
// analogRead() returns values in this range.
// If the THROTTLE constants in ecu_old.cpp (THROTTLE1_MAX = 160 etc.)
// were calibrated against Teensy ADC readings, they will need to be
// re-calibrated against actual sensor readings on the STM32 hardware.
// The STM32duino analogReadResolution() function can set 8, 10, or 12 bits.
```

---

### `src/ecu_old.cpp`

**Location:** `/src/ecu_old.cpp`  
**Status:** MUST CHANGE (same patterns as ecu.cpp)

#### 1. Include directives

```cpp
// BEFORE:
#include <Arduino.h>
#include <FlexCAN_T4.h>

// AFTER:
#include <Arduino.h>
#ifdef ARDUINO_ARCH_STM32
#include <STM32_CAN.h>
#endif
```

#### 2. CAN object declaration

```cpp
// BEFORE:
FlexCAN_T4<CAN1, RX_SIZE_256, TX_SIZE_16> MotorCAN;

// AFTER:
STM32_CAN MotorCAN(FDCAN1, DEF, RX_SIZE_256, TX_SIZE_16);
```

#### 3. CAN read/write

```cpp
// The STM32_CAN library mirrors the FlexCAN_T4 CAN read/write API:
//   MotorCAN.read(rmsg)  → no change
//   MotorCAN.write(rmsg) → no change
// The `rmsg` variable uses CAN_message_t which is handled by
// CAN_message_t.hpp (see that section).
```

#### 4. `send_message()` — uses `rmsg` for both TX and RX

The `send_message()` function in `ecu_old.cpp` reuses the global `rmsg` variable for both incoming CAN messages AND as a transmit buffer. This is already bad practice and may cause subtle bugs. On STM32, the same struct is used for both Rx and Tx with `STM32_CAN`, so functionally it will still compile and run — but consider refactoring to use a dedicated transmit buffer variable.

#### 5. `map()` function name collision

`ecu_old.cpp` defines its own `int map(int input, int min, int max)`. The Arduino framework (on both Teensy and STM32duino) provides a built-in `map()` function with a **different signature**: `long map(long x, long in_min, long in_max, long out_min, long out_max)`. This is already a potential issue on Teensy; it will remain an issue on STM32duino. The local `map()` in `ecu_old.cpp` has 3 parameters vs the Arduino `map()` with 5. They are overloaded (C++ resolves by parameter count), so no compilation error, but be aware of the naming ambiguity.

---

### `lib/shared/util.hpp`

**Location:** `/lib/shared/util.hpp`  
**Status:** MAY NEED CHANGE (see Serial note)

#### Current content:
```cpp
#ifdef BUILD_MODE_TEST
#include <cstdio>
#define PRINTF(...) printf(__VA_ARGS__)
#else
#include "Arduino.h"
#define PRINTF(...) Serial.printf(__VA_ARGS__)
#endif
```

#### Analysis:

`Serial.printf()` **is supported** in STM32duino (it is part of the STM32duino `HardwareSerial` class, unlike standard Arduino where `printf` is not available on `Serial`). So this macro will compile without errors.

**However**, there is one potential issue: the default `Serial` object in the STM32duino `nucleo_g474re` board variant may be mapped to the **ST-Link virtual COM port** (which uses a UART connected to the on-board debugger), NOT to USART1 (PA9/PA10). On the custom ProjectAtlas PCB, there is no ST-Link — the USB-to-UART is via the CP2102N chip connected to USART1 (PA9/PA10).

**Recommended fix:** Add a `HardwareSerial` declaration to redirect Serial to the correct UART:

```cpp
// In util.hpp or in the top of ecu.cpp, before any Serial use:
#ifdef ARDUINO_ARCH_STM32
// On ProjectAtlas PCB, Serial is USART1 on PA9 (TX) / PA10 (RX),
// connected to the CP2102N USB-to-UART bridge.
// STM32duino 'nucleo_g474re' may default Serial to a different UART.
// Uncomment and adjust if needed:
// HardwareSerial Serial(PA10, PA9);  // RX=PA10, TX=PA9
#endif
```

If your custom board JSON correctly specifies USART1 as the default Serial, no change is needed. Otherwise, this must be addressed.

No other changes are required in `util.hpp` — `millis()`, `Trigger`, `Timer`, `str_hash()`, `std::optional` are all portable.

---

## CAN Library Deep-Dive

### Why FlexCAN_T4 cannot be used on STM32

`FlexCAN_T4` is hardcoded for the NXP i.MX RT series and Teensy hardware. It accesses NXP-specific CAN controller registers directly. It will not compile on STM32.

### Recommended Replacement: STM32_CAN (classic CAN, mimics FlexCAN_T4 API)

There are two separate libraries that carry a similar name — be sure to use the correct one:

| | Community library (nopnop2002) | Official STM32duino library |
|---|---|---|
| **GitHub** | https://github.com/nopnop2002/Arduino-STM32-CAN | https://github.com/stm32duino/STM32-CAN |
| **PlatformIO lib_deps** | GitHub URL above, or search "STM32_CAN" | `stm32duino/STM32_CAN` |
| **CAN-FD** | No (classic CAN only) | No (classic CAN compatibility mode) |
| **API similarity to FlexCAN_T4** | High — same `CAN_message_t`, `.begin()`, `.read()`, `.write()` | Similar; verify struct layout before using |

> **Recommendation:** Prefer the **official `stm32duino/STM32_CAN`** library (PlatformIO registry entry `stm32duino/STM32_CAN`). This is maintained by the STM32duino core team and is more likely to track STM32G4 FDCAN support. If it is unavailable in the PlatformIO registry at the time of migration, use the GitHub URL directly. Verify the `CAN_message_t` struct layout matches the test-build struct in `lib/shared/CAN_message_t.hpp` before finalising.

**Key advantage of either STM32_CAN library:** Both intentionally mimic the `FlexCAN_T4` API — same `CAN_message_t` struct, same `.begin()`, `.setBaudRate()`, `.read()`, `.write()` calls.  
**FDCAN peripherals:** The STM32G474RE's FDCAN peripherals operate in **classic CAN compatibility mode** at 250 kbps, which is exactly what this project needs (`CAN_BAUD_RATE = 250000`).

### Alternative: `ACAN_STM32` (for CAN-FD)

If you ever need CAN-FD (longer frames, higher data-phase bit rates), use:
- **PlatformIO lib_deps:** `pierremolinaro/acan-stm32`
- This library uses a **different message struct** (`ACAN_STM32_Message`) and a **different API**. Choosing this option requires more invasive code changes throughout the project.

### FDCAN peripheral-to-pin mapping on STM32G474RE

The STM32G474RE supports multiple alternate function pin sets for each FDCAN peripheral. The ProjectAtlas PCB uses:

| FDCAN Peripheral | RX Pin | TX Pin | STM32 Alternate Function |
|---|---|---|---|
| FDCAN1 | PA11 | PA12 | AF9 |
| FDCAN2 | PB12 | PB13 | AF9 |
| FDCAN3 | PA8  | PA15 | AF11 |

These are valid alternate functions per the STM32G474RE datasheet. When using the `STM32_CAN` library, ensure that the library's pin configuration matches these alternate functions. You may need to pass explicit pin arguments if the library's default pin map is different.

### CAN message format compatibility

The project currently uses **standard 11-bit CAN IDs** (e.g., `MessageId::ControlCommand = 192`, well within the 11-bit range of 0–2047). CAN-FD extended frames are not used. Classic CAN mode at 250 kbps is fully compatible.

---

## Serial / Debug Output

### Teensy behaviour
On Teensy, `Serial` is a **USB CDC** virtual serial port. No UART hardware is used. `Serial.printf()` is a Teensy extension to the Arduino Serial class.

### STM32duino behaviour on ProjectAtlas PCB
On the ProjectAtlas PCB:
- There is **no direct USB connection to the STM32** for CDC. The USB connector is connected to a **CP2102N USB-to-UART bridge**, which connects via UART to **USART1 (PA9 TX, PA10 RX)** on the STM32.
- `Serial.printf()` is supported in STM32duino. The `HardwareSerial` class in the STM32duino core includes a `printf()` method.
- You must ensure that `Serial` in your code maps to **USART1** (PA9/PA10). With the `nucleo_g474re` board variant, the default `Serial` may be mapped to the ST-Link UART (USART3 on the Nucleo board). On your custom PCB (no ST-Link), you must override this.

**Recommended approach:**  
In `ecu.cpp` (or a shared header), before `Serial.begin()`:
```cpp
#ifdef ARDUINO_ARCH_STM32
// Remap Serial to USART1 on ProjectAtlas PCB (CP2102N bridge on PA9/PA10)
HardwareSerial Serial(PA10, PA9);  // Constructor: (RX_pin, TX_pin)
#endif
```
Or, define a **custom board JSON** for your PCB where the UART mapping is set correctly, so that the default `Serial` object already points to USART1.

---

## Analog Inputs (ADC)

### Resolution
Both Teensy 4.0 and STM32G474RE provide **12-bit ADC** (0–4095). `analogRead()` behaviour is the same.

### Reference voltage
- **Teensy 4.0:** 3.3V ADC reference (built-in)
- **STM32G474RE (ProjectAtlas):** The PCB connects pin 28 (`VREF+`) to the `/Vdda` net. `VDDA` is the analog supply. The PCB appears to use 3.3V for VDDA. The `analogReference()` function can set the reference voltage in STM32duino if needed.

### 5V-tolerant ADC inputs
The STM32G474RE **analog pins are NOT 5V tolerant** (maximum input voltage is VDDA + 0.3V ≈ 3.6V for analog functions). The PCB uses MOSFET-based level translators on the `/5V I/O 1` and `/5V I/O 2` lines (PA3, PA4) and voltage dividers or protection circuits for the `/5V Data` lines. Verify the divider ratios and update `THROTTLE1_HIGH`, `THROTTLE1_LOW`, etc. in `constants.hpp` to match the actual ADC readings on the STM32 hardware.

---

## GPIO / Digital I/O

### `INPUT_PULLDOWN`
STM32duino **fully supports** `INPUT_PULLDOWN`. No change needed.

### `INPUT_PULLUP`
Supported. No change needed.

### `digitalRead()` / `digitalWrite()` / `pinMode()`
All work identically in STM32duino. No API changes needed.

### Pin naming
In STM32duino, GPIO pins are referenced as `PA0`, `PA1`, `PB12`, `PC4`, etc. (preprocessor constants defined in the STM32duino variant headers). **Do not use bare integer pin numbers from Teensy** (e.g., `18`, `19`, `20`, `21`) — they will map to wrong or undefined pins on the STM32G474RE.

---

## Clock & Timing

### `millis()` and `delay()`
Both work identically in STM32duino. No changes needed anywhere in the code.

### `setjmp` / `longjmp`
Used in `ecu.cpp` for soft-reset handling (`#ifdef ENABLE_DEBUGGING`). The C standard library on STM32 (newlib) supports `setjmp`/`longjmp`. No changes needed. The `#include <setjmp.h>` already present in `ecu.cpp` is correct.

### System clock speed
- **Teensy 4.0:** 600 MHz (Cortex-M7)
- **STM32G474RE:** Up to 170 MHz (Cortex-M4F)

This speed difference does not affect correctness since the code uses `millis()`-based timing throughout and does not use busy-wait loops calibrated to clock speed. However, if any tight timing loops are ever added, note this difference.

### External crystal
The ProjectAtlas PCB connects an external crystal to `PF0-OSC_IN` / `PF1-OSC_OUT` on the STM32. The custom board JSON or STM32duino startup code must be configured to use this external clock source (`HSE`) if it differs from the default internal oscillator (`HSI`) used by the `nucleo_g474re` variant. Incorrect clock configuration will cause `millis()` and `Serial` baud rates to be inaccurate.

---

## Build System

### Summary of all `platformio.ini` / config file changes

| Change | Old Value | New Value |
|---|---|---|
| `platform` | `teensy` | `ststm32` |
| `board` | `teensy41` | `nucleo_g474re` (or custom board JSON) |
| `lib_deps` CAN | `https://github.com/tonton81/FlexCAN_T4` | `stm32duino/STM32_CAN` |
| `build_flags` CAN system include | `-isystem .pio/libdeps/ecu_debug/FlexCAN_T4` | Remove entirely |
| `build_flags` HAL modules | *(not needed on Teensy)* | `-DHAL_FDCAN_MODULE_ENABLED` |
| C++ standard | `-std=gnu++20` | Keep `-std=gnu++20` |
| `-Wno-variadic-macros` | Present | Keep (still needed for macro usage) |

### Custom board JSON
If the ProjectAtlas PCB differs from the Nucleo-G474RE in any of the following ways, you **must** create a custom board JSON file:
- Different external crystal frequency
- Different flash or RAM size (unlikely — the G474RE has 512 KB flash / 128 KB RAM)
- Different default UART for `Serial`
- Different `upload_protocol` (since there's no ST-Link on the custom PCB, you'll use DFU or SWD via an external debugger)

---

## Known Caveats & Risks

### 1. CAN-FD vs Classic CAN
The STM32G474RE FDCAN peripherals support CAN-FD. The project uses classic CAN at 250 kbps. The `STM32_CAN` library operates in classic CAN compatibility mode, which is correct for this application. **Do not enable CAN-FD bit rates in the library without verifying that all CAN nodes (motor controller, etc.) also support CAN-FD.**

### 2. Serial mapping must be verified
As described above, the default `Serial` in the `nucleo_g474re` PlatformIO board variant may not map to USART1 (PA9/PA10). If this is not corrected, all `Serial.printf()` / `Serial.println()` / `Serial.begin()` calls will silently do nothing (output to the wrong UART that is not connected to the CP2102N bridge). Verify this before assuming debug output is working.

### 3. ADC calibration required
The Teensy throttle/brake ADC constants (`THROTTLE1_LOW = 255`, `THROTTLE1_HIGH = 35`, etc. in `constants.hpp`) were calibrated against actual sensor readings on the Teensy hardware. After porting, these values must be re-measured on the STM32 hardware since:
- The input voltage divider circuits on the PCB will produce different ADC counts than direct sensor wiring on Teensy.
- The STM32's ADC reference may differ slightly from Teensy's.

### 4. Upload / programming method
The Teensy uses a built-in USB bootloader. The STM32G474RE on the custom PCB has no built-in programmer. You will need to use one of:
- **ST-Link external debugger** (SWDIO=PA13, SWCLK=PA14, SWO=PB3) — most reliable
- **STM32 DFU bootloader** over USB (requires BOOT0 pin = PB8 on ProjectAtlas to be pulled high during reset)
- **UART bootloader** over USART1 (PA9/PA10) — available on all STM32 devices as a factory option

Update `platformio.ini` with the correct `upload_protocol` for your chosen method:
```ini
; For ST-Link:
upload_protocol = stlink
; For DFU:
upload_protocol = dfu
; For UART (serial) bootloader:
upload_protocol = serial
```

### 5. `FlexCAN_T4` template parameter `CAN1` / `CAN2`
These are C preprocessor macros defined by `FlexCAN_T4.h` that refer to NXP-specific CAN peripheral base addresses. They do **not** exist in STM32duino. The STM32_CAN library uses its own identifiers (`FDCAN1`, `FDCAN2`, `FDCAN3`) which are defined in the STM32 HAL headers. Do not attempt to use `CAN1`/`CAN2` on STM32 — always use `FDCAN1`/`FDCAN2`/`FDCAN3`.

### 6. `Serial.printf` format specifier portability
`ecu.cpp` uses `%lu` for `unsigned long` values:
```cpp
Serial.printf("File hash %lu\n", (unsigned long) str_hash(file));
```
On STM32 (ARM 32-bit), `unsigned long` is 32 bits — same as Teensy. This is safe as written.

### 7. `std::optional` and C++ standard library headers
The code uses `#include <optional>` and `#include <cmath>`. These are from the C++17 standard library. The STM32duino core uses **newlib** as the C runtime. Newlib supports C++17 when compiled with `-std=gnu++20` (as already set in `config/common.ini`). This should work without changes, but ensure that `libstdc++` is linked — PlatformIO does this automatically for the `arduino` framework.

### 8. `Adafruit_BNO08x` / GPS libraries in `dcf.cpp`
These are pure Arduino libraries with no Teensy-specific code. They should work on STM32duino, but you must check which hardware Serial or I²C port they are configured to use and ensure those map to the correct STM32 pins on the ProjectAtlas PCB.

---

## Files That Do NOT Need Changes

The following files are hardware-agnostic and require **no modifications** for the STM32 port:

| File | Reason |
|---|---|
| `lib/ecu/ecu_logic.cpp` | Pure C++ logic, no hardware dependencies |
| `lib/ecu/ecu_logic.hpp` | Header only, no hardware includes |
| `lib/ecu/constants.hpp` | Constants only (ADC values may need re-calibration but no code change) |
| `lib/shared/assert.cpp` | Pure C++, no hardware dependencies |
| `lib/shared/assert.hpp` | Header only |
| `lib/shared/can_serde.cpp` | Operates on `CAN_message_t` structs, no hardware calls |
| `lib/shared/can_serde.hpp` | Header only (depends on `CAN_message_t.hpp` which IS changing) |
| `lib/shared/util.cpp` | Pure C++ (`Trigger`, `Timer`, `str_hash`) — hardware-agnostic |
| `test/test_can_serde/main.cpp` | Uses `BUILD_MODE_TEST` path, runs on native |
| `test/test_ecu/main.cpp` | Uses `BUILD_MODE_TEST` path, runs on native |
| `config/native_tests.ini` | Tests run on native platform, not affected |
| `config/common.ini` | Common flags, no platform-specific content |
| `src/native.cpp` | Currently empty |

---

## Quick-Reference Change Checklist

- [ ] `config/ecu.ini` — Change platform, board, lib_deps, build_flags
- [ ] `config/dcf.ini` — Change platform, board, lib_deps, build_flags
- [ ] `config/ecu_old.ini` — Change platform, board, lib_deps, build_flags
- [ ] `platformio.ini` — Update `default_envs` to new STM32 env name
- [ ] `lib/shared/CAN_message_t.hpp` — Add `ARDUINO_ARCH_STM32` branch with STM32_CAN include
- [ ] `src/ecu.cpp` — Replace FlexCAN_T4 include, update CAN object declaration, fix LED pin
- [ ] `src/dcf.cpp` — Replace FlexCAN_T4 include, update CAN object declarations, update pin constants
- [ ] `src/ecu_old.cpp` — Replace FlexCAN_T4 include, update CAN object declaration
- [ ] `lib/shared/util.hpp` — Verify/fix Serial mapping for USART1 (PA9/PA10)
- [ ] Verify Serial → USART1 mapping in board variant or `ecu.cpp`
- [ ] Re-calibrate ADC constants in `lib/ecu/constants.hpp` after hardware testing
- [ ] Create custom board JSON if `nucleo_g474re` variant does not match PCB
- [ ] Set correct `upload_protocol` in PlatformIO config for the programmer being used
