# rivena 的 Android 17（LineageOS 24.0）设备树移植包

**已开源：**
- https://github.com/15367482130/android_device_realme_rivena （默认分支 lineage-24.0）
- https://github.com/15367482130/android_device_oppo_sm8250-common （默认分支 lineage-24.0）
- https://github.com/15367482130/android_hardware_oplus （lineage-24.0 = 上游 24.0 + 16 个 rivena 定制提交）
- https://github.com/15367482130/android_kernel_realme_sm8250 （lineage-24.0，与 23.2 同 commit）

给 **真我 GT 大师探索版（RMX3366 / rivena）** 做的 `lineage-24.0` 设备树，含两棵树 + 8 个补丁。
**不是凭空写的**：逐条对照了官方 `LineageOS/android_device_oneplus_sm8250-common` 从
`lineage-23.2` 到 `lineage-24.0` 的真实 diff —— 一加 8/8T/9R 是同一颗 SM8250（kona）平台，
它的 24.0 迁移就是 rivena 的参考答案。

## 一句话结论

**设备树本体几乎不用改，Android 17 的活儿全在公共树 `sm8250-common` 上。**
一加的 `lemonade`（机型树）在 23.2 和 24.0 两个分支上内容完全一致（同一个 commit `e8403eb`），
变的只有 `sm8250-common`。rivena 同理：机型树只加了一份状态说明，7 个功能性改动都在公共树。

## 包里有什么

```
rivena-android17-dt/
├── rivena-lineage-24.0.tar.gz          整棵机型树（含 .git，分支 lineage-24.0）
├── sm8250-common-lineage-24.0.tar.gz   整棵公共树（含 .git，7 个提交）
├── patches/sm8250-common/              8 个补丁（7 个功能性改动 + README）
├── patches/rivena/                     2 个补丁（移植说明 + README）
├── push.sh                             推到 GitHub 的一键脚本（需 GITHUB_TOKEN）
├── WHAT-ELSE.md                        还差什么：逐仓库缺口、证据、时间预算
├── local_manifests/rivena-24.0.xml     24.0 完整依赖清单（8 个仓库）
├── build-24.sh                         24.0 分阶段构建脚本（config→sepolicy→boot→full）
├── tools/analyze-build-log.sh          编译日志分类（soong/缺模块/sepolicy/vintf/native/link）
├── tools/avc2sepolicy.py               从日志生成 SELinux allow 规则草稿
└── README.md
```

## 7 个改动逐条说明

| # | 提交 | 改了什么 | 为什么（Android 17 的实际变化） |
|---|---|---|---|
| 1 | `Drop TARGET_KERNEL_NO_GCC` | `BoardConfigCommon.mk` 删一行 | 内核编译已是纯 clang 流程，新 build 系统不再读这个开关 |
| 2 | `Switch to the legacy libion implementation` | 引入 `device/lineage/sepolicy/libion/sepolicy.mk` + `soong_config_set_bool(libion, legacy_impl, true)` | Android 17 把 libion 拆成可配置实现，老 vendor blob 走 ION ioctl 必须显式选 legacy，否则 sepolicy neverallow 卡编译 |
| 3 | `Drop explicit fastbootd package` | `common.mk` 去掉 `fastbootd` | 已由 fastboot HAL 依赖自动带出 |
| 4 | `Install the recovery init rc as its own module` | `init.oplus.hw.rc.recovery` → `init.oplus.hw.recovery.rc`，并在 `init/Android.bp` 拆出 `recovery: true` 的 prebuilt_etc | `recovery_available` 语义变了，不再把模块塞进 recovery 镜像 |
| 5 | `Drop PRODUCT_ENABLE_UFFD_GC` | 删产品变量 | 新版本 UFFD GC 已是默认行为 |
| 6 | `Update the Bluetooth audio policy include` | `bluetooth_hearing_aid_...xml` → `bluetooth_audio_policy_configuration.xml` | 助听器策略合并进通用 BT Audio HAL，旧文件已不存在，继续引用会解析失败 |
| 7 | `Drop the advanced reboot actions overlay` | 删 `config_restartActionsList` | LOS 24 的 lineage-sdk 不再提供该资源，RRO 引用不存在的资源会编译失败 |

机型树 `rivena`：加了一份 `PORTING-24.md`（状态、依赖、验证顺序、刷机注意事项），代码未动。

## 怎么用

**方式 A：整棵树替换（推荐）**

```bash
mkdir -p ~/android/los24/device/realme ~/android/los24/device/oppo
tar xzf rivena-lineage-24.0.tar.gz -C ~/android/los24/device/realme
tar xzf sm8250-common-lineage-24.0.tar.gz -C ~/android/los24/device/oppo
```

**方式 B：在你自己的 fork 上打补丁**

```bash
cd device/oppo/sm8250-common && git checkout -b lineage-24.0 lineage-23.2 \
    && git am /path/patches/sm8250-common/*.patch
cd device/realme/rivena      && git checkout -b lineage-24.0 lineage-23.2 \
    && git am /path/patches/rivena/*.patch
```

```bash
git remote add mine git@github.com:<你的账号>/android_device_oppo_sm8250-common.git
git push mine lineage-24.0
```

然后把 `local_manifests/rivena.xml` 里的 `revision="lineage-23.2"` 全部改成 `lineage-24.0`，
用 `build.sh -b lineage-24.0` 起编。

## 已验证 vs 未验证

**已验证**：7 个改动与官方一加 sm8250-common 24.0 的对应改动逐行一致；
文件路径、模块名、soong 语法均按官方 24.0 原文照抄；
补丁在 `lineage-23.2` 上实测 `git am` 全过，打完后与源分支逐文件比对完全一致。

**未验证**：**没有跑过 24.0 的编译器**。沙箱里同时在编 Android 16，磁盘只剩 ~200 GB，
再拉一份 24.0 源码会直接爆盘。所以这份树是"编译起点"，不是"能开机的 ROM"。

第一次编译必然会撞上的三类问题（详见机型树里的 `PORTING-24.md`）：

1. **vendor blobs**：`vendor/realme/rivena` 要从 RMX3366 底包（≥ 14.0.0.2100 CN01）重抽；
   `libwfdservice.so` 大概率要做 `types-V4 → V5` 的 `replace_needed`（用 `readelf -d` 核实）。
2. **sepolicy**：`device/oppo/sm8250-common/sepolicy` 按 neverallow 逐条补，只能看日志。
3. **HAL**：`manifest.xml` 里的 HIDL keymaster 4.1 / secure_element 1.2 是否仍被 24.0 提供。

## 现实提醒

这份东西能省掉你"从 23.2 硬抬"最费时间的排查，但**不等于移植完成**。
真机上相机、指纹、VOOC 充电、待机功耗都要逐个验，周期按周算。
如果只是想用新系统，Android 16 的 23.2 仍是 rivena 的稳态版本。
