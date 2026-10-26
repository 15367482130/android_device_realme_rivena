# 编 Android 17 还差什么

对着 `lineage-24.0` 的整条链路逐仓库查了一遍，结论先给：

| 组件 | 现状 | 还差什么 | 工作量 |
|---|---|---|---|
| 机型树 `device/realme/rivena` | ✅ 已有 24.0 分支 | 无（代码与 23.2 一致） | 0 |
| 公共树 `device/oppo/sm8250-common` | ✅ 7 个改动已提交 | 无（编译后可能补 sepolicy） | 0 |
| 内核 `kernel/oppo/sm8250` | ✅ 4.19.325 | **无**：上游同平台 23.2→24.0 提交数 **0** | 0 |
| `hardware/oplus` | ✅ 已有 24.0（上游 24.0 + 16 个 rivena 定制提交） | 无 | 0 |
| `vendor/realme/rivena` | ⚠️ 只有 23.2 | blob 无需换版本，先复用 | 0（WFD 可能补 fixup） |
| `vendor/oppo/sm8250-common` | ✅ 已有 24.0（2026-10-22） | 无 | 0 |
| `vendor/oplus/camera` | ⚠️ 只有 23.1 | 先试，编译/运行崩了再处理 | 未知 |
| sepolicy | ⚠️ 未知 | 只能靠编译日志逐条补 | 最花时间 |
| local_manifest | ✅ 已写 `local_manifests/rivena-24.0.xml` | — | 0 |

**一句话：仓库层面已经齐了，剩下的是 sepolicy 和真机验证——这两件只能靠编译日志和真机，写不出来。**

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

## 1. hardware/oplus 已经合并好

fork 相对上游 23.2 只有 **16 个提交**（sensors AIDL 多 HAL、振动波形、oplus-fwk 相机类、Doze 传感器 overlay），
共同祖先很近（2026-06-23），所以直接 rebase 到上游 24.0 **零冲突**：

```
lineage-24.0 = LineageOS/android_hardware_oplus lineage-24.0 (66 提交)
             + rivena 16 个定制提交
已推送：https://github.com/15367482130/android_hardware_oplus (lineage-24.0)
```

一个坑记下来：GitHub 拒收带 `.github/workflows` 的推送（token 缺 `workflow` scope），
所以推送前把该目录从历史里剔了（`git filter-branch --index-filter 'git rm -r --cached --ignore-unmatch .github'`），
CI 配置和编译无关，不影响功能。代价是这条分支的 commit hash 与上游不再一致。

以后同步上游：

```bash
git fetch up lineage-24.0 && git rebase up/lineage-24.0   # 16 个提交会浮在最上面
```

如果怀疑是定制提交引入的编译错误，先把清单切回纯上游 24.0（`rivena-24.0.xml` 里有注释好的备选行）。

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

## 3. sepolicy：只能打日志补（规则该放哪）

**不要**在 `device/qcom/sepolicy_vndr` 里改（那是平台策略，跟上游同步时必冲突）。
rivena 的设备规则实际上都在 **`hardware/oplus/sepolicy/qti/vendor/`** —— 那 16 个定制提交里
就有一条 `sepolicy: qti: Label sensors AIDL multihal`，说明维护者就是在这里加规则的。
新增规则放这里，跟着 hardware/oplus 的 24.0 分支一起走。

老 vendor + 新框架，`neverallow` 是最常见的拦路虎。典型会撞到的几类：

- `libion` 相关（已在公共树里加了 legacy 实现和 sepolicy include）
- `vendor_init` 写 sysfs（充电、触摸、屏幕）
- `hal_*_default` 访问新分区的属性
- 新增的 `vendor_*_service` 没有 domain

补法只有一个：编译报 `neverallow` 的那条规则，照它给的 allow 建议（日志里会写 "The following accesses
were denied" / "neverallow ... on line NNN"）往 `hardware/oplus/sepolicy/qti/vendor/` 里加，
**不要**图省事开 `permissive` 或 `SELINUX_IGNORE_NEVERALLOWS`，那样即使开机也会在真机上出各种权限崩溃。

用 `tools/avc2sepolicy.py` 可以从日志直接生成规则草稿：

```bash
tools/avc2sepolicy.py out/build-sepolicy-1026-1030.log -o rules.te
```

它会**分开**列 neverallow 违规和 avc 拒绝 —— 前者加 allow 是无效的（核心策略的硬约束，得改访问路径或建 vendor domain），
后者才是能直接补的。

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

## 6. 工具（本目录 tools/）

| 工具 | 用途 |
|---|---|
| `analyze-build-log.sh` | 把 `mka` 日志按 soong 配置 / 缺模块 / sepolicy / vintf / native / link 分类，每类给前 5 条 |
| `avc2sepolicy.py` | 从日志提取 neverallow 与 avc 拒绝，去重计数，生成 `allow` 规则草稿 |

用法：

```bash
./build-24.sh --stage sepolicy 2>&1 | tee /tmp/sep.log
tools/analyze-build-log.sh /tmp/sep.log
tools/avc2sepolicy.py /tmp/sep.log -o /tmp/rules.te
```
