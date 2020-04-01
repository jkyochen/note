## Cgroups

Linux Cgroups 的全称是 Linux Control Group

> 最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。
> 还能够对进程进行优先级设置、审计，以及将进程挂起和恢复等操作。

**为什么需要 Cgroups**

虽然容器内的第 1 号进程在“障眼法”的干扰下只能看到容器里的情况，但是宿主机上，它作为第 100 号进程与其他所有进程之间依然是平等的竞争关系。

### 查看

在 Linux 中，Cgroups 给用户暴露出来的操作接口是文件系统，即它以文件和目录的方式组织在操作系统的 `/sys/fs/cgroup` 路径下。

```sh
# 显示可以被 Cgroups 进行限制的资源种类
mount -t cgroup


# 这些都是我这台机器当前可以被 Cgroups 进行限制的资源种类。
root@kubermaster:~# ls /sys/fs/cgroup
blkio  cpu  cpuacct  cpu,cpuacct  cpuset  devices  freezer  hugetlb  memory  net_cls  net_cls,net_prio  net_prio  perf_event  pids  rdma  systemd


# 显示该类资源具体可以被限制的方法
ls /sys/fs/cgroup/cpu
# cfs_period 和 cfs_quota 需要组合使用
# 用来限制进程在长度为 cfs_period 的一段时间内，只能被分配到总量为 cfs_quota 的 CPU 时间。
```

+ Cgroups 的每一项子系统都有其独有的资源限制能力
    + blkio ，为​​​块​​​设​​​备​​​设​​​定​​​ I/O 限​​​制，一般用于磁盘等设备；
    + cpuset ，为进程分配单独的 CPU 核和对应的内存节点；
    + memory ，为进程设定内存使用的限制。

### 使用

```sh
root@kubermaster:/sys/fs/cgroup/cpu# mkdir container
root@kubermaster:/sys/fs/cgroup/cpu# ls container/
cgroup.clone_children  cpuacct.stat   cpuacct.usage_all     cpuacct.usage_percpu_sys   cpuacct.usage_sys   cpu.cfs_period_us  cpu.shares  notify_on_release
cgroup.procs           cpuacct.usage  cpuacct.usage_percpu  cpuacct.usage_percpu_user  cpuacct.usage_user  cpu.cfs_quota_us   cpu.stat    tasks

# 这个目录就称为一个“控制组”。操作系统会在你新创建的 container 目录下，自动生成该子系统对应的资源限制文件。


$ while : ; do : ; done &
[1] 226

$ top
%Cpu0 :100.0 us, 0.0 sy, 0.0 ni, 0.0 id, 0.0 wa, 0.0 hi, 0.0 si, 0.0 st


cat cpu.cfs_quota_us
# -1
cat cpu.cfs_period_us
# 100000

# 20000 us , 20 / 100
echo 20000 > cpu.cfs_quota_us
# 它意味着在每 100 ms 的时间里，被该控制组限制的进程只能使用 20 ms 的 CPU 时间，也就是说这个进程只能使用到 20% 的 CPU 带宽。

# 我们把被限制的进程的 PID 写入 container 组里的 tasks 文件，上面的设置就会对该进程生效了
echo 226 > tasks

$ top
%Cpu0 : 20.3 us, 0.0 sy, 0.0 ni, 79.7 id, 0.0 wa, 0.0 hi, 0.0 si, 0.0 st

# 可通过 rmdir 删除
rmdir container
```


Linux Cgroups 就是一个子系统目录加上一组资源限制文件的组合。

```sh
docker run -it --cpu-period=100000 --cpu-quota=20000 ubuntu /bin/bash
cat /sys/fs/cgroup/cpu/docker/5d5c9f67d/cpu.cfs_period_us
# 100000
cat /sys/fs/cgroup/cpu/docker/5d5c9f67d/cpu.cfs_quota_us
# 20000
```

### 总结

一个正在运行的 Docker 容器，其实就是一个启用了多个 Linux Namespace 的应用进程，而这个进程能够使用的资源量，则受 Cgroups 配置的限制。


**这也是容器技术中一个非常重要的概念，即：容器是一个“单进程”模型。**

单进程意思不是只能运行一个进程，而是只有一个进程是可控的。


由于一个容器的本质就是一个进程，用户的应用进程实际上就是容器里 PID=1 的进程，也是其他后续创建的所有进程的父进程。这就意味着，在一个容器中，你没办法同时运行两个不同的应用，除非你能事先找到一个公共的 PID=1 的程序来充当两个不同应用的父进程，这也是为什么很多人都会用 systemd 或者 supervisord 这样的软件来代替应用本身作为容器的启动进程。

希望容器和应用能够同生命周期，这个概念对后续的容器编排非常重要。否则，一旦出现类似于“容器是正常运行的，但是里面的应用早已经挂了”的情况，编排系统处理起来就非常麻烦了。

### 缺点

Linux 下的 `/proc` 目录存储的是记录当前内核运行状态的一系列特殊文件，用户可以通过访问这些文件，查看系统以及当前正在运行的进程的信息，比如 CPU 使用情况、内存占用率等，这些文件也是 top 指令查看系统信息的主要数据来源。

如果在容器里执行 top 指令，就会发现，它显示的信息居然是宿主机的 CPU 和内存数据，而不是当前容器的数据。

造成这个问题的原因就是，`/proc` 文件系统并不知道用户通过 Cgroups 给这个容器做了什么样的资源限制，即：`/proc` 文件系统不了解 Cgroups 限制的存在。

1. 使用 lxcfs 解决，把宿主机的 `/var/lib/lxcfs/proc/memoinfo` 文件挂载到Docker容器的 `/proc/meminfo` 位置后。
2. Linux 4.6 有 cgroup namespace

