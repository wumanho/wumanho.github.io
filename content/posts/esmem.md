---
title: "ElasticSearch 内存占用百分比问题"
date: 2020-12-20T18:03:52+08:00
draft: false
categories: ["技术"]
tags: ["ElasticSearch"]
---

由于对 ES 查看集群健康状态 API `get _cat/nodes?v` 所展示的内存占用百分比有疑问，网上相关资料又比较少，所以简单整理说明一下，便于日后翻查。

公司使用的 ES 集群版本号为`5.5.3`，最近一次故障排查的时候查到 ES 这边，发现一个比较奇怪的现象，当时还对我们造成了一些误导，为我们排错制造了一些弯路。

在 Kibana 使用`get _cat/nodes?v` 查看集群所有节点状态的时候，看到所有节点的内存占用百分比都非常高：

![kibana结果](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/ES/rampercent.png)

有两个节点甚至到了 100%，而根据官方文档显示，该列的值所表示的意思是**该ES实例所在的物理主机的内存使用率**。

但是通过 zabbix 查看服务器状态却截然不同，实际上物理主机的内存可用百分比还有很多。

![zabbix结果](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/ES/zabbix.png)

这就跟官方文档的说法造成了一些冲突，到底谁才是正确的呢？

&nbsp;

## 排错过程

由于这个问题搜出来的结果不多，包括在 ES 社区也没有针对这个值的详细说明，在 [Stackoverflow](https://stackoverflow.com/questions/37350418/confused-about-elasticsearch-memory-consumption) 上面有一个提问比较接近，但是并不完全匹配

![stackoverflow提问](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/ES/stackoverflow.png)

他的问题是：关于 ES 内存占用的疑惑。他说明明只分配了 30G 给 ES 实例，为什么会显示我使用了 200G 的内存？

显然提问者不知道到他指定的`-Xms`和`-Xmx`是为 JVM 指定堆内存的参数，并不是指所有内存，下面的回答也指出了，而且还补充了一点，就是关于 Lucene（ES 底层搜索库）会通过占用操作系统的内存来为 Lucene 的索引做缓存加速，[官方文档](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/heap-sizing.html#_give_less_than_half_your_memory_to_lucene)对此也有详细说明。

![官网截图](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/ES/guanwang.png)

可以看到，ES 社区表示**务必**要预留足够大的内存给 Lucene 作为缓存使用，不要将物理主机的所有内存都分配给 ES JVM 实例使用，不然会大大影响性能。

但是，直到这里，我们的疑问还是没有得到解决，为什么明明 ES 提示占用了这么多内存，我们在 zabbix 上还看到这么多剩余的可用内存？

&nbsp;

## 原因

由于没有搜到更多的内容，于是便想到 ES 的 github 仓库上面碰碰运气，结果发现了这个 ISSUE ：[ram.percent值具有误导性](https://github.com/elastic/elasticsearch/issues/32393)。

这个 ISSUE 在 18 年提交，目前还是 open 的，提交者表示，`ram.percent` 这个值显示的是「空闲内存」，即完全没有被使用的内存，而不是「可用内存」，所以这个值实际上让人感到迷惑。

什么意思呢？该用户还贴出了一个链接说明这件事：[Linux吃了我的内存！](https://www.linuxatemyram.com/)

![Linux吃了我的内存](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/ES/atemyram.png)

这篇文章很清晰地说明了，Linux 的磁盘缓存可以让操作系统更快地相应，而且没有缺点，**它并不会以任何方式占用应用程序的内存**。

![Linux吃了我的内存](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/ES/atemyram2.png)

如果主机上的其他应用程序需要申请内存，这些被借出去作为磁盘缓存的内存会立即回收。

![Linux吃了我的内存](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/ES/atemyram3.png)

这篇文章还列举了很多例子，包括实际操作验证，可以打开链接自行了解。

&nbsp;

## 总结

由于 ES 的实时性需求，Lucene 在设计时为了提高整体性能，将会大量使用上面所描述的「操作系统磁盘缓存」，使得操作系统的内存看起来被占用了非常多，实则不然，建议在监控 ES 的时候，**主要以观察 heap 内存为主**，实际物理主机占用的内存空间，可以在 zabbix 查看。