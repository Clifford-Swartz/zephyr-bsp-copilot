# Zephyr BSP Starter Kit

A Claude Code starter kit for engineers bringing up new Microchip MCUs on Zephyr RTOS.

## What This Is

This repo contains a `CLAUDE.md` prompt and reference materials that turn Claude Code into a guided BSP development assistant. Instead of a template pipeline that generates boilerplate, this takes a different approach: **Claude works alongside the engineer**, making hardware-specific decisions, writing drivers from register-level documentation, and teaching Zephyr architecture as it goes.

## Why Not a Template Pipeline?

Template-based generators work when:
- The new chip's registers are compatible with existing upstream drivers
- You just need SoC/board scaffolding with the right addresses plugged in

They break when:
- The chip uses a different register header style (DFP vs ASF)
- New drivers must be written from scratch
- Pin assignments require board schematic knowledge
- Clock trees differ from existing families

For Microchip's newer chips (PIC32CM, PIC32CK, and any DFP-based part), the register headers are incompatible with existing `atmel,sam0-*` Zephyr drivers. A template can get the memory map right but produces drivers that won't compile. Claude can read the DFP headers and write working drivers. The same workflow also applies to ASF-based parts (SAM D/L/C, SAM D5x/E5x) where existing drivers usually need only new compatible strings and board scaffolding.

## How to Use

### Prerequisites
- [Claude Code](https://claude.ai/claude-code) installed
- Zephyr workspace set up (`west init` / `west update`)
- Target board on hand for hardware testing
- Board schematic or user guide (for pin assignments)

### Setup
1. Clone this repo into your Zephyr workspace or anywhere accessible
2. Open Claude Code in your Zephyr workspace directory
3. Claude will pick up the `CLAUDE.md` and reference materials automatically

### Workflow
Tell Claude what chip and board you're targeting. It will walk you through:

1. **Reconnaissance** — Parse the ATDF file, identify CPU core, memory map, peripherals, pin mux
2. **SoC Foundation** — Create SoC definition files (DTS, Kconfig, soc.h, soc.yml)
3. **Console UART** — First peripheral: get serial output working for debugging
4. **Peripheral Drivers** — Add I2C, SPI, WDT, etc. one at a time with hardware testing
5. **Upstream Preparation** — Format code to meet Zephyr maintainer standards

At each step, build and test before moving on.

## Reference Materials

| File | Contents |
|------|----------|
| [CLAUDE.md](CLAUDE.md) | Main prompt — phases, driver patterns, architecture guidance |
| [references/upstream-checklist.md](references/upstream-checklist.md) | Pre-submission checklist extracted from real maintainer reviews |
| [references/driver-patterns.md](references/driver-patterns.md) | DFP vs ASF explanation, SERCOM G1 driver templates, binding YAML patterns |
| [references/pr-intelligence.md](references/pr-intelligence.md) | Top 10 PR rejection reasons with maintainer quotes and fixes |

## What Claude Handles vs What You Handle

### Claude handles:
- Mechanical code generation (DTS nodes, Kconfig files, driver boilerplate)
- Register-level driver implementation from DFP headers
- ATDF parsing for addresses, IRQs, pin functions
- Zephyr build system integration (CMakeLists, Kconfig wiring)
- Formatting to upstream standards (TAB 8, SPDX, commit messages)

### You handle:
- Board schematic verification (pin assignments, debug UART, LEDs)
- Hardware testing (flash, boot, probe peripherals)
- Architectural decisions (which peripherals to support, clock configuration choices)
- Upstream review responses (maintainer questions about design choices)
- DCO sign-off (only humans sign the Developer Certificate of Origin)

## License

Apache-2.0
