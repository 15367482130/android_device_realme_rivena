# android_device_realme_rivena

LineageOS device tree for the **realme GT Master Explorer Edition** (RMX3366, codename `rivena`,
Qualcomm Snapdragon 870 / SM8250-AC "kona").

> Fork of [RMX3366-Development/android_device_realme_rivena](https://github.com/RMX3366-Development/android_device_realme_rivena)
> (maintainer: ermyltsor). All original work is theirs; this fork only tracks the Android 17 bring-up.

## Branches

| Branch | Android | State |
|---|---|---|
| `lineage-24.0` | 17 | Bring-up branch. Device tree itself is unchanged vs 23.2 — the real work lives in `device/oppo/sm8250-common`. **Not compile-tested yet.** |
| `lineage-23.2` | 16 | Upstream baseline, untouched. |

## Why the device tree barely changes

The sibling tree [`LineageOS/android_device_oneplus_lemonade`](https://github.com/LineageOS/android_device_oneplus_lemonade)
has byte-identical content on `lineage-23.2` and `lineage-24.0` (same commit). Same story here:
version-specific code lives in the shared `sm8250-common` tree, not in the per-device tree.

See [PORTING-24.md](PORTING-24.md) for the current status, remaining blockers and the
recommended build/verify order.

## Building

Needs a full LineageOS 24.0 tree plus these repos:

```
device/realme/rivena          lineage-24.0
device/oppo/sm8250-common     lineage-24.0   <- 7 required Android 17 changes
kernel/oppo/sm8250            lineage-24.0
hardware/oplus                lineage-24.0
hardware/dolby                16
vendor/realme/rivena          lineage-24.0   <- must be re-extracted from a ≥ RMX3366_14.0.0.2100(CN01) dump
vendor/oppo/sm8250-common     lineage-24.0
vendor/oplus/camera           lineage-23.1
```

```bash
source build/envsetup.sh
lunch lineage_rivena-userdebug
m nothing        # cheap: validates PRODUCT_/BOARD_ vars + soong namespaces
m sepolicy
mka bacon
```

## Flashing notes

- Non-A/B (A-only) device: flash through recovery / `adb sideload`, not `fastboot flash boot`.
- Stock firmware base: **RMX3366_14.0.0.2100(CN01)** or newer.
- If a stock OTA refuses to flash afterwards: `fastboot wipe-super super_empty.img`.

## Credits

- ermyltsor and the RMX3366-Development org — original device tree, vendor blobs, kernel support
- bijoyv9 — kernel tree
- provasish — Dolby / OPLUS camera
- The LineageOS Project
