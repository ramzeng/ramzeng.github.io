---
title: "Docker 容器资源视图隔离"
date: 2023-08-21T14:07:02+08:00
draft: false
tags: ["Linux", "Docker"]
---

## 背景

“敏捷”和“高性能”是容器相较于虚拟机最大的优势，基于 Linux Namespace 的隔离机制相比于虚拟化技术也有很多不足之处，比如隔离得不彻底。

## 问题

在容器内执行 `top` 或者 `free`，会显示宿主机的 CPU 和内存信息，不是当前容器的资源数据。因为 `/proc` 文件系统感知不到 [Cgroups](/posts/cgroups) 做的资源限制。

如果不处理这个问题，应用程序在容器内读取到的资源信息都是宿主机的，会导致应用程序的行为不符合预期。

例如 Nginx 配置文件中的 `worker_processes: auto`，会根据宿主机的 CPU 核数来设置 Worker 进程数。

## LXCFS

[LXCFS](https://github.com/lxc/lxcfs) 是一个 FUSE 文件系统，主要用于为 LXC 和其他容器提供一个虚拟化的视图，使得容器内的进程能够看到正确的资源信息，如 CPU、内存、磁盘等。通过 LXCFS，容器内的进程可以访问宿主机上的部分虚拟化文件，以获取正确的资源信息。

### Docker 容器

```bash
docker run -it --name ubuntu --cpuset-cpus 0 --memory 256m --memory-swap 256m \
      -v /var/lib/lxcfs/proc/cpuinfo:/proc/cpuinfo:rw \
      -v /var/lib/lxcfs/proc/diskstats:/proc/diskstats:rw \
      -v /var/lib/lxcfs/proc/meminfo:/proc/meminfo:rw \
      -v /var/lib/lxcfs/proc/stat:/proc/stat:rw \
      -v /var/lib/lxcfs/proc/swaps:/proc/swaps:rw \
      -v /var/lib/lxcfs/proc/uptime:/proc/uptime:rw \
      -v /var/lib/lxcfs/proc/slabinfo:/proc/slabinfo:rw \
      -v /var/lib/lxcfs/sys/devices/system/cpu:/sys/devices/system/cpu:rw \
      ubuntu:18.04 /bin/bash
```

- --cpuset-cpus：指定容器可以使用的 CPU 核心
- --memory：指定容器可以使用的内存大小
- --memory-swap：指定容器可以使用的交换空间大小

--cpus 和 --cpuset-cpus 有一定的区别，前者不能指定具体的 CPU 核心

```bash
docker run --cpus=2 <image>
```

上面这条命令将限制容器在每个 100ms 的时间窗口内，容器最多可以使用 200ms 的 CPU 时间。

这种限制方法允许容器在所有可用的 CPU 核心上运行，但限制了容器在给定时间窗口内可以使用 CPU 的时间。
