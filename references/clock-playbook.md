# Clock Controller Playbook — Microchip

## Overview

Every peripheral on the chip depends on the clock controller. It's the first driver that must work after GPIO/pinctrl. The Microchip clock system has three major blocks:

1. **OSCCTRL** — Clock sources (XOSC, DFLL48M, DPLL)
2. **GCLK** — Generic Clock controller (multiplexes sources to peripheral channels)
3. **MCLK** — Main Clock controller (bus gates for register access)

## Two Clocks Per Peripheral

Every SERCOM (and most other peripherals) needs two clock enables:

| Clock | Purpose | What Happens Without It |
|-------|---------|------------------------|
| **GCLK channel** | Provides the reference clock for baud rate generation | Peripheral has no clock source — baud register has no effect |
| **MCLK bus gate** | Enables register access on the APB bus | Reading/writing registers returns bus fault or zeros |

Enable GCLK first (so the peripheral has a clock), then MCLK (so you can access its registers).

## GCLK Architecture

```
Clock Sources          GCLK Generators (0-N)          Peripheral Channels
+-----------+         +------------------+            +-------------------+
| XOSC0     |-------->| Generator 0      |----------->| SERCOM0 (ch 21)   |
| XOSC1     |    +--->| Generator 1      |--+-------->| SERCOM1 (ch 22)   |
| DFLL48M   |----+    | Generator 2      |  |    +--->| TC0 (ch 45)       |
| DPLL0     |         | ...              |  |    |    | ...               |
| DPLL1     |         +------------------+  +----+    +-------------------+
+-----------+
```

Each generator selects one source and divides it. Each peripheral channel selects one generator.

## MCLK Bus Gates

MCLK has per-bus enable registers:
- `MCLK.APBAMASK` — APB-A peripherals (PM, SUPC, RSTC, GCLK, MCLK, etc.)
- `MCLK.APBBMASK` — APB-B peripherals (USB, DSU, etc.)
- `MCLK.APBCMASK` — APB-C peripherals (SERCOM0-5, TCC, TC, etc.)
- `MCLK.APBDMASK` — APB-D peripherals (SERCOM6-9, etc.)

The MCLK_ID_APB value from the ATDF tells you which bus and bit position.

## Clock Control API

The Zephyr clock control API maps to:

```c
/* clock_control_on() — enable a peripheral's clock */
/* For GCLK: write PCHCTRL[channel].GEN = generator_id, CHEN = 1 */
/* For MCLK: set the bit in APBxMASK */

/* clock_control_get_rate() — return the frequency */
/* Read back the GCLK generator's source and divider, compute frequency */
```

## DTS Clock Encoding

Each peripheral node needs two clock references:
```dts
clocks = <&gclk 21>,    /* GCLK peripheral channel 21 */
         <&mclk 31>;    /* MCLK APB bit 31 */
clock-names = "gclk", "mclk";
```

The channel and bit numbers come from the ATDF file (`GCLK_ID` and `MCLK_ID_APB` parameters on each peripheral instance). If using the soc-profile.yaml from the pipeline, these are pre-extracted.

## Init Sequence

The clock controller itself initializes early (PRE_KERNEL_1):

1. **Configure oscillator sources** — enable XOSC, DFLL, DPLLs as needed
2. **Configure GCLK generators** — assign sources, set dividers
3. **Don't enable peripheral channels yet** — individual drivers call `clock_control_on()` during their init

## Common Pitfalls

- **Forgetting MCLK gate**: The peripheral appears to work in the debugger (which bypasses bus gates) but faults at runtime
- **Wrong GCLK generator**: Generator 0 is often the main CPU clock at high frequency. Peripheral baud calculation assumes the GCLK frequency — if you assign the wrong generator, baud is wrong
- **DFLL not locked**: If using DFLL48M as a source, wait for DFLLRDY before configuring generators that use it
- **Bus gate ordering**: Some peripherals (like NVMCTRL) need their MCLK gate enabled before flash operations work — this can affect boot if the clock driver initializes too late
