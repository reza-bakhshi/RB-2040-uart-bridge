RB-2040 UART Bridge
====================

A USB-to-UART bridge firmware for the Raspberry Pi Pico (RP2040) and Pico 2
(RP2350). It exposes both hardware UARTs as two independent USB CDC-ACM serial
ports, so a single board behaves like two USB-to-UART adapters at once.

Based on [pico-uart-bridge](https://github.com/Noltari/pico-uart-bridge) by
Álvaro Fernández Rojas.

Suggested Hardware
------------------

This firmware is designed and tested for the
[**RB-2040**](https://github.com/reza-bakhshi/RB-2040), a compact USB-dongle-style
RP2040 development board.

| Spec      | Detail                                                    |
|:---------:|:----------------------------------------------------------|
| MCU       | Raspberry Pi RP2040 (dual Arm Cortex-M0+, up to 133 MHz)  |
| Flash     | 4 MiB W25Q32JV QSPI                                       |
| USB       | USB-C, native USB 1.1 device, ESD-protected data lines    |
| Power     | Dedicated 3.3 V LDO, power indicator LED                  |
| Buttons   | BOOTSEL and RESET                                         |
| Debug     | SWDIO, SWCLK and GND test pads                            |
| Expansion | 2×6 header: GP8–GP15, GP26, 3V3, VBUS and GND             |
| PCB       | 2-layer, 1.6 mm, designed with KiCad                      |

The firmware also runs on a standard Raspberry Pi Pico or Pico 2.

Features
--------

- Two independent USB CDC serial ports, one per hardware UART.
- Line settings (baud rate, data bits, parity, stop bits) configurable by the
  host; defaults to 115200 8N1.
- 2560-byte RX and TX buffers per port.
- USB serial number derived from the flash unique ID, so each board gets a
  stable and unique identity.
- Non-blocking UART RX interrupt handler: the bridge always drains the RX FIFO
  and never blocks on a mutex, so it cannot hang when nothing is reading the
  port. If the host cannot keep up, excess bytes are dropped.
- Ignores host DTR: data flows as soon as a port is opened, even with tools that
  never assert DTR.
- Builds for both Pico 1 (RP2040) and Pico 2 (RP2350).

UART Pinout
-----------

Two firmware variants are built. They differ only in the UART0 pin pair; UART1
is GP8/GP9 in both.

| Function | GP12-13 image | GP16-17 image | USB port | RB-2040 J2 |
|:--------:|:-------------:|:-------------:|:--------:|:----------:|
| UART0 TX | GP12          | GP16          | CDC 0    | pin 12     |
| UART0 RX | GP13          | GP17          | CDC 0    | pin 10     |
| UART1 TX | GP8           | GP8           | CDC 1    | pin 11     |
| UART1 RX | GP9           | GP9           | CDC 1    | pin 9      |

The `RB-2040 J2` column applies to the GP12-13 image; GP16/GP17 are not broken
out on the RB-2040 header.

Firmware images:

- `RB-2040-uart-bridge-GP12-13.uf2` — UART0 on GP12/GP13. Use this for the
  RB-2040; both ports are then available on the J2 header.
- `RB-2040-uart-bridge-GP16-17.uf2` — UART0 on GP16/GP17. Use this for the older
  module (or a standard Raspberry Pi Pico, where GP16/GP17 are on pins 21/22).

The TX and RX pins are configured with internal pull-ups to keep floating inputs
from generating noise that looks like received data.

USB Identity
------------

| Field     | Value                                        |
|:---------:|:--------------------------------------------|
| VID       | `0x2E8A` (Raspberry Pi)                      |
| PID       | `0x000A` (Raspberry Pi Pico SDK CDC)         |
| Product   | `RB-2040-uart-bridge`                        |
| Serial    | Derived from the flash unique ID             |

Hardware flow control (RTS/CTS) is not implemented. DTR and RTS are not used to
gate data: the bridge starts passing bytes as soon as the port is open, even if
the host never asserts DTR.

Requirements
------------

- CMake 3.13 or newer
- ARM GCC toolchain (`arm-none-eabi-gcc`)
- Python 3
- Git

Building
--------

1. Fetch the submodules (only needed once):

   ```sh
   git submodule update --init --recursive
   ```

2. Build for the Pico 1 (default):

   ```sh
   ./build.sh
   ```

   Both images are written to `build/pico/`:

   - `build/pico/RB-2040-uart-bridge-GP12-13.uf2`
   - `build/pico/RB-2040-uart-bridge-GP16-17.uf2`

3. Build for the Pico 2 instead:

   ```sh
   PICO_BOARD=pico2 ./build.sh
   ```

   Both images are written to `build/pico2/`.

To configure and build manually:

```sh
cmake -B build/pico -DPICO_BOARD=pico
make -C build/pico
```

Flashing
--------

1. Hold the **BOOTSEL** button while plugging the board into USB. It appears as
   a mass-storage device named `RPI-RP2`.
2. Copy the image that matches your board (for the RB-2040, use
   `RB-2040-uart-bridge-GP12-13.uf2`) onto that drive.
3. The board reboots automatically and two serial ports appear.

Alternatively, with `picotool`:

```sh
picotool load -f build/pico/RB-2040-uart-bridge-GP12-13.uf2
```

Usage
-----

- **Linux / macOS:** `ttyACM0` and `ttyACM1` (the order may vary; use the USB
  serial number or `/dev/serial/by-id/` to tell them apart).
- **Windows:** two COM ports.

Open either port with a terminal and set the desired baud rate there, for
example:

```sh
screen /dev/ttyACM0 115200
```

The line coding selected by the host is applied to the corresponding UART.

Troubleshooting
---------------

- **Port appears but no data flows in either direction.** Older firmware only
  serviced an interface after the host asserted DTR, so tools that never raise
  DTR saw both ports dead. This version ignores DTR and services every port once
  USB is configured.
- **No data in one or both directions.** Check that the target's TX goes to the
  Pico RX pin for that port and that both devices share a common ground. Also
  make sure you flashed the image that matches your board (GP12-13 or GP16-17).
- **The bridge stops responding while the target keeps logging.** Older firmware
  could hang here: the RX interrupt handler blocked on a mutex or returned with
  data still in the FIFO. This version uses `mutex_try_enter()` and always
  drains the FIFO, dropping bytes when the host is not reading, so it recovers
  on its own instead of needing a replug.
- **Dropped characters at high baud rates.** Like most CDC bridges, data is lost
  if no program is reading the port or the host cannot keep up. Keep the
  terminal open and avoid heavy USB contention.

Architecture
------------

- **Core 0:** applies host line coding to the UART and moves data from the USB
  buffer out to the UART.
- **Core 1:** runs the TinyUSB device task and moves data from the UART buffer
  out to USB.
- **UART RX interrupt:** drains the RX FIFO into the per-port buffer without
  ever blocking.

Credits and License
-------------------

Originally created by Álvaro Fernández Rojas and released under the MIT License.
This fork is maintained by Reza Bakhshi. It bundles TinyUSB and the Raspberry Pi
Pico SDK. See [LICENSE.md](LICENSE.md) for details.

This software is provided without warranty, according to the MIT License, and
should therefore not be used where it may endanger life, financial stakes, or
cause discomfort and inconvenience to others.
