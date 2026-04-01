# Driver Patterns for Microchip Zephyr BSP

## DFP vs ASF Headers — The #1 Compatibility Question

Microchip provides two styles of register definitions. They are **NOT interchangeable**.

### ASF (Atmel Software Framework) Headers
Used by existing `atmel,sam0-*` Zephyr drivers.
```c
// Type names: bare module name
Sercom *sercom = (Sercom *)base;

// Access: bitfield unions
sercom->USART.CTRLA.bit.ENABLE = 1;
sercom->USART.CTRLA.reg = SERCOM_USART_CTRLA_MODE_USART_INT_CLK;

// Defined in: component/sercom.h (ASF-style)
typedef union {
    struct {
        uint32_t SWRST:1;
        uint32_t ENABLE:1;
        uint32_t MODE:3;
        // ...
    } bit;
    uint32_t reg;
} SERCOM_USART_CTRLA_Type;
```

### DFP (Device Family Pack) Headers
Used by newer Microchip chips (PIC32CM, PIC32CK, etc.).
```c
// Type names: module_registers_t
sercom_registers_t *regs = (sercom_registers_t *)base;

// Access: direct register read/write with mask macros
regs->USART_INT.SERCOM_CTRLA |= SERCOM_USART_INT_CTRLA_ENABLE_Msk;
regs->USART_INT.SERCOM_CTRLA = SERCOM_USART_INT_CTRLA_MODE_USART_INT_CLK;

// Defined in: component/sercom.h (DFP-style)
typedef struct {
    __IO uint32_t SERCOM_CTRLA;
    __IO uint32_t SERCOM_CTRLB;
    __IO uint32_t SERCOM_CTRLC;
    __IO uint16_t SERCOM_BAUD;
    // ...
} sercom_usart_int_registers_t;
```

### Decision Tree

```
Does the chip have DFP headers (module_registers_t types)?
├── NO → Existing atmel,sam0-* drivers likely work. Try them first.
└── YES → Do existing drivers use ASF types?
    ├── YES → New drivers required. Cannot mix ASF and DFP.
    └── NO → Check if register layout matches. May just need new compatible string.
```

For PIC32CM JH: DFP headers → new drivers required for all peripherals.

## SERCOM Generations — Not All SERCOMs Are Equal

Different Microchip chip families have different SERCOM generations. The DFP type names are the same (`sercom_registers_t`, `sercom_i2cm_registers_t`), but the struct contents differ:

| Feature | PIC32CM (SERCOM G1) | PIC32CZ CA80/CA90 |
|---------|--------------------|--------------------|
| DATA register | 8-bit (`uint8_t`) | **32-bit** (`uint32_t`) |
| CTRLC register | not present | **DATA32B, FIFOEN, thresholds** |
| FIFO | none | **16-byte TX/RX FIFO** |
| CTRLA.FILTSEL | not present | **10ns/50ns input filter** |
| CTRLA.SLEWRATE | not present | **SM/FM/FMP/HS slew control** |
| CTRLA.SPEED | not present | **SM/FMP/HS mode select** |
| CTRLB.FIFOCLR | not present | **TX/RX FIFO clear** |
| Max I2C speed | 400 kHz | **3.4 MHz (HS mode)** |
| INTFLAG extras | none | **TXFE, RXFF (FIFO flags)** |
| Instances | typically 6 | **10 (sercom0–sercom9)** |

**When writing a driver for PIC32CZ:**
- Cast DATA reads to `uint8_t` (only low byte valid in byte mode)
- Explicitly zero CTRLC to disable FIFO and DATA32B
- Set FILTSEL for noise rejection (50ns recommended)
- Set SLEWRATE to match the bus speed
- Set CTRLA.SPEED for Fast Mode Plus (required — won't work without it)
- The FIFO is optional — polled byte-at-a-time works fine, FIFO+DMA is a future enhancement

**When writing a driver for PIC32CM:**
- DATA is 8-bit, no cast needed
- No CTRLC, no FIFO, no FILTSEL, no SLEWRATE
- Max speed is Fast Mode (400 kHz)

**Key insight:** A driver written for PIC32CM will NOT work on PIC32CZ without modifications (32-bit DATA, missing CTRLC/FILTSEL/SLEWRATE init). A driver written for PIC32CZ with proper CTRLC=0 and byte-mode DATA will be more portable.

## SERCOM G1 Driver Template

SERCOM is a multi-protocol peripheral (UART/I2C/SPI). Each mode has its own register overlay within the same `sercom_registers_t` union.

### Common Structure

```c
#define DT_DRV_COMPAT microchip_sercom_g1_<mode>

#include <soc.h>
#include <zephyr/device.h>
#include <zephyr/drivers/<subsystem>.h>
#include <zephyr/drivers/clock_control.h>
#include <zephyr/drivers/pinctrl.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(<driver_name>, CONFIG_<SUBSYS>_LOG_LEVEL);

struct <mode>_mchp_config {
    sercom_registers_t *regs;
    const struct device *clock_dev;
    clock_control_subsys_t gclk_sys;   /* GCLK channel for baud generation */
    clock_control_subsys_t mclk_sys;   /* MCLK bus gate for register access */
    const struct pinctrl_dev_config *pcfg;
    uint32_t bitrate;                  /* or clock-frequency */
};

struct <mode>_mchp_data {
    /* Runtime state: mutex, buffers, callbacks, etc. */
};
```

### Init Sequence (Same for All Modes)

```c
static int <mode>_mchp_init(const struct device *dev)
{
    const struct <mode>_mchp_config *cfg = dev->config;
    <mode>_registers_t *regs = &cfg->regs-><MODE>;

    /* 1. Enable clocks — GCLK first, then MCLK bus gate */
    clock_control_on(cfg->clock_dev, cfg->gclk_sys);
    clock_control_on(cfg->clock_dev, cfg->mclk_sys);

    /* 2. Software reset */
    regs->SERCOM_CTRLA = SERCOM_<MODE>_CTRLA_SWRST_Msk;
    while (regs->SERCOM_SYNCBUSY & SERCOM_<MODE>_SYNCBUSY_SWRST_Msk) {}

    /* 3. Configure mode-specific registers (CTRLA, CTRLB, BAUD) */
    regs->SERCOM_CTRLA = SERCOM_<MODE>_CTRLA_MODE_<mode_value>;
    /* ... mode-specific config ... */

    /* 4. Apply pinctrl */
    pinctrl_apply_state(cfg->pcfg, PINCTRL_STATE_DEFAULT);

    /* 5. Enable peripheral */
    regs->SERCOM_CTRLA |= SERCOM_<MODE>_CTRLA_ENABLE_Msk;
    while (regs->SERCOM_SYNCBUSY & SERCOM_<MODE>_SYNCBUSY_ENABLE_Msk) {}

    return 0;
}
```

### Clock Subsystem IDs

Each SERCOM needs two clock IDs in the DTS:
- `gclk` — Generic Clock channel (for baud rate generation)
- `mclk` — Main Clock bus gate (for register access)

```dts
clocks = <&gclkgen 0 19>,           /* GCLK generator 0, peripheral channel 19 */
         <&mclkperiph CLOCK_MCHP_MCLKPERIPH_ID_APBC_SERCOMx>;
clock-names = "gclk", "mclk";
```

The clock IDs are encoded with `MCHP_CLOCK_DERIVE_ID(type, bus, bit, gclk_mask, unique_id)`. Check the ATDF file for the correct MCLK bus (APBA/APBB/APBC/APBD) and bit position.

### SERCOM Mode Register Overlays

| Mode | Union Member | Registers Type |
|------|-------------|----------------|
| UART | `regs->USART_INT` | `sercom_usart_int_registers_t` |
| I2C Host | `regs->I2CM` | `sercom_i2cm_registers_t` |
| I2C Target | `regs->I2CS` | `sercom_i2cs_registers_t` |
| SPI Host | `regs->SPIM` | `sercom_spim_registers_t` |
| SPI Target | `regs->SPIS` | `sercom_spis_registers_t` |

### SPI-Specific: LOG_MODULE_REGISTER Ordering

**Critical:** When writing an SPI driver, `LOG_MODULE_REGISTER()` MUST come BEFORE `#include "spi_context.h"`. The `spi_context.h` header contains inline functions that reference logging symbols (`__log_level`, `__log_current_const_data`) which are defined by `LOG_MODULE_REGISTER`.

```c
/* CORRECT order */
LOG_MODULE_REGISTER(spi_mchp_sercom_g1);
#include "spi_context.h"

/* WRONG — will fail to compile */
#include "spi_context.h"
LOG_MODULE_REGISTER(spi_mchp_sercom_g1);
```

### I2C-Specific: Address Phase

After writing the address byte, wait for **both** MB and SB flags:
```c
ret = wait_bus_flag(i2cm, SERCOM_I2CM_INTFLAG_MB_Msk | SERCOM_I2CM_INTFLAG_SB_Msk);
```
MB fires for write-mode address ACK; SB fires for read-mode. Waiting for either avoids a subtle bug where the wrong flag is checked.

### I2C-Specific: CTRLB Write Hazard

**Critical:** CTRLB.CMD is a write-only action field (bits 17:16). When you do `|=` on CTRLB, the CPU reads the register first — but CMD always reads back as 0, not the last value written. This means `|=` is *safe* for CMD (you won't accidentally re-trigger a command), but you **must not** assume the read-back value of CMD reflects the current state.

The safer pattern is to build CTRLB from scratch each time:
```c
/* CORRECT: build the full value, don't read-modify-write */
uint32_t ctrlb = SERCOM_I2CM_CTRLB_SMEN_Msk;  /* preserve smart mode */
ctrlb |= SERCOM_I2CM_CTRLB_ACKACT_Msk;         /* NACK */
ctrlb |= SERCOM_I2CM_CTRLB_CMD(CMD_STOP);       /* issue STOP */
i2cm->SERCOM_CTRLB = ctrlb;
wait_sync(i2cm);

/* RISKY: |= re-reads CTRLB, CMD reads as 0, ACKACT may have stale state */
i2cm->SERCOM_CTRLB |= SERCOM_I2CM_CTRLB_ACKACT_Msk;
```

### I2C-Specific: Smart Mode vs Manual ACK

SERCOM I2C master has two ACK approaches — pick ONE, not both:

1. **Smart mode (SMEN=1):** Reading DATA automatically sends ACK and clocks the next byte. You only need to set ACKACT=NACK before reading the *last* byte. Simpler code, fewer bus transactions.

2. **Manual mode (SMEN=0):** After reading DATA, you must explicitly write CMD=READ_ACK to CTRLB to send ACK and clock the next byte. More control but more code.

If you enable smart mode AND write manual ACK commands, you risk double-triggering (auto-ACK from the read, then another ACK from your CMD write). The symptom is reading garbage or skipping bytes.

**Recommendation:** Use smart mode (SMEN=1) and only set ACKACT for the last byte. Do NOT also write CMD_READ_ACK.

### I2C-Specific: Read Sequence with Smart Mode

With smart mode enabled, the correct read loop is:
```c
for (uint32_t i = 0; i < msg->len; i++) {
    if (i == msg->len - 1) {
        /* Before reading last byte, set NACK so smart mode sends NACK */
        uint32_t ctrlb = SERCOM_I2CM_CTRLB_SMEN_Msk |
                          SERCOM_I2CM_CTRLB_ACKACT_Msk;
        if (msg->flags & I2C_MSG_STOP) {
            ctrlb |= SERCOM_I2CM_CTRLB_CMD(CMD_STOP);
        }
        i2cm->SERCOM_CTRLB = ctrlb;
        wait_sync(i2cm);
    }

    /* Reading DATA clears SB flag; smart mode auto-ACKs (or NACKs if last) */
    msg->buf[i] = (uint8_t)i2cm->SERCOM_DATA;

    if (i < msg->len - 1) {
        /* Wait for next byte to arrive */
        ret = wait_bus_flag(i2cm, SERCOM_I2CM_INTFLAG_SB_Msk);
        if (ret) {
            send_stop(i2cm);
            return ret;
        }
    }
}
```

### I2C-Specific: Baud Rate Calculation

The simplified formula `BAUD = fGCLK / (2 * fSCL) - 1` works for Standard Mode but becomes inaccurate at higher speeds. The datasheet formula accounts for the 10-cycle fixed overhead:

```c
/* Accurate formula per datasheet: fSCL = fGCLK / (10 + 2*BAUD) */
/* Solving for BAUD: BAUD = (fGCLK - 10*fSCL) / (2*fSCL) */
#define I2C_BAUD(fgclk, fscl) \
    MIN(((fgclk) - 10U * (fscl)) / (2U * (fscl)), 255U)
```

At 48 MHz / 400 kHz: simplified gives 59, accurate gives 55. At 1 MHz the error grows larger. Always use the datasheet formula.

### I2C-Specific: Use WAIT_FOR Macro

Zephyr provides `WAIT_FOR()` in `<zephyr/sys/util.h>` for polling loops. Use it instead of hand-rolled timeout loops:
```c
#include <zephyr/sys/util.h>

/* Zephyr idiomatic polling */
bool ready = WAIT_FOR(i2cm->SERCOM_INTFLAG & flag,
                      I2C_TIMEOUT_US, k_busy_wait(1));
if (!ready) {
    return -ETIMEDOUT;
}

/* Instead of hand-rolled: */
uint32_t timeout = I2C_TIMEOUT_US;
while (!(i2cm->SERCOM_INTFLAG & flag)) {
    if (--timeout == 0) return -ETIMEDOUT;
    k_busy_wait(1);
}
```

### I2C-Specific: Timeout Value

Use 10ms (10000us), not 100ms. At 100ms, an `i2c scan` across 112 addresses takes 11+ seconds when no devices respond. 10ms is more than enough for any I2C transaction at Standard Mode and above.

### I2C-Specific: Use LOG_DBG in Init

Use `LOG_DBG` (not `LOG_INF`) for init success messages. `LOG_INF` during driver init creates noise on the console and Zephyr maintainers will flag it.

### SPI-Specific: DOPO/DIPO Pin Mapping

| DOPO | MOSI Pad | SCK Pad |
|------|----------|---------|
| 0 | PAD[0] | PAD[1] |
| 1 | PAD[2] | PAD[3] |
| 2 | PAD[3] | PAD[1] |
| 3 | PAD[0] | PAD[3] |

DIPO selects which pad is MISO (0–3). Verify against ATDF pin multiplexing table.

### UART-Specific: TXPO/RXPO

| TXPO | TX Pad | Notes |
|------|--------|-------|
| 0 | PAD[0] | Most common |
| 1 | PAD[2] | |
| 2 | PAD[0] | With RTS/CTS flow control |
| 3 | PAD[0] | With RS485 |

RXPO selects which pad is RX (0–3). Must be consistent with pin function from ATDF.

## WDT Driver Pattern

The WDT is simpler than SERCOM — no mode switching, single configuration register.

Key points:
- WDT runs on a fixed 1.024 kHz oscillator — period is in cycles of that clock
- Clear/reset uses a key value: `WDT_CLEAR_CLEAR_KEY` (0xA5)
- Configuration is write-once after reset — must software-reset to reconfigure
- Window mode: write must occur within a window, not before `OFFSET` cycles
- Early Warning interrupt fires at configurable period before reset

```c
/* Period encoding: log2(cycles/8) */
static int timeout_to_period(uint32_t timeout_ms)
{
    uint32_t cycles = (timeout_ms * 1024) / 1000;  /* 1.024 kHz */
    /* Find the register value: 8 * 2^val cycles */
    for (int val = 0; val <= 11; val++) {
        if ((8U << val) >= cycles) {
            return val;
        }
    }
    return -EINVAL;
}
```

## DTS Binding YAML Pattern

Every new `compatible` string needs a binding file in `dts/bindings/`.

```yaml
# SPDX-License-Identifier: Apache-2.0

description: Microchip SERCOM G1 I2C controller

compatible: "microchip,sercom-g1-i2c"

include:
  - name: i2c-controller.yaml    # Provides #address-cells, #size-cells
  - name: pinctrl-device.yaml    # Provides pinctrl-0, pinctrl-names

properties:
  reg:
    required: true
  interrupts:
    required: true
  clocks:
    required: true
  clock-names:
    required: true
```

**Important:** Including `i2c-controller.yaml` automatically requires `#address-cells` and `#size-cells` in the node. Including `spi-controller.yaml` does the same. The board DTS must provide these.

## Kconfig + CMakeLists Pattern

### Kconfig (e.g., `drivers/i2c/Kconfig.mchp_sercom_g1`)
```kconfig
# Copyright (c) 2025 Microchip Technology Inc.
# SPDX-License-Identifier: Apache-2.0

config I2C_MCHP_SERCOM_G1
	bool "Microchip SERCOM G1 I2C driver"
	default y
	depends on DT_HAS_MICROCHIP_SERCOM_G1_I2C_ENABLED
	select PINCTRL
	help
	  I2C host driver for Microchip SERCOM G1 peripheral
	  using DFP register definitions.
```

### CMakeLists.txt addition
```cmake
zephyr_library_sources_ifdef(CONFIG_I2C_MCHP_SERCOM_G1 i2c_mchp_sercom_g1.c)
```

### Kconfig source inclusion
Add to the parent `Kconfig` file:
```kconfig
source "drivers/i2c/Kconfig.mchp_sercom_g1"
```
