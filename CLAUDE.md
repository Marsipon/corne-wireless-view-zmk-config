# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

ZMK firmware configuration for the **Eyelash Corne** — a wireless split Corne keyboard
(nRF52840 / nice_nano_v2-class halves) with nice_view displays, an EC11 rotary encoder,
WS2812 RGB underglow, PWM backlight, and mouse/pointing support. There is no application
source here; this repo customizes upstream ZMK via Devicetree (`.dtsi`/`.dts`/`.keymap`),
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
- `config/west.yml` pins dependencies: ZMK at `v0.3.0` and the out-of-tree board module
  `eyelash_corne` from `github.com/a741725193/zmk-new_corne@main`. Bumping firmware versions
  happens here.

## Layout: two keymaps, two roles

There are **two** `eyelash_corne.keymap` files and the distinction matters:

- **`config/eyelash_corne.keymap`** — the **active** user keymap that ships in the firmware.
  Base layer is a **Graphite** alpha layout, with a **QWERTY alternate base** on layer 1 reachable
  by toggle. This is almost always the file to edit.
- **`boards/arm/eyelash_corne/eyelash_corne.keymap`** — the board's **default/fallback** keymap
  (a separate QWERTY) that lives with the board definition. Unrelated to the layer-1 QWERTY above;
  edit only when changing the board's shipped default, not for personal layout changes.

`build.yaml` sets `board_root: .`, so the in-repo `boards/arm/eyelash_corne/` *is* the board
definition (it does not come from the upstream module despite the same name).

## Keymap architecture (`config/eyelash_corne.keymap`)

The header comment in the file is the source of truth; this summarizes it. **OS keyboard layout is
assumed to be Swedish ISO on both macOS and Windows** — that assumption drives every symbol keycode
(see below). The file header comment block also documents the layers and thumbs; keep it in sync.

### Layers (index → name)
`0` BASE (Graphite) · `1` QWERTY (alt base) · `2` NAV · `3` NUM · `4` SYM · `5` FUN ·
`6` MAC (flag, all-transparent) · `7` SYMMAC (macOS bracket overrides) · `8` ACC (accents).

- **QWERTY** is an *alternate base* deliberately placed at layer 1 — **below** the functional
  layers (2–8) — so `&tog 1` (on FUN) overlays it onto Graphite while NAV/NUM/SYM/etc. still win on
  top. Adding any new alternate base must follow this rule (low index, below functional layers),
  or it will clobber the momentary layers.
- **FUN** is reached as a conditional layer (hold NUM+SYM); **SYMMAC** is conditional on SYM+MAC.
  See the `conditional_layers` node. Renumbering any layer means updating: the `&lt`/`&mo`/`&tog`
  bindings on BASE/QWERTY thumbs and FUN, the `conditional_layers` `if-layers`/`then-layer`, and
  the header comment.

### Key positions
Positions are absolute indices into the 48-key matrix (0–47), numbered left-to-right, top row
first. Rows: `0–12` (13, incl. center UP), `13–27` (15, incl. center LEFT/ENTER/RIGHT), `28–41`
(14, incl. center SPACE/DOWN), `42–47` thumbs (L-outer, L-mid, L-inner, R-inner, R-mid, R-outer).
**Any key add/move/reorder must update**: `hml`/`hmr` `hold-trigger-key-positions`, the combo
`key-positions`, and the SYMMAC overlay (which must line up with the SYM positions it overrides).

### Thumbs (BASE and QWERTY share this layout)
```
L-outer  &lt 8 ESC        tap Esc      / hold ACC (accents)
L-mid    &lt_sk 2 LSHFT   tap ⇧(sticky)/ hold NAV
L-inner  &lt 4 SPACE      tap Space    / hold SYM
R-inner  &kp BSPC
R-mid    &lt_sk 3 RSHFT   tap ⇧(sticky)/ hold NUM
R-outer  &kp RALT         AltGr
```

### Custom behaviors
- **`hml` / `hmr`** — home-row mods (hold-tap, `balanced`, 280ms, `require-prior-idle-ms 150`,
  `hold-trigger-on-release`, positional via opposite-hand `hold-trigger-key-positions`). No
  `hold-while-undecided` (it misfired on rolls). Mods are **GACS minus shift**: GUI/ALT/CTRL on the
  left three home keys, CTRL on the right inner-index, GUI on the right pinky. Shift is *not* on the
  home row — it lives on the sticky thumbs, plus a plain `&kp LSHFT` on bottom-left (pos 28) for a
  held shift (e.g. CONSTANT_CASE).
- **`lt_sk`** — layer-tap whose tap is a **sticky** modifier: `bindings = <&mo>, <&sk>`. Used for
  the shift-on-tap / layer-on-hold thumbs. `balanced` so hold-and-type reliably reaches the layer.
- **`sqt`** — mod-morph: tap `'`, Shift `"`. Needs BOTH `mods` *and* `keep-mods` (without
  `keep-mods` the morph strips Shift and emits a bare `2`). Lives on BASE pos 7 and SYM pos 25.
- **`grave` / `tilde` / `caret`** — macros for the Swedish **dead keys** `` ` `` `~` `^`: they send
  the dead key then SPACE to emit a literal. **Windows behavior; macOS dead-key output is unverified.**
- **Accents (layer 8)** — `ä/ö/å` (`SQT`/`SEMI`/`LBKT`) on the BASE `A`/`O`/`E` key positions; hold
  the L-outer thumb. Replaced the old vowel tap-dances (which collided with doubled letters like
  "book"). Meaningful only over Graphite — in QWERTY those keys are different letters (but Swedish
  QWERTY has dedicated `å ä ö` anyway).

### Swedish symbols — IMPORTANT
Because the OS layout is Swedish, **do not use the named US symbol keycodes** (`AMPS`, `LBRC`,
`PIPE`, …) — they're defined as US Shift/AltGr combos and emit the wrong glyph. Always use the
explicit Swedish combo: e.g. `{` = `RA(N7)`, `[` = `RA(N8)`, `@` = `RA(N2)`, `$` = `RA(N4)`,
`/` = `LS(N7)`, `?` = `LS(MINUS)`, `;` = `LS(COMMA)`, `-` = `FSLH`, `+` = `MINUS`. `<>|` use
`NON_US_BSLH` (`<` = `NON_US_BSLH`, `>` = `LS(NON_US_BSLH)`, `|` = `RA(NON_US_BSLH)`).

`{ } \ |` differ between Windows and macOS Swedish. SYM holds the **Windows** values; toggling Mac
mode (`&tog 6` on FUN) activates the **SYMMAC** overlay (layer 7) which overrides only those four
positions (8, 9, 26, 40) with the macOS combos.

### SYM layer cheat-sheet (hold L-inner thumb)
```
left hand            right hand
 &  *  (  )           `  {  }  <  >  ·
 $  %  ^              ?  [  ]  '"  \  ;
 !  @  #             -  +  =  /  |  ~
```
`'"` is the `sqt` morph (tap `'`, Shift `"`). `` ` `` `~` `^` are the dead-key macros.

### Other layers / hardware glue
- **NUM** (hold R-mid): number pad on the left hand (`7 8 9 / 4 5 6 / 1 2 3 / 0`) plus `+ : -`.
- **NAV** (hold L-mid): right-hand arrows, mouse move/click in the center columns, copy/paste, Home/End.
- **Mouse/pointing**: `mmv`/`msc` tuning + input-processor scalers at the top of the file; `&mmv MOVE_*`
  / `&mkp` / `&msc SCRL_*` bindings live in the center columns of NAV/SYM/FUN.
- **Encoder** via `sensor-bindings`: volume on the bases, `scroll_encoder` (mouse scroll) elsewhere.
- **Combo** `softoff` (positions `1 15 29`) → `&soft_off` powers the board down.

## Hardware definition (`boards/arm/eyelash_corne/`)

- `eyelash_corne.dtsi` — shared hardware: 5×7 GPIO matrix `kscan0` (col2row), `default_transform`
  (the 14×5 matrix → physical key mapping), EC11 `left_encoder`, WS2812 `led_strip` (21 LEDs on
  spi3), PWM backlight, battery sensing, flash partitions, nice_view on spi0.
- `eyelash_corne_left.dts` / `eyelash_corne_right.dts` — per-half overrides. Left enables the
  encoder; right applies `col-offset = <7>` so the two halves share one column space.
- `eyelash_corne-layouts.dtsi` — the ZMK Studio physical layout (key positions/rotation for the
  on-screen editor). Must stay consistent with `default_transform`.
- `*_defconfig`, `Kconfig.board`, `Kconfig.defconfig`, `board.cmake`, `*.yaml`, `*.zmk.yml` —
  board registration/metadata.

## Firmware-wide settings (`config/eyelash_corne.conf`)

Kconfig flags applied to all builds: 1-hour idle sleep, RGB underglow (off at start, auto-off on
idle), NKRO, EC11, pointing + smooth scrolling, backlight, soft-off, +8dB BLE TX power, and
8ms debounce. Toggle keyboard-wide features here.

`config/eyelash_corne.json` is the ZMK Studio keymap/layout snapshot — generally regenerated by
Studio rather than hand-edited.
