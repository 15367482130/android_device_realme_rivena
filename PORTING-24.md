# rivena — LineageOS 24.0（Android 17）移植状态

本分支 `lineage-24.0` 从 `lineage-23.2`（Android 16）拉出，用于跟进 Android 17。

## 已完成

设备树本身几乎没有版本相关代码，真正吃改动的是同平台的公共树
`device/oppo/sm8250-common`（见该仓库 `lineage-24.0` 分支的 7 个提交）。
本仓库只补了这份状态说明，代码与 23.2 一致 —— 对照
`LineageOS/android_device_oneplus_lemonade` 的 `lineage-24.0` 也是同样情况（内容未变）。

## 依赖

```
device/realme/rivena            lineage-24.0   ← 本仓库
device/oppo/sm8250-common       lineage-24.0   ← 7 个必要改动已提交
kernel/oppo/sm8250              lineage-24.0   ← 尚未验证，4.19.325
hardware/oplus                  lineage-24.0   ← 尚未验证
hardware/dolby                  16
vendor/realme/rivena            lineage-24.0   ← 需从 RMX3366 底包重抽
vendor/oppo/sm8250-common       lineage-24.0   ← 平台侧已有该分支
vendor/oplus/camera             lineage-23.1
```

## 还没解决（编译时会依次撞上）

1. **vendor blobs**：`vendor/realme/rivena` 的 24.0 分支必须重新跑
   `extract-files.py`（底包 ≥ RMX3366_14.0.0.2100(CN01)）。WFD 相关的
   `libwfdservice.so` 在 Android 17 上大概率要做
   `replace_needed('android.media.audio.common.types-V4-cpp.so', '...-V5-cpp.so')`，
   参考一加 sm8250-common 24.0 的做法，但**必须以 `readelf -d` 的实际结果为准**。
2. **sepolicy**：`device/oppo/sm8250-common/sepolicy` 需要按 24.0 的 neverallow 逐条补，
   这块只能靠编译日志，没法预先写完。
3. **HAL 清单**：`manifest.xml` 里 HIDL keymaster 4.1 / secure_element 1.2 /
   `vendor.qti.esepowermanager` 1.1 —— 确认 24.0 仍提供这些 HIDL 实现。
4. **相机**：`vendor/oplus/camera` 还停在 23.1，camera provider 的 AIDL 版本要对齐。
5. **内核**：4.19 非 GKI，主要看 cgroup v2 / BPF 相关 backport 是否齐。

## 验证顺序（别一上来就 brunch）

```bash
source build/envsetup.sh
lunch lineage_rivena-userdebug
m nothing          # 配置层：PRODUCT_/BOARD_ 变量、soong namespace
m sepolicy         # 策略层：错误最密集
mka bootimage      # 内核 + ramdisk
mka systemimage vendorimage productimage
mka bacon
```

## 刷机前必须做的

- 底包先刷到 RMX3366_14.0.0.2100(CN01) 或更新
- 非 A/B 机型，走 recovery / sideload，别用 fastboot flash boot
- 首次移植版**不要当主力机**，先在备用机上验证通话、相机、充电、指纹
