# Installation

Install the firmware on the controller board. Install the daemon and Klipper
plugin on the host, usually the Raspberry Pi running Klipper. The RIP and
encoder can run on that host or another computer.

## 1. Firmware (controller board)

Build with the Raspberry Pi Pico SDK:

```sh
cd paintress-firmware
mkdir build && cd build
cmake .. && make -j
```

Flash `paintress_firmware.uf2` through BOOTSEL mass storage or picotool. Run
the following command from the `paintress-firmware` directory:

```sh
picotool load -v -x build/paintress_firmware.uf2
```

The board appears as a USB CDC serial device. It sends protocol frames over
USB, including `LOG` frames that the daemon displays as text.

## 2. Daemon (host)

The daemon requires Python 3.9 or newer and `pyserial`:

```sh
cd paintress-daemon
pip install pyserial
python -m paintress_daemon --head c6n90
```

Set `--head` to the fitted printhead. The board cannot identify it, so the
daemon uses this setting to reject jobs made for another head.

It listens on TCP port 9000 and opens the firmware serial port when asked.
To run it as a service:

```ini
# /etc/systemd/system/paintressd.service
[Unit]
Description=Paintress daemon
After=network.target

[Service]
ExecStart=/usr/bin/python3 -m paintress_daemon --head c6n90
WorkingDirectory=/home/pi/paintress-daemon
Restart=on-failure
User=pi

[Install]
WantedBy=multi-user.target
```

```sh
sudo systemctl enable --now paintressd
```

## 3. Klipper plugin (host)

Copy all three plugin files into Klipper:

```sh
cp paintress-klipper-extras/extras/paintress.py \
   paintress-klipper-extras/extras/paintress_head.py \
   paintress-klipper-extras/extras/paintressd_client.py \
   ~/klipper/klippy/extras/
```

`paintress_head.py` registers the `[paintress_head <name>]` config sections.
Install all three files, even if you are not using a head section yet. A
missing module causes Klipper to reject those sections.

Add a `[paintress]` section and the trigger `[output_pin]` to `printer.cfg`,
covered in [Configuration](configuration.md), then restart Klipper.

## 4. RIP and encoder (anywhere)

```sh
cd paintress-rip-encoder
pip install -r requirements.txt   # optional: numba (faster), scipy (blue noise)
```

The RIP and encoder run offline. Copy the resulting job files into the
plugin's `base_path`, which defaults to `/home/pi/printer_data/paintress/`.

Next: [Configuration](configuration.md), then [Printing](printing.md).
