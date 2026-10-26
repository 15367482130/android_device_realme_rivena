# 编 Android 17 还差什么

对着 `lineage-24.0` 的整条链路逐仓库查了一遍，结论先给：

| 组件 | 现状 | 还差什么 | 工作量 |
|---|---|---|---|
| 机型树 `device/realme/rivena` | ✅ 已有 24.0 分支 | 无（代码与 23.2 一致） | 0 |
| 公共树 `device/oppo/sm8250-common` | ✅ 7 个改动已提交 | 无（编译后可能补 sepolicy） | 0 |
| 内核 `kernel/oppo/sm8250` | ✅ 4.19.325 | **无**：上游同平台 23.2→24.0 提交数 **0** | 0 |
| `hardware/oplus` | ⚠️ fork 停在 23.2 | **换上游 24.0**（66 提交 / 123 文件） | 换源即可起编 |
| `vendor/realme/rivena` | ⚠️ 只有 23.2 | blob 无需换版本，先复用 | 0（WFD 可能补 fixup） |
| `vendor/oppo/sm8250-common` | ✅ 已有 24.0（2026-10-22） | 无 | 0 |
| `vendor/oplus/camera` | ⚠️ 只有 23.1 | 先试，编译/运行崩了再处理 | 未知 |
| sepolicy | ⚠️ 未知 | 只能靠编译日志逐条补 | 最花时间 |
| local_manifest | ✅ 已写 `local_manifests/rivena-24.0.xml` | — | 0 |

**一句话：真正卡住的只剩 `hardware/oplus` 和 sepolicy。**

## 证据

```
内核（LineageOS/android_kernel_oneplus_sm8250）
  compare lineage-23.2...lineage-24.0 → commits: 0, files: 0        ← 不用动

hardware/oplus（LineageOS/android_hardware_oplus）
  compare lineage-23.2...lineage-24.0 → commits: 66, files: 123     ← 必须跟
  主要变更：aidl/touch 接口升版（IOplusTouch.aidl / 新增 OplusTouchStatus）、
           Dolby codec2 service、oplusSensor AIDL、charger AIDL v11、hbp-daemon 删除
```

上游 24.0 仍然声明 `OPLUS_LINEAGE_LIVEDISPLAY_HAL` 这个 soong 配置命名空间
（`aidl/livedisplay/Android.bp`），所以设备树里的
`soong_config_set_bool(OPLUS_LINEAGE_LIVEDISPLAY_HAL,ENABLE_SE,false)` 不会报未知命名空间 —— 换源是安全的。

## 1. hardware/oplus 怎么处理

**先换上游**（`rivena-24.0.xml` 里已经这么写了）：直接 `LineageOS/android_hardware_oplus` 的 `lineage-24.0`。
rivena 现在用的 fork 比上游 23.2 多了 55 处定制（vibrator 波形、触摸轮询率、`oplus-fwk` 里额外的 app/bluetooth/content 类），
这些是**调优和功能补丁**，不是编译必需——先让系统编出来、开起来，再把这些逐个搬回去：

```bash
# 把 fork 的定制搬到 24.0 上（在源码树里操作）
cd hardware/oplus
git remote add mine https://github.com/RMX3366-Development/android_hardware_oplus
git fetch mine lineage-23.2
git diff HEAD..mine/lineage-23.2 -- aidl/vibrator aidl/touch oplus-fwk > /tmp/rivena-hw-tweaks.patch
git apply --3way /tmp/rivena-hw-tweaks.patch   # 冲突就按 24.0 的接口改
```

搬完再 `mka` 一遍，触摸/振动这类功能才有得测。

## 2. vendor/realme/rivena 为什么可以先不重抽

blob 是 vendor 的二进制，跟 Android 框架版本没有强绑定；维护者 23.2 的 vendor 树就是从
`RMX3366_14.0.0.2100(CN01)` 抽的，24.0 继续用同一批 blob 是常态。

唯一容易踩的是 **WFD（无线投屏）**：上游 24.0 把
`libwfdservice.so` 依赖的 `android.media.audio.common.types-V4-cpp.so` 换成了 V5。
真要用投屏，先确认依赖：

```bash
readelf -d out/target/product/rivena/vendor/lib64/libwfdservice.so | grep NEEDED
```

如果确实依赖 V4 而系统只有 V5，就在 `extract-files.py` 里加：

```python
'system_ext/lib64/libwfdservice.so': blob_fixup()
    .replace_needed('android.media.audio.common.types-V4-cpp.so',
                    'android.media.audio.common.types-V5-cpp.so'),
```

## 3. sepolicy：只能打日志补

老 vendor + 新框架，`neverallow` 是最常见的拦路虎。典型会撞到的几类：

- `libion` 相关（已在公共树里加了 legacy 实现和 sepolicy include）
- `vendor_init` 写 sysfs（充电、触摸、屏幕）
- `hal_*_default` 访问新分区的属性
- 新增的 `vendor_*_service` 没有 domain

补法只有一个：编译报 `neverallow` 的那条规则，照它给的 allow 建议（日志里会写 "The following accesses
were denied" / "neverallow ... on line NNN"）往 `device/oppo/sm8250-common/sepolicy/vendor/` 里加，
**不要**图省事开 `permissive` 或 `SELINUX_IGNORE_NEVERALLOWS`，那样即使开机也会在真机上出各种权限崩溃。

## 4. 第一次编译大概率撞到的错误（按出现顺序）

| 报错关键词 | 含义 | 处理 |
|---|---|---|
| `Unknown soong config namespace` | 某树没定义这个 namespace | 确认 hardware/oplus 用的是 24.0 分支 |
| `module "xxx" depends on undefined module` | 24.0 删了某个包 | 从 `PRODUCT_PACKAGES` 里删掉，或找替代品 |
| `neverallow` | sepolicy | 见上 |
| `VINTF` / `checkvintf` | HAL 版本与 framework 要求不符 | 改 `manifest.xml` 里的版本号，或补 HAL |
| `error: use of undeclared identifier`（HAL 源码） | 上游接口升版 | 按 24.0 的 AIDL 头重建调用 |

## 5. 时间预算（说实话）

| 阶段 | 时间 |
|---|---|
| `repo sync` 24.0（清华源，~110 GB） | 1.5~3 h |
| 首次 `m nothing` + `m sepolicy` | 20~40 min |
| 首次完整 `mka bacon` | 1~2 h（32 核） |
| 修 sepolicy / HAL 报错，跑到能开机 | **几天到两周**，取决于能投入多少时间 |
| 真机功能对齐（相机、指纹、VOOC、待机） | 再一到两周 |

所以：**"能编译出 zip" 是几小时到几天的事，"能当日常机用" 是几周的事**。这个预期先摆清楚。
