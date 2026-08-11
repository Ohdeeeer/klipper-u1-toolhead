# Snapmaker U1 Toolhead on a Raspberry Pi — Complete Build Guide

This document describes how to connect a **Snapmaker U1 toolhead** (the small
toolhead board with its own extruder, heater, fans, and bed-detection coil) to a
**Raspberry Pi**, and drive it with a **mainline Klipper port** (the
`mainline-port` tree that adds AT32 + inductance-coil + LIS2DW support).

It is written for a person or AI starting with zero context. All machine-specific
values (IP addresses, serial numbers, local file paths, usernames) are shown as
`<placeholders>` — use your own.

---

## 1. What you need

- A **Raspberry Pi** runs the ported **klippy** (host software) from the
  `mainline-port` tree.
- A **main control board** (currently a BTT **Octopus Pro v1.1**, STM32H723)
  provides X/Y/Z stepper motion and the main `[mcu]`.
- The **U1 toolhead** is connected to the Pi over **USB** and acts as the
  printer's **extruder** (its own MCU, `[mcu e0]`).

The toolhead board is a full standalone MCU. It drives its own extruder motor,
heater cartridge, thermistor, nozzle fan, part-cooling fan, filament sensor, an
accelerometer, and a **bed-detection coil** used as a Z-probe.

> **Accelerometer availability varies by toolhead.** Only the **left toolhead**
> (tool 0, the "special" one) has the LIS2DW accelerometer components populated
> on its board. The other toolheads (1–3) do **not** — their LIS2DW reads back
> `0xff` (WHO_AM_I fail) and `ACCELEROMETER_*` commands error with
> `Invalid lis2dw id`. The **replacement toolhead boards Snapmaker sells are all
> fully populated** and do have the accelerometer.

---

## 2. Hardware

### 2.1 U1 toolhead

- Main chip: **AT32F415RC** (Artery). This is marketed as an STM32F105 drop-in
  and runs Klipper firmware compiled for the **`stm32f105xc`** target. It is the
  reason the board self-identifies to USB as `stm32f105xc`.
- MCU clock: **144 MHz** from an **8 MHz** crystal.
- USB: **USB-OTG on PA11/PA12**, enumerated as a Klipper CDC-ACM device
  (VID/PID `1d50:614e`).

### 2.2 The USB-C toolhead cable

The USB-C connection is real differential USB. The pin
positions on the USB-C connector follow the standard USB-C pinout — only the
*function* of some pins differs from a data-only cable (VCC carries +24 V, and
GND also carries the 24 V return):

| Pin   | Wire color (if you cut the cable open) | Function                            |
|-------|----------------------------------------|-------------------------------------|
| GND   | Black                                  | Ground (also carries the 24 V return)|
| VCC   | Red                                    | **+24 V** power to the toolhead     |
| D-    | White                                  | Differential USB data (to PA11/PA12)|
| D+    | Green                                  | Differential USB data (to PA11/PA12)|
| CC    | Yellow                                 | 8 kΩ to GND on the hub side (USB-C presence detect?) |

Note: the **CC pin does not appear to be needed for the toolhead to function** —
it looks like a USB-C presence/connection detect that is not actually used here.

> **Power warning:** whatever USB-C cable and added cabling you use, the toolhead
> can draw around **80 W** of power — make sure the cable, connectors, and +24 V
> supply are all rated for it.

So to connect a toolhead to a Pi you need:

1. A **USB-C** connection carrying **D+/D-** into a USB-A/C port on the Pi
   (either directly or through a hub).
2. **+24 V power delivered through the same USB-C cable**. The cable is modified
   to feed **+24 V and GND** to the toolhead side (the toolhead does not draw
   power from the USB data lines); the Pi side of the power wires is **not
   connected** — the Pi only sees the D+/D- data pair.

### 2.3 Raspberry Pi

- Any Raspberry Pi with a USB port.
- The toolhead can plug **directly into a Pi USB port** or into a USB hub.
- The Pi does **not** need any special hardware to talk to the toolhead — it's
  just a USB serial device.

### 2.4 Main motion board

Any board standard klipper supports.

---

## 3. Software

### 3.1 The `mainline-port` Klipper tree (required)

The port adds native support by porting the AT32
firmware sources, the inductance-coil modules, and the LIS2DW module into a
**mainline Klipper tree** (the `mainline-port` workspace). Klippy and the
toolhead firmware are **protocol-compatible without matching version strings**
(message/command CRC compatibility).

- Get the port source and place it on the Pi, e.g. `/home/<user>/mainline-port`.
- Point klippy at the port. In the systemd `klipper.service`/`klipper.env`, set
  `KLIPPER_ARGS` to `<port>/klippy/klippy.py`.
- The work documented in this guide was done from **Windows** (editing and
  deploying files via SSH); the Pi itself is Linux. Nothing in the procedure is
  Windows-specific.

### 3.2 Build the C helper

The port's `chelper/c_helper.so` is built for the port tree. `klippy/chelper`
auto-rebuilds it on startup when the C sources change (via
`chelper/__init__.py` → `check_build_c_library()`); a missing or stale `.so`
causes an `OSError` at `reactor.Reactor()` init (the main thread hangs at
`_shutdown`).


### 3.3 Firmware on the toolhead

**You do not need to reflash the toolhead** — the stock Snapmaker firmware on it
is protocol-compatible with the ported klippy and works as-is.

- The toolhead runs stock fork firmware (e.g. `lava_1.4.0_20260522-4-g101c7dfd`,
  138 MCU commands).
- Klippy does **not** require version strings to match.
- The host may warn `MCU 'e0' has deprecated code (missing feature
  'STEPPER_STEP_BOTH_EDGE')` — harmless. The stock toolhead firmware declares
  the older `STEPPER_BOTH_EDGE` feature name; the ported host looks for the newer
  `STEPPER_STEP_BOTH_EDGE` name, so single-edge stepping is used. No functional
  loss.

If you must build toolhead firmware, the build config is: MCU `stm32f105xc`,
`CONFIG_MACH_AT32F415=y`, 8 MHz xtal → 144 MHz, `CONFIG_USBSERIAL=y`,
`CONFIG_STM32_USB_PA11_PA12=y`, VID/PID `1d50:614e`. Flashing requires a
power-cycle + BOOT0 procedure (the toolhead has no software DFU trigger) — a
separate exercise, not needed to make it work with the Pi. 
**Note: reflashing a toolhead has not been tested yet.**

---

## 4. Toolhead pinout

From the U1 printer config (`[extruder]`/`e0` section):

| Function          | Pin      | Notes                                     |
|-------------------|----------|-------------------------------------------|
| Extruder step     | `e0:PA8` | 1/16 microsteps, rotation_distance 4.95   |
| Extruder dir      | `e0:PA9` |                                           |
| Extruder enable   | `!e0:PB2`| Active low                                 |
| Heater cartridge  | `e0:PB5` |                                           |
| Thermistor        | `e0:PA2` | `NTC_100K_3950_PRECISE`                    |
| Nozzle/hotend fan | `!e0:PB0`| **Active low**, tach on `e0:PA10`          |
| Part-cooling fan  | `e0:PB3` | Tach on `e0:PB4`                           |
| Bed coil (probe)  | `e0:PA0` | Eddy-current/inductance coil (see §6)      |
| Filament sensor   | `e0:PA1` |                                           |
| Accelerometer     | `e0:PA4` | LIS2DW, software SPI PA5/6/7               |
| Power-loss detect | `e0:PB8` |                                           |

All 4 toolheads share this pinout (PA8/PA9/PB2/PB5/PA2/PA3/PB3).

> **Accelerometer note:** the LIS2DW on `e0:PA4` (software SPI on PA5/6/7) is
> **only populated on the left toolhead** (tool 0). Toolheads 1–3 have the
> footprint but no chip — reading their WHO_AM_I returns `0xff`. Replacement
> boards from Snapmaker are fully populated and include the accelerometer.

---

## 5. Config files

The port ships with an universal config for a single toolhead,
`config/sample-snapmaker-u1-toolhead.cfg` (the full U1 toolhead section, see
§5.2).

### 5.1 Main config (`printer.cfg`)

Only the parts that matter for the toolhead / ported features are shown. All
mainboard-specific sections (steppers, heaters, fans, `[mcu]` serial, bed-mesh
dimensions, z-tilt positions, kinematics limits) are **yours to fill in** —
use your own board's pinout and values.

```ini
# Your own machine config goes here (steppers, [mcu], heaters, fans, etc.)
#   -> use your own board's pinout; none of that is toolhead-specific.
[include toolhead.cfg]   # the U1 toolhead (see §5.2)

[mcu]
serial: /dev/serial/by-id/<your-main-board-usb-serial>
restart_method: command

# Your [printer] section: kinematics, max_velocity, max_accel, etc.

# Ported/toolhead-relevant pieces only:
[force_move]
enable_force_move: True   # optional; useful when testing

[resonance_tester]
accel_chip: lis2dw e0_lis2dw
accel_per_hz: 70
probe_points:
    147,154,20 # an example - use your own bed point
```

Notes:

- `min_temp: -100` on the toolhead extruder is a convenience when testing
  without a thermistor (floating ADC reads roughly -50..-90 °C). Raise it when
  real sensors are present.

### 5.2 Toolhead config (`toolhead.cfg`; ships as `config/sample-snapmaker-u1-toolhead.cfg`)

The toolhead **must** be `[extruder]` (index 0). The host iterates `extruder`,
`extruder1`, ... in `kinematics/extruder.py:add_printer_objects` and **stops at
the first missing section** — so if there is no `[extruder]`, no extruders load.

```ini
# U1 toolhead (AT32F415RC) as the single extruder

[mcu e0]
serial: /dev/serial/by-id/usb-Klipper_stm32f105xc-<version>-<unique-serial>-if00

[extruder]
step_pin: e0:PA8
dir_pin: e0:PA9
enable_pin: !e0:PB2
microsteps: 16
rotation_distance: 4.95
nozzle_diameter: 0.400
filament_diameter: 1.750
max_extrude_only_accel: 5000
max_extrude_only_velocity: 100
max_extrude_cross_section: 500
heater_pin: e0:PB5
sensor_type: Generic 3950
sensor_pin: e0:PA2
control: pid
pid_Kp: 33.540
pid_Ki: 9.317
pid_Kd: 30.186
min_temp: -100
max_temp: 300
min_extrude_temp: 170
max_extrude_only_distance: 200
smooth_time: 2

[heater_fan e0_nozzle_fan]
pin: !e0:PB0
heater: extruder
heater_temp: 45
fan_speed: 1
shutdown_speed: 0

[inductance_coil extruder]
pin: e0:PA0
z_offset: -0.05
freq_cal_mode: 1
samples: 4
samples_tolerance: 0.020
samples_tolerance_retries: 10
capture_over_cnt: 200
cal_window_size: 5
cal_time_out: 2
speed: 2.0
sample_retract_dist: 1
relative_trigger_freq: 300
max_freq: 1650000
min_freq: 1200000
horizontal_move_z: 100

[lis2dw e0_lis2dw]
cs_pin: e0:PA4
spi_bus: spi1
axes_map: y,x,z
```

Notes:
- `sensor_type: Generic 3950` is used ; the U1 uses `NTC_100K_3950_PRECISE`.

### 5.3 Find the toolhead's serial

Each toolhead has a **unique** USB serial, so the by-id path always differs:

```
ls /dev/serial/by-id/
```

Look for `usb-Klipper_stm32f105xc-...-if00` and use that full path in
`[mcu e0]`. If the toolhead is unplugged, klippy fails with
`mcu 'e0': Unable to open serial port ... No such file or directory`.

#### By-id vs by-path (which serial to use)

Two options exist for the `[mcu e0]` `serial:` value:

- **`/dev/serial/by-id/...`** — follows the **device**, not the port. Each
  physical toolhead has a unique serial baked into its by-id name, so this is
  stable no matter which USB port the toolhead is plugged into. Downside: if
  you swap to a *different* toolhead, the by-id name changes and you must edit
  the config.
- **`/dev/serial/by-path/...`** — follows the **port**, not the device. Every
  toolhead plugged into the *same* USB port gets the same by-path name, so you
  can hot-swap toolheads without editing the config. Downside: if you move the
  toolhead to a different port (or the hub path changes), the by-path name
  changes and klippy can't find the MCU.

Find the port-based name with:

```
ls /dev/serial/by-path/   # match the platform-...-usb-...-if00 entry
```

On the **U1 itself**, each toolhead already has a fixed port on the internal
USB hub (by-path `usb-0:1.x`), so by-path would work there. On a **printer**,
by-path is fine **if you always plug the toolhead into the same Pi port/hub
port**. By-id is the more robust default (recommended): it is immune to port
changes, and you only edit it when you swap physical toolheads.

---

## 6. The bed-detection coil as a Z-probe

The U1's bed detection is an **eddy-current / inductance coil** on `e0:PA0`. It
is an LC oscillator whose resonant frequency (~1.0–2.0 MHz) changes with the
distance to a **conductive (metal)** surface. The toolhead MCU measures the
frequency and can fire a `pulse_endstop` when a threshold is crossed.

The coil is handled by two ported modules in the `mainline-port` tree:

- `klippy/extras/inductance_coil.py` — the frequency sensor + `pulse_endstop`.
- `klippy/extras/probe_inductance_coil.py` — probe session, calibration,
  z-offset, and the `PROBE`/`BED_MESH_CALIBRATE` command plumbing.

The **first** `[inductance_coil <name>]` section installs the printer-wide
`probe` object. Section name = the extruder section it belongs to (`extruder`,
`extruder1`, ...). Pin must match that extruder's `PA0`.

### 6.1 Config

Add to `toolhead.cfg` (mirrors the U1's working values):

```ini
[inductance_coil extruder]
pin: e0:PA0
z_offset: -0.05
freq_cal_mode: 1
samples: 4
samples_tolerance: 0.020
samples_tolerance_retries: 10
capture_over_cnt: 200
cal_window_size: 5
cal_time_out: 2
speed: 2.0
sample_retract_dist: 1
relative_trigger_freq: 300
max_freq: 1650000
min_freq: 1200000
horizontal_move_z: 100
```


### 6.2 Firmware requirements

All in `src/stm32/inductance_coil.c`, compiled **only** when the firmware is
built with `CONFIG_MACH_AT32F4x=y` (the toolhead config sets this):

- `inductance_coil_config`
- `query_inductance_coil`
- `query_inductance_coil_status`
- `query_inductance_coil_config_info`
- `virtual_gpio_trigger`
- `virtual_gpio_trigger_with_timer`
- the `pulse_endstop` pin type (`config_endstop ... is_pulse_gpio`)

### 6.3 Known firmware-version bug (and the fix)

The toolhead firmware version matters here:

- Newer stock toolhead firmware (e.g. `lava_1.4.0`, 138 MCU commands) has
  `virtual_gpio_trigger_with_timer`.
- Older stock firmware (e.g. `lava_0.7.2`, 137 commands) does **not**.

`inductance_coil.py` originally called
`lookup_command("virtual_gpio_trigger_with_timer ...")` unconditionally in
`_build_config`. On old firmware this aborts the connection:

```
mcu 'e0': Unknown command: virtual_gpio_trigger_with_timer
MCU Protocol error
```

**Fix (no reflash needed):** patch `klippy/extras/inductance_coil.py` to use
`try_lookup_command()` and fall back to the non-timer command when the
with-timer variant is absent:

```python
self.set_trig_freq_with_timer_cmd = self._mcu.try_lookup_command(
    "virtual_gpio_trigger_with_timer oid=%c absolute_mode=%u trigger_mode=%u trigger_invert=%u trg_freq_ht=%u trg_freq_lt=%u force_update=%u clock=%u")
```

```python
def _cmd_set_trig_freq_with_timer(self, freq_ht, freq_lt, clock, absolute_mode=True, trigger_mode=0, trigger_invert=False, force_update=True):
    if self.set_trig_freq_with_timer_cmd is None:
        # Older firmware only supports the non-timer variant
        self._cmd_set_trig_freq(freq_ht, freq_lt, absolute_mode, trigger_mode, trigger_invert, force_update)
        return
    self.set_trig_freq_with_timer_cmd.send([self._oid, absolute_mode, trigger_mode, trigger_invert, freq_ht, freq_lt, force_update, clock])
```

This is safe because the default probe path already uses the non-timer variant
(`sample_wait_before_setup=0.1` / `sample_wait_after_setup=0.05`); the
with-timer variant is only used when both waits are 0.

### 6.4 Verify the eddy coil works

Query the current frequency (response text goes to the gcode terminal and
klippy.log; the API only returns `{"result":"ok"}`):

```
curl -X POST http://localhost:7125/printer/gcode/script \
  -d 'script=INDUCTANCE_COIL_QUERY PROBE=extruder'
```

Expected klippy.log output with no metal nearby (air baseline) roughly:

```
capture_freq: 1487127 virtual_gpio: TRIGGERED
frequency data: 307.8334s: 1487127Hz
```

- **No metal nearby:** ~1.49 MHz (air baseline).
- **Metal brought near / nozzle pressed against a metal bed:** the frequency
  changes (it was observed to **rise** with pressure on the nozzle).
- Out-of-range (< ~1.0 MHz or > ~2.0 MHz) is treated as an error sample.


### 6.6 EMI and the trigger frequency

The coil's frequency is disturbed by motor current/vibration while the printer
moves. The probe's trigger band is `current_freq ± relative_trigger_freq`; if a
motor swings the coil frequency past the band during a probe descent, you get a
**false trigger** (`No trigger on probe after full movement` on the next sample).

For me, the worst offenders were the **X/Y gantry steppers**; Z motion was
essentially silent to the coil (near the noise floor), so Z probing itself was
EMI-clean. On my printer, the gantry swung the coil frequency up to **1728 Hz**,
way past the stock **300 Hz** band, so I raised `relative_trigger_freq` to
**1900 Hz**.

**How to test for EMI on your machine:**

1. Start a bulk recording of the coil frequency (toggle — run the same command
   again to stop):
   ```
   FREQUENCY_MEASURE PROBE=extruder
   ```
2. While it records, move the axes (e.g. `G28`, or isolated `G1` moves with
   `G4` dwells between them inside one script).
3. Stop it by running `FREQUENCY_MEASURE PROBE=extruder` again.
4. Check the CSV under `~/printer_data/gcodes/frequency_data/` for how far the
   frequency deviates during motion vs idle. If it exceeds your
   `relative_trigger_freq`, raise that value until it no longer false-triggers.

---

## 7. Bring-up sequence (recommended order)

1. **Build the port** — compile the firmware from
   `mainline-port` and flash it to the main board (SD card: copy
   `out/klipper.bin` to `firmware.bin` on a FAT32 card, power cycle).
2. **Start with the main board only** — confirm `Printer is ready` with the
   main `[mcu]`.
3. **Plug in the toolhead**, find its serial (`ls /dev/serial/by-id/`), add the
   `[mcu e0]` + `[extruder]` sections, restart klippy.
4. Verify the toolhead:
   - Thermistor reads ambient temperature.
   - Nozzle fan turns on above `heater_temp: 45` and off below.
5. **Add the `[inductance_coil extruder]` section**, restart, confirm the
   `Unknown command` bug is gone (or apply the §6.3 patch).
6. Run `INDUCTANCE_COIL_QUERY PROBE=extruder` — expect ~1.49 MHz in air.
7. Verify the accelerometer: `ACCELEROMETER_QUERY CHIP=e0_lis2dw` — expect
   `lis2dw_dev_id: 44`. (Missing on toolheads 1–3.)

---

## 8. Gotchas checklist

| Symptom | Cause / fix |
|---|---|
| `OSError` at `reactor.Reactor()` init | `chelper/c_helper.so` missing or stale — rebuild for the port tree. |
| No extruders load at all | Toolhead must be `[extruder]` (index 0); the host stops at the first missing extruder section. |
| `mcu 'e0': Unknown command: virtual_gpio_trigger_with_timer` | Old toolhead firmware; apply §6.3 patch. |
| `Invalid lis2dw id (got ff vs 44)` / accel reads `0xff` | Toolhead 1–3 has no accelerometer chip populated. Only the **left toolhead** (tool 0) carries the LIS2DW; Snapmaker replacement boards all have it. |
| `Unable to open serial port ... No such file or directory` | Toolhead unplugged or wrong by-id path (also after swapping toolheads — see §5.3). |
| Floating temp readings ~-50..-90 °C | No thermistor attached; set `min_temp: -100` for testing use. |
| Probe false-triggers / `No trigger on probe after full movement` | Coil frequency is being swung past the trigger band by X/Y gantry EMI. Raise `relative_trigger_freq` (my printer needed 1900 Hz; see §6.6). |
| `Missing STEPPER_STEP_BOTH_EDGE` / `has deprecated code` | Expected with un-reflashed stock toolhead firmware; single-edge stepping, no functional loss. |
| MCUs vanish from USB after power loss | Re-enumerate, then `FIRMWARE_RESTART` (or `sudo systemctl restart klipper`). |
| `.py` host-module edits don't take effect | `RESTART`/`firmware_restart` do not reload Python modules (sys.modules cache). Use `sudo systemctl restart klipper`, then firmware_restart to reset MCUs. |

---

## 9. Reference facts

- Toolhead MCU self-identifies as `stm32f105xc` (AT32F415RC running the
  STM32F105 klipper target).
- Toolhead firmware versions seen: `lava_1.4.0_20260522-4-g101c7dfd`
  (138 commands, has `virtual_gpio_trigger_with_timer`) and
  `lava_0.7.2_20250906-14-g7d8b3657` (137 commands, does not).
- The ported host requires the toolhead's **extended stepper protocol**
  (`config_stepper ... type=%u index=%u`, `queue_step ... line=%u`) — ported
  from the fork's power-loss wire format so the un-reflashed toolhead works.
- The main board runs **mainline-port firmware**, protocol-compatible with the ported
  host; it does **not** need to match the toolhead's version.

---
