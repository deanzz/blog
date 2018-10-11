# 1.Kubernetes系列-容器的本质

>Kubernetes是当今最流行的容器编排引擎，由Google开源。
如果你想更好的理解容器编排，就要先深入理解一下容器技术，下面咱们就来讲讲容器的本质。
注意，这个系列的文章都是基于Linux的。

## 虚拟机
经常会有人将虚拟机和容器相混淆，所以这里有必要聊一下虚拟机与容器之间的区别，从而可以让大家更加理解容器的本质。  
虚拟机（Virtual Machine）指通过软件模拟的具有完整硬件系统功能的、运行在一个完全隔离环境中的完整计算机系统，  
既然是一个完整的计算机系统，那么它会拥有自己独立的操作系统、cpu、内存、磁盘、网络等资源以及独立操作系统内核，为什么要强调内核，后面会说到。  
所以，虚拟机的框架图是下面这样的，  

<img src="https://github.com/deanzz/blog/blob/master/k8s/img/vm.png?raw=true" width="50%" height="50%" />

其中Server是服务器硬件；  
Host OS是宿主机操作系统；  
Hypervisor是虚拟机器监视器，是用来建立与执行虚拟机器的软件、固件或硬件；  
Guest OS是虚拟机操作系统，之上的App就是运行在虚拟机中的服务。  

## 容器
现在来说说容器，如果让大家实现一个容器，大家会想到需要实现什么？  
很容易想到的一点就是近似虚拟机一样的环境隔离，  
即在容器里只能看到自己的世界，比如说，容器只能看到自己的文件目录、主机名、进程列表等等。  
还有一点就是资源限制，  
即容器只能使用分配给自己的cpu、内存等资源。  

那么容器到底是如何实现的呢？  
咱们还是从进程说起，进程是什么呢？  
概括来讲，进程就是一个程序运行起来后的计算机执行环境的总和，  
对于进程来说，程序就是它的静态表现，一旦程序运行起来，它就变成了计算机里的数据和状态，这就是它的动态表现。  

容器技术的核心功能，就是通过约束和修改进程的动态表现，从而为其创造出一个边界，这个边界不仅是隔离，还有限制。  

对于Docker这样的Linux容器，  
它使用Namespace技术来修改进程的视图，达到环境隔离的目的；  
它使用Cgroups技术对进程使用的资源进行约束，达到资源限制的目的。  

那么什么是Namespace技术呢？  
它其实只是Linux创建新进程的一个可选参数，最常用的就是创建线程的系统调用clone(),  
```
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```
其中第3个参数flags，就是可以设置Namespace参数的地方。  
Linux提供了PID、Mount、UTS、IPC、Network和User这些Namespace，具体请见下表， 
 
Namespace|系统调用参数|隔离内容
:----|:---------|:-------
UTS|CLONE_NEWUTS|主机名与域名
IPC|CLONE_NEWIPC|信号量、消息队列和共享内存
PID|CLONE_NEWPID|进程编号
Network|CLONE_NEWNET|网络设备、网络栈、端口等等
Mount|CLONE_NEWNS|挂载点（文件系统）
User|CLONE_NEWUSER|用户和用户组

通过添加上面提到的参数，你创建的进程就会只看到自己的世界，  
比如执行下面的代码，新建进程就会看到自己的PID是1，和自己设置的主机名。
```
int pid = clone(container_main, STACK_SIZE,
            CLONE_NEWUTS | CLONE_NEWPID | SIGCHLD, NULL);
```

那么什么是Cgroups技术呢？  
Linux Cgroups的全称是Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括CPU、内存、磁盘、网络带宽等等。  
在Linux中，Cgroups给用户提供的操作接口是文件系统，即它以文件和目录的方式组织在操作系统的/sys/fs/cgroup路径下。  
在Ubuntu 16.10上，我们可以用mount命令查看，  
```bash
sa@hadoop4:~$ mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
```
具体看一下cpu的目录，  
```bash
sa@hadoop4:/sys/fs/cgroup/cpu$ ls container/
cgroup.clone_children  cpu.cfs_period_us  cpu.shares  cpuacct.stat   cpuacct.usage_all     cpuacct.usage_percpu_sys   cpuacct.usage_sys   notify_on_release
cgroup.procs           cpu.cfs_quota_us   cpu.stat    cpuacct.usage  cpuacct.usage_percpu  cpuacct.usage_percpu_user  cpuacct.usage_user  tasks
```
其中tasks文件中记录的是需要被进行资源限制的进程号，其他文件是设置各种资源上限的参数，这样操作系统就知道对于tasks文件中的进程，该如何进行资源限制了。  

针对Namespace技术和Cgroups技术，后续文章会有更详细的讲解。  

所以，现在大家应该能回答"容器的本质是什么"这个问题了，  

<b>容器，其实就是一种特殊的进程。</b>  

所以容器的框架图是下面这样的  

<img src="https://github.com/deanzz/blog/blob/master/k8s/img/container.png?raw=true" width="50%" height="50%" />

到目前为止，大家应该对容器的本质有了一个更清晰的认识。  

## 容器和虚拟机的优势与劣势
虚拟机和容器都具有的优势就是深入到操作系统级别的运行环境的一致性。  
但是使用虚拟机作为应用的沙盒，就必须要由Hypervisor来负责创建虚拟机，这个虚拟机是真实存在的，并且它里面必须运行一个完整的Guest OS才能执行用户的应用进程，  
这就不可避免地带来了额外的资源损耗和占用，  
损耗表现在运行在虚拟机里面的用户进程，在对宿主机操作系统的调用时，就不可避免地要经过虚拟化软件的拦截和处理，导致对计算资源、网络和磁盘IO的性能损耗；  
占用表现在虚拟机自己的资源使用，比如虚拟机本身就需要200M的内存。  

相比之下，容器化后的用户应用，只是宿主机上的一个普通进程，这就意味着虚拟机带来的额外资源损耗和占用是不存在的。  
但是容器也有缺点，基于Linux Namespace的隔离机制相比于虚拟机技术也有一些不足之处，其中最主要的问题就是，隔离不彻底。  
首先，运行在同一宿主机的多个容器之间使用的都是宿主机操作系统的内核，这意味着，如果你想在低版本的Linux宿主机上运行高版本的Linux容器，是行不通的。  
其次，在Linux内核中，有很多资源和对象是不能被Namespace化的，比如说时间，  
如果你在容器中的程序使用settimeofday()系统调用修改了时间，那么整个宿主机的时间都会被修改。  
但是对于虚拟机，这些都不是问题，你可以在虚拟机中随便折腾，也不会影响宿主机。  

## 总结
总结一下这篇文章的重点，大家可以巩固一下，  
1. 了解虚拟机与容器的本质区别  
    虚拟机是由Hypervisor创建的运行一个完整Guest OS，里面运行着用户的应用进程；  
    容器，其实就是宿主机上一个特殊的进程。  
2. Linux下的容器是通过Namespace技术和Cgroups技术实现的  
3. 容器和虚拟机的优势与劣势  
    虚拟机的优势就是隔离的很彻底，劣势就是比较重量级，并且存在额外的资源损耗和占用；  
    容器的优势就是轻量级，而且没有额外的资源损耗和占用，劣势就是隔离不彻底。  

