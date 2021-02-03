---
weight: 2
title: "命令行的本质和四要素（一）"
date: 2020-11-27T14:10:08+08:00
draft: false
categories: ["技术"]
tags: ["bash","ssh"]
---

## 内核与壳 :turtle:

内核（Kernel）是计算机操作系统用于与硬件打交道、进程调度、访问文件系统的一小段短小精悍的代码，他为我们屏蔽了硬件的差异，并且工作在一个非常特殊的状态，叫「内核态」。

壳（Shell）就是我们用于跟内核打交道的方式，Shell 是**一类程序的统称**，并不单单指某一个特定程序。

我们耳熟能详的 Shell 包括 bash、zsh、csh 等等，其实 Windows 提供的图形界面也是一个 Shell。

![shell示意图](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cmd/shell.png)

&nbsp;

## 命令行的本质 :thinking:

在 Linux 系统启动后，内核会自行启动一个 init 进程，所谓的 init 进程是一个由内核启动的用户级进程。  

init 进程始终会是 Linux 系统的第一个进程，其余必要的进程会由 init 进程 fork 出来，注意这个时候是没有 SHELL 的，只有一堆后台进程在运行，其中还包括一个叫做 「SSHD」的进程。

![init示意图](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cmd/proc.png)

SSHD 的 「D」是 Daemon 的意思，在计算机领域常用于表示一个**后台进程**，类似的还有「firewalld」等。  

SSHD 进程默认监听 22 端口，监听端口可以通过修改 /etc/ssh/sshd_config 配置文件指定，一旦有远程设备发起对 22 端口的请求，验证通过之后，SSHD 就会 fork 出来一个新的进程，该新进程名字就叫做 bash（或者其他 Shell）。

Shell 进程为用户提供一个可交互的窗口，它的工作就是读取用户输入的命令并执行，他的执行结果可能是 fork 一个新的进程，或者输出一些内容，取决于用户输入的具体命令，这其实就是命令行的本质。

以上的过程可以通过 `pstree` 命令验证：

![pstree命令](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cmd/pstree.png)

图中可以看到，bash 是 SSHD fork 出来的子进程，而当我执行 pstree 命令之后，bash 又 fork 出来一个 pstree 进程。

&nbsp;

### 如何指定 Shell :golfing_man:

Linux 有一个著名的文件`/etc/passwd`，该文件中保存了系统中所有用户的信息，每一行就是一个用户。

```bash
# cat /etc/passwd
root: x:0:0:root:/root:/bin/bash
bin: x:1:1:bin:/bin:/sbin/nologin
daemon: x:2:2:daemon:/sbin:/sbin/nologin
adm: x:3:4:adm:/var/adm:/sbin/nologin
lp: x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync: x:5:0:sync:/sbin:/bin/sync
shutdown: x:6:0:shutdown:/sbin:/sbin/shutdown
halt: x:7:0:halt:/sbin:/sbin/halt
mail: x:8:12:mail:/var/spool/mail:/sbin/nologin
operator: x:11:0:operator:/root:/sbin/nologin
games: x:12:100:games:/usr/games:/sbin/nologin
ftp: x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody: x:99:99:Nobody:/:/sbin/nologin
systemd-bus-proxy: x:999:998:systemd Bus Proxy:/:/sbin/nologin
systemd-network: x:192:192:systemd Network Management:/:/sbin/nologin
dbus: x:81:81:System message bus:/:/sbin/nologin
polkitd: x:998:997:User for polkitd:/:/sbin/nologin
tss: x:59:59:Account used by the trousers package to sandbox the tcsd daemon:/dev/null:/sbin/nologin
sshd: x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
postfix: x:89:89::/var/spool/postfix:/sbin/nologin
chrony: x:997:995::/var/lib/chrony:/sbin/nologin
```

每一行的最后一个`:`后面的内容，就是对应该用户登录之后，默认打开的 Shell 进程是哪个一个，如果是`nologin`的话，则以为着该用户无法登录。

只要修改这个配置文件，就可以为每个用户指定默认的 Shell 进程。

&nbsp;

### Shell 的查找机制 :sassy_woman:

当用户输入一个「内置」的命令，例如`cd`，那么 Shell 就会立马作出反应，执行命令，切换至目标目录。

但如果用户输入的命令不是一个内置命令，那么当前的 Shell 就会根据查找机制去找到命令所在的路径再去执行，如果找不到就报错。

![](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/cmd/path.png)

如上图所示，当我执行`mvn`命令时，系统提示「找不到命令」错误，但我输入`docker`的时候，就可以正确执行，这其实就是 Shell 根据查找机制没有找到 mvn 命令导致的。

当用户输入一个非内置命令时，当前的 Shell 就会开始尝试找到这个命令的位置，它会去哪里找呢？

默认情况下，它会去 `$PATH`环境变量定义的路径中找：

```
# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```

当 Shell 没有在 `$PATH`环境变量定义的路径中找到用户输入的命令对应的可执行程序时，就会报错，通常是非常熟悉的`command not found`错误。

而一旦找到了，那么 Shell 就会立马去执行，而并不关心这到底是什么。

&nbsp;

### 命令的执行 :dart:

在大多数情况下，执行一个命令，就等于 fork 一个新的进程。

例如当用户在命令行窗口输入`node`命令并敲下回车，bash 就会 fork 一个子进程 node，之后将所有参数以及控制权交给这个进程，这之后发生的事情就是程序自己的事情了，bash 并不会管，直到程序执行完成或者用户退出。

```
# node
> console.log("Hello")
Hello
undefined
> .exit
# 
```

同理，也有另外一种常见的情况，如果用户创建一个脚本 script.sh，内容是 export 一个 AAA 环境变量，值为 123：

```
# cat script.sh 
export AAA=123
```

执行一下，却发现什么也没有发生：

```
# sh script.sh 
# echo $AAA

#
```

原因是用户是用`sh`命令，执行 script.sh 的话，script.sh 其实就相当于是 `sh` 命令的参数，当前的 Shell 会为 `sh` 命令 fork 一个新的子进程，然后读取参数 script.sh 并且在**新的子进程中**执行，执行完毕后自动退出，而 export 只会新的子进程环境中生效，所以当前进程中，是看不到 AAA 变量的。

如何解决呢？可以利用`source`命令。

`source`命令可以使指定的 Shell 程序文件在当前环境中执行并生效：

```
# source script.sh 
# echo $AAA
123
# 
```

&nbsp;

（下一篇更新会介绍命令行的关键四要素）

&nbsp;