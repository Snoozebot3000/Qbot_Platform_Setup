# Qbot Platform Setup — Jetson Orin Nano + AVermedia D131L + OV9281

Getting the Quanser QBot Platform with a Jetson Orin Nano and AVermedia D131L carrier board running the latest JetPack has been a journey. This guide documents the full process — from setting up flashing tools to getting the OV9281 downward-facing MIPI camera working — so you don't have to figure it out yourself.

---

## Table of Contents

1. [Hardware Overview](#hardware-overview)
2. [Prerequisites](#prerequisites)
3. [Understanding the Software Stack](#understanding-the-software-stack)
4. [Step 1 — Prepare the Host PC](#step-1--prepare-the-host-pc)
5. [Step 2 — Download L4T and the AVermedia D131L BSP](#step-2--download-l4t-and-the-avermedia-d131l-bsp)
6. [Step 3 — Apply the BSP to L4T](#step-3--apply-the-bsp-to-l4t)
7. [Step 4 — Configure the OV9281 MIPI Camera](#step-4--configure-the-ov9281-mipi-camera)
8. [Step 5 — Put the Jetson Orin Nano into Force Recovery Mode](#step-5--put-the-jetson-orin-nano-into-force-recovery-mode)
9. [Step 6 — Flash the Board](#step-6--flash-the-board)
10. [Step 7 — First Boot and SDK Component Installation](#step-7--first-boot-and-sdk-component-installation)
11. [Step 8 — Verify the Camera](#step-8--verify-the-camera)
12. [Troubleshooting](#troubleshooting)
13. [References](#references)

---

## Hardware Overview

| Component | Details |
|---|---|
| **SOM (System-on-Module)** | NVIDIA Jetson Orin Nano (8 GB or 4 GB) |
| **Carrier Board** | AVermedia D131L |
| **Downward-Facing Camera** | OV9281 (1 MP global shutter, MIPI CSI-2) |
| **Host PC OS** | Ubuntu 20.04 LTS or Ubuntu 22.04 LTS (required for SDK Manager) |
| **Target JetPack Version** | JetPack 6.x (L4T 36.x) — latest stable release |

> **Note:** NVIDIA SDK Manager only runs on Ubuntu. If your development machine runs Windows or macOS, use a native Ubuntu installation or a virtual machine (VMware/VirtualBox) with USB 3.0 passthrough enabled.

---

## Prerequisites

### Hardware

- Jetson Orin Nano SOM seated in the AVermedia D131L carrier board
- USB-C cable (data-capable) or Micro-USB cable for the recovery/flashing connection
- USB-A to USB-C adapter if your host PC lacks a USB-C port
- 12 V DC power supply for the D131L (check the D131L datasheet for exact specs)
- OV9281 MIPI CSI-2 camera module connected to the CSI connector on the D131L
- microSD card (optional, for rootfs on microSD configurations) or NVMe SSD for rootfs

### Host PC Software

- Ubuntu 20.04 LTS or Ubuntu 22.04 LTS (x86_64)
- At least 40 GB of free disk space for L4T, BSP, and SDK Manager downloads
- Internet access
- `sudo` privileges

---

## Understanding the Software Stack

Before flashing, it helps to understand what each piece does:

```
┌────────────────────────────────────────────────────┐
│                  JetPack 6.x                       │
│  (CUDA, cuDNN, TensorRT, VPI, Multimedia API, …)   │
├────────────────────────────────────────────────────┤
│          L4T 36.x  (Linux for Tegra)               │
│  ┌─────────────────────────────────────────────┐   │
│  │  AVermedia D131L BSP (board-specific files) │   │
│  │  - Device tree  (.dtb / .dtbo)              │   │
│  │  - Kernel modules                           │   │
│  │  - U-Boot configuration                     │   │
│  └─────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────┐               │
│  │  OV9281 Camera Driver / Overlay  │               │
│  └──────────────────────────────────┘               │
├────────────────────────────────────────────────────┤
│     Jetson Orin Nano SOM  +  AVermedia D131L        │
└────────────────────────────────────────────────────┘
```

- **L4T (Linux for Tegra):** NVIDIA's downstream Linux kernel and root filesystem for Jetson. Version 36.x corresponds to JetPack 6.
- **BSP (Board Support Package):** Carrier-board-specific files from AVermedia that patch L4T to support the D131L's peripherals (power rails, USB, PCIe, camera connectors, etc.).
- **JetPack SDK:** The CUDA/AI libraries installed on top of L4T, either during flashing via SDK Manager or afterwards via `apt`.

---

## Step 1 — Prepare the Host PC

### 1.1 Install SDK Manager

NVIDIA SDK Manager is the recommended tool for downloading L4T components and flashing Jetson boards.

1. Go to the [NVIDIA SDK Manager download page](https://developer.nvidia.com/sdk-manager) (NVIDIA Developer account required — free to register).
2. Download the `.deb` package for your Ubuntu version.
3. Install it:

```bash
sudo apt update
sudo apt install ./sdkmanager_*_amd64.deb
```

4. Launch SDK Manager and sign in with your NVIDIA Developer account:

```bash
sdkmanager
```

> **Alternative — Manual (command-line) flashing:** SDK Manager wraps the same `flash.sh` script used in manual flashing. The sections below show the manual approach, which gives you more control and is easier to script. Both methods work.

### 1.2 Install Required Host Dependencies

> **Note:** Several packages below (`simg2img`, `abootimg`, `lbzip2`, `qemu-user-static`, etc.) live in the **universe** and **multiverse** repositories. Enable them before running `apt install`:
>
> ```bash
> sudo add-apt-repository universe
> sudo add-apt-repository multiverse
> sudo apt update
> ```

```bash
sudo apt update
sudo apt install -y \
    git \
    python3 \
    python3-pip \
    device-tree-compiler \
    u-boot-tools \
    qemu-user-static \
    binfmt-support \
    libxml2-utils \
    simg2img \
    lbzip2 \
    nfs-kernel-server \
    abootimg \
    whois
```

---

## Step 2 — Download L4T and the AVermedia D131L BSP

### 2.1 Download L4T Driver Package and Sample Root Filesystem

Go to the [NVIDIA L4T Archive](https://developer.nvidia.com/embedded/jetson-linux-archive) and download the files that match your JetPack 6 target (L4T 36.x):

- **Jetson Linux Driver Package (BSP)** — e.g., `Jetson_Linux_R36.x.x_aarch64.tbz2`
- **Sample Root Filesystem** — e.g., `Tegra_Linux_Sample-Root-Filesystem_R36.x.x_aarch64.tbz2`

Replace `36.x.x` with the latest released revision listed on the NVIDIA archive page (e.g., `36.3.0` or `36.4.0`).

```bash
# Create a working directory
mkdir -p ~/jetson_flash && cd ~/jetson_flash

# Extract L4T Driver Package
tar xjf Jetson_Linux_R36.x.x_aarch64.tbz2

# Extract Sample Root Filesystem into the correct subdirectory
cd Linux_for_Tegra/rootfs
sudo tar xjpf ../../Tegra_Linux_Sample-Root-Filesystem_R36.x.x_aarch64.tbz2
cd ../..
```

### 2.2 Download the AVermedia D131L BSP

AVermedia distributes their Jetson carrier board BSPs through their support portal and/or GitHub repository.

1. Visit the [AVermedia Embedded Computing resources page](https://www.avermedia.com/professional/product/d131l/overview) or contact AVermedia support to obtain the D131L BSP tarball for JetPack 6 / L4T 36.x.
   - The file may be named something like `d131l_bsp_r36.x.x.tbz2` or `avermedia_d131l_l4t36_bsp.tar.gz`. The commands in Step 3 use `avermedia_d131l_l4t36_bsp.tar.gz` as the example filename — **replace this with the actual filename you received**.
2. Download or copy the BSP tarball into `~/jetson_flash/`.

> **Important:** Confirm the BSP revision matches the L4T revision you downloaded. Mismatched versions are the most common cause of boot failures.

---

## Step 3 — Apply the BSP to L4T

The AVermedia BSP tarball contains overlay files that must be copied on top of the base L4T tree **before** running `apply_binaries.sh`.

### 3.1 Extract and Overlay the BSP

```bash
cd ~/jetson_flash

# Extract the AVermedia D131L BSP
tar xf avermedia_d131l_l4t36_bsp.tar.gz

# The BSP typically contains a Linux_for_Tegra/ subdirectory.
# Copy/overlay its contents onto the existing Linux_for_Tegra/ tree.
# Adjust the source path to match the actual extracted directory name.
cp -r avermedia_d131l_bsp/Linux_for_Tegra/* Linux_for_Tegra/
```

The BSP overlay may include:

| Path (under `Linux_for_Tegra/`) | Purpose |
|---|---|
| `bootloader/` | U-Boot binaries and configuration for the D131L |
| `kernel/dtb/` | Device tree blobs (`.dtb`) and overlays (`.dtbo`) |
| `p3768-*/` or `d131l-*/` config dirs | Board configuration scripts used by `flash.sh` |
| `source/kernel/` | Kernel patches or additional drivers (if provided) |

### 3.2 Run `apply_binaries.sh`

This script installs NVIDIA-provided binaries into the root filesystem:

```bash
cd ~/jetson_flash/Linux_for_Tegra
sudo ./apply_binaries.sh
```

### 3.3 Identify the Correct Board Configuration

The `flash.sh` script requires a **board configuration name** and a **storage target**. Check the AVermedia D131L BSP release notes for the exact string. Common patterns:

```bash
# List available board configs provided by the BSP
ls ~/jetson_flash/Linux_for_Tegra/ | grep -i d131
```

The configuration name is typically something like `avermedia-d131l-orin-nano` or matches a `p3767-*` Orin Nano config modified by the BSP. Confirm with AVermedia documentation.

---

## Step 4 — Configure the OV9281 MIPI Camera

The OV9281 is an Omnivision 1 MP global shutter sensor connected via MIPI CSI-2. Enabling it requires either a device tree overlay (`.dtbo`) or a modification to the board's main device tree.

### 4.1 Check if the BSP Includes an OV9281 Overlay

Many carrier board BSPs ship with pre-built overlays for common cameras:

```bash
find ~/jetson_flash/Linux_for_Tegra/kernel/dtb/ -iname '*ov9281*'
```

If one or more `.dtbo` files are found, note their names — you will apply them during or after flashing.

### 4.2 Build the OV9281 Overlay from Source (if not pre-built)

If the BSP does not include a ready-made OV9281 overlay:

1. **Obtain the kernel source** matching your L4T revision from the [Jetson Linux source archive](https://developer.nvidia.com/embedded/jetson-linux-archive).
2. Locate or create the device tree source for the OV9281. NVIDIA's L4T source tree contains a reference overlay at:
   ```
   kernel/kernel-jammy-src/arch/arm64/boot/dts/nvidia/
   ```
   Look for `tegra234-camera-ov9281-*.dts` or adapt an existing camera overlay.
3. Key device tree properties for OV9281:

```dts
/* Example fragment — adapt port, reg, and clock values to match your hardware */
ov9281_cam0: ov9281@60 {
    compatible = "ovti,ov9281";
    reg = <0x60>;
    reset-gpios = <&gpio ...>;

    clocks = <&bpmp TEGRA234_CLK_EXTPERIPH1>;
    clock-names = "extclk";
    clock-frequency = <24000000>;

    port {
        ov9281_out0: endpoint {
            remote-endpoint = <&mipi_cal_in0>;
            data-lanes = <1 2>;
            link-frequencies = /bits/ 64 <400000000>;
        };
    };
};
```

4. Compile the overlay:

```bash
cd ~/jetson_flash/Linux_for_Tegra/source/kernel/kernel-jammy-src
# Note: the kernel source directory name varies by L4T version and Ubuntu base.
# For L4T 36.x (Ubuntu 22.04 Jammy base) it is typically 'kernel-jammy-src'.
# Verify the actual directory name with: ls ~/jetson_flash/Linux_for_Tegra/source/kernel/
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- dtbs
# The built .dtbo will appear under arch/arm64/boot/dts/nvidia/
```

5. Copy the compiled `.dtbo` to the L4T DTB directory:

```bash
cp arch/arm64/boot/dts/nvidia/tegra234-camera-ov9281-overlay.dtbo \
   ~/jetson_flash/Linux_for_Tegra/kernel/dtb/
```

### 4.3 Register the Overlay for Automatic Application at Boot

Edit (or create) the `config.txt` overlay configuration for the board so the overlay is applied at every boot:

```bash
# On the target filesystem (after flashing) this file lives at:
# /boot/extlinux/extlinux.conf  or  /boot/tegra-ubl.conf
# During the build, you can pre-populate it in the rootfs:

OVERLAY_DTBO="tegra234-camera-ov9281-overlay.dtbo"
```

Alternatively, after first boot, use `jetson-io.py` (included in L4T) to configure the overlay interactively:

```bash
# Run on the Jetson after first boot
sudo /opt/nvidia/jetson-io/jetson-io.py
```

In the menu, navigate to **Configure Jetson for compatible hardware** → select the OV9281 overlay → **Save and reboot**.

---

## Step 5 — Put the Jetson Orin Nano into Force Recovery Mode

Force Recovery Mode allows the host PC to communicate with the Jetson via USB and flash new firmware.

### 5.1 Physical Steps (AVermedia D131L)

1. Power off the board completely (remove the DC power supply).
2. Locate the **Force Recovery** button/jumper on the D131L. Refer to the [AVermedia D131L hardware guide](https://www.avermedia.com/professional/product/d131l/overview) for exact pin/button location.
3. While holding the **Force Recovery** button (or with the recovery jumper installed):
   - Connect the USB-C (or Micro-USB) flashing cable between the D131L and your host PC.
   - Apply DC power to the board.
4. Wait approximately 2 seconds, then release the Force Recovery button.

### 5.2 Verify Recovery Mode on the Host PC

```bash
lsusb | grep -i nvidia
```

You should see a line similar to:

```
Bus 00X Device 00Y: ID 0955:7523 NVIDIA Corp. APX
```

The vendor ID `0955` and product ID `7523` (or `7e19` for Orin-family devices) confirm the Jetson is in recovery mode.

---

## Step 6 — Flash the Board

### 6.1 Flash with `flash.sh` (Manual Method)

```bash
cd ~/jetson_flash/Linux_for_Tegra

# General syntax:
# sudo ./flash.sh <board-config> <storage-device>
#
# For internal eMMC:
sudo ./flash.sh avermedia-d131l-orin-nano internal

# For NVMe SSD (if rootfs is on NVMe):
sudo ./flash.sh avermedia-d131l-orin-nano nvme0n1p1
```

> Replace `avermedia-d131l-orin-nano` with the exact board configuration name from the AVermedia BSP documentation.

The flashing process takes approximately **10–20 minutes**. You will see progress output ending with:

```
Flashing completed
```

### 6.2 Flash with NVIDIA SDK Manager (GUI Method)

1. Launch `sdkmanager` on the host PC.
2. Select:
   - **Product:** Jetson
   - **Hardware:** Jetson Orin Nano (or Jetson Orin Nano 8GB / 4GB as applicable)
   - **Target OS:** JetPack 6.x
3. In the **Customize** section, point SDK Manager to the BSP directory you prepared, or let it download a base L4T and then manually overlay the AVermedia BSP files before clicking **Flash**.
4. Follow on-screen prompts. SDK Manager will detect the device in recovery mode and begin flashing.

> **Note:** SDK Manager may not natively know about the AVermedia D131L board configuration. The manual `flash.sh` approach gives you explicit control over the board config name and is the recommended method for custom carrier boards.

---

## Step 7 — First Boot and SDK Component Installation

### 7.1 First Boot

After flashing completes:

1. Disconnect the USB flashing cable.
2. Connect a display via HDMI and a USB keyboard/mouse (or use serial console via the D131L's UART header).
3. Power-cycle the board.
4. Complete the Ubuntu OOBE (Out of Box Experience): set username, password, time zone, etc.

### 7.2 Install JetPack SDK Components via `apt`

The sample root filesystem does not include CUDA, cuDNN, or TensorRT by default. Install them on the Jetson:

```bash
sudo apt update
sudo apt install -y nvidia-jetpack
```

This installs the full JetPack software stack matching the flashed L4T version, including:

- CUDA Toolkit
- cuDNN
- TensorRT
- VPI (Vision Programming Interface)
- Multimedia API
- OpenCV (with CUDA support)

> The download is several GB. Ensure the Jetson has internet access and sufficient storage.

### 7.3 Verify JetPack Version

```bash
cat /etc/nv_tegra_release
# or
jetson_release   # if jetson-stats is installed
```

---

## Step 8 — Verify the Camera

### 8.1 Check That the OV9281 Is Detected

```bash
# List V4L2 video devices
ls /dev/video*

# Check kernel messages for camera probe
dmesg | grep -i ov9281
```

Expected output in `dmesg`:

```
ov9281 10-0060: ov9281 sensor probed
```

### 8.2 Capture a Test Frame with v4l2-ctl

```bash
sudo apt install -y v4l-utils

# Query camera capabilities
v4l2-ctl --device=/dev/video0 --all

# Capture a single raw frame (RAW10 / GREY format for OV9281)
v4l2-ctl --device=/dev/video0 \
    --set-fmt-video=width=1280,height=800,pixelformat=GREY \
    --stream-mmap --stream-count=1 \
    --stream-to=/tmp/ov9281_frame.raw
```

### 8.3 Capture with GStreamer (Recommended for Production Use)

```bash
gst-launch-1.0 \
    nvarguscamerasrc sensor-id=0 ! \
    'video/x-raw(memory:NVMM),width=1280,height=800,framerate=120/1' ! \
    nvvidconv ! \
    jpegenc ! \
    filesink location=/tmp/ov9281_test.jpg
```

> The OV9281 supports up to **120 fps** at full 1280×800 resolution in RAW10 mode, making it well-suited for high-frequency optical flow on the QBot platform.

### 8.4 Test with Python / OpenCV

```python
import cv2

cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 800)

ret, frame = cap.read()
if ret:
    cv2.imwrite('/tmp/ov9281_test.png', frame)
    print("Frame captured successfully")
else:
    print("Failed to capture frame")

cap.release()
```

---

## Troubleshooting

### Jetson Not Detected in Recovery Mode

- Verify the USB cable is data-capable (not a charge-only cable).
- Try a different USB port on the host (prefer USB 3.x blue ports).
- Check that the Force Recovery button/jumper was held **before** power was applied.
- Confirm with `lsusb` — NVIDIA device with ID `0955:7e19` should appear.

### `flash.sh` Fails with "Unknown board config"

- Double-check the board configuration name. Run `ls Linux_for_Tegra/` and look for config directories/scripts starting with your board prefix.
- Ensure the AVermedia BSP was extracted **over** the L4T directory (Step 3.1), not in a separate location.

### BSP / L4T Version Mismatch

- The BSP must match the exact L4T point release (e.g., R36.3.0). Check the AVermedia BSP release notes for the supported L4T revision.
- Mixing BSP from L4T 36.2 with L4T 36.3 binaries will cause boot failures.

### Board Boots to Black Screen

- Check HDMI cable and monitor compatibility.
- Connect via UART serial console (115200 8N1) on the D131L debug UART header to see early boot messages.
- Verify the device tree is correct — wrong DTB will cause peripheral init failures.

### OV9281 Not Detected (`/dev/video0` Missing)

- Confirm the physical CSI connector and cable orientation (CSI ribbon cables are keyed but easy to reverse).
- Verify the overlay `.dtbo` was applied: `cat /boot/extlinux/extlinux.conf` should reference the overlay.
- Check `dmesg | grep -E 'csi|ov9281|camera'` for driver probe errors.
- Confirm I2C address: use `i2cdetect` to scan available buses — OV9281 default I2C address is `0x60`. First identify the correct bus by checking `dmesg | grep -i 'i2c\|cam'` for the CSI I2C adapter number, then scan it (the bus number for CSI cameras is commonly 9 or 10 on Orin, but may differ):
  ```bash
  # Identify I2C buses
  ls /dev/i2c-*
  # Scan a specific bus (replace N with the bus number from dmesg)
  sudo i2cdetect -y -r N
  ```

### OV9281 Shows Corrupted / All-Black Frames

- Verify MCLK (master clock) is 24 MHz and present on the CSI connector.
- Check CSI lane polarity in the device tree — swapped P/N lanes produce scrambled data.
- Try reducing the link frequency in the device tree (e.g., `200000000` instead of `400000000`).

### `nvidia-jetpack` apt Install Fails

- Ensure the correct NVIDIA apt repository is configured:
  ```bash
  cat /etc/apt/sources.list.d/nvidia-l4t-apt-source.list
  ```
  It should contain a line pointing to `l4t-r36.*` or the relevant L4T release.
- Run `sudo apt update` and check for GPG key errors. Re-install the NVIDIA apt keyring if needed.

---

## References

- [NVIDIA Jetson Linux (L4T) Archive](https://developer.nvidia.com/embedded/jetson-linux-archive)
- [NVIDIA SDK Manager Documentation](https://docs.nvidia.com/sdk-manager/index.html)
- [NVIDIA JetPack 6 Release Notes](https://developer.nvidia.com/embedded/jetpack-sdk-60)
- [AVermedia D131L Product Page](https://www.avermedia.com/professional/product/d131l/overview)
- [Omnivision OV9281 Datasheet](https://www.ovt.com/products/ov9281/)
- [NVIDIA Jetson Camera Architecture Guide](https://docs.nvidia.com/jetson/archives/r36.3/DeveloperGuide/SD/CameraDevelopment/CameraDevGuide.html)
- [Jetson Linux Device Tree Customization](https://docs.nvidia.com/jetson/archives/r36.3/DeveloperGuide/HR/JetsonEepromLayout.html)
- [V4L2 Camera Testing on Jetson](https://docs.nvidia.com/jetson/archives/r36.3/DeveloperGuide/SD/CameraDevelopment/CameraDevGuide.html#v4l2-camera-testing)
- [Jetson-IO Tool Documentation](https://docs.nvidia.com/jetson/archives/r36.3/DeveloperGuide/SD/CameraDevelopment/JetsonIO.html)
