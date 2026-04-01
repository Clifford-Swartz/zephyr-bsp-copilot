# UART Playbook — Microchip SERCOM

## Register Overlay

Access UART registers via the SERCOM union:
```c
sercom_usart_int_registers_t *usart = &cfg->regs->USART_INT;
```

## TXPO/RXPO Pin Mapping

| TXPO | TX Pad | Notes |
|------|--------|-------|
| 0 | PAD[0] | Most common |
| 1 | PAD[2] | |
| 2 | PAD[0] | With RTS/CTS flow control |
| 3 | PAD[0] | With RS485 |

RXPO selects which pad is RX (0-3). Must be consistent with pin function from ATDF.

**This is the #1 source of UART bring-up failures.** Always verify TX/RX pin assignments against the board schematic. The debugger CDC virtual COM port pins are NOT guessable — they vary per board layout.

## Console UART Setup

For the board to have console output, three things must be correct in the board DTS:

1. **UART node enabled** with correct SERCOM instance, pinctrl, and baud rate
2. **`chosen` node** must include:
   ```dts
   chosen {
       zephyr,console = &sercomN;
       zephyr,shell-uart = &sercomN;
   };
   ```
3. **Board defconfig** must include `CONFIG_SERIAL=y` and `CONFIG_CONSOLE=y`

## Baud Rate Calculation

UART baud uses a different formula than I2C. For asynchronous mode with 16x oversampling:

```c
/* BAUD = 65536 * (1 - 16 * fBAUD / fREF) */
uint64_t ratio = ((uint64_t)16 * baudrate * 65536) / fref;
uint16_t baud = 65536 - (uint16_t)ratio;
```

For arithmetic mode (CTRLA.SAMPR = 0x1):
```c
/* BAUD = fREF / (16 * fBAUD) - 1 */
uint16_t baud = fref / (16 * baudrate) - 1;
```

Check which sample rate modes your SERCOM supports in the DFP header.

## Interrupt vs Polled

For console output, polled TX is sufficient (write DATA, wait for DRE flag). For shell and high-throughput use, implement interrupt-driven TX/RX:

- **DRE** (Data Register Empty): TX ready for next byte
- **RXC** (Receive Complete): byte received in DATA
- **TXC** (Transmit Complete): all bytes shifted out (useful for RS485 direction control)
- **ERROR**: frame error, parity error, buffer overflow

## Flow Control

For RTS/CTS hardware flow control:
- Set TXPO=2 in CTRLA
- Configure RTS and CTS pins via pinctrl
- The SERCOM automatically deasserts RTS when the RX buffer is full
