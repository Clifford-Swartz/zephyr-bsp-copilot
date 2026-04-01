# WDT Playbook — Microchip

## Overview

The WDT is simpler than SERCOM — no mode switching, single configuration register. It runs on a fixed 1.024 kHz oscillator, independent of the main clock.

## Key Points

- WDT runs on a fixed 1.024 kHz oscillator — period is in cycles of that clock
- Clear/reset uses a key value: `WDT_CLEAR_CLEAR_KEY` (0xA5)
- Configuration is write-once after reset — must software-reset to reconfigure
- Window mode: write must occur within a window, not before `OFFSET` cycles
- Early Warning interrupt fires at configurable period before reset

## Period Encoding

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

This gives periods from ~8ms (val=0, 8 cycles) to ~16s (val=11, 16384 cycles).

## Zephyr WDT API Mapping

| Zephyr API | WDT Register Action |
|-----------|-------------------|
| `wdt_setup()` | Set CONFIG.PER, CONFIG.WINDOW, enable WDT |
| `wdt_feed()` | Write CLEAR = 0xA5 |
| `wdt_disable()` | Software reset (CTRLA.SWRST) — only way to disable |
| `wdt_install_timeout()` | Store period/window config for next `setup()` |

## Window Mode

If `CONFIG.WINDOW` is set to a nonzero value, feeds must arrive AFTER the window-open point but BEFORE the timeout. Early feeds trigger a reset. This is useful for detecting stuck loops that feed too fast.

## Early Warning Interrupt

The EW interrupt (INTFLAG.EW) fires at a configurable period before the WDT timeout. Use this to log diagnostic information or attempt graceful shutdown before the hard reset hits.

```c
/* In the ISR: */
if (wdt->INTFLAG & WDT_INTFLAG_EW_Msk) {
    wdt->INTFLAG = WDT_INTFLAG_EW_Msk;  /* clear flag */
    /* Log, save state, notify user callback */
}
```
