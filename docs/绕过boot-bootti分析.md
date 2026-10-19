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