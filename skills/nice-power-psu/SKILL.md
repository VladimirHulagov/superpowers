---
name: nice-power-psu
description: Use when controlling Nice Power or Kuaiqu programmable DC power supplies over serial USB, setting voltage or current, reading actual measurements, toggling output on/off, or writing test scripts that involve power supply automation
---

# Nice Power / Kuaiqu PSU Control

## Overview

Python interface for Nice-Power / Kuaiqu programmable DC power supplies (tested with R-SPS3060-232) via serial USB. Uses a simple `<CMD><VALUE><PAD>` protocol over 9600 baud RS-232.

## When to Use

- Controlling a Nice-Power / Kuaiqu PSU from Python
- Setting voltage or current programmatically
- Reading actual (measured) voltage/current
- Toggling DC output on/off
- Automated test sequences involving power cycling

## Protocol

**Serial config:** `/dev/ttyUSB0`, 9600 baud, 8N1, timeout 1s

**Command format:** `<CCvvvvvvPPP>`
- `CC` — 2-char command code
- `vvvvvv` — 6-digit value (3 integer + 3 decimal, no dot)
- `PPP` — 3-char padding (`000`)

**Response:** ends with `>`. Value responses: byte 1 = mode (`1`=CV, `C`=CC), bytes 3-8 = 6-digit value (multiply by 1e-3 for real units). OK responses contain `OK`.

## Quick Reference

| Function | Command String | Returns |
|---|---|---|
| Read actual voltage | `<02000000000>` | float (V) |
| Read actual current | `<04000000000>` | float (A) |
| Set voltage | `<01vvvvvv000>` | OK |
| Set current | `<03vvvvvv000>` | OK |
| Output ON | `<07000000000>` | OK |
| Output OFF | `<08000000000>` | OK |
| Remote mode | `<09100000000>` then `<01004580000>` then `<03006920000>` | OK |
| Local mode | `<09200000000>` | OK |

**Value encoding example:** 24.000V → `"024000"`, 1.500A → `"001500"` (format: `{:07.3f}` then remove dot)

## Implementation

Full Python interface: see `nice_power_kuaiqu_interface.py` in this skill directory.

### Minimal Usage

```python
import serial, time

ser = serial.Serial(port="/dev/ttyUSB0", baudrate=9600, timeout=1)
ser.flush()

def psu_write(cmd):
    ser.write(cmd.encode())

def psu_read_decode():
    data = ser.read_until(b">")
    val = float(data[3:9].decode()) * 1e-3
    return val

def psu_read_ok():
    data = ser.read_until(b">").decode()
    return 1 if 'OK' in data else 0

def set_voltage(val):
    v = "{:07.3f}".format(val).replace(".", "")
    psu_write("<01" + v + "000>")
    return psu_read_ok()

def set_current(val):
    v = "{:07.3f}".format(val).replace(".", "")
    psu_write("<03" + v + "000>")
    return psu_read_ok()

# Sequence: remote → set voltage → set current → output on
psu_write("<09100000000>"); psu_read_ok()
psu_write("<01004580000>"); psu_read_ok()
psu_write("<03006920000>"); psu_read_ok()
set_voltage(12.0)
set_current(0.5)
psu_write("<07000000000>"); psu_read_ok()  # output ON
```

## Common Mistakes

- **Forgot remote mode** — `set_psu_remote()` must be called before any set commands
- **Wrong device path** — check `ls /dev/ttyUSB*` or `dmesg | grep ttyUSB`
- **Permissions** — user must be in `dialout` group: `sudo usermod -aG dialout $USER`
- **PSU off reads as CC mode** — when output is off, mode byte is `C` (constant current), not `1`
- **Value overflow** — max voltage/current depends on model (R-SPS3060: 30V / 60A)

## Hardware Notes

- Tested on R-SPS3060-232, should work on other Nice-Power/Kuaiqu models
- Requires `pyserial` package (`pip3 install pyserial`)
- Uses RS-232 serial over USB (USB-to-serial adapter or built-in port)
- Protocol reference: https://www.eevblog.com/forum/testgear/spps3010d-power-supply-softwareprotocol/
