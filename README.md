# MagicNeko-ROM · OnePlus 10T (ovaltine) manifests

crDroid 16.0（Android 16）在 OnePlus 10T / 一加 Ace Pro（`ovaltine`，SM8475 / cape）上的构建用 local manifest。

本仓只放 `.repo/local_manifests/` 需要的补充清单，**不是**完整 manifest —— 上游 crDroid 的几千个项目仍由官方 manifest 提供。

## 快速开始

```bash
# 1) 拉上游 crDroid 16.0（必须带 --git-lfs，否则 webview.apk 等是 LFS 指针，编不过）
repo init -u https://github.com/crdroidandroid/android.git -b 16.0 --git-lfs

# 2) 放入本仓的补充清单
mkdir -p .repo/local_manifests
curl -L -o .repo/local_manifests/ovaltine-16.0.xml \
  https://raw.githubusercontent.com/MagicNeko-ROM/local_manifests/main/ovaltine-16.0.xml

# 3) 同步
repo sync -c -j"$(nproc)" --force-sync --no-clone-bundle --no-tags

# 4) 构建
source build/envsetup.sh
lunch lineage_ovaltine-bp4a-userdebug
mka bacon
```

## 清单内容

| 路径 | 来源 | 说明 |
|---|---|---|
| `device/oneplus/ovaltine` | MagicNeko-ROM | 机型设备树 |
| `device/oneplus/sm8450-common` | MagicNeko-ROM | 共用设备树 |
| `hardware/oplus` | MagicNeko-ROM | 基于 crDroid 官方 16.0 的修改版 |
| `vendor/oneplus/sm8450-common` | MagicNeko-ROM | 供应商库 |
| `vendor/oneplus/ovaltine` | **GitLab 上游** | 含固件；见下方「已知限制」 |
| `kernel/oneplus/sm8450` | MagicNeko-ROM | 内核，顶端含内存扩展改动 |
| `kernel/oneplus/sm8450-devicetrees` | MagicNeko-ROM | 设备树 DTB/DTBO |
| `kernel/oneplus/sm8450-modules` | MagicNeko-ROM | 外编内核模块 |
| `kernel/oneplus/gki-modules` | MagicNeko-ROM | GKI 预编译模块 |

所有 `revision` 一律钉**具体 commit SHA**，不用分支名 —— 同样的清单同步出来的树是同一份。

## 已知限制

- **`vendor/oneplus/ovaltine` 指向上游 GitLab**。那个仓的历史里有两个超重固件
  （`modem.img` 271MB、`dsp.img` 64MB），GitHub 放不下。本项目对固件版本做过 pin
  （24 个分区里 `abl`/`xbl`/`xbl_config` 锁 15.0.0.700，其余随 15.0.0.1601），
  这部分改动**不在公开仓里**，需要时请对照 `device/oneplus/ovaltine` 里
  `BoardConfigVendor.mk` 的 `AB_OTA_PARTITIONS` 与 `radio/` 目录自行核对。
- **跨仓改动以 patch 形式存在设备树里**，同步后需要打一次：

  ```bash
  bash device/oneplus/sm8450-common/upstream-patches/apply.sh
  ```

  幂等脚本，重复执行安全（已应用会跳过）；`--check` 只检查不落盘。
  目前三份：display HAL（UDFPS hbm_enable 链路，缺了指纹不工作）、
  vendor/lineage（depmod 容错 + 头文件冲突）、内核 KSM fragment。

## 内核说明

内核本身是 **5.10.246-gki**。GKI 的设计把内核版本与 Android 框架版本解耦，sm8450
出厂即 5.10，GKI 5.10 LTS 覆盖 Android 12–16，因此 Android 16 配 5.10 是正常搭配。

`android_kernel_oneplus_sm8450` 的 `lineage-23.2` 顶端包含本项目的内存扩展改动：

- `arch/arm64/configs/vendor/oplus/ksm.config` → `CONFIG_KSM=y`
- 运行时由 `device/oneplus/sm8450-common/init/init.kernel.post_boot-cape.sh` 的
  `configure_memory_extension()` 按内存自适应启用 zram（`min(RAM×50%, 6GB)`、lz4）
  与 KSM；配套 SELinux 授权在 `device/oneplus/sm8450-common/sepolicy/`（`sysfs_ksm`）。

> 刷机提示：改了 SELinux policy 后**必须把 `vendor` 与 `odm` 一起刷** ——
> 运行时加载的是 `/odm/etc/selinux/precompiled_sepolicy`，它不校验 vendor 的哈希，
> 只刷 vendor 不会生效。

## 目录

```
ovaltine-16.0.xml   # 放进 .repo/local_manifests/ 的补充清单
README.md           # 本文件
```

## 许可

清单文件本身按原样提供。各项目源码分别遵循其自身许可（AOSP / LineageOS / crDroid / 高通 CAF 等）。
