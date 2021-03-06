# 大文件存储解决方案

## 背景

1. 客户原来的文件服务器不可用，要用到我司的文件服务器。故要把原客户文件服务器里的近60TiB的数据转移到这边。

2. 后续支持跟客户方交互调用接口上传文件，按照一定的业务规则（命名方式）存放到服务器中。

## 问题

1. 现有的文件服务器不支持60TiB+的存储空间

2. 如果用新的服务器，那么有没有支持60TiB或者是PiB级别的物理磁盘

3. 如果以上不可行的话，那只能采用分布式网络文件存储，又如何具体实施

## 提出的解决方案

首先要确认现代系统中是有大容量的磁盘，根据[资料显示](https://zh.wikipedia.org/wiki/NTFS)（维基百科），NTFS 格式下的单卷最大容量与文件最多数据信息如下

```
最大文件尺寸理论值 = 16EiB – 1KiB  

最大文件尺寸实际值 = 16TiB - 64KiB（windows7，windows server 2008R2-）            

                 = 256TiB – 64KiB（windows8，windows server 2012+）  

最大卷容量理论值 = 2^64 – cluster size（默认是64KiB）  

最大卷容量实际值 = 256TiB - 64KiB  

最大文件数量 = 2^32 - 1 = 4294967295  最

长文件名 = 255 个 utf-16 编码单元  
```

以上数据支持的系统：

Windows NT 3.1+、OSX 10.3+、Linux 内核版本2.2+、ReactOS（只读）

单个磁盘驱动是可以到达TiB以上的。配合例如群晖NAS可以到达64TiB的容量，还节省了运维资源（缺点是成本相对较高）

### 磁盘阵列

磁盘整列又名RAID，全名是[独立磁盘冗余阵列](https://zh.wikipedia.org/wiki/RAID)（Redunant Array of InDependent Disks）。

就是说可以将多个高性能的物理磁盘驱动器组成一个子系统，从而能提供海量的存储性能和数据冗余技术。

磁盘阵列实际上是虚拟化技术，把多个物理磁盘虚拟成一个独立的大容量的逻辑驱动器。对外部来说RAID就是一个单一的、快速可靠的大容量磁盘驱动器。

这样的话，就能让运维人员维护这个单个驱动器来组织和存储应用数据，并且还支持动态扩容，增加磁盘驱动器，还可以自动进行数据校验和数据重建。

#### 优势

减少了开发人员的研发工作量，让其专心开发业务逻辑。磁盘的扩容和与维护交给专门的运维人员维护。并且达到了职责分离的效果。

#### 关于不同的RAID方案的成本问题

磁盘阵列排列组合方式有很多种，有的很经济划算，有的非常可靠但成本很高。

经查资料，一般用作文件存储器的方案一般采用 RAID0，RAID10，RAID5这三种组合方式。但是各有各的优劣势，可以酌情选择。

RAID0：磁盘数量2+

读写性能强，但是不具备数据恢复能力，也就是说磁盘坏了，数据就找不回。但是价格最低，磁盘利用率100%

RAID5：磁盘数量3+

具有数据恢复能力，读性能高，写性能低。磁盘利用率（N-1）/N，假设磁盘有3个，那么磁盘利用率就是3-1/3。成本折中。

RAID10：磁盘数量4个或2N（N>=2）

先做镜像（具有数据恢复能力），再做条带，磁盘利用率位50%（即我要存8TiB的数据，我要16TiB的磁盘空间，剩下的8TiB做数据备份，故障率要比RAID0要低。成本高

### Hadoop 分布式存储

Hadoop与RAID不同，Hadoop是将多个物理机通过软件（交换机）集中管理的一个分布式存储的基础架构。并且具有分布式计算能力（多个物理机，多机多核CPU计算能力）。安装简单，只需要再任意个物理机上（3个以上）安装hadoop环境并且即可。

hadoop主要由两部分组成，代表它核心的两个功能

HDFS：分布式文件系统，通过一个命名节点存储文件元信息，然后映射到多个数据节点上的具体文件信息。

MapReduce：是一个分布式计算框架，核心思想就是把计算任务分配给集群内的服务器里执行，通过对计算任务的拆分，再根据任务调度器对任务进行分布式计算。

#### 优势

开源，免费（只需要物理机即可）易安装，管理，适合大文件存储（支持TiB/PiB级别文件上传），性能强悍。支持数据恢复，自我检测故障。

hadoop最主要的功能点用在对大数据的分析与抓取上，也就是要利用MapReduce来计算出文件的内容

适用的主要场景：超大文件上传、大数据分析挖掘（如广告精准投放，日志分析，机器学习等）

#### 劣势

尽管hadoop是开源免费的，但是物理机却不是免费的。同样的内存要求，物理机和单磁盘驱动器比较来说，成本要高出不少。

并且针对于.net，Hadoop没有客户端与之进行交互。目前查到的只有微软官方的AZure云才支持对Hadoop的支持。

## 如何选型

因为要求的文件服务器只是简单的存储与读取，并不需要对文件进行挖掘分析，我认为用hadoop并不合适。再者技术实现难度上也比磁盘阵列要难的多。