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

## Files added

- `boards/arm/ms88sf21/board.cmake`
- `boards/arm/ms88sf21/Kconfig.board`
- `boards/arm/ms88sf21/ms88sf21.dts`
- `build.yaml`

## Usage

This module can be used as a custom ZMK keyboard board definition.
Update the pin mappings in `boards/arm/ms88sf21/ms88sf21.dts` once the actual matrix wiring is available.

If you use `unified-zmk-config-template` for GitHub Actions, the top-level `build.yaml` provides the board matrix for the workflow.
