## TWRP Device Tree for Lenovo Tab P11 (TB-J606F)

Device
=====
- **Name:** Lenovo Tab P11
- **Model:** TB-J606F
- **Codename:** tbj606f
- **SoC:** Qualcomm SM6115 (Snapdragon 460/662) — bengal platform
- **CPU:** Octa-core Cortex-A73 (arm64)
- **GPU:** Adreno 610
- **Storage:** UFS (`4804000.ufshc`), SD card (`mmcblk0`, ~125GB)
- **Display:** Himax HX83102P BOE 2K video mode DSI panel (1200x2000, mounted upside-down)
- **Touch:** Himax HX83102 touchscreen + pen (SPI), Elan HID touchpad (I2C, keyboard accessory)
- **Android:** 11 (SDK 30)
- **Kernel:** 4.19.157-perf+ (stock prebuilt, from boot image)
- **Device type:** WiFi-only tablet (A/B, dynamic partitions)

Current Status
==============

### Working
- **Boot:** Recovery image boots correctly via `fastboot boot` or `fastboot flash recovery`
- **Display:** Correct 180° rotation (panel is physically mounted upside-down, `TW_ROTATION := 180`)
- **Touch:** Full touchscreen + pen support (Himax firmware included in recovery image)
- **ADB:** USB gadget working via explicit ConfigFS + direct UDC bind
- **Battery/clock:** Charging indicator, CPU temperature, clock, power button all functional
- **Unencrypted partitions:** `system_ext`, `product`, `metadata`, `lenovocust`, `persist` mount fine
- **SD card:** Detected and mountable (`/dev/block/mmcblk0p1`)

### Known Limitations

#### Cannot mount `/system`, `/vendor`, or `/data` (kernel limitation)

The stock kernel's ext4 and f2fs drivers are compiled **without encryption support**:
- `CONFIG_EXT4_FS_ENCRYPTION` is **not set** → system/vendor (encrypted ext4) fail to mount
- `CONFIG_F2FS_FS_ENCRYPTION` is **not set** → /data (encrypted f2fs) fails to mount

The kernel has the core encryption framework (`CONFIG_FS_ENCRYPTION=y`,
`CONFIG_BLK_INLINE_ENCRYPTION=y`, `CONFIG_FS_ENCRYPTION_INLINE_CRYPT=y`) with 64 fscrypt
symbols, but the per-filesystem encryption drivers are not enabled. Stock Android handles
encryption through hardware inline crypto (ICE) at the UFS controller level, which isn't
initialized in recovery.

The system and vendor partitions have `EXT4_FEATURE_INCOMPAT_ENCRYPT` (0x4000) set in their
ext4 superblock. The kernel refuses to mount them:
```
EXT4-fs (dm-0): couldn't mount RDWR because of unsupported optional features (4000)
```

#### Why the stock recovery works despite missing encryption

The stock recovery **never mounts encrypted filesystems**. It operates on **raw block devices**
for OTA updates — writing directly to partitions without mounting them as filesystems. TWRP
is fundamentally different: it needs to mount filesystems to provide its GUI, file management,
and backup/restore functionality.

#### Kernel investigation — recovery images compared

Both recovery images from the latest firmware update use the **exact same kernel** as the boot
image. Neither has per-filesystem encryption support:

| | recovery_a.img | recovery_b.img | boot image |
|---|---|---|---|
| Kernel size | 15,263,704 | 15,264,834 | 15,263,704 |
| Kernel version | 4.19.157-perf+ | 4.19.157-perf+ | 4.19.157-perf+ |
| Build date | Jun 3, 2024 | Mar 2, 2024 | Jun 3, 2024 |
| Build server | bjsl152 | bjsl158 | bjsl152 |
| OS patch level | 2024-05 | 2023-02 | — |
| fscrypt symbols | 64 | 64 | 64 |
| ext4/f2fs enc funcs | 0 | 0 | 0 |
| recovery DTBO | 1,886,746 | 1,886,746 | 0 (in boot hdr) |

All three kernels have the VFS-level fscrypt framework (64 symbols) but **zero** ext4/f2fs
encryption function implementations. The encryption limitation is consistent across all
firmware versions.

#### MTP

MTP can be enabled from TWRP Settings → System, but without /data mounted (encrypted),
there is no internal storage to share. The SD card can serve as MTP storage once mounted.

#### Building a kernel with encryption support

To fix the mounting issues, the kernel needs to be rebuilt with:
```
CONFIG_EXT4_FS_ENCRYPTION=y
CONFIG_F2FS_FS_ENCRYPTION=y
```

This requires kernel source code. The following options were investigated:

| Source | Status |
|--------|--------|
| Lenovo Android 11 kernel | **Not open-sourced** — only Android 10 source available |
| Lenovo Android 10 kernel | **Unbuildable** — code hacks, python2 dependency, obsolete toolchain |
| Qualcomm SM6115 kernel | **Not suitable** — targeted at phones, not this tablet form factor |

If kernel source becomes available, add the two config options above to the defconfig and
build with `mka recoveryimage`, then replace `prebuilt/kernel`.

Files Modified (from juice device tree)
========================================

| File | Change |
|------|--------|
| `BoardConfig.mk` | Added `BOARD_INCLUDE_RECOVERY_DTBO`, `BOARD_PREBUILT_DTBOIMAGE`, `BOARD_DTB_OFFSET`, fixed partition sizes (96MB), `TW_ROTATION := 180`, `TW_EXCLUDE_DEFAULT_USB_INIT := true` |
| `prebuilt/dtbo.img` | Recovery DTBO extracted from stock recovery image (1.8MB) |
| `recovery/root/init.recovery.usb.rc` | Minimal USB gadget with explicit ConfigFS + FunctionFS mount + direct UDC bind |
| `recovery/root/init.recovery.qcom.rc` | Health HAL service + block device symlinks |
| `recovery/root/vendor/firmware/Himax_firmware.bin` | Touch firmware from device dump (128KB) |
| `recovery/root/vendor/firmware/Himax_mpfw.bin` | Touch firmware from device dump (128KB) |
| `recovery.fstab` | Uses explicit `/dev/block/mapper/*_a` paths for dynamic partitions; SD card entry |

### ADB USB Init Details

The USB init uses a two-phase explicit approach in `init.recovery.usb.rc`:
- **`on init`**: Mounts ConfigFS, creates minimal gadget with only ADB function, mounts FunctionFS at `/dev/usb-ffs/adb`
- **`on boot`**: Sets USB controller (`4e00000.dwc3`), writes directly to UDC to bind the gadget, starts adbd

This bypasses TWRP's default USB init (`TW_EXCLUDE_DEFAULT_USB_INIT := true`) because the
default init's auto-detection and property trigger mechanism didn't work correctly on this device.

### Recovery fstab Details

The fstab uses explicit device-mapper paths for dynamic partitions:
```
/dev/block/mapper/system_a     /system       ext4  ro,barrier=1,discard  wait
/dev/block/mapper/system_ext_a /system_ext   ext4  ro,barrier=1,discard  wait
/dev/block/mapper/product_a    /product      ext4  ro,barrier=1,discard  wait
/dev/block/mapper/vendor_a     /vendor       ext4  ro,barrier=1,discard  wait
/dev/block/by-name/userdata    /data         f2fs  ...                   latemount,wait,...
/dev/block/mmcblk0p1           /sdcard       vfat  defaults              voldmanaged=sdcard1:auto
```

The `wait,logical` format (used by the juice device) was tried but didn't work — TWRP couldn't
resolve the mapper paths. Using explicit `/dev/block/mapper/*_a` paths resolved `system_ext`
and `product` mounting. `system` and `vendor` still fail due to the encryption limitation.

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

### Recovery DTBO
- **juice:** Includes `prebuilt/dtbo.img` (`BOARD_INCLUDE_RECOVERY_DTBO := true`)
- **tbj606f:** Now includes `prebuilt/dtbo.img` extracted from stock recovery image (was initially missing, caused build failure)

### Kernel command line
- **juice:** Does not include `loop.max_part=7`
- **tbj606f:** Adds `loop.max_part=7` (present in stock boot image cmdline)

### Partition layout
- **juice:** Has `cust` partition (Xiaomi-specific); SD card support
- **tbj606f:** Has `lenovocust` partition (Lenovo-specific); SD card detected (`mmcblk0`, ~125GB)

### Recovery fstab
- **juice:** Maps `cust` partition; uses `wait,logical` for dynamic partitions
- **tbj606f:** Maps `lenovocust` and `metadata` partitions; uses explicit `/dev/block/mapper/*_a` paths; includes SD card entry

### Display
- **juice:** Standard orientation
- **tbj606f:** Panel mounted upside-down, requires `TW_ROTATION := 180`

### USB init
- **juice:** Uses TWRP's default USB init
- **tbj606f:** Uses custom `init.recovery.usb.rc` with explicit ConfigFS + UDC bind (`TW_EXCLUDE_DEFAULT_USB_INIT := true`)

### Touch firmware
- **juice:** No additional firmware needed
- **tbj606f:** Requires `Himax_firmware.bin` and `Himax_mpfw.bin` in `/vendor/firmware/`

### OTA assert
- **juice:** `TARGET_OTA_ASSERT_DEVICE := citrus,lime,pomelo,lemon,juice`
- **tbj606f:** `TARGET_OTA_ASSERT_DEVICE := tbj606f,J606F`

### Shared (unchanged from juice)
- Qualcomm bengal platform, Cortex-A73 architecture
- All kernel offsets, base address, page size, DTB address
- Boot header version 2
- UFS storage path
- FBE decryption (qcom_decrypt + qcom_decrypt_fbe)
- Dynamic partitions (super partition)
- Brightness path (`/sys/class/backlight/panel0-backlight/brightness`, max 2047)
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

The AOSP build system requires **bash**. If your default shell is fish or zsh, run bash first.

**Important:** `ALLOW_MISSING_DEPENDENCIES=true` is required. The minimal TWRP manifest removes
many repos (test frameworks, car modules, benchmark tools, etc.) that are not needed for recovery.
Without this flag, the build will fail on undefined modules from those removed repos. With it,
the build system skips any modules whose dependencies are not available, and only builds what is
needed for the recovery image.

```bash
bash
export ALLOW_MISSING_DEPENDENCIES=true
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
- Recovery DTBO is extracted from the stock recovery image (not the boot image) because the boot image reports `recovery dtbo size: 0`.
- The stock recovery kernel and the latest firmware recovery kernels (A and B slots) all use the same kernel binary with identical encryption limitations.