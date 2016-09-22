#Linux namespace技术简介
**原文链接：[http://coolshell.cn/articles/17010.html](http://coolshell.cn/articles/17010.html)**

时下最热的技术莫过于Docker了，很多人都觉得Docker是个新技术，其实不然，Docker除了其编程语言用go比较新外，其实它还真不是个新东西，也就是个新瓶装旧酒的东西，所谓的The New “Old Stuff”。Docker和Docker衍生的东西用到了很多很酷的技术，我会用几篇 文章来把这些技术给大家做个介绍，希望通过这些文章大家可以自己打造一个山寨版的docker。

当然，文章的风格一定会尊重时下的“流行”——我们再也没有整块整块的时间去看书去专研，而我们只有看微博微信那样的碎片时间（那怕我们有整块的时间，也被那些在手机上的APP碎片化了）。所以，这些文章的风格必然坚持“马桶风格”（希望简单到占用你拉一泡屎就时间，而且你还不用动脑子，并能学到些东西.废话少说，我们开始。先从Linux Namespace开始。

##1. 简介
Linux Namespace是Linux提供的一种内核级别环境隔离的方法。不知道你是否还记得很早以前的Unix有一个叫chroot的系统调用（通过修改根目录把用户jail到一个特定目录下），chroot提供了一种简单的隔离模式：chroot内部的文件系统无法访问外部的内容。Linux Namespace在此基础上，提供了对UTS、IPC、mount、PID、network、User等的隔离机制。

举个例子，我们都知道，Linux下的超级父亲进程的PID是1，所以，同chroot一样，如果我们可以把用户的进程空间jail到某个进程分支下，并像chroot那样让其下面的进程 看到的那个超级父进程的PID为1，于是就可以达到资源隔离的效果了（不同的PID namespace中的进程无法看到彼此）

Linux Namespace 有如下种类，官方文档在这里《[Namespace in Operation](http://lwn.net/Articles/531114/)》


|**分类**|**系统调用参数**|**隔离内容**|**相关内核版本**|
|:-----:|:------------:|:---------:|:------------:|
|Mount namespaces|CLONE_NEWNS|挂载点（文件系统）| Linux 2.4.19|
|UTS namespaces|CLONE_NEWUTS|主机名与域名|Linux 2.6.19|
|IPC namespaces|CLONE_NEWIPC|	信号量、消息队列和共享内存|Linux 2.6.19|
|PID namespaces|CLONE_NEWPID|进程编号|Linux 2.6.24|
|Network namespaces|CLONE_NEWNET|网络设备、网络栈、端口等等|Linux 2.6.29|
|User namespaces|CLONE_NEWUSER|用户和用户组|Linux 3.8|

Linux针对namespace提供了三个系统调用

* clone() – 实现线程的系统调用，用来创建一个新的进程，并可以通过设计上述参数达到隔离。
* unshare() – 使某进程脱离某个namespace
* setns() – 把某进程加入到某个namespace

大家可以man查看

##2. UTS Namespace
UTS namespace提供了主机名和域名的隔离，这样每个容器就可以拥有了独立的主机名和域名，在网络上可以被视作一个独立的节点而非宿主机上的一个进程。

```C
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
 
/* 定义一个给 clone 用的栈，栈大小1M */
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];
 
char* const container_args[] = {
    "/bin/bash",
    NULL
};

int container_main(void* arg)
{
    printf("Container - inside the container!\n");
    sethostname("container",10); /* 设置hostname */
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
 
int main()
{
    printf("Parent - start a container!\n");
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | SIGCHLD, NULL); /*启用CLONE_NEWUTS Namespace隔离 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

运行上面的程序你会发现（需要root权限），子进程的hostname变成了 container。

```bash
hchen@ubuntu:~$ sudo ./uts
Parent - start a container!
Container - inside the container!
root@container:~# hostname
container
root@container:~# uname -n
container
```
##3. IPC Namespace
IPC全称 Inter-Process Communication，是Unix/Linux下进程间通信的一种方式，IPC有共享内存、信号量、消息队列等方法。所以，为了隔离，我们也需要把IPC给隔离开来，这样，只有在同一个Namespace下的进程才能相互通信。如果你熟悉IPC的原理的话，你会知道，IPC需要有一个全局的ID，即然是全局的，那么就意味着我们的Namespace需要对这个ID隔离，不能让别的Namespace的进程看到。

要启动IPC隔离，我们只需要在调用clone时加上CLONE_NEWIPC参数就可以了。

```C
int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWIPC | SIGCHLD, NULL);
```

首先，我们先创建一个IPC的Queue（如下所示，全局的Queue ID是0）

```bash
hchen@ubuntu:~$ ipcmk -Q 
Message queue id: 0
 
hchen@ubuntu:~$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0xd0d56eb2 0          hchen      644        0            0
```

如果我们运行没有CLONE_NEWIPC的程序，我们会看到，在子进程中还是能看到这个全启的IPC Queue。

```bash
hchen@ubuntu:~$ sudo ./uts
Parent - start a container!
Container - inside the container!
 
root@container:~# ipcs -q
 
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0xd0d56eb2 0          hchen      644        0            0
```

但是，如果我们运行加上了CLONE_NEWIPC的程序，我们就会下面的结果：

```bash
root@ubuntu:~$ sudo./ipc
Parent - start a container!
Container - inside the container!
 
root@container:~/linux_namespace# ipcs -q
 
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```

我们可以看到IPC已经被隔离了。

##4. PID Namespace
PID namespace隔离非常实用，它对进程PID重新标号，即两个不同namespace下的进程可以有同一个PID。每个PID namespace都有自己的计数程序。内核为所有的PID namespace维护了一个树状结构，最顶层的是系统初始时创建的，我们称之为root namespace。他创建的新PID namespace就称之为child namespace（树的子节点），而原先的PID namespace就是新创建的PID namespace的parent namespace（树的父节点）。通过这种方式，不同的PID namespaces会形成一个等级体系。所属的父节点可以看到子节点中的进程，并可以通过信号等方式对子节点中的进程产生影响。反过来，子节点不能看到父节点PID namespace中的任何内容。由此产生如下结论

* 每个PID namespace中的第一个进程“PID 1“，都会像传统Linux中的init进程一样拥有特权，起特殊作用。
* 一个namespace中的进程，不可能通过kill或ptrace影响父节点或者兄弟节点中的进程，因为其他节点的PID在这个namespace中没有任何意义。
* 如果你在新的PID namespace中重新挂载/proc文件系统，会发现其下只显示同属一个PID namespace中的其他进程。
* 在root namespace中可以看到所有的进程，并且递归包含所有子节点中的进程。

我们继续修改上面的程序：

```bash
int container_main(void* arg)
{
    /* 查看子进程的PID，我们可以看到其输出子进程的 pid 为 1 */
    printf("Container [%5d] - inside the container!\n", getpid());
    sethostname("container",10);
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
 
int main()
{
    printf("Parent [%5d] - start a container!\n", getpid());
    /*启用PID namespace - CLONE_NEWPID*/
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWPID | SIGCHLD, NULL); 
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

运行结果如下（我们可以看到，子进程的pid是1了）：

```bash
hchen@ubuntu:~$ sudo ./pid
Parent [ 3474] - start a container!
Container [    1] - inside the container!
root@container:~# echo $$
1
```

子进程的进程号是1。可能有的读者在子进程的shell中执行了ps aux/top之类的命令，发现还是可以看到所有父进程的PID，那是因为我们还没有对文件系统进行隔离，ps/top之类的命令调用的是真实系统下的/proc文件内容，看到的自然是所有的进程。此外，与其他的namespace不同的是，为了实现一个稳定安全的容器，PID namespace还需要进行一些额外的工作才能确保其中的进程运行顺利。

###4.1 PID namespace中的init进程

当我们新建一个PID namespace时，默认启动的进程PID为1。我们知道，在传统的UNIX系统中，PID为1的进程是init，地位非常特殊。他作为所有进程的父进程，维护一张进程表，不断检查进程的状态，一旦有某个子进程因为程序错误成为了“孤儿”进程，init就会负责回收资源并结束这个子进程。所以在你要实现的容器中，启动的第一个进程也需要实现类似init的功能，维护所有后续启动进程的运行状态。

看到这里，可能读者已经明白了内核设计的良苦用心。PID namespace维护这样一个树状结构，非常有利于系统的资源监控与回收。Docker启动时，第一个进程也是这样，实现了进程监控和资源回收，它就是dockerinit。

###4.2 信号与init进程

PID namespace中的init进程如此特殊，自然内核也为他赋予了特权——信号屏蔽。如果init中没有写处理某个信号的代码逻辑，那么与init在同一个PID namespace下的进程（即使有超级权限）发送给它的该信号都会被屏蔽。这个功能的主要作用是防止init进程被误杀。

那么其父节点PID namespace中的进程发送同样的信号会被忽略吗？父节点中的进程发送的信号，如果不是SIGKILL（销毁进程）或SIGSTOP（暂停进程）也会被忽略。但如果发送SIGKILL或SIGSTOP，子节点的init会强制执行（无法通过代码捕捉进行特殊处理），也就是说父节点中的进程有权终止子节点中的进程。

一旦init进程被销毁，同一PID namespace中的其他进程也会随之接收到SIGKILL信号而被销毁。理论上，该PID namespace自然也就不复存在了。但是如果/proc/[pid]/ns/pid处于被挂载或者打开状态，namespace就会被保留下来。然而，保留下来的namespace无法通过setns()或者fork()创建进程，所以实际上并没有什么作用。

我们常说，Docker一旦启动就有进程在运行，不存在不包含任何进程的Docker，也就是这个道理。

###4.3 挂载proc文件系统

前文中已经提到，如果你在新的PID namespace中使用ps命令查看，看到的还是所有的进程，因为与PID直接相关的/proc文件系统（procfs）没有挂载到与原/proc不同的位置。所以如果你只想看到PID namespace本身应该看到的进程，需要重新挂载/proc，命令如下。

```bash
root@NewNamespace:~# mount -t proc proc /proc
root@NewNamespace:~# ps a
  PID TTY      STAT   TIME COMMAND
    1 pts/1    S      0:00 /bin/bash
   12 pts/1    R+     0:00 ps a
```
可以看到实际的PID namespace就只有两个进程在运行。

注意：因为此时我们没有进行mount namespace的隔离，所以这一步操作实际上已经影响了 root namespace的文件系统，当你退出新建的PID namespace以后再执行ps a就会发现出错，再次执行mount -t proc proc /proc可以修复错误。

###4.4 unshare()和setns()

在开篇我们就讲到了unshare()和setns()这两个API，而这两个API在PID namespace中使用时，也有一些特别之处需要注意。

unshare()允许用户在原有进程中建立namespace进行隔离。但是创建了PID namespace后，原先unshare()调用者进程并不进入新的PID namespace，接下来创建的子进程才会进入新的namespace，这个子进程也就随之成为新namespace中的init进程。

类似的，调用setns()创建新PID namespace时，调用者进程也不进入新的PID namespace，而是随后创建的子进程进入。

为什么创建其他namespace时unshare()和setns()会直接进入新的namespace而唯独PID namespace不是如此呢？因为调用getpid()函数得到的PID是根据调用者所在的PID namespace而决定返回哪个PID，进入新的PID namespace会导致PID产生变化。而对用户态的程序和库函数来说，他们都认为进程的PID是一个常量，PID的变化会引起这些进程奔溃。

换句话说，一旦程序进程创建以后，那么它的PID namespace的关系就确定下来了，进程不会变更他们对应的PID namespace。

##5. Mount namespaces
Mount namespace通过隔离文件系统挂载点对隔离文件系统提供支持，它是历史上第一个Linux namespace，所以它的标识位比较特殊，就是CLONE_NEWNS。隔离后，不同mount namespace中的文件结构发生变化也互不影响。你可以通过/proc/[pid]/mounts查看到所有挂载在当前namespace中的文件系统，还可以通过/proc/[pid]/mountstats看到mount namespace中文件设备的统计信息，包括挂载文件的名字、文件系统类型、挂载位置等等。

下面的例程中，我们在启用了mount namespace并在子进程中重新mount了/proc文件系统。

```C
int container_main(void* arg)
{
    printf("Container [%5d] - inside the container!\n", getpid());
    sethostname("container",10);
    /* 重新mount proc文件系统到 /proc下 */
    system("mount -t proc proc /proc");
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
 
int main()
{
    printf("Parent [%5d] - start a container!\n", getpid());
    /* 启用Mount Namespace - 增加CLONE_NEWNS参数 */
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS | SIGCHLD, NULL);
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

运行结果如下：

```bash
hchen@ubuntu:~$ sudo ./pid.mnt
Parent [ 3502] - start a container!
Container [    1] - inside the container!
root@container:~# ps -elf 
F S UID        PID  PPID  C PRI  NI ADDR SZ WCHAN  STIME TTY          TIME CMD
4 S root         1     0  0  80   0 -  6917 wait   19:55 pts/2    00:00:00 /bin/bash
0 R root        14     1  0  80   0 -  5671 -      19:56 pts/2    00:00:00 ps -elf
```

上面，我们可以看到只有两个进程 ，而且pid=1的进程是我们的/bin/bash。我们还可以看到/proc目录下也干净了很多：

```bash
root@container:~# ls /proc
1          dma          key-users   net            sysvipc
16         driver       kmsg        pagetypeinfo   timer_list
acpi       execdomains  kpagecount  partitions     timer_stats
asound     fb           kpageflags  sched_debug    tty
buddyinfo  filesystems  loadavg     schedstat      uptime
bus        fs           locks       scsi           version
cgroups    interrupts   mdstat      self           version_signature
cmdline    iomem        meminfo     slabinfo       vmallocinfo
consoles   ioports      misc        softirqs       vmstat
cpuinfo    irq          modules     stat           zoneinfo
crypto     kallsyms     mounts      swaps
devices    kcore        mpt         sys
diskstats  keys         mtrr        sysrq-trigger
```

这里，多说一下。在通过CLONE_NEWNS创建mount namespace后，父进程会把自己的文件结构复制给子进程中。而子进程中新的namespace中的所有mount操作都只影响自身的文件系统，而不对外界产生任何影响。这样可以做到比较严格地隔离。

你可能会问，我们是不是还有别的一些文件系统也需要这样mount? 是的。


##6. User Namespace
User namespace主要隔离了安全相关的标识符（identifiers）和属性（attributes），包括用户ID、用户组ID、root目录、key（指密钥）以及特殊权限。说得通俗一点，一个普通用户的进程通过clone()创建的新进程在新user namespace中可以拥有不同的用户和用户组。这意味着一个进程在容器外属于一个没有特权的普通用户，但是他创建的容器进程却属于拥有所有权限的超级用户，这个技术为容器提供了极大的自由。

User namespace是目前的六个namespace中最后一个支持的，并且直到Linux内核3.8版本的时候还未完全实现（还有部分文件系统不支持）。因为user namespace实际上并不算完全成熟，很多发行版担心安全问题，在编译内核的时候并未开启USER_NS。实际上目前Docker也还不支持user namespace，但是预留了相应接口，相信在不久后就会支持这一特性。所以在进行接下来的代码实验时，请确保你系统的Linux内核版本高于3.8并且内核编译时开启了USER_NS（如果你不会选择，可以使用Ubuntu14.04）。

User Namespace主要是用了CLONE_NEWUSER的参数。使用了这个参数后，内部看到的UID和GID已经与外部不同了，默认显示为65534。那是因为容器找不到其真正的UID所以，设置上了最大的UID（其设置定义在/proc/sys/kernel/overflowuid）。

要把容器中的uid和真实系统的uid给映射在一起，需要修改 **/proc/<pid>/uid_map** 和 **/proc/<pid>/gid_map** 这两个文件。这两个文件的格式为：

```bash
ID-inside-ns ID-outside-ns length
```
其中：

* 第一个字段ID-inside-ns表示在容器显示的UID或GID，
* 第二个字段ID-outside-ns表示容器外映射的真实的UID或GID。
* 第三个字段表示映射的范围，一般填1，表示一一对应。

比如，把真实的uid=1000映射成容器内的uid=0

```bash
$ cat /proc/2465/uid_map
         0       1000          1
```

另外，需要注意的是：

* 写这两个文件的进程需要这个namespace中的CAP_SETUID (CAP_SETGID)权限（可参看[Capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html)）
* 写入的进程必须是此user namespace的父或子的user namespace进程。
* 另外需要满如下条件之一：1）父进程将effective uid/gid映射到子进程的user namespace中，2）父进程如果有CAP_SETUID/CAP_SETGID权限，那么它将可以映射到父进程中的任一uid/gid。


这些规则看着都烦，我们来看程序吧（下面的程序有点长，但是非常简单，如果你读过《Unix网络编程》上卷，你应该可以看懂）：

```bash
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/mount.h>
#include <sys/capability.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
 
#define STACK_SIZE (1024 * 1024)
 
static char container_stack[STACK_SIZE];
char* const container_args[] = {
    "/bin/bash",
    NULL
};
 
int pipefd[2];
 
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
 
    /* 等待父进程通知后再往下执行（进程间的同步） */
    char ch;
    close(pipefd[1]);
    read(pipefd[0], &ch, 1);
 
    printf("Container [%5d] - setup hostname!\n", getpid());
    //set hostname
    sethostname("container",10);
 
    //remount "/proc" to make sure the "top" and "ps" show container's information
    mount("proc", "/proc", "proc", 0, NULL);
 
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}
 
int main()
{
    const int gid=getgid(), uid=getuid();
 
    printf("Parent: eUID = %ld;  eGID = %ld, UID=%ld, GID=%ld\n",
            (long) geteuid(), (long) getegid(), (long) getuid(), (long) getgid());
 
    pipe(pipefd);
  
    printf("Parent [%5d] - start a container!\n", getpid());
 
    int container_pid = clone(container_main, container_stack+STACK_SIZE, 
            CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWUSER | SIGCHLD, NULL);
 
     
    printf("Parent [%5d] - Container [%5d]!\n", getpid(), container_pid);
 
    //To map the uid/gid, 
    //   we need edit the /proc/PID/uid_map (or /proc/PID/gid_map) in parent
    //The file format is
    //   ID-inside-ns   ID-outside-ns   length
    //if no mapping, 
    //   the uid will be taken from /proc/sys/kernel/overflowuid
    //   the gid will be taken from /proc/sys/kernel/overflowgid
    set_uid_map(container_pid, 0, uid, 1);
    set_gid_map(container_pid, 0, gid, 1);
 
    printf("Parent [%5d] - user/group mapping done!\n", getpid());
 
    /* 通知子进程 */
    close(pipefd[1]);
 
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```

上面的程序，我们用了一个pipe来对父子进程进行同步，为什么要这样做？因为子进程中有一个execv的系统调用，这个系统调用会把当前子进程的进程空间给全部覆盖掉，我们希望在execv之前就做好user namespace的uid/gid的映射，这样，execv运行的/bin/bash就会因为我们设置了uid为0的inside-uid而变成#号的提示符。

整个程序的运行效果如下：

```bash
hchen@ubuntu:~$ id
uid=1000(hchen) gid=1000(hchen) groups=1000(hchen)
 
hchen@ubuntu:~$ ./user #<--以hchen用户运行
Parent: eUID = 1000;  eGID = 1000, UID=1000, GID=1000 
Parent [ 3262] - start a container!
Parent [ 3262] - Container [ 3263]!
Parent [ 3262] - user/group mapping done!
Container [    1] - inside the container!
Container: eUID = 0;  eGID = 0, UID=0, GID=0 #<---Container里的UID/GID都为0了
Container [    1] - setup hostname!
 
root@container:~# id #<----我们可以看到容器里的用户和命令行提示符是root用户了
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
```

虽然容器里是root，但其实这个容器的/bin/bash进程是以一个普通用户hchen来运行的。这样一来，我们容器的安全性会得到提高。

我们注意到，User Namespace是以普通用户运行，但是别的Namespace需要root权限，那么，如果我要同时使用多个Namespace，该怎么办呢？一般来说，我们先用一般用户创建User Namespace，然后把这个一般用户映射成root，在容器内用root来创建其它的Namesapce。

##7. Network Namespace
Network的Namespace比较啰嗦。在Linux下，我们一般用ip命令创建Network Namespace（Docker的源码中，它没有用ip命令，而是自己实现了ip命令内的一些功能——是用了Raw Socket发些“奇怪”的数据，呵呵）。这里，我还是用ip命令讲解一下。

首先，我们先看个图，下面这个图基本上就是Docker在宿主机上的网络示意图（其中的物理网卡并不准确，因为docker可能会运行在一个VM中，所以，这里所谓的“物理网卡”其实也就是一个有可以路由的IP的网卡）
![docker0](./docker0.jpg)

上图中，Docker使用了一个私有网段，172.40.1.0，docker还可能会使用10.0.0.0和192.168.0.0这两个私有网段，关键看你的路由表中是否配置了，如果没有配置，就会使用，如果你的路由表配置了所有私有网段，那么docker启动时就会出错了。

当你启动一个Docker容器后，你可以使用ip link show或ip addr show来查看当前宿主机的网络情况（我们可以看到有一个docker0，还有一个veth22a38e6的虚拟网卡——给容器用的）：

```bash
hchen@ubuntu:~$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state ... 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc ...
    link/ether 00:0c:29:b7:67:7d brd ff:ff:ff:ff:ff:ff
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    link/ether 56:84:7a:fe:97:99 brd ff:ff:ff:ff:ff:ff
5: veth22a38e6: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc ...
    link/ether 8e:30:2a:ac:8c:d1 brd ff:ff:ff:ff:ff:ff
```

那么，要做成这个样子应该怎么办呢？我们来看一组命令：

```bash
## 首先，我们先增加一个网桥lxcbr0，模仿docker0
brctl addbr lxcbr0
brctl stp lxcbr0 off
ifconfig lxcbr0 192.168.10.1/24 up #为网桥设置IP地址
 
## 接下来，我们要创建一个network namespace - ns1
 
# 增加一个namesapce 命令为 ns1 （使用ip netns add命令）
ip netns add ns1 
 
# 激活namespace中的loopback，即127.0.0.1（使用ip netns exec ns1来操作ns1中的命令）
ip netns exec ns1   ip link set dev lo up 
 
## 然后，我们需要增加一对虚拟网卡
 
# 增加一个pair虚拟网卡，注意其中的veth类型，其中一个网卡要按进容器中
ip link add veth-ns1 type veth peer name lxcbr0.1
 
# 把 veth-ns1 按到namespace ns1中，这样容器中就会有一个新的网卡了
ip link set veth-ns1 netns ns1
 
# 把容器里的 veth-ns1改名为 eth0 （容器外会冲突，容器内就不会了）
ip netns exec ns1  ip link set dev veth-ns1 name eth0 
 
# 为容器中的网卡分配一个IP地址，并激活它
ip netns exec ns1 ifconfig eth0 192.168.10.11/24 up
 
 
# 上面我们把veth-ns1这个网卡按到了容器中，然后我们要把lxcbr0.1添加上网桥上
brctl addif lxcbr0 lxcbr0.1
 
# 为容器增加一个路由规则，让容器可以访问外面的网络
ip netns exec ns1     ip route add default via 192.168.10.1
 
# 在/etc/netns下创建network namespce名称为ns1的目录，
# 然后为这个namespace设置resolv.conf，这样，容器内就可以访问域名了
mkdir -p /etc/netns/ns1
echo "nameserver 8.8.8.8" > /etc/netns/ns1/resolv.conf
```

上面基本上就是docker网络的原理了，只不过，

* Docker的resolv.conf没有用这样的方式，而是用了前面Mount Namesapce的那种方式
* 另外，docker是用进程的PID来做Network Namespace的名称的。

了解了这些后，你甚至可以为正在运行的docker容器增加一个新的网卡：

```bash
ip link add peerA type veth peer name peerB 
brctl addif docker0 peerA 
ip link set peerA up 
ip link set peerB netns ${container-pid} 
ip netns exec ${container-pid} ip link set dev peerB name eth1 
ip netns exec ${container-pid} ip link set eth1 up ; 
ip netns exec ${container-pid} ip addr add ${ROUTEABLE_IP} dev eth1 ;
```

上面的示例是我们为正在运行的docker容器，增加一个eth1的网卡，并给了一个静态的可被外部访问到的IP地址。

这个需要把外部的“物理网卡”配置成混杂模式，这样这个eth1网卡就会向外通过ARP协议发送自己的Mac地址，然后外部的交换机就会把到这个IP地址的包转到“物理网卡”上，因为是混杂模式，所以eth1就能收到相关的数据，一看，是自己的，那么就收到。这样，Docker容器的网络就和外部通了。

当然，无论是Docker的NAT方式，还是混杂模式都会有性能上的问题，NAT不用说了，存在一个转发的开销，混杂模式呢，网卡上收到的负载都会完全交给所有的虚拟网卡上，于是就算一个网卡上没有数据，但也会被其它网卡上的数据所影响。

这两种方式都不够完美，我们知道，真正解决这种网络问题需要使用VLAN技术，于是Google的同学们为Linux内核实现了一个IPVLAN的驱动，这基本上就是为Docker量身定制的。