# 会话上下文提炼 —— 供 AI 接手本机内核二次开发

> 本文件由 2026-08-19 一次真实的 Linux 内核 7.2.0 编译会话提炼而成。
> 目的是：把本次会话中「机器事实、工具链、决策、坑、后续建议」完整地交给另一个 AI，
> 让它无需重新摸索即可在本机继续内核的二次开发（调优、裁剪、RT、新驱动等）。
> 配合 `README.md`、`docs/kernel-reading-guide.md`、`logs/` 与 Release 中的产物使用。

---

## 1. 机器事实（不可变环境）

| 项 | 值 |
|---|---|
| 操作系统 | Ubuntu 26.04 LTS (resolute)，主机名 `a`，桌面 GNOME/Wayland |
| CPU | Intel Core i7-7700HQ (Kaby Lake, 4C8T, 2.8~3.8GHz)，微架构 skylake |
| 内存 | 22 GiB + 8 GiB swap，NUMA 单节点 |
| 存储 | Kingston A2000 NVMe 512G（根分区 nvme0n1p2，`/` 457G，EFI 分区 nvme0n1p1） |
| 显卡 | Intel HD 630（i915，主显示）+ NVIDIA GTX 1050（on-demand prime，nvidia-driver-580） |
| 无线网卡 | Intel Wireless-AC 3168NGW（PCI 04:00.0，8086:24fb），iwlwifi/iwlmvm，固件 iwlwifi-3168-29.ucode |
| 有线网卡 | Realtek RTL8111/8168/8211/8411（03:00.1），r8169 |
| 蓝牙 | Intel Wireless-AC 3168 Bluetooth（8087:0aa7），btusb |
| 其他 | 摄像头 Chicony、读卡器 RTL8411B、Intel HM175 声卡 |
| 原内核 | 7.0.0-29-generic（Ubuntu 官方），/boot/config-7.0.0-29-generic 为配置基底 |
| 用户 | 用户名 `a`（uid 1000），sudo 密码不在本仓库记录（安全起见勿写明文） |
| 工作目录 | `<HOME>/work/`（**切勿删除**，DSH 会话依赖；内核源码在 `<HOME>/work/kernel-update/`，`<HOME>` 即本机用户主目录） |

## 2. 工具链（编译时已确认可用）

- gcc 15.2.0、GNU Make 4.4.1、flex、bison、bc、gawk
- libssl-dev、libelf-dev、libdw-dev（dwarf.h）、libncurses-dev
- dwarves/pahole 1.31（BTF 必需）
- dpkg-dev、debhelper（deb 打包必需）
- python3-sphinx 8.2.3 + alabaster + pyyaml + graphviz（htmldocs 必需）

## 3. 编译决策与依据（为什么这么做）

1. **配置基底 = 发行版 config**：`cp /boot/config-7.0.0-29-generic .config` + `make olddefconfig`。
   理由：硬件全覆盖、经过发行版测试、省去裁剪风险。
2. **本机优化 = `CONFIG_X86_NATIVE_CPU=y`**：7.2 的 x86_64 CPU 选择只剩这一个开关，
   等价 `-march=native -mtune=native`（实测 cc1 带 `-march=skylake -mtune=skylake`）。
   代价：内核不可移植。
3. **清空信任密钥**：`CONFIG_SYSTEM_TRUSTED_KEYS=""`、`CONFIG_SYSTEM_REVOCATION_KEYS=""`。
   发行版配置指向 `debian/canonical-certs.pem`，上游源码树没有该文件，不清空编译即失败。
4. **LOCALVERSION="-custom"**：最终版本串 `7.2.0-custom`，与发行版内核区分。
5. **模块签名保留**（`CONFIG_MODULE_SIG=y`）：构建时自动生成 `certs/signing_key.pem`；
   `MODULE_SIG_FORCE=n` 保证未签名外部模块（如重编的 nvidia）仍可加载（带 taint）。
6. **固件策略**：不启用 `CONFIG_EXTRA_FIRMWARE`，固件全部来自系统 linux-firmware 包（/lib/firmware）。
7. **文档**：`make htmldocs` 全量生成 4024 页 HTML（Sphinx ≥3.4.3），独立打包发布。

## 4. 编译/安装/打包实际命令（可直接复现）

```bash
# 配置
tar -xf linux-7.2.tar.xz && cd linux-7.2
cp /boot/config-$(uname -r) .config
# 编辑 .config：清空两个 KEYS、LOCALVERSION="-custom"、X86_NATIVE_CPU=y
make olddefconfig
# 编译
make -j8            # 全量输出 >1MB，建议 tee 到日志
# 文档
make htmldocs && tar -cJf ../linux-7.2-docs-htmldocs.tar.xz -C Documentation/output .
# 安装（先安装后打包，符合要求）
sudo make modules_install   # → /lib/modules/7.2.0-custom/（6806 模块）
sudo make install           # → /boot/*，自动 update-initramfs + update-grub
# 打包
make bindeb-pkg             # → 4 个 deb
# buildconfig 导出
mkdir -p buildconfig && cp .config buildconfig/config-7.2.0-custom && cp Module.symvers buildconfig/
```

## 5. 踩坑记录（后续开发避免重蹈）

| 症状 | 根因 | 对策 |
|---|---|---|
| `dwarf.h: No such file or directory`（gendwarfksyms） | 缺 libdw-dev | `apt install libdw-dev` |
| `/bin/sh: gawk: not found`（modules.builtin.ranges） | 缺 gawk | `apt install gawk` |
| `unmet build dependencies: debhelper-compat (= 12)` | 缺 debhelper | `apt install debhelper` |
| `debian/canonical-certs.pem` 不存在 | 发行版证书不在上游源码 | 清空 TRUSTED/REVOCATION KEYS |
| 编译接近尾声 CPU 满但无 cc1 | pahole 在给模块编码 BTF | 正常，等待即可 |
| 模块加载失败 `vermagic` 不匹配 | 换内核后驱动未重编 | 用 headers deb 重编 DKMS/外部模块 |

## 6. 已知状态与遗留事项

- ✅ 新内核已安装，GRUB 默认启动 7.2.0-custom；旧内核保留可回退
- ⚠️ **NVIDIA 驱动未适配新内核**：本机 nvidia-580 模块随 `linux-modules-nvidia-580-7.0.0-29-generic`
  包提供（非 DKMS），7.2.0-custom 下无独显模块。核显 i915 正常。
  处理方案：`sudo apt install dkms` 后重装 nvidia-driver-580，或手工用
  `make -C /lib/modules/7.2.0-custom/build M=/usr/src/nvidia-580.173.02 modules` 编译。
- ⚠️ initramfs 未含 WiFi 固件（正常）：根 fs 在 NVMe，启动阶段无需网络。
- ℹ️ 构建日志（去敏后）在 `logs/`；原始日志含个人路径，已按 `<KERNEL_SRC>/<BUILD_DIR>/<HOME>` 替换。

## 7. 后续二次开发方向建议（供 AI 参考）

1. **实时性**：评估 `CONFIG_PREEMPT_RT`（7.2 支持），注意与 `X86_NATIVE_CPU` 共存及模块兼容。
2. **裁剪**：若追求更小内核，可关闭不用的驱动（蓝牙、读卡器、虚拟机、罕见 fs）——
   但必须先确认本机无依赖（用 `lsmod` + `/sys/bus` 反向核对）。
3. **无线增强**：iwlwifi 可调 `iwlwifi.power_save=0`（功耗 vs 延迟）、`11n_disable`、`bt_coex_active`
   等模块参数；或研究 mac80211 层做自定义调度/省电策略。
4. **性能调优**：`CONFIG_HZ=1000` 已开；可进一步测 `SCHED_MUQSS`/`BORE` 等外部调度器补丁（注意社区成熟度）。
5. **新功能**：本机 CPU 支持 SGX/VMX，可研究 KVM 直通；或加 eBPF 观测工具链（BTF 已就绪）。
6. **驱动开发教学**：以 iwlwifi 为范本（第 5 节阅读指南有路线），练手改参数/加 debugfs 节点。

## 8. 安全提醒

- 本仓库为公开仓库，**不含**：sudo 密码、GitHub token、任何机器序列号/MAC/IP。
- 日志已去敏；后续如追加日志，请先替换 `/home/<user>` 等个人路径再提交。
- 内核产物仅本机可用，勿外传安装。

---

*提炼自 2026-08-19 DSH 会话；由 DSH agent 编写。*
