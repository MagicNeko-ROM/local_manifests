# MagicNeko-ROM · OnePlus 10T (ovaltine) local manifests

> Build manifests for OnePlus 10T / OnePlus Ace Pro (`ovaltine`, SM8475).

**Other languages:** [Chinese](README.zh-CN.md)

This repo only holds the **supplementary** manifest for `.repo/local_manifests/`.
It is **not** a complete manifest — the thousands of upstream projects still come
from the official manifest.

---

## Requirements

| | |
|---|---|
| OS | Linux x86_64 / aarch64, ≥ 400 GB free disk |
| Tools | `repo`, `git`, `git-lfs`, `python3` |
| RAM | ≥ 32 GB recommended (16 GB works with `-j` lowered) |

`git-lfs` is **mandatory**. Some repos carry `.so` files larger than 50 MiB via
LFS; without it you get 130-byte pointer files and the build fails.

---

## Quick start

```bash
# 1) Upstream base (git-lfs required)
repo init -u https://github.com/LineageOS/android.git -b lineage-23.2 --git-lfs

# 2) Drop in the supplementary manifest
mkdir -p .repo/local_manifests
curl -L -o .repo/local_manifests/ovaltine-23.2.xml \
  https://raw.githubusercontent.com/MagicNeko-ROM/local_manifests/main/ovaltine-23.2.xml

# 3) Sync
repo sync -c -j"$(nproc)" --force-sync --no-clone-bundle --no-tags

# 4) Build
source build/envsetup.sh
lunch lineage_ovaltine-bp4a-userdebug
m bacon
```

Output: `out/target/product/ovaltine/AviumUI-*.zip`

---

## Manifest contents

All revisions are pinned to a concrete commit SHA, never a branch name, so builds
are reproducible.

| Path | Repo | Note |
|---|---|---|
| `device/oneplus/ovaltine` | `android_device_oneplus_ovaltine` | Device tree |
| `device/oneplus/sm8450-common` | `android_device_oneplus_sm8450-common` | Common device tree |
| `vendor/oneplus/ovaltine` | `proprietary_vendor_oneplus_ovaltine` | **Firmware needs an extra step** |
| `vendor/oneplus/sm8450-common` | `proprietary_vendor_oneplus_sm8450-common` | Vendor blobs |
| `vendor/extra` | `android_vendor_extra` | Extra components |
| `hardware/oplus` | `android_hardware_oplus` | OPlus HALs incl. fingerprint shim |
| `hardware/qcom-caf/sm8450/display` | `android_hardware_qcom_display` | **Overrides upstream** · UDFPS HBM |
| `hardware/lineage/interfaces` | `android_hardware_lineage_interfaces` | **Overrides upstream** · SDM degrade fix |
| `kernel/oneplus/sm8450` | `android_kernel_oneplus_sm8450` | Kernel |
| `kernel/oneplus/sm8450-devicetrees` | `android_kernel_oneplus_sm8450-devicetrees` | DTB / DTBO |
| `kernel/oneplus/sm8450-modules` | `android_kernel_oneplus_sm8450-modules` | Out-of-tree modules |
| `kernel/oneplus/gki-modules` | `android_oneplus_kernel_gki-modules` | Prebuilt GKI modules |

---

## Why two upstream repos are overridden

Both exist in the upstream manifest already. This device needs changes upstream
does not carry.

### 1. `hardware/qcom-caf/sm8450/display` — UDFPS HBM

The panel DTSI is `oplus,ofp-fp-type = <0x08>`, with **no BIT(4) local HBM** —
it cannot light up only the fingerprint area. The whole screen has to enter HBM,
and the circular look comes from the SystemUI dim layer. The HBM trigger lives in
`UdfpsExtension`, which the sm8450 display HAL **does not have** (only sm8550 /
sm8750 do). Without it HBM never turns on:

```
hbmMode=off
[FP_HAL][OpticalCore]: checkFingerUpTooFast, FP_FINGER_UP FP_ERR_FINGERUP_TOO_FAST
[FP_HAL][IFingerprintCore]: captureImg:323: result:10108
```

`10108` = `FP_ERR_FINGERUP_TOO_FAST`. The screen is dimmed but never brightened,
so the sensor gets no usable image and enrollment always fails.

This repo additionally relaxes `CoreImpl::CreateCore`'s null-display condition:
`HWInfoDRM::Init` fails when the DRM master is already taken by SurfaceFlinger
(SF is class `main` and starts before composer's class `hal`), while
`HandleNullDisplay()` succeeds. But `enable_null_display_` gets recomputed as
`!IsDisplayHWAvailable()`, which reports "available" and contradicts the failure,
so the original composed condition rejected the intended fallback and composer
crash-looped.

### 2. `hardware/lineage/interfaces` — degrade instead of abort

That change lets composer survive. But **livedisplay calls the same
`CoreImpl::Init()`**. SDM allows only one process to hold the DRM master;
composer takes it first, so livedisplay gets a null display and its
`CreateCore` fails. `SDMController.cpp` used to be:

```cpp
if (init() != android::OK) {
    LOG(FATAL) << "Failed to initialize SDM backend";   // abort
}
```

Result: SIGABRT every 5 seconds, init respawns it, Launcher never gets focus, the
system sits on `FallbackHome`.

livedisplay is display tuning — optional. Failure should degrade, not abort
(`CONTROLLER_CHECK` already guards on `hctx_ == 0`, so degrading is safe):

```cpp
LOG(ERROR) << "Failed to initialize SDM backend, display control unavailable";
hctx_ = 0;
return;
```

Full write-up: `device/oneplus/ovaltine/docs/fingerprint-fix-2026-10-29.md`

---

## Overriding an upstream project

`repo` merges the upstream manifest and every file in `.repo/local_manifests/`
into one list before syncing, and refuses two `<project>` entries on the same
path. A local manifest therefore cannot simply redeclare something upstream
already ships — it has to remove the upstream entry first:

```xml
<remove-project name="LineageOS/android_hardware_qcom_display"
                path="hardware/qcom-caf/sm8450/display" />
<project name="android_hardware_qcom_display"
         path="hardware/qcom-caf/sm8450/display" remote="mkn" revision="..." />
```

Skipping the `remove-project` fails with
`error: duplicate path <path> in .repo/manifest.xml`.

**Nested paths** are the other trap. Overwriting a *parent* project with a
project that lives *inside* it looks harmless but is not: upstream declares
`hardware/lineage/interfaces`, and this manifest declares the same repository —
the complete `interfaces` tree, not just `livedisplay` — so its `path` must be
`hardware/lineage/interfaces`, and the upstream parent must be removed. Pointing
it at `.../interfaces/livedisplay` puts a whole copy inside the parent checkout,
and every child file shows up as untracked.

---

## Known limitations

### Firmware needs an extra step

`proprietary_vendor_oneplus_ovaltine` on GitHub **does not contain the complete
firmware**. The original history carried `modem.img` (271 MB) and `dsp.img`
(64 MB), both over GitHub's per-file limit, so they were stripped with
`git filter-repo` and moved to Release assets.

After `repo sync`, some `*.img` under `vendor/oneplus/ovaltine` will be empty or
missing. You need to:

1. Download the firmware archive from [Releases](../../releases);
2. Extract into `vendor/oneplus/ovaltine/proprietary/`;
3. Cross-check the firmware version pins in `BoardConfigVendor.mk` and `radio/`.

### Easy-to-miss repo names

- `kernel/oneplus/gki-modules` maps to **`android_oneplus_kernel_gki-modules`**,
  **not** `android_kernel_oneplus_gki-modules`. The word order differs; getting it
  wrong makes `repo sync` fail.

- `android_kernel_oneplus_sm8450-modules` is maintained on
  **`lineage-23.0-new-kmod`**, not `lineage-23.2`. The manifest pins that
  branch's tip SHA.

### XML gotcha

`--` is **not allowed inside an XML comment**. If you edit the manifest and
mention a command-line flag verbatim in the header comment, `repo` will refuse
to parse the file (the error looks like `not well-formed (invalid token)` at some
line/column). Use `=` for separator lines, not `-`.

### Driver blobs and Git LFS

The third-party Adreno driver files under `gpu-q3/` in
`proprietary_vendor_oneplus_sm8450-common` were once converted to Git LFS, but
those LFS objects never reached the server — a fresh clone got 132-133 byte
pointers and `repo sync` died at the smudge stage. They are now tracked as
regular git objects (largest 55 MB, under GitHub's 100 MB limit), so **git-lfs is
not needed for this tree anymore**. Pushes may still show a soft warning above
50 MB (GH001); that is harmless.
