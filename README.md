# MagicNeko-ROM · OnePlus 10T (ovaltine) local manifests

> Build manifests for OnePlus 10T / 一加 Ace Pro（`ovaltine`，SM8475）。
> OnePlus 10T / 一加 Ace Pro（`ovaltine`，SM8475）的构建用 local manifest。

This repo only holds the **supplementary** manifest for `.repo/local_manifests/`.
It is **not** a complete manifest — the thousands of upstream projects still come
from the official manifest.

本仓只放 `.repo/local_manifests/` 需要的**补充清单**，**不是**完整 manifest ——
上游的几千个项目仍由官方 manifest 提供。

---

## Requirements / 环境要求

| | |
|---|---|
| OS | Linux x86_64 / aarch64，≥ 400 GB free disk<br/>Linux x86_64 / aarch64，≥ 400 GB 空闲磁盘 |
| Tools | `repo`、`git`、`git-lfs`、`python3`<br/>`repo`、`git`、`git-lfs`、`python3` |
| RAM | ≥ 32 GB recommended (16 GB works with `-j` lowered)<br/>建议 ≥ 32 GB（16 GB 需降低 `-j`） |

`git-lfs` is **mandatory**. Some repos carry `.so` files larger than 50 MiB via
LFS; without it you get 130-byte pointer files and the build fails.

`git-lfs` **必须装**。部分仓有大于 50 MiB 的 `.so` 走 LFS，不装下来的是一堆
130 字节的指针文件，编不过。

---

## Quick start / 快速开始

```bash
# 1) Upstream base (git-lfs required)
#    上游基线（必须带 git-lfs）
repo init -u https://github.com/LineageOS/android.git -b lineage-23.2 --git-lfs

# 2) Drop in the supplementary manifest
#    放入补充清单
mkdir -p .repo/local_manifests
curl -L -o .repo/local_manifests/ovaltine-23.2.xml \
  https://raw.githubusercontent.com/MagicNeko-ROM/local_manifests/main/ovaltine-23.2.xml

# 3) Sync
#    同步
repo sync -c -j"$(nproc)" --force-sync --no-clone-bundle --no-tags

# 4) Build
#    构建
source build/envsetup.sh
lunch lineage_ovaltine-bp4a-userdebug
m bacon
```

Output / 产物：`out/target/product/ovaltine/AviumUI-*.zip`

---

## Manifest contents / 清单内容

All revisions are pinned to a concrete commit SHA, never a branch name, so builds
are reproducible.

所有 revision 都钉**具体 commit SHA**，不用分支名，保证产物可复现。

| Path / 路径 | Repo / 仓库 | Note / 说明 |
|---|---|---|
| `device/oneplus/ovaltine` | `android_device_oneplus_ovaltine` | Device tree / 机型设备树 |
| `device/oneplus/sm8450-common` | `android_device_oneplus_sm8450-common` | Common device tree / 共用设备树 |
| `vendor/oneplus/ovaltine` | `proprietary_vendor_oneplus_ovaltine` | **Firmware needs extra step** / **固件需另外获取** |
| `vendor/oneplus/sm8450-common` | `proprietary_vendor_oneplus_sm8450-common` | Vendor blobs / 供应商库 |
| `vendor/extra` | `android_vendor_extra` | Extra components / 额外组件 |
| `hardware/oplus` | `android_hardware_oplus` | OPlus HALs incl. fingerprint shim / 含指纹 shim |
| `hardware/qcom-caf/sm8450/display` | `android_hardware_qcom_display` | **Overrides upstream** · UDFPS HBM / **替换上游** |
| `hardware/lineage/interfaces` | `android_hardware_lineage_interfaces` | **Overrides upstream** · SDM degrade fix / **替换上游** |
| `kernel/oneplus/sm8450` | `android_kernel_oneplus_sm8450` | Kernel / 内核 |
| `kernel/oneplus/sm8450-devicetrees` | `android_kernel_oneplus_sm8450-devicetrees` | DTB / DTBO |
| `kernel/oneplus/sm8450-modules` | `android_kernel_oneplus_sm8450-modules` | Out-of-tree modules / 外编模块 |
| `kernel/oneplus/gki-modules` | `android_oneplus_kernel_gki-modules` | Prebuilt GKI modules / GKI 预编译模块 |

---

## Why two upstream repos are overridden / 为什么有两处替换上游

Both exist in the upstream manifest already. This device needs changes upstream
does not carry.

这两个仓在上游 manifest 里本来就有，但本机型需要的改动上游没有。

### 1. `hardware/qcom-caf/sm8450/display` — UDFPS HBM

The panel DTSI is `oplus,ofp-fp-type = <0x08>`, with **no BIT(4) local HBM** —
it cannot light up only the fingerprint area. The whole screen has to enter HBM,
and the circular look comes from the SystemUI dim layer. The HBM trigger lives in
`UdfpsExtension`, which the sm8450 display HAL **does not have** (only sm8550 /
sm8750 do). Without it HBM never turns on:

本面板 DTSI 是 `oplus,ofp-fp-type = <0x08>`，**没有 BIT(4) local HBM** ——
不能只点亮指纹那一小块，只能整屏进 HBM，靠 SystemUI 的压暗层压出圆形观感。
HBM 的触发点编在 `UdfpsExtension` 里，而 sm8450 的 display HAL **根本没有这个
文件**（只有 sm8550 / sm8750 有）。缺了它 HBM 永远不开：

```
hbmMode=off
[FP_HAL][OpticalCore]: checkFingerUpTooFast, FP_FINGER_UP FP_ERR_FINGERUP_TOO_FAST
[FP_HAL][IFingerprintCore]: captureImg:323: result:10108
```

`10108` = `FP_ERR_FINGERUP_TOO_FAST`. The screen is dimmed but never brightened,
so the sensor gets no usable image and enrollment always fails.

`10108` 就是 `FP_ERR_FINGERUP_TOO_FAST`。屏幕被压暗、亮度没抬起来，采不到清晰
图像，录入必然失败。

This repo additionally relaxes `CoreImpl::CreateCore`'s null-display condition:
`HWInfoDRM::Init` fails when the DRM master is already taken by SurfaceFlinger
(SF is class `main` and starts before composer's class `hal`), while
`HandleNullDisplay()` succeeds. But `enable_null_display_` gets recomputed as
`!IsDisplayHWAvailable()`, which reports "available" and contradicts the failure,
so the original composed condition rejected the intended fallback and composer
crash-looped.

本仓还放宽了 `CoreImpl::CreateCore` 的 null-display 判定：`HWInfoDRM::Init` 在
DRM master 被 SurfaceFlinger 占住时会失败（SF 属 class `main`，先于 composer 的
class `hal` 启动），而 `HandleNullDisplay()` 是成功的。但
`enable_null_display_` 会被重算成 `!IsDisplayHWAvailable()`，那个函数报「可用」，
与失败自相矛盾，于是原本的复合条件把降级路径否掉，composer 进入崩溃循环。

### 2. `hardware/lineage/interfaces/livedisplay` — degrade instead of abort

### 2. `hardware/lineage/interfaces/livedisplay` — 降级而非 abort

That change lets composer survive. But **livedisplay calls the same
`CoreImpl::Init()`**. SDM allows only one process to hold the DRM master;
composer takes it first, so livedisplay gets a null display and its
`CreateCore` fails. `SDMController.cpp` used to be:

上面的改动让 composer 活下来了。但 **livedisplay 也调同一个
`CoreImpl::Init()`**。SDM 只允许一个进程持有 DRM master，composer 先拿走，所以
livedisplay 拿到 null display，`CreateCore` 失败。而 `SDMController.cpp` 原本是：

```cpp
if (init() != android::OK) {
    LOG(FATAL) << "Failed to initialize SDM backend";   // abort
}
```

Result: SIGABRT every 5 seconds, init respawns it, Launcher never gets focus, the
system sits on `FallbackHome`.

结果每 5 秒一次 SIGABRT，init 反复重拉，Launcher 抢不到前台，系统停在
`FallbackHome`。

livedisplay is display tuning — optional. Failure should degrade, not abort
(`CONTROLLER_CHECK` already guards on `hctx_ == 0`, so degrading is safe):

livedisplay 只是显示调节，属可选组件，失败应降级而非 abort（`CONTROLLER_CHECK`
本就检查 `hctx_ == 0`，降级是安全的）：

```cpp
LOG(ERROR) << "Failed to initialize SDM backend, display control unavailable";
hctx_ = 0;
return;
```

Full write-up / 完整排查记录：
`device/oneplus/ovaltine/docs/fingerprint-fix-2026-10-29.md`

---

## Overriding an upstream project / 覆盖上游项目

`repo` merges the upstream manifest and every file in `.repo/local_manifests/`
into one list before syncing, and refuses two `<project>` entries on the same
path. A local manifest therefore cannot simply redeclare something upstream
already ships — it has to remove the upstream entry first:

`repo` 在 sync 之前会把上游 manifest 和 `.repo/local_manifests/` 里的每个文件
合并成一张表，同一个 path 出现两次就直接报错。所以本地清单**不能只是重新声明**
上游已有的项目，必须先把它摘掉：

```xml
<remove-project name="LineageOS/android_hardware_qcom_display"
                path="hardware/qcom-caf/sm8450/display" />
<project name="android_hardware_qcom_display"
         path="hardware/qcom-caf/sm8450/display" remote="mkn" revision="..." />
```

Skipping the `remove-project` fails with
`error: duplicate path <path> in .repo/manifest.xml`.
漏掉 `remove-project` 会直接报 `error: duplicate path <path>`。

**Nested paths 嵌套路径**: overwriting a *parent* project with a project that
lives *inside* it is the other trap. Upstream declares
`hardware/lineage/interfaces`; this manifest declares the same repository — the
complete `interfaces` tree, not just `livedisplay` — so its `path` must be
`hardware/lineage/interfaces`, and the upstream parent must be removed. Pointing
it at `.../interfaces/livedisplay` puts a whole copy inside the parent checkout,
and every child file shows up as untracked.
覆盖**父项目**时，`path` 必须与上游一致（写成父路径），否则整仓会被塞进子目录，
父项目 checkout 里全是未跟踪文件。

## Known limitations / 已知限制

### Firmware needs an extra step / 固件需另外获取

`proprietary_vendor_oneplus_ovaltine` on GitHub **does not contain the complete
firmware**. The original history carried `modem.img` (271 MB) and `dsp.img`
(64 MB), both over GitHub's per-file limit, so they were stripped with
`git filter-repo` and moved to Release assets.

`proprietary_vendor_oneplus_ovaltine` 在 GitHub 上**不含完整固件**。原始仓历史里
有 `modem.img`（271 MB）、`dsp.img`（64 MB），超过 GitHub 单文件限制，已用
`git filter-repo` 从历史中摘除，改走 Release 附件。

After `repo sync`, some `*.img` under `vendor/oneplus/ovaltine` will be empty or
missing. You need to:

sync 完成后，`vendor/oneplus/ovaltine` 里部分 `*.img` 是空的或缺失。需要：

1. Download the firmware archive from [Releases](../../releases)；
   从 [Releases](../../releases) 下载固件包；
2. Extract into `vendor/oneplus/ovaltine/proprietary/`；
   解压到 `vendor/oneplus/ovaltine/proprietary/`；
3. Cross-check the firmware version pins in `BoardConfigVendor.mk` and `radio/`.
   核对 `BoardConfigVendor.mk` 与 `radio/` 里的固件版本 pin。

### Easy-to-miss repo names / 容易写错的仓名

- `kernel/oneplus/gki-modules` maps to **`android_oneplus_kernel_gki-modules`**,
  **not** `android_kernel_oneplus_gki-modules`. The word order differs; getting it
  wrong makes `repo sync` fail.
- `kernel/oneplus/gki-modules` 对应 **`android_oneplus_kernel_gki-modules`**，
  **不是** `android_kernel_oneplus_gki-modules`。词序不同，写错会 sync 失败。

- `android_kernel_oneplus_sm8450-modules` is maintained on
  **`lineage-23.0-new-kmod`**, not `lineage-23.2`. The manifest pins that
  branch's tip SHA.
- `android_kernel_oneplus_sm8450-modules` 的维护分支是
  **`lineage-23.0-new-kmod`**，不是 `lineage-23.2`。清单钉的是该分支顶端 SHA。

### XML gotcha / XML 注意事项

`--` is **not allowed inside an XML comment**. If you edit the manifest and
mention a command-line flag verbatim in the header comment, `repo` will refuse
to parse the file (the error looks like `not well-formed (invalid token)` at some
line/column). Use `=` for separator lines, not `-`.

XML 注释里**不能出现 `--`**。编辑清单时若要提到命令行参数，别把带双横线的写法
原样抄进注释，否则 `repo` 直接解析失败（报错形如 `not well-formed (invalid
token)` 并给出行列）。分隔线用 `=`，不要用 `-`。
