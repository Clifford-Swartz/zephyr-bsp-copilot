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

## Peripheral Generations — Never Assume Register Compatibility

Microchip reuses peripheral names (SERCOM, MCLK, PORT, NVMCTRL) across chip families, but the register layouts evolve between generations. **Same peripheral name does NOT mean same registers.**

### How to Check Before Writing a Driver

Before writing or porting any driver, run this checklist against the DFP headers for your specific chip (found in `modules/hal/microchip/packs/<family>/include/component/`):

1. **Open the component header** (e.g., `sercom.h`) and find the register struct (e.g., `sercom_i2cm_registers_t`). List every field — its type, offset, and name.
2. **Compare field-by-field** against any reference driver you're adapting from. Look for:
   - **Different register widths** (e.g., DATA is `uint8_t` on one chip, `uint32_t` on another)
   - **Extra registers** (e.g., CTRLC, FIFOSPACE, LENGTH — not present on older parts)
   - **Extra bitfields in existing registers** (e.g., CTRLA gaining FILTSEL, SLEWRATE, SPEED fields)
   - **Missing registers** (older chips may lack features newer drivers assume)
3. **For every extra register/field you find:**
   - Decide whether it needs explicit initialization (even if you're not using the feature)
   - Check the reset default — is it safe, or could a bootloader have configured it?
   - Add a comment in your driver explaining why you set (or don't set) it
4. **For every field width change:**
   - A 32-bit DATA register still works for 8-bit I2C, but you must cast reads to `uint8_t`
   - Check if there's a mode control (e.g., CTRLC.DATA32B) that selects byte vs word mode

### Example: SERCOM Across Generations

| What to check | Older generation | Newer generation |
|---------------|-----------------|------------------|
| DATA width | 8-bit | 32-bit → cast reads, check for DATA32B mode bit |
| Extra control reg | No CTRLC | CTRLC exists → zero it to disable FIFO/32-bit mode |
| FIFO support | None | 16-byte FIFO → explicitly disable if not using |
| Input filter | Not available | FILTSEL field → set for noise rejection |
| Slew rate | Not available | SLEWRATE field → set to match bus speed |
| Speed mode | Implicit | SPEED field → must set for FM+ or HS |
| Extra IRQ flags | MB, SB only | TXFE, RXFF → don't assume flag register width |

**The principle:** Don't start from a reference driver and assume it works. Start from the DFP header for YOUR chip, list what's there, and build the driver to match.

## SERCOM Driver Template

SERCOM is a multi-protocol peripheral (UART/I2C/SPI). Each mode has its own register overlay within the same `sercom_registers_t` union.

### SERCOM Mode Register Overlays

| Mode | Union Member | Registers Type |
|------|-------------|----------------|
| UART | `regs->USART_INT` | `sercom_usart_int_registers_t` |
| I2C Host | `regs->I2CM` | `sercom_i2cm_registers_t` |
| I2C Target | `regs->I2CS` | `sercom_i2cs_registers_t` |
| SPI Host | `regs->SPIM` | `sercom_spim_registers_t` |
| SPI Target | `regs->SPIS` | `sercom_spis_registers_t` |

### Common Structure

```c
#define DT_DRV_COMPAT microchip_sercom_<mode>

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
clocks = <&gclk 21>,    /* GCLK peripheral channel */
         <&mclk 31>;    /* MCLK APB bit */
clock-names = "gclk", "mclk";
```

The clock IDs come from the ATDF file (`GCLK_ID` and `MCLK_ID_APB` parameters). If using the soc-profile from the pipeline, these are pre-extracted.

## DTS Binding YAML Pattern

Every new `compatible` string needs a binding file in `dts/bindings/`.

```yaml
# SPDX-License-Identifier: Apache-2.0

description: Microchip SERCOM I2C controller

compatible: "microchip,sercom-i2c"

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

### Kconfig
```kconfig
# Copyright (c) 2026 Microchip Technology Inc.
# SPDX-License-Identifier: Apache-2.0

config I2C_MCHP_SERCOM
	bool "Microchip SERCOM I2C driver"
	default y
	depends on DT_HAS_MICROCHIP_SERCOM_I2C_ENABLED
	select PINCTRL
	help
	  I2C host driver for Microchip SERCOM peripheral
	  using DFP register definitions.
```

### CMakeLists.txt addition
```cmake
zephyr_library_sources_ifdef(CONFIG_I2C_MCHP_SERCOM i2c_mchp_sercom.c)
```

### Kconfig source inclusion
Add to the parent `Kconfig` file:
```kconfig
source "drivers/i2c/Kconfig.mchp_sercom"
```

## Peripheral-Specific Playbooks

For detailed guidance on each peripheral type, see:
- `references/i2c-playbook.md` — Smart mode, CTRLB hazard, baud formula, FILTSEL
- `references/spi-playbook.md` — DOPO/DIPO mapping, spi_context.h ordering, chip select
- `references/uart-playbook.md` — TXPO/RXPO mapping, console setup, baud formula
- `references/wdt-playbook.md` — Period encoding, window mode, early warning
- `references/clock-playbook.md` — GCLK/MCLK architecture, init sequence, common pitfalls
- `references/flash-playbook.md` — NVMCTRL write granularity, erase alignment, region locks, cache
- `references/timer-playbook.md` — TC vs TCC, counter API vs PWM API, SYNCBUSY, double-buffering
