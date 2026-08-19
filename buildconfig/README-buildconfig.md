# 本机构建配置（BuildConfig）说明

> ⚠️ **仅适用于本机（This config is machine-specific ONLY）**

本配置文件 `config-7.2.0-custom` 是为 **本机** 定制生成的 Linux 内核构建配置，
用于后续重新编译内核时保持一致的配置。

## 适用机器硬件

| 组件 | 型号 |
|---|---|
| CPU | Intel Core i7-7700HQ (Kaby Lake, 4C8T) |
| 内存 | 22 GiB |
| 显卡 | Intel HD Graphics 630 + NVIDIA GTX 1050 Mobile (Optimus, on-demand) |
| 无线网卡 | Intel Dual Band Wireless-AC 3168NGW (iwlwifi) |
| 有线网卡 | Realtek RTL8111/8168/8211/8411 (r8169) |
| SSD | Kingston A2000 NVMe (SM2263EN) |
| 声卡 | Intel HM175 HD Audio + NVIDIA HDA |
| 蓝牙 | Intel Wireless-AC 3168 Bluetooth |

## 关键配置项

- `CONFIG_LOCALVERSION="-custom"` → 内核版本号为 `7.2.0-custom`
- `CONFIG_X86_NATIVE_CPU=y` → 使用 `-march=native -mtune=native` 按本机 CPU 优化
  （**该内核仅能在本机或同型号 CPU 上运行，不能移植到其他机器**）
- `CONFIG_CFG80211=m` / `CONFIG_MAC80211=m` / `CONFIG_IWLWIFI=m` / `CONFIG_IWLMVM=m`
  → 无线网卡模块（本机高度依赖无线网络）
- `CONFIG_SYSTEM_TRUSTED_KEYS=""` / `CONFIG_SYSTEM_REVOCATION_KEYS=""`
  → 清除了 Ubuntu 的 Debian 签名证书（源码树中没有该证书，编译会失败）
- `CONFIG_DEBUG_INFO_BTF=y` / `CONFIG_DEBUG_INFO_DWARF5=y`（需安装 pahole/dwarves）
- 其余配置基于 Ubuntu 26.04 官方 generic 内核配置（`/boot/config-7.0.0-29-generic`）
  并运行 `make olddefconfig` 迁移到 7.2

## 使用方法

```bash
# 在解压后的 linux-7.2 源码目录中：
cp /path/to/buildconfig/config-7.2.0-custom .config
make olddefconfig        # 若迁移到更高版本内核，建议执行
make -j$(nproc)          # 编译
```

## 编译依赖（本机已装）

- gcc 15.2、make 4.4.1、flex、bison、bc、gawk
- libssl-dev、libelf-dev、libdw-dev（dwarf.h）、libncurses-dev
- dwarves/pahole（BTF 需要）、dpkg-dev、debhelper（deb 打包需要）
- python3-sphinx + alabaster + pyyaml（内核文档 htmldocs 需要）
- graphviz（文档图表）

## 固件说明

- 内核不捆绑编译固件；固件由系统包 `linux-firmware` 提供于 `/lib/firmware`
- 本机 WiFi 固件：`/lib/firmware/intel/iwlwifi/iwlwifi-3168-29.ucode.zst`
- 仅编译系统上已存在的固件所需模块，未启用 `CONFIG_EXTRA_FIRMWARE`

## 生成时间

2026-08-19，由 DSH agent 在本机生成
