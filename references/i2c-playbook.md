# I2C Playbook — Microchip SERCOM

## Register Overlay

Access I2C master registers via the SERCOM union:
```c
sercom_i2cm_registers_t *i2cm = &cfg->regs->I2CM;
```

For I2C target mode: `sercom_i2cs_registers_t *i2cs = &cfg->regs->I2CS;`

## Address Phase

After writing the address byte, wait for **both** MB and SB flags:
```c
ret = wait_bus_flag(i2cm, SERCOM_I2CM_INTFLAG_MB_Msk | SERCOM_I2CM_INTFLAG_SB_Msk);
```
MB fires for write-mode address ACK; SB fires for read-mode. Waiting for either avoids a subtle bug where the wrong flag is checked.

## CTRLB Write Hazard

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

## Smart Mode vs Manual ACK

SERCOM I2C master has two ACK approaches — pick ONE, not both:

1. **Smart mode (SMEN=1):** Reading DATA automatically sends ACK and clocks the next byte. You only need to set ACKACT=NACK before reading the *last* byte. Simpler code, fewer bus transactions.

2. **Manual mode (SMEN=0):** After reading DATA, you must explicitly write CMD=READ_ACK to CTRLB to send ACK and clock the next byte. More control but more code.

If you enable smart mode AND write manual ACK commands, you risk double-triggering (auto-ACK from the read, then another ACK from your CMD write). The symptom is reading garbage or skipping bytes.

**Recommendation:** Use smart mode (SMEN=1) and only set ACKACT for the last byte. Do NOT also write CMD_READ_ACK.

## Read Sequence with Smart Mode

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

## Baud Rate Calculation

The simplified formula `BAUD = fGCLK / (2 * fSCL) - 1` works for Standard Mode but becomes inaccurate at higher speeds. The datasheet formula accounts for the 10-cycle fixed overhead:

```c
/* Accurate formula per datasheet: fSCL = fGCLK / (10 + 2*BAUD) */
/* Solving for BAUD: BAUD = (fGCLK - 10*fSCL) / (2*fSCL) */
#define I2C_BAUD(fgclk, fscl) \
    MIN(((fgclk) - 10U * (fscl)) / (2U * (fscl)), 255U)
```

At 48 MHz / 400 kHz: simplified gives 59, accurate gives 55. At 1 MHz the error grows larger. Always use the datasheet formula.

## Use WAIT_FOR Macro

Zephyr provides `WAIT_FOR()` in `<zephyr/sys/util.h>` for polling loops. Use it instead of hand-rolled timeout loops:
```c
#include <zephyr/sys/util.h>

/* Zephyr idiomatic polling */
bool ready = WAIT_FOR(i2cm->SERCOM_INTFLAG & flag,
                      I2C_TIMEOUT_US, k_busy_wait(1));
if (!ready) {
    return -ETIMEDOUT;
}
```

## Timeout Value

Use 10ms (10000us), not 100ms. At 100ms, an `i2c scan` across 112 addresses takes 11+ seconds when no devices respond. 10ms is more than enough for any I2C transaction at Standard Mode and above.

## FILTSEL (Input Filter Selection)

If your chip's SERCOM has a FILTSEL field in CTRLA, set it during configuration. This controls the input filter on the SDA/SCL lines for noise rejection. Without it, fast-edge signals on long traces or noisy boards can cause spurious start/stop conditions.

```c
/* In the CTRLA setup, before enabling: */
ctrla |= SERCOM_I2CM_CTRLA_FILTSEL_FILTER1;  /* or FILTER2, FILTER3 */
```

Check your DFP header for available filter values — they vary by chip. If the field doesn't exist in the header, the chip doesn't support it and you can skip this.

## Implement All API Callbacks

Zephyr's `i2c_driver_api` struct includes `configure`, `transfer`, and `get_config`. Implement all of them — `i2c_get_config` is trivial but its absence causes runtime failures when upper layers query the bus speed:

```c
static int i2c_mchp_sercom_get_config(const struct device *dev, uint32_t *dev_config)
{
    struct i2c_mchp_sercom_data *data = dev->data;
    *dev_config = data->dev_config;
    return 0;
}

static DEVICE_API(i2c, i2c_mchp_sercom_api) = {
    .configure = i2c_mchp_sercom_configure,
    .transfer = i2c_mchp_sercom_transfer,
    .get_config = i2c_mchp_sercom_get_config,
};
```

Store `dev_config` in your driver data struct during `configure()` so `get_config()` can return it.

## Use LOG_DBG in Init

Use `LOG_DBG` (not `LOG_INF`) for init success messages. `LOG_INF` during driver init creates noise on the console and Zephyr maintainers will flag it.
