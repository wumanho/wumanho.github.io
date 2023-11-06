---
weight: 2
title: "命令行的本质（下）- 命令行的关键四要素"
summary: "介绍命令行的关键四要素，以及四要素如何影响一条命令的执行"
date: 2020-12-17T15:25:07+08:00
draft: false
categories: ["技术","推荐阅读"]
tags: ["bash","环境变量"]
---

在[上一篇博客](https://wumanho.cn/posts/cmd/)中探讨了命令行的本质和 Shell 的查找机制等基础问题之后，接下来需要重点关注的内容是关于命令行的`四要素`。

&nbsp;

## 命令的四要素

它们分别是：

* 环境变量
* 可执行程序
* 参数
* 工作路径

&nbsp;

### 环境变量

环境变量最广泛的用途便是将配置从应用程序中解耦。

首先，任何一个现代的操作系统都支持环境变量（在 Windows 上是不区分大小写的），通常是由一组键值对组成，可以为每个键设置对应的值，例如[上一篇博客](https://wumanho.cn/posts/cmd/)里，在探讨 Shell 的查找机制过程中，我们使用了 Linux 著名的环境变量 `$PATH`。

```Code
[root@vm10-0-0-132 ~]# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```

其中 `$`符表示这是一个环境变量，`PATH`便是环境变量的键名，打印出来的结果是环境变量对应的值。

其次，所有编程语言都原生支持环境变量，也就是说，如果你需要在两个进程之间实现交互，你是可以**完全放心大胆地使用环境变量而不必担心不被支持**。

为了证明这一点，我们可以观察一下以下的例子：

```
[root@vm10-0-0-132 ~]# export AAA=123
[root@vm10-0-0-132 ~]# node
> console.log(process.env['AAA'])
123
>
```

先声明了一个环境变量 AAA（环境变量建议大写），并赋值123，然后在 nodejs 里面获取到这个值。

同理，我们用 golang 试一试，发现同样可以获取到相同的值：

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	fmt.Println(os.Getenv("AAA"))
}
```

```code
[root@vm10-0-0-132 ~]# go run demo.go
123
```

需要强调的是，在上一篇博客中我们提到过，「在大多数情况下，执行一个命令，就等于 fork 一个新的进程」。

也就是说在上面的例子中，`node`和`go`这两个命令，其实是分别 fork 了一个新的进程去运行的，这其实可以说明一个重点：**被当前进程所派生的所有子进程，都会继承父进程中所有环境变量**。

不过，子进程中创建的环境变量，却无法反向传递到父进程中：

```
[root@vm10-0-0-132 ~]# node
> process.env.TEST = "Test"
'test'
> console.log(process.env['TEST'])
test
> .exit
[root@vm10-0-0-132 ~]# echo $TEST

[root@vm10-0-0-132 ~]#
```

环境变量除了可以用于解耦配置之外，还经常被用于初始化程序、保存重要信息等，需要注意的是，通过 `export`声明的环境变量是临时的，如果需要声明永久的环境变量，则需要将 `export` 语句写在当前用户的启动配置文件`~/.bash_profile`中（不同的 shell 配置文件也不同）。

简单介绍几个经典的环境变量和他们的作用：

`$PATH`：PATH 环境变量用于记录可执行程序的位置，绝大多数命令都需要从 PATH 环境变量定义的值中寻找执行路径，具体可以参考上一篇博客[Shell 的查找机制](https://wumanho.cn/posts/cmd/#shell-%E7%9A%84%E6%9F%A5%E6%89%BE%E6%9C%BA%E5%88%B6-)这一节；

`PS1`：定义命令提示符格式；

`CLASSPATH`：用于指示 JVM 如何搜索 class

`GOPATH`：指定 golang 工作目录

&nbsp;

### 可执行程序

可执行程序就是你每次输入命令的第一个单词，例如：`cd`、`docker`、`node`、`mvn`等等。

在 Windows 中，可执行程序就是常见的 `.exe`、`.bat`文件。

在 Linux 中，如果一个文件具有 「x」权限的话，那么它就是一个可执行程序，「x」是Linux 权限系统的其中一个标记，可以通过`chmod +x`命令让一个文件成为可执行程序，Linux 权限系统就不在这边赘述。

当用户输入一个可执行程序并按下回车，Shell 会扫描 `$PATH`环境变量，找到用户需要的可执行程序，如果没有会报「找不到命令」错误，这个时候用户其实还可以通过指定文件的**相对路径**或者**绝对路径**去执行这个可执行程序：

```code
[root@vm10-0-0-132 ~]# vim demo.sh
[root@vm10-0-0-132 ~]# cat demo.sh
echo "Hello"
[root@vm10-0-0-132 ~]# chmod +x demo.sh
[root@vm10-0-0-132 ~]# demo.sh
-bash: demo.sh: command not found
[root@vm10-0-0-132 ~]# ./demo.sh
Hello
```

一个可执行文件可以被执行是因为它拥有「x」权限，那这个可执行文件本身又是什么呢，可以通过`file`命令查看：

```code
[root@vm10-0-0-132 ~]# file /usr/local/go/bin/go
/usr/local/go/bin/go: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), not stripped
```

根据结果可以看到 go 是一个 ELF 格式 64 位的二进制可执行程序，它可直接被机器执行而不需要解释器，与之有别的是被我们称为「脚本」的可执行文件，它需要我们我们指定解释器。

写一个最简单的脚本：

```code
[root@vm10-0-0-132 ~]# vim script.sh
[root@vm10-0-0-132 ~]# chmod +x script.sh
[root@vm10-0-0-132 ~]# cat script.sh
echo "Hello"
```

让它打印一个 “Hello”，直接执行看看：

```
[root@vm10-0-0-132 ~]# ./script.sh
Hello
```

这里我们没有执行解释器，却可以执行脚本打印结果，是因为这个脚本被 Shell 直接解析执行了，再试一下：

```code
[root@vm10-0-0-132 ~]# vim script1.sh
[root@vm10-0-0-132 ~]# cat script1.sh
console.log("123")
[root@vm10-0-0-132 ~]# ./script1.sh
./script1.sh: line 1: syntax error near unexpected token `"123"'
./script1.sh: line 1: `console.log("123")'
[root@vm10-0-0-132 ~]# 
```

`console.log()`是 JS 语法，Shell 不认识这个语法，所以会提示你语法错误，如果要让脚本顺利执行，则需要指定执行它的解释器：

```code
[root@vm10-0-0-132 ~]# vim script1.sh
[root@vm10-0-0-132 ~]# cat script1.sh
#! /usr/bin/node
console.log("123")

[root@vm10-0-0-132 ~]# ./script1.sh
123
```

执行正常，`#!`这个特殊语法必须声明在脚本文件的第一行，为该脚本指定一个解释器，也可以理解为 Shell 帮你将该脚本传递到你指定的解释器。

&nbsp;

### 参数

当用户在输入一个命令的时候，例如：

```code
[root@vm10-0-0-132 ~]# docker run -d --name redis-01 redis
```

这个命令里面，`docker`是可执行程序，而后面的所有都是「参数」，Shell 所做的事情就是将这些参数全部丢给可执行程序。

Shell 不在乎可执行程序是否可以接受这些参数，因为解释这些参数是这个可执行程序的事。

所以这样听起来，当你的命令运行执行失败的时候，似乎就跟 Shell 无关了？其实不一定，因为 Shell 有可能会在参数中偷偷做手脚，而这些动作可能对用户产生一些误导，甚至制造麻烦。

看一个例子，准备一个 go 程序 script.go，打印出我们传给它的所有参数：

```go
package main

import (
    "fmt"
    "os"
)

func main()  {
    for k,v:= range os.Args{
        fmt.Printf("args[%v]=[%v]\n",k,v)
    }
}
```

```code
[root@vm10-0-0-132 ~]# ls
demo.go  go.mod  script  script.go  test1.go  test2.go

[root@vm10-0-0-132 ~]# ./script *.go
args[0]=[./script]
args[1]=[demo.go]
args[2]=[script.go]
args[3]=[test1.go]
args[4]=[test2.go]
[root@vm10-0-0-132 ~]#
```

可以发现，由于 Shell 帮我们将通配符做了展开，所以这就造成了一个现象，就是用户**传给程序的参数跟程序接收到的参数不一致**。

这在有些时候会误导用户，因为通配符展开是由 Shell 来替我们完成的，所以如果用户在代码中派生一个子进程，然后传递带 * 通配符的参数，希望程序可以将这个通配符展开，实际上是做不到的，同理，`?`通配符也一样。

对于参数还有一个需要注意的坑是「变量展开」，在 Linux 系统中，`$`符号默认被用作变量声明，如果文件名包含`$`号，同样会被展开：

```code
[root@vm10-0-0-132 ~]# export TEST=123
[root@vm10-0-0-132 ~]# touch 1$TEST.go
[root@vm10-0-0-132 ~]# ls
1123.go
```

展开问题可以通过为参数加单引号` ' '`来声明参数是一个整体：

```code
[root@vm10-0-0-132 ~]# touch '1$TEST.go'
[root@vm10-0-0-132 ~]# ls
1123.go  1$TEST.go
```

上述两个是 Shell 参数比较常见的坑，日常开发时需要多注意，少走弯路。

&nbsp;

### 工作路径

工作路径可以使用`pwd`命令查看，Linux 用户登录到系统之后，每时每刻都处于某一个目录当中，这个时候用户所处的目录就称为工作目录。

我们知道**绝对路径**是从`/`开始到指定文件的具体路径，而**相对路径**所相对的就是用户的工作目录。

如果用户在工作目录下执行一个可执行程序，那么对于这个进程来说，这个目录的路径就是它的工作路径。

我们引用上面的 script.go 代码来演示，这个 go 代码打印出当前目录下所有以  .go 节为的文件名：

```code
[root@vm10-0-0-132 ~]# ls
1123.go  1$TEST.go  go.mod  script
[root@vm10-0-0-132 ~]# ./script *.go
args[0]=[./script]
args[1]=[1123.go]
args[2]=[1$TEST.go]
```

改变一下 script 程序的目录，将它移动到 `/root` 下面

```code
[root@vm10-0-0-132 ~]# mv ./script /root
[root@vm10-0-0-132 ~]# cd
[root@vm10-0-0-132 ~]# ls
script a.txt t.txt test.go
[root@vm10-0-0-132 ~]# ./script *.go
args[0]=[./script]
args[1]=[test.go]
```

script 被移动后，我们在 `/root` 目录直接下执行 script，由于工作路径改变，所以打印的内容也会有所改变。

&nbsp;

## 总结

四要素之所以重要，是因为它们是命令运行机制的关键所在，理解它们可以帮助你解决关于命令行的一切问题。

如果命令的四要素完全相同的话，那么命令执行的结果一定可以重现。

如果相同的命令执行结果跟你预想的不一样，那么一定是四要素中一个或多个发生了改变，所以当碰到奇怪的问题时，例如相同的代码在一台机器上运行正常，在另一台机器上执行失败，可以先问一下自己，当前执行的这个命令的四要素分别是什么。

