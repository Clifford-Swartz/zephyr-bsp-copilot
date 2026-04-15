# Flash (NVMCTRL) Playbook — Microchip

## Overview

NVMCTRL controls internal flash read/write/erase. It's unlike SERCOM drivers — no baud rate, no pins, no clock routing for data. But it has its own class of hazards: silent data corruption from wrong write granularity, erase alignment violations, and region lock surprises.

## Register Overlay

```c
nvmctrl_registers_t *nvm = (nvmctrl_registers_t *)cfg->base;
```

Key registers:
| Register | Purpose |
|----------|---------|
| CTRLA | Command execution (CMD + CMDEX key), write mode (WMODE), auto wait states |
| CTRLB | Read wait states (RWS), manual write (MANW), cache control (CACHEDIS, READMODE) |
| STATUS | Error flags: PROGE, LOCKE, NVME, LOAD |
| ADDR | Target address for erase/lock/unlock operations |
| LOCK | 16-bit region lock bitmap |
| PARAM | Read-only: page size and page count |
| INTFLAG | READY (operation complete), ERROR |

## Command Execution Key

Every command write requires the execution key `0xA5` in the CMDEX field. Without it, the command is silently ignored.

```c
/* PIC32CM JH style — command via CTRLA */
nvm->NVMCTRL_CTRLA = NVMCTRL_CTRLA_CMD_WP | NVMCTRL_CTRLA_CMDEX_KEY;

/* PIC32CZ style — command via CTRLB */
nvm->NVMCTRL_CTRLB = NVMCTRL_CTRLB_CMD_EB | NVMCTRL_CTRLB_CMDEX_KEY;
```

Check your DFP header — the command register moved from CTRLA to CTRLB on newer generations.

## Write Granularity

This is the #1 source of flash driver bugs. The minimum write unit depends on the write mode:

| WMODE | Write Unit | Description |
|-------|-----------|-------------|
| MAN (0) | Manual — requires explicit write command after loading buffer | Most control |
| ADW (1) | 8 bytes (double-word) | Auto-commits after 8 bytes written |
| AQW (2) | 16 bytes (quad-word) | Auto-commits after 16 bytes written |
| AP (3) | Full page (typically 512 bytes) | Auto-commits when page buffer is full |

**Critical rules:**
- All writes to the page buffer MUST be 32-bit aligned. 8-bit or 16-bit writes cause a bus fault.
- The page buffer is memory-mapped — you write directly to the flash address and it goes into the buffer.
- Call PBC (Page Buffer Clear) before starting a new write sequence.
- Check READY in INTFLAG after every command.

```c
/* Clear page buffer first */
nvm->NVMCTRL_CTRLA = NVMCTRL_CTRLA_CMD_PBC | NVMCTRL_CTRLA_CMDEX_KEY;
WAIT_FOR(nvm->NVMCTRL_INTFLAG & NVMCTRL_INTFLAG_READY_Msk, 10000, k_busy_wait(1));

/* Write 32-bit words directly to flash address (fills page buffer) */
volatile uint32_t *dst = (volatile uint32_t *)target_addr;
const uint32_t *src = (const uint32_t *)data;
for (int i = 0; i < words; i++) {
    dst[i] = src[i];
}

/* Issue write command (if manual mode) */
nvm->NVMCTRL_CTRLA = NVMCTRL_CTRLA_CMD_WP | NVMCTRL_CTRLA_CMDEX_KEY;
WAIT_FOR(nvm->NVMCTRL_INTFLAG & NVMCTRL_INTFLAG_READY_Msk, 10000, k_busy_wait(1));
```

## Erase Before Write

Flash can only be written from 1→0. To set bits back to 1, you must erase first. Erase sets all bits to 0xFF.

- Erase block size is typically 8 KB (check DTS `erase-block-size` property)
- Erase offset and size MUST be aligned to the erase block size
- Always erase before writing to a region that isn't already 0xFF

```c
/* Set target address */
nvm->NVMCTRL_ADDR = target_addr;

/* Issue erase command */
nvm->NVMCTRL_CTRLA = NVMCTRL_CTRLA_CMD_ER | NVMCTRL_CTRLA_CMDEX_KEY;
WAIT_FOR(nvm->NVMCTRL_INTFLAG & NVMCTRL_INTFLAG_READY_Msk, 50000, k_busy_wait(10));
```

Use a longer timeout for erase (50ms) — it's significantly slower than write.

## Region Locks

Flash is divided into 16 lock regions (region size = total flash / 16). Locked regions reject write and erase commands with a LOCKE error.

```c
/* Unlock region containing target address */
nvm->NVMCTRL_ADDR = target_addr;
nvm->NVMCTRL_CTRLA = NVMCTRL_CTRLA_CMD_UR | NVMCTRL_CTRLA_CMDEX_KEY;
WAIT_FOR(nvm->NVMCTRL_INTFLAG & NVMCTRL_INTFLAG_READY_Msk, 10000, k_busy_wait(1));

/* Re-lock after write/erase */
nvm->NVMCTRL_CTRLA = NVMCTRL_CTRLA_CMD_LR | NVMCTRL_CTRLA_CMDEX_KEY;
```

Check LOCK register to see current lock state. Unlock before write/erase, re-lock after.

## Error Checking

Always check error flags after every command:

```c
uint16_t status = nvm->NVMCTRL_STATUS;
if (status & (NVMCTRL_STATUS_PROGE_Msk | NVMCTRL_STATUS_LOCKE_Msk | NVMCTRL_STATUS_NVME_Msk)) {
    LOG_ERR("flash error: STATUS=0x%04x", status);
    /* Clear error flags by writing 1 */
    nvm->NVMCTRL_STATUS = status;
    return -EIO;
}
```

| Flag | Meaning |
|------|---------|
| PROGE | Programming error — wrong alignment or write to protected area |
| LOCKE | Lock error — region is locked |
| NVME | NVM error — generic controller error |

## Unaligned Write Handling

If `CONFIG_FLASH_HAS_UNALIGNED_WRITE` is enabled, the driver must handle writes that don't start or end on a write-block boundary. This requires read-modify-write:

1. Read the existing block into a temp buffer
2. Overlay the caller's data at the correct offset
3. Write the full block back

This is fiddly — get the aligned case working first, then add unaligned support.

## Cache Considerations

NVMCTRL has a read cache (controlled by CTRLB.CACHEDIS and CTRLB.READMODE). After writing or erasing flash:

- The cache may contain stale data
- Issue INVALL (Invalidate All Cache Lines) command after write/erase operations
- Or disable cache entirely during flash operations

```c
nvm->NVMCTRL_CTRLA = NVMCTRL_CTRLA_CMD_INVALL | NVMCTRL_CTRLA_CMDEX_KEY;
```

## User Row (Auxiliary Space)

A separate 512-byte area at a fixed address (SOC_NV_USERROW_BASE_ADDR) for configuration data. It has different write rules:
- Only writable via quad-word (16-byte) blocks
- Erase via EAR (Erase Auxiliary Row) command
- Write via WAP (Write Auxiliary Page) command
- Contains calibration data at known offsets — **do not overwrite these**

If implementing `ex_op` for user row access, document which offsets are reserved for calibration.

## Zephyr Flash API Mapping

| Zephyr API | NVMCTRL Action |
|-----------|---------------|
| `read()` | Direct memory read from flash address (no command needed) |
| `write()` | PBC → load page buffer → WP/WQW command |
| `erase()` | Set ADDR → ER command, repeat per block |
| `get_parameters()` | Return write_block_size, erase_value (0xFF) |
| `get_size()` | Return total flash size from DTS or PARAM register |
| `page_layout()` | Return array of {page_size, page_count} |

## Thread Safety

Use a spinlock (not mutex) for flash operations — flash writes can happen from ISR context in some scenarios, and mutex can't be held across ISR boundaries.

```c
struct flash_mchp_data {
    struct k_spinlock lock;
};

/* In write/erase: */
k_spinlock_key_t key = k_spin_lock(&data->lock);
/* ... flash operation ... */
k_spin_unlock(&data->lock, key);
```

## Generation Differences

| Feature | PIC32CM JH (older) | PIC32CZ CA (newer) |
|---------|-------------------|-------------------|
| Command register | CTRLA | CTRLB |
| Write modes | MAN, ADW, AQW, AP | Similar but check header |
| Erase command | ER (Erase Row) | EB (Erase Block) |
| Erase block size | Row-based | Block-based (8KB typical) |
| CTRLB fields | RWS, MANW, READMODE, CACHEDIS | May differ — read header |

Always check your specific DFP header. The command encoding and register layout shift between generations.
