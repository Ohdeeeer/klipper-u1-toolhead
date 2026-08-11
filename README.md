# Klipper with Snapmaker U1 toolhead support

Fork of [mainline Klipper](https://www.klipper3d.org/) that adds support for the
Snapmaker U1 toolhead for toolchangers. The toolhead is an AT32F415RC
("STM32F105 drop-in") MCU with an inductance-coil bed probe and a LIS2DW
accelerometer.

## Getting started

**[Build and bring up a U1 toolhead with a Raspberry Pi](docs/TOOLHEAD_ON_RASPBERRY_PI.md)**

## What this fork adds (vs. upstream Klipper)

Firmware (`src/stm32/`):

- **AT32F415RC / AT32F403A** MCU support (native AT32 clocking, USB-OTG,
  USBCANBUS, CAN, SPI, hard-PWM for the toolhead/main-board)
- **Inductance-coil bed probe** (`inductance_coil.c`): the U1's eddy-current
  bed-detection sensor (`pulse_endstop` pin type, frequency trigger)
- **Extended stepper protocol** so un-reflashed stock U1 toolhead firmware can
  be driven

Host (`klippy/`):

- `extras/inductance_coil.py` + `extras/probe_inductance_coil.py`: coil
  frequency sensor, `pulse_endstop`, and probe session/calibration
- `extras/lis2dw.py`: stock-toolhead-compatible LIS2DW accelerometer (SPI-only)
- `klippy/mcu.py`: `is_pulse_gpio` + `pulse_endstop` pin support
- `klippy/stepper.py`, `src/stepper.{c,h}`: extended `queue_step` /
  `config_stepper` for the stock toolhead firmware

## Configs / docs

- `config/sample-snapmaker-u1-toolhead.cfg`: U1 toolhead as a single
  `[extruder]` with the coil probe and LIS2DW
- [TOOLHEAD_ON_RASPBERRY_PI.md](docs/TOOLHEAD_ON_RASPBERRY_PI.md): build +
  bring-up guide for driving a U1 toolhead from a Raspberry Pi with this port
- `config/at32f415.config.resolved`: verified toolhead firmware `.config`
- `config/octopus-pro-v1.1.config.resolved`: example main-board `.config`
  (BTT Octopus Pro v1.1, STM32H723)

## Building

Same as upstream Klipper: `make menuconfig` (select the AT32F415/AT32F403A MCU
for toolheads) then `make`. See the setup guide for the exact `.config`.

## License

Same as upstream Klipper: GPLv3 (see `COPYING`).
