# zmk-keyboard-eidolon

ZMK firmware module for the [Eidolon](https://github.com/jonnyirwin/eidolon) — a 30-key column-staggered wireless split keyboard. Kailh Choc v1 switches, one Seeed XIAO nRF52840 per half, both halves fully independent wireless units (no TRRS cable). The left half is the BLE central: it holds the keymap, talks to the host, and receives key events from the right half.

```
        ring  mid  idx                 idx  mid  ring
pinky   ring  mid  idx  inner   inner  idx  mid  ring  pinky
pinky   ring  mid  idx  inner   inner  idx  mid  ring  pinky
                   thumb thumb   thumb thumb
```

Pinky splayed 5°, ring 3°, thumbs −8°; 18 × 17 mm Choc spacing.

## Firmware artifacts

Every push builds the matrix in `build.yaml` via GitHub Actions (`zmkfirmware/zmk` `build-user-config` workflow, ZMK `v0.3`). Download the zip from the run's *Artifacts* section:

| Artifact | Flash to | Purpose |
|----------|----------|---------|
| `eidolon_left` | left half | central — keymap, host connection |
| `eidolon_right` | right half | peripheral |
| `eidolon_left_studio` | left half | central with [ZMK Studio](https://zmk.dev/docs/features/studio) support — flash *instead of* `eidolon_left` to edit the keymap live over USB |
| `settings_reset` | either half | wipes stored settings incl. BLE bonds — see [troubleshooting](#troubleshooting) |

There is no Studio build for the right half: ZMK Studio connects to the central, the only half that evaluates the keymap.

## Flashing

1. Plug the XIAO into USB and double-tap its reset button — it mounts as a USB drive named `XIAO-SENSE`.
2. Copy the matching `zmk.uf2` (rename artifacts to keep left/right apart) onto the drive.
3. The board reboots into the new firmware automatically.

## First-time pairing

1. Flash both halves, power both on near each other. The halves bond to each other automatically (left = central, right = peripheral).
2. Pair the left half with your host via the OS Bluetooth menu — it advertises as **Eidolon**.
3. Five host profiles are available; select with `BT_SEL 0–2` (nav layer, bottom-left) or add bindings for profiles 3–4.

## Default keymap

Adapted from the in-tree [hummingbird](https://github.com/zmkfirmware/zmk/tree/main/app/boards/shields/hummingbird) shield, which has the identical column structure (2-key pinky/inner columns, 3-key ring/middle/index, 2 thumbs per half) — bindings carry over column-for-column.

**Default layer**

```
       W   E   R                 U   I   O
  Q    S   D   F    T       H    J   K   L    P
  A    X   C   V    G       N    M   ,   .    '
            TAB  RET       SPC  BSPC
```

- Homerow-style mods (hold): `GUI` on A / `'`, `ALT` on S / L, `CTRL` on D / K, `SHIFT` on F / J.
- Missing letters are bottom-row combos: `Z` = X+C, `B` = C+V, `Y` = M+`,`, `/` = `,`+`.`.
- Thumb holds: TAB → **Nav**, SPC → **Num**, BSPC → **Sym**.

**Nav** — arrows/HOME/END/PG on the right hand; ESC/DEL on right thumbs. The left hand carries radio controls: `BT_CLR`, `BT_SEL 0–2`, `OUT_TOG` (USB↔BLE) across the bottom row, and `&studio_unlock` on the right pinky home key.

**Num / Sym** — numpad-style digits (shifted symbols on Sym) on the left hand, brackets on the pinky/inner keys, `0`/`-` (`)`/`_`) on the left thumbs.

Customize by editing `boards/shields/eidolon/eidolon.keymap`, or flash the Studio build and use [ZMK Studio](https://zmk.dev/docs/features/studio) (unlock via the nav-layer binding first).

## Using as a module

To keep your keymap in a personal [zmk-config](https://zmk.dev/docs/user-setup) instead, add this repo to `config/west.yml`:

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: jonnyirwin
      url-base: https://github.com/jonnyirwin
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: v0.3
      import: app/west.yml
    - name: zmk-keyboard-eidolon
      remote: jonnyirwin
      revision: main
  self:
    path: config
```

then reference `shield: eidolon_left` / `eidolon_right` with `board: seeeduino_xiao_ble` in that repo's `build.yaml`, and override the keymap with a `config/eidolon.keymap`.

The shield also builds against ZMK `main` (verified 2026-06), where the Zephyr hardware-model-v2 migration renamed the board: use `board: xiao_ble` there instead of `seeeduino_xiao_ble`. If you track `main`, pin a known-good commit SHA in `west.yml` rather than the branch so `west update` can't break your builds.

## Building locally

With docker (uses the official ZMK build image):

```sh
mkdir -p /tmp/zmk-ws/config
cat > /tmp/zmk-ws/config/west.yml <<'EOF'
manifest:
  remotes: [{ name: zmkfirmware, url-base: https://github.com/zmkfirmware }]
  projects: [{ name: zmk, remote: zmkfirmware, revision: v0.3, import: app/west.yml }]
  self: { path: config }
EOF

docker run --rm -v /tmp/zmk-ws:/work -v "$PWD":/module:ro -w /work \
  zmkfirmware/zmk-build-arm:stable bash -c '
    west init -l config && west update --narrow -o=--depth=1 &&
    west build -p -s zmk/app -d build_left  -b seeeduino_xiao_ble -- -DSHIELD=eidolon_left  -DZMK_EXTRA_MODULES=/module &&
    west build -p -s zmk/app -d build_right -b seeeduino_xiao_ble -- -DSHIELD=eidolon_right -DZMK_EXTRA_MODULES=/module'
```

Firmware lands in `/tmp/zmk-ws/build_*/zephyr/zmk.uf2`.

## Hardware / wiring reference

COL2ROW diode matrix, identical nets on both (mirrored) halves:

| Net | Role | XIAO pin |
|-----|------|----------|
| C0–C4 | columns, pinky → inner | D0–D4 |
| R0–R3 | rows: top, home, bottom, thumb | D5–D8 |

Because the halves are true mirrors (not one reversible PCB), the right half scans the same local columns but maps them reversed in the matrix transform via `col-offset = <5>` (`eidolon_right.overlay`).

Power: battery + → slide switch → XIAO `B+`; battery − → GND. Battery voltage sensing is built into the `seeeduino_xiao_ble` board definition — no shield config needed. Deep sleep is enabled (`CONFIG_ZMK_SLEEP=y`, 15 min idle default); any keypress wakes the half.

PCB sources (Ergogen) and case (OpenSCAD) live in the hardware repo: <https://github.com/jonnyirwin/eidolon>.

## Repository layout

```
boards/shields/eidolon/
  eidolon.dtsi            kscan matrix + matrix transform (shared)
  eidolon-layouts.dtsi    physical layout (key positions for ZMK Studio)
  eidolon_left.overlay    left half
  eidolon_right.overlay   right half (col-offset)
  eidolon.keymap          default keymap
  eidolon.conf            shared config (deep sleep)
  Kconfig.shield          shield definitions
  Kconfig.defconfig       keyboard name, split roles
  eidolon.zmk.yml         shield metadata
build.yaml                GitHub Actions build matrix
config/west.yml           west manifest (pins ZMK v0.3)
zephyr/module.yml         Zephyr module definition
```

## Troubleshooting

- **Halves stop talking to each other / phantom host pairing**: flash `settings_reset` to *both* halves, then re-flash the normal firmware to both and re-pair from scratch ([docs](https://zmk.dev/docs/troubleshooting/connection-issues)).
- **Host sees "Eidolon" but nothing types**: you're probably paired to a stale profile — `BT_CLR` (nav layer) clears the active profile, then re-pair.
- **Enter bootloader without the keymap**: double-tap the XIAO's physical reset button.

## License

[MIT](LICENSE). Default keymap adapted from the MIT-licensed ZMK hummingbird shield (© ZMK Contributors).
