# 3.Kubernetes系列-Linux Cgroup

>上一篇文章讲了Linux namespace技术是如何实现隔离容器环境的，
这一篇咱们来聊聊如何使用Linux Cgroup技术实现容器资源的限制。  
接着上篇文章中写的"山寨版docker容器"，咱们在容器中运行几个占用相关资源的进程来测试一下cgroup的资源限制，带你完成你自己的山寨docker。

<b>咱们先看看cgroup都提供了哪些资源的限制，在cgroup中他们被称作子系统。</b>    
blkio    — 这​​​个​​​子​​​系​​​统​​​为​​​块​​​设​​​备​​​设​​​定​​​输​​​入​​​/输​​​出​​​限​​​制​​​，比​​​如​​​物​​​理​​​设​​​备​​​（磁​​​盘​​​，固​​​态​​​硬​​​盘​​​，USB 等​​​等​​​）。  
cpu      — 这​​​个​​​子​​​系​​​统​​​使​​​用​​​调​​​度​​​程​​​序​​​提​​​供​​​对​​​ CPU 的​​​ cgroup 任​​​务​​​访​​​问​​​。​​​  
cpuacct  — 这​​​个​​​子​​​系​​​统​​​自​​​动​​​生​​​成​​​ cgroup 中​​​任​​​务​​​所​​​使​​​用​​​的​​​ CPU 报​​​告​​​。  ​​​
cpuset   — 这​​​个​​​子​​​系​​​统​​​为​​​ cgroup 中​​​的​​​任​​​务​​​分​​​配​​​独​​​立​​​ CPU（在​​​多​​​核​​​系​​​统​​​）和​​​内​​​存​​​节​​​点​​​。​​​  
devices  — 这​​​个​​​子​​​系​​​统​​​可​​​允​​​许​​​或​​​者​​​拒​​​绝​​​ cgroup 中​​​的​​​任​​​务​​​访​​​问​​​设​​​备​​​。​​​  
freezer  — 这​​​个​​​子​​​系​​​统​​​挂​​​起​​​或​​​者​​​恢​​​复​​​ cgroup 中​​​的​​​任​​​务​​​。​​​  
memory   — 这​​​个​​​子​​​系​​​统​​​设​​​定​​​ cgroup 中​​​任​​​务​​​使​​​用​​​的​​​内​​​存​​​限​​​制​​​，并​​​自​​​动​​​生​​​成​​​​​内​​​存​​​资​​​源使用​​​报​​​告​​​。​​​  
net_cls  — 这​​​个​​​子​​​系​​​统​​​使​​​用​​​等​​​级​​​识​​​别​​​符​​​（classid）标​​​记​​​网​​​络​​​数​​​据​​​包​​​，可​​​允​​​许​​​ Linux 流​​​量​​​控​​​制​​​程​​​序​​​（tc）识​​​别​​​从​​​具​​​体​​​ cgroup 中​​​生​​​成​​​的​​​数​​​据​​​包​​​。  ​​​
net_prio — 这个子系统用来设计网络流量的优先级   
hugetlb  — 这个子系统主要针对于HugeTLB系统进行限制，这是一个大页文件系统。   

<b>再来看看cgroup的特点：</b>   
1. cgroup的API以一个伪文件系统的方式实现，即用户可以通过文件操作实现cgroup的组织管理。  
2. cgroup的组织管理操作单元可以细粒度到线程级别，用户态代码也可以针对系统分配的资源创建和销毁cgroup，从而实现资源再分配和管理。  
3. 所有资源管理的功能都以“subsystem（子系统）”的方式实现，接口统一。  
4. 子进程创建之初与其父进程处于同一个cgroup的控制组。  

咱们今天主要实现最常用的4种资源的限制，`CPU时间`、`CPU使用的核`、`内存`和`磁盘IO`。   

## 容器中的测试进程
首先咱们需要写4个测试程序，分别是测试`限制CPU时间`、`限制CPU使用的核`、`限制内存`和`限制IO`的程序。  
### 限制CPU时间
就是空跑一个无限循环。
cpu_time.c:  
```
#include <stdio.h>
int main()
{
    while(1){}
    return 0;
}
```
### 限制CPU使用的核
启动n个线程，每个线程跑一个无限循环。
cpu_core.c:  
```
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/syscall.h>

const int NUM_THREADS = 5;

void *thread_main(void *threadid)
{
    long tid;
    tid = (long)threadid;
    printf("Hello World! It's me, thread #%ld, pid #%ld!\n", tid, syscall(SYS_gettid));
    while(1) {
    }
    pthread_exit(NULL);
}

int main (int argc, char *argv[])
{
    int num_threads;
    if (argc > 1){
        num_threads = atoi(argv[1]);
    }
    if (num_threads<=0 || num_threads>=100){
        num_threads = NUM_THREADS;
    }
    pthread_t* threads = (pthread_t*) malloc (sizeof(pthread_t)*num_threads);
    int rc;
    long t;
    for(t=0; t<num_threads; t++){
        printf("In main: creating thread %ld\n", t);
        rc = pthread_create(&threads[t], NULL, thread_main, (void *)t);
        if (rc){
            printf("ERROR; return code from pthread_create() is %d\n", rc);
            exit(-1);
        }
    }
    /* Last thing that main() should do */
    pthread_exit(NULL);
    free(threads);
}
```

### 限制内存
在一个无限循环里，每隔1秒，申请1M的内存。
mem.c:  
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <unistd.h>
#define MB (1024 * 1024)
int main(int argc, char *argv[])
{
    char *p;
    int i = 0;
    while(1) {
        p = (char *)malloc(MB);
        memset(p, 0, MB);
        printf("%dM memory allocated\n", ++i);
        sleep(1);
    }
    return 0;
}
``` 

### 限制IO
dd命令，读入if指定的文件，输出到/dev/null
```
dd if=/data/cn_windows_7_ultimate_with_sp1_x64_dvd_u_677408.iso of=/dev/null
```

## 创建cgroup层级
首先咱们要在宿主机上创建一个cgroup的层级，只限制咱们期望的进程，这里起名叫dean。使用如下命令
```bash
# 使用apt-get install cgroup-bin安装相关命令
sudo cgcreate -a sa:sa -t sa:sa -g cpu,cpuset,memory,blkio:dean
```
这里我们为cpu、cpuset、memory和blkio增加了一个新的层级dean，并且这个层级是sa用户有权限操作的。
此时你可以查看/sys/fs/cgroup/目录下cpu、cpuset、memory和blkio子系统下，分别会生成一个叫dean的文件夹。

## 限制CPU时间
首先我们编译测试程序cpu_time.c，将编译好的文件cpu_time复制到上一篇文章中准备好的rootfs的/app目录下。
```bash
sa@hadoop4:/srv/chroot/xenial_amd64/app$ ls
cpu_time
```
然后启动上一篇文章写好的容器进程bns，并且进入/app目录，执行cpu_time程序。  
```bash
sa@hadoop4:~/c$ ./bns
Parent - start a container!
Parent: eUID = 1000;  eGID = 1000, UID=1000, GID=1000
Container [    1] - inside the container!
Container: eUID = 0;  eGID = 0, UID=0, GID=0
root@dean:/# cd app/
root@dean:/app# ls
cpu_time
root@dean:/app# ./cpu_time


```
新开一个终端窗口，使用top命令查看，此时使用top命令查看cpu_time进程占用100%的cpu时间。  
```bash
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 4364 sa        20   0    4224    628    556 R 100.0  0.0   0:13.22 cpu_time
```
cgroup登场，再新开一个终端窗口，ls一下cgroup的cpu子系统的目录，  
```bash
sa@hadoop4:/sys/fs/cgroup/cpu$ ls
cgroup.clone_children  cpu.cfs_period_us  cpu.stat       cpuacct.usage_all         cpuacct.usage_percpu_user  dean               release_agent  user.slice
cgroup.procs           cpu.cfs_quota_us   cpuacct.stat   cpuacct.usage_percpu      cpuacct.usage_sys          init.scope         system.slice
cgroup.sane_behavior   cpu.shares         cpuacct.usage  cpuacct.usage_percpu_sys  cpuacct.usage_user         notify_on_release  tasks
```
其中需要设置的是cpu.cfs_quota_us，它表示在单位周期时间（cpu.cfs_period_us）内cpu的执行时间是多少微秒，
周期时间cpu.cfs_period_us默认为100000微妙，所以如果你想让进程只占用10%的cpu时间，就设置cpu.cfs_quota_us为10000微妙。
```bash
echo 10000 > dean/cpu.cfs_quota_us
```
接下来就要把你的容器进程的pid写入tasks文件，先"ps -ef"一下  
```bash
sa@hadoop4:/sys/fs/cgroup/cpu$ ps -ef
...
sa        4336  3166  0 12:20 pts/1    00:00:00 ./bns
sa        4337  4336  0 12:20 pts/1    00:00:00 /bin/bash
...
```
可以看到bns的进程，但是真正执行容器里测试进程的是在bns进程中启动的bash进程，所以tasks文件中要填写bash的进程号。
```bash
echo 4337 > dean/tasks
```
切换到容器的终端窗口，重新启动cpu_time进程
```bash
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 4608 sa        20   0    4224    720    644 R  10.0  0.0   0:03.24 cpu_time
```
可以看到cpu_time进程此时只占用了预期的10%CPU时间。

## 限制CPU使用的核
首先我们来编译cpu_core.c，因为会用到线程库，需要使用如下命令编译
```bash
gcc -o cpu_core cpu_core.c -lpthread
```
将编译好的文件cpu_core复制到上一篇文章中准备好的rootfs的/app目录下。  
然后启动上一篇文章写好的容器进程bns，并且进入/app目录，执行cpu_core程序，开启了5个线程跑无限循环。  
```bash
sa@hadoop4:~/c$ ./bns
Parent - start a container!
Parent: eUID = 1000;  eGID = 1000, UID=1000, GID=1000
Container [    1] - inside the container!
Container: eUID = 0;  eGID = 0, UID=0, GID=0
root@dean:/# cd app
root@dean:/app# ./cpu_core
In main: creating thread 0
In main: creating thread 1
In main: creating thread 2
In main: creating thread 3
In main: creating thread 4
Hello World! It's me, thread #1, pid #7!
Hello World! It's me, thread #0, pid #6!
Hello World! It's me, thread #2, pid #8!
Hello World! It's me, thread #3, pid #9!
Hello World! It's me, thread #4, pid #10!

```  
看到在没限制的情况下，它会使用其父级的配置。
```bash
sa@hadoop4:/sys/fs/cgroup/cpuset/dean$ cd ..
sa@hadoop4:/sys/fs/cgroup/cpuset$ cat cpuset.cpus
0-1
sa@hadoop4:/sys/fs/cgroup/cpuset$ cat cpuset.mems
0
```
于是进程就跑满了宿主机的两核。  
```bash
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 6527 sa        20   0       0      0      0 Z 199.7  0.0   0:32.30 cpu_core
```
并且使用taskset查看进程默认使用的是cpu是所有核，也就是0和1这两个核。  
```bash
sa@hadoop4:~/c$ taskset -c -p 6527
pid 6527's current affinity list: 0,1
```
cgroup登场，再新开一个终端窗口，ls一下cgroup的cpuset子系统的目录，
```bash
sa@hadoop4:/sys/fs/cgroup/cpuset/dean$ ls
cgroup.clone_children  cpuset.cpus            cpuset.mem_exclusive   cpuset.memory_pressure     cpuset.mems                      notify_on_release
cgroup.procs           cpuset.effective_cpus  cpuset.mem_hardwall    cpuset.memory_spread_page  cpuset.sched_load_balance        tasks
cpuset.cpu_exclusive   cpuset.effective_mems  cpuset.memory_migrate  cpuset.memory_spread_slab  cpuset.sched_relax_domain_level
```
这里需要设置的是cpuset.mems和cpuset.cpus，  
cpuset.mems是用来配置cpu使用的内存节点的，要最先设置，否则后续设置会报错。
cpuset.cpus是用来配置使用哪些核。
```bash
echo 0 > dean/cpuset.mems
echo 1 > dean/cpuset.cpus
```
我们设置只能使用第1号cpu核，最后别忘了设置将容器的bash进程编号写入tasks文件。  
接着回到容器中再次执行cpu_core，   
```bash
root@dean:/app# ./cpu_core
In main: creating thread 0
In main: creating thread 1
In main: creating thread 2
In main: creating thread 3
In main: creating thread 4
Hello World! It's me, thread #4, pid #18!
Hello World! It's me, thread #3, pid #17!
Hello World! It's me, thread #2, pid #16!
Hello World! It's me, thread #1, pid #15!
Hello World! It's me, thread #0, pid #14!

```
再次使用top命令查看，现在只会占满宿主机一个cpu核，
```bash
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 6834 sa        20   0       0      0      0 Z 100.0  0.0   0:44.71 cpu_core
```
最后来看看进程占用的是哪个cpu核，正是我们设置的第1号cpu核。
```bash
sa@hadoop4:/etc$ taskset -c -p 6834
pid 6834's current affinity list: 1
```
## 限制内存
首先我们编译测试程序mem.c，将编译好的文件mem复制到上一篇文章中准备好的rootfs的/app目录下,  
然后启动上一篇文章写好的容器进程bns，这次先设置内存限制，再启动测试程序。  
cgroup登场，再新开一个终端窗口，ls一下cgroup的memory子系统的目录，  
```bash
sa@hadoop4:/sys/fs/cgroup/memory$ ls
cgroup.clone_children  memory.force_empty              memory.kmem.tcp.max_usage_in_bytes  memory.oom_control          notify_on_release
cgroup.event_control   memory.kmem.failcnt             memory.kmem.tcp.usage_in_bytes      memory.pressure_level       release_agent
cgroup.procs           memory.kmem.limit_in_bytes      memory.kmem.usage_in_bytes          memory.soft_limit_in_bytes  system.slice
cgroup.sane_behavior   memory.kmem.max_usage_in_bytes  memory.limit_in_bytes               memory.stat                 tasks
dean                   memory.kmem.slabinfo            memory.max_usage_in_bytes           memory.swappiness           user.slice
init.scope             memory.kmem.tcp.failcnt         memory.move_charge_at_immigrate     memory.usage_in_bytes
memory.failcnt         memory.kmem.tcp.limit_in_bytes  memory.numa_stat                    memory.use_hierarchy
```
其中需要设置的是memory.swappiness和memory.limit_in_bytes，  
memory.swappiness是使用的swap空间的大小，如果这个不设置为0，不太容易看出memory.limit_in_bytes设置的效果，因为当内存不足时，内核会将内存中的内容移动到swap空间。  
memory.limit_in_bytes就是使用的内存大小了。  
```bash
echo 0 > dean/memory.swappiness
echo 10M > dean/memory.limit_in_bytes
```
这里设置了10M的内存限制，因为memory.oom_control中设置的内存溢出的行为是默认的杀死进程，所以待会儿你会看到mem进程在达到10M内存的时候被kill，  
接着将容器的bash进程号写入dean/tasks文件。  
下面在容器中执行mem，
```bash
sa@hadoop4:~/c$ ./bns
Parent - start a container!
Parent: eUID = 1000;  eGID = 1000, UID=1000, GID=1000
Container [    1] - inside the container!
Container: eUID = 0;  eGID = 0, UID=0, GID=0
root@dean:/# cd app/
root@dean:/app# ./mem
1M memory allocated
2M memory allocated
3M memory allocated
4M memory allocated
5M memory allocated
6M memory allocated
7M memory allocated
8M memory allocated
9M memory allocated
Killed
root@dean:/app#
```  
大家可以看到进程在达到10M内存的时候被kill掉了，符合预期。  

## 限制IO
首先你得准备一个大点儿的测试文件，比如我的就是个windows7的安装文件，近4g大小。  
我们会使用dd命令测试读取的IO速度，从测试文件中读取内容，输出到/dev/null，然后使用iotop命令监测读取的IO速度。
首先先测试一下不使用cgroup限制的情况，
```bash
root@dean:/app# dd if=/data/cn_windows_7_ultimate_with_sp1_x64_dvd_u_677408.iso of=/dev/null
6680776+0 records in
6680776+0 records out
3420557312 bytes (3.4 GB, 3.2 GiB) copied, 3.86763 s, 884 MB/s
```
执行很快，都来不及复制粘贴iotop的内容就完成了，速度是884 MB/s。
cgroup登场，再新开一个终端窗口，ls一下cgroup的blkio子系统的目录，
```bash
sa@hadoop4:/sys/fs/cgroup/blkio/dean$ ls
blkio.io_merged                   blkio.io_service_time_recursive  blkio.reset_stats                blkio.throttle.write_bps_device   cgroup.procs
blkio.io_merged_recursive         blkio.io_serviced                blkio.sectors                    blkio.throttle.write_iops_device  notify_on_release
blkio.io_queued                   blkio.io_serviced_recursive      blkio.sectors_recursive          blkio.time                        tasks
blkio.io_queued_recursive         blkio.io_wait_time               blkio.throttle.io_service_bytes  blkio.time_recursive
blkio.io_service_bytes            blkio.io_wait_time_recursive     blkio.throttle.io_serviced       blkio.weight
blkio.io_service_bytes_recursive  blkio.leaf_weight                blkio.throttle.read_bps_device   blkio.weight_device
blkio.io_service_time             blkio.leaf_weight_device         blkio.throttle.read_iops_device  cgroup.clone_children
```
其中需要设置的是blkio.throttle.read_bps_device，其格式为{major}:{minor} {size}。  
其中{major}:{minor}为主次设备号，可以通过如下方式找到， 
```bash
# 找到期望的磁盘，我这里选的是/对应的磁盘
sa@hadoop4:/sys/fs/cgroup/blkio/dean$ df -h
Filesystem                    Size  Used Avail Use% Mounted on
udev                          1.5G     0  1.5G   0% /dev
tmpfs                         294M  4.8M  290M   2% /run
/dev/mapper/hadoop1--vg-root   59G  7.6G   48G  14% /
tmpfs                         1.5G     0  1.5G   0% /dev/shm
tmpfs                         5.0M     0  5.0M   0% /run/lock
tmpfs                         1.5G     0  1.5G   0% /sys/fs/cgroup
/dev/sda1                     472M   61M  387M  14% /boot
tmpfs                         294M     0  294M   0% /run/user/1000
# 查找其对应的主次设备号
sa@hadoop4:/sys/fs/cgroup/blkio/dean$ ls -l /dev/mapper/hadoop1--vg-root
lrwxrwxrwx 1 root root 7 Nov  9 16:31 /dev/mapper/hadoop1--vg-root -> ../dm-0
sa@hadoop4:/sys/fs/cgroup/blkio/dean$ ls -l /dev/dm-0
brw-rw---- 1 root disk 252, 0 Nov  9 16:31 /dev/dm-0
```
其中252就是主设备号，0就是次设备号。  
现在来设置其读取的IO速度，
```bash
echo "252:0 10485760" > dean/blkio.throttle.read_bps_device
```
这里设置的磁盘IO速度为每秒10M。  
最后别忘了设置将容器的bash进程编号写入tasks文件。
下面再次在容器内执行测试命令，并开一个新终端窗口，查看iotop，   
```bash
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 7369 be/4 sa          9.87 M/s    0.00 B/s  0.00 % 95.88 % dd if=/data/cn_windows_7_ultimate_with_sp1_x64_dvd_u_677408.iso of=/dev/null
```
可以看到磁盘读取速度被限制在了10M/s，符合预期。

## 总结
今天我带大家使用上一篇完成一半的山寨docker容器，使用cgroup对`CPU时间`、`CPU使用的核`、`内存`和`磁盘IO`的添加了资源的限制，  
最终完成了咱们自己的山寨docker容器。此时你应该对容器技术的认识已经比较深入了，  
所以大家应该不难总结出docker实现容器最核心的原理就是为待创建的用户进程：  
1. 启用Linux Namespace配置  
2. 设置指定的Cgroup参数  
3. 切换进程的根目录到特定的rootfs  



