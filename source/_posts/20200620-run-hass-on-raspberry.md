---
title: 在树莓派上运行 Home Assistant，支持米家等 IoT 设备接入 HomeKit
date: 2020-06-20 15:22:08
categories:
tags:
---

# 前言

## 捡了块树莓派

临时被叫回学校办毕业手续，想起当年（五年前）我在淘宝买了块树莓派 2B，给家里做了个通过 HTTP 代理自动分流国内外流量的工具。那块树莓派被我带到了学校，放着吃灰三年多了。这次回学校，就顺手把这块树莓派带了回来。

最近优化了家里的网络，大学四年也断断续续买了不少米家的 IoT 设备。米家设备能够原生支持 HomeKit 也应该只是最近的事，我手上只有一个台灯、三个智能插座能够加入 HomeKit 管理。因此，我打算把这台树莓派挂在 IoT 设备专用中继路由器下，作为一个中介，将其他无法原生支持 HomeKit 的米家设备接入 HomeKit，实现统一管理。

为什么我不直接用米家做管理呢？一是米家的操作逻辑比较恼人；二是米家无法长期运行于后台，一些和离家、回家相关的自动化逻辑无法很好的运行。接入 HomeKit 后，这种情况会得到一定的改善。

## 桥接平台的选择

通过资料调查和实际测试，我最终选择了 Home Assistant 作为接入智能家居设备的平台。同类解决方案还有很多，比如同样提供智能家居接入方案的 Home Bridge，ioBroker 等。我同时尝试了 Home Bridge，但是由于米家相关插件文档模糊，接入后无法正常连接挂载在网关的设备，最终作罢。

其实 Home Assistant 的安装也并不容易。安装过程存在各种各样的问题和 bug，我尝试了非常多种方案才成功运行。

# 树莓派的初步配置

## SD 卡安装树莓派系统

我掏出了之前留在机器上的传家宝 TF 卡 三星 EVO+ 32G，不料卡在经过几次刷写后，自动写保护了。于是我又从 NS 里掏出了从 tb 20 RMB 买的朗科 32G TF 卡。现在 TF 卡的价格真的越来越便宜了，我打算屯一堆，免得弄坏了没有替换卡。

<!--more-->

{% asset_img 1.disk-management-on-windows.png Windows Disk Management %}

打开 Windows 自带的 Disk Management，将原有 SD 卡分区删除，创建一个大的 FAT32 分区。这一步也可以用树莓派官方自带的 `Raspberry Pi Imager` 格盘。这个软件可以在 https://www.raspberrypi.org/downloads/ 下载到，推荐用这个软件烧写系统盘。

{% asset_img 2.raspi-imager.png Raspberry Pi Imager %}

由于是树莓派 2B，性能一般，所以需要尽可能地压榨硬件资源。在系统选择页面 https://www.raspberrypi.org/downloads/raspberry-pi-os/ 中，我选择了不带图形界面的 `Raspberry Pi OS (32-bit) Lite`。下载并解压，然后用官方烧写工具烧写系统镜像。这张 TF 卡的速度意外的很快，在 USB 3.0 读卡器下，大约有80-90MB/S 的写入速度。

{% asset_img 3.raspi-imager-write.png Raspberry Pi Imager Write OS %}

{% asset_img 4.raspi-imager-write-successful.png Raspberry Pi Imager Write Successful %}

现在的树莓派官方系统是默认不开启 SSH 功能的。开启 SSH 功能的方法除了通过外接显示器在命令行输入 `sudo raspi-config` 外，也可以直接在烧写后的 TF 卡根目录下创建命名为 `SSH` 的空文件，以激活 SSH 功能。烧写完成后，由于软件会自动弹出对应设备，需要重新插拔一下读卡器，并创建对应空文件。

我预先在 DHCP 服务器上设置了分配的静态 IP。如果不清楚树莓派被分配到的 IP，可以在路由器中 DHCP 租约下找到被分配的 IP 地址。建议要在路由器上做好 IP 分配。

## 镜像源

官方系统默认的账户和密码是 pi / raspberry。登入 SSH 后，更换清华的镜像以提高 apt 更新速度。以 Debian 10（buster）为例

> # 编辑 `/etc/apt/sources.list` 文件，删除原文件所有内容，用以下内容取代：
> deb http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib rpi
> deb-src http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib rpi
> 
> # 编辑 `/etc/apt/sources.list.d/raspi.list` 文件，删除原文件所有内容，用以下内容取代：
> deb http://mirrors.tuna.tsinghua.edu.cn/raspberrypi/ buster main ui

```bash
sudo apt update
sudo apt upgrade
```

# 安装 Home Assistant

## Home Assistant，Hass 和 Hassbian 的关系

这三者之间的关系把我绕进去了，我也是在经过大量搜索才明白了他们到底是什么。这个统一管理的程序叫 **Home Assistant**，它曾经叫 **Hass.io**。 **Hassbian** 是一个由官方封装好的，可以开箱即用运行 Home Assistant 的系统，且仅能运行 Home Assistant 系统，命名应该是参考了 Raspbian ，并且是基于其构建的。

## 安装依赖

因为我们使用的是基于 Python3 运行的 Home Assistant，所以要安装 Py3 相关的依赖。同时，为了避免对运行的其他程序的影响，更方便维护，使用 python venv 作为虚拟环境运行程序。由于我在上海，所以我更换了交大的 pypi 镜像。也可以考虑清华镜像或科大镜像。

```bash
sudo apt install -y python3 python3-dev python3-venv python3-pip libffi-dev libssl-dev autoconf
sudo apt install -y libavahi-compat-libdnssd-dev
sudo pip3 install -i https://mirrors.sjtug.sjtu.edu.cn/pypi/web/simple pip -U
sudo pip3 config set global.index-url https://mirrors.sjtug.sjtu.edu.cn/pypi/web/simple

# Tsinghua tuna pypi
# sudo pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple pip -U
# sudo pip3 config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

## 启动虚拟环境

创建 homeassistant 用户并配置 python 虚拟环境用于运行程序

```bash
sudo useradd -rm homeassistant

cd /srv
sudo mkdir homeassistant
sudo chown homeassistant:homeassistant homeassistant

sudo su -s /bin/bash homeassistant
cd /srv/homeassistant
python3 -m venv .
source bin/activate
```

后期重新进入虚拟环境的命令是

```bash
sudo -u homeassistant -H -s
source /srv/homeassistant/bin/activate
```

可以将 `sudo -u homeassistant -H -s && cd ~` 写入 `/home/homeassistant/.bashrc` 尾部，这样当切换到 homeassistant 用户时，就会默认激活虚拟环境。

## 安装并运行 Home Assistant

由于程序 BUG，在安装最新版本 Home Assistant 时，不会自动下载所有依赖。经过各种资料查找和尝试，我确认了可以先安装 0.100.3 版本并运行，再安装最新版本程序。

在控制台运行

```bash
pip3 install homeassistant==0.100.3
hass
```

等待 Home Assistant 安装所需的依赖并启动服务器。当控制台提示

```bash
2020-06-20 18:27:32 INFO (MainThread) [homeassistant.core] Timer:starting
```

时，代表程序已经可以正常运行了。此时 Ctrl + C 结束进程，然后安装最新版的 Home Assistant。

```bash
pip3 install homeassistant -U
hass
```

继续等待一段时间的依赖安装，然后进入 http://IP:8123 配置 Home Assistant。

{% asset_img 5.install-finished.png Home Assistant 安装完成 %}

# 配置 Home Assistant

## 接入米家

小米在米家 app 开放了部分绿米 IoT 设备的管理接口，允许第三方管理程序管理 IoT 智能设备。但是大多数米家设备并没有在明面上开放管理密钥。但是实际情况中，所有通过蓝牙或 Zigbee 协议连接的智能设备都需要配置网关，网关需要接入 WiFi，算上原本就通过 WiFi 接入的设备，所有米家设备都可以从内网进行访问。米家 app 也是通过内网 WiFi 进行控制的

感谢开源社区和开发者的工作，封装好了各种插件，使我们能够直接配置获取到的 token 和设备地址，将 IoT 设备接入 Home Assistant。设备的接入方式可以在 https://www.home-assistant.io/integrations/#search/xiaomi 页面找到，配置文件样例也包含在页面中。可以通过 Android 系统下 `米家 5.4.54` 版本程序泄露的日志获取 token ，参考 https://zhuanlan.zhihu.com/p/62681849

{% asset_img 6.hass-sample.png Home Assistant 界面 %}

根据各处的教程，我将我的设备接入了 Home Assistant。上面的截图就是界面平时的样子，可以手动更改界面排序。我打算在之后用一块独立的电子墨水屏显示状态。由于我刚组了 Mesh，重命名了 WiFi ，所以以前的部分设备掉线了，图中没有相关的设备信息

支持的米家设备可以在 https://www.jianguoyun.com/p/DbzdYzoQp5HMBhjZ4IkB 查询到，可以说基本覆盖全了

## 桥接至 Apple HomeKit

桥接至 HomeKit 的方法也很简单。在 /home/homeassistant/.homeassistant 下的 `configuration.yaml` 中新增一行

```yaml
homekit:
```

然后重启程序即可。重启后就可以在左下角的通知处看到接入 HomeKit 的消息。至于自动化、联动等操作，我打算全部通过 HomeKit 和米家管理。因此，这部分配置我没有继续研究下去

{% asset_img 7.homekit-code.png 接入 HomeKit %}

# References 

- https://blog.csdn.net/fanmengmeng1/article/details/46366695
- https://mirrors.tuna.tsinghua.edu.cn/help/raspbian/
- https://www.home-assistant.io/docs/installation/raspberry-pi/
- https://home-assistant.cc/installation/general/
- https://github.com/home-assistant/core/issues/28361
- https://www.jianguoyun.com/p/DbzdYzoQp5HMBhjZ4IkB
- https://zhuanlan.zhihu.com/p/62681849
