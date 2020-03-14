---
title: Syncthing 文件同步工具部署和 iOS 替代方案
date: 2020-03-14 22:45:40
categories:
- Web
- Tool
tags:
- Syncthing
- Sync
---

Syncthing Official Site: https://syncthing.net/

Syncthing Github: https://github.com/syncthing/

KodExplorer Official Site: https://kodcloud.com/

KodExplorer Github: https://github.com/kalcaddle/KodExplorer

# Introduction

> Syncthing is a continuous file synchronization program. It synchronizes files between two or more computers in real time, safely protected from prying eyes. Your data is your data alone and you deserve to choose where it is stored, whether it is shared with some third party, and how it's transmitted over the internet.

*Syncthing* 是一个实时的文件同步程序，可以在两台以上的设备之间进行实时、端对端的文件同步。在不同设备间同步的时候，还可以对每个设备分别设置文件版本控制，保留被删除的文件的副本或者更改前的旧版本，在办公、科研、文档共享和数据共享上都有很大用处。

我将 *Syngthing* 安装在了学校电脑、 Surface 和远端服务器上，远端服务器有公网 IP ，方便进行快速同步。这样只要我本地的数据发生更新，远端服务器就会同步数据，并且保存文件的旧版本。

# 服务器安装

虽然是开源项目，但是我不建议使用源代码安装。源码安装不利于软件的升级和版本控制。

官网对文件安装的文档说明已经非常详细了，这边简单摘抄一些，并分享在安装过程中发现的问题并提供解决方案。

全平台（除了 iOS）的安装文件可以在这里找到 https://syncthing.net/downloads/ 。

## Deb 系

用户可以在两条 release 轨道中进行选择，分别是"stable" (latest release) 和 "candidate" (earlier release candidate) 。

同时强烈建议通过 HTTPS 从 apt 源进行下载。

Candidate Track 的版本升级会比 Stable Track **早大约三个星期**，追求刺激的用户可以使用 Candidate Track 。

### Stable Track

```bash
# Add apt https support
sudo apt-get install apt-transport-https

# Add the release PGP keys:
curl -s https://syncthing.net/release-key.txt | sudo apt-key add -

# Add the "stable" channel to your APT sources:
echo "deb https://apt.syncthing.net/ syncthing stable" | sudo tee /etc/apt/sources.list.d/syncthing.list

# Update and install syncthing:
sudo apt-get update
sudo apt-get install syncthing
```

<!--more-->

### Candidate Track

```bash
# Add apt https support
sudo apt-get install apt-transport-https

# Candidate Version
# Add the release PGP keys:
curl -s https://syncthing.net/release-key.txt | sudo apt-key add -

# Add the "candidate" channel to your APT sources:
echo "deb https://apt.syncthing.net/ syncthing candidate" | sudo tee /etc/apt/sources.list.d/syncthing.list

# Update and install syncthing:
sudo apt-get update
sudo apt-get install syncthing
```

## RedHat 系

RedHat 系统可以直接在官网下载已经编译好的程序，并直接执行。为了方便执行，可以通过 `ln -s` 将程序可执行文件软连接到 `/usr/bin` 下。

## Windows & MacOS

可以直接在官网下载到可执行程序，执行后将会自动弹出 Web 页面。

# 打通 Syncthing 节点之间的第一条 P2P 连接

让我们开始连接第一条端到端同步线路。这条线路将会在本地 PC 和远端服务器之间同步数据。

我打算将我的服务器作为文件存放的主服务器，一大原因是服务器有公网地址，不需要通过 NAT 穿透来访问，这会大大提高同步速度。我的主服务器是 Debian buster。

因为 *Syncthing* 并没有 iOS 客户端。为了在 iOS 上使用 *Syncthing* ，我采用了一个 Trick，用 *KodExplorer* 作为网页访问文件的工具，并将同步文件夹放置在 *KodExplorer* 的目录下。所以，如果想保持 *KodExplorer* 和 *Syncthing* 可以正常协同工作，需要单独创建一个用户来运行 *Syncthing* 和 *PHP* ，保证他们的 **读写权限相同** 。同时，官方也建议不要使用 root 账户运行 *Syncthing* 。

我使用 `www` 用户运行 *PHP* ，所以现在默认我已经切换到 `www` 账户运行 *Syncthing* 了。

我使用 `screen` 控制程序后台运行。在安装完程序后，执行命令 `syncthing` 让其在 `~/.config/syncthing` 下生成初始配置文件。程序默认监听 127.0.0.1:8384 作为其 Web GUI 页面。如果你有 *NginX* 或者 *Apache* ，可以设置一个反向代理到这个端口，就能直接进入网站页面。如果没有，需要 `Ctrl + C` 终止程序运行，并修改 `~/.config/syncthing/config.xml` 中的监听地址为 `0.0.0.0` ，然后通过公网 IP 访问该地址进行配置。如果这个页面是暴露在公网上的，请 **务必设置管理员密码** 。

远端服务器节点配置完成后，我们开始配置本地 PC 节点。在 PC 中下载程序并运行，在 GUI 页面中点击右下角 **添加远程设备** ，输入从远端服务器的 ID（服务器 GUI 界面右上角操作-显示 ID），然后等待一段时间，远端服务器会收到本地 PC 发起的远程设备连接请求，同意即可。

如果没有收到请求，可以检查一下远端服务器是否打开了端口 22000/TCP ，21027/UDP 防火墙。

远端接受请求后，就可以添加本地同步文件夹，并和远端进行共享了。

理论上说，如果两个设备都在 NAT 设备背后，也可以进行数据同步。但是在测试过程中发现数据同步速度相对缓慢，并且无法用本文的 Trick 让 iOS 设备访问。

# Syncthing 和 KodExplorer 的协同工作

为了让这两个程序协作，让 iOS 设备也能访问远端文件，我们需要把同步文件夹放在 *KodExplorer* 程序的 `data/对应用户/home` 文件夹下。这样，iOS 设备就可以通过浏览器访问远程文件了。

*KodExplorer* 的安装这里不赘述。

当然，这也会有安全问题。将私人文件放在可以直接被访问到的服务器目录下是一件很危险的行为。这时候需要对 `data/` 文件夹进行访问权限控制，在 *NginX* 配置文件中增加一条即可。

```nginx
location /data/ {
    deny all;
}
```

{% asset_img 403-Foibbden.png NginX 403 Foibbden %}

这样，在 iOS 设备上，可以直接通过访问 *KodExplorer* 页面来访问文件，并且能够在多端间进行同步。
