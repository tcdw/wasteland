---
title: 在树莓派上运行 OpenWrt 作为软路由
date: 2020-07-19 13:10:55
categories:
- [Network, Router]
- [Raspi]
tags:
- Raspi
- Router
---

# 前言

之前我用 NAS 虚拟化了一台 OpenWrt 用作软路由。随着使用频率的增加，我越发离不开它带来的优秀的上网体验。但是用虚拟化的软路由总觉得有些不舒适，比如万一 NAS 掉线（虽然暂时没出现过），内网就会瞬间瘫痪。同时，用这种方法虚拟出来的软路由和小米路由自带的 Mesh 功能有冲突，小米路由无法获取 Mesh 节点的拓扑图，也没有办法进行完整的设备管理（识别不出设备）。

解决方案是在小米路由器的上级部署软路由，将原本放在小米路由器内网的软路由移动到光猫内网。我先用树莓派 4 作为软路由进行测试，另一块 1037U 主板在路上。光猫只有一个千兆网口，但是恰好前些时间我买了台网件的 GS105 ，得以在光猫的内网中接入更多千兆设备。

{% asset_img 1.netgear-gs105.jpg 网件 GS105 交换机 %}

这是一个五口非网管型交换机，自适应 10/100/1000M，带指示灯，发热量不大，价格百元内。这样一来，网络拓扑就可以进行下面的改动

{% asset_img 2.change-of-topology.jpg 网络拓扑变化 %}

将内网的 DHCP 分发任务交给小米路由，允许小米路由 app 完全发挥功能，带来更严格的 WiFi 接入控制和 Mesh 组网体验。另一方面，改变静态 IP 配置将小米路由的网关设置为软路由地址，让其流量通过作为软单臂路由的树莓派，达到科学上网等目的。

# 树莓派组装

<!--more-->

树莓派和外壳都是我从咸鱼购入的。我前期测试过树莓派 3B 的性能，的确不太行，因此这次买了树莓派 4 做软路由，外加一个红色的外壳、一个噪音超大的蓝光散热风扇和几块散热片，总价不到 230。

{% asset_img 3.pi.jpg 树莓派和配件 %}

先上散热片。考虑后期可能会更换其他散热，我没有直接把散热片粘在机器上，而是先粘上一层导热垫，用导热垫自身的黏性固定散热片。其中两块大的散热垫分别对应 CPU 和内存，还有一块不知道贴哪，就随便贴了个图一乐。

然后将外壳风扇的 5V 供电安装在树莓派 5V 针脚和接地上。根据 https://www.raspberrypi.org/documentation/usage/gpio/ 的定义，将红色和黑色两条线分别接在 4、6 号针脚即可。

{% asset_img 4.pi-assembling.jpg Pi 组装 %}

实际使用中，风扇转速太大，噪音感人，光害也很令人头疼。我打算目前不接这个风扇。如果未来热量堆积严重，我会考虑更换其他的静音风扇。

# OpenWrt 安装和配置

## 烧录 OpenWrt 系统

从 OpenWrt 的官网可以检查当前 OpenWrt 在树莓派各版本系统镜像的发行情况。在树莓派 4 上，官方目前没有正式发行的 Release 版本，只有 Snapshot 版本的 OpenWrt 镜像。Snapshot 版本的镜像不带 LuCi 界面，因此，在安装过程中可能会遇到一些麻烦。

{% asset_img 5.openwrt-pi-versions.jpg OpenWrt %}

> The Raspberry Pi is supported in the brcm2708 target.
> Subtargets are bcm2708 for Raspberry Pi 1, bcm2709 for the Raspberry Pi 2, bcm2710 for the Raspberry Pi 3, bcm2711 for the Raspberry Pi 4.

根据官方说明，树莓派 4 选择的对应 `subtarget` 为 `bcm2711`，因此我们可以顺藤摸瓜到 https://downloads.openwrt.org/snapshots/targets/bcm27xx/bcm2711/ 下载对应的系统镜像 `rpi-4-ext4-factory.img.gz`。如果网不好，也可以到我们的老朋友 TUNA 镜像的另一个分身 - 北外开源软件镜像站 https://mirrors.bfsu.edu.cn/openwrt/snapshots/targets/bcm27xx/bcm2711/ 下载。为什么不直接从 TUNA 下载呢？因为截止目前，所有对 https://mirrors.tuna.tsinghua.edu.cn/openwrt/snapshots/ 的访问都会被重定向至 https://mirrors.bfsu.edu.cn/openwrt/snapshots/ 。

插上读卡器，打开 `Raspberry Pi Imager` ，烧录系统。

{% asset_img 6.pi-imager Pi Imager %}

## OpenWrt 网络初始配置

把 SD 卡插进树莓派，通电开机。第一次开机时，先将电脑直接和树莓派的网口相连。因为 OpenWrt 默认 IP 为 192.168.1.1 并默认开启 DHCP ，只要将电脑接入，IP 就会自动分配。为了避免可能的路由冲突，可以禁用其他网卡。

{% asset_img 7.first-connect.jpg %}

由于树莓派是单网口的，如果这个网口用于连接 PC，就没有办法上网了。此时需要通过 SSH 连接树莓派进行网络配置。Snapshot 的 OpenWrt 系统默认无密码，在使用 SSH 客户端连接时，需要选择 `none` 的连接方式。

{% asset_img 8.first-ssh.png 首次连接 SSH %}

先运行命令 `passwd` 设置 root 密码。密码设置完成后，断开并重连 SSH 测试密码设置是否正确。

然后修改网络配置。我准备接入网段为 192.168.1.1/24 的局域网，网关是 192.168.1.1，为了避免 IP 冲突，让树莓派可以上网，需要修改网络配置。

```bash
vi /etc/config/network
```

可以看到默认的 `lan` 网络设置为

```
config interface 'lan'
        option type 'bridge'
        option ifname 'eth0'
        option proto 'static'
        option ipaddr '192.168.1.1'
        option netmask '255.255.255.0'
        option ip6assign '60'
```

根据需求，修改 IP 地址，增加网关地址和 DNS

```
config interface 'lan'
        option type 'bridge'
        option ifname 'eth0'
        option proto 'static'
        option ipaddr '192.168.1.2'
        option netmask '255.255.255.0'
        option ip6assign '60'
        option gateway '192.168.1.1'
        option dns '119.29.29.29'
```

最后运行

```bash
service network restart
```

此时控制台会显示 `'radio0' is disabled` 然后无响应，因为网络接口发生了变化。

将树莓派接回交换机，SSH 连接刚才设置的 IP 地址。如果设置无误，此时可以连上树莓派。初次连接的网络配置完成。

## 安装 LuCi

替换 OpenWrt 软件源为北外源

```bash
sed -i 's_downloads.openwrt.org_mirrors.bfsu.edu.cn/openwrt_' /etc/opkg/distfeeds.conf
```

```bash
opkg update
opkg install luci
```

如有需要，可以安装语言包

```bash
opkg install luci-i18n-base-zh-cn
```

直接访问设置好的 IP 地址，就能在图形界面正常使用 OpenWrt 了。

{% asset_img 9.raspi4-luci.jpg %}

# 性能测试

为了让树莓派能够胜任原有 NAS 上虚拟化的软路由的工作，我对树莓派 + Clash 的性能做了一次测试。目标代理用的是 V2 + TLS + WS。

{% asset_img 10.compare.jpg %}

在不走代理的情况下，树莓派的网络吞吐和 NAS 相比在误差范围内。 在代理的情况下，由于 ARM 的性能和 x86 的 NAS 有一定差距，NAS 基本跑满了代理的带宽，而树莓派只有大约 2/3 的性能，日常使用差别不会太明显。

# References

- https://openwrt.org/toh/raspberry_pi_foundation/raspberry_pi
- https://www.raspberrypi.org/documentation/usage/gpio/
- https://openwrt.org/docs/guide-user/luci/luci.essentials
- https://openwrt.org/zh-cn/doc/uci/network
- https://mirrors.bfsu.edu.cn/help/openwrt/
- https://openwrt.org/packages/pkgdata/luci-i18n-base-lang
- https://openwrt.org/docs/guide-user/services/nas/sftp.server
