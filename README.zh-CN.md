# MagicNeko-ROM · 一加 10T（ovaltine）local manifests

> 一加 10T / 一加 Ace Pro（`ovaltine`，SM8475）的构建用 local manifest。

**其他语言：** [English](README.md)

本仓只放 `.repo/local_manifests/` 需要的**补充清单**，**不是**完整 manifest ——
上游的几千个项目仍由官方 manifest 提供。

---

## 环境要求

| | |
|---|---|
| 系统 | Linux x86_64 / aarch64，≥ 400 GB 空闲磁盘 |
| 工具 | `repo`、`git`、`git-lfs`、`python3` |
| 内存 | 建议 ≥ 32 GB（16 GB 需降低 `-j`） |

`git-lfs` **必须装**。部分仓有大于 50 MiB 的 `.so` 走 LFS，不装下来的是一堆
130 字节的指针文件，编不过。

---

## 快速开始

```bash
# 1) 上游基线（必须带 git-lfs）
repo init -u https://github.com/LineageOS/android.git -b lineage-23.2 --git-lfs

# 2) 放入补充清单
mkdir -p .repo/local_manifests
curl -L -o .repo/local_manifests/ovaltine-23.2.xml \
  https://raw.githubusercontent.com/MagicNeko-ROM/local_manifests/main/ovaltine-23.2.xml

# 3) 同步
repo sync -c -j"$(nproc)" --force-sync --no-clone-bundle --no-tags

# 4) 构建
source build/envsetup.sh
lunch lineage_ovaltine-bp4a-userdebug
m bacon
```

产物：`out/target/product/ovaltine/AviumUI-*.zip`

---

## 清单内容

所有 revision 都钉**具体 commit SHA**，不用分支名，保证产物可复现。

| 路径 | 仓库 | 说明 |
|---|---|---|
| `device/oneplus/ovaltine` | `android_device_oneplus_ovaltine` | 机型设备树 |
| `device/oneplus/sm8450-common` | `android_device_oneplus_sm8450-common` | 共用设备树 |
| `vendor/oneplus/ovaltine` | `proprietary_vendor_oneplus_ovaltine` | **固件需另外获取** |
| `vendor/oneplus/sm8450-common` | `proprietary_vendor_oneplus_sm8450-common` | 供应商库 |
| `vendor/extra` | `android_vendor_extra` | 额外组件 |
| `hardware/oplus` | `android_hardware_oplus` | 含指纹 shim 的 OPlus HAL |
| `hardware/qcom-caf/sm8450/display` | `android_hardware_qcom_display` | **替换上游** · UDFPS HBM |
| `hardware/lineage/interfaces` | `android_hardware_lineage_interfaces` | **替换上游** · SDM 降级修复 |
| `kernel/oneplus/sm8450` | `android_kernel_oneplus_sm8450` | 内核 |
| `kernel/oneplus/sm8450-devicetrees` | `android_kernel_oneplus_sm8450-devicetrees` | DTB / DTBO |
| `kernel/oneplus/sm8450-modules` | `android_kernel_oneplus_sm8450-modules` | 外编模块 |
| `kernel/oneplus/gki-modules` | `android_oneplus_kernel_gki-modules` | GKI 预编译模块 |

---

## 为什么有两处替换上游

这两个仓在上游 manifest 里本来就有，但本机型需要的改动上游没有。

### 1. `hardware/qcom-caf/sm8450/display` —— UDFPS HBM

本面板 DTSI 是 `oplus,ofp-fp-type = <0x08>`，**没有 BIT(4) local HBM** ——
不能只点亮指纹那一小块，只能整屏进 HBM，靠 SystemUI 的压暗层压出圆形观感。
HBM 的触发点编在 `UdfpsExtension` 里，而 sm8450 的 display HAL **根本没有这个
文件**（只有 sm8550 / sm8750 有）。缺了它 HBM 永远不开：

```
hbmMode=off
[FP_HAL][OpticalCore]: checkFingerUpTooFast, FP_FINGER_UP FP_ERR_FINGERUP_TOO_FAST
[FP_HAL][IFingerprintCore]: captureImg:323: result:10108
```

`10108` 就是 `FP_ERR_FINGERUP_TOO_FAST`。屏幕被压暗、亮度没抬起来，采不到清晰
图像，录入必然失败。

本仓还放宽了 `CoreImpl::CreateCore` 的 null-display 判定：`HWInfoDRM::Init` 在
DRM master 被 SurfaceFlinger 占住时会失败（SF 属 class `main`，先于 composer 的
class `hal` 启动），而 `HandleNullDisplay()` 是成功的。但
`enable_null_display_` 会被重算成 `!IsDisplayHWAvailable()`，那个函数报「可用」，
与失败自相矛盾，于是原本的复合条件把降级路径否掉，composer 进入崩溃循环。

### 2. `hardware/lineage/interfaces` —— 降级而非 abort

上面的改动让 composer 活下来了。但 **livedisplay 也调同一个
`CoreImpl::Init()`**。SDM 只允许一个进程持有 DRM master，composer 先拿走，所以
livedisplay 拿到 null display，`CreateCore` 失败。而 `SDMController.cpp` 原本是：

```cpp
if (init() != android::OK) {
    LOG(FATAL) << "Failed to initialize SDM backend";   // abort
}
```

结果每 5 秒一次 SIGABRT，init 反复重拉，Launcher 抢不到前台，系统停在
`FallbackHome`。

livedisplay 只是显示调节，属可选组件，失败应降级而非 abort（`CONTROLLER_CHECK`
本就检查 `hctx_ == 0`，降级是安全的）：

```cpp
LOG(ERROR) << "Failed to initialize SDM backend, display control unavailable";
hctx_ = 0;
return;
```

完整排查记录：`device/oneplus/ovaltine/docs/fingerprint-fix-2026-10-29.md`

---

## 覆盖上游项目

`repo` 在 sync 之前会把上游 manifest 和 `.repo/local_manifests/` 里的每个文件
合并成一张表，同一个 path 出现两次就直接报错。所以本地清单**不能只是重新声明**
上游已有的项目，必须先把它摘掉：

```xml
<remove-project name="LineageOS/android_hardware_qcom_display"
                path="hardware/qcom-caf/sm8450/display" />
<project name="android_hardware_qcom_display"
         path="hardware/qcom-caf/sm8450/display" remote="mkn" revision="..." />
```

漏掉 `remove-project` 会直接报 `error: duplicate path <path>`。

**嵌套路径**是另一个坑。用一个位于**父项目内部**的项目去覆盖父项目，看着无害其实
不是：上游声明了 `hardware/lineage/interfaces`，本清单声明的是同一个仓 —— 完整的
`interfaces` 树，不只是 `livedisplay` —— 所以它的 `path` 必须是
`hardware/lineage/interfaces`，同时把上游父项目摘掉。写成
`.../interfaces/livedisplay` 会把整仓塞进父项目的 checkout 里，子文件全部变成未
跟踪文件。

---

## 已知限制

### 固件需另外获取

`proprietary_vendor_oneplus_ovaltine` 在 GitHub 上**不含完整固件**。原始仓历史里
有 `modem.img`（271 MB）、`dsp.img`（64 MB），超过 GitHub 单文件限制，已用
`git filter-repo` 从历史中摘除，改走 Release 附件。

sync 完成后，`vendor/oneplus/ovaltine` 里部分 `*.img` 是空的或缺失。需要：

1. 从 [Releases](../../releases) 下载固件包；
2. 解压到 `vendor/oneplus/ovaltine/proprietary/`；
3. 核对 `BoardConfigVendor.mk` 与 `radio/` 里的固件版本 pin。

### 容易写错的仓名

- `kernel/oneplus/gki-modules` 对应 **`android_oneplus_kernel_gki-modules`**，
  **不是** `android_kernel_oneplus_gki-modules`。词序不同，写错会 sync 失败。

- `android_kernel_oneplus_sm8450-modules` 的维护分支是
  **`lineage-23.0-new-kmod`**，不是 `lineage-23.2`。清单钉的是该分支顶端 SHA。

### XML 注意事项

XML 注释里**不能出现 `--`**。编辑清单时若要提到命令行参数，别把带双横线的写法
原样抄进注释，否则 `repo` 直接解析失败（报错形如 `not well-formed (invalid
token)` 并给出行列）。分隔线用 `=`，不要用 `-`。

### 驱动 blob 与 Git LFS

`proprietary_vendor_oneplus_sm8450-common` 里 `gpu-q3/` 下的第三方 Adreno 驱动
曾被转成 Git LFS，但那些 LFS 对象从未到达服务器 —— 新 clone 拿到的是 132~133
字节的指针，`repo sync` 在 smudge 阶段直接失败。现在它们按普通 git 对象存储
（最大 55 MB，低于 GitHub 100 MB 硬限），**这棵树不再需要 git-lfs**。push 时可能
出现 50 MB 以上的软警告（GH001），无害。
