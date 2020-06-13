---
title: '家庭网络管理(2): 部署和使用 UNRAID'
date: 2020-06-08 16:17:07
categories:
- [Network]
- [NAS]
tags:
- NAS
---

# 选择 UNRAID

在组装完 NAS 之后，下一步就是考虑如何将 NAS 的硬件性能高效利用。经过对群晖、UNRAID、FreeNAS 等 NAS 操作系统的调研后，我决定使用 UNRAID 作为我的 NAS OS。一是因为 UNRAID 对 Docker 和 VM 的支持较好，有庞大的社区支持；二是因为 UNRAID 的阵列功能很好用，自带的软 RAID 功能可以只用一张 Parity Check 盘完成其他所有盘的校验；三是因为 UNRAID 的 Web UI （相对）比较符合我的审美。话不多说，立刻开始破解版 UNRAID 安装。

# 安装

UNRAID 的官网是 https://unraid.net/ ，价格表在 https://unraid.net/pricing 。从价格上看，Basic 订阅只要 60 刀，还是比较合适的。由于我第一次使用这个操作系统，所以先从破解版入手。

{% asset_img unraid-pricing.png UNRAID 价格表 %}

安装 UNRAID 需要准备一个 U 盘作为启动盘和系统盘，并且之后整个系统就安装在这块 U 盘中。这个 U 盘在以后使用 UNRAID 时，需要一直插在 USB 口上。不能使用 SD 卡 + 读卡器，因为可能获取不到正确的设备 ID。 UNRAID 系统根据闪存的设备 ID 分发密钥。在安装之前，需要将闪存格式化成 FAT32 格式。如果 Windows 不支持格式化为 FAT32 ，可以使用 [Windows GUI version of _fat32format_](http://www.ridgecrop.demon.co.uk/index.htm?guiformat.htm) 将闪存格式化为 FAT32 格式。__切记，在进行格式化操作之前，务必做好数据备份。__

截至目前，最新的破解版是 `UNRAID 6.8.2` ，下载地址在 http://www.hopol.cn/2020/06/1675/ 。这个安装包自带密钥计算工具，可以使用 https://koolshare.cn/thread-181253-1-1.html 提供的一键安装工具进行安装；或者按照破解版下载页面的安装方法设置 UNRAID U 盘 启动，然后再回到 Windows 进行破解密钥生成。

# 初期配置

## 点亮

先将显示器接上运行 UNRAID 的机器。在开机启动时，狂按 Del 进入 BIOS 配置页面，修改默认主板设置。我先修改默认启动引导至装有 UNRAID 系统的 U 盘上，然后打开了 CPU 的虚拟化功能（VT-d），F10 保存重启。

启动选项选择 `Unraid OS (Headless)` 。如果正常启动，在命令行可以看到 UNRAID 被分配到的 IP 地址。通过 Web 访问即可。默认账户为 root ，无密码。

<!--more-->

## 配置网络

{% asset_img unraid-network-settings.png UNRAID 静态 IP 配置 %}

第一步需要配置网络，保证机器使用固定的 IP 地址。在阵列停止的情况下，进入 SETTINGS - Network Settings 配置网络。为了让 IP 不发生变化，建议给 UNRAID 设置一个静态 IP，网关指向当前主路由。由于目前我的网络没有运行 IPv6 服务，故没有设置 v6 地址。

## 配置阵列

UNRAID 支持软 RAID 0 数据盘（data divices）、校验盘（parity devices）和缓存盘（cache devices）。其中，数据盘和校验盘建议使用机械硬盘，并且校验盘的容量必须大于任意一个数据盘。缓存盘可以使用固态硬盘。使用缓存盘将会极大改善用户体验。所有新安装的硬盘都需要先格式化再运行，格式化选项可以在主页看到。

{% asset_img unraid-array-setting.png 阵列配置 %}

我有两块 HDD 和一块 SSD ，因此我将 SSD 作为缓存盘，HDD 作为数据盘。我暂时不打算上校验盘。

### 共享文件夹的缓存方式

{% asset_img user-share-folder.png User Share %}

我们可以通过 User Share 面板创建内网共享目录，创建出的共享目录可以被 Windows 发现。在系统中挂载了缓存盘的情况下， UNRAID 有四种共享文件夹的缓存策略。

- __Yes__ 只要缓存上的可用空间大于设置的最小可用空间，总是将新文件写入缓存，反之就直接把文件写入数据盘。  
  __mover*__ 运行时，如果文件没有被占用，它就会尝试回写。哪个阵列驱动器将得到文件是由分配方法和共享的Split级别设置的组合控制的。这是一种传统的、缓存工作的方式。
- __No__ 直接将数据写入数据盘。  
  __mover*__ 运行时，即使缓存中有理论上属于对应共享文件夹的数据，也不会写回。这种共享方式不使用缓存。
- __Only__ 直接向缓存中写入新文件。如果缓存上的空闲空间低于设置的最小剩余空间，将抛出空间不足错误并写入失败。  
  __mover*__ 运行时，即使数据盘中有逻辑上属于这个共享的文件，它也不会对这个共享的文件采取任何行动。这意味着这个共享文件夹只存放于缓存，并不存放于数据盘。
- __Prefer__ 只要缓存上的可用空间大于设置的最小可用空间，总是将新文件写入缓存，反之就直接把文件写入数据盘。  
  __mover*__ 运行时，只要缓存上的可用空间大于设置的缓存最小可用空间，它就会尝试将数据盘上对应的共享文件夹移回缓存。  
  这样做的好处是让 VM / Docker 和存放于 appdata 中的配置文件能够更快地被读取和修改，这将大大加速配置文件的读写速度。特别是在需要校验盘进行奇偶校验计算时。即使暂时没有缓存，这个设置也适用。  
  这种策略比较特殊。它意味着 UNRAID 在这个文件共享下更__“倾向于”__使用缓存，但是没有缓存也不会影响实际使用。

\* 此处的 Mover 指的是根据缓存设置，在数据盘和缓存盘之间同步数据的工具，由 UNRAID 默认提供。

### 缓存性能测试

在并不算太好的 HDD 和 SSD 组成的阵列下，我进行了一系列缓存性能的测试。由于电脑和路由器只有千兆网口，速度并不能达到原有 SSD 的读写水平。

__缓存策略 Yes__

{% asset_img cache-test-yes.png 缓存策略 Yes %}

__缓存策略 No__

{% asset_img cache-test-no.png 缓存策略 No %}

可以看到，虽然读写速度受限于网口带宽上限，但是使用了缓存的共享目录读写性能仍有一定的提升。我打算在之后升级万兆网卡，这个瓶颈应该可以得到解决。

在机器上直接对硬盘做读写速度测试，可以得到以下结果。从理论上说，如果能够升级万兆内网，读写性能应该可以到达理论速度。下图的测试中，sdb，sdc 是机械硬盘，sdd 是固态硬盘。

{% asset_img all-disk-test.png 所有硬盘设备读写测试 %}

# 常用插件安装

## 官方 Community Apps 支持

UNRAID 社区已经发展到一定的规模了，社区中不少人会制作长期适用于 UNRAID 的官方 Apps。他们有的是封装好 UNRAID 模板的 Docker 容器，有的可以给 UNRAID 增加功能。进入 Plugins - Install Plugin，输入 https://raw.githubusercontent.com/Squidly271/community.applications/master/plugins/community.applications.plg 安装。安装完成后刷新 Web 页面，就可以看到上方菜单多出了一个 Apps 栏。在 Apps 中，就能安装很多 NAS 常用的工具了。

## Nerd Tools

{% asset_img NerdPack_GUI.png NerdPack GUI %}

可以使用官方 Apps 后，首先我会考虑安装的是 __NerdPack GUI__ 。这是一个可以安装额外软件（大多是 CLI 软件）的工具箱，可以理解为一个简化了的 _apt 源_ ，源里的大部分程序包是 Linux 控制台操作中常用的那部分。点击 `Plugins` - `Nerd Tools` ，点击左下角 Check Update ，就可以选择安装对应软件包。这对日常使用，特别是习惯于操作常见 Linux 发行版的用户来说非常方便。

{% asset_img NerdPack_GUI_2.png NerdPack GUI %}

## Unassigned Devices

{% asset_img Unassigned_Devices.png Unassigned Devices %}

这是一个支持挂载其他 NFS / SMB 虚拟盘作为数据盘的工具，在 `APPS` 里可以安装。具体使用场景虽然我还没用上，但是可以考虑一个虚拟场景：

我有一块群晖数据盘，不想搬迁数据。我可以通过直通将数据盘挂载在 UNRAID 虚拟出来的黑群晖上，然后黑群晖开 SMB / NFS 给 UNRAID 使用。或者局域网中的其他 BT 下载机的下载盘，挂载在 UNRAID 下，用于 PLEX 或者 Jellyfin 播放。

## Dynamix S3 Sleep

{% asset_img dynamix-s3-sleep.png Dynamix S3 Sleep %}

这是一个可以自动判断硬盘是否在工作，在特定时间让 UNRAID 服务器休眠的工具，在 `APPS` 里可以安装。但是由于后来我安装了软路由，所以就没有继续开启睡眠功能。有需要的时候这个工具还是很方便的，可以定时关机，降低功耗和噪音。

# 其他调整

## 更换 Docker 镜像

由于 UNRAID 对 Docker 生态有较大的依赖，很多 Docker 应用都可以和 UNRAID 协同工作。需要更换 Docker 国内镜像，以达到较好的使用体验。这里推荐阿里云的 Docker 镜像加速器，访问 https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors 就可以申请。拿到自己的 Docker 镜像加速器地址后，在 Web Terminal 中执行以下命令对 Docker 源进行更换。

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://{YOUR_DOCKER_MIRROR_ID}.mirror.aliyuncs.com"]
}
EOF
```

为了在每次重启时自动应用配置，建议将其写入 `\flash\config\go` 中。其中 `\flash` 指装有 UNRAID 的 U 盘根目录。

## 允许 Docker 虚拟机访问宿主机

由于 Docker 的网络隔离，有时候我想进行一些特殊的操作就比较困难。比如我将所有的服务用一个 NginX 反代，但是使用 `Host` 或者 `Bridge` 方式暴露端口在宿主机上的服务就无法被反代。因此，我需要手工修改 UNRAID 的 Docker 设置。进入 `Settings` - `Docker` ，点击右上方的 `Basic View` ，切换成 `Advanced View` ，然后修改对应选项即可。这么做存在一定风险。我认真检查防火墙，保证外网的访问能够安全地被处理后，才进行这个操作。

{% asset_img docker-host-access-to-custom-networks.png Unassigned Devices %}


---
以上，我的 UNRAID 就做好作为一台 NAS 进行工作的准备了。下一章我会继续介绍我安装的应用程序和配置，合理发挥 NAS 的功能。

# References

- https://unraid.net/
- https://forums.unraid.net/topic/38582-plug-in-community-applications/
- https://wiki.unraid.net/UnRAID_6/Getting_Started
- https://wiki.unraid.net/UnRAID_6/Storage_Management
- https://wiki.unraid.net/Check_Harddrive_Speed
- https://forums.unraid.net/topic/38582-plug-in-community-applications/
- https://forums.unraid.net/topic/35866-unraid-6-nerdpack-cli-tools-iftop-iotop-screen-kbd-etc/
- https://forums.unraid.net/topic/77390-cannot-access-dockers-using-custombr0/
