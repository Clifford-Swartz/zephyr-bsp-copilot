# Zephyr BSP PR Intelligence

Distilled from real maintainer reviews on Zephyr BSP pull requests. Use this to avoid the most common rejection reasons.

## Maintainers to Know

These are the most active reviewers for SoC/board/driver PRs in the Microchip/Atmel area:

- **nordicjm** — Very active board/SoC reviewer. Strict on formatting, PR scope, and commit order.
- **nandojve** — Microchip/Atmel area maintainer. Catches HAL boundary issues and TAB width problems.
- **maass-hamburg** — Kconfig expert. Will flag `configdefault` vs `config default` issues.
- **asmellby** — Catches west.yml issues and C++ comment style in DTS.
- **JarmouniA** — Copyright header and Kconfig list issues.

## What Gets Merged Smoothly

1. **Small PRs.** Boot + console + GPIO only in the first PR. One peripheral per follow-up PR.
2. **Correct commit order.** SoC first → board second → drivers third.
3. **Minimal defconfig.** Only settings that differ from defaults.
4. **Clean formatting.** TAB 8, single space around `=` in DTS, no C++ comments.
5. **Complete docs.** Board doc using template, `.webp` photo, twister YAML.
6. **Working `west flash`.** `board.cmake` with correct runner configuration.
7. **CI green.** All compliance checks passing before requesting review.

## Top 10 Rejection Reasons (Ranked by Frequency)

### 1. PR Too Large
> "split this PR into an initial PR with basic SoC/board support (basically just what is needed for booting, logging and GPIOs) and leave the rest for smaller, incremental PRs."

**Fix:** First PR = SoC definition + board definition + clock + GPIO + pinctrl + UART. Everything else in follow-ups.

### 2. Wrong Tab Width
> "Is your editor using TAB 4 instead 8? Check all files to use correct indentation."

**Fix:** Set editor to TAB 8 for all Zephyr files. Check with `cat -A` or `expand -t 8`.

### 3. Commits Out of Order
> "commits are out of order, a board cannot be added before the requirements for the board have been added"

**Fix:** SoC commit → board commit → driver commits. Use `git rebase -i` to reorder if needed.

### 4. `config default y` Instead of `configdefault`
> "use a configdefault here"

**Fix:** In `Kconfig.defconfig`, use:
```kconfig
configdefault NET_L2_ETHERNET
    default y
```
Not:
```kconfig
config NET_L2_ETHERNET
    default y
```

### 5. DTS Formatting Issues
> "format non compliant... fix in whole PR"
> "missing newline gaps between nodes"
> "gap after = seems large, change to 1 space not a tab"

**Fix:** Single space around `=`. Blank line between peer nodes. No blank line before `}`. C-style comments only.

### 6. Personal Fork in west.yml
> "Please point to the PR in the upstream using the pull/95/head revision, don't introduce your personal remote."

**Fix:** Reference upstream HAL PRs with `revision: pull/<N>/head`, not your fork URL.

### 7. HAL Code in Zephyr Tree
> "Shouldn't this be at hal?"

**Fix:** SoC header definitions, DFP packs, and register-level includes go in `hal_microchip`. Only Zephyr-specific glue code (soc.h, Kconfig) goes in the Zephyr tree.

### 8. Stray Files
> "Something very wrong with this PR, likely stray files that should not have been added"

**Fix:** Run `git diff --stat origin/main..HEAD` and review every file. Remove anything not intentionally added.

### 9. Bad Board Image
> "replace picture with one without any drawn overlays, with outside space clipped"

**Fix:** Clean photo of the actual board. No annotations, arrows, or text overlays. Crop to board edges. Save as `.webp`.

### 10. Scope Creep
> "This board should not be part of the PR. It seems to be from another manufacturer and it is out of scope."
> "The change in this specific file should be on another PR. Drop it."

**Fix:** Only touch files directly related to your board/SoC. If you find a bug in an existing driver, fix it in a separate PR.

## PR Title and Commit Message Format

### PR Titles
```
soc: microchip: add PIC32CM JH SoC support
boards: microchip: add PIC32CM JH01 Curiosity Nano
drivers: serial: add SERCOM G1 UART driver
drivers: i2c: add SERCOM G1 I2C driver
```

### Commit Messages
```
soc: microchip: pic32cm: add PIC32CM JH SoC definition

Add SoC support for the Microchip PIC32CM JH family (Cortex-M0+,
48 MHz, up to 512 KB flash, 64 KB SRAM). Includes device tree,
Kconfig symbols, and clock controller driver.

Signed-off-by: Your Name <your@email.com>
```

### Key Rules
- Subject line under 72 characters
- Imperative mood ("add", not "added" or "adds")
- Body explains *what* and *why*, not *how*
- `Signed-off-by:` on every commit (DCO requirement)
- For AI-assisted code, add `Assisted-by:` tag but only humans sign DCO

## Vendor Prefix Rules

| Chip Family | Vendor Prefix | Directory |
|-------------|--------------|-----------|
| SAM3x, SAM4x, SAME70, SAMV71 | `atmel` | `boards/atmel/sam/`, `soc/atmel/sam/` |
| SAMD, SAML, SAMC (SAM0) | `atmel` | `boards/atmel/sam0/`, `soc/atmel/sam0/` |
| PIC32CM, PIC32CK, PIC32CX | `microchip` | `boards/microchip/`, `soc/microchip/` |

The vendor prefix must match an entry in `dts/bindings/vendor-prefixes.txt`.

## PIC32CX SG Case Study (PR #76503)

This PR was **closed** after extensive review. Key lessons:

1. PIC32CX SG is register-compatible with SAME54 → no new drivers needed, just SoC/board definitions + HAL additions.
2. SoC headers were placed in Zephyr tree instead of `hal_microchip` → rejected.
3. TAB 4 used throughout → formatting rejection on all files.
4. PR included too many changes at once → "split this PR".
5. Personal fork referenced in `west.yml` → rejected.

This PR is a roadmap of what NOT to do. Study it before submitting.
