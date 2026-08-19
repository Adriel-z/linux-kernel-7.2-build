# Linux Kernel 7.2.0 本机编译备份（linux-kernel-7.2-build）

> ## ⚠️⚠️ 重要警告（务必阅读）⚠️⚠️
>
> **本仓库仅仅是一台特定笔记本（本机）的内核编译信息备份，不是可分发/可移植的内核发布包。**
>
> - ❌ **不要**把本仓库的 `.deb` 包、`config`、`buildconfig` 或任何产物直接拿到其他机器上安装使用
> - ❌ **不要**把本仓库视为官方内核或发行版内核的替代品
> - ✅ 本仓库的内容**仅供参考**：用于记录编译过程、配置思路、排错经验，以及作为后续内核二次开发的起点资料
> - 该内核以 `CONFIG_X86_NATIVE_CPU=y`（`-march=native`）编译，**仅适用于编译它的那台机器**（Intel i7-7700HQ / Kaby Lake），拿到别的机器上可能无法启动或性能异常
>
> 编译信息、配置参数、日志均来自一次真实的本机编译实践，不代表通用最佳实践。

---

## 目录

- [仓库内容](#仓库内容)
- [机器硬件环境](#机器硬件环境)
- [内核编译实际情况与参数](#内核编译实际情况与参数)
- [内核编译情况说明](#内核编译情况说明)
- [无线网络模块编译信息](#无线网络模块编译信息)
- [内核文档（htmldocs）](#内核文档htmldocs)
- [buildconfig 说明](#buildconfig-说明)
- [文件说明](#文件说明)
- [注意事项](#注意事项)
- [常见问题 FAQ](#常见问题-faq)
- [供 AI 二次开发使用的会话上下文](#供-ai-二次开发使用的会话上下文)

---

## 仓库内容

| 路径 | 内容 |
|---|---|
| `README.md` | 本文件：编译全过程的说明与经验 |
| `config/config-7.2.0-custom` | 实际使用的内核 .config（编译产物同款） |
| `buildconfig/` | 单独导出的本机专用 buildconfig（config + Module.symvers + 说明） |
| `logs/` | 去敏后的编译日志（kernel / docs / deb 构建） |
| `docs/kernel-reading-guide.md` | **内核阅读与理解教学指南**（面向想学习内核源码的人） |
| `ai-context/session-context.md` | 本次编译会话上下文提炼，**用于后续喂给 AI 做内核二次开发** |
| `Release`（GitHub Releases） | 全部编译产物：4 个 `.deb` 包 + 内核文档包 `linux-7.2-docs-htmldocs.tar.xz` |

> 注：内核源码（linux-7.2）本身不包含在本仓库中，请从 <https://www.kernel.org/> 获取。

---

## 机器硬件环境

| 组件 | 型号 | 内核驱动 |
|---|---|---|
| CPU | Intel Core i7-7700HQ (Kaby Lake, 4C8T, 2.8GHz) | intel_pstate, intel_idle |
| 内存 | 22 GiB DDR4 + 8 GiB swap | — |
| 核显 | Intel HD Graphics 630 (Kaby Lake GT2) | i915 |
| 独显 | NVIDIA GTX 1050 Mobile (GP107M) | nvidia-580（DKMS 外部模块，见注意事项） |
| 无线网卡 | **Intel Dual Band Wireless-AC 3168NGW** | **iwlwifi / iwlmvm** |
| 有线网卡 | Realtek RTL8111/8168/8211/8411 | r8169 |
| NVMe | Kingston A2000 (SM2263EN) 512G | nvme |
| 声卡 | Intel HM175 HD Audio + NVIDIA HDA | snd_hda_intel |
| 蓝牙 | Intel Wireless-AC 3168 Bluetooth | btusb |
| 摄像头 | Chicony USB2.0 | uvcvideo |
| 读卡器 | Realtek RTL8411B | rtsx_pci |
| 系统 | Ubuntu 26.04 LTS (resolute)，内核基线 7.0.0-29-generic | — |

主板为 Intel 100/C230 系列芯片组（HM175），主机名 `a`，桌面 GNOME/Wayland。

---

## 内核编译实际情况与参数

### 基本信息

| 项目 | 值 |
|---|---|
| 内核版本 | **7.2.0**（`linux-7.2.tar.xz`，官方主线稳定版） |
| 本地版本后缀 | `-custom`（`CONFIG_LOCALVERSION="-custom"`） |
| 最终版本串 | `7.2.0-custom` |
| 编译器 | gcc 15.2.0（Ubuntu 26.04 自带） |
| make | GNU Make 4.4.1 |
| 编译并行度 | `make -j8`（8 线程，物理 4 核 + HT） |
| 编译耗时 | 约 2~3 小时（含首次失败重试） |
| 磁盘占用 | vmlinux ~512MB（含 DWARF5 调试信息），modules 6806 个 .ko |
| 配置文件来源 | `/boot/config-7.0.0-29-generic`（Ubuntu 官方 generic 配置）→ `make olddefconfig` 迁移 |
| 签名 | 模块签名启用（构建时自动生成 `certs/signing_key.pem`），`MODULE_SIG_FORCE=n` |
| 信任密钥 | `CONFIG_SYSTEM_TRUSTED_KEYS=""`、`CONFIG_SYSTEM_REVOCATION_KEYS=""`（已清空） |

### 关键实际参数（config 中生效值）

```text
CONFIG_LOCALVERSION="-custom"
CONFIG_X86_NATIVE_CPU=y                # -march=native -mtune=native（本机 Kaby Lake）
CONFIG_HZ_1000=y / CONFIG_HZ=1000      # 桌面响应优化
CONFIG_PREEMPT_LAZY=y / CONFIG_PREEMPT_DYNAMIC=y
CONFIG_CPU_FREQ_DEFAULT_GOV_SCHEDUTIL=y
CONFIG_CC_OPTIMIZE_FOR_PERFORMANCE=y   # -O2
CONFIG_DEBUG_INFO=y / CONFIG_DEBUG_INFO_DWARF5=y
CONFIG_DEBUG_INFO_BTF=y / CONFIG_DEBUG_INFO_BTF_MODULES=y   # 需要 pahole
CONFIG_MODULE_SIG=y / CONFIG_MODULE_SIG_ALL=y / CONFIG_MODULE_SIG_SHA512=y
CONFIG_MODULE_SIG_KEY="certs/signing_key.pem"
CONFIG_SYSTEM_TRUSTED_KEYS=""          # 已清空（原为 debian/canonical-certs.pem，源码树中不存在）
CONFIG_SYSTEM_REVOCATION_KEYS=""
CONFIG_CFG80211=m / CONFIG_MAC80211=m  # 无线协议栈
CONFIG_IWLWIFI=m / CONFIG_IWLMVM=m / CONFIG_IWLDVM=m  # Intel WiFi
CONFIG_BLK_DEV_NVME=y
CONFIG_DRM_I915=y
CONFIG_R8169=m
```

### 实际编译命令序列

```bash
# 0. 依赖（Ubuntu 26.04）
sudo apt install -y flex bison dwarves libelf-dev libdw-dev libssl-dev \
    libncurses-dev gawk dpkg-dev debhelper graphviz rsync cpio \
    python3-sphinx python3-yaml  # 文档需要 sphinx/alabaster/pyyaml

# 1. 解包
tar -xf linux-7.2.tar.xz && cd linux-7.2

# 2. 配置：以本机当前发行版配置为基底
cp /boot/config-$(uname -r) .config
#   —— 编辑 .config ——
#   CONFIG_SYSTEM_TRUSTED_KEYS=""   CONFIG_SYSTEM_REVOCATION_KEYS=""（清空 Debian 证书）
#   CONFIG_LOCALVERSION="-custom"
#   CONFIG_X86_NATIVE_CPU=y（-march=native 本机优化）
make olddefconfig          # 迁移到 7.2 并补齐新符号

# 3. 编译
make -j8

# 4. 文档（全部 htmldocs，单独打包）
make htmldocs
tar -cJf ../linux-7.2-docs-htmldocs.tar.xz -C Documentation/output .

# 5. 安装进系统（先于打包）
sudo make modules_install      # 6806 个模块 → /lib/modules/7.2.0-custom/
sudo make install              # vmlinuz/initrd/System.map → /boot，update-initramfs + update-grub

# 6. 打包成 deb（后打包）
make bindeb-pkg                # 生成 4 个 .deb（image / image-dbg / headers / libc-dev）

# 7. 导出本机 buildconfig
mkdir -p buildconfig
cp .config buildconfig/config-7.2.0-custom
cp Module.symvers buildconfig/
```

---

## 内核编译情况说明

### 为什么用发行版配置打底

Ubuntu 官方 generic 配置已覆盖本机全部硬件（i915、nvidia、iwlwifi、r8169、nvme、snd-hda 等），
且经过发行版大量测试。以它为基底做「本机最优」微调，比从零 `make defconfig` 或全手动裁剪更稳、更省事，
既能保证所有硬件可用，又能享受针对本机的优化。

### 本机最优化的核心：`CONFIG_X86_NATIVE_CPU`

Kernel 7.2 的 x86_64 CPU 选择被重构为单一开关 `CONFIG_X86_NATIVE_CPU`（替代旧版 MCORE2/MSKYLAKE 等型号枚举）。
开启后编译命令中实际出现：

```
-march=skylake -mmmx -mpopcnt -msse -msse2 -msse3 -mssse3 -msse4.1 -msse4.2
-mavx -mavx2 -mfma -mbmi -mbmi2 -maes -mpclmul ... -mtune=skylake
```

（日志中可见，`cc1` 命令行明确带 `-march=skylake -mtune=skylake`。）
代价：该内核二进制只保证在本机（或同代指令集机器）上正常运行。

### 编译中踩过的坑（重要经验）

1. **缺 `dwarf.h`** → `scripts/gendwarfksyms` 编译失败。解决：装 `libdw-dev`。
2. **缺 `gawk`** → `modules.builtin.ranges` 生成失败（`/bin/sh: gawk: not found`）。解决：装 `gawk`。
3. **缺 `debhelper`** → `make bindeb-pkg` 报 `unmet build dependencies: debhelper-compat (= 12)`。解决：装 `debhelper`。
4. **Debian 证书不存在** → 沿用发行版配置时 `CONFIG_SYSTEM_TRUSTED_KEYS="debian/canonical-certs.pem"`，
   源码树里没有该文件，必须清空，否则 `make` 直接报错。
5. **日志很大**：`make -j8` 全量输出可达 1MB+，建议 `make -j8 > build.log 2>&1` 重定向；
   用 `tail -100` 管道收尾的方式适合后台任务，但看不到实时进度，推荐 `tee`。
6. **BTF 编码是最后瓶颈**：`pahole -J` 逐个给 6806 个模块编码 BTF，接近尾声时 CPU 全满但无 cc1，
   属正常现象，不要误判为卡死。

### 安装与打包顺序（按要求：先安装、后打包）

1. `make modules_install`：模块装入 `/lib/modules/7.2.0-custom/`，depmod 完成；
2. `make install`：通过 `/sbin/installkernel` 调用 Ubuntu 的 `update-initramfs` + `update-grub`，
   生成 `/boot/vmlinuz-7.2.0-custom`、`initrd.img-7.2.0-custom`、`System.map-7.2.0-custom`；
   GRUB 默认启动项已指向 `7.2.0-custom`（`GRUB_DEFAULT=0`，新内核排在最前）。
3. `make bindeb-pkg`：产出 4 个 deb（见 Release），`dpkg --dry-run` 验证可安装。

---

## 无线网络模块编译信息

本机**高度依赖无线网络**（Intel Wireless-AC 3168NGW），以下为专项处理记录：

### 模块与固件

| 项 | 值 |
|---|---|
| 无线网卡 | Intel Dual Band Wireless-AC 3168NGW (PCI 04:00.0, device 8086:24fb) |
| 驱动模块 | `iwlwifi.ko`（=`m`）+ `iwlmvm.ko`（=`m`）+ `iwldvm.ko`（=`m`） |
| 协议栈 | `cfg80211.ko`（=`m`）、`mac80211.ko`（=`m`） |
| 固件 | `/lib/firmware/intel/iwlwifi/iwlwifi-3168-29.ucode.zst`（系统 linux-firmware 已有，**未捆绑进内核**） |
| 监管数据库 | `/lib/firmware/regulatory.db`（`wireless-regdb` 包提供，`CONFIG_CFG80211_REQUIRE_SIGNED_REGDB=y`） |

### 编译配置（config 中）

```text
CONFIG_NETDEVICES=y
CONFIG_WLAN=y
CONFIG_WLAN_VENDOR_INTEL=y
CONFIG_IWLWIFI=m
CONFIG_IWLWIFI_LEDS=y
CONFIG_IWLWIFI_OPMODE_MODULAR=y
CONFIG_IWLMVM=m
CONFIG_IWLDVM=m
CONFIG_CFG80211=m
CONFIG_CFG80211_WEXT=y
CONFIG_CFG80211_CRDA_SUPPORT=y
CONFIG_MAC80211=m
```

### 固件策略（按要求：只编译/使用系统上已有的）

- 未启用 `CONFIG_EXTRA_FIRMWARE`（不把固件编进内核二进制），固件全部来自发行版 `linux-firmware` 包
- 编译后验证：`modinfo iwlwifi.ko` 显示 `firmware: iwlwifi-3168-29.ucode` 需求，
  而 `/lib/firmware/intel/iwlwifi/iwlwifi-3168-29.ucode.zst` 确实存在 → 匹配 ✅
- `modules.dep` 依赖链正确：`iwlmvm.ko → iwlwifi.ko → cfg80211.ko`、`mac80211.ko`、`libarc4.ko`

### 安装后专项验证

```bash
ls /lib/modules/7.2.0-custom/kernel/drivers/net/wireless/intel/iwlwifi/   # dvm/ iwlwifi.ko mld/ mvm/
/sbin/modinfo /lib/modules/7.2.0-custom/kernel/drivers/net/wireless/intel/iwlwifi/iwlwifi.ko | grep -E "firmware|depends|vermagic"
# firmware: iwlwifi-3168-29.ucode  ← 存在 ✅
# depends:  cfg80211              ← 已装 ✅
# vermagic: 7.2.0-custom SMP preempt mod_unload modversions
grep iwlwifi /lib/modules/7.2.0-custom/modules.dep                       # 依赖解析 ✅
```

> 提示：initramfs 中未包含 iwlwifi 固件属正常 —— 根文件系统在 NVMe 上，启动阶段不需要网络，
> WiFi 由 NetworkManager 在系统启动后加载模块并从 `/lib/firmware` 取固件。

---

## 内核文档（htmldocs 及全部格式）

**全部 10 种文档格式均已构建**（`Documentation/Makefile` 定义的目标）：

| 格式 | 目标 | 产物 | 规模 |
|---|---|---|---|
| HTML | `make htmldocs` | 4024 页 | 580MB（压缩 58MB） |
| PDF | `make pdfdocs`（xelatex） | **64 个 PDF**（含 4 个大文档） | 94MB |
| LaTeX | `make latexdocs` | 683 个文件 | 172MB |
| EPUB | `make epubdocs` | 4221 个文件 | 130MB |
| XML | `make xmldocs` | 4021 个文件 | 160MB |
| Man pages | `make mandocs` | 80142 个文件 | 357MB |
| Texinfo | `make texinfodocs` | 134 个文件 | 129MB |
| Info | `make infodocs` | TheLinuxKernel.info | — |

- **全部格式合集**：`linux-7.2-docs-all-formats.tar.xz`（226MB）→ **已发布到本仓库 Release**
- **HTML 单独包**：`linux-7.2-docs-htmldocs.tar.xz`（58MB）→ 同上
- 阅读入口：HTML 解包后打开 `index.html`；PDF 可直接阅读（core-api 1351 页 / admin-guide 1587 页 / arch 749 页 / translations 658 页）

### 构建经验（PDF 部分）

1. `pdfdocs` 需要 **TeX Live**（`texlive-xetex` + `texlive-latex-extra`），系统默认未装
2. 4 个大文档（core-api/admin-guide/arch/translations）默认 TeX 内存（`main_memory=5000000`）**不够**，
   报 `TeX capacity exceeded` / `Dimension too large`。解法：在 `/etc/texmf/texmf.d/` 增加配置
   `main_memory = 20000000` 等并运行 `update-texmf`，**同时需 `fmtutil-sys --byfmt xelatex` 重建 format**
   （否则 xelatex 仍用旧内存限制）
3. `infodocs` 需要 **texinfo** 包（提供 `makeinfo`）
4. sphinx-build-wrapper 并行构建大 PDF 时会因资源竞争失败，可手动 `xelatex xxx.tex` 串行构建后归位
5. 大量 `Dimension too large`（fancybox 框架）为无害排版警告，不影响 PDF 产出

- **面向初学者的内核阅读/理解教学指南**：见 [docs/kernel-reading-guide.md](docs/kernel-reading-guide.md)

---

## buildconfig 说明

单独导出的 `buildconfig/` 目录（同时已另存于本机 `<HOME>/work/kernel-update/buildconfig/` 及 `/boot/config-7.2.0-custom`），
供**后续在本机重新编译内核**使用：

- `config-7.2.0-custom`：完整 `.config`（与本次编译完全一致，SHA256 见 `README-buildconfig.md`）
- `Module.symvers`：模块符号表，编译外部模块（如 DKMS）时有用
- `README-buildconfig.md`：详细说明（硬件清单、关键配置项、编译依赖、使用方法）

⚠️ 该 buildconfig **仅适用于本机**（含 `X86_NATIVE_CPU` 机器级优化），详见 `buildconfig/README-buildconfig.md`。

---

## 文件说明

| 文件 | 说明 |
|---|---|
| `config/config-7.2.0-custom` | 本次编译实际使用的 .config（可直接 `cp` 使用） |
| `buildconfig/` | 本机专用 buildconfig 三件套 |
| `logs/build-kernel.log` | 内核编译日志（**已去除个人路径/邮箱等敏感信息**，路径替换为 `<KERNEL_SRC>`、`<BUILD_DIR>`、`<HOME>`） |
| `logs/build-kernel2.log` | 第二次（成功）内核编译收尾日志 |
| `logs/build-docs.log` | htmldocs 构建日志 |
| `logs/build-deb.log` / `build-deb2.log` | deb 打包日志（第一次缺 debhelper 失败 + 第二次成功） |
| `docs/kernel-reading-guide.md` | 内核阅读教学指南 |
| `ai-context/session-context.md` | 会话上下文提炼（供 AI 二次开发） |
| Release: `linux-image-7.2.0-custom_7.2.0-4_amd64.deb` | 内核镜像包（125MB） |
| Release: `linux-image-7.2.0-custom-dbg_7.2.0-4_amd64.deb` | 调试符号包（1.5GB，含 vmlinux 调试信息） |
| Release: `linux-headers-7.2.0-custom_7.2.0-4_amd64.deb` | 头文件包（11MB，编译外部模块必需） |
| Release: `linux-libc-dev_7.2.0-4_amd64.deb` | 用户态 libc 开发头文件（1.5MB） |
| Release: `linux-7.2-docs-htmldocs.tar.xz` | 内核全量 HTML 文档（58MB） |

---

## 注意事项

1. **本机专用内核**：`-march=native` 编译，勿移植到其他机器。
2. **NVIDIA 驱动需重编**：本机装的是 `nvidia-driver-580`（模块随 `linux-modules-nvidia-580-<旧内核>` 包发布，非 DKMS）。
   换到 7.2.0-custom 内核后，独显模块不会自动出现。若需要独显，用 `linux-headers-7.2.0-custom` 头文件包 +
   `/usr/src/nvidia-580.173.02` 源码手工编译（或装 dkms 后重装驱动）。核显 i915 不受影响，日常显示正常。
3. **重启才生效**：内核安装后需重启进入 7.2.0-custom；旧内核 7.0.0-29-generic 保留在 GRUB 中可回退。
4. **模块签名**：内核使用构建时自动生成的签名密钥；发行版签名的 DKMS 外部模块加载时会有
   taint 警告但可加载（`MODULE_SIG_FORCE=n`）。
5. **调试符号包巨大**（1.5GB）：仅调试需要时安装 `linux-image-...-dbg`，日常可省。
6. **文档构建依赖**：`make htmldocs` 需要 Sphinx ≥ 3.4.3 + alabaster + pyyaml + graphviz（图表），
   缺失时 Sphinx 构建会报错或跳图。
7. **本仓库不含内核源码**：源码请从 kernel.org 获取；本仓库只有配置、日志、文档与产物备份。

---

## 常见问题 FAQ

**Q1：为什么用 Ubuntu 的 config 而不是 `make defconfig`？**
A：发行版 config 已覆盖全部硬件驱动且经大量测试；在其上做本机优化（native CPU）风险最小、收益最大。

**Q2：`make` 报 `dwarf.h: No such file or directory`？**
A：装 `libdw-dev`（提供 gendwarfksyms 需要的 dwarf.h）。

**Q3：报 `gawk: not found`？**
A：装 `gawk`（`modules.builtin.ranges` 生成依赖）。

**Q4：`bindeb-pkg` 报 debhelper 依赖不满足？**
A：装 `debhelper`（`debhelper-compat (= 12)`）。

**Q5：`CONFIG_SYSTEM_TRUSTED_KEYS="debian/canonical-certs.pem"` 报错？**
A：发行版配置引用了 Ubuntu 打包用的证书文件，上游源码树没有它，必须清空为 `""`。

**Q6：编译到最后 CPU 很忙但没有 cc1 进程？**
A：正常 —— 正在用 `pahole` 给数千个模块生成 BTF 信息。

**Q7：WiFi 在 initramfs 里找不到固件？**
A：正常。启动阶段（NVMe 根文件系统）不需要网络；进入系统后由 NetworkManager 加载模块取固件。

**Q8：新内核起不来/有问题怎么办？**
A：GRUB 高级选项里选旧内核 7.0.0-29-generic 回退；日志里保留了两轮完整构建记录便于排查。

**Q9：这些 deb 能装到别的电脑上吗？**
A：**不能**（`-march=native`）。只能作为本机备份。参见顶部警告。

**Q10：如何在本机复现这次构建？**
A：参考 [buildconfig/README-buildconfig.md](buildconfig/README-buildconfig.md) 与本文「实际编译命令序列」。

---

## 供 AI 二次开发使用的会话上下文

本次内核编译的完整会话上下文（环境事实、工具链、决策依据、踩坑记录、后续开发建议）已提炼为：

👉 **[ai-context/session-context.md](ai-context/session-context.md)**

该文件专为「把上下文喂给 AI，让它接手本机内核的二次开发」设计，包含：
本机软硬件事实、编译命令与配置决策、依赖清单、已知问题与对策、
以及后续（如开启 RT、性能调优、nvidia 模块重编、DKMS 化等）开发方向建议。

---

*本仓库由 DSH agent 于 2026-08-19 在本机生成并推送。*
