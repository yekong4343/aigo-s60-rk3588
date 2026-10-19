# RK3588 NVR 启动路径总览

# RK3588 NVR 飞牛/Mine 双系统 — 启动路径与现状总览

> 整理时间：2026-10-19　设备：Rockchip RK3588 NVR DEMO LP4 V10 S60 Board
> 当前内核：`6.18.18-trim #4`（自编译 vanilla 6.18.18 + fnOS config，LOCALVERSION=-trim）
> 主用系统：fnOS 飞牛（根 `nvme0n1p2`，btrfs）　备用系统：Mine Debian 12（根 `nvme0n1p1`，ext4）

---

## 一、设备与连接

| 系统 | 根分区 | IP | 账号/密码 | 用途 |
| --- | --- | --- | --- | --- |
| fnOS 飞牛 | `nvme0n1p2` (30G, btrfs, label `rootfs_nvme2`) | 192.168.31.221（静态） | steven / steven | 主力 NAS 系统 |
| Mine Debian 12 | `nvme0n1p1` (32G, ext4) | 192.168.31.179（DHCP，可能变） | linaro / 123456 | 备用 / 编译用途（无 sudo，用 `echo 123456 ``\| su root -c`） |

- fnOS 侧有 sudo：`echo steven \| sudo -S -p '' <cmd>`。
- 串口：COM5，1500000 8N1。
- 智能插座（断电重启）：`curl -X POST http://192.168.31.135:8123/api/webhook/tuya_shufang -H "Content-Type: application/json" -d '{"action":"on"}'`（`off` 为断电）。
- Mine 冷启动后 SSH 就绪约需 10 分钟，排查失连时别过早下结论。

---

## 二、存储介质与分区

### eMMC（14.6G）

| 分区 | 大小 | 文件系统 | 标签 | 作用 |
| --- | --- | --- | --- | --- |
| p1 | 4M | — | — | **U-Boot（4MB 签名 FIT）**，启动ROM 先加载它 |
| p2 | 4M | — | — | 备份区 |
| p3 | 64M | — | — | 原厂保留 |
| p4 | 128M | — | — | 原厂保留 |
| p5 | 32M | — | — | 原厂保留 |
| **p6** | 3G | ext4 | — | **控制面**：U-Boot 脚本 + 开关文件（BOOTCOMMAND 固定指向它） |
| p7 | 128M | ext4 | oem | 挂 `/oem` |
| p8 | 7G | btrfs | rootfs | **fnOS 旧根（回退备份）** |
| **p9** | 768M | ext4 | srbboot | **数据面**：内核 / dtb 载荷 |
| p10 | 3.5G | — | — | 原厂保留 |

> 注：`/boot/srb/uboot.cmd` 的注释里写的是 p6=6656MiB / p9=1742MiB，与实测 3G / 768M 不符（分区后来调整过），以 `lsblk` 实测为准。

### NVMe（119.2G）

| 分区 | 大小 | 文件系统 | 标签 | 作用 |
| --- | --- | --- | --- | --- |
| p1 | 32G | ext4 | — | Mine 根 |
| p2 | 30G | btrfs | rootfs\_nvme2 | **fnOS 根（当前在用）** |
| p3 | 57.2G | linux\_raid\_member | steven:0 | 存储池 → md0 → LVM `trim_74ce...` → LV "0" (ext4) → `/vol1` |

> ⚠ `/vol00/RemovableDisk`(p6) 与 `RemovableDisk_3`(p9) 是飞牛自动挂载的**引导链分区**，绝不可在 Web UI 里格式化。

---

## 三、启动链（核心）

### 3.1 当前真实路径（2026-10-19 实测确认）

```
 ① eMMC p1（4MB 签名 FIT）
      │  启动ROM 加载
      ▼
 ② mainline U-Boot
      │  编译期 BOOTCOMMAND 固定：
      │  load mmc 0:6 ${scriptaddr} /boot/srb/uboot.scr; source ${scriptaddr}
      ▼
 ③ 执行 p6 的 /boot/srb/uboot.scr
      │  探测 p9：能 load 到 /srbboot.id ⇒ srbdev=9，否则回退 srbdev=6
      ▼
 ④ 启动菜单（bootmenu，10 秒超时）
      │  默认项 index=3
      ▼
 ⑤ srb_boot_fnos_nvme2_stable：
      setenv bootargs "…root=/dev/nvme0n1p2 rootfstype=btrfs…pci=nomsi"   ← 覆盖 dtb 里的 /chosen/bootargs
      load mmc 0:9 0x02000000 /boot/srb/Image.fnos-618-stable
      load mmc 0:9 0x1A000000 /boot/srb/dtb-fnos-618-nord3.dtb
      booti 0x02000000 - 0x1A000000
      ▼
 ⑥ 直接进入 6.18.18-trim，挂载 nvme0n1p2 为根，跑 systemd
      ▼
 ⑦ triminit 按 postgres trim 库 mount 表挂载 /vol1（存储池）
```

**关键结论：现在是 U-Boot 直接 **`booti`**，不再走 kexec。**

判定依据（两点实证）：

1. `uboot.cmd` 里默认项 `bootmenu_3` 就是 `srb_boot_fnos_nvme2_stable`，本身就是 `booti`。
2. **bootargs 被 U-Boot 覆盖**：p9 上 `dtb-fnos-618-nord3.dtb` 的 `/chosen/bootargs` 写的是 `root=/dev/nvme0n1p5`，而实际 `/proc/cmdline` 是 `root=/dev/nvme0n1p2`——只有 U-Boot 的 `setenv bootargs` 才会产生这种覆盖。
3. fnOS 系统里 `srb-autojump.service` **根本不存在**（`No such file or directory`），它属于 p6 的 6.1.75 系统。

### 3.2 启动菜单

| 项 | 标题 | 命令 | 内核 + dtb |
| --- | --- | --- | --- |
| 0 | fnOS 6.18.18 / NVMe p2 | `srb_boot_fnos_nvme2` | Image.fnos-618 + dtb-fnos-nord3.dtb |
| 1 | fnOS 6.18.18 / eMMC p8（旧根备份） | `srb_boot_fnos_emmc` | Image.fnos-618 + dtb-fnos-nord3.dtb |
| 2 | Mine 6.1.75 / NVMe rootfs | `srb_boot_nvme` | Image.mine + dtb-nvme-root.dtb |
| **3** | **fnOS 6.18.18 重编稳定版 / NVMe p2（默认）** | `srb_boot_fnos_nvme2_stable` | Image.fnos-618-stable + dtb-fnos-618-nord3.dtb |
| 4 | U-Boot shell / 命令行 | `srb_boot_shell` | — |

- `bootmenu_delay=10`、`bootmenu_default=3`；按 ESC / 不选则落到脚本末尾 fallback：`run srb_boot_fnos_nvme2`（等同菜单 0）。
- 原厂系统不在菜单里，需命令行 `run srb_boot_stock`（Image.raw + fdt-stock.dtb）。

### 3.3 开关文件（都在 p6 的 `/boot/srb/` 下，Linux 侧 touch / rm 即可）

| 文件 | 作用 |
| --- | --- |
| `.uboot-prompt` | 停在 U-Boot 命令行（优先于菜单） |
| `.uboot-nomenu` | 跳过菜单，直接引导 fnOS/NVMe p2（无人值守） |
| `.uboot-nettest` | 网卡自检（ping 网关 / 编译机 / dhcp） |
| `.uboot-tftpscan` | 遍历 blksize 量 RX 帧长上限 |
| `.uboot-nvme` | NVMe 自检 |
| `.uboot-tftp` | 内核与 dtb 全部从编译机 TFTP 取，不走 eMMC |
| `.autojump-fail-count` | kexec 失败计数（**清零 = 删除该文件**） |

### 3.4 双分区冗余设计

- **p6 控制面**：BOOTCOMMAND 编译期固定指向它，存脚本与开关。
- **p9 数据面**：存内核 / dtb 载荷，脚本运行时自动探测切换。
- 探测到 p9 用 p9，探测不到回退 p6 ⇒ 任一分区损坏，另一份能顶上。
- 想彻底脱离 p6：重编 U-Boot 把 `BOOTCOMMAND` 的 `mmc 0:6` 改成 `mmc 0:9`，其余逻辑不变。

### 3.5 历史路径：mini-kexec（当前未使用，仅存档）

旧路径是 `U-Boot → p6 的 6.1.75 内核 → srb-autojump.service → kexec 跳 6.18.18`。相关文件仍在：
`/usr/local/sbin/srb-autojump.sh`、`srb-go-stable`（insmod `soft_reboot.ko img=… dtb=… go=1`）。

⚠ **踩过的坑**：`srb-go-stable` 传的 `dtb=dtb-fnos-618-nord3-p2.dtb` **实际不生效**——曾只改 p6 上这个 dtb，重启后 `/proc/device-tree` 里 `dsa,member` 依旧是旧值。所以改 dtb 必须**把 p6 + p9 的候选全部改掉**才能覆盖实际生效的那份。

---

## 四、当前生效的内核与 dtb

| 文件 | 位置 | 说明 |
| --- | --- | --- |
| `Image.fnos-618-stable` | p9 `/boot/srb/` | **当前在用**，6.18.18-trim #4，含 trimacl 别名补丁 + PCIe PHY 内置 |
| `dtb-fnos-618-nord3.dtb` | p9 `/boot/srb/` | **当前在用**（U-Boot 从 `mmc 0:9` 加载） |

内核构建要点：

- 必须内置（非模块）：`CONFIG_PHY_ROCKCHIP_SNPS_PCIE3=y`、`CONFIG_GPIO_SYSCON=y`——**无 initrd**，根挂载前不能加载模块，否则 NVMe 不出现、卡在 `Waiting for root`。
- `LOCALVERSION="-trim"`（vermagic `6.18.18-trim`），否则与官方模块目录不匹配。
- 存储池需 MD\_LLBITMAP 支持：曾通过替换 `/lib/modules/6.18.18-trim/` 下的 md 模块解决。

---

## 五、网口拓扑（10 口，已全部修好）

| 网卡 | 交换机 | 芯片 | 端口 |
| --- | --- | --- | --- |
| end1（gmac1） | switch0 | RTL8367S（compatible 写 `realtek,rtl8365mb`） | eth1 \~ eth5（CPU 口 port@7） |
| end0（gmac0） | switch1 | RTL8367RB-VB | eth6 \~ eth10（CPU 口 **port@6**） |

- eth1 挂 OVS（`eth1-ovs` 持有 192.168.31.221）。
- eth8 已插线并实测：ping 网关 4/4 通、0% 丢包、0.5ms；其余口 NO-CARRIER 是没插线，非故障。

**曾经不通的根因与修复**（本次最重要的修复）：

- 症状：链路 UP、RX 有计数，但配 IP 后 ping 不通，ARP 一直 INCOMPLETE；dmesg 每秒刷 `net end0: could not find user for port 2`（累计 4257 次）。
- 根因：switch1 的设备树里 `dsa,member = <1 1>` ⇒ 注册成 **tree 1 的 switch index 1**；而 DSA 收包走 `dsa_conduit_find_user(dev, switch_id=0, port)`，**switch\_****id 硬编码 0** ⇒ 永远匹配不上 ⇒ 入站包全丢。switch0 没有该属性（index 天然为 0）所以一直正常。
- 修复：`dsa,member` 改为 `<1 0>`（保留 tree 1，只把 switch index 归零）。p6 + p9 共 **65 个** dtb 全部改，备份后缀 `.bak-dsamember-20261019-0915`。
- ⚠️ **绝不能直接删除该属性**：两个交换机会都落到 tree 0 index 0 → `A DSA switch with index 0 already exists in tree 0` / `-EEXIST` → switch1 注册失败 → eth6\~eth10 全部消失。

---

## 六、fnOS 存储与 trimacl 现状

- 存储池 57.2G：`nvme0n1p3 → md0(linear) → LVM → ext4`，挂载点 `/vol1`，开机（含断电冷启动）自动挂载。
- 挂载点由 `triminit` 在启动时按 postgres `trim` 库 `mount` 表的 id 挂载（不是 `storage` 表，那是云盘/网盘表）。池**创建后必须重启**才会挂载。
- 挂载命令形如 `mount -t ext4 -o trimacl,prjquota /dev/dm-0 /vol1`。
- `trimacl`** 是 fnOS 私有挂载选项**，本内核打了别名补丁（`trimacl` → `Opt_acl`），解决"挂载报 exit 32 / 创建存储空间失败"。注意这只是别名，**不等于官方完整的 Windows 风格 ACL**（无 allow/deny、私有 xattr handler、`trim_syscall_*`）。

---

## 七、已知遗留问题

| 问题 | 状态 |
| --- | --- |
| `/fs` 挂载失败：`unknown filesystem type 'trimafs'`（主线无 TRIMAFS2\_FS） | 未解决，导致 `trim_trashbind.service` 每 5s 失败重启，无害 |
| 官方完整 trimacl 能力不可用 | 别名补丁替代，已另装 `fnos-appdir-perm-guard.service` 兜底应用中心权限 |
| `pcie30-avdd` -517、HDMI/DRM 报错 | 无害 |
| 内核日志曾大量 `net end0: could not find user for port N` | **已归零** |
| 官方内核 `linux-image-6.18.18.c1145-trim` 的 rtl8365mb 不认 `rtl8367rb` | 官方内核上网口 6-10 仍不通，故坚持用自编译内核 |

---

## 八、操作速查

**改菜单 / 默认项**（不必重编 U-Boot）

```
1. 编辑 p6 /boot/srb/uboot.cmd
2. mkimage -C none -A arm64 -T script -d uboot.cmd uboot.scr
3. 同步到 p9，用 cmp 校验
4. bootmenu 编号必须从 0 连续
```

工作区留档：`edit_uboot_menu.py`（增删菜单项）、`edit_uboot_default.py`（改默认索引）、`uboot.cmd.new-20261018`。

**改 dtb**

```
fdtput -t u <dtb> /switch1 dsa,member 1 0     # 类型只有 s/i/u/x + b/h/l 前缀，写 u32 会报 usage
```

⚠ 必须 p6 + p9 **所有候选 dtb 一起改**（原因见 3.5）。

**回滚**

- dtb：`*.bak-dsamember-20261019-0915`
- 内核：`Image.fnos-618-stable.bak-pretrimacl-20261019-071208`、`Image.fnos-618-stable.bak-prephy-20261018-230640`
- U-Boot 脚本：`uboot.{cmd,scr}.bak-20261018-*`（p6/p9 各一份）
- NVMe 分区表：`gpt-backup-nvme0n1-20261018.txt`

**排查要点**

- 内核日志曾被 `net end0: could not find user for port N` 刷屏，排查其他问题时记得 `grep -v` 过滤（现已修复）。
- fnOS 私有内核源码拿不到：repo.fnnas.com 只有 linux-image/headers，无 linux-source。

---

## 九、应急

- **变砖风险兜底**：U-Boot 菜单 + p6/p9 双份冗余，最坏可手选菜单项或 `run srb_boot_stock` 回原厂。
- **彻底救砖**：eMMC p1 MaskROM/Loader 通道 + rkdeveloptool。
- **断电重启**：智能插座 webhook（见第一节）。
