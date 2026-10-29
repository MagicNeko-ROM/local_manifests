# MagicNeko-ROM · OnePlus 10T (ovaltine) local manifests

OnePlus 10T / 一加 Ace Pro（`ovaltine`，SM8475）的构建用 local manifest。

本仓只放 `.repo/local_manifests/` 需要的**补充清单**，不是完整 manifest —— 上游的几千个项目仍由官方 manifest 提供。

---

## 清单列表

| 文件 | 上游基线 | 用途 |
|---|---|---|
| `ovaltine-23.2.xml` | LineageOS `lineage-23.2` | AviumUI 16.2.2（含屏下指纹修复） |
| `ovaltine-16.0.xml` | crDroid `16.0` | crDroid 16.0 构建 |

---

## 快速开始（lineage-23.2）

```bash
# 1) 拉上游基线。必须带 --git-lfs：本仓有大于 50MiB 的 .so 走 LFS，
#    不加的话下来的是一堆 130 字节的指针文件，编不过。
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

---

## 清单内容（ovaltine-23.2.xml）

| 路径 | 来源 | 说明 |
|---|---|---|
| `device/oneplus/ovaltine` | MagicNeko-ROM | 机型设备树 |
| `device/oneplus/sm8450-common` | MagicNeko-ROM | 共用设备树 |
| `vendor/oneplus/ovaltine` | MagicNeko-ROM | 供应商库（**固件需另外获取，见下**） |
| `vendor/oneplus/sm8450-common` | MagicNeko-ROM | 供应商库 |
| `vendor/extra` | MagicNeko-ROM | 额外组件 |
| `hardware/oplus` | MagicNeko-ROM | OPlus HAL（含指纹 shim） |
| `hardware/qcom-caf/sm8450/display` | MagicNeko-ROM | **替换上游**，含 UDFPS HBM |
| `hardware/lineage/interfaces/livedisplay` | MagicNeko-ROM | **替换上游**，SDM 后端降级修复 |
| `kernel/oneplus/sm8450` | MagicNeko-ROM | 内核 |
| `kernel/oneplus/sm8450-devicetrees` | MagicNeko-ROM | DTB / DTBO |
| `kernel/oneplus/sm8450-modules` | MagicNeko-ROM | 外编内核模块 |
| `kernel/oneplus/gki-modules` | MagicNeko-ROM | GKI 预编译模块 |

所有 revision 都钉**具体 commit SHA**，不用分支名，保证产物可复现。

---

## 为什么有两处「替换上游」

这两个仓在上游 manifest 里本来就有，本机型需要改动而上游没有：

### 1. `hardware/qcom-caf/sm8450/display` — 屏下指纹 HBM

面板 DTSI 是 `oplus,ofp-fp-type = <0x08>`，**没有 BIT(4) local HBM** —— 不能只点亮指纹那一小块，只能整屏进 HBM，靠 SystemUI 的压暗层压出圆形观感。

HBM 的触发点编在 `UdfpsExtension` 里，而 sm8450 的 display HAL **根本没有这个文件**（只有 sm8550 / sm8750 有）。缺了它 HBM 永远不开：

```
hbmMode=off
[FP_HAL][OpticalCore]: checkFingerUpTooFast, FP_FINGER_UP FP_ERR_FINGERUP_TOO_FAST
[FP_HAL][IFingerprintCore]: captureImg:323: result:10108   (10108 = FP_ERR_FINGERUP_TOO_FAST)
```

屏幕被压暗、亮度没抬起来 → 采不到清晰图像 → 录入失败。

本仓在此基础上还放宽了 `CoreImpl::CreateCore` 的 null-display 判定条件：`HWInfoDRM::Init` 在 DRM master 被 SurfaceFlinger 占住时会失败（SF 属 class main，先于 composer 的 class hal 启动），而 `HandleNullDisplay()` 是成功的；但 `enable_null_display_` 会被重算成 `!IsDisplayHWAvailable()`，那个函数报「可用」，与失败自相矛盾，于是原本的条件把降级路径否掉，composer 进入崩溃循环。

### 2. `hardware/lineage/interfaces/livedisplay` — 不再 abort

上面那个改动让 composer 走通了降级路径，但 **livedisplay 也调同一个 `CoreImpl::Init()`**。SDM 只允许一个进程持有 DRM master，composer 先拿走，livedisplay 那边拿到 null display，随后 `CreateCore` 失败，而 `SDMController.cpp` 原本是：

```cpp
if (init() != android::OK) {
    LOG(FATAL) << "Failed to initialize SDM backend";   // ← abort
}
```

结果每 5 秒一次 SIGABRT，init 反复重拉，Launcher 抢不到前台，系统停在 `FallbackHome`。

livedisplay 只是显示调节，属可选组件，失败应降级而非 abort（`CONTROLLER_CHECK` 本就检查 `hctx_ == 0`，降级安全）。改成：

```cpp
LOG(ERROR) << "Failed to initialize SDM backend, display control unavailable";
hctx_ = 0;
return;
```

完整排查过程见 `device/oneplus/ovaltine/docs/fingerprint-fix-2026-10-29.md`（随 device 仓一起分发）。

---

## 已知限制

### 固件需另外获取

`proprietary_vendor_oneplus_ovaltine` 在 GitHub 上**不含完整固件**：原始仓历史里有 `modem.img`（271MB）、`dsp.img`（64MB）等超大文件，超过 GitHub 单文件限制，已用 `git filter-repo` 从历史中摘除，改走 Release 附件。

因此 sync 完成后，`vendor/oneplus/ovaltine` 里的部分 `*.img` 是**空的或缺失**。需要：

1. 从本仓 [Releases](../../releases) 下载固件包；
2. 解压到 `vendor/oneplus/ovaltine/proprietary/`；
3. 核对 `BoardConfigVendor.mk` 与 `radio/` 里的固件版本 pin。

### 另一个容易写错的仓名

`kernel/oneplus/gki-modules` 对应的仓库是 **`android_oneplus_kernel_gki-modules`**，
不是 `android_kernel_oneplus_gki-modules` —— 词序不同，写错会 sync 失败。

### 内核模块仓的分支

`android_kernel_oneplus_sm8450-modules` 的维护分支是 **`lineage-23.0-new-kmod`**，
不是 `lineage-23.2`。清单里钉的是该分支的顶端 SHA。
