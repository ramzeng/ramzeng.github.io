---
title: "Cgroups"
date: 2023-08-21T10:50:18+08:00
draft: false
tags: ["Linux", "Docker"]
---

## 介绍

cgroups 是 Linux 内核中的一项功能，用于限制、记录和隔离一组进程的资源使用，例如 CPU、内存、磁盘 I/O 等。

> cgroups 是 control groups 的缩写，中文名为控制组。

## 子系统

cgroups 由多个子系统组成，每个子系统负责一种资源的限制和隔离。

- cpu：限制进程的 CPU 使用率
- cpuacct：统计进程的 CPU 使用报告
- cpuset：为进程分配独立的 CPU 节点或内存节点
- memory：限制进程的内存使用量
- blkio：限制进程的块设备 I/O
- devices：限制进程访问设备的权限
- net_cls：为进程分配独立的网络标识符
- freezer：挂起或恢复进程
- ns：为进程分配独立的命名空间

## CPU 使用率限制实践

进入 cgroups 的 CPU 子系统目录

```bash
cd /sys/fs/cgroup/cpu
```

创建一个名为 playground 的 cgroup

```bash
mkdir playground
```

执行一个 CPU 密集型任务

```bash
while : ; do : ; done &
```

修改 playground 的 CPU 使用率限制为 50%

```bash
echo 50000 > playground/cpu.cfs_quota_us
```

> cpu.cfs_period_us 的默认值为 100000，表示 100ms。cpu.cfs_quota_us 的默认值为 -1，表示不限制。
>
> 将 cpu.cfs_quota_us 设置为 50000，表示 100ms 内只能使用 50ms 的 CPU，即 CPU 使用率为 50%。

将进程 ID 添加到 playground/tasks 中

```bash
echo ${process_id} > playground/tasks
```

> process_id 为上一步中执行的 CPU 密集型任务的进程 ID。

查看进程的 CPU 使用率

```bash
top
```

> 按下 Shift + P，按照 CPU 使用率排序，可以看到 CPU 使用率为 50%。
