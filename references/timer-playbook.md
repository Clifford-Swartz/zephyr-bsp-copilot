# Timer/Counter Playbook — Microchip TC and TCC

## Overview

Microchip has **two** timer peripherals that look similar but serve different roles:

| Peripheral | Width | Channels | Use Case |
|-----------|-------|----------|----------|
| **TC** (Timer/Counter) | 8/16/32-bit selectable | 2 compare channels | Counting, alarms, basic timing |
| **TCC** (Timer/Counter for Control) | 16 or 24-bit fixed | 4-8 output channels | PWM, motor control, waveform generation |

They map to **different Zephyr APIs**:
- TC → `counter` API (start/stop/alarm/top-value)
- TCC → `pwm` API (set_cycles/get_cycles_per_sec)

Do not confuse them. Read the DFP headers for both — they have different register layouts.

## TC Register Overlay

```c
tc_registers_t *tc = (tc_registers_t *)cfg->base;
```

TC has three register views depending on counter width:
- `tc->COUNT8` — 8-bit mode registers
- `tc->COUNT16` — 16-bit mode registers
- `tc->COUNT32` — 32-bit mode registers

Select mode via `TC_CTRLA_MODE`:

```c
tc->COUNT32.TC_CTRLA = TC_CTRLA_MODE_COUNT32 | TC_CTRLA_PRESCALER_DIV1;
```

**Gotcha:** The CC register width changes with the mode. In COUNT8, CC is `uint8_t`. In COUNT32, CC is `uint32_t`. Use the correct union member or you'll read/write wrong offsets.

## TCC Register Overlay

```c
tcc_registers_t *tcc = (tcc_registers_t *)cfg->base;
```

TCC has a single register view (no mode union). Counter width is fixed per device — 16-bit on PIC32CM JH, 24-bit on PIC32CZ CA. Check your DFP header.

## Prescaler Options

Both TC and TCC share the same prescaler values:

| Value | Divider |
|-------|---------|
| 0 | DIV1 |
| 1 | DIV2 |
| 2 | DIV4 |
| 3 | DIV8 |
| 4 | DIV16 |
| 5 | DIV64 |
| 6 | DIV256 |
| 7 | DIV1024 |

Expose the prescaler as a DTS property. The effective timer frequency is `fGCLK / prescaler`.

## SYNCBUSY — Critical for Both TC and TCC

TC and TCC registers cross clock domains. After writing CTRLA, CTRLB, COUNT, CC, or PER, you **must** wait for SYNCBUSY to clear.

```c
/* TC sync wait */
if (!WAIT_FOR(!(tc->COUNT32.TC_SYNCBUSY & TC_SYNCBUSY_Msk),
              10000, k_busy_wait(1))) {
    LOG_ERR("TC sync timeout");
    return -ETIMEDOUT;
}

/* TCC sync wait */
if (!WAIT_FOR(!(tcc->TCC_SYNCBUSY & TCC_SYNCBUSY_Msk),
              10000, k_busy_wait(1))) {
    LOG_ERR("TCC sync timeout");
    return -ETIMEDOUT;
}
```

**Gotcha:** Some older devices use `TC_STATUS_SYNCBUSY` instead of a separate SYNCBUSY register. Check the DFP header:
```c
/* Newer devices */
while (tc->COUNT32.TC_SYNCBUSY & TC_SYNCBUSY_ENABLE_Msk) {}

/* Older devices (no SYNCBUSY register) */
while (tc->COUNT32.TC_STATUS & TC_STATUS_SYNCBUSY_Msk) {}
```

## TC Init Sequence (Counter API)

```c
static int tc_mchp_init(const struct device *dev)
{
    tc_count32_registers_t *tc32 = &cfg->regs->COUNT32;

    /* 1. Enable clocks (GCLK + MCLK) */
    clock_control_on(cfg->clock_dev, cfg->gclk_sys);
    clock_control_on(cfg->clock_dev, cfg->mclk_sys);

    /* 2. Software reset */
    tc32->TC_CTRLA = TC_CTRLA_SWRST_Msk;
    WAIT_FOR(!(tc32->TC_SYNCBUSY & TC_SYNCBUSY_SWRST_Msk), 10000, k_busy_wait(1));

    /* 3. Configure: 32-bit mode, prescaler, wave generation */
    tc32->TC_CTRLA = TC_CTRLA_MODE_COUNT32
                   | TC_CTRLA_PRESCALER(cfg->prescaler)
                   | TC_CTRLA_PRESCSYNC_PRESC;

    /* 4. Set wave generation mode (NFRQ for counter, MPWM for match) */
    tc32->TC_WAVE = TC_WAVE_WAVEGEN_NFRQ;

    /* 5. Apply pinctrl (only if using external clock or output) */
    pinctrl_apply_state(cfg->pcfg, PINCTRL_STATE_DEFAULT);

    /* 6. Enable — don't start counting yet (start() API does that) */
    tc32->TC_CTRLA |= TC_CTRLA_ENABLE_Msk;
    WAIT_FOR(!(tc32->TC_SYNCBUSY & TC_SYNCBUSY_ENABLE_Msk), 10000, k_busy_wait(1));

    return 0;
}
```

## TC Start/Stop Commands

TC uses CTRLB commands (not CTRLA) to start and stop:

```c
/* Start: retrigger command resets count and starts */
tc32->TC_CTRLBSET = TC_CTRLBSET_CMD_RETRIGGER;
WAIT_FOR(!(tc32->TC_SYNCBUSY & TC_SYNCBUSY_CTRLB_Msk), 10000, k_busy_wait(1));

/* Stop */
tc32->TC_CTRLBSET = TC_CTRLBSET_CMD_STOP;
WAIT_FOR(!(tc32->TC_SYNCBUSY & TC_SYNCBUSY_CTRLB_Msk), 10000, k_busy_wait(1));
```

## TC Counter Read — Requires READSYNC

Reading the COUNT register on an asynchronous timer requires a sync command first:

```c
static uint32_t tc_read_count(tc_count32_registers_t *tc32)
{
    tc32->TC_CTRLBSET = TC_CTRLBSET_CMD_READSYNC;
    WAIT_FOR(!(tc32->TC_SYNCBUSY & TC_SYNCBUSY_CTRLB_Msk), 10000, k_busy_wait(1));
    /* Now COUNT is safe to read */
    return tc32->TC_COUNT;
}
```

Without READSYNC, you may read a stale or partially-updated value.

## TC Alarms (Compare Channels)

TC has 2 compare channels: CC0 and CC1. Map these to Zephyr alarm channels:

```c
/* Set alarm on channel 0 */
tc32->TC_CC[0] = alarm_ticks;
WAIT_FOR(!(tc32->TC_SYNCBUSY & TC_SYNCBUSY_CC0_Msk), 10000, k_busy_wait(1));

/* Enable match interrupt */
tc32->TC_INTENSET = TC_INTENSET_MC0_Msk;

/* In ISR: */
if (tc32->TC_INTFLAG & TC_INTFLAG_MC0_Msk) {
    tc32->TC_INTFLAG = TC_INTFLAG_MC0_Msk; /* clear by writing 1 */
    /* Fire alarm callback */
}
```

## TCC Init Sequence (PWM API)

```c
static int tcc_mchp_init(const struct device *dev)
{
    tcc_registers_t *tcc = cfg->regs;

    /* 1. Enable clocks */
    clock_control_on(cfg->clock_dev, cfg->gclk_sys);
    clock_control_on(cfg->clock_dev, cfg->mclk_sys);

    /* 2. Software reset */
    tcc->TCC_CTRLA = TCC_CTRLA_SWRST_Msk;
    WAIT_FOR(!(tcc->TCC_SYNCBUSY & TCC_SYNCBUSY_SWRST_Msk), 10000, k_busy_wait(1));

    /* 3. Configure prescaler */
    tcc->TCC_CTRLA = TCC_CTRLA_PRESCALER(cfg->prescaler);

    /* 4. Set waveform mode — NPWM for normal PWM */
    tcc->TCC_WAVE = TCC_WAVE_WAVEGEN_NPWM;
    WAIT_FOR(!(tcc->TCC_SYNCBUSY & TCC_SYNCBUSY_WAVE_Msk), 10000, k_busy_wait(1));

    /* 5. Apply pinctrl (PWM output pins) */
    pinctrl_apply_state(cfg->pcfg, PINCTRL_STATE_DEFAULT);

    /* 6. Enable */
    tcc->TCC_CTRLA |= TCC_CTRLA_ENABLE_Msk;
    WAIT_FOR(!(tcc->TCC_SYNCBUSY & TCC_SYNCBUSY_ENABLE_Msk), 10000, k_busy_wait(1));

    return 0;
}
```

## TCC Double-Buffering

TCC has buffered registers (`PERBUF`, `CCBUF[]`) for glitch-free updates. Write to the buffer registers — hardware loads them into active registers at the next cycle boundary:

```c
/* Update period and duty cycle without glitches */
tcc->TCC_PERBUF = new_period;
tcc->TCC_CCBUF[channel] = new_pulse;
/* Hardware applies at next counter reset — no explicit update command needed */
```

**Gotcha:** If you write directly to PER or CC instead of PERBUF/CCBUF, the change takes effect immediately and can cause a glitch (partial cycle with wrong timing).

## TCC Output Polarity

To invert a PWM channel's output polarity:

```c
/* Must disable TCC first */
tcc->TCC_CTRLA &= ~TCC_CTRLA_ENABLE_Msk;
WAIT_FOR(!(tcc->TCC_SYNCBUSY & TCC_SYNCBUSY_ENABLE_Msk), 10000, k_busy_wait(1));

/* Toggle inversion for channel N */
tcc->TCC_DRVCTRL ^= TCC_DRVCTRL_INVEN(1 << channel);

/* Re-enable */
tcc->TCC_CTRLA |= TCC_CTRLA_ENABLE_Msk;
```

Cannot change polarity while TCC is running.

## Zephyr Counter API Mapping (TC)

| Zephyr API | TC Action |
|-----------|----------|
| `start()` | CTRLBSET CMD_RETRIGGER |
| `stop()` | CTRLBSET CMD_STOP |
| `get_value()` | CMD_READSYNC then read COUNT |
| `set_alarm()` | Write CC[n], enable MC[n] interrupt |
| `cancel_alarm()` | Disable MC[n] interrupt |
| `set_top_value()` | Write CC[1] as top (or use PER if available) |
| `get_pending_int()` | Check INTFLAG |

## Zephyr PWM API Mapping (TCC)

| Zephyr API | TCC Action |
|-----------|-----------|
| `set_cycles()` | Write PERBUF (period), CCBUF[ch] (pulse) |
| `get_cycles_per_sec()` | Return fGCLK / prescaler |

## Generation Differences

| Feature | PIC32CM JH | PIC32CZ CA |
|---------|-----------|-----------|
| TCC counter width | 16-bit | 24-bit |
| TCC output channels | 4 | up to 8 |
| TCC capture channels | CPTEN[3:0] | CPTEN[7:0] |
| TC modes | 8/16/32-bit | 8/16/32-bit |
| TC SYNCBUSY | Separate register | Separate register |

**Key gotcha:** PIC32CZ has more TCC channels — loop bounds and DTS channel count must reflect the actual device. Don't hardcode `4`.

## DTS Properties

### TC (Counter)
```dts
tc0: tc@42002000 {
    compatible = "microchip,tc-counter";
    reg = <0x42002000 0x20>;
    interrupts = <73 0>;
    clocks = <&gclk 27>, <&mclk 14>;
    clock-names = "gclk", "mclk";
    prescaler = <64>;
};
```

### TCC (PWM)
```dts
tcc0: tcc@41016000 {
    compatible = "microchip,tcc-pwm";
    reg = <0x41016000 0x80>;
    interrupts = <61 0>;
    clocks = <&gclk 25>, <&mclk 11>;
    clock-names = "gclk", "mclk";
    prescaler = <1>;
    channels = <4>;
    #pwm-cells = <3>;
};
```
