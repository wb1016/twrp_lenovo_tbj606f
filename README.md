## TWRP Device Tree for Lenovo Tab P11 (TB-J606F)

Device
=====
- **Name:** Lenovo Tab P11
- **Model:** TB-J606F
- **Codename:** tbj606f
- **SoC:** Qualcomm SM6115 (Snapdragon 460/662) — bengal platform
- **CPU:** Octa-core Cortex-A73 (arm64)
- **GPU:** Adreno 610
- **Storage:** UFS (`1d84000.ufshc`)
- **Display:** panel0-backlight
- **Android:** 11 (SDK 30)
- **Kernel:** 4.19.157 (stock, from boot image)
- **Device type:** WiFi-only tablet

Differences from twrp_device_xiaomi_juice (POCO M3)
===================================================

This device tree is based on `twrp_device_xiaomi_juice` which supports the POCO M3 (citrus).
Both devices share the same Qualcomm SM6115 (bengal) SoC, which is why the juice tree is a
suitable reference. Key differences:

### Device identity
- **juice:** Xiaomi POCO M3 (citrus/lime/pomelo/lemon) — unified multi-device tree
- **tbj606f:** Lenovo Tab P11 (tbj606f/J606F) — single device

### No unified script
- **juice:** Has `unified-juice.sh` to set device props at runtime for multiple devices sharing one tree
- **tbj606f:** No unified script needed (single device). Properties are set directly by the build system via `twrp_tbj606f.mk`

### No dtbo.img prebuilt
- **juice:** Includes `prebuilt/dtbo.img` (`BOARD_INCLUDE_RECOVERY_DTBO := true`)
- **tbj606f:** No dtbo.img — stock boot image has `recovery dtbo size: 0`, so `BOARD_INCLUDE_RECOVERY_DTBO := false`

### Kernel command line
- **juice:** Does not include `loop.max_part=7`
- **tbj606f:** Adds `loop.max_part=7` (present in stock boot image cmdline)

### Partition layout
- **juice:** Has `cust` partition (Xiaomi-specific); SD card support
- **tbj606f:** Has `lenovocust` partition (Lenovo-specific); WiFi-only tablet, no SD card slot

### Recovery fstab
- **juice:** Maps `cust` partition and includes SD card (`/dev/block/mmcblk0p1`)
- **tbj606f:** Maps `lenovocust` partition; no SD card entries; includes `metadata` partition for encryption

### twrp.flags
- **juice:** Includes Xiaomi `cust` partition and MicroSD storage
- **tbj606f:** Includes Lenovo `lenovocust` partition; no MicroSD; USB-OTG retained

### VINTF manifest
- **juice:** Contains Xiaomi-specific HALs (`vendor.xiaomi.hardware.displayfeature`, etc.)
- **tbj606f:** Removed all Xiaomi-specific HALs; retains standard Qualcomm framework HALs

### Build properties
- **juice:** `ro.adb.secure=0` in system.prop
- **tbj606f:** Same system.prop (OTG, gatekeeper SPU, ADB insecure)

### OTA assert
- **juice:** `TARGET_OTA_ASSERT_DEVICE := citrus,lime,pomelo,lemon,juice`
- **tbj606f:** `TARGET_OTA_ASSERT_DEVICE := tbj606f,J606F`

### Shared (unchanged from juice)
- Qualcomm bengal platform, Cortex-A73 architecture
- All kernel offsets, base address, page size, DTB address
- Boot header version 2
- UFS storage path (`1d84000.ufshc`)
- FBE decryption (qcom_decrypt + qcom_decrypt_fbe)
- Dynamic partitions (super partition)
- Brightness path (`/sys/class/backlight/panel0-backlight/brightness`, max 2047)
- USB gadget init (`init.recovery.usb.rc`)
- Qualcomm security binaries (`qseecomd`, keymaster, gatekeeper)

How to Build
===========

This device tree is designed for the [minimal TWRP manifest](https://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp/tree/twrp-11)
based on `android-11.0.0_r29`.

### 1. Initialize the TWRP build environment

```bash
mkdir twrp && cd twrp
repo init -u https://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp.git -b twrp-11
repo sync -j$(nproc)
```

### 2. Clone the device tree

```bash
git clone https://github.com/wb1016/twrp_lenovo_tbj606f.git device/lenovo/tbj606f
```

### 3. Clean stale directories

The minimal TWRP manifest removes several projects (VTS, CTS, hardware interfaces, etc.) that
are not needed for TWRP. However, if `repo sync` was run previously with these directories
already present, they remain on disk and cause build errors. Clean them:

```bash
# Clean stale test/VTS directories (defines VtsHalDriverDefaults, VtsHalProfilerDefaults, etc.)
rm -rf test/vts test/vts-testcase test/framework test/mlts test/suite_harness test/mts test/vti

# Clean stale hardware interfaces (references removed VTS modules)
rm -rf hardware/interfaces hardware/nxp frameworks/hardware/interfaces

# Clean stale vendor interfaces
rm -rf vendor/qcom/opensource/interfaces

# Remove the duplicate vts_stubs if it was copied from the device tree
rm -rf vts_stubs

# Clean the soong build cache
rm -rf out/soong
```

The device tree's own `vts_stubs/` directory provides stub definitions for the VTS modules
that remaining hardware projects reference.

### 4. Set up the build environment and build

The AOSP build system requires **bash**. If your default shell is fish or zsh, run bash first:

```bash
bash
source build/envsetup.sh
lunch twrp_tbj606f-eng
mka recoveryimage
```

### 5. Flash

The resulting recovery image will be at `out/target/product/tbj606f/recovery.img`.

```bash
# Boot into fastboot mode, then:
fastboot flash recovery out/target/product/tbj606f/recovery.img
```

To boot without flashing:
```bash
fastboot boot out/target/product/tbj606f/recovery.img
```

Notes
=====
- Uses the **stock kernel and DTB** from the original boot image — no custom kernel compilation needed.
- The stock kernel is extracted from `original-boot.img` using `unpack_bootimg.py` and placed in `prebuilt/`.
- If you need to update the prebuilt kernel, re-run the extraction and replace the files in `prebuilt/`.
- The `loop.max_part=7` kernel cmdline parameter is required for this device (loop device partition support).