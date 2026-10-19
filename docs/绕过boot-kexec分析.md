# 在签名锁死的 RK3588 上换内核：从 Loader 侦察到手写 mini-kexec

> 设备：RK3588 S60 超级网关（eMMC + GPT、A/B 双槽 U-Boot、LPDDR4X 8GB、NVMe 128G）
> 目标：不刷 SPL、不动 U-Boot、不碰验签，把运行中的 Linux 内核换成自己编译的
> 结果：已跑通，自编译 `6.1.75-mine` 稳定作为主力系统，启动 ~20 秒
> 适用：你**自己拥有**的设备，目的是拿回安装自己系统的自由

---

## 一句话结论

**正统 kexec 在这台机器上是死的**（`CONFIG_KEXEC is not set`）。
我们真正做的事是：**先排除掉所有 bootloader 侧的执行入口，再确认 Linux 侧的 kexec 被裁掉，
最后用一个自写的 LKM 自己实现了 kexec 的核心那一跳** —— 就是 `soft_reboot.ko`（mini-kexec）。

---

## 第一部分：侦察 —— 为什么 bootloader 这条路走不通

### 1.1 先摸清启动链与验签边界

```
BootROM → SPL（RKSS @ LBA 0x40） → u-boot.itb（p1 @ LBA 0x4000）
        → boot.img（p3 @ LBA 0x8000） → Linux（root=eMMC p6）
```

两级验签：

| 阶段 | 验什么 | 是否强制 |
|---|---|---|
| SPL | `uboot.itb` A/B 两槽（加载顺序**硬编码** `0x4000 → 0x5000`，misc priority 无效） | 强制 |
| U-Boot proper | p3 / p4 的 FIT，`conf required` | **强制** |

换钥匙这条路是**结构性死结**：`hashed-nodes` 里包含 `/images/fdt/hash` ⇒
不改 hash → `Bad hash`；改 hash → conf 签名失效。两头堵死。

顺带查明：OTP 的 `BootKeyHash`（`@0x260`）**全零** ⇒ SecureBoot 实际未熔断。
所以理论上有第四条路 —— patch SPL 里的 `fit_config_verify_sig`，但**写错就是砖**，
且本机没有 SD 卡槽、没有可用 SPI NOR，救砖唯一出路是 Maskrom→Loader，成功率不稳定。

### 1.2 进 Loader，把所有"能执行代码"的入口逐个试一遍

进 Loader / Maskrom 的三条路（都实测过）：

| 方式 | 落点 |
|---|---|
| p3 的 conf 签名故意做坏 → `altbootcmd` | **自动进 RKUSB**，全盘 `0x0~0x1d1f000` 可读写（最安全的试验场） |
| `reboot-mode` 寄存器 `0xfd588080` 写 `loader` | 进 Loader（寄存器是**粘性**的，软重启不清） |
| 串口 Ctrl+B + 重新上电 | 落 `2207:350B` = **Maskrom**（BootROM 下载模式） |

PID 判别（这条很关键，能省掉大量误判）：

- `2207:350B` + bcd `0x0100` + 字符串描述符索引全 0 ⇒ **BootROM Maskrom**
- `2207:350B` + bcd `0x0223` + 字符串非 0 ⇒ Windows 驱动下的 Loader
- `2207:0006` ⇒ Linux 侧看到的真 Loader

**在 Loader 里实测到的 rockusb 命令集**：

```
0x14 0x15 0x16 0x17 0x18 0x19 0x1b 0xaa 0xff   ← 全是存储类
```

**没有 boot、没有 exec、没有任何"执行一段代码"的命令。**
U-Boot 侧同样封死：`Cmd interface: disabled`、`bootdelay=0`、
env 只编译了 `nowhere` 后端（改 env 等于换钥匙的同一个死结）、fastboot 没有 `boot` 命令。

### 1.3 由此确立的一条硬判据

```
SPL 崩 → 落 Maskrom (2207:350B)
    ├─ 灌 loader 成功 → Loader (2207:0006) → 有 LBA → 写回原厂 → ✅ 有救
    └─ 灌不进 loader  → 永远停在 350B → BootROM 里没有 LBA 代码 → ❌ 就是砖
```

> **"进 Maskrom ≈ 砖，进 Loader 才有救。"**

LBA 读写代码在 loader 里，不在 BootROM 里。这条判据决定了：**任何可能让 SPL 崩溃的实验，
都必须先打通 Maskrom → Loader，否则就是拿机器赌。**

### 1.4 侦察结论

**bootloader 这一层，能给的是"读写磁盘"，给不了"执行代码"。**
所以"换内核"这件事只能到 Linux 侧去做。这是整个转向的起点。

---

## 第二部分：Linux 侧的正统等价物，也都被裁掉了

`booti` 是 U-Boot 命令，Linux 起来后 U-Boot 已退出、内存被回收，用不了。
那就只剩 kexec。直接查配置：

```bash
zcat /proc/config.gz | grep KEXEC
# CONFIG_KEXEC is not set
# CONFIG_KEXEC_FILE is not set
# CONFIG_HIBERNATION is not set
# CONFIG_KPROBES is not set
```

再看 `/proc/kallsyms`（73637 行，`kptr_restrict=0`，符号可见）：

| 能查到 | 查不到（关键） |
|---|---|
| `machine_power_off`、`kernel_restart`、`psci_sys_reset` | `cpu_soft_restart` |
| `__enable_mmu`、`__cpu_setup`、`dcache_inval_poc` | `machine_kexec` |
| `alloc_contig_pages` | `arm64_relocate_new_kernel` |

**关 MMU 跳转的那一整套基础设施根本没编进内核** —— 不是"函数没导出"，是没编译。
`CONFIG_KPROBES` 又没开，连 kprobe 取地址的技巧都用不了。

---

## 第三部分：突破口 —— LKM

换内核需要内核态执行能力。查配置发现这条是活的：

```
CONFIG_MODULES=y
CONFIG_MODULE_UNLOAD=y
CONFIG_MODULE_SIG       未启用
CONFIG_MODVERSIONS      未启用
CONFIG_STRICT_DEVMEM=y  （但 LKM 直接 ioremap，可绕）
```

加上设备上有配套的 `/usr/src/linux-headers-6.1-rockchip` 和 gcc12，
于是**可以 insmod 未签名的自编 .ko**（`hello.ko` 编/加/卸全通，先做的实锤）。

⚠ 但要清楚 LKM 的边界：**它不能绕 FIT 验签**（验签在 U-Boot 阶段早做完了），
**也不能替代 kexec** —— 它只是给了"内核态任意代码"这个前提。
真正的那一跳，得自己写。

---

## 第四部分：手写 mini-kexec（`soft_reboot.ko`）

### 4.1 原理

真 kexec 缺的是 `arm64_relocate_new_kernel`：把新内核装好、关掉 MMU、跳过去。
我们自己在模块里实现这一段：

```
① alloc_contig_pages 拿一大块连续物理内存
② 把裸 arm64 Image + dtb + trampoline 拷进去
③ 现场构造一张 identity 页表（VA 数值 == PA）
④ 停副核 → 关中断 → 刷 cache → 切 TTBR0
⑤ 跳 trampoline：关 MMU → br 新内核入口
```

**为什么必须 identity 映射**：关 MMU 的瞬间 PC 立刻按物理地址取指，
而内核线性映射是 `VA = PA + 常量偏移` ⇒ 模块自身地址、线性映射地址**全都不能用**。
同时栈指针在关 MMU 后即刻失效 ⇒ **trampoline 全程不能碰栈**。

### 4.2 载荷布局

| 偏移 | 内容 |
|---|---|
| `+0` | arm64 裸 Image（41,447,936 B） |
| `+ALIGN(image_size, 2M)` = **42 MiB** | dtb |
| `+43 MiB` | trampoline（60 B，不用栈） |

Image 头 `flags = 0xa`（bit3=1）⇒ **可放在任意 2 MiB 对齐的物理地址**，不必是 `0x400000`。
`image_size = 42,205,184`（40 MiB + 256 KiB，**含 BSS**）—— 这个数决定了 dtb 该放哪。

### 4.3 trampoline（已 objdump 验证）

```asm
mrs   x9, sctlr_el1
and   x9, x9, #~1          /* M  : MMU off    */
and   x9, x9, #~4          /* C  : dcache off */
and   x9, x9, #~0x1000     /* I  : icache off */
msr   sctlr_el1, x9
isb
tlbi  vmalle1is
dsb   sy
isb
mov   x10, x0              /* 新内核入口 */
mov   x0,  x1              /* dtb 物理地址 */
mov   x1,  xzr
mov   x2,  xzr
mov   x3,  xzr
br    x10
```

arm64 Linux 入口约定：`x0 = dtb PA`，`x1/x2/x3 = 0`，MMU 与 dcache 均关闭。

### 4.4 踩过的六个坑（每一个都是实测死机换来的）

**① dtb 必须带 `/memory` 节点**
从 boot.img FIT 里静态抠出的 dtb **没有 `/memory`** —— RK 的 U-Boot 是在启动时由
`ft_system_setup()` 动态注入 DRAM 布局的。我们绕过 U-Boot 直接跳内核，就没人做这一步
⇒ `memblock` 空 ⇒ `paging_init` 分不到页表页 ⇒ `panic: Failed to allocate page table page`。
**解法：用 `/sys/firmware/fdt`**（运行内核的完整 dtb，U-Boot 已注入 `/memory`）。

**② dtb 落点必须避开 BSS，否则完全静默死机**
首轮把 dtb 放在 `+40 MiB`，正好落在 `[39.53, 40.25) MiB` 的 BSS 区间内。
`__primary_switched` 先存 `__fdt_pointer` 再 `__pi_memset` 清 BSS ⇒ dtb 被抹零
⇒ `setup_machine_fdt()` 失败进死循环，而 `setup_arch()` 早于 `console_init()`
⇒ **串口一个字都不吐**。改成 `ALIGN(image_size, 2M)` = 42 MiB 后解决。

**③ 本机跑在 EL2 且开了 VHE**
`HCR_EL2 = 0x488000002` ⇒ `E2H=1`、`TGE=1`：
必须同时写 `ttbr0_el2` / `sctlr_el2`；TLB 失效必须补 **`tlbi alle2is`**
（`vmalle1is` 管不到 EL2 的表项）。

**④ GIC / ITS 残留 —— 最隐蔽的一个**
软重启不经过复位，中断控制器的状态全留着：

```
GICv3: CPU0: Booted with LPIs enabled, memory probably corrupted
Call trace: valid_col ← its_send_single_command ← ... ← pci_msi_setup_msi_irqs
```

ITS 命令队列/Collection 表残留 ⇒ PCIe MSI 分配失败 ⇒ **NVMe 拿不到中断 ⇒ rootfs 挂不上**。
**解法**：跳转前 `GITS_CTLR &= ~1` 并轮询 bit31 Quiescent；每个 GICR 的
`GICR_CTLR &= ~1`（EnableLPIs）；**不要动 `GICR_WAKER`**（否则新内核可能叫不醒 RD）。

**⑤ USB 与 GMAC 的复位脉冲 —— `0xA00`，不是 `0x400`**
CRU `SOFTRST_CON(x)` 的基址偏移是 **`0xA00 + x*4`**（按 dtb 里的 reset id 打脉冲）。
写成 `0x400` 会打到 CLKSEL 上，直接毁掉时钟配置。
USB 还需要 `dr_mode` 改 `peripheral` + 删掉 `usb-role-switch`（绕开 FUSB302 仲裁）。
GMAC 则必须在 kexec 前 **unbind**，否则新内核 probe 时 DMA 控制器还留着上一任的状态。

**⑥ 编译限制**
内核用 `-mgeneral-regs-only` ⇒ **printk 里禁止 `%f` / double**，改用 `>> 20` + `& 0xFFFFF`。

### 4.5 一个实现细节：符号未导出怎么办

`alloc_contig_pages` 没导出，`kallsyms_lookup_name` 也没导出，`CONFIG_KPROBES` 又没开。
**解法**：从 `/proc/kallsyms` 读地址，作为 insmod 参数传进去。
因为 `RANDOMIZE_BASE` 未开，地址是稳定的：

```bash
grep -w alloc_contig_pages /proc/kallsyms   # ffffffc008203514
insmod soft_reboot.ko acp_addr=0xffffffc008203514 go=1
```

⚠ `acp_addr` **必须带 `0x` 前缀**，否则内核按十进制解析，报 `invalid for parameter`。

---

## 第五部分：怎么用（操作手册）

### 5.1 模块参数

| 参数 | 说明 |
|---|---|
| `img=` | 裸 arm64 Image 路径（从原厂 boot.img 提取，或自编译产物） |
| `dtb=` | dtb 路径（**必须是 `/sys/firmware/fdt` 派生的**，含 `/memory`） |
| `acp_addr=` | `alloc_contig_pages` 的地址（必需，带 `0x`） |
| `go=` | `0`=只诊断（安全）；`1`=真的跳 |
| `probe=` | `1`=只验证 identity 映射 + cache flush，不跳（安全） |
| `quiesce=` | `1`=跳转前 quiesce GIC ITS 并关 GICR LPIs（默认开） |
| `usb_reset=` | `1`=对 USBDP PHY0 / usb2phy0 / dwc3 打 CRU 复位脉冲 |
| `cru_softrst=` | CRU `SOFTRST_CON` 基址偏移，RK3588 = **`0xA00`** |
| `gmac_reset=` | `0`=关；`1`=打 GMAC1/GMAC0 脉冲；`2`=assert 并保持 |

### 5.2 物料（都放在 `/boot/srb/`，即 p6 里 — 不在验签链上）

```
Image.mine                自编译内核（42,715,648 B）
dtb-mmc-root.dtb          从 /sys/firmware/fdt 派生，root=/dev/mmcblk0p8
dtb-nvme-root.dtb         同上，root=/dev/nvme0n1p1（回退根）
soft_reboot-mine.ko       mini-kexec 模块（vermagic: 6.1.75-mine SMP preempt）
soft_reboot.ko            同上，vermagic: 6.1.75（原厂内核下用）
srb-go-mine               一键跳转脚本
```

### 5.3 一键跳转（`srb-go-mine`，已精简）

```sh
#!/bin/sh
S=/boot/srb

# ① 按当前内核挑匹配的模块（vermagic 必须对上，否则静默失败）
case "$(uname -r)" in
    *-mine) KO=soft_reboot-mine.ko ;;
    *)      KO=soft_reboot.ko ;;
esac

# ② 关键：kexec 前解绑 GMAC 与 Realtek 交换机
#    不这么做，新内核 probe GMAC1 时 DMA 还留着旧状态，end0 永远起不来
for d in fe1b0000.ethernet fe1c0000.ethernet; do
    echo "$d" > /sys/bus/platform/drivers/rk_gmac-dwmac/unbind 2>/dev/null
done
for s in switch switch1; do
    echo "$s" > /sys/bus/platform/drivers/realtek-smi/unbind 2>/dev/null
done

# ③ 取 alloc_contig_pages 地址，跳转
ACP=$(grep -w alloc_contig_pages /proc/kallsyms | awk '{print $1}')
insmod "$S/$KO" \
  acp_addr=0x$ACP \
  img=$S/Image.mine \
  dtb=$S/dtb-mmc-root.dtb \
  go=1 usb_reset=1 cru_softrst=0xA00 gmac_reset=0
```

用法：`srb-go-mine`（默认 eMMC p8 根）/ `srb-go-mine nvme`（回退到 NVMe 根）。

### 5.4 开机自动跳转

```ini
# /etc/systemd/system/srb-autojump.service
[Service]
Type=oneshot
ExecStart=/usr/local/sbin/srb-autojump.sh   # 内部调 srb-go-mine
```

完整链路：

```
上电 → SPL → U-Boot(验 p3) → 原厂内核(root=eMMC p6)
     → srb-autojump.service → srb-go-mine → 6.1.75-mine + eMMC p8 根
```

**eMMC 全程只读**（只往 p6/p8 这两个不在验签链的分区写），失败断电重启即回原厂系统。

---

## 第六部分：成果、边界与遗留

### 已达成

- 自编译 `6.1.75-mine` 稳定运行，启动 ~20 秒（原 96 秒），Docker / KVM / NPU / Mali / 温控全正常
- 10 个网口全部注册（两颗 RTL8367 交换机），系统数据全在 eMMC，拔掉 NVMe 照样自持运行
- **没有改动启动信任链**：重启后仍走 U-Boot 的 `key-dev` 验签路径，
  mini-kexec 只在本次运行期间生效 —— 这是"不改固件换内核"的关键性质

### 边界（别抱错的期待）

- **不是绕过 SecureBoot**：验签在 U-Boot 阶段已经完成，LKM 碰不到它
- **不是持久化 rootkit**：重启即回到原厂验签路径
- **死机需人工断电**：跳转类操作必然如此，已发生多次
- **重编内核后必须重编模块并同步**：vermagic 对不上会静默失败（`Invalid module format`）

### 已知遗留

`end0`（GMAC1，`fe1b0000`）在 kexec 之后起不来 —— `Failed to reset the dma`，
导致挂在它后面的 eth6~eth10 全部 DOWN。

已实测排除：dtb 配置、时钟（396M/150M 与原厂金标准完全一致）、复位线、电源域、
交换机驱动、kexec 前状态、重试时机。**原厂内核下手动 `ip link set end0 up` 完全正常**
（`Link is Up - 1Gbps/Full`，ping 网关 4/4 通），所以是 kexec 迁移的残留问题，不是硬件故障。
目前可用的是 eth1~eth5（`br1`，带管理 IP）。

---

## 附：判断同类设备能否照搬的 checklist

```bash
# 1. 正统 kexec 在不在
zcat /proc/config.gz | grep -E "CONFIG_KEXEC|CONFIG_KEXEC_FILE"

# 2. 关 MMU 跳转的基础设施在不在
grep -E " (cpu_soft_restart|machine_kexec|arm64_relocate_new_kernel)$" /proc/kallsyms

# 3. LKM 能不能加载（决定能不能自己写）
zcat /proc/config.gz | grep -E "CONFIG_MODULES=|CONFIG_MODULE_SIG|CONFIG_MODVERSIONS"
ls /usr/src/linux-headers-$(uname -r) 2>/dev/null

# 4. 符号取不取得到
cat /proc/sys/kernel/kptr_restrict          # 需要 0
grep -w alloc_contig_pages /proc/kallsyms   # 需要能查到地址
```

判定：

- **1 或 2 有一样是活的** ⇒ 直接用标准 kexec，不用自己写
- **都死了但 3、4 是活的** ⇒ 可以照搬本文的 mini-kexec 路线
- **3 也死了** ⇒ 没有内核态执行能力，只能走 patch SPL（高风险，需先打通 Maskrom→Loader）

---

## 免责

本文记录的是在**自己拥有的设备**上获取安装自由系统的技术过程，
用于学习 ARM64 启动流程与 Linux 内核引导机制。
所有操作均在本人设备上完成，未涉及第三方设备的未授权访问。
請勿用于绕过你不拥有的设备的安全机制。
