---
title: 优雅地管理内网集群
date: 2020-04-02 00:32:57
categories: 
- [Linux]
- [Network]
tags:
- V2Ray
- NAT Traverse
---

本文主要介绍了我在对内网集群进行管理的时候遇上的和解决的问题，包括统一控制，装机脚本，堡垒机，内网穿透等一系列问题。

# 突然有了内网集群

由于科研需要，实验室购置和申请了大量服务器和显卡，并申请了从超算中虚拟出来的计算节点。如何将他们统一管理成了一个很大的问题。同时，由于机器都托管在数据中心，需要 VPN 进行访问。数据中心提供的 VPN 是上古的 `L2TP/IPsec with Pre-shared key`，还要在注册表写信息允许弱加密才能在 Windows 10 上使用， Mac / Linux 电脑完全无法连上这批机器。同时，VPN 还严格限制同时只能一个人使用，一旦使用的人数增加，需要多次申请不同的账号，管理起来十分麻烦。为了解决一系列的问题，我开始探索方案。

# 统一管理：堡垒机

## 选择 JumpServer

俺找了有相关经验的好朋友[奶鱼](https://www.milkfish.site/) ，他推荐了一套堡垒机方案 [JumpServer](https://jumpserver.org/)。这是一套开源的堡垒机程序，支持身份认证，账号管理，资产授权，操作审计等管理功能，也支持批量部署，Web Terminal 等通过 Web 操作机器的功能。整体上看，非常符合我管理内网三十多台机器的需求。

> 开源仓库一览
>
>- JumpServer https://github.com/jumpserver/jumpserver
>- Koko (SSH, WS server) https://github.com/jumpserver/koko
>- Luna (Web terminal) https://github.com/jumpserver/luna

## 前期准备

要想部署一个能够管理所有服务器的堡垒机的平台，就需要给每个节点部署一个带 sudo 权限的用户。这个用户最好在不同机器上是统一的。现有的服务器用户虽然也有部分统一，但使用一个独立的用户管理所有服务器更符合直觉。我先创建一个统一的用户 `touko` ，并赋予其 `sudo NOPASSWD` 权限。

```bash
echo "SUDO_PASSWORD" | sudo -S useradd -m touko -s /bin/bash
echo "touko ALL=(ALL:ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/touko
```

为了安全，我创建了一组 `ECDSA` 算法的密钥对，然后将它的公钥导入 `authorized_keys` 作为登陆密钥。

```bash
ssh-keygen -t ecdsa -C 'jumpserver'
cd ~/.ssh
cat id_ecdsa.pub >> authorized_keys
```

然后，将创建的 ECDSA 私钥 `~/.ssh/id_ecdsa` 备份并删除。在其他的机器上，编写脚本，拷贝私钥至脚本中，就能一键完成部署（顺便退出命令行）。

<!--more-->

```bash
echo "SUDO_PASSWORD" | sudo -S useradd -m touko -s /bin/bash
echo "touko ALL=(ALL:ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/touko

sudo su touko
mkdir ~/.ssh
echo "ecdsa-sha2-nistp256 YOUR_ECDSA_KEY" >> ~/.ssh/authorized_keys
echo "PubkeyAuthentication yes" | sudo tee -a /etc/ssh/sshd_config
echo "AuthorizedKeysFile      .ssh/authorized_keys .ssh/authorized_keys2" | sudo tee -a /etc/ssh/sshd_config
sudo systemctl restart ssh
exit
exit
exit
```

这样一来，就可以用同样的私钥登录每台服务器。对需要被管理的服务器来说，只需登上去运行一次脚本，就能完成加入堡垒机的前期配置。

## JumpServer 的安装

JumpServer 的安装有详细的文档，具体可以[在此查看](https://jumpserver.readthedocs.io/zh/master/install/step_by_step/)。由于规模不大，我直接使用了官方提供的 [极速安装](https://jumpserver.readthedocs.io/zh/master/install/setup_by_fast/) 指令。该指令只适用于 `CentOS 7.x`。该指令会安装 JumpServer 主程序，并在 Docker 下运行 koko 和 luna。koko 和 luna 有时候会假死（表现为 Web Terminal 无响应），需要手工重启 JumpServer。

```bash
cd /opt
yum -y install wget git
git clone --depth=1 https://github.com/jumpserver/setuptools.git
cd setuptools
cp config_example.conf config.conf
vi config.conf

# Install
./jmsctl.sh install
```

```bash 手工重启 JumpServer
sudo docker stop jms_koko
sudo docker stop jms_guacamole
sudo systemctl stop jms_core

sudo systemctl start jms_core 
sudo docker start jms_koko
sudo docker start jms_guacamole
```

在使用极速安装之前，我一度偷懒使用 Docker 部署。但实际测试起来，用 Docker 运行的堡垒机非常不稳定，特别是 Web Terminal 经常跑着跑着就断线。最后我还是额外申请了一台 CentOS 的服务器（已有服务器全是 Ubuntu），作为集群的堡垒机。

## JumpServer 的使用

JumpServer 的设计非常用户友好，基本不需要额外学习就能很快上手。下图是我添加完所有节点后的截图，可以一目了然的知道当前系统有多少机器，机器如何分组规划。

{% asset_img JumpServer-Sample.png JumpServer Sample %}

# V2Ray 内网穿透

堡垒机的问题解决后，下一个要解决的问题是内网穿透。我只打算将堡垒机作为用户管理和应急使用的 Web Terminal，一是由于就算是极速安装的 JumpServer ，依然会出现 Web Terminal 假死的情况。目测是由于其他两个运行于 Docker 下的程序（koko，Luna）中某个程序出问题导致的。

这时候就需要内网穿透工具。经过一系列的调研和尝试，最终选择了我熟悉的 V2Ray，提供 JumpServer Web UI 和用户直接访问机器的穿透服务。

## 原理

内网穿透依赖的是 V2Ray 中的反向代理功能，该功能在 V2Ray 4.0+ 后支持。我用一张图来解释 V2Ray 中的内网穿透是如何工作的。

{% asset_img v2ray-NAT-traversal.png V2Ray NAT Traversal %}

在内网穿透用的 V2Ray 程序中，节点的两个属性被定义。一个叫 `bridge`，另一个叫 `portal` 。这两个属性定义于 V2Ray 配置文件中 `ReverseObject` 项。`bridge` 作为内网的一方，会主动向 `portal` 方发起连接并长期保持。`bridge` 和 `portal` 都需要手工定义路由表来实现定向的流量转发。

建议先运行公网服务器 `portal`，再运行内网服务器 `bridge`。内网服务器会根据配置的 vmess 连接方式，连接公网服务器对应端口，并接收其所有流量。这就完成了内网穿透的第一步，保持一条从内网到公网的隧道（`tunnel`），让内网服务器可以主动接受其他连接。为了实现反向代理流量的转发，我们还需要手工配置路由表。直接从配置文件入手。

## 配置文件样例

此处我直接修改来自 V2Ray 白话文教程的配置文件样例。

```json 内网服务器
{
  "reverse": {
    "bridges": [{
      "tag": "bridge",
      "domain": "same.as.others.com"
    }]
  },
  "outbounds": [{
    "tag": "tunnel",
    "protocol": "vmess",
    "settings": {  // 公网服务器 vmess 连接方式
      "vnext": [{
        "address": "public.touko.moe",  
        "port": 10086,
        "users": [{
          "id": "b831381d-6324-4d53-ad4f-8cda48b30811",
          "alterId": 64
        }]
      }]
    }
  }, {
    "protocol": "freedom",
    "tag": "lan"
  }],
  "routing": {
    "rules": [{  // Rule 1
      "type": "field",
      "inboundTag": ["bridge"],
      "domain": ["full:same.as.others.com"],
      "outboundTag": "tunnel"
    }, {  // Rule 2
      "type": "field",
      "inboundTag": ["bridge"],
      "outboundTag": "lan"
    }]
  }
}
```

`Rule 1` 定义 `bridge` 应该主动连接 tag 为 tunnel 的公网服务器，并附带识别域名 `same.as.others.com` 。这个域名需要保证在 `bridge` 、 `portal` 和其路由表中都是一样的（共四处）。`Rule 2` 定义当 `bridge` 收到来自 `tunnel` 的反向连接后，转发至 tag 为 `lan` 的 outbound，在本配置文件中是 freedom，也就是直接从内网服务器的网卡发出请求。这样一来，就能直接访问内网资源。

```json 公网服务器
{
  "reverse": {
    "portals": [{
      "tag": "portal",
      "domain": "same.as.others.com"
    }]
  },
  "inbounds": [{
    "tag": "web",
    "port": 80,
    "protocol": "dokodemo-door",
    "settings": { // 内网服务器需要暴露的 IP、端口和协议
      "address": "127.0.0.1",  
      "port": 8080,
      "network": "tcp"  
    }
  }, {
    "tag": "tunnel",
    "port": 10086,
    "protocol": "vmess",
    "settings": {
      "clients": [{
        "id": "b831381d-6324-4d53-ad4f-8cda48b30811",
        "alterId": 64
      }]
    }
  }],
  "routing": {
    "rules": [{  // Rule 3
      "type": "field",
      "inboundTag": ["tunnel"],
      "domain": ["full:same.as.others.com"],
      "outboundTag": "portal"
    }, {  // Rule 4
      "type": "field",
      "inboundTag": ["web"],
      "outboundTag": "portal"
    }]
  }
}
```

`Rule 3` 定义当流量从 `tunnel` 进入时，转发至 `portal`。同样的，`Rule 4` 定义当流量从 `web` 进入时，转发至 `portal`。这两个路由规则一一对应图中的 Method B 和 Method A。如果使用 Method B 访问，；流量的出口就是内网服务器的 LAN，就实现了和挂 VPN 相同的效果。如果使用 Method A 访问，就会根据 tag 为 web 的 `dokodemo-door` setting 中的配置访问对应的端口。这分别对应内网穿透暴露端口的功能和挂 VPN 访问其他内网资源的功能。

值得一提的是，这里复用了内网穿透中接收内网服务器反向连接的隧道和用户访问 Method B 的隧道。在实际使用过程中，可以创建更多的 vmess inbound 并定义路由表来实现 vmess uuid 的分离。

# 结语

通过内网穿透的配置，我将堡垒机的管理面板暴露至公网，并用 NginX 反代，成功实现了不使用 VPN 管理内网集群的方法，并打包了一组 V2Ray 程序代替 VPN，使得需要进行计算任务的同学可以直接使用本地的 HTTP 代理或直接转发服务器 SSH 端口访问集群，不再需要通过 VPN。这方便了实验室硬件资源的管理，并能够更好控制由于人员流动带来的权限吊销问题。在实际运行的一个季度中，整套系统稳定性较高，V2Ray 的反向代理也没有出现崩溃的情况。除了上传文件的速度受限于公网服务器的带宽（阿里云，只有突发 5M），其他并没有什么不好的体验。JumpServer 也非常好用，特别是可以配置 2FA 严格控制用户的登录行为，甚至可以重播用户在 Web Terminal 进行的操作。在服务器宕机时，能够更方便地寻找原因。

这套系统还在不断进行改进，今后会根据实际情况调整。

# References

- https://jumpserver.readthedocs.io/zh/master/setup_by_fast.html
- https://superuser.com/questions/67765/sudo-with-password-in-one-command-line
- https://blog.csdn.net/hejinjing_tom_com/article/details/7767127
- https://guide.v2fly.org/app/reverse.html
- https://www.v2ray.com/chapter_02/reverse.html
