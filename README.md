# immortalwrt-rax3000me-build

RAX3000Me / RAX3000M 云编译仓库。源码来自 `tfnhui/immortalwrt-mt798x-25.12`
（上游 `chasey-dev/immortalwrt-mt798x-rebase`），内核 6.12，ImmortalWrt 25.12，
MTK 闭源无线驱动 mt_wifi（SDK 7.6.7.3）。

推送 `.github/workflows/build.yml` 即自动编译，产物发布到 Releases。

## 刷机前必做：备份

Telnet / SSH 进原厂系统：

```sh
cat /proc/mtd           # 确认 BL2 / u-boot-env / Factory / FIP 各是 mtd 几
dd if=/dev/mtd0 of=/tmp/mtd0_BL2.bin
dd if=/dev/mtd1 of=/tmp/mtd1_u-boot-env.bin
dd if=/dev/mtd2 of=/tmp/mtd2_Factory.bin
dd if=/dev/mtd3 of=/tmp/mtd3_FIP.bin
```

`Factory` 是无线校准 EEPROM，信号好坏全靠它，务必传回电脑保存。

## 硬件判定（决定刷哪个 sysupgrade）

```sh
dmesg | grep -iE "7531|7530|8855|switch|mdio"
```

- 出现 `mt7531` / `mt7530` → 老硬件，sysupgrade 用 **cmcc_rax3000m** 的
- 出现 `8855` → 新硬件，sysupgrade 用 **cmcc_rax3000me** 的

本仓库 DDR3 机型（带 USB3.0 的 RAX3000Me）：BL2 / FIP **只能**用
`cmcc_rax3000me-nand-ddr3-` 开头的两个文件，刷 ddr4 版必砖。

## 刷机步骤

1. 原厂系统里写入新 bootloader

```sh
mtd write /tmp/...-cmcc_rax3000me-nand-ddr3-preloader.bin BL2
mtd write /tmp/...-cmcc_rax3000me-nand-ddr3-bl31-uboot.fip FIP
```

分区名大小写报错就用设备节点：`BL2` -> `/dev/mtd0`，`FIP` -> `/dev/mtd3`。

2. TFTP 起临时系统

- 把 `initramfs-recovery.itb` 重命名，**去掉文件名里的版本号**，
  改成 `immortalwrt-mediatek-filogic-cmcc_rax3000me-initramfs-recovery.itb`
- 电脑有线网卡设 `192.168.1.254`，掩码 `255.255.255.0`，网关 `192.168.1.1`
- tftpd64 指向该文件目录，Server interfaces 绑 `192.168.1.254`
- 网线接 LAN 口，断电 -> 按住 Reset 通电 -> 等 5~6 秒松手

3. 浏览器开 `192.168.1.1`，上传对应的 `squashfs-sysupgrade.itb`

板型名不匹配时（Me 的机器刷 M 的固件）用：`sysupgrade -F -n /tmp/xxx.itb`

## 固件内容

- 闭源无线与加速：mtwifi-cfg、turboacc-mtk、eqos-mtk、kmod-mt_wifi、kmod-warp、kmod-mediatek_hnat
- 常用：磁盘管理、ksmbd 文件共享、ttyd、UPnP、WOL、DDNS、WireGuard、Tailscale、
  流量统计、SQM、watchcat、minidlna、udpxy、Adblock、statistics、Argon / Material3 主题
- 智能家居：mosquitto（MQTT）、umdns、igmpproxy、USB 串口驱动（Zigbee 协调器用）

默认管理地址与密码以刷入固件说明为准。

## 救砖

TTL（CH340）接 GND/RX/TX，用 mtk_uartboot 加载 `mt7981-ram-ddr3-bl2.bin`
（tfnhui Release 里有）内存启动后重刷。
