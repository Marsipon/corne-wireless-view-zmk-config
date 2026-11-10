# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

ZMK firmware configuration for the **Eyelash Corne** — a 42-key wireless split Corne keyboard
(nRF52840 / nice_nano_v2-class halves) with nice_view displays, WS2812 RGB underglow, and PWM
backlight. The encoder and center D-pad supported by the original board are not populated. There is
no application source here; this repo customizes upstream ZMK via Devicetree (`.dtsi`/`.dts`/`.keymap`),
Kconfig (`.conf`/`defconfig`), and a west manifest.

## Build

Firmware is built by **GitHub Actions**, not locally. Pushing (or opening a PR) triggers
`.github/workflows/build.yml`, which calls ZMK's reusable `build-user-config.yml`. Download
the resulting `.uf2` artifacts from the Actions run and flash each half (double-tap reset to
enter the bootloader, copy the `.uf2` to the mass-storage device).

- `build.yaml` is the build matrix — it lists every firmware artifact produced:
  - `eyelash_corne_right` + `nice_view`
  - `eyelash_corne_left` + `nice_view` with ZMK Studio enabled (`studio-rpc-usb-uart` snippet,
    `-DCONFIG_ZMK_STUDIO=y`), output as `eyelash_corne_studio_left`
  - `nice_nano_v2` + `settings_reset` (flash this to clear stored settings / re-pair BLE)
  - To debug over USB serial, uncomment the `zmk-usb-logging` snippet on a board entry.
- `config/west.yml` pins ZMK at `v0.3.0`. Bumping firmware versions happens here.

## Keymap

**`config/eyelash_corne.keymap`** is the active user keymap that ships in the firmware. Its only
base layer is **Graphite** (no QWERTY). The unused, divergent board fallback keymap was removed so
there is only one keymap to maintain.

`zephyr/module.yml` sets `board_root: .`, so the in-repo `boards/arm/eyelash_corne/` *is* the board
definition.

## Keymap architecture (`config/eyelash_corne.keymap`)

The header comment in the file is the source of truth; this summarizes it. **OS keyboard layout is
assumed to be Swedish ISO on both macOS and Windows** — that assumption drives every symbol keycode
(see below). The file header comment block also documents the layers and thumbs; keep it in sync.

### Layers (index → name)
`0` BASE (Graphite) · `1` WIN (flag; swaps home-row Ctrl/GUI) · `2` NAV · `3` NUM · `4` SYM ·
`5` FUN · `6` SYMWIN (Windows `{ } \ |`) · `7` ACC (accents) · `8` NAVWIN (Windows NAV mods + Ctrl zoom).

- **WIN must stay at layer 1**, right above BASE. It is not transparent (it overrides home-row
  positions 13, 15, 20, 22), so any layer *below* it would be hidden at those keys on Windows
  (this bit us once: SYM showed `n`/`t`/`a`/`i` instead of `+ = )`).
- **FUN** (NUM+SYM), **SYMWIN** (SYM+WIN) and **NAVWIN** (NAV+WIN) are conditional layers. See the
  `conditional_layers` node. Renumbering any layer means updating: the `&lt_bal`/`&mo`/`&tog`
  bindings on BASE and FUN, the `conditional_layers` `if-layers`/`then-layer`, and the header comment.
- The WIN overrides keep the shortcut modifier on the middle finger (Cmd on macOS = `T`/`A`; Ctrl on
  Windows = `T`/`A`).

### Key positions
Positions are absolute indices into the 42-key matrix (0–41), numbered left-to-right, top row
first. Rows: `0–11`, `12–23`, `24–35`, then `36–41` for the thumbs
(L-outer, L-mid, L-inner, R-inner, R-mid, R-outer).
**Any key add/move/reorder must update**: `hml`/`hmr` `hold-trigger-key-positions`, the combo
`key-positions`, and the SYMWIN overlay (which must line up with the SYM positions it overrides).

### Thumbs
```
L-outer  &lt_bal 7 ESC     tap Esc      / hold ACC (accents)
L-mid    &mo 2             hold NAV
L-inner  &lt_bal 4 SPACE   tap Space    / hold SYM
R-inner  &kp BSPC
R-mid    &mo 3             hold NUM
R-outer  &kp ENTER
```

### Custom behaviors
- **`hml` / `hmr`** — home-row mods (hold-tap, `balanced`, 280ms, `require-prior-idle-ms 150`,
  `hold-trigger-on-release`, positional via opposite-hand `hold-trigger-key-positions`). No
  `hold-while-undecided` (it misfired on rolls). Full set on both hands, macOS mapping: left
  `N R T S` = Ctrl Alt Cmd Shift, right `H A E I` = Shift Cmd Alt Ctrl. Windows (layer 1) swaps
  Ctrl/GUI on N/T/A/I. Shift lives on the index home keys; pos 24 is `&caps_word` (its
  `continue-list` keeps it alive through the Swedish `_`). No sticky shift anywhere.
- **`lt_bal`** — layer-tap, `balanced` (`<&mo>, <&kp>`). Used for the Space/SYM and Esc/ACC thumbs;
  the default `&lt` is tap-preferred and sent Space+letter when a symbol was pressed quickly.
- **`sqt`** — mod-morph: tap `'`, Shift `"`. Needs BOTH `mods` *and* `keep-mods` (without
  `keep-mods` the morph strips Shift and emits a bare `2`). Lives on BASE pos 6 and SYM pos 18.
- **`grave` / `tilde` / `caret`** — macros for the Swedish **dead keys** `` ` `` `~` `^`: they send
  the dead key then SPACE to emit a literal. **Verified on both Windows and macOS.**
- **Accents (layer 7)** — `ä/ö/å` (`SQT`/`SEMI`/`LBKT`) on the BASE `A`/`O`/`E` key positions (20, 8,
  21); hold the L-outer thumb. Replaced the old vowel tap-dances (which collided with doubled
  letters like "book").

### Swedish symbols — IMPORTANT
Because the OS layout is Swedish, **do not use the named US symbol keycodes** (`AMPS`, `LBRC`,
`PIPE`, …) — they're defined as US Shift/AltGr combos and emit the wrong glyph. Always use the
explicit Swedish combo: e.g. `{` = `RA(N7)`, `[` = `RA(N8)`, `@` = `RA(N2)`, `$` = `RA(N4)`,
`/` = `LS(N7)`, `?` = `LS(MINUS)`, `;` = `LS(COMMA)`, `-` = `FSLH`, `+` = `MINUS`. `<>|` use
`NON_US_BSLH` (`<` = `NON_US_BSLH`, `>` = `LS(NON_US_BSLH)`, `|` = `RA(NON_US_BSLH)`).

`{ } \ |` differ between Windows and macOS Swedish. SYM holds the **macOS** values (the default OS);
holding NUM+SYM and tapping the physical X key toggles Windows mode (`&tog 1` on FUN), activating
the **WIN** flag, which turns on **SYMWIN** (layer 6, overrides `\ { } |` at positions 30–33) and
**NAVWIN**. Layer toggles are not persistent, so a keyboard reboot returns to macOS mode.

### SYM layer cheat-sheet (hold L-inner thumb)
```
left (pinky→inner)        right (inner→pinky, outer)
 !  &  ?  ~  ^             `  [  ]  <  >  :
 +  -  =  *  /             '" (  )
 #  $  %  @  _             \  {  }  |
```
`'"` is the `sqt` morph. `` ` `` `~` `^` are dead-key macros. `\ { } |` are the macOS values; the
SYMWIN overlay replaces them on Windows.

### Other layers / hardware glue
- **NUM** (hold R-mid): number pad on the left hand (`7 8 9 / 4 5 6 / 1 2 3 / 0`) plus `* = . + : -`.
- **NAV** (hold L-mid): vim-style arrows on the right home row (`← ↓ ↑ →` index→pinky),
  Home/End/PgUp/PgDn, Delete, and zoom in/reset/out stacked in the
  inner column (Cmd on macOS, Ctrl via NAVWIN). The left home row has plain instant mods
  (macOS: Ctrl Alt Cmd Shift; Windows: GUI Alt Ctrl Shift) for Shift/Ctrl+arrow selection.
  Copy/paste/undo are ordinary chords: hold the home-row Cmd/Ctrl (`A` or `T`) + `C`/`V`/`X`/`Z`.
- **FUN** has `&ext_power EP_ON` on the second right-hand home-row key as a display/rail recovery key.
- **Combo** `softoff` (positions `1 14 25`) → `&soft_off` powers the board down.

## Hardware definition (`boards/arm/eyelash_corne/`)

- `eyelash_corne.dtsi` — shared hardware: 4×7 GPIO matrix `kscan0` (col2row), `default_transform`
  (the 14×4 matrix → 42-key physical mapping), WS2812 `led_strip` (21 LEDs on spi3), PWM backlight,
  battery sensing, flash partitions, and nice_view on spi0.
- `eyelash_corne_left.dts` / `eyelash_corne_right.dts` — per-half definitions. The right half
  applies `col-offset = <7>` so the halves share one column space.
- `eyelash_corne-layouts.dtsi` — the ZMK Studio physical layout (key positions/rotation for the
  on-screen editor). Must stay consistent with `default_transform`.
- `*_defconfig`, `Kconfig.board`, `Kconfig.defconfig`, `board.cmake`, `*.yaml`, `*.zmk.yml` —
  board registration/metadata.

## Firmware-wide settings (`config/eyelash_corne.conf`)

Kconfig flags applied to all builds: 1-hour idle sleep, RGB underglow (off at start, auto-off on
idle, but not allowed to disable the external-power rail shared with nice_view), NKRO, backlight
(100% at start, auto-off on idle), soft-off, +8dB BLE TX power, and 8ms debounce. Toggle
keyboard-wide features here.

`config/eyelash_corne.json` is the ZMK Studio keymap/layout snapshot — generally regenerated by
Studio rather than hand-edited.
