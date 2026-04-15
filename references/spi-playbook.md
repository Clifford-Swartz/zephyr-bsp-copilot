# SPI Playbook — Microchip SERCOM

## Register Overlay

Access SPI master registers via the SERCOM union:
```c
sercom_spim_registers_t *spim = &cfg->regs->SPIM;
```

For SPI target mode: `sercom_spis_registers_t *spis = &cfg->regs->SPIS;`

## LOG_MODULE_REGISTER Ordering

**Critical:** `LOG_MODULE_REGISTER()` MUST come BEFORE `#include "spi_context.h"`. The `spi_context.h` header contains inline functions that reference logging symbols (`__log_level`, `__log_current_const_data`) which are defined by `LOG_MODULE_REGISTER`.

```c
/* CORRECT order */
LOG_MODULE_REGISTER(spi_mchp_sercom_g1);
#include "spi_context.h"

/* WRONG — will fail to compile */
#include "spi_context.h"
LOG_MODULE_REGISTER(spi_mchp_sercom_g1);
```

## DOPO/DIPO Pin Mapping

| DOPO | MOSI Pad | SCK Pad |
|------|----------|---------|
| 0 | PAD[0] | PAD[1] |
| 1 | PAD[2] | PAD[3] |
| 2 | PAD[3] | PAD[1] |
| 3 | PAD[0] | PAD[3] |

DIPO selects which pad is MISO (0-3). Verify against ATDF pin multiplexing table.

## Sync-Busy Polling — Use `WAIT_FOR()`

SERCOM registers that are synchronized across clock domains (CTRLA, CTRLB, ENABLE, SWRST) have a SYNCBUSY register. After writing these, you must wait for the sync to complete. **Use the `WAIT_FOR()` macro** from `<zephyr/sys/util.h>` instead of bare `while()` loops — it's Zephyr-idiomatic and prevents infinite hangs if hardware misbehaves.

```c
#include <zephyr/sys/util.h>

/* After software reset */
spim->SERCOM_CTRLA = SERCOM_SPIM_CTRLA_SWRST_Msk;
if (!WAIT_FOR(!(spim->SERCOM_SYNCBUSY & SERCOM_SPIM_SYNCBUSY_SWRST_Msk),
              10000, k_busy_wait(1))) {
    LOG_ERR("SWRST sync timeout");
    return -ETIMEDOUT;
}

/* After enabling peripheral */
spim->SERCOM_CTRLA |= SERCOM_SPIM_CTRLA_ENABLE_Msk;
if (!WAIT_FOR(!(spim->SERCOM_SYNCBUSY & SERCOM_SPIM_SYNCBUSY_ENABLE_Msk),
              10000, k_busy_wait(1))) {
    LOG_ERR("ENABLE sync timeout");
    return -ETIMEDOUT;
}
```

Use 10ms (10000 µs) as the timeout — sync operations complete in microseconds, so 10ms is generous. Apply `WAIT_FOR()` to every SYNCBUSY poll: SWRST, ENABLE, CTRLB writes.

For data transfer flags (DRE, RXC, TXC), `WAIT_FOR()` is also recommended:
```c
if (!WAIT_FOR(spim->SERCOM_INTFLAG & SERCOM_SPIM_INTFLAG_DRE_Msk,
              1000, k_busy_wait(1))) {
    LOG_ERR("DRE timeout");
    return -ETIMEDOUT;
}
```

## Chip Select Handling

Zephyr's SPI subsystem manages chip select via GPIO by default. The SERCOM hardware CS (SS) pin is generally not used in master mode. Let the SPI framework handle `cs-gpios` from the DTS.

If you need hardware CS, set `CTRLB.MSSEN = 1` and configure the SS pad via pinctrl. But this limits you to a single device per bus — GPIO CS is more flexible.

## DMA Considerations

For polled drivers, simply write/read DATA in a loop. For DMA:
- TX DMA trigger: `SERCOM_DMAC_ID_TX` from the instance header
- RX DMA trigger: `SERCOM_DMAC_ID_RX` from the instance header
- DMA source/dest address: `&spim->SERCOM_DATA`
- DATA register width matters: 32-bit on newer parts, 8-bit on older

## Frame Format

CTRLA.DORD controls bit order (0 = MSB first, 1 = LSB first).
CTRLA.CPOL and CTRLA.CPHA set the SPI mode (0-3).
CTRLB.CHSIZE sets the character size (0 = 8-bit, 1 = 9-bit).

Map Zephyr's `SPI_MODE_*` flags to these register fields in `configure()`.
