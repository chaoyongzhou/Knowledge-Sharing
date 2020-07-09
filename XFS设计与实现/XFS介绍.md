1、背景

XFS是一个面向小文件的高性能存储系统，文件带目录存储。

定义：在XFS中，小文件是指不超过64MB大小的文件。

XFS按照单盘单进程管理，支持单盘的上下线操作。

XFS是RFS（Random access File System）的延续，增加了裸盘管理和DMA分级缓存能力。

2、架构

![图 2-1]()


XFS对外提供三类接口：

（1）REST API： HTTP协议，支持长连接和短连接通信

（2）XFS API：私有协议，长连接通信

（3）CONSOLE：私有协议，长连接通信

3、编译

（1）编译XFS主可执行文件

    make -j xfs -f Makefile.xfs

（2）编译XFS初始化工具可执行文件

     make -j xfs_tool -f Makefile.tool

（3）编译CONSOLE可执行文件

    make console -f Makefile.console

4、运行

XFS的运行依赖归一后的磁盘软链和配置文件。

其中，配置文件描述XFS进程对应的IP地址、服务端口、网络拓扑和配置参数，本文不表。

4.1、环境准备

创建磁盘软链，归一处理。举例：

    # ln -s /dev/sdf /data/cache/rnode1

4.2、初始化XFS元数据

    # bash /usr/local/xfs/bin/xfs_init.sh

4.3、启动XFS

    # /usr/local/xfs/bin/xfs -tcid 10.10.67.18 -node_type xfs -xfs_sata_path /data/cache/rnode1 -logp /data/proclog/log/xfs -sconfig /usr/local/xfs/bin/config.xml -d

启动命令完整格式为：

    # /usr/local/xfs/bin/xfs -tcid <tcid> -node_type xfs -xfs_sata_path <sata path> [-xfs_ssd_path <ssd path>] -logp /data/proclog/log/xfs -sconfig /usr/local/xfs/bin/config.xml -d

即，ssd盘可选。

4.4、停止XFS

    # kill -15 <xfs pid>

5、接口

XFS提供三种访问接口如下。

5.1、 REST API接口

举例：

写文件

    # curl -d "hello world" http://127.0.0.1:718/xfs/setsmf/top/level01/level02/level03/a.dat

读文件

    # curl -v http://127.0.0.1:718/xfs/getsmf/top/level01/level02/level03/a.dat

删文件

    # curl -v http://127.0.0.1:718/xfs/dsmf/top/level01/level02/level03/a.dat

删目录

    # curl -v http://127.0.0.1:718/xfs/ddir/top/level01

5.2、CONSOLE口

启动CONSOLE口

    # /usr/local/console/bin/console -tcid 0.0.0.64 -sconfig /usr/local/xfs/bin/config.xml

举例：

查看name node信息

    # hsxfs 0 show npp on tcid 10.10.67.18 at console

查看data node信息

    # hsxfs 0 show dn on tcid 10.10.67.18 at console

5.3、XFS API

举例：

写文件

    EC_BOOL cxfs_write(const UINT32 cxfs_md_id, const CSTRING *file_path, const CBYTES *cbytes)

更新文件（覆盖写）

    EC_BOOL cxfs_update(const UINT32 cxfs_md_id, const CSTRING *file_path, const CBYTES *cbytes)

读文件

    EC_BOOL cxfs_read(const UINT32 cxfs_md_id, const CSTRING *file_path, CBYTES *cbytes)

读部分文件

    EC_BOOL cxfs_read_e(const UINT32 cxfs_md_id, const CSTRING *file_path, UINT32 *offset, const UINT32 max_len, CBYTES *cbytes)

删文件

    EC_BOOL cxfs_delete_file(const UINT32 cxfs_md_id, const CSTRING *path)

删目录

    EC_BOOL cxfs_delete_dir(const UINT32 cxfs_md_id, const CSTRING *path)

XFS API是XFS对外暴露接口的全集，截止目前共115个接口。

6、设计

6.1、存储模型

存储本质上由两部分组成：name node和data node，即要完成两次映射。
Name node的作用就是从文件路径映射到文件内容的实际存储位置：

    Name node： path -------→ location

比如，读文件前，我们把文件的完整路径告诉给name node，name node检索后将结果返回，这样我们就知道文件存储在哪块磁盘、哪个物理文件（可选）、偏移量是多少、文件长度是多少等信息：

    （disk_no, file_no, offset, len）= NameNode(path)

Data node用来管理磁盘空间的分配与回收。 比如，写文件时，我们把文件长度告诉给data node，data node经过一番折腾后返回，告诉我们在哪块磁盘、哪个物理文件（可选）、起始偏移量是多少的地方有所需要的空间：

    （disk_no, file_no, offset） = DataNode(file_size)

示意图：

![图 6-1-1]()


6.2、Name node设计

name node采用inode方式管理目录结构。

6.2.1、dnode/fnode

XFS是带目录存储系统，在设计中，dnode用来表达目录层次结构，fnode用来记录文件位置信息、文件大小等，其中记录文件位置信息的就是inode，包含虚拟磁盘号（disk no），块号（block no）、页码（page no）等。

dnode用红黑树（rb tree）组织，树根就是当前目录。由于子目录依然是红黑树，因此XFS的name node本质上是多层红黑树。

比如，XFS存储三个文件：/a/b/c, /a/d/e, /a/d/f，name node中组织形式表达为：

![图 6-2-1-1]()

其中，{a, b, d}是一棵红黑树，{b, c}是一棵红黑树，{d, e, f}是一棵红黑树。

6.2.2、item/key

XFS中文件或目录的路径（path），按照斜线（/）分割成path segment（又称key，以下用该称呼）。

XFS以item的方式来组织dnode/fnode，且包含key信息。

当前，主流XFS版本的key按照63字节存储，item只记录key在key zone中的相对偏移量。

如果客户端请求文件的路径的key超过63字节，则通过MD5压缩到32字节。

在不影响理解的语境下，约定成俗，我们口头上不区分item/inode。比如说一个inode占64字节，实际是指一个item占64字节。

![图 6-2-2-1]()

6.3、Data node设计

data node负责磁盘空间的分配与回收。

data node按照卷（volume）、虚拟磁盘（virtual disk）、块（block）、页（page）四级管理磁盘空间。其中，volume最大支持到64TB， virtual disk固定1TB（或其它，比如32GB），block固定64MB、page最小4KB（或其它，比如32KB）。

data node可分配的最小单位为page，即4KB（或其它，比如32KB），最大单位为block，即64MB。

 ![图 6-3-1]()


data node空间管理核心算法是分裂与合并，是一对互逆操作。

6.4、分裂与合并

分裂原理一句话概括为：一分为二取其左，反复迭代。

合并原理一句话概括为：看奇偶，看空闲，左右合并，反复迭代。

分裂与合并与伙伴系统高度相似。

具体示例：假设最小页定义为4KB，现有一个128KB的空闲空间，用绿色块代表空闲的4KB页，红色块代表已被占用的4KB页。

（1）分配1个页

![图 6-4-1]()

S1~S5为分裂步骤

（2）再分配5个页

![图 6-4-2]()

S1~S2为分裂步骤

（3）再分配9个页

![图 6-4-3]()

S1~S3为分裂步骤

（4）再分配3个页

![图 6-4-4]()

S1为分裂步骤

（5）再分配3个页

![图 6-4-5]()

S1为分裂步骤

（6）释放第0页

![图 6-4-6]()

M1为合并终止点

（7）再释放第28~30页

![图 6-4-7]()

M1为合并终止点

（8）再释放第8~12页

![图 6-4-8]()

M1为合并终止点

（9）再释放第4~6页

![图 6-4-9]()

M1为合并终止点


6.5、磁盘布局

XFS管理的磁盘一般由元数据区和数据区构成。元数据主要是name node和data node，数据区是指存储文件内容的区域。

6.5.1、磁盘布局

根据位置的不同，磁盘布局有如下三种方式。不失一般性，以SATA盘示例。假设XFS虚拟磁盘大小为32GB。

（1）元数据区在磁盘头部位置

![图 6-5-1-1]()


（2）元数据区在磁盘尾部位置

![图 6-5-1-2]()

（3）元数据区在磁盘之外

SATA盘为纯数据盘，元数据放在SSD盘。

![图 6-5-1-3]()

XFS对以上三种布局都支持过，最新版本保留对第（2）、（3）两种方式的支持。

6.5.2、元数据区布局

![图 6-5-2-1]()

其中，config用来引导XFS的启动，记录各区域偏移量和大小信息，包括但不限于：
（1）SATA盘的容量、
（2）数据区在SATA盘的起始偏移量
（3）SATA盘XFS元数据大小
（4）虚拟磁盘大小、虚拟磁盘个数
（5）name node在SATA盘的偏移范围
（6）data node在SATA盘的偏移范围
（7）SSD盘的容量（如果开启分级缓存中的SSD缓存）

7、分级缓存

分级缓存（内存缓存、SSD缓存、SATA缓存）的效果，是将内存、SSD、SATA作为一个整体，暴露给XFS访问，即从XFS（或其它应用）视角看到的是一块SATA盘，通过DMA接口访问；而从DMA视角看到的是三级缓存，需要根据性能优先、读写优先的原则决定访问顺序，并根据一定的策略决定数据的升级与降级。

XFS管理它所看到的“SATA盘”的元数据，而DMA则管理内存缓存和SSD缓存的元数据。内存缓存和SSD缓存本质上是一个小型的、简化版的XFS，即也是一个存储系统。

关于分级缓存细节，请参见相关文档。