# RFS技术原理与实现剖析

目录

1  背景

2  架构

2.1  网络图

2.2  接口图

2.3  交互图

3  接口

3.1  REST API接口

3.2  CONSOLE口

3.3  RFS API接口

4  name node原理与实现

4.1  dnode/fnode/inode

4.2  item/key

5  data node原理与实现

5.1  分裂与合并原理

5.2  分裂与合并实现

5.3  空间优化



## 1 背景

RFS（Random access File System）是当年为了解决nginx用作CDN Cache软件时，没有相应的存储系统（Cache Storage），从而基于BGN平台开发的。

RFS是BGN平台上的众多模块之一，其基础组件（比如网络通信、内存管理、日志系统、接口库等）依赖BGN。

CDN行业存在一个行业的痛点：刷新。RFS在设计之初希望切除该痛点，因此选择带目录存储的架构，实现URL和目录的真实秒刷能力。

根据存储与业务分离原则，RFS不负责切片、不负责回源，仅负责读、写、删三个核心功能。在XCACHE设计时，由于合并回源功能的汇聚点选择落在存储上，因此RFS额外承担了其中的汇聚辅助功能。

RFS面向块设备，但当时没有直接进行裸盘管理，其进化版本XFS提供了裸盘管理、AIO、分级缓存等能力。

RFS是被重度设计的，元数据较大，这使得通过调整元数据大小、修改元数据组织形式，即可衍生出不同种类的存储，比如HFS、SFS等。

## 2 架构

### 2.1 网络图

![图 2-1](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/RFS%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/rfs-2-1.png)

	注: NP - Name Space/ Name Node, DN - Data Node

（1） share-nothing

（2）scale-out

（3）单盘单进程管理

（4）负载均衡

### 2.2 接口图

![图 2-2](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/RFS%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/rfs-2-2.png)

（1） 控制台访问（CONSOLE）

（2）远端访问（RFS API/ REST API）

（3）中继访问（RFS API）

### 2.3 交互图

![图 2-3](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/RFS%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/rfs-2-3.png)

客户端访问RFS时，RFS首先查询Name Node，获得location信息，然后访问Data Node，读写数据。Data Node与磁盘交互。

## 3 接口

RFS提供三种访问接口

### 3.1 REST API接口

举例：

	写文件：    curl -d "hello world" http://127.0.0.1:718/rfs/setsmf/top/level01/level02/level03/a.dat
	
	读文件： curl -v http://127.0.0.1:718/rfs/getsmf/top/level01/level02/level03/a.dat
	
	删文件： curl -v http://127.0.0.1:718/rfs/dsmf/top/level01/level02/level03/a.dat
	
	删目录：    curl -v http://127.0.0.1:718/rfs/ddir/top/level01

### 3.2 CONSOLE口

启动CONSOLE口： ./console -tcid 0.0.0.64

举例：

	查看name node信息：hsrfs 0 show npp on tcid 10.10.67.18 at console
	
	查看data node信息：hsrfs 0 show dn on tcid 10.10.67.18 at console

### 3.3 RFS API接口：

即BGN接口，通过私有协议访问RFS接口。

举例：

	写文件：EC_BOOL crfs_write(const UINT32 crfs_md_id, const CSTRING *file_path, const CBYTES *cbytes)
	
	读文件：EC_BOOL crfs_read(const UINT32 crfs_md_id, const CSTRING *file_path, CBYTES *cbytes)
	
	删文件：EC_BOOL crfs_delete_file(const UINT32 crfs_md_id, const CSTRING *path)
	
	删目录：EC_BOOL crfs_delete_dir(const UINT32 crfs_md_id, const CSTRING *path)

## 4 name node原理与实现

name node采用inode方式管理目录结构。

### 4.1 dnode/fnode/inode

RFS是带目录存储系统，在设计中，dnode用来表达目录层次结构，fnode用来记录文件位置信息、文件大小等，其中记录文件位置信息的就是inode，包含虚拟磁盘号（disk no），块号（block no）、页码（page no）等。

dnode用红黑树（rb tree）表达目录层次信息，树根就是当前目录，树叶就是该目录下的文件，树枝则是该目录下的子目录。由于子目录依然是红黑树，因此RFS的name node的组织形式，本质上是多层红黑树。

比如，RFS存储三个文件：/a/b, /a/c/d, /a/c/e，name node中组织形式表达为：

![图 4-1](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/RFS%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/rfs-4-1.png)

其中，{a, b, c}是一棵红黑树，{c, d, e}是一棵红黑树。


### 4.2 item/key

RFS中文件或目录的路径（path），按照斜线（/）分割成path segment（又称key，以下用该称呼）。

RFS以item的方式来组织dnode/fnode，且包含key信息。

按照Linux POSIX文件系统的路径标准，RFS的key长度应为255字节，那么item对齐的话，需要512字节。RFS在发展过程中，为了缩减item的大小，放弃了支持这一标准。

当前，主流RFS版本的key按照63字节存储。以前的版本中，key嵌入item，总共占128字节，后来分离，各占64字节，item只记录key在key zone中的相对偏移量。

如果客户端请求文件的路径的key超过63字节，则通过MD5压缩到32字节。

在不影响理解的语境下，约定成俗，我们口头上不区分item/inode。比如说一个inode占64字节，实际是指一个item占64字节。

![图 4-2](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/RFS%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/rfs-4-2.png)

## 5 data node原理与实现

data node负责磁盘空间的分配与回收。

data node按照卷（volume）、虚拟磁盘（virtual disk）、块（block）、页（page）四级管理磁盘空间。其中，volume最大支持到64TB， virtual disk固定1TB（或其它，比如32GB），block固定64MB、page最小4KB。

data node可分配的最小单位为page，即4KB，最大单位为block，即64MB。

![图 5-0](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/RFS%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/rfs-5-0.png)


data node空间管理核心算法是分裂与合并，是一对互逆操作。


### 5.1 分裂与合并原理

分裂原理一句话概括为：一分为二取其左，反复迭代。

合并原理一句话概括为：看奇偶，看空闲，左右合并，反复迭代。

以从64KB连续空间上分配4KB为例：

![图 5-1-1](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/RFS%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/rfs-5-1-1.png)

将64KB一分为二，右边32KB标记为空闲空间，

![图 5-1-2](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/RFS%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/rfs-5-1-2.png)

左边32KB再一分为二，右边16KB标记为空闲空间，

![图 5-1-3](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/RFS%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/rfs-5-1-3.png)

左边16KB再一分为二，右边8KB标记为空闲空间，

![图 5-1-4](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/RFS%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/rfs-5-1-4.png)

左边8KB再一分为二，右边4KB标记为空闲空间

![图 5-1-5](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/RFS%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/rfs-5-1-5.png)

左边4KB即为所分配的空间。

![图 5-1-6](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/RFS%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/rfs-5-1-6.png)


64K Model表示64KB空闲空间的集合，并依次编号为page 0, page 1, ....，上述过程可以描述为：

	（64K Model, page 0） => (32K Model, page 0) + (32K Model, page 1)
	
	（32K Model, page 0） => (16K Model, page 0) + (16K Model, page 1)
	
	（16K Model, page 0） => (8K Model, page 0)  + (8K Model, page 1)
	
	（8K Model, page 0） => (4K Model, page 0)  + (4K Model, page 1)

空间回收时触发合并：

如果回收(4K Model, page 0)，则检查(4K Model, page 1)是否空闲，若是，则合并，即

	(4K Model, page 0)  + (4K Model, page 1) => （8K Model, page 0）

重复该过程：

检查(8K Model, page 1)是否空闲，若是，则合并，即

	(8K Model, page 0)  + (8K Model, page 1) => （16K Model, page 0）

检查(16K Model, page 1)是否空闲，若是，则合并，即

	(16K Model, page 0)  + (16K Model, page 1) => （32K Model, page 0）

检查(32K Model, page 1)是否空闲，若是，则合并，即

	(32K Model, page 0)  + (32K Model, page 1) => （64K Model, page 0）

注意，如果回收(4K Model, page 1)，则应检查(4K Model, page 0)是否空闲，即要判断page no的奇偶性。


![图 5-1-7](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/RFS%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0/rfs-5-1-7.png)

RFS以64MB为最大可分配空间、4KB为最小可分配空间，因此分裂最深是从64MB到4KB，合并最深是从4KB到64MB。

RFS所管理的磁盘空间起始偏移量是64MB的整数倍。

分裂与合并算法有如下特点：

	（1）空间偏移量规整
	
	     比如，4KB空间偏移量必定4KB对齐，8KB空间偏移量必定8KB对齐
	
	（2）分裂与合并算法互逆
	
	（3）分裂与合并算法有稳定的时间复杂度

### 5.2 分裂与合并实现

首先将磁盘空间按照volume、virtual disk、block划分。然后在block中建立64M Model，32M Model， .... ，4K Model的空闲页表，初始时仅有64M Model表中有一个表项，即一个64MB空闲页。

按Model建立空闲页表的做法，同样适用于virtual disk和volume。比如，block中如果有64MB空闲页，那么这个block就挂到对应的virtual disk的64M Model中。

来看看如何分配一个4KB的空间：

首先，按照4K Model，8K Model，...，64M Model的次序查volume的空闲页表，找到第一个virtual disk，比如是在64K Model中找到的，这就是说该virtual disk存在64KB空闲页，但不存在比64KB更小的空闲页。

然后，按照64K Model, 128K Model, ..., 64M Model的次序查virtual disk的空闲表，找到第一个block，比如是在64K Model中找到的。

然后，按照64K Model, 128K Model, ..., 64M Model的次序查block的空闲表，找到第一个page并摘除， 比如是在64K Model中找到的，进行分裂操作，分裂出来的空闲页挂到对应的Model表中，返回最终分配到的4KB空间。

还没完，继续调整一下：

检查64K Model的block的空闲表是否已空，如果非空，流程结束；如果空，则将该block从virtual disk的64K Model表中摘除；

检查64K Model的virtual disk的空闲表是否已空，如果非空，流程结束；如果空，则将该virtual disk从volume的64K Model表中摘除，流程结束。

回收4KB的空间是一个逆过程：

根据4KB页的奇偶性，检查对应的block的4K Model表中的相邻页是否空闲，如果不空闲，则将该4KB页挂入block的4K Model表中，流程结束；否则，将相邻页从block的4K Model表中摘除，合并为一个8KB页；
再根据8KB页的奇偶性，检查对应的block的8K Model表中的相邻页是否空闲，...，如此迭代，至多迭代到block的64M Model表就会终止。相应地，block、virutal disk作出Model表的调整。

从以上流程中可以看到，Model表在频繁执行增、删、查操作，效率是关键，技术手段有：

	（1）位操作：与、或、移位。比如Model的确认、Model表的切换，page no的切换等。
	
	（2）位图：在每个Model表中建立位图，每比特表示该Model下对应页的空闲与否，用空间换时间。
	
	（3）红黑树：Model表上的空闲页按照红黑树管理，页的插入与删除操作的时间复杂度稳定。

### 5.3 空间优化

假如将所有Model表的所有空闲页投影到一条直线上，那么一定不会出现投影重叠的现象，因为空闲页是分裂而来的。

由于空闲页至少是4KB，所以所有Model表的所有空闲页总数不会超过磁盘空间可以划分成4KB页的个数（磁盘空间 / 4KB）。

进一步，由于合并算法的存在，所以所有Model表的所有空闲页总数不会超过磁盘空间可以划分成4KB页的个数一半（磁盘空间 / 4KB / 2）。

也就是说，要管理所有Model表的所有空闲页，只需要准备至多（磁盘空间 / 4KB / 2）个RB NODE即可。
