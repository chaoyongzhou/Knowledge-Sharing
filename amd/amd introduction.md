# 三级存储AMD技术介绍

1 背景

2 存储基本知识

2.1 四层缓存

2.2 page cache

2.3 direct io

2.4 mmap

2.5 数据丢失

3 AIO基础知识

3.1 glibc版本的aio

3.2 内核版本的aio

3.2.1 关于io_cancel

3.2.2 关于偏移量和长度

3.2.3 优缺点

4 AIO模块

4.1 性能数据

5 存储设计

6 内存缓存（Mem Cache）

7 SSD缓存（SSD Cache）

8 三级缓存

8.1 三级缓存模型

8.2 AMD设计

8.3 流控与重试

8.4 页面流转

# 1 背景
   
AMD = AIO + Mem Cache + SSD Cache

AMD技术产生的背景主要有两点：

（1） SATA盘的IO能力挖掘不够，阻塞式随机读写，只能达到磁盘性能的50%。

（2） 分级缓存下，Page Cache技术的运用，按照SSD -> Page Cache -> SATA读写顺序进行，不符合性能优先、读写优先原则，需要调整到 Mem Cache -> SSD -> SATA读写顺序。

# 2 存储基本知识

## 2.1 四层缓存

读（写）磁盘文件时，通常要经历四个缓存：

    (1) 设备层缓存（disk cache）

    (2) 内核层缓存（page cache）

    (3) GLIBC缓存（clib buffer）

    (4) 应用层缓存（application buffer）

以读文件为例：

如果应用层自己有缓存，那么应用层可以直接从自己的缓存读取，无需POSIX文件系统调用；

否则， 如果应用层使用的是f**接口（比如fread, fwrite等），则接口检查GLIBC的缓存是否命中；否则，系统调用，进入内核查看page cache是否命中；否则，落盘访问。

如果开启O_DIRECT，则可以跳过page cache，直接落盘访问。

![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/2-1-1.png)


在服务器端，通常不考虑GLIBC缓存，而设备层缓存（disk cache）不受应用层和内核控制，也可以不考虑。磁盘的设备层缓存（disk cache）又称DRAM，缺省关闭。

因此，服务器端主要考虑应用层缓存（application buffer）和内核层缓存（page cache）两种。

## 2.2 page cache

内核缓存（page cache）又分为两种：

一种是buffer cache，用于裸盘访问，见free命令输出的buffers列；

另一种是page cache，用于非裸盘访问，见free命令输出的cached列。


![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/2-2-1.png)


## 2.3 direct io

内核缓存（page cache）是可选项，通过文件描述符的O_DIRECT来控制开启与否：

当设置O_DIRECT标志时，系统调用跳过内核缓存（page cache），通过IO队列（IO queue）和设备驱动（driver）与磁盘交互，内核不缓存数据，俗称direct io。

当不设置O_DIRECT标志时，系统调用读写请求与内核缓存（page cache）交互，内核线程pdflush检测内核缓存（page cache）的脏页并刷新到磁盘，或者从磁盘加载到内核缓存（page cache）。

设置O_DIRECT标志的方法： 可以在打开文件（open）时，传入标志；或者通过fcntl设置。示例代码如下：

打开时设置：

	fd = open(file_name, O_DIRECT | O_RDWR, 0666);

或者设置已打开文件描述符属性： 


    int  flags;

    flags = fcntl(fd, F_GETFL);

    if(-1 == flags)
    {
        return (-1);
    }

    if(0 > fcntl(fd, F_SETFL, flags | O_DIRECT))
    {
        return (-1);
    }

注：内核缓存（page cache）缺省开启。如需关闭，需要显式设立O_DIRECT标志。

direct io的第一个坑是对齐。

在读写文件时，需要四个参数：文件描述符（fd），文件的偏移量（offset），读写长度（size），应用层内存（buffer）。

direct io要求后三个参数：文件的偏移量（offset），读写长度（size），应用层内存（buffer），必须与磁盘扇区大小对齐，通常是512字节对齐。

这为应用层对文件的读写操作带来了不便，换言之，应用层对文件的任意偏移量的访问便捷性，要归功于page cache，后者对应用层屏蔽了对齐的限制。

## 2.4 mmap

mmap本质是将内核空间映射到用户空间。比如direct io关闭时，通过mmap将 文件（page cache）映射到用户空间，以减少内核态与用户态之间的切换，应用层通过基址+偏移量的方式访问连续的虚拟内存，mmap将该访问映射到文件的访问，并自动落盘，即对mmap内存的操作等同于对文件的操作。

在direct io开启时，使用mmap映射的方式不具备一般性，是否需要满足direct io的对齐要求等，个人尚未实践经验，可行性存疑。

## 2.5 数据丢失

多级缓存的目标是数据的最终一致。读写数据在多级缓存之间存在丢失的问题，以及数据不一致的问题。


- 考虑内核缓存（page cache）

应用层将数据交给内核缓存（page cache）后立即返回，认为写成功。而page cache并非立即交给驱动（driver）写盘，而是经过一段时间后，由内核线程pdflush持久化到磁盘上。

设备掉电： 假如在数据持久化到磁盘之前或之中，突然掉电，那么数据丢失；

进程退出： 假如在数据持久化到磁盘之前或之中，应用层进程意外退出，内核线程pdflush将数据持久化到磁盘，数据不丢失。


- 考虑应用层缓存（application buffer）

这里姑且认为应用层缓存是内存缓存（mem cache）。

应用层将数据交给内存缓存（mem cache）后立即返回，认为写成功。

设备掉电： 在数据持久化到SSD缓存（SSD Cache）前，如果掉电，数据丢失；

进程退出： 在数据持久化到SSD缓存（SSD Cache）前，如果进程意外退出，数据丢失。

程序异常： 在数据持久化到SSD缓存（SSD Cache）前，如果程序异常，比如内存不足导致必须释放掉一部分缓存时，数据丢失，应竭力避免此类异常。


- 考虑SSD缓存（SSD Cache）

数据降级持久化到SSD缓存（SSD Cache）后，且尚未降级持久化到SATA盘，如果SSD盘损坏，数据丢失或读到脏数据。

此时，应用层数据认为数据已持久化，在SSD缓存（SSD Cache）数据丢失后，从SATA盘加载，但SATA盘并没有接收来自SSD盘降级的数据，出现不一致。

# 3 AIO基础知识

AIO即异步IO，本文特指磁盘的异步IO。

AIO的核心是读写磁盘时，不用同步等待读写完成，而是提交一个异步读写事件，在事件完成时反向通知应用来处理即可。这种反向通知的机制就是熟知的触发机制，比如信号机制、epoll机制等。

Linux有两个版本的AIO：一个是glibc用线程模拟的aio版本，一个是Linux内核实现的aio版本（又称原生AIO）。

## 3.1 glibc版本的aio

该版本提供的接口通俗易懂，简单易用，支持direct io和非direct io，可以利用内核缓存（page cache）提高效率。但是由于多线程的缘故，真实的效率并不高，不适用高性能服务器。

接口列表如下：


	int aio_read(struct aiocb *aiocbp);  /* 提交一个异步读 */
	int aio_write(struct aiocb *aiocbp); /* 提交一个异步写 */
	int aio_cancel(int fildes, struct aiocb *aiocbp); /* 取消一个异步请求（或基于一个fd的所有异步请求，aiocbp==NULL） */
	int aio_error(const struct aiocb *aiocbp);         /* 查看一个异步请求的状态（进行中EINPROGRESS？还是已经结束或出错？） */
	ssize_t aio_return(struct aiocb *aiocbp);          /* 查看一个异步请求的返回值（跟同步读写定义的一样） */
	int aio_suspend(const struct aiocb * const list[], int nent, const struct timespec *timeout); /* 阻塞等待请求完成 */
	
其中，struct aiocb主要包含以下字段：

	int                   aio_fildes;        /* 要被读写的fd */
	void *                aio_buf;           /* 读写操作对应的内存buffer */
	__off64_t             aio_offset;        /* 读写操作对应的文件偏移 */
	size_t                aio_nbytes;        /* 需要读写的字节长度 */
	int                   aio_reqprio;       /* 请求的优先级 */
	struct sigevent       aio_sigevent;      /* 异步事件，定义异步操作完成时的通知信号或回调函数 */

实现原理：


	1、异步请求被提交到request_queue中；
	2、request_queue实际上是一个表结构，"行"是fd、"列"是具体的请求。也就是说，同一个fd的请求会被组织在一起；
	3、异步请求有优先级概念，属于同一个fd的请求会按优先级排序，并且最终被按优先级顺序处理；
	4、随着异步请求的提交，一些异步处理线程被动态创建。这些线程要做的事情就是从request_queue中取出请求，然后处理之；
	5、为避免异步处理线程之间的竞争，同一个fd所对应的请求只由一个线程来处理；
	6、异步处理线程同步地处理每一个请求，处理完成后在对应的aiocb中填充结果，然后触发可能的信号通知或回调函数（回调函数是需要创建新线程来调用的）；
	7、异步处理线程在完成某个fd的所有请求后，进入闲置状态；
	8、异步处理线程在闲置状态时，如果request_queue中有新的fd加入，则重新投入工作，去处理这个新fd的请求（新fd和它上一次处理的fd可以不是同一个）；
	9、异步处理线程处于闲置状态一段时间后（没有新的请求），则会自动退出。等到再有新的请求时，再去动态创建；

此版本不做进一步介绍。

## 3.2  内核版本的aio

内核版本的aio仅支持direct io，使用的门槛比较高，执行效率也高。该版本的接口没有对应的glibc接口，因此需要采用系统调用的方式，比如：


	syscall(__NR_io_setup, nr_reqs, ctx)


接口列表如下：


	int io_setup(int maxevents, io_context_t *ctxp);  /* 创建一个异步IO上下文（io_context_t是一个句柄） */
	int io_destroy(io_context_t ctx);  /* 销毁一个异步IO上下文（如果有正在进行的异步IO，取消并等待它们完成） */
	long io_submit(aio_context_t ctx_id, long nr, struct iocb **iocbpp);  /* 提交异步IO请求 */
	long io_cancel(aio_context_t ctx_id, struct iocb *iocb, struct io_event *result);  /* 取消一个异步IO请求 */
	long io_getevents(aio_context_t ctx_id, long min_nr, long nr, struct io_event *events, struct timespec *timeout)  /* 等待并获取异步IO请求的事件（也就是异步请求的处理结果） */
	
其中，struct iocb主要包含以下字段：

	__u16     aio_lio_opcode;     /* 请求类型（如：IOCB_CMD_PREAD=读、IOCB_CMD_PWRITE=写、等） */
	__u32     aio_fildes;         /* 要被操作的fd */
	__u64     aio_buf;            /* 读写操作对应的内存buffer */
	__u64     aio_nbytes;         /* 需要读写的字节长度 */
	__s64     aio_offset;         /* 读写操作对应的文件偏移 */
	__u64     aio_data;           /* 请求可携带的私有数据（在io_getevents时能够从io_event结果中取得） */
	__u32     aio_flags;          /* 可选IOCB_FLAG_RESFD标记，表示异步请求处理完成时使用eventfd进行通知 */
	__u32     aio_resfd;          /* 有IOCB_FLAG_RESFD标记时，接收通知的eventfd */
	
其中，struct io\_event主要包含以下字段：

	__u64     data;                  /* 对应iocb的aio_data的值 */
	__u64     obj;                   /* 指向对应iocb的指针 */
	__s64     res;                   /* 对应IO请求的结果（>=0: 相当于对应的同步调用的返回值；<0: -errno） */


这里高效和奇妙之处在于aio\_buf，这是用户态地址空间中分配的，而aio请求时交给内核设备驱动（driver）执行，也就是说，内核直接访问用户空间地址。

也正因为此，内核版本的aio要求传入的参数aio\_buf（应用层内存），aio\_nbytes（文件读写长度）, aio\_offset（文件偏移量）必须满足设备驱动（driver）的对齐要求。

用于反向通知的字段是aio\_resfd。内核版aio的实现机制是，如果aio请求指定了事件通知文件描述符（aio\_resfd），那么在读写完成后，内核向该文件描述符指向的文件中写入一个整数，表示有多少个读写事件完成。

这个描述符可以通过eventfd2或eventfd生成一个，然后放到epoll中查询读事件，当内核向该描述符中写入后，epoll读事件被触发。

注意，通过这个事件通知文件描述符，我们只知道有无读写事件完成、有多少个读写事件完成，但是具体并不知道是哪些已经提交的读写事件完成了。这个时候需要由系统调用接口io\_getevents查询，以获得读写完成的事件列表。

归纳内核版本aio的使用流程如下：


	1、创建aio事件通知文件描述符，设置其读事件，挂入epoll中
	2、创建aio上下文（io_setup），并记录下来
	3、创建aio读写事件请求，通过已创建的aio上下文提交给内核（io_submit）
	4、内核设备驱动（driver）读写完成，内核向aio事件通知文件描述符写入完成的事件数
	5、aio事件通知文件描述符的epoll读事件触发，查询已完成的aio事件列表（io_getevents）
	6、对已完成的aio事件逐一处理（典型的处理比如通知应用层读写完成）

### 3.2.1 关于io\_cancel

该系统调用取消一个aio请求。

aio请求交给内核后进入IO队列（io queue），随后交给设备驱动（driver）执行。事实上，aio请求在交给设备驱动（driver）后，不能被取消，因此在实践中，应用层发起aio请求后，被成功取消的几率基本为零。因此，该系统调用的实际意义并不大，对于应用层aio请求取消的需求，其设计与实现不应依赖该系统调用。

### 3.2.2 关于偏移量和长度

内核版本aio请求中的偏移量和长度要求按扇区大小（512字节）对齐。为了描述的便利，不失一般性，假定应用层定义的页大小为1KB，即应用层发出的aio请求中的偏移量和长度按照1KB对齐，自然也满足设备驱动（driver）的对齐要求。

假定应用层需要访问的数据的偏移量是1000（单位字节），长度为2000（单位字节），也就是访问区间[1000, 3000)，按页边界划分成三个区间：[1000, 1024)，[1024, 2048)，[2048, 3000)，如下图所示。

也就是，第一页的[1000, 1024)，第二页的[0, 1024)，第三页的[0，952)，这里数字代表页内偏移。


![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/3-2-1.png)


如果该访问为读，那么应用层要发出三个aio请求，分别是读第一、二、三页：（page 1, read），（page2, read），（page 3， read）。aio读写完成后，应用层截取每页对应的区间即可。

如果该访问为写，那么应用层要发出五个aio请求，分别是读第一页，写第一页，写第二页，读第三页，写第三页：（page 1, read），（page 1, write）（page2, write），（page 3， read），（page 3，write）。

以第一页为例，应用层首先读取完整页内容，然后将待写数据覆盖掉相应的区间，再将更新过的页写入存储。先读后写严格序，不可错乱。第三页类似。 也就是，非对齐写时需要预读。

可见，非对齐写访问，会带来一到两次额外的aio请求，严格序还会带来访问延迟。

### 3.2.3 优缺点

优点：高效，零拷贝

缺点：技术复杂，门槛很高，应用层要做好适配，满足对齐和异步要求

应用层的适配可以由内存缓存（Mem Cache）完成，可视为用户空间版本的Page Cache。


# 4 AIO模块

AIO模块是指为了使用和适配原生AIO（即Linux内核版本aio）的功能集，包括但不限于


（1）映射应用层随机读写请求到原生aio请求

参见3.2.2 关于偏移量和长度

（2）解决对齐问题

参见3.2.2 关于偏移量和长度

（3）解决读写竞争问题

读写竞争问题涵盖读写顺序问题和读写并发问题：

第一种是对同一个应用层访问，由于边界不对齐，导致同时需要读和写两类aio请求，以及严格序要求带来的读写竞争问题，参见3.2.2。

第二种是对多个并发的应用层访问同一个页带来的读写竞争，有的要求读，有的要求写，如下图：


![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/4-1-1.png)


R代表对此页的读操作，W代表对此页的写操作。

首先，访问同一个页的操作要严格按照FIFO的原则排队，对上层承诺可靠的时序。

其次，严格遵循基本读写序关系：在前序R未完成前，后续R和W不可执行，保证数据一致性。

（4）解决读写合并问题

读写合并可以由AIO模块或者内存缓存（Mem Cache）负责，或者两者同时负责。

以上图为例，对R和W操作编号：


![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/4-1-2.png)


先声明：这里Page是应用层内存，对应到存储的相同大小的区间，并最终和该区间同步。意味着所有该page上的操作，先在内存操作该page，然后发起aio请求，同步到存储。

在R1完成前， R2 ~ W6必须等待。 在R1完成后，R2 ~ W6可以依序立即执行，不用发起任何aio请求，在内存完成读写合并，因为R1与存储中的数据完全一致。

再看下图：


![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/4-1-3.png)

在W1的aio请求发出前，R2 ~ W6可以依序立即执行，在内存完成读写合并，然后W1发起aio请求，因为W1与存储中的数据将最终一致。

## 4.1 性能数据

以256KB页面大小为例。AIO模块按256KB页为单位读写磁盘，应用层随机读写文件的偏移量和长度，被AIO模块处理、映射。设备为8 x 8T SATA，网络吞吐稳定在2.5Gbps ~ 1.7Gbps。

![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/4-1-4.png)


下表为工具测试某厂商SATA盘的结果，视为磁盘吞吐极限：

![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/4-1-5.png)


对比可以看到，AIO模块达到了磁盘吞吐极限。



# 5 存储设计

所有的存储，本质上由两部分组成：name node和data node。

![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/5-1-1.png)


Name node的作用就是从文件路径映射到文件内容的实际存储位置：

	Name node： path -------→ location

比如，读文件前，我们把文件的完整路径告诉给name node，name node检索后将结果返回，这样我们就知道文件存储在哪块磁盘、哪个物理文件（可选）、偏移量是多少、文件长度是多少等信息：

	（disk_no, file_no, offset, len）= NameNode(path)

那么data node的作用又是什么呢？data node就是用来管理磁盘空间的。 比如，写文件时，我们把文件长度告诉给data node，data node经过一番折腾后返回，告诉我们在哪块磁盘、哪个物理文件（可选）、起始偏移量是多少的地方有所需要的空间：

	（disk_no, file_no, offset） = DataNode(file_size)

不同的Name Node和Data Node的设计，形成不同的存储系统。

比如，如果Name Node和Data Node都在内存中，无需持久化到磁盘，那么就是内存缓存（Mem Cache）。

又比如，如果文件路径是某个特定的key，比如表示一段存储空间的区间 [存储介质序号，开始页，结束页)，那么这个存储就可以作为另一个存储的缓存，比如内存缓存（Mem Cache）、SSD缓存（SSD Cache）。

以三级缓存为例：

SATA盘持久化数据，SSD缓存（SSD Cache）作为二级缓存， 内存缓存（Mem Cache）作为一级缓存。

读文件时，先检查SATA盘的Name Node确认文件存在，再检查内存缓存（Mem Cache）是否命中；若否，再检查SSD缓存（SSD Cache）是否命中；若否，读SATA盘。

写文件时的数据流方案有多个，这里举一个例子：先写入内存缓存（Mem Cache），然后内存缓存（Mem Cache）的LRU机制和降级机制联动，降级到SSD缓存（SSD Cache），然后SSD缓存（SSD Cache）的LRU机制和降级机制联动，降级到SATA。

SATA盘以文件路径（path）为键值，文件的位置信息用[磁盘号，开始页，结束页)标识。内存缓存（Mem Cache）和SSD缓存（SSD Cache）以[磁盘号，开始页，结束页)为键值。通过键值的变化，联系三级缓存。

以上三级缓存只是一种设计方式，供读者参考。

# 6 内存缓存（Mem Cache）

内存缓存（Mem Cache）特指在用户空间开辟一块连续的内存空间，存储用户数据。应用层读写数据时，首先与内存缓存交互，避免用户态和内核态切换、与磁盘交互带来的消耗，提高性能。

广义讲，内核缓存（page cache）也是一种内存缓存，我们不考虑内存缓存（Mem Cache）和内核缓存（page cache）同时存在的情形，这有点重复和多余。

在direct io开启时，内存缓存（Mem Cache）的意义尤为明显，变相地充当了内核缓存（page cache）的功效，只是不用切换到内核态。

内存缓存（Mem Cache）根据业务的特点，可以简化为对特定大小页（比如，256KB，512KB等 ）的存储。特化带来的好处有两个：简单高效、满足AIO对齐要求。

内存缓存（Mem Cache）所使用的内存空间有限，在持续缓存至给定的内存空间耗尽时，需要根据LRU算法淘汰当前最冷的数据至下一级缓存，比如SSD缓存（SSD Cache）。

设计请参照第4节。


# 7 SSD缓存（SSD Cache）

SSD缓存（SSD Cache）指在SSD盘上开辟一块空间（或者整张盘），存储用户数据。数据的来源通常是内存缓存（Mem Cache）或者SATA盘。

SSD缓存（SSD Cache）主要是利用SSD的高IOPS能力、高读写吞吐（特别是读吞吐）能力，缓存热点数据，减少用户空间文件读写请求打到SATA盘上，从而提高存储性能。

相较于内存缓存（Mem Cache），SSD缓存（SSD Cache）具有如下不同点：

（1）更大的存储空间

不言而喻。

（2）数据持久化

内存缓存（Mem Cache）的数据是易失的， 而SSD缓存（SSD Cache）可以持久化数据。

SSD缓存（SSD Cache）根据业务的特点，可以简化为对特定大小页（比如，256KB，512KB等 ）的存储，有利于direct io。

SSD缓存（SSD Cache）所使用的磁盘空间有限，在持续缓存至给定的磁盘空间耗尽时，需要根据LRU算法淘汰当前最冷的数据。

设计请参照第4节。

SSD缓存（SSD Cache）可以考虑使用direct io + aio。需要指出的是，对于direct io，只需要满足对齐要求即可，对于aio，由于异步的存在，会导致分级存储的联动复杂度上升。

# 8 三级缓存

## 8.1 三级缓存模型

三级缓存模型即Mem Cache + SSD Cache + SATA构成的缓存系统，根据数据的冷热程度缓存到不同的介质上，如下图所示。

至下而上，IOPS能力和吞吐能力逐步升高、数据热度逐步升高、存储容量逐步降低。

![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/8-1-1.png)


## 8.2 AMD设计

对于设备配置，举例：8 x 12T SATA + 4 x 960G SSD，将SSD盘一分为二，每个分区480G，对应到一块SATA盘。数据采用严格降级方式缓存。

写数据时，首先写入内存缓存，然后降级到SSD盘，然后降级到SATA盘；

读数据时，首先检查内存缓存（Mem Cache），如果未命中，检查SSD缓存（SSD Cache），如果命中，加载至内存缓存（Mem Cache）；如果未命中，从SATA盘加载至内存缓存（Mem Cache）。


每个线程（或进程）创建一个AMD模块实例，对外提供交互接口，对内协调内存缓存（Mem Cache）、 SSD缓存（SSD Cache）和SATA访问，进行流程控制。


![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/8-2-1.png)



POSIX文件读写接口关注四个参数：文件描述符、文件偏移量、文件长度、文件内容（或buffer，不用特别区分）。

考虑对同一文件的操作，因此只考虑后面三个参变量，读写接口可以表达为：POSIX（偏移量， 长度， 内容）。

又考虑到（偏移量，长度）与（起始偏移量，结束偏移量）完全等价，因此读写接口可以表达为：POSIX（起始偏移量，结束偏移量，内容）。

AMD为了对上层应用提供透明接口，读写接口同样表达为：AMD（起始偏移量，结束偏移量，内容）。

AMD中的AIO、内存缓存（Mem Cache）和SSD缓存（SSD Cache）也同样表达：AIO（起始偏移量，结束偏移量，内容），Mem（起始偏移量，结束偏移量，内容）， SSD（起始偏移量，结束偏移量，内容）。

换言之，AMD内外与POSIX接口保持一致。这也意味着，AIO，内存缓存（Mem Cache）和SSD缓存（SSD Cache）与POSIX兼容，且具备独立部署能力。

![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/8-2-2.png)


## 8.3 流控与重试

在三级缓存体系中，SATA盘的IO能力最弱。数据从SSD降级存储到SATA盘（对应SATA盘写操作）时， 很容易打爆SATA，因此需要进行流控处理，控制从SSD盘到SATA盘的数据流速。

反过来，SSD到SATA盘的流控，又限制了内存缓存（Mem Cache）的数据流入速度，即不能超过流控标定的速度，换言之，SATA盘的写入能力，决定了三级缓存整体的写入能力。

考虑到内存缓存（Mem Cache）容量最小且易失，内存缓存（Mem Cache）向SSD缓存（SSD Cache）降级存储过程中发出的AIO请求，容易撑爆AIO队列，引发超时或内存耗尽等异常。

作为重试（确保内存数据能够持久化到磁盘）的手段之一，可以考虑向SSD盘或者SATA盘，直接发起DIO（direct io）请求，即通过阻塞的方式延长读写请求处理时间，确保数据能够持久化到磁盘，同时也是一种流控手段。这就是AIO降级到DIO策略。特别警告：该策略考验对严格时序的控制能力，极难控制，强烈建议不要采用，变相地可以用重试策略替代。

数据的存储降级要持续进行，不能等到缓存已满或将满时进行，否则来不及处理。可以采用化整为零的方式持续进行：结合LRU，周期性地将当前缓存中的一部分最冷数据，降级到下一级缓存。这是降级存储机制。

当前缓存中的数据，如果已经被降级存储，即变为可丢弃数据。可以在缓存已满时，或者周期性地淘汰可丢弃数据。这是淘汰机制。

降级存储机制和淘汰机制都建立在LRU之上，结合流控来看，降级和淘汰应协调一致，确保三级缓存运行流畅。

![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/8-3-1.png)


## 8.4 页面流转

内存缓存（Mem Cache）和SSD缓存（SSD Cache）各自维护一个页面（page）的LRU链表，页面从表头插入，表尾淘汰。

SATA也维护一个类似的LRU链表，但不受AMD控制。

内存缓存（Mem Cache）的页面从LRU表尾淘汰前，由降级机制刷到SSD缓存（SSD Cache）。

SSD缓存（SSD Cache）的页面从LRU表尾淘汰前，由降级机制刷到SATA。

访问的页面在SSD缓存（SSD Cache）或者SATA命中时，升级到内存缓存（Mem Cache）。

![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/amd/8-4-1.png)

