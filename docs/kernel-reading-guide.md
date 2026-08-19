# Linux 内核阅读与理解教学指南

> 面向对象：想真正读懂内核源码、理解其运行机制的学习者（包括准备交给 AI 辅助阅读的场景）。
> 本文不贴大量代码，而是讲「怎么读、按什么顺序读、每个概念在代码里长什么样、文本/符号代表什么意思」。
> 配合本仓库 Release 中的 `linux-7.2-docs-htmldocs.tar.xz`（内核官方文档）效果最佳。

---

## 目录

1. [读内核前必备的概念地图](#1-读内核前必备的概念地图)
2. [源码目录结构速览（每目录是干什么的）](#2-源码目录结构速览)
3. [推荐阅读顺序（由浅入深 6 步）](#3-推荐阅读顺序)
4. [如何理解内核的运行机制](#4-如何理解内核的运行机制)
5. [内核文本/符号的「语义字典」](#5-内核文本符号的语义字典)
6. [内核构建系统的文本含义（Kconfig / Kbuild / Makefile）](#6-内核构建系统的文本含义)
7. [实践：从一次开机到一次系统调用的完整链路](#7-实践从开机到系统调用的完整链路)
8. [给 AI 辅助阅读的提示词模板](#8-给-ai-辅助阅读的提示词模板)

---

## 1. 读内核前必备的概念地图

内核不是普通程序，读之前先建立这几个核心心智模型：

| 概念 | 一句话理解 | 代码里去哪找 |
|---|---|---|
| 内核态 / 用户态 | 内核运行在更高特权级（ring 0），用户程序在 ring 3；两者通过**系统调用**交界 | `arch/x86/entry/`（syscall 入口） |
| 进程 / 线程 | 内核调度和执行的最小单位 = task；线程是共享地址空间的 task | `kernel/sched/`, `include/linux/sched.h` |
| 虚拟内存 | 每个进程有独立虚拟地址空间，由页表（page table）映射到物理内存 | `mm/`, `arch/x86/mm/` |
| 中断 / 异常 | 硬件事件（中断）与 CPU 异常打断当前执行流 | `kernel/irq/`, `arch/x86/kernel/irq.c` |
| 系统调用 | 用户程序请求内核服务的唯一正规入口 | `kernel/sys.c`, `arch/x86/entry/syscalls/` |
| 内核对象 / 引用计数 | 内核里几乎所有实体（文件、设备、模块）都是带引用计数的对象 | `include/linux/kref.h`, `lib/kobject.c` |
| 锁 / 并发 | 内核随时可能被抢占，必须用锁保护共享数据 | `include/linux/spinlock.h`, `kernel/locking/` |
| 模块 | 可动态加载的内核代码单元（`.ko` 文件） | `kernel/module/`, 各 driver 目录 |
| 设备模型 | 设备、驱动、总线三者通过匹配机制绑定 | `drivers/base/`, `include/linux/device.h` |

> 关键认知：**内核是事件驱动的并发系统** —— 一切工作要么发生在「某个进程的上下文里」，
> 要么发生在「中断上下文里」。阅读任何函数前先问：它跑在什么上下文？

---

## 2. 源码目录结构速览

```
linux-7.2/
├── arch/           # 架构相关代码（x86/arm64/...），本机看 arch/x86
│   └── x86/        #   启动、页表、syscall 入口、CPU 特性
├── kernel/         # 核心：进程调度、锁、信号、时间、模块、printk...
├── mm/             # 内存管理：页分配、slab、vmscan、mmap
├── fs/             # 文件系统：VFS 抽象层 + 各具体 fs（ext4/btrfs/...）
├── net/            # 网络协议栈：socket 层、TCP/IP、netfilter、无线（mac80211 在此）
├── drivers/        # 设备驱动（最大目录）：gpu/drm、net/wireless、nvme、usb...
├── include/        # 头文件：linux/（内核 API）、uapi/（用户态可见 ABI）、asm/
├── security/       # LSM（Linux Security Module）、SELinux、AppArmor 等
├── crypto/         # 加密算法（对称/非对称/哈希）
├── block/          # 块设备层：I/O 调度、bio、分区
├── sound/          # ALSA 音频栈
├── virt/           # KVM 虚拟化
├── init/           # 内核启动初始化（start_kernel 相关）
├── ipc/            # 进程间通信（消息队列/信号量/共享内存）
├── lib/            # 内核通用库函数（kobject、klist、vsprintf...）
├── scripts/        # 构建脚本（Kconfig/Kbuild 工具链）
├── tools/          # 用户态辅助工具（perf、objtool、bpf...）
├── Documentation/  # 官方文档（RST 格式，htmldocs 来源）
└── certs/          # 模块签名/信任密钥
```

本机相关重点目录：`drivers/net/wireless/intel/iwlwifi/`（WiFi 驱动）、
`drivers/gpu/drm/i915/`（核显）、`drivers/nvme/`（SSD）、`drivers/net/ethernet/realtek/`（有线网卡）。

---

## 3. 推荐阅读顺序

> 原则：**自顶向下，先框架后细节；跟着一条真实数据流走，不要漫无目的浏览。**

### 第 1 步：读官方文档建立全局观（1~2 天）
- 解包 `linux-7.2-docs-htmldocs.tar.xz`，先读：
  - `index.html` → 总体地图
  - `admin-guide/index.html`（系统管理视角）
  - `kernel-hacking/index.html`（**开发入门必读**）
  - `process/howto.rst`、`process/submitting-patches.rst`（社区流程）
- 目的：知道「有什么」而不是「是什么」，建立文档检索习惯。

### 第 2 步：跟着启动流程走一遍（理解骨架）
阅读顺序（x86_64）：
1. `arch/x86/boot/` 与 `arch/x86/kernel/head_64.S`（汇编引导，能看懂流程即可，不必逐行）
2. `init/main.c` 的 `start_kernel()` —— **整个内核的初始化主线**，跟着它走：
   `setup_arch` → `mm_init` → `sched_init` → `init_IRQ` → `time_init` → `rest_init`
3. `rest_init` 创建 `kernel_init`（PID 1）和 `kthreadd`（PID 2）
4. `kernel_init` → `kernel_init_freeable` → `run_init_process("/sbin/init")` → 进入用户态

### 第 3 步：选一条系统调用深挖（理解交互）
推荐 `read()` 或 `open()`，链路：用户态 libc → `arch/x86/entry/entry_64.S` 的
`entry_SYSCALL_64` → `do_syscall_64` → `fs/read_write.c:ksys_read()` → `vfs_read` →
具体文件系统（如 `fs/ext4/file.c`）→ 页缓存 `mm/filemap.c` → 块层 → 驱动。

### 第 4 步：理解核心子系统（按需深入）
- 进程：`kernel/sched/core.c`（`__schedule` 是调度的心脏）、`kernel/fork.c`（`copy_process`）
- 内存：`mm/page_alloc.c`（页分配器）、`mm/slub.c`（小对象分配）、`mm/vmscan.c`（回收）
- 并发：`kernel/locking/`、`include/linux/mutex.h`、`spinlock.h`
- 中断：`kernel/irq/manage.c`、`arch/x86/kernel/irq.c`

### 第 5 步：读一个完整驱动（理解设备模型）
推荐本机 WiFi 驱动 `drivers/net/wireless/intel/iwlwifi/`：
1. `iwlwifi/iwl-drv.c`：模块入口（`module_init`）、probe 流程、firmware 加载（`iwl_req_fw_callback`）
2. 看它如何注册到 `cfg80211`（`iwlwifi/mvm/mac80211.c` 的 `iwl_mvm_mac_ops`）
3. 体会：驱动 = 「向内核框架注册一组回调 + 处理硬件中断/命令队列」

### 第 6 步：实战修改与调试
- 在某个函数里加 `printk`/`pr_info`，重编、安装、`dmesg` 观察（本仓库 buildconfig 可直接复现）
- 用 `trace-cmd`/`bpftrace` 看函数调用（内核已开 BTF，`CONFIG_DEBUG_INFO_BTF=y`）
- 用 `make htmldocs` 的 `kernel-doc` 注释反查 API 用法

---

## 4. 如何理解内核的运行机制

### 4.1 三大执行上下文（最重要的一课）
| 上下文 | 何时发生 | 特点 | 能做什么 |
|---|---|---|---|
| 进程上下文 | 进程主动调用系统调用/发生缺页等 | 可睡眠、可调度 | 几乎一切 |
| 软中断（softirq）/tasklet | 中断返回后、下半部处理 | 不可睡眠 | 网络收包等 |
| 硬中断（hardirq） | 硬件打断 | 极短、不可睡眠、不可持有普通锁 | 应答硬件、标记工作 |

判读技巧：函数里出现 `might_sleep()` / 调用了 `kmalloc(GFP_KERNEL)` → 进程上下文；
出现 `in_interrupt()` / `irq_disabled()` → 中断上下文。

### 4.2 内核的「对象化」：kobject / sysfs / 引用计数
```text
kobject（内核对象）→ 挂到 sysfs（/sys），通过 kref 管理生命周期
```
读 `/sys` 就是「看内核对象树」。`include/linux/kobject.h` 是理解设备模型的门票。

### 4.3 异步与事件流
内核大量使用：回调（callback）、工作队列（workqueue）、等待队列（waitqueue）、
completion、定时器。看到一个 `struct xxx_ops` 或函数指针表，就是「框架要回调你」的信号。

### 4.4 内存管理的层次
```
进程虚拟地址 → mmap/vma（mm/mmap.c）
   → 页表（arch/x86/mm/pgtable.c）
      → 物理页（伙伴系统 mm/page_alloc.c）
         → 小对象（slub mm/slub.c）
         → 页缓存/匿名页（mm/filemap.c / mm/shmem.c）
         → 回收（mm/vmscan.c）
```

### 4.5 网络栈分层（本机 WiFi 相关）
```
应用 → socket（net/socket.c）→ inet（net/ipv4/af_inet.c）→ TCP/UDP（net/ipv4/tcp.c）
    → IP（net/ipv4/ip_output.c/ip_input.c）→ 邻居（net/core/neighbour.c）
    → 驱动接口（net/core/dev.c: dev_queue_xmit / net_rx_action）
    → 无线：mac80211（net/mac80211/）→ cfg80211（net/wireless/）→ iwlwifi 驱动
    → 固件/硬件
```

---

## 5. 内核文本/符号的「语义字典」

### 5.1 常见前缀/后缀的含义
| 文本 | 含义 |
|---|---|
| `xxx_ops` | 操作函数表（回调集合），如 `file_operations`、`net_device_ops` |
| `xxx_register_*` / `xxx_unregister_*` | 向框架注册/注销 |
| `xxx_probe` / `xxx_remove` | 驱动发现设备/移除设备时被调用 |
| `xxx_init` / `xxx_exit` | 模块加载/卸载入口（`module_init`/`module_exit` 展开而来） |
| `__init` / `__exit` | 标记初始化/退出函数（`__init` 内存用后释放） |
| `static inline` | 内联函数（头文件里最常见） |
| `XXX_` 大写常量 | 编译期宏/标志位 |
| `pr_info`/`pr_err`/`dev_info` | 日志宏（`dev_*` 带设备信息） |
| `GFP_KERNEL` / `GFP_ATOMIC` | 内存分配标志：可否睡眠 |
| `RCU` | Read-Copy-Update 无锁读同步机制 |
| `WRITE_ONCE`/`READ_ONCE` | 防止编译器优化掉对共享变量的读写 |

### 5.2 Kconfig 符号语义（.config 里的 =y/=m/=n）
| 写法 | 含义 |
|---|---|
| `CONFIG_XXX=y` | 编进内核（built-in），vmlinux 的一部分 |
| `CONFIG_XXX=m` | 编成模块（.ko），可动态加载 |
| `# CONFIG_XXX is not set` | 禁用 |
| `CONFIG_XXX="..."` | 字符串参数（如 `CONFIG_LOCALVERSION="-custom"`） |
| `CONFIG_XXX=42` | 数值参数（如 `CONFIG_HZ=1000`） |

> 判断某个功能是否可用：`grep CONFIG_XXX /boot/config-$(uname -r)`；
> 判断模块是否加载：`lsmod`；设备是否绑定：`ls /sys/bus/pci/devices/.../driver`。

### 5.3 版本号与 vermagic
`vermagic: 7.2.0-custom SMP preempt mod_unload modversions` 的含义：
- `7.2.0-custom`：内核版本 + LOCALVERSION
- `SMP`：对称多处理支持开启
- `preempt`：可抢占内核
- `mod_unload`：模块可卸载
- `modversions`：模块版本校验（符号 CRC）
> 模块与内核 vermagic 不匹配将拒绝加载 —— 这是「换内核后驱动必须重编」的根本原因。

### 5.4 文档中的 RST 文本
内核文档是 reStructuredText（`.rst`）：
- `===` 标题、`.. _anchor:` 锚点、`:ref:\`xxx\`` 交叉引用
- `.. kernel-doc::` 指令会把 C 源码里的 `/** */` 注释自动提取成文档 —— **读 C 注释就是读文档**

---

## 6. 内核构建系统的文本含义（Kconfig / Kbuild / Makefile）

| 文件/目标 | 作用 |
|---|---|
| `Kconfig`（各目录） | 定义配置项树（menuconfig 的文本来源） |
| `Kbuild` / `Makefile`（各目录） | 定义如何编译该目录 |
| `scripts/Makefile.build` | 核心编译规则（`obj-y` 编入内核，`obj-m` 编成模块） |
| `make menuconfig` | 文本菜单配置界面 |
| `make olddefconfig` | 用默认值补齐新符号（旧配置迁移用） |
| `make -j8` | 编译 vmlinux + 全部模块 |
| `make modules_install` | 安装模块到 `/lib/modules/$(uname -r)/` |
| `make install` | 安装内核到 /boot 并触发 initramfs/grub 更新 |
| `make bindeb-pkg` | 生成二进制 deb 包（image/headers/libc-dev/dbg） |
| `make htmldocs` | 从 `.rst` + 源码注释生成 HTML 文档 |

读懂一行 `Kbuild`：
```makefile
obj-$(CONFIG_IWLWIFI) += iwlwifi.o    # CONFIG_IWLWIFI=m → 编译 iwlwifi.ko
iwlwifi-objs := iwlwifi/iwl-drv.o ... # 模块由哪些 .o 组成
ccflags-y += -D...                    # 该目录附加编译参数
```

---

## 7. 实践：从一次开机到一次系统调用的完整链路

### 7.1 开机链路（x86_64, UEFI + GRUB）
```
UEFI 固件 → GRUB（/boot/grub/grub.cfg 选内核+initrd）
  → vmlinuz（bzImage，含引导头+压缩内核）
  → arch/x86/boot/head_64.S（解压、设置页表、进入 long mode）
  → arch/x86/kernel/head_64.S（初始页表、CPU 初始化）
  → start_kernel()（init/main.c）
  → 挂载 initramfs（initrd.img，含必需驱动与 busybox 式 init）
  → /sbin/init（systemd）→ 启动服务 → NetworkManager 加载 iwlwifi → WiFi 连上
```

### 7.2 一次 read() 系统调用链路
```
app: read(fd, buf, n)
  → libc syscall 封装
  → entry_SYSCALL_64（arch/x86/entry/entry_64.S）
  → do_syscall_64（syscall 表查表，arch/x86/entry/common.c）
  → ksys_read（fs/read_write.c）
  → vfs_read → file->f_op->read_iter（VFS 多态分发）
  → 具体 fs（ext4）→ 页缓存（mm/filemap.c: filemap_read）
  → 缺页/未命中 → 块层 bio → nvme 驱动（drivers/nvme/host/）
  → 硬件 DMA → 中断应答 → 数据拷贝回用户空间 → 返回
```

> 建议：读代码时手动画这条链路的调用图，比读十篇博客都有效。

---

## 8. 给 AI 辅助阅读的提示词模板

把本仓库的 `ai-context/session-context.md` + 下列模板一起喂给 AI，可显著提高二次开发效率：

```text
你是 Linux 内核专家。基于以下上下文辅助我进行内核二次开发：
[粘贴 ai-context/session-context.md 内容]

我的开发目标是：______（例如：为本机开启 PREEMPT_RT / 为 iwlwifi 增加自定义参数 / 裁剪内核到最小）
请：
1. 指出需要修改/新增的配置项及其在 Kconfig 中的位置
2. 指出涉及的核心源文件与关键函数（给出文件路径和函数名）
3. 评估风险（稳定性、与现有模块的依赖、回退方案）
4. 给出修改后的编译与验证步骤（基于我的 buildconfig）
5. 如果涉及无线模块，检查 iwlwifi/cfg80211/mac80211 的交互影响
```

---

*本文档由 DSH agent 于 2026-08-19 生成，基于 linux-7.2 源码与本次实际编译实践。*
