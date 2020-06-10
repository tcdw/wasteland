---
title: Docker 限制磁盘 IO 无效排错及总结
date: 2020-03-16 01:47:48
categories:
- [Linux]
- [Docker]
tags:
- Docker
- blkio
- Cgroups
---

> 该文完稿于 2020-03-13 凌晨

# Background

最近在做一个与 Docker 相关的实验，其中需要限制 Docker 容器中应用程序的 IO，比如 NginX 的 IO。这听起来很简单，毕竟远在 Feb 4th, 2016 release 的 Docker v1.10 就在其功能中加入了限制容器 IO 的参数

> **Constraints on disk I/O:** Various options for setting constraints on disk I/O have been added to `docker run`: `--device-read-bps`, `--device-write-bps`, `--device-read-iops`, `--device-write-iops`, and `--blkio-weight-device`.
>
> https://www.docker.com/blog/docker-1-10/

就在一切都顺利进行，我写完包含了 `NginX` 的 `Dockerfile` ，准备满心欢喜地开始我的 `1MB/s` 实验的时候，一道晴天霹雳打在我心上——

```bash
root@53ace8551c27:/#$ dd if=500M.file bs=1M count=500 of=/dev/null
500+0 records in
500+0 records out
524288000 bytes (524 MB, 500 MiB) copied, 0.132448 s, 4.0 GB/s
```

当然问题现在已经解决了。为了重现当时的情况，我们从头开始。

# Toolbox

我们简单地使用 Debian 作为测试的 Docker Image。

```bash
$ docker pull debian
```

并且使用 `dd` 命令生成一个 500M 的 000 文件，测试磁盘读写速度。

```bash
$ dd if=/dev/zero of=500M.file bs=1M count=500
$ dd if=500M.file bs=1M count=500 of=/dev/null
```

# Yesterday Once More *

> \* [Yesterday Once More – Carpenters](https://open.spotify.com/track/3wM6RTAnF7IQpMFd7b9ZcL)

## 初次碰壁

拉镜像

```bash
$ docker pull debian

Using default tag: latest
latest: Pulling from library/debian
50e431f79093: Pull complete
Digest: sha256:a63d0b2ecbd723da612abf0a8bdb594ee78f18f691d7dc652ac305a490c9b71a
Status: Downloaded newer image for debian:latest
docker.io/library/debian:latest
```

找到宿主机的设备路径

```bash
$ sudo fdisk -l

Disk /dev/vda: 50 GiB, 53687091200 bytes, 104857600 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: -

Device      Start       End   Sectors  Size Type
/dev/vda1  227328 104857566 104630239 49.9G Linux filesystem
/dev/vda14   2048     10239      8192    4M BIOS boot
/dev/vda15  10240    227327    217088  106M EFI System

Partition table entries are not in disk order.
```

起一个 Docker，限制对应设备的读写速度

```bash
$ docker run -it --rm --device-read-bps /dev/vda:1MB  --device-write-bps /dev/vda:1MB debian

root@9f79e6469b67:/#
```

```bash
$ dd if=/dev/zero of=500M.file bs=1M count=500

500+0 records in
500+0 records out
524288000 bytes (524 MB, 500 MiB) copied, 0.728238 s, 720 MB/s

$ dd if=500M.file bs=1M count=500 of=/dev/null

500+0 records in
500+0 records out
524288000 bytes (524 MB, 500 MiB) copied, 0.132448 s, 4.0 GB/s
```

**？**

这是一个非常可怕的事情。我限制的 `1MB/s` 并不工作。这个实验是基于这个假设进行的，如果没有办法限制设备的 IO，实验也没有办法继续进行了。

<!--more-->

## 另辟蹊径

为了完成实验，我找了大量的资料。首先我把焦点放在使用 `systemd` 控制资源限制上。

根据 `systemd` 的文档（ https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#Options ），我们可以在对应服务的 systemd 配置文件中增加一些参数来实现自动化的资源控制，包括 CPU 资源限制，Memory 资源限制，进程数资源限制和 IO 限制。

> IOAccounting =
>
> > Turn on Block I/O accounting for this unit, if the unified control group hierarchy is used on the system. Takes a boolean argument. Note that turning on block I/O accounting for one unit will also implicitly turn it on for all units contained in the same slice and all for its parent slices and the units contained therein. The system default for this setting may be controlled with `DefaultIOAccounting=` in [systemd-system.conf(5)](https://www.freedesktop.org/software/systemd/man/systemd-system.conf.html#).
> >
> > This setting replaces `BlockIOAccounting=` and disables settings prefixed with `BlockIO` or `StartupBlockIO`.
>
> IOReadBandwidthMax=*`device`* *`bytes`*`, `IOWriteBandwidthMax=*`device`* *`bytes`*
>
> > Set the per-device overall block I/O bandwidth maximum limit for the executed processes, if the unified control group hierarchy is used on the system. This limit is not work-conserving and the executed processes are not allowed to use more even if the device has idle capacity. Takes a space-separated pair of a file path and a bandwidth value (in bytes per second) to specify the device specific bandwidth. The file path may be a path to a block device node, or as any other file in which case the backing block device of the file system of the file is used. If the bandwidth is suffixed with K, M, G, or T, the specified bandwidth is parsed as Kilobytes, Megabytes, Gigabytes, or Terabytes, respectively, to the base of 1000. (Example: "/dev/disk/by-path/pci-0000:00:1f.2-scsi-0:0:0:0 5M"). This controls the "`io.max`" control group attributes. Use this option multiple times to set bandwidth limits for multiple devices. For details about this control group attribute, see [IO Interface Files](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#io-interface-files).
> >
> > These settings replace `BlockIOReadBandwidth=` and `BlockIOWriteBandwidth=` and disable settings prefixed with `BlockIO` or `StartupBlockIO`.
> >
> > Similar restrictions on block device discovery as for `IODeviceWeight=` apply, see above.

这个方法一听就非常靠谱。 `systemd` 是一个让人又爱又恨的工具，我曾经为了从 `service` 切换到 `systemctl` 不知道背了多久这个命令的单词拼写。由于我们需要限制 NginX 的 IO 资源，首先要找到 NginX 的 `systemd` 配置文件。

```bash
$ sudo systemctl status nginx

● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset:
   Active: active (running) since Wed 2020-03-11 14:17:51 UTC; 24h ago
     Docs: man:nginx(8)
  Process: 3677 ExecStop=/sbin/start-stop-daemon --quiet --stop --retry QUIT/5
  Process: 3685 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (co
  Process: 3678 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_proces
 Main PID: 3687 (nginx)
    Tasks: 17 (limit: 4915)
   CGroup: /system.slice/nginx.service
           ├─3687 nginx: master process /usr/sbin/nginx -g daemon on; master_p
           ├─3688 nginx: worker process
           ...
```

然后进入 `/lib/systemd/system/nginx.service` ，修改 `[Service]` 块，增加三行。

```bash
IOAccounting=true
IOReadBandwidthMax=/dev/vda 1M
IOWriteBandwidthMax=/dev/vda 1M
```

Reload daemon and nginx。

```bash
$ sudo systemctl daemon-reload
$ sudo systemctl restart nginx
```

由于我在 NginX 的网页目录下放了一个测试下载速度的文件，我换了内网其他机器去拉。

```bash
$ wget http://10.10.194.18/dash/test.file
--2020-03-12 15:07:57--  http://10.10.194.18/dash/test.file
Connecting to 10.10.194.18:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 104857600 (100M) [application/octet-stream]
Saving to: ‘test.file’

test.file           100%[=================>] 100.00M   339MB/s    in 0.3s

2020-03-12 15:07:57 (339 MB/s) - ‘test.file’ saved [104857600/104857600]
```

这个结果无疑告诉我这次尝试又失败了。

## 曙光初现

这个问题真的很奇怪，为什么我对资源的限制会不起作用。我后来在 NginX 的 `systemd` 配置文件中增加了 Memory 的 limit 是 work 的，但是 IO 相关的就不行。在本地虚拟机测试中，不管是 Docker 的 IO 限制还是 NginX 的 `systemd` 资源控制都是生效的，甚至一度让我怀疑是远程服务器的镜像问题。因为根据我咨询运维人员的情况来看，远程机器的镜像都经过特殊定制，有可能是这个原因。

不过，后来我的 Teammate 给了我一个 **很重要的提示** 。通过给 `dd` 命令增加参数 `oflag=direct` ，在 Docker 中可以得到限速后的效果。

```bash
root@9f79e6469b67:/# dd if=500M.file bs=1M count=500 of=500.out oflag=direct
^C
18+0 records in
18+0 records out
18874368 bytes (19 MB, 18 MiB) copied, 18.0052 s, 1.0 MB/s
```

这个参数的作用是什么呢？GNU https://www.gnu.org/software/coreutils/manual/html_node/dd-invocation.html#dd-invocation 介绍如下

> **‘oflag=flag[,flag]…’**
>
> Access the output file using the flags specified by the flag argument(s). (No spaces around any comma(s).)
>
> Here are the flags. Not every flag is supported on every operating system.
>
> > **‘direct’**
> >
> > Use direct I/O for data, avoiding the buffer cache. Note that the kernel may impose restrictions on read or write buffer sizes. For example, with an ext4 destination file system and a Linux-based kernel, using ‘oflag=direct’ will cause writes to fail with `EINVAL` if the output buffer size is not a multiple of 512.

这说明远程服务器的读写缓存对硬盘 IO 产生了巨大的影响。虽然我记得之前不知道在哪里看到过说，Cgroups 的磁盘 IO 限制模块 blkio 会对读写 buffer 进行限制，但是由于这是远程服务器，并且镜像经过定制，可能在系统底层绕开了这一限制，用于提升服务器 IO。

我也曾经考虑过读写 buffer 的问题，但是当时执行命令清空读写缓存时，遇上了这样的问题

```bash
$ sudo echo 1 > /proc/sys/vm/drop_caches
-bash: /proc/sys/vm/drop_caches: Permission denied
```

我便没有继续。

## 问题解决

最终，我们把问题锁定在服务器的读写缓存上。既然已经知道了问题所在，解决起来也就相对容易。虽然没有办法直接将清除缓存命令写进特定位置，但是可以用这条命令解决。

```bash
$ sudo sh -c "/bin/echo 1 > /proc/sys/vm/drop_caches"
```

这是工作的。至于为啥，我没研究。

还有另一套方案，将磁盘缓存的超时时间设置极低，也可以解决。

```bash
$ sudo echo 100 > /proc/sys/vm/dirty_expire_centisecs
$ sudo echo 100 > /proc/sys/vm/dirty_writeback_centisecs
```

当然，这些命令都要在宿主机进行操作，因为 Docker 本质只是一个在宿主机上虚拟化的线程。`/proc/sys/vm/` 文件夹对 Docker 容器来说，只是一个 Read-Only 的文件系统。

```bash
root@9f79e6469b67:/# echo 100 > /proc/sys/vm/dirty_writeback_centisecs
bash: /proc/sys/vm/dirty_writeback_centisecs: Read-only file system
```

最终，在清除缓存后，一切都变得正常起来。

```bash
root@9f79e6469b67:/# dd if=500M.file bs=1M count=500 of=/dev/null
^C
44+0 records in
43+0 records out
45088768 bytes (45 MB, 43 MiB) copied, 44.4906 s, 1.0 MB/s
```

```bash
$ wget 10.10.194.18/dash/test.file
--2020-03-12 15:42:29--  http://10.10.194.18/dash/test.file
Connecting to 10.10.194.18:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 104857600 (100M) [application/octet-stream]
Saving to: ‘test.file.1’

test.file.1         100%[=================>] 100.00M  1005KB/s    in 1m 45s

2020-03-12 15:44:14 (977 KB/s) - ‘test.file.1’ saved [104857600/104857600]
```

# Appendix

我觉得这篇文章要是就这么结束，内容应该有点太少了。在填坑过程中，我还研究了不少和系统资源控制相关的内容。

## Cgroup

Cgroup 是 Linux 用于控制进程资源的一种方式，从 2.6.24 内核中开始搭载，v2 版本于 4.5 内核开始搭载。它的配置文件在文件系统中的组织方式是 `/sys/fs/cgroup/{Resource}/{defaultConfigs}` 和 `/sys/fs/cgroup/{Resource}/{Groups}/.../{configs}` 。对应的限制内容会被写在目录的文件下，限制进程的 `pid` 会被写在目录的 `tasks` 文件夹下。简单看看本文主角 **blkio** 文件夹下的结构。

```bash
/sys/fs/cgroup/blkio$ ls
blkio.io_merged                   blkio.throttle.io_serviced
blkio.io_merged_recursive         blkio.throttle.read_bps_device
blkio.io_queued                   blkio.throttle.read_iops_device
blkio.io_queued_recursive         blkio.throttle.write_bps_device
blkio.io_service_bytes            blkio.throttle.write_iops_device
blkio.io_service_bytes_recursive  blkio.time
blkio.io_service_time             blkio.time_recursive
blkio.io_service_time_recursive   blkio.weight
blkio.io_serviced                 blkio.weight_device
blkio.io_serviced_recursive       cgroup.clone_children
blkio.io_wait_time                cgroup.procs
blkio.io_wait_time_recursive      cgroup.sane_behavior
blkio.leaf_weight                 docker
blkio.leaf_weight_device          notify_on_release
blkio.reset_stats                 release_agent
blkio.sectors                     system.slice
blkio.sectors_recursive           tasks
blkio.throttle.io_service_bytes   user.slice
```

之前我们修改的 `systemd/nginx.service` 的内容被放在 `system.slice` 下，docker 的资源限制被放在 `docker/` 和 `docker/container_id` 下。

看看 cgroup 支持哪些资源

```bash
$ lssubsys  -m
cpuset /sys/fs/cgroup/cpuset
cpu,cpuacct /sys/fs/cgroup/cpu,cpuacct
blkio /sys/fs/cgroup/blkio
memory /sys/fs/cgroup/memory
devices /sys/fs/cgroup/devices
freezer /sys/fs/cgroup/freezer
net_cls,net_prio /sys/fs/cgroup/net_cls,net_prio
perf_event /sys/fs/cgroup/perf_event
hugetlb /sys/fs/cgroup/hugetlb
pids /sys/fs/cgroup/pids
rdma /sys/fs/cgroup/rdma
```

简单验证一下生效的几个配置。

## Docker 资源限制

Docker ID: 9f79e6469b67

```bash
$ docker inspect 9f79e6469b67 | grep Pid
            "Pid": 6510,
            "PidMode": "",
            "PidsLimit": null,
```

```bash
$ cd /sys/fs/cgroup/blkio/docker/9f79e6469b67... 
$ ls
blkio.io_merged                   blkio.sectors_recursive
blkio.io_merged_recursive         blkio.throttle.io_service_bytes
blkio.io_queued                   blkio.throttle.io_serviced
blkio.io_queued_recursive         blkio.throttle.read_bps_device
blkio.io_service_bytes            blkio.throttle.read_iops_device
blkio.io_service_bytes_recursive  blkio.throttle.write_bps_device
blkio.io_service_time             blkio.throttle.write_iops_device
blkio.io_service_time_recursive   blkio.time
blkio.io_serviced                 blkio.time_recursive
blkio.io_serviced_recursive       blkio.weight
blkio.io_wait_time                blkio.weight_device
blkio.io_wait_time_recursive      cgroup.clone_children
blkio.leaf_weight                 cgroup.procs
blkio.leaf_weight_device          notify_on_release
blkio.reset_stats                 tasks
blkio.sectors
$ cat tasks
6510
$ cat blkio.throttle.read_bps_device
252:0 1048576
```

其中，252 : 0 是磁盘设备号 `<major>:<minor>` ，1048576 = 1024 * 1024

```bash
$ ls -l /dev/vda
brw-rw---- 1 root disk 252, 0 Mar 11 13:32 /dev/vda
```

这说明在我们启动一个资源受限的 Docker 时，Docker 会自动在自身 cgroup 资源限制组下生成名为 `container_id` 的文件夹，然后将对应容器的 Pid 和限制资源规则写入。

## Systemd 资源限制

```bash
$ cd /sys/fs/cgroup/blkio/system.slice/nginx.service
$ cat tasks
6648
6651
...
```

```bash
$ cat blkio.throttle.read_bps_device
252:0 1000000
$ sudo systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset:
   Active: active (running) since Thu 2020-03-12 15:06:14 UTC; 1h 5min ago
     Docs: man:nginx(8)
  Process: 6632 ExecStop=/sbin/start-stop-daemon --quiet --stop --retry QUIT/5
  Process: 6646 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (co
  Process: 6633 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_proces
 Main PID: 6648 (nginx)
 ...
```

NginX 的所有进程 Pid 和 `.../system.slice/nginx.service/tasks` 中的 Pid 一一对应。

`systemd` 的资源限制工作方式和 Docker 类似。

有趣的是，Docker 采用 2 ^ 10  作为单位，而 systemd 采用 1000 作为单位。

# References

- [Systemd] https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html#Options
- [dd] https://www.gnu.org/software/coreutils/manual/html_node/dd-invocation.html#dd-invocation
- [Disk Cache] https://stackoverflow.com/questions/20215516/disabling-disk-cache-in-linux
- [Disk Cache] https://unix.stackexchange.com/questions/48138/how-to-throttle-per-process-i-o-to-a-max-limit
- [Disk Cache] https://unix.stackexchange.com/questions/109496/echo-3-proc-sys-vm-drop-caches-permission-denied-as-root
- [Cgroup] https://coolshell.cn/articles/17049.html
- [Cgroup] https://tech.meituan.com/2015/03/31/cgroups.html
- [Cgroup] https://cizixs.com/2017/08/25/linux-cgroup/
- [Cgroup blkio] https://andrestc.com/post/cgroups-io/
- [Cgroup blkio] https://www.kernel.org/doc/Documentation/cgroup-v1/blkio-controller.txt
- [Docker IO] https://stackoverflow.com/questions/36145817/how-to-limit-io-speed-in-docker-and-share-file-with-system-in-the-same-time

