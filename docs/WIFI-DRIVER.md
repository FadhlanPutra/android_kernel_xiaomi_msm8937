# WiFi USB Driver (RTL8188EUS) for This Kernel

Note: this kernel is based on the msm8917/msm8937 platform, which by design is shared across several Xiaomi devices (riva, rolex, ugglite, tiare, land, prada, santoni, ugg, and others, depending on which config fragment is used). This repository has only been tested on the **Xiaomi Redmi 5A** (codename **riva**). If you are using a different device from the same family, the guide below can be used as a general framework, but you will need to adjust the config steps yourself. It is not guaranteed to work out of the box.

There are two ways to use this driver:

Option A: use the prebuilt .ko file already provided in this repository. Fastest option, no cloning or building required.

Option B: build the driver yourself from source. More involved, but useful if you're on a different device or kernel build, or if you want to modify the driver source.

## Option A: Use the prebuilt driver

The `.ko` file is available as a release asset: [riva-8188eu-v1](https://github.com/FadhlanPutra/android_kernel_xiaomi_msm8937/releases/tag/riva-8188eu-v1).

### Requirements

The `.ko` file is tied to a specific kernel build, it cannot be used on a different kernel just because it's also "msm8937". Before using it, confirm that the kernel version running on your device matches exactly what is listed here:

```
4.19.325-cip132-st16-Mi8937v2-riva
```

Check the kernel version on your device:
```bash
adb shell su -c "uname -r"
```

If the result differs from the version above, this driver will not load (`insmod` will fail with `Invalid module format`). In that case, use Option B and build from the exact kernel source your device is running.

### Usage

```bash
curl -LO https://github.com/FadhlanPutra/android_kernel_xiaomi_msm8937/releases/download/riva-8188eu-v1/8188eu.ko
adb push 8188eu.ko /sdcard/
adb shell
su
cd /sdcard
insmod 8188eu.ko
dmesg | tail -30
```

Check `dmesg` to confirm the driver loaded and detected the USB WiFi device. Also check `ip link` or `iwconfig`, a new interface usually appears (often `wlan1`, since `wlan0` is typically the internal WiFi).

Do not use `insmod -f` (force load). If the version or symbols don't match, letting `insmod` reject it normally is safe. Forcing an incompatible module to load can cause a kernel panic (device reboot).

## Option B: Build the driver from source

Use this if you're on a different kernel build, a different device from the same msm8917/msm8937 family, or if you want to modify the driver source.

### Prerequisites

- **Linux** (this guide was written on **Arch Linux**, adjust package manager commands for other distros)
- ARM64 cross-compile toolchain:
  ```bash
  sudo pacman -S aarch64-linux-gnu-gcc aarch64-linux-gnu-binutils
  ```
- A rooted device that will run the exact same kernel you build (not just a "similar" old kernel, it must be identical)

### 1. Fork and clone this kernel

```bash
git clone https://github.com/FadhlanPutra/android_kernel_xiaomi_msm8937
cd android_kernel_xiaomi_msm8937
```

### 2. Set the cross-compile environment

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
```

### 3. Generate the kernel config

This kernel uses a defconfig plus config fragment system that gets merged together. The example below is for riva. If you're on a different device, check the fragments available under `arch/arm64/configs/vendor/xiaomi/msm8937/` (or `vendor/xiaomi/sdm439/` for a different chipset) and adjust the combination accordingly.

```bash
make ARCH=arm64 O=out vendor/msm8937-perf_defconfig \
  vendor/xiaomi/msm8937/common.config \
  vendor/xiaomi/msm8937/mi8917.config \
  vendor/xiaomi/msm8937/riva.config
```

If you see a `WARNING: unmet direct dependencies` message related to `ARCH_MSM89xx` at this step, it usually means your device's fragment hasn't explicitly set `CONFIG_ARCH_MSMxxxx=y` and/or hasn't disabled sibling devices in the same family. See the contents of `riva.config` in this repository as a reference for how this was resolved.

### 4. Prepare headers and scripts for building external modules

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- O=out modules_prepare
```

### 5. Record the kernel release string

```bash
cat out/include/config/kernel.release
```

Save the result (e.g. `4.19.325-cip132-st16-Mi8937v2-riva`), it's needed in step 7. This must also match `uname -r` on the device the driver will be installed on.

### 6. Clone the driver source

```bash
cd ..
git clone https://github.com/aircrack-ng/rtl8188eus -b v5.3.9
cd rtl8188eus
```

### 7. Build the driver module

```bash
make ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- \
     KSRC=/path/to/android_kernel_xiaomi_msm8937/out \
     KVER=<result-from-step-5> \
     -j$(nproc)
```

Do not override `EXTRA_CFLAGS` from the command line. This driver's Makefile internally appends include paths (such as the `include/` directory) to `EXTRA_CFLAGS`. Overriding this variable from the command line, whether using `=` or `+=`, causes GNU Make to skip all internal assignments in the Makefile, which breaks the build with errors like `drv_types.h: No such file or directory`. If you need to add custom defines, edit the driver's Makefile directly instead.

On success, you'll see:
```
LD [M]  .../rtl8188eus/8188eu.ko
```

### 8. Verify the build

```bash
ls -la 8188eu.ko
file 8188eu.ko   # should report an ARM aarch64 ELF file
```

### 9. Push and load on the device

```bash
adb push 8188eu.ko /sdcard/
adb shell
su
cd /sdcard
insmod 8188eu.ko
dmesg | tail -30
```

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `insmod: ERROR: Invalid module format` | The kernel running on the device doesn't exactly match the `KVER` used to build |
| `Unknown symbol` in `dmesg` | The kernel source used to build differs from the kernel actually running on the device |
| `drv_types.h: No such file or directory` during compile | `EXTRA_CFLAGS` was overridden from the command line (see the note in step 7) |
| `WARNING: unmet direct dependencies` during config generation | The device's config fragment is incomplete (see the note in step 3) |

Do not use `insmod -f`. If the version or symbols don't match, letting `insmod` reject the module normally is safe, the system keeps running fine, only the driver fails to load. In the worst realistic case of forcing it anyway, a kernel panic causes a reboot, not a permanent brick, since `insmod` never touches the boot or system partitions.