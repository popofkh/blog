---
layout: article
title: Linux Namespace详解(一)：综述
date: 2021-01-08 00:20:16
tags: Linux
---



>参考来源：
>
>https://cizixs.com/2017/08/29/linux-namespace/
>
>https://coolshell.cn/articles/17010.html
>
>https://mp.weixin.qq.com/s/10HgkUE14wVI_RNmFdqkzA

# 简介

Linux Namespace是Linux提供的一种内核级别的环境隔离方法。在Unix时代，有一个叫做chroot的系统调用，它提供了一种简单的隔离模式：chroot内部的文件系统无法访问外部的内容。目前Linux 内核提供了以下几种Namespace：

Namespace在此基础上，提供了对UTS、IPC、mount、PID、network、User等的隔离机制。

| 名称   | 宏定义          | 隔离资源                                                     | 内核版本 |
| ------ | --------------- | ------------------------------------------------------------ | -------- |
| mnt    | CLONE_NEWNS     | Mount point                                                  | 2.4.19   |
| ipc    | CLONE_NEWIPC    | System V IPC, POSIX message queue                            | 2.6.19   |
| net    | CLONE_NEWNET    | network device interface, IPv4 and IPv6 protocol stack, IP routing table, firewall rule, the /proc/net and /sys/class/net directory tree, socket, etc | 2.6.24   |
| pid    | CLONE_NEWPID    | Process ID                                                   | 2.6.24   |
| user   | CLONE_NEWUSER   | User and group ID                                            | 3.8      |
| UTS    | CLONE_NEWUTS    | Hostname and NIS domain name                                 | 2.6.19   |
| cgroup | CLONE_NEWCGROUP | Control group root directory                                 | 4.6      |

这些namespace基本上覆盖了一个程序运行所需的环境，包括主机名、用户权限、文件系统、网络、进程号、进程间通信。所有的进程都会有namespace，可以理解为namespace是进程的一个属性。

每个进程都对应一个`/proc/[pid]/ns`目录，里面保存了该进程所在namespace的链接文件：

```shell
[root@control-plane ~]# ps
   PID TTY          TIME CMD
  2718 pts/0    00:00:00 bash
 18121 pts/0    00:00:00 ps
[root@control-plane ~]# ll /proc/2718/ns/
total 0
lrwxrwxrwx. 1 root root 0 Jan  7 11:43 ipc -> ipc:[4026531839]
lrwxrwxrwx. 1 root root 0 Jan  7 11:43 mnt -> mnt:[4026531840]
lrwxrwxrwx. 1 root root 0 Jan  7 11:43 net -> net:[4026531956]
lrwxrwxrwx. 1 root root 0 Jan  7 11:43 pid -> pid:[4026531836]
lrwxrwxrwx. 1 root root 0 Jan  7 11:43 user -> user:[4026531837]
lrwxrwxrwx. 1 root root 0 Jan  7 11:43 uts -> uts:[4026531838]
```

上述每个文件都是对应namespace的文件描述符，方括号中的值是文件的inode，如果两个进程所在的namespace是相同的，那么inode值也是相同的。如果某个namespace中没有进程了，它会被自动删除，但当某个应用程序打开了namespace文件时它不会被删除，通过这个特性，完美可以持有一个空的namspace，以便向其中添加进程。

# 三个系统调用

Linux内核提供的功能都会提供系统调用接口供应用程序使用，namespace自然也不例外。和namespace相关的系统调用主要有三个：`clone`、`setns`、`unshare`

## clone：创建新进程并设置它的namespace

clone类似于fork系统调用，可以创建一个新进程，不同的是我们可以指定紫禁城要执行的函数，以及通过参数控制子进程的运行环境。下面是clone的定义：

```c
# include <sched.h>
int clone(int (*fn) (void *), void *child_stack, int flags, void *arg, ...);
```

它有四个重要的参数：

* `fn`参数是一个函数指针，子进程启动的时候会调用这个函数来执行
* `child_stack`参数指定了子进程stack开始的内存地址，因为stack都会从高位到地委增长，所以这个指针需要指向分配stack的最高位地址
* `flags`用来控制子进程的特性，比如新创建的进程是否与父进程共享虚拟内存等。比如可以传入`CLONE_NEWNS`标志使得新创建的新城拥有独立的`Mount Namespace`，也可以传入多个flags如`CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC`
* `arg`作为`fn`函数的参数

## setns：让进程加入已经存在的namespace

`setns`能够把某个进程加入到给定的namespace，它的定义是这样的：

```c
int setns(int fd, int nstype);
```

* `fd`参数是一个文件描述符，指向`/proc/[pid]/ns/`目录下的某个namespace，调用这个函数的进程就会被加入到`fd`指向文件所代表的namespace，`fd`可以通过打开namespace对应的文件获取
* `nstype`限定进程可以加入的namespace，可能的取值有：（如果不知道 `fd` 指向的 namespace 类型，比如`fd`是其他进程打开的，然后在应用中希望明确指定特种类型的 `namespace`，`nstype` 就非常有用）
  * 0：可以加入任意的namespace
  * `CLONE_NEWIPC`：fd 必须指向`IPC namespace`
  * `CLONE_NEWNET`：fd 必须指向`network namespace`
  * `CLONE_NEWNS`：fd 必须指向`mount namespace`
  * `CLONE_NEWPID`：fd 必须指向`PID namespace`
  * `CLONE_NEWUSER`： fd 必须指向`user namespace`
  * `CLONE_NEWUTS`： fd 必须指向`UTS namespace`

## unshare：让进程脱离到新的namespace

`unshare()`系统调用用于将当前进程和所在的namespace分离，并加入到一个新的namespace中，只需要指定想要分离的namespace即可，它的定义如下：

```c
int unshare(int flags);
```

*  `flags`，它的含义和 `clone` 的 `flags` 相同。
  * `CLONE_FILES`: 子进程一般会共享父进程的文件描述符，如果子进程不想共享父进程的文件描述符了，可以通过这个flag来取消共享。
  * `CLONE_FS`: 使当前进程不再与其他进程共享文件系统信息。
  * `CLONE_SYSVSEM`: 取消与其他进程共享SYS V信号量。
  * `CLONE_NEWIPC`: 创建新的`ipc namespace`，并将该进程加入进来。

> **注意**：`unshare()`和`setns()`系统调用对`PID Namespace`的处理不太相同。
>
> 发生系统调用时，调用进程会为它的子进程分配一个新的`pid namespace`，且调用进程第一个创建的子进程会成为新namespace中的pid=1的进程，但是调用进程本身不会被移到新的`pid namespace`中。
>
> 为什么创建其他的namespace时`unshare()`和`setns()`会直接进入新的namespace，而唯独PID Namespace不是如此呢？
>
> 因为调用`getpid()`函数得到的pid是根据调用者所在的PID Namespace而决定返回哪个pid，进入新的`pid namespace`会导致pid变化。而对用户态的程序和库函数来说，他们都认为进程的pid是一个常量，pid的变化会引起这些进程奔溃。
>
> 换句话说，一旦程序进程创建以后，那么它的`pid namespace`的关系就确定下来了。