# RAX3000Me 定制固件与刷机记录

> **设备**：CMCC RAX3000Me（USB3.0 / DDR3 / 128MB SPI NAND）
> **固件**：ImmortalWrt 25.12-SNAPSHOT（内核 6.12.87）+ MTK 闭源无线驱动 mt_wifi
> **编译仓库**：https://github.com/Fancok/immortalwrt-rax3000me-build
> **日期**：2026-10-01

---

## 一、硬件判定（最关键的一步）

RAX3000Me 存在两种硬件版本，**刷错固件会导致四个网口全部失效**：

| 交换机 | dmesg 关键字 | 应刷的固件 |
|---|---|---|
| **MT7531AE**（老批次） | `mt7530 mdio-bus:1f` | `cmcc_rax3000m` |
| **AN8855**（新批次） | `8855` | `cmcc_rax3000me` |

判定命令（在原厂 Telnet 里执行）：

```sh
dmesg | grep -iE "7531|7530|8855|switch|mdio"
```

**本机实测输出**：

```
[   76.066962] mt7530 mdio-bus:1f lan2: Link is Up - 1Gbps/Full
```

→ 判定为 **MT7531AE**，因此系统固件用 `cmcc_rax3000m`，BL2/FIP 用 `cmcc_rax3000me-nand-ddr3`。

### 内存类型同样不能搞错

| 内存 | bootloader 后缀 |
|---|---|
| DDR3（本机） | `nand-ddr3` |
| DDR4 | `nand-ddr4` |

**刷错内存版本 = 变砖。** 官方 25.12 源码里 `cmcc_rax3000me` 已被改成 AN8855，只有 24.10 分支的 `rax3000me` 仍是 MT7531 —— 这就是本项目 bootloader 取自 24.10 的原因。

---

## 二、原厂分区表

```
mtd0  128MB   spi0.0        ← 整片 flash，禁止操作
mtd1    1MB   BL2
mtd2  512KB   u-boot-env
mtd3    2MB   Factory       ★ 无线校准 EEPROM，必须备份
mtd4    2MB   FIP
mtd5   61MB   ubi
mtd6   37MB   plugins
mtd7    8MB   fwk
mtd8    8MB   fwk2
```

⚠️ 编号不是常见的 0~3 布局，`spi0.0` 占了 mtd0，备份命令记得 +1。

---

## 三、刷机流程（已验证通过）

### 1. 备份（原厂 Telnet）

```sh
mkdir -p /tmp/bak
dd if=/dev/mtd1 of=/tmp/bak/mtd1_BL2.bin
dd if=/dev/mtd2 of=/tmp/bak/mtd2_u-boot-env.bin
dd if=/dev/mtd3 of=/tmp/bak/mtd3_Factory.bin
dd if=/dev/mtd4 of=/tmp/bak/mtd4_FIP.bin
ls -la /tmp/bak          # 应为 1MB / 512KB / 2MB / 2MB
cd /tmp/bak
tftp -p -l mtd3_Factory.bin -r mtd3_Factory.bin 你的电脑IP
```

### 2. 写入 bootloader

```sh
mtd write /tmp/preloader.bin BL2      # 可能被拒绝，见踩坑 1
mtd write /tmp/fip.bin FIP            # 这个必须成功
```

### 3. TFTP 起临时系统

- 电脑有线改静态：`192.168.1.254` / `255.255.255.0` / 网关 `192.168.1.1`
- tftpd64 目录放 `immortalwrt-mediatek-filogic-cmcc_rax3000me-initramfs-recovery.itb`
- **断开 WiFi**，网线插 LAN 口
- 断电 → 按住 Reset → 通电 → 15 秒后松手
- 传完自动进系统，浏览器开 `http://192.168.1.1`

### 4. 刷正式固件

系统 → 备份/升级 → 上传 `immortalwrt-mediatek-filogic-cmcc_rax3000m-squashfs-sysupgrade.itb`

---

## 四、当前固件功能清单

### 无线与网络

| 功能 | 说明 |
|---|---|
| 双频 WiFi | 2.4G + 5G，5G 已开 160MHz（2401 Mbps） |
| 发射功率 | 23 dBm（2.4G）/ 24 dBm（5G），高于开源的 20 dBm 上限 |
| 硬件加速 | HNAT（双 PPE）+ WARP 无线加速 + 全锥形 NAT |
| TurboACC | 流量卸载、DNS 缓存开关 |
| EQoS | MTK 硬件限速，可按设备限速 |
| SQM + CAKE | 抗 bufferbloat |
| UPnP / DDNS / WOL | 端口映射、动态域名、网络唤醒 |
| udpxy | IPTV 组播转单播 |

### 存储与电视播放

| 功能 | 说明 |
|---|---|
| SMB 共享 | ksmbd（内核态），电视/电脑直接访问 |
| DLNA | minidlna，电视直接浏览播放 |
| 磁盘管理 | diskman：分区、格式化、挂载 |
| 文件系统 | ext4 / NTFS3 / exFAT / F2FS / Btrfs / VFAT 通吃 |
| USB3.0 + UASP | 高速传输，插盘自动挂载 |
| wsdd2 | Windows 网上邻居发现 |
| smartmontools | 硬盘健康检测 |

### 服务与监控

- Adblock 全网去广告
- TTyd 网页终端
- Watchcat 看门狗 / 定时重启
- collectd + rrdtool 实时图表
- nlbwmon 每设备流量统计
- 首页显示 CPU / WiFi 温度、负载、内存

### 系统

- apk 软件包管理（在线装插件）
- 硬件加密加速（safexcel）
- zram 内存压缩、BBR 拥塞控制
- 中文界面 + Argon 主题

### 下一版新增（正在编译）

| 功能 | 用途 |
|---|---|
| hd-idle | 硬盘空闲自动休眠 |
| aria2 + ariang | 磁力/BT/HTTP 下载机，AriaNg 网页界面 |
| tailscale | 异地访问家里，无需公网 IP |
| smartdns | 更快更准的 DNS 解析，游戏/下载均受益 |
| filebrowser | 网页文件管理器，管理下载目录 |

---

## 五、踩坑记录（真实遇到过）

### 1. BL2 分区写不进去

```
mtd write /tmp/preloader.bin BL2
Could not open mtd device: BL2
```

**原因**：原厂把 BL2 设为只读保护。**解决**：不用刷 BL2 —— 官方指引原话是"只写入 FIP 分区就能启动"，且原厂 BL2 本身就是 DDR3 的正确版本。

### 2. tftpd64 切到 Tftp Client 标签后服务端停止

**原因**：tftpd64 的服务只在对应标签激活时运行。**解决**：始终停在 Tftp Server 标签。诊断：`netstat -an -p udp | grep ":69"`。

### 3. TFTP 传不进去，日志全空

排查顺序：ping 192.168.1.1 通不通 → tftpd64 的 Server interfaces 是否选了 192.168.1.254 → Windows 防火墙是否放行。

### 4. uboot 请求的文件名

必须是 `immortalwrt-mediatek-filogic-cmcc_rax3000me-initramfs-recovery.itb`（去掉版本号）。名字用 rax3000me（uboot 硬编码请求），内容用 rax3000m 的镜像（MT7531 设备树），这样刷正式固件时不用加 `-F`。

### 5. 分区编号不是 0~3

原厂第一个是 `spi0.0`，所以 BL2 是 mtd1、Factory 是 mtd3。操作前先 `cat /proc/mtd`。

### 6. GitHub Actions 的 YAML heredoc 陷阱

```
Invalid workflow file: build.yml#L45 - You have an error in your yaml syntax
```

**原因**：`run: |` 块里写 heredoc，YAML 要求缩进一致而 shell 要求 EOF 顶格，冲突。
**解决**：配置单独放 `extra.config`，workflow 里只写 `cat ../extra.config >> .config`。

### 7. 官方 Release 里没有 RAX3000Me 固件

tfnhui 的 Release 只有 `cmcc_rax3000m`（且 bootloader 是 DDR4 版，本机不能用），必须自己编译。

### 8. luci-app-tailscale 源里已移除

第一次编译没编进去才发现的。加包前先验证：
`immortalwrt/packages/<分类>/<包名>/Makefile` 和 `immortalwrt/luci/applications/luci-app-<名>/Makefile`。

### 9. GitHub 连接器权限不足（403）

**解决**：GitHub → Settings → Applications → Installed GitHub Apps → codebuddy-connector → Configure → 权限改 Read and write。

### 10. apk 没有 list-installed 命令

用原生语法 `apk info`（`apk info | grep luci-app`）。

### 11. LuCI 菜单里"网络共享"不在网络下

入口是 **NAS → 网络共享**；无线配置在 **网络 → 无线**（闭源驱动已接入标准无线页面）。

### 12. 编译设备太多会超时

上游 `mt7981-ax3000.config` 默认开 80+ 设备，必超 Actions 的 6 小时上限。用 sed 只留需要的两台。

---

## 六、常用命令

```sh
# 查看已装包
apk info | grep luci-app

# 装/卸软件
apk update && apk add 包名
apk del 包名

# 硬件加速状态
lsmod | grep -E "warp|mt_wifi|hnat"
dmesg | grep -iE "warp|wed|hnat" | head

# 备份当前配置（刷机前必做）
sysupgrade -b /tmp/backup.tar.gz

# 查看硬盘
lsblk
cat /proc/mounts | grep sd
```

---

## 七、救砖方案

TTL（CH340）接 GND / RX / TX，用 `mt7981-ram-ddr3-bl2.bin` + mtk_uartboot 内存启动后重刷。
原厂备份（BL2 / u-boot-env / Factory / FIP）保存在本机 backup 目录。

---

## 八、上游项目与致谢

本固件不是独立作品，完全站在这些开源项目之上。

### 核心源码

| 项目 | 说明 |
|---|---|
| [chasey-dev/immortalwrt-mt798x-rebase](https://github.com/chasey-dev/immortalwrt-mt798x-rebase) | **核心**：ImmortalWrt 25.12 + MTK OpenWrt Feeds 补丁，提供闭源无线驱动 `mt_wifi`（SDK 7.6.7.3）与硬件加速 |
| [tfnhui/immortalwrt-mt798x-25.12](https://github.com/tfnhui/immortalwrt-mt798x-25.12) | 上游自动同步镜像，本仓库编译时拉取此源 |
| [immortalwrt/immortalwrt](https://github.com/immortalwrt/immortalwrt) | ImmortalWrt 发行版 |
| [openwrt/openwrt](https://github.com/openwrt/openwrt) | OpenWrt 上游 |
| [immortalwrt/packages](https://github.com/immortalwrt/packages) | 软件包源（aria2、hd-idle、tailscale、smartdns、filebrowser、transmission 等） |
| [immortalwrt/luci](https://github.com/immortalwrt/luci) | LuCI 界面源（luci-app-* 各插件） |

### Bootloader 来源

| 文件 | 来源 |
|---|---|
| `cmcc_rax3000me-nand-ddr3-preloader.bin` / `-bl31-uboot.fip` | [ImmortalWrt 24.10-SNAPSHOT](https://downloads.immortalwrt.org/releases/24.10-SNAPSHOT/targets/mediatek/filogic/)（该分支的 rax3000me 仍是 MT7531，匹配本机硬件） |
| `mt7981-ram-ddr3-bl2.bin`（救砖用） | 同上，配合 TTL + mtk_uartboot 使用 |

### 主要插件上游

| 插件 | 上游项目 |
|---|---|
| aria2 | https://github.com/aria2/aria2 |
| AriaNg | https://github.com/mayswind/AriaNg |
| hd-idle | https://sourceforge.net/projects/hd-idle/ |
| tailscale | https://github.com/tailscale/tailscale |
| smartdns | https://github.com/pymumu/smartdns |
| filebrowser | https://github.com/filebrowser/filebrowser |
| ksmbd | https://github.com/cifsd-team/ksmbd |
| minidlna | https://sourceforge.net/projects/minidlna/ |
| mt_wifi / warp / hnat（闭源） | 联发科 MTK OpenWrt Feeds |

### 参考过的资料

- [OpenWrt TOH - CMCC RAX3000M](https://openwrt.org/toh/cmcc/rax3000m) — 硬件版本与刷机指引
- [hanwckf/bl-mt798x](https://github.com/hanwckf/bl-mt798x) — MT798x uboot（本项目未直接使用，作参考）
