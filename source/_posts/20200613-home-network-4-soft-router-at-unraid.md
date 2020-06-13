---
title: '家庭网络管理(4): 单网口安装 OpenWRT 实现单臂软路由'
date: 2020-06-13 16:52:38
categories:
- [Network, Router]
- [NAS]
tags:
- NAS
- Router
---

# 软路由

为了进一步发挥 NAS 的作用，我决定在它上面部署一个软路由。因为软路由可以更好地管理流量，部署科学上网，特别是在主路由小米 AC2100 `MT7621` 方案性能有限的情况下，x86 架构的软路由能够进行的加解密运算能力远远超过主路由。我的 NAS 只有板载千兆单网口，实现起来没有多网口路由器那么优雅，好在还是能够用单臂路由的方法实现软路由。

# 网络拓扑规划

## 设备一览

在规划网络之前，需要先清点一下我有哪些设备，这些设备分别有哪些上网的需求。

路由器，我有

- 一台联通的光猫
- 一台小米 AC2100
- 一台小米 R3
- 一台老旧的小米 3C（大概就不用了）

终端设备我有

- 若干 PC
- 若干手机

IoT 设备我有

- 遍布房间的传感器
- 智能插座
- 开关

因此，我打算将光猫作为拨号终端；小米 AC2100 作为主路由，保留小米原有系统（后来我把它刷成 OpenWRT 了）；小米 R3 无线桥接 AC2100，作为 IoT 的接入点（因为位置的原因，AC 2100 摆放在光猫边上，2.4G 没有办法很好地覆盖整个屋子，所以要用 R3 中继）。光猫的无线功能关闭，避免影响其他网络的速度。

## 拓扑规划

NAS 使用网线连接小米 AC2100。UNRAID 系统通过网桥（一般为 br0）让虚拟机和 Docker 上网，可以理解为一个二层交换机。我简单做个一个拓扑图，解释我打算规划的拓扑结构。

{% asset_img 单臂路由网络拓扑图.png 单臂路由网络拓扑图 %}

我的规划如图，光猫负责拨号，小米 AC2100 作为主路由处理流量，R3 通过无线中继服务 IoT 设备，NAS 虚拟化各种服务和软路由提供功能。其中，主路由的 IP 地址为 192.168.31.1，软路由的 IP 地址为 192.168.31.2。为了让所有流量在转发至公网时经过软路由处理，由 AC2100、R3 所有通过 AP 接入的设备和有线接入的设备都要以 192.168.31.2（软路由） 作为网关，而软路由要以 192.168.31.1（主路由） 作为网关。为了实现这个功能，整个 LAN 中只能由软路由一个 DHCP 服务端。即，让新加入的设备自动以软路由作为网关。

# 单臂软路由配置

## 选择系统

<!--more-->

首先下载 OpenWRT 系统安装文件。进入 https://downloads.openwrt.org/releases/ 选择对应版本和对应系统架构。由于我是 x86，所以我下载了 x86 generic 版本（openwrt-19.07.3-x86-generic-combined-ext4.img）[https://downloads.openwrt.org/releases/19.07.3/targets/x86/generic/](https://downloads.openwrt.org/releases/19.07.3/targets/x86/generic/)。将其解压，复制到 NAS 上任意位置。由于该镜像已经打包好，可以作为硬盘使用，所以不用额外预留其他空间用于存放文件。

{% asset_img openwrt-image.png OpenWRT 镜像 %}

## 部署虚拟机

{% asset_img openwrt-vm-setting.png OpenWRT VM 设置 %}

添加 VM - Linux，直通 CPU 建议全选上，毕竟处理网络流量比较耗费算力。BIOS 改为 SeaBIOS，主硬盘 Manual，指向 OpenWRT 镜像位置，其他保持默认即可。

{% asset_img openwrt-network-config.png OpenWRT 初次运行网络设置 %}

创建虚拟机并自动运行。进入 VNC，先用 `passwd` 命令修改默认密码，然后进入 `vi /etc/config/network` 修改为预定的内网 IP 地址。保存配置文件， `/etc/init.d/network restart` 重启网络配置。浏览器访问内网 IP 地址，一切顺利的话，机器就能启动了。由于我是演示，使用 192.168.31.3 作为 IP。

如果 VNC 中观察到 `Switching to clocksource tsc` 或者其他错误导致按回车也没反应，可以考虑更换其他架构的镜像。

## 初期配置

{% asset_img openwrt-interface-settings.png OpenWRT UI 界面网络设置 %}

安装完路由系统后，首先需要对系统进行一些初期配置。浏览器进入设置好的 IP，`Network` - `Interfaces` - `Edit LAN`，增加 `gateway` 为主路由（在我这是小米 AC2100）的 IP 地址，IPv4 broadcast
 保持默认，DNS 随意设置（但是一定要设置）。切换到 `DHCP Server` 选项卡，将 `Advanced Settings` 的 `Force` 勾选。

SSH 登录路由，执行 `sed -i 's_downloads.openwrt.org_mirrors.tuna.tsinghua.edu.cn/openwrt_' /etc/opkg/distfeeds.conf` 更换国内 OpenWRT 源，更新软件包 `opkg update` ，执行 ` opkg install luci-i18n-base-zh-cn` 安装中文 Web 界面。如果安装过程无法正常进行，请重新检查联网设置。

## 测试联网情况

{% asset_img local-test.png 本地联网测试 %}

为了测试软路由工作情况，我们可以手工设置本地网关，修改网关至软路由对应的 IP 地址，查看连接情况。如果一切正常，就可以关掉主路由上的 DHCP 服务，让软路由提供 DHCP 服务了。一般来说，只要初期配置中路由命令行能够正常更新软件包，这一步也不会有太多问题。

在软路由提供 DHCP 服务后，新接入的设备和需要更新 DHCP 租约的设备都会以软路由作为网关访问外网。从路径上看，就是 终端 - 主路由 - 软路由（处理流量） - 主路由 - WAN。

# 性能测试

## 延迟

我使用高精度延迟测试工具 hrping 进行测试，这个工具可以从 http://www.cfos.de/en/download/start-download.htm?file=/hrping-v507.zip 下载。我将一台电脑连接至联通光猫 192.168.1.1 ，关闭防火墙，进行长时间 ping 测试。

**软路由作为网关**

tracert

{% asset_img tracert.png Tracert 结果 %}

ping

{% asset_img ping-test-with-gateway.png 软路由作为网关 %}

**主路由作为网关**

ping

{% asset_img ping-test-without-gateway.png 主路由作为网关 %}

可以看到，使用软路由作为路由，仅增加了 0.5ms 的延迟，稳定性也没有较大的变化。相对于软路由的好处来说，这个延迟增加是可以接受的。

## 带宽和功耗

### TCP

我使用 iperf3 作为带宽测试工具，测试终端间、终端到软路由、软路由和 NAS 之间的带宽和功耗。以下简单列出一些测试结果

AC2100 的网口为千兆

#### NAS 静止功耗

29.59W

#### PC（软路由网关）- PC（光猫 LAN）

{% asset_img bandwidth-test-1.png 软路由 PC - 光猫 LAN %}

由于联通的光猫只有一个千兆口，剩下全是百兆口，所以实际测速只有百兆。此时的 NAS 功耗为 37 - 40W

#### PC（软路由网关） - 软路由

800Mb 49W

#### PC（软路由网关） - NAS

930Mb 53W

#### 软路由 - NAS （网桥内网）

6.5-7.5Gb 58W

#### 软路由 V2Ray 加密带宽和功耗

我通过 OpenClash 挂 V2Ray 的梯子，远程访问我的服务器，测试经过 V2Ray 加密之后的功耗。实际测试结果为

100Mb 37W
260Mb 43W

### TCP with V2Ray

上述 V2Ray 的测试让我突发奇想，打算测试一下 NAS CPU 在满载 V2Ray 时的负载，看看这块 CPU 对 V2Ray 流量的处理能力。为了让 iperf3 通过代理，我简单编写了一个（自攻自受）的反向代理，让 V2Ray 在此期间进行一轮加密和一轮解密。iperf3 的流量穿过一层 V2Ray 的隧道，在网桥内网中测试。

**软路由 - NAS （没有 V2Ray）**

6.5-7.5Gb 58W

{% asset_img test-without-v2ray.png 没有 V2Ray， CPU 负载 %}
{% asset_img test-without-v2ray-res.png 没有 V2Ray， 带宽 %}

**软路由 - NAS （V2Ray 一次加密，一次解密 + 反向代理）**

2.5-3Gb 58W

{% asset_img test-with-v2ray.png 没有 V2Ray， CPU 负载 %}
{% asset_img test-with-v2ray-res.png 没有 V2Ray， 带宽 %}

结果表明这块 CPU 的性能和功耗还是非常棒的，在满载的情况下可以同时处理 2.5Gb 带宽以上的 V2Ray 流量（实际情况应该和这个不同，实际情况中不需要一次加密和一次解密，但是需要考虑其他运算，例如 ws，tls），应付日常使用绰绰有余。

# 展望

一切看起来都很美好，但是也有美中不足的地方。UNRAID 运行的 Docker 无法以软路由作为网关，即 UNRAID Docker 的流量无法经过软路由。目前我并不知道为什么会这样，可能是由于启动顺序不同导致的问题。因为如果要修改 UNRAID 的默认网关，需要停止整个阵列。一旦阵列停止，Docker 和 VM（包括软路由）也需要停止。如果修改完设置，启动阵列时，Docker 会因为没有 DHCP 服务器而无法分配到 IP 地址（一般来说 VM 启动优先级更慢）。我正在继续寻找解决方案。

由于 CPU 性能足够，我打算下一步上万兆网卡和万兆交换机，让内网进一步加速。同时，我打算直接在 NAS 上安装万兆网卡。这块主板还有一个 PCIE x16 插槽，可以插一张多口万兆网卡实现软路由硬件直通，进一步提高效率。

# References

- https://openwrt.org/toh/views/toh_fwdownload
- http://www.cfos.de/en/ping/ping.htm
- https://mirrors.tuna.tsinghua.edu.cn/help/openwrt/
- https://post.smzdm.com/p/az502200/
- https://www.jianshu.com/p/da01ce070688
- https://post.smzdm.com/p/ax027g83/
- https://toutyrater.github.io/app/reverse.html
- https://iperf.fr/iperf-doc.php
- 感谢拓扑绘画工具 https://www.processon.com/ 尽管使用中还遇上很多 BUG
