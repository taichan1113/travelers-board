# ZMK MS88SF21 Keyboard Module

This repository defines a custom ZMK board for the MS88SF21 nRF52840 module.
The board is a single-piece ortholinear keyboard with a logical 7x6 matrix and a physical 4-column symmetric layout of 12 / 12 / 12 / 6 keys.

## Board details

- MCU: nRF52840 (MS88SF21 module)
- Matrix: 7 rows x 6 columns (42 keys)
- Physical layout: 4 columns, 12 / 12 / 12 / 6 keys per column, left/right symmetric
- Diode direction: col2row
- Encoder: none
- OLED: none
- Pointing device: none
- GPIO pin assignments: dummy pins are provided in the DTS and can be updated later

## Files

- `boards/arm/ms88sf21/board.cmake`
- `boards/arm/ms88sf21/Kconfig.board`
- `boards/arm/ms88sf21/Kconfig.defconfig`
- `boards/arm/ms88sf21/ms88sf21.dts`
- `boards/arm/ms88sf21/ms88sf21_defconfig`
- `build.yaml`
- `config/ms88sf21.keymap`
- `config/west.yml`
- `.github/workflows/build.yml`
- `zephyr/module.yml`

## Usage

This module can be used as a custom ZMK keyboard board definition.
Update the pin mappings in `boards/arm/ms88sf21/ms88sf21.dts` once the actual matrix wiring is available.

From the workspace root, initialize this config and build it:

```sh
just init config/zmk-config-travelers_board
just build ms88sf21
```

This repository is based on `zmkfirmware/unified-zmk-config-template`.
The top-level `build.yaml` contains the `ms88sf21` board target for GitHub Actions and local builds.

## Notes

- [ZMK firmware を SWD 経由で HEX 書き込みする手順](docs/swd-hex-flashing.md)
