---
title: Building for Seeed XIAO nRF52840 (BLE / Sense)
page_id: xiao-ble
---

This fork adds support for using a **Seeed XIAO nRF52840** (or **XIAO nRF52840 Sense**) board as a Crazyradio 2.0 replacement. The board enumerates as `1915:7777 Bitcraze Crazyradio 2.0` over USB and can be used with `cfclient` / `cflib` to communicate with a Crazyflie.

## Why a XIAO?

The XIAO uses the same nRF52840 SoC as the production Crazyradio 2.0, ships with an Adafruit-compatible UF2 bootloader, and is cheap and easy to obtain. No SWD probe is required — flashing is drag-and-drop.

## Prerequisites

* Seeed XIAO nRF52840 or XIAO nRF52840 Sense
* USB-C cable
* Linux or macOS host with `uv` and `git` installed
* The factory-shipped Adafruit nRF52 UF2 bootloader (already present on new boards)

## Step 1 — Get the source

```bash
git clone https://github.com/YOUR-USER/crazyradio2-firmware.git
cd crazyradio2-firmware
```

## Step 2 — Set up the build environment

```bash
uv venv
source ./.venv/bin/activate
uv pip install .
just fetch-zephyr-sdk
just fetch-zephyr
```

This installs the Zephyr SDK to `~/.local/` and fetches Zephyr + its Python dependencies.

## Step 3 — Build for `xiao_ble`

```bash
west build -b xiao_ble --pristine
```

The `--pristine` flag is only needed the first time, or if you previously built for a different board. Subsequent builds:

```bash
west build
```

Successful output ends with:

```
Converted to uf2, output size: 117248, start address: 0x27000
Wrote 117248 bytes to crazyradio2.uf2
```

The `start address: 0x27000` is critical — it matches what the Adafruit bootloader expects (after the SoftDevice S140 v7 region).

## Step 4 — Flash via UF2

1. Plug the XIAO into USB.
2. Double-tap the reset button **quickly**. A USB drive named **`XIAO-SENSE`** (or similar) will appear.
3. Copy the UF2 to that drive:

```bash
cp build/zephyr/crazyradio2.uf2 /media/$USER/XIAO-SENSE/
```

The board reboots automatically and disappears from the file manager. Verify enumeration:

```bash
lsusb | grep 1915
# Bus 001 Device XXX: ID 1915:7777 Nordic Semiconductor ASA Bitcraze Crazyradio 2.0
```

## Step 5 — Use with cfclient

1. Install the Bitcraze udev rules (one-time, host system):

```bash
sudo tee /etc/udev/rules.d/99-crazyflie.rules <<'EOF'
SUBSYSTEM=="usb", ATTRS{idVendor}=="1915", ATTRS{idProduct}=="7777", MODE="0664", GROUP="plugdev"
EOF
sudo udevadm control --reload-rules && sudo udevadm trigger
sudo usermod -aG plugdev $USER
# log out / back in after this
```

2. Launch `cfclient`, click **Scan**. Found drones appear in the "Select an interface" dropdown as `radio://0/<channel>/<rate>`.

## Notes on the XIAO build

The XIAO support adds three files:

| File | Purpose |
|---|---|
| `boards/xiao_ble.conf` | Disables UF2 partition override (which forces `FLASH_LOAD_OFFSET=0x26000` for nRF52840), disables CDC ACM (conflicts with vendor USB class), keeps `BUILD_OUTPUT_UF2=y`. |
| `boards/xiao_ble.overlay` | Routes Zephyr console to RTT and removes the CDC ACM UART node that the upstream XIAO DTS adds. |
| `src/main.c` (modified) | Waits for the HFCLK crystal (HFXO) to be stable before initializing the radio, so the RF frequency is accurate from the first packet. |

The flash layout is dictated by the Adafruit nRF52 bootloader:

```
0x00000000  SoftDevice s140 v7  (156 kB, kept untouched by UF2)
0x00027000  Application         (788 kB)   <-- crazyradio2 app
0x000ec000  Storage             (32 kB)
0x000f4000  UF2 bootloader      (48 kB)
```

## Troubleshooting

* **Bootloader drive doesn't appear** → double-tap the reset button more quickly; on some boards the window is short (~300ms between presses).
* **`lsusb` shows the dongle but `cfclient` finds no Crazyflie** → make sure the Crazyflie is powered via its **own battery** with the **power button pressed** (LEDs fully booted). The drone's nRF51822 keeps the radio off when powered only via USB, since it doubles as the power-management chip.
* **You want to go back to the original XIAO firmware** → grab Adafruit's `xiao_nrf52840_*.uf2` from their releases and drop it on the bootloader drive.
