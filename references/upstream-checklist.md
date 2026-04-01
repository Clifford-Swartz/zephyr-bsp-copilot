# Zephyr Upstream BSP Checklist

Use this checklist before submitting a PR to `zephyrproject-rtos/zephyr`. Every item has been flagged by maintainers in real reviews.

## PR Structure

- [ ] **PR 1 is boot-only:** SoC + board + clock + GPIO + pinctrl + UART console. Nothing else.
- [ ] **Commit order:** SoC commits before board commits. Maintainers will reject out-of-order commits.
- [ ] **No scope creep:** No extra boards, no unrelated driver fixes, no HAL changes in the same PR.
- [ ] **HAL changes in separate repo:** SoC headers and DFP packs go in `hal_microchip`, not in the Zephyr tree. Reference the HAL PR in `west.yml` using `pull/<N>/head` revision — never your personal fork.

## File Formatting

### Devicetree (.dts / .dtsi)
- [ ] **TAB 8 indentation** (not TAB 4 — this kills PRs)
- [ ] **Single space around `=`** — `status = "okay";` not `status	= "okay";`
- [ ] **Blank line between peer nodes** (no squashed blocks)
- [ ] **No blank line before closing `}`**
- [ ] **C-style comments only** (`/* */`) — never `//`
- [ ] **No commented-out code** (remove it entirely)
- [ ] **Node names describe hardware accurately** ("these are not LEDs")

### Kconfig
- [ ] **TAB 8 indentation** under config blocks
- [ ] **100-character line limit**
- [ ] **`configdefault`** for overriding existing defaults — NOT `config ... default y`
- [ ] **No redundant settings** (don't enable things that are already default `y`)
- [ ] **Help text:** one tab + two spaces indent
- [ ] **`BOARD_*` and `BOARD_*_*` symbols are auto-created** — do NOT define their type

### C Source
- [ ] **Linux kernel style** (8-char tabs, snake_case identifiers)
- [ ] **`#define DT_DRV_COMPAT` before any `#include`** — Zephyr convention, common review nit
- [ ] **Don't `#include` headers that don't exist yet** — if a dependency (e.g., clock driver) hasn't been written, note it as a TODO comment rather than including a nonexistent header
- [ ] **No dynamic allocation** in drivers
- [ ] **Inclusive terminology** (no master/slave — use controller/target, host/device)
- [ ] Passes `checkpatch` and `clang-format`

### All Files
- [ ] **SPDX header on every file:**
  - C/DTS: `/* SPDX-License-Identifier: Apache-2.0 */`
  - Kconfig/YAML/CMake: `# SPDX-License-Identifier: Apache-2.0`
- [ ] **Copyright line:** `Copyright (c) 2025 Your Name` — clean, no extra text

## Devicetree Content

- [ ] **`chosen` nodes include:** `zephyr,console`, `zephyr,sram`, `zephyr,flash`
- [ ] **Pinctrl in separate `-pinctrl.dtsi` file**
- [ ] **No pinmux conflicts** between enabled peripherals (add comments if potential conflicts exist)
- [ ] **Peripheral nodes `status = "disabled"` by default** in SoC DTSI — boards enable what they use
- [ ] **Compatible strings have matching binding YAMLs**

## Kconfig Content

- [ ] **defconfig is minimal** — only settings that differ from defaults
- [ ] **`Kconfig.<board>`** selects `SOC_*` symbols mapped to board qualifiers
- [ ] **`Kconfig.defconfig`** wraps everything in `if BOARD_<NAME>` / `endif`

## Board Files

- [ ] **`board.yml`** — vendor matches `dts/bindings/vendor-prefixes.txt`
- [ ] **`board.cmake`** — configures flash/debug runner (OpenOCD, J-Link, etc.)
- [ ] **Twister YAML** — correct `flash:`, `ram:`, `supported:` list, `toolchain:` list
- [ ] **`doc/index.rst`** — uses `doc/templates/board.tmpl` template
- [ ] **`doc/<board>.webp`** — clean photo, no drawn overlays, clipped edges

## Commits

- [ ] **Format:** `subsystem: area: short summary` (under 72 chars)
  - `soc: microchip: pic32cm: add PIC32CM JH SoC support`
  - `boards: microchip: add PIC32CM JH01 Curiosity Nano`
- [ ] **`Signed-off-by:`** on every commit (DCO requirement — legal name + email)
- [ ] **`west.yml` updated** if HAL dependency changed

## Pre-Submit Validation

```bash
# Compliance checks
./scripts/ci/check_compliance.py -c HEAD~1..HEAD

# Build hello_world
west build -b <board> samples/hello_world -p always

# Twister build test
west twister -p <board> -s tests/drivers/build_all/gpio

# Check for stray files
git status
git diff --stat origin/main..HEAD
```

## Common Rejection Patterns

| Mistake | Maintainer Quote |
|---------|-----------------|
| PR too large | "split this PR into an initial PR with basic SoC/board support" |
| Wrong commit order | "commits are out of order, a board cannot be added before the requirements" |
| TAB 4 not TAB 8 | "Is your editor using TAB 4 instead 8? Check all files" |
| `config default y` | "use a configdefault here" |
| C++ comments in DTS | "C++ style comment" |
| Personal fork in west.yml | "point to the PR in the upstream using the pull/N/head revision" |
| Stray files | "Something very wrong with this PR, likely stray files" |
| Bad board image | "replace picture with one without any drawn overlays" |
