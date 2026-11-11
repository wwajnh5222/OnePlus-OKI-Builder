# OnePlus Ace 3 自编译内核说明（PJE110）

本仓库 = `siri666666/OnePlus-OKI-Builder` + 针对 **OnePlus Ace 3 (PJE110)** 的少量修复，用于自编译可开机的 **SukiSU + SUSFS** 内核。

## 相对原版 workflow 已做的改动

1. **SUSFS 源换新**：`gitlab.com/simonpunk/susfs4ksu` → `github.com/ShirkNeko/susfs4ksu`（分支 `gki-android13-5.15`，当前 = **SUSFS v2.4.0**）。原版 SUSFS 1.5.x 与新版 SukiSU（41148+）不兼容，会报 `susfs_set_priority_uid_filter` 隐式声明错误。
2. **补 Kconfig**：新增 `KSU_SUSFS_PRIORITY_UID_FILTER` 的 Kconfig 定义，并在 defconfig 打开（否则链接报 `undefined symbol`）。
3. **放行 -Werror**：Fallback 构建加 `-Wno-error=implicit-function-declaration -Wno-error=unused-function`（SukiSU 源码不 include susfs.h）。
4. **BBR**：打开 `CONFIG_TCP_CONG_BBR` + `CONFIG_NET_SCH_FQ`（默认拥塞算法仍是 cubic，可自行切换）。
5. **Image 兜底**：Make AnyKernel3 增加全工作区查找 Image，避免官方脚本输出路径不被识别。
6. **SukiSU 41176 笔误修复**：新版 SukiSU（41176）`kernel/selinux/rules.c` 把 `pol->latest_granting` 误写成 `policydb->latest_granting`（`policydb` 是 struct 类型名不是变量），导致编译 SELinux 时报错；workflow 已加一步自动 `sed` 修正。

## 如何编译新版本

1. 打开 **Actions → 构建一加内核 → Run workflow**。
2. 填写参数：

| 参数 | 值 |
|---|---|
| FILE | `一加 Ace 3（oneplus_ace3_b）` |
| DEVICE_KERNEL | `5.15.180`（看手机 设置→关于→内核版本） |
| DEVICE_FIRMWARE | `16.0.3.500`（看手机系统版本） |
| KSU_META | `main/builtin/OnePlus-OKI/`（默认，自动用最新 SukiSU） |
| KPM | `N/A` |
| SUSFS | 勾选 |
| SUSFS_META | 留空（用最新，当前 v2.4.0） |
| SUSFS_DEV | 不勾 |
| SECURITY_FIXES | 勾选 |
| LSM_BBG | 勾选 |
| UNICODE_BYPASS | 勾选 |
| SUFFIX | `RuI324-KuXunShaoLuo-OneplusAce3` |
| BUILD_TIME | `Fri Dec 12 11:37:03 UTC 2025` |
| FAST_BUILD | **不勾**（走 Fallback 官方脚本） |
| BUILD_NOCACHE | 不勾 |
| SPACE_NOCLEAN | 不勾 |

3. 跑完（约 25~50 分钟）下载 artifact（AnyKernel3），用 ReKernelFlasher 刷入。

## 升级系统后重新编译

- 改 `DEVICE_FIRMWARE` 为新版本号（如 `16.0.4.xxx`）。
- 若内核版本变了，改 `DEVICE_KERNEL`。
- `BUILD_TIME` 改成新固件的构建时间：刷机前在旧系统 `adb shell uname -v` 看，抄 `PREEMPT` 后面的时间（格式 `Fri Dec 12 11:37:03 UTC 2025`）。
- `SUFFIX` 可保持不变（你自己的自定义标签）。

## 刷入与救砖

- 刷前务必用 ReKernelFlasher **备份 boot_a**。
- 卡开机：fastboot 刷回原厂 boot：
  ```
  fastboot flash boot_a 原厂boot.img
  fastboot reboot
  ```

## 注意

- **不要开 Droidspaces**（`SYSVIPC`/`IPC_NS` 会破坏 kABI，实测开过导致不开机）。
- BBR 已编入但默认仍是 cubic；想切 bbr：
  ```
  su -c 'echo bbr > /proc/sys/net/ipv4/tcp_congestion_control'
  ```
- **SukiSU 跟随上游最新版**（KSU_META 默认），版本会自动升级（41168 → 41176 → …）；若新版 SukiSU 又引入新的编译错误，可仿照「已做的改动 6」在 workflow 里补一个 sed 修复，或用 KSU_META 第 4 段提交哈希把 SukiSU 锁定到某个能编过的版本。