# 2.Kubernetes系列-Linux Namespace

>上一篇文章讲了容器的本质，即容器是一个特殊的进程，
还有Linux下的容器主要是通过Namespace技术和Cgroups技术实现的，
下面我们就来详细聊聊Linux的Namespace技术。

Linux的Namespace技术是Linux内核提供的一种环境隔离的机制，它能保证每个namespace下的资源对于其他namespace下的资源都是透明，不可见的。
下面我们就依照下表来具体看看每个namespace，带大家通过系统调用clone去实现一下每个namespace，最终可以模拟出一个"山寨的docker容器"。

Namespace|系统调用参数|隔离内容
:----|:---------|:-------
UTS|CLONE_NEWUTS|主机名与域名
IPC|CLONE_NEWIPC|信号量、消息队列和共享内存
PID|CLONE_NEWPID|进程编号
Mount|CLONE_NEWNS|挂载点（文件系统）
User|CLONE_NEWUSER|用户和用户组
Network|CLONE_NEWNET|网络设备、网络栈、端口等等


## 隔离主机名与域名
直接上测试的C语言代码，将其写入bns.c，其中有两个函数，  
一个是container_main，是容器进程（clone系统调用）执行的代码，我们通过一些namespace设置后，让其执行"/bin/bash"命令，这样你就能进入容器内部，并且让父进程等待容器进程退出；      
一个是main，是主进程的入口函数。  
根据上面表格说的，隔离主机名和域名需要的参数是CLONE_NEWUTS，于是咱们就在clone系统调用的第3个参数位置，添加这个参数，  
当然你还得调用另一个系统调用sethostname，去修改容器进程的主机名，在这里我设置成了我的英文名"dean"。  
```
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/mount.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
#include <stdlib.h>

#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];

char* const container_args[] = {
    "/bin/bash",
    NULL
};

int container_main(void* arg)
{
    printf("Container [%5d] - inside the container!\n", getpid());
    sethostname("deanzz",10);
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main()
{
    printf("Parent - start a container!\n");
    int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWUTS | SIGCHLD, NULL);
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```
然后编译上面代码，  
```bash
gcc -o bns bns.c
```
接着执行，这里需要root权限，执行结果如下，  
```bash
sa@hadoop4:~/c$ sudo ./bns
Parent - start a container!
Container [ 2352] - inside the container!
root@dean:~/c#
root@dean:~/c# uname -n
dean
root@dean:~/c#
```
大家可以看到宿主机的用户是sa，主机名是hadoop4，但是容器进程内的用户已经变成了root（sudo的结果），主机名已经变成了咱们设置的"dean"，  
使用"uname -n"命令也能查看当前进程的主机名。  
最后咱们执行"exit"，退出容器进程，接着看下面的namespace。

## 隔离信号量、消息队列和共享内存
IPC全称 Inter-Process Communication，是Unix/Linux下进程间通信的一种方式，IPC有共享内存、信号量、消息队列等方法。  
所以，为了隔离，我们也需要把IPC给隔离开来，这样，只有在同一个Namespace下的进程才能相互通信。  

要启动IPC隔离，我们只需要在调用clone时加上CLONE_NEWIPC参数就可以了。
```
int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWUTS | CLONE_NEWIPC | SIGCHLD, NULL);
```
为了验证IPC隔离，我们首先在宿主机上创建一个Queue，
```bash
sa@hadoop4:~/c$ ipcmk -Q
Message queue id: 0
sa@hadoop4:~/c$ ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x2eaae7b3 0          sa         644        0            0

sa@hadoop4:~/c$
```
然后编译代码，使用root权限执行，执行结果如下，
```bash
sa@hadoop4:~/c$ sudo ./bns
Parent - start a container!
Container [ 2421] - inside the container!
root@dean:~/c# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages

root@dean:~/c#
```
此时，我们在容器进程里，是看不到你在宿主机创建的Queue的，IPC已经被隔离了。

## 隔离进程编号
要启动PID隔离，我们同样只需在调用clone时，加上CLONE_NEWPID参数就可以了。
```
int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWIPC | SIGCHLD, NULL);
```
为了看到效果，我们修改一下container_main的第一行日志，查看容器进程的pid，  
```
printf("Container [%5d] - inside the container!\n", getpid());
```
然后编译代码，使用root权限执行，执行结果如下，
```bash
sa@hadoop4:~/c$ sudo ./bns
Parent - start a container!
Container [    1] - inside the container!
root@dean:~/c# echo $$
1
root@dean:~/c#
```
我们通过日志，看到当前进程的pid是1，再通过"echo $$"查看当前shell的进程号，同样是1，达到了pid隔离的目的。  
但是此时我们执行一下"ps -A"命令，发现展示的进程还是宿主机上的进程，并不是容器内的进程，比如1是systemd进程，而不是当前的shell进程，  
```bash
root@dean:~/c# ps -A
  PID TTY          TIME CMD
    1 ?        00:00:01 systemd
    2 ?        00:00:00 kthreadd
    3 ?        00:00:00 ksoftirqd/0
    5 ?        00:00:00 kworker/0:0H
    7 ?        00:00:00 rcu_sched
    8 ?        00:00:00 rcu_bh
    ...
```
这是因为，像ps、top这些命令会去读/proc文件系统，所以，因为/proc文件系统在父进程和子进程都是一样的，所以这些命令显示的东西都是一样的。

## 隔离挂载点（文件系统）
要启用Mount隔离，首先要在调用clone时，添加"CLONE_NEWNS"参数，  
```
int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWIPC | SIGCHLD, NULL);
```
然后在container_main中，执行"/bin/bash"命令之前，重新挂载proc文件系统到/proc目录下，  
```
if (mount("proc", "/proc", "proc", 0, NULL) !=0 ) {
        perror("proc");
}
```
然后编译代码，使用root权限执行，执行结果如下，
```bash
sa@hadoop4:~/c$ sudo ./bns
Parent - start a container!
Container [    1] - inside the container!
root@dean:~/c# ps -A
  PID TTY          TIME CMD
    1 pts/0    00:00:00 bash
   13 pts/0    00:00:00 ps
root@dean:~/c# top
top - 12:22:40 up  2:09,  1 user,  load average: 0.00, 0.00, 0.00
Tasks:   2 total,   1 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni, 99.9 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  3009512 total,  2460924 free,    95012 used,   453576 buff/cache
KiB Swap:  4190204 total,  4190204 free,        0 used.  2693936 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    1 root      20   0   22572   5080   3212 S   0.0  0.2   0:00.02 bash
   14 root      20   0   41900   3568   3080 R   0.0  0.1   0:00.00 top
```
现在可以看到容器进程内的ps和top命令的结果都是容器内部的了，与宿主机隔离。  
但是当你退出容器进程后，你在宿主机上执行"ps -A"时，会有如下提示，
```bash
sa@hadoop4:~/c$ ps -A
Error, do this: mount -t proc proc /proc
sa@hadoop4:~/c$
```
这是因为容器进程在挂载自己的proc虚拟文件系统时，挂载点是/proc文件目录，  
又因为容器进程此时看到的文件系统与父进程的一样（没有为容器进程提供独立的rootfs）， 
所以  
在容器进程执行前，挂载点/proc目录挂载的是父进程的proc虚拟文件系统，  
容器进程运行后，挂载点/proc目录挂载的是容器进程的proc虚拟文件系统，  
当容器进程结束后，挂载点/proc目录挂载的容器进程的proc虚拟文件系统就消失了，  
所以需要你重新将挂载点/proc目录挂载到宿主机的proc虚拟文件系统。（自己的拙见，也许不完全正确）

## 切换容器根目录rootfs
上面我们在使用上面Mount namespace时遇到了问题，即文件目录没有被隔离，  
于是我们就会想到那就把所有需要的目录重新挂载一遍，不过Mount namespace对容器进程视图的改变，一定是伴随着挂载操作(mount)才能生效，
可是我们期望的是一个更友好的情况：每当创建一个新容器时，都希望容器进程看到的文件系统就是一个独立的隔离环境，而不是继承自宿主机的文件系统。  
为了达到这个目的，Linux提供了一个名叫chroot的命令完成这个工作，它的作用就是改变进程的根目录到你指定的位置。  

首先我们先按照Ubuntu提供的教程[DebootstrapChroot](https://help.ubuntu.com/community/DebootstrapChroot)，  
制作一个64位Ubuntu 16.04 LTS (Xenial Xerus)版本的chroot目录，做完的目录只有247M，里面包含了根目录下所有必须的内容。
当然你也可以手动复制你需要的命令、动态链接库和配置文件...，但是很麻烦，也容易出错和遗漏。  

我们来看一下做好的chroot目录，和你宿主机上的根目录其实一模一样。  
```bash
sa@hadoop4:/srv/chroot/xenial_amd64$ ll
total 84
drwxr-xr-x 21 root root 4096 Oct 27 18:58 ./
drwxr-xr-x  3 root root 4096 Oct 27 18:43 ../
drwxr-xr-x  2 root root 4096 Oct 27 18:58 bin/
drwxr-xr-x  2 root root 4096 Apr 13  2016 boot/
drwxr-xr-x  4 root root 4096 Oct 27 18:58 dev/
drwxr-xr-x 63 root root 4096 Oct 27 18:58 etc/
drwxr-xr-x  2 root root 4096 Apr 13  2016 home/
drwxr-xr-x 11 root root 4096 Oct 27 18:58 lib/
drwxr-xr-x  2 root root 4096 Oct 27 18:58 lib64/
drwxr-xr-x  2 root root 4096 Oct 27 18:58 media/
drwxr-xr-x  2 root root 4096 Oct 27 18:58 mnt/
drwxr-xr-x  2 root root 4096 Oct 27 18:58 opt/
drwxr-xr-x  2 root root 4096 Apr 13  2016 proc/
drwx------  2 root root 4096 Oct 27 18:58 root/
drwxr-xr-x  6 root root 4096 Oct 27 18:58 run/
drwxr-xr-x  2 root root 4096 Oct 27 18:58 sbin/
drwxr-xr-x  2 root root 4096 Oct 27 18:58 srv/
drwxr-xr-x  2 root root 4096 Feb  5  2016 sys/
drwxrwxrwt  2 root root 4096 Oct 27 21:35 tmp/
drwxr-xr-x 10 root root 4096 Oct 27 18:58 usr/
drwxr-xr-x 11 root root 4096 Oct 27 18:58 var/
```
下面我们就来切换容器进程的根目录，然后再mount /proc，
在container_main中添加如下代码即可，
```
if ( chdir("/srv/chroot/xenial_amd64") != 0 || chroot("./") != 0 ){
        perror("chdir/chroot");
}
```  
然后编译代码，使用root权限执行，执行结果如下，
```
sa@hadoop4:~/c$ sudo ./bns
Parent - start a container!
Container [    1] - inside the container!
root@dean:/# ls
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var
root@dean:/# ps -A
  PID TTY          TIME CMD
    1 ?        00:00:00 bash
    5 ?        00:00:00 ps
root@dean:/# top
top - 08:01:24 up  5:47,  0 users,  load average: 0.00, 0.00, 0.00
Tasks:   2 total,   1 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  3009512 total,  2429464 free,    58476 used,   521572 buff/cache
KiB Swap:  4190204 total,  4190204 free,        0 used.  2686268 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    1 root      20   0   18248   3268   2784 S   0.0  0.1   0:00.00 bash
    6 root      20   0   36676   3080   2616 R   0.0  0.1   0:00.00 top
```
你会发现容器进程内的目录变了，从原来的"~/c"变成了"/"，而且ls一下，发现就是chroot目录中的内容，你无法看到宿主机的其他目录了，    
ps和top命令也显示正常，接下来退出容器进程，在宿主机中执行"ps -A"
```bash
root@dean:/# exit
exit
Parent - container stopped!
sa@hadoop4:~/c$ ps -A
  PID TTY          TIME CMD
    1 ?        00:00:01 systemd
    2 ?        00:00:00 kthreadd
    3 ?        00:00:00 ksoftirqd/0
    5 ?        00:00:00 kworker/0:0H
    7 ?        00:00:00 rcu_sched
    8 ?        00:00:00 rcu_bh
    9 ?        00:00:00 migration/0
   10 ?        00:00:00 lru-add-drain
  ... 
```
一切OK，文件系统被彻底的隔离了。

## 隔离用户和用户组
User Namespace主要是用了CLONE_NEWUSER的参数。使用了这个参数后，内部看到的UID和GID已经与外部不同了，默认显示为65534。  
那是因为容器找不到其真正的UID所以，设置上了最大的UID。  

要把容器中的uid和真实系统的uid给映射在一起，需要修改/proc/<pid>/uid_map和/proc/<pid>/gid_map这两个文件。这两个文件的格式为：  
```
ID-inside-ns ID-outside-ns length
```
第一个字段ID-inside-ns表示在容器显示的UID或GID，  
第二个字段ID-outside-ns表示容器外映射的真实的UID或GID，  
第三个字段表示映射的范围，一般填1，表示一一对应。  

为什么需要将容器中的uid/gid和宿主机的uid/gid映射到一起呢，这是为了安全，  
在容器进程中，其可以有很高的权限，比如root，但是当其需要访问容器外的内容时，Linux就需要参考宿主机上的uid/gid看是否给其权限操作了，  
为了找到宿主机上的uid/gid，所以需要这样一个映射。

另外，需要注意的是：  
1. 写这两个文件的进程需要这个namespace中的CAP_SETUID (CAP_SETGID)权限。
2. 写入的进程必须是此user namespace的父或子的user namespace进程。
3. 另外需要满如下条件之一：  
   1）父进程将effective uid/gid映射到子进程的user namespace中，
   2）父进程如果有CAP_SETUID/CAP_SETGID权限，那么它将可以映射到父进程中的任一uid/gid。
   
接下来让我们来修改代码，主要就是clone调用添加了CLONE_NEWUSER参数，  
还有新增了3个方法：set_map、set_uid_map和set_gid_map，并在main函数中调用，  
将容器进程内的root与容器外的sa用户和用户组映射起来，还打印了一些显示uid/gid的日志。
```
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/mount.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
#include <stdlib.h>

#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];

char* const container_args[] = {
    "/bin/bash",
    NULL
};

void set_map(char* file, int inside_id, int outside_id, int len) {
    FILE* mapfd = fopen(file, "w");
    if (NULL == mapfd) {
        perror("open file error");
        return;
    }
    fprintf(mapfd, "%d %d %d", inside_id, outside_id, len);
    fclose(mapfd);
}

void set_uid_map(pid_t pid, int inside_id, int outside_id, int len) {
    char file[256];
    sprintf(file, "/proc/%d/uid_map", pid);
    set_map(file, inside_id, outside_id, len);
}

void set_gid_map(pid_t pid, int inside_id, int outside_id, int len) {
    char file[256];
    sprintf(file, "/proc/%d/gid_map", pid);
    set_map(file, inside_id, outside_id, len);
}

int container_main(void* arg)
{
    printf("Container [%5d] - inside the container!\n", getpid());
    printf("Container: eUID = %ld;  eGID = %ld, UID=%ld, GID=%ld\n",
            (long) geteuid(), (long) getegid(), (long) getuid(), (long) getgid());
    sethostname("dean",10);
    if ( chdir("/srv/chroot/xenial_amd64") != 0 || chroot("./") != 0 ){
        perror("chdir/chroot");
    }
    if (mount("proc", "/proc", "proc", 0, NULL) !=0 ) {
        perror("proc");
    }
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main()
{
    printf("Parent - start a container!\n");
    const int gid = getgid(), uid = getuid();
    printf("Parent: eUID = %ld;  eGID = %ld, UID=%ld, GID=%ld\n",
            (long) geteuid(), (long) getegid(), (long) getuid(), (long) getgid());
    int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWUSER | CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWIPC | SIGCHLD, NULL);

    set_uid_map(container_pid, 0, uid, 1);
    set_gid_map(container_pid, 0, gid, 1);

    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```
然后编译代码，这回就不需要root权限执行了，因为通过User namespace，我们已经修改了容器内的用户为root，执行结果如下，
```bash
sa@hadoop4:~/c$ ./bns
Parent - start a container!
Parent: eUID = 1000;  eGID = 1000, UID=1000, GID=1000
Container [    1] - inside the container!
Container: eUID = 0;  eGID = 0, UID=0, GID=0
root@dean:/# ls
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var
root@dean:/# id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
root@dean:/#
```
在宿主机上，没有使用sudo执行了./bns，但是在容器进程中用户和组已经切换到root了，并且使用id命令查看，uid和gid都为期望的0，  
于是用户和用户组也隔离完毕。

## 隔离网络设备、网络栈、端口等等
接下来是网络的隔离了，这个是相当啰嗦了，由于时间原因，我还没有完全搞明白，以后补充。
你现在可以先了解到是用ip命令实现的。  

## 总结
通过上面对每种namespace的讲解和实践，咱们的"山寨docker"容器其实已经完成了一半，大家最好跟着做一遍，会理解的更深刻。  
不过如果你要用以上的容器去执行一个程序，比如一个很耗费资源的程序，容器依然会抢占宿主机的资源，甚至占满宿主机的资源，  
这是因为没有对容器进程进行资源的限制。    
所以接下来就需要Cgroups技术去限制容器的资源，从而最终完成一个容器，    
下一篇我们来详细聊聊Cgroups技术。

