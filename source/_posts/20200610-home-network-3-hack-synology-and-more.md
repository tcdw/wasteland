---
title: '家庭网络管理(3): UNRAID 安装黑群晖和其他常用软件'
date: 2020-06-12 09:28:27
categories:
- [Network]
- [NAS]
tags:
- NAS
---

# NAS 该用来做什么

我打算将 NAS 定位为一个家庭用的小型多媒体平台，并作为软路由处理流量。因此， NAS 应该要能满足我挂 BT 下载的需求，并且还要能够直接读取 BT 下载的内容，以家庭影院的形式展现。经过筛选，我选择安装以下 Docker 软件

- Transmission （BT 下载，Web UI 控制）
- Plex Media Server （家庭影院）
- Pi-Hole （DNS 广告过滤和自定义 Host）
- V2Ray （公网访问内网，等于 VPN）
- NginX （统一反代 Web UI，管理端口暴露规则）

以及，感谢 UNRAID 对 kvm 的良好支持，我还安装了

- 黑群晖

# 黑群晖

群晖是一个卖软件的 NAS 硬件商，其软件十分好用，新手友好。自然，我先选择安装破解的群晖系统。实际上，装完群晖后，我几乎没有使用里面的功能，因为我还是比较倾向于用 UNRAID 自带的 Docker 完成我需要的服务。不过我在研究黑群晖安装的过程中，学习了一些经验，这里做一下记录。

## 安装准备

{% asset_img hacksyno/xpenology-download.png XPENOLOGY 黑群晖引导 %}

安装黑群晖的原理是安装一个第三方的、破解后的引导，来欺骗群晖系统镜像正常工作。因此我们需要下载破解的引导作为启动盘，然后去官方网站上下载安装镜像就可以了。破解的引导我推荐在 https://xpenology.club/downloads/ 下载。网站中 DSM 6.2指的是支持 `DSM 6.2` 的系统，引导的文件名最后一串为群晖的机型。在安装时注意下载对应系统和机型的镜像。可以在 https://archive.synology.com/download/DSM/ 下载。

由于我的虚拟机无法正常启动 `Jun’s Loader v1.04b DS918+` ，这里以 `Jun’s Loaders DSM 6.2` - `Jun’s Loader v1.03b DS3617xs` 为例。下载对应引导并解压，将引导拷贝至 NAS。我创建了一个共享文件夹 HackSynology 用于放置群晖相关文件。

<!--more-->

{% asset_img hacksyno/creat-share.png XPENOLOGY 创建共享目录 %}

{% asset_img hacksyno/xpenology-boot.png XPENOLOGY 黑群晖引导文件 %}

然后到群晖官网，下载对应的系统镜像。在此处样例中，我下载了 `DSM_DS3617xs_23739.pat` 文件。将这两个文件准备好，就可以开始安装黑群晖了。

{% asset_img hacksyno/dsm-download.png 群晖官方系统镜像 %}

## 创建虚拟机

{% asset_img hacksyno/vm-and-bios.png 创建 Linux 虚拟机，设置 BIOS %}

创建一个 Linux 虚拟机，命名，选择直通的 CPU 核心。UNRAID 的这点我很喜欢，它能够手动指定虚拟机使用其中的某几个核心，避免虚拟机大量占用资源。这里我选择了 CPU 1 作为黑群晖的核心。RAM 1-2GB 就够了。由于兼容性问题，此处需要选择 `i440fx-4.2` 和 `Sea-BIOS` 。

{% asset_img vm-disk.png 设置硬盘和启动引导分区 %}

启动引导和挂载硬盘如图所示设置。其中挂载硬盘的大小可以按照个人喜好修改，总线填写 `SATA`，类型为 `qcow2` ，这样动态分配空间，不需要创建硬盘时就占用对应容量。主硬盘指向下载的破解启动引导，总线类型为 USB。

{% asset_img vm-mac-and-start.png 关闭创建后启动 %}

网卡随意，手动关闭最下方的 `Start VM atfer creation` 并保存。

{% asset_img vm-net-model-type.png 找到虚拟网卡 %}

{% asset_img vm-net-model-type-changed.png 修改虚拟网卡为 e1000 %}

重新进入编辑页面，点击右上方的 `FORM VIEW` ，转为 `XML VIEW` ，搜索 `<model type=` ，将虚拟网卡型号修改为 `e1000` 或 `vmxnet3` （理论性能更佳，未验证）。

{% asset_img vm-start.png 启动虚拟机 %}

此时可以保存，启动机器。

## 安装

{% asset_img synology-boot-normally.png 虚拟机正常启动 %}

如果一切正常，VNC 中可以看到上方的情况。此时可以打开 http://find.synology.com/ ，搜索内网的群晖新机器地址。如果找不到，可能需要从 https://www.synology.com/en-global/support/download 下载 Synology Assistant 进一步寻找。

{% asset_img lan-synology-device-found.png 找到内网群晖 %}

{% asset_img manual-install.png 手动选择安装镜像 %}

正常进行安装流程。当需要安装系统镜像时，选择手动安装，上传从官网下载的系统镜像。一定要保证版本和机型对应。如果无法安装系统镜像，首先检查版本是否正确，如果都没问题，可以考虑更换破解引导的版本。我一开始测试的是 `DS918+` 机型的破解引导，后来发现不能工作，于是更换了现在的 `DS3617xs` 。

{% asset_img restart.png 完成安装，重启 %}

安装完成后，如果 Web 界面显示重启，黑群晖就成功以虚拟机的形式被安装了。之后进入群晖系统，进行初期配置。群晖的交互足够用户友好，不存在什么困难，这里就不再赘述。

{% asset_img success.png %}

# 其他 Docker 应用

## Transmission (with web UI)

{% asset_img transmission-install.png Transmission 安装 %}

Transmission 的安装很容易。进入 Apps 搜索 Transmission，找到作者是 linuxserver 的，下载即可。我简单提供一份配置样本。

{% asset_img transmission-settings.png Transmission 配置 %}

9091 端口是 Transmission 的 Web UI，/downloads 是默认下载文件夹，其他按照默认配置就好。值得一提的是，我额外增加了一个参数 `TRANSMISSION_WEB_HOME` ，设置为 `/transmission-web-control/` ，这是一个自带的 Web 界面美化插件，类似一个主题，添加 Variable 变量即可。

{% asset_img transmission-web.png Transmission Web %}

安装完成后，可以从 Docker - Transmission 进入 Web UI，添加下载或是管理做种了。

## PLEX

### 主程序安装

{% asset_img plex-install.png PLEX 安装 %}

PLEX 在 Apps 中自带官方打包好的版本，直接安装即可。

{% asset_img plex-settings.png PLEX 配置 %}

设置可以参考上图。其中 `Key 1: PLEX_CLAIM` 需要在官网 https://www.plex.tv/ 注册后，进入 https://plex.tv/claim 申请。PLEX Claim ID 有效期很短，申请后建议立刻启动服务器，进入 Web UI 界面完成绑定流程。失效了也没有关系，重新进入 claim 页面申请、重新填写 Docker 设置中的变量就可以了。

值得一提的是，为了将 Transmission 下载的内容直接导入 PLEX ，我将 Transmission 下载文件夹对应的宿主机文件夹同时挂载在 PLEX 对应的数据文件夹下。可以参考设置中最后一项 `Transmission Share:` 。这样一来，当 Transmission 完成下载，就不需要手动将媒体文件移动至 PLEX 数据文件夹中。

### PLEX AniDB 插件

PLEX 自带的刮削器大多是欧美电影 / 电视节目的资料，较少包含动画的信息。这个问题可以通过安装第三方插件解决。这里推荐 Hama.bundle，安装方式参考 README - Installation (https://github.com/ZeroQI/Hama.bundle#installation)，这里不赘述。

> * "Scanners"         "Scanners/Series" folder needs creating. Absolute series Scanner.py" goes inside. 
>			 https://raw.githubusercontent.com/ZeroQI/Absolute-Series-Scanner/master/Scanners/Series/Absolute%20Series%20Scanner.py
> * "Plug-ins"         https://github.com/ZeroQI/Hama.bundle > "Clone or download > Download Zip. Copy Hama.bundle-master.zip\Hama.bundle-master in plug-ins folders but rename to "Hama.bundle" (remove -master) 

- https://github.com/ZeroQI/Hama.bundle
- https://github.com/ZeroQI/Absolute-Series-Scanner/

## Pi-Hole

{% asset_img pihole-install.png Pi-Hole 安装 %}

这是一个通过污染 DNS 解析来封锁广告的工具，也可以直接在官方的 Apps 中下载到。配置流程按照官方提供的模板微调配置即可。建议通过网桥接入，自定义一个 IP 地址。注意 Docker 设置的 IP 地址要和模板后半部分设置的 ServerIP 相同。面板的密码 `WEBPASSWORD` 可以随便写一个，在程序运行后，进入 Docker 命令行，运行命令 `pi-hole password` 重置密码。

在后期修改配置的过程中，我改动了 Pi-Hole 的 IP 地址导致面板无法正常访问。此时也可以通过进入 Docker 的 Pi-Hole 命令行，运行 `sudo pihole -a -p` 重新配置程序。

{% asset_img pihole-web.png Pi-Hole Web 界面 %}

如果有想使用域名访问 Pi-Hole 的需求，可以参考这篇文章。我还没有测试，不过看起来 it works。

- https://discourse.pi-hole.net/t/how-to-enable-remote-internet-access-for-pi-hole-web-dashboard/7025

## 其他

关于 Nginx 和 V2Ray 的安装，主要是用于改善我个人体验和远程访问内网需求的。这些软件安装和其他平台无异。关于用法和安装过程，我会在之后的文章介绍。

# Reference

- https://www.reddit.com/r/unRAID/comments/7sf0hg/use_your_unraid_box_as_an_ad_blocking_dns_server/
- https://forums.plex.tv/t/how-do-you-install-the-official-plex-for-unraid/206596
- https://support.plex.tv/articles/200264746-quick-start-step-by-step-guides/
- https://support.plex.tv/articles/201373823-nas-devices-and-limitations/
- https://handlers.sans.edu/gbruneau/pihole.htm
