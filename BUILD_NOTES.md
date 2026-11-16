# OnePlus Ace 3 自编译内核说明（PJE110）

本仓库 = `siri666666/OnePlus-OKI-Builder` + 针对 **OnePlus Ace 3 (PJE110)** 的少量修复，用于自编译可开机的 **SukiSU + SUSFS** 内核。

## 相对原版 workflow 已做的改动

1. **SUSFS 源**：用上游 `gitlab.com/simonpunk/susfs4ksu` 的 `gki-android13-5.15` 分支（当前 = **SUSFS v2.4.0**，含 SukiSU 41148+ 需要的 `susfs_set_priority_uid_filter` 等函数）。早期的 1.5.x 分支与新版 SukiSU 不兼容会编译报错；曾用过的 `ShirkNeko/susfs4ksu` GitHub 镜像已下线（账号改名为 `SukiSU-Nova`），所以统一用 GitLab 上游。
2. **补 Kconfig**：新增 `KSU_SUSFS_PRIORITY_UID_FILTER` 的 Kconfig 定义，并在 defconfig 打开（否则链接报 `undefined symbol`）。
3. **放行 -Werror**：Fallback 构建加 `-Wno-error=implicit-function-declaration -Wno-error=unused-function`（SukiSU 源码不 include susfs.h）。
4. **BBR**：打开 `CONFIG_TCP_CONG_BBR` + `CONFIG_NET_SCH_FQ`（默认拥塞算法仍是 cubic，可自行切换）。
5. **Image 兜底**：Make AnyKernel3 增加全工作区查找 Image，避免官方脚本输出路径不被识别。
6. **SukiSU rules.c 修复**：新版 SukiSU（41176+）的 `kernel/selinux/rules.c` 有几个问题——(a) `pol->latest_granting` 笔误成 `policydb->latest_granting`（`policydb` 是 struct 类型名不是变量），编译 SELinux 时报错；(b) 新代码每次注入规则都把策略 `latest_granting` 加 1 并递增通知，导致 `access.avd.seqno` 涨到 6，Duck Detector / Hunter / 春秋 读到“非初始 epoch”判可疑。workflow 已加三条 sed：`policydb->latest_granting` → `pol->latest_granting`（修编译）；`reset_avc_cache(pol->latest_granting)` → `reset_avc_cache(0)`（通知序号归零）；`pol->latest_granting = old_pol->latest_granting + 1;` → `pol->latest_granting = old_pol->latest_granting;`（策略代际保持不变，`access.avd.seqno` 回到 stock 值）。
7. **Swap 更稳**：3G swap 文件优先建在工作区大挂载点（根分区满时也能建），两处都失败就只警告并继续，不再直接中止构建。
8. **SukiSU 权威源 = ShirkNeko 自建 Forgejo**：`git.zako.shirkneko.top:20004/shirkneko/SukiSU` 是作者的最新源；GitHub 的 `SukiSU-Nova/SukiSU-Nova` 是滞后的镜像（例如 GitHub 上管理器还停在 versionCode 30000，Forgejo 已到 35000）。workflow 已改为优先从 Forgejo 拉内核源码（`kernel/setup.sh`）和版本号，GitHub 作为 fallback。
9. **管理器 APK 跟随最新版**：每次构建优先从 Forgejo 的 latest release 下载对应 `.apk`（当前 v3.0.0 / versionCode **35000**），取不到再回退 GitHub release，最后回退 CI 的 Manager artifact。所以 Nova 发新版后重新构建就会自动带上新管理器。

## 验证结果

- **2026-11-16**：按上面配置编出的 AK3（SukiSU Nova + SUSFS v2.4.0 + BBG + BBR，含 rules.c 三项修复）刷入后，Duck Detector / Hunter / 春秋 的 SELinux `access.avd.seqno` 异常已消失，**检测通过**。

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
- **SukiSU 跟随 ShirkNeko Forgejo 最新版**（KSU_META 默认），版本会自动升级；若新版又引入新的编译错误，可仿照「已做的改动 6」在 workflow 里补一个 sed 修复，或用 KSU_META 第 4 段提交哈希把 SukiSU 锁定到某个能编过的版本。