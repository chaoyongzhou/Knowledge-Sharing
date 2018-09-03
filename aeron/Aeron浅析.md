
目录

 1 引言

 2 基础知识：UDP通讯

 2.1 通讯五元组

 2.2 UDP单播

 2.3 UDP组播

 2.4 UDP广播

 2.5 Aeron中的单播

 2.6 Aeron中的组播

 2.7 Aeron中的广播

 2.8 Aeron中的组播地址约定

 3 基础知识：共享内存通讯，目录结构

 3.1 命名共享内存

 3.2 共享内存目录

 3.3 原子操作

 4 基础知识：多线程通讯

 4.1 多线程通讯

 4.2 aeron中的多线程

 5 Design Overview

 5.1 Media

 5.2 Media Driver

 5.3 Client

 6 Ring Buffer

 7 Log Buffer

 8 Frame

 8.1 Setup Frame

 8.2 Status Frame

 8.3 Data Frame

 8.4 PAD Frame

 8.5 Heartbeat Frame

 8.6 NAK Frame

 8.7 RTT Frame

 8.8 Error Frame

 9 Term

 10 Fragment

 11 Channel

 12 Stream

 13 Correlation Id

 14 IPC

 15 Congestion Control & Flow Control

 16 JAVA、C++、C版本异同

 17 Archive，重要的辅助功能


# 1 引言

Aeron提供可靠、高吞吐、延迟可期的UDP传输，项目开源地址：[Aeron](https://github.com/real-logic/aeron)

本文期望用浅显易懂的语言，讲解aeron的概念、原理和技巧。

我本人怀着欣赏的心情、夹杂着一点点批判的态度，审视aeron的设计哲学，剖析实现技巧，尽量不做源码分析，尽量不装逼，最好能点到即止。

至于aeron有多强悍、适用场景有多广泛、owner背景有多深，这种吹牛逼的事情留给aeron owner吧，本文行文目的不在此。

适当地，本文会先介绍一点背景知识，比如UDP通讯，共享内存通讯、多线程等，选择性地而非面面俱到介绍，请酌情阅读或跳过。

# 2 基础知识：UDP通讯

## 2.1 通讯五元组

凡用套接字（socket）通讯的，必涉及到通讯五元组：（源IP，源端口，目标IP，目标端口，协议族），也称为通讯三元组：（源EndPoint，目标EndPoint，协议族），此处EndPoint就是一对（IP，端口）。

通俗来讲，就是要描述通讯的关键要素：发送者、接收者、双方约定的通讯协议。

对于UDP通讯而言，协议族已确定。

对于目标端而言，IP和端口通常需要事先指定（除非是在类似p2p穿透场景下的猜端口，或者由第三方协助交换端口信息，本文不考虑此类场景），这类似于服务器的概念，即目标端是在特定的IP和端口处监听的UDP Server，因此需要先绑定（bind）一下。

对于源端而言，发送后不管，因此用什么端口无所谓，内核和协议栈关心罢了。通常会事先创建一个套接字，拥有一个随机的端口，后面的发送通过该套接字发送即可。

UDP通讯面向无连接，无须握手，没有流控（flow control），没有拥塞控制（congestion control），没有确认反馈（ack），没有回包，因此是不可靠通讯。要实现可靠UDP通讯，只能在应用层解决这些问题。

UDP显然是单向通讯，aeron需要反馈机制来实现可靠UDP通讯，因此在通讯双方之间采用一对UDP通讯：一个用来传输数据，一个用来传输控制指令和反馈等。

注意，在这个层面上描述通讯关键要素时，原始的通讯五元组中的源和目标的信息熵不够用了，需要升级一下概念：发送者（sender）、接收者（receiver）。

aeron的发送者，既是源端，向目标端发送数据；又是目标端，在此接收反馈。

aeron的接收者，既是目标端，在此接收数据；又是源端，向目标端发送反馈。

其实就是两个UDP通讯，各行其是。传输数据的，称为数据通道（data channel）；传输反馈的，称为控制通道（control channel）。

题外话，这种数据与控制分离的玩法的祖师爷当属高通，在无线通讯领域，高通最早弄出来信令面（signal plane）与用户面（ user plane）分离的概念，影响深远。

## 2.2 UDP单播

UDP单播（unicast）是指点到点通讯，发送端向特定的接收端发送。非组播和广播的（IP）地址，都可以用作单播（IP）地址。

UDP单播套用一下通讯五元组，不难理解。


## 2.3 UDP组播

UDP组播（multicast）是指一点发、多点收通讯，发送端把数据放到组播地址上，各接收端自己收。

UDP组播地址是特定的D类地址，即224.0.0.0至239.255.255.255之间，进一步划分如下：

（1）局域网组播地址：在224.0.0.0～224.0.0.255之间。路由器不对外转发。

（2）广域网组播地址：在224.0.1.0～238.255.255.255之间。

（3）管理权限组播地址：在239.0.0.0～239.255.255.255之间，局域网内使用，路由器不对外转发。

再强调一下通讯五元组，发送端往特定的端口发，接收端也必须在同样的端口上收。

## 2.4 UDP广播

UDP广播（broadcast）是指一点发，不知道谁收通讯，发送端把数据放到广播地址上，谁爱收不收。

UDP广播地址为255.255.255.255，仅限于局域网内，交换机不转发。

UDP广播是UDP组播的一个特例，这得从IP包携带的目标端MAC地址来看：目标端MAC地址的第48比特如果是1，则为组播地址；目标端MAC地址的比特全是1，则为广播地址；而单播，则就是目标端真实的MAC地址。

题外话：在局域网内，在交换机的同一个网段，只要想收，无论单播、组播、广播数据，你总能用杂凑模式（promiscuous mode）收到（想想抓包工具），不说那点龌龊之事了。

## 2.5 Aeron中的单播

与UDP单播无异，不过是一对UDP通讯罢了。

## 2.6 Aeron中的组播

有两种，一种与UDP组播无异，不过是发送端有一个监听端口，接收端需要往这个端口发送反馈罢了；另一种是模拟组播，aeron中称之为MDC（Multi-Destination-Cast），发送端和接收端之间都是UDP单播，只是发送端发送数据时，要向所有的接收端发送，另外，发送端有一个监听端口，接收端需要往这个端口发送反馈。

## 2.7 Aeron中的广播

aeron没有用UDP广播，和它没啥关系。aeron中的广播是面向客户端（client）的共享内存通讯而言的，是一个单生产者多消费者（spmc）模型。

## 2.8 Aeron中的组播地址约定

aeron为了减少一点配置量（太多了），约定：用来传数据的组播地址（data endpoint）必须是奇数，用来传控制的组播地址（control endpoint）为该地址加1。

例如，如果配置224.10.9.7为data endpoint，那么aeron自动将224.10.9.8作为control endpoint。


# 3 基础知识：共享内存通讯

## 3.1 命名共享内存

通常共享内存通讯采用的是命名共享内存，看起来像是POSIX文件系统中的一个文件。进程或者线程通过mmap的方式，将该文件映射到所在的进程空间，进行读写操作，完成数据交换。

## 3.2 共享内存目录

Linux的发行版带有一个/dev/shm的共享内存目录，只是不在硬盘上，而是在内存中，支持shell命令，如df， rm等。在这个目录下创建的子目录、文件，依然是在共享内存中，这为组织和管理共享内存提供了便利。

aeron正是利用了共享内存目录，达到高效通讯、不受实现语言的限制、不受通讯参与方数量限制的目的，按aeron协议约定交换数据即可。

aeron的共享内存，主要用于与外界，即客户端（client），的交互。

## 3.3 原子操作

共享内存属于临界资源，不可避免的需要原子操作，aeron采用内存屏障技术达到性能影响最小化目标。关于这部分代码，建议好好阅读、理解和剽窃。

aeron在共享内存上，建立单生产者多消费者（spmc）模型、多生产者单消费者（mpsc）模型，采用Log Buffer和Ring Buffer数据结构。

# 4 基础知识：多线程通讯

## 4.1 多线程通讯

在一个进程中开多个线程，共同拥有一个进程空间。

多线程通讯的优点是可以不通过message交换数据，直接内存访问即可，缺点是需要通过锁、原子操作解决临界资源竞争问题。

虽然message与锁等价，但message交换需要编解码（encoding & decoding），对于特定的业务场景，如果锁的开销和复杂度远远小于message交换，那么多线程模型就是适用的。

具体问题具体分析，不一而足。

## 4.2 aeron中的多线程

aeron选择多线程模型。aeron内部主要有三个的角色：conductor、sender、receiver。aeron抽象出runner的概念，每个角色交给一个runner执行，每个runner的执行承载体为线程。

由此，aeron提供了若干种模式：

* shared模式：三个runner在同一个线程上执行

* shared network模式：condctor runner一个线程，sender runner + receiver runner一个线程

* dedicated模式：三个runner分别在三个不同的线程上执行

交互发生在conductor和sender之间、conductor和receiver之间，通过单生产者单消费者（spsc）队列交换数据。

# 5 Design Overview

先从剖析aeron的架构图开始。 ![Architecture](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/architecture.png)


这张图画得挺炫，但有蛊惑之处，需要探究一番。

## 5.1 Media

图的最中间为Media，传输媒介。通俗地说，凡是支持套接字UDP接口的，比如sendmsg，都可以作为aeron的Media。老外忽悠起来一套一套的，逼格很高。

## 5.2 Media Driver

紧挨着Media的是Media Driver，这是一个抽象的概念，是aeron的核心，由三个角色组成：conductor、sender、receiver。

（1）conductor与client（客户端）通过共享内存交互（Ring Buffer），从client接收指令、将反馈广播给客户端。

指令集：暂略

（2）conductor与sender通过spsc队列，向sender发送指令

指令集：暂略

（3）conductor与receiver通过spsc队列，向receiver发送指令

指令集：暂略

（4）sender负责向Media发送UDP报文

那么问题来了：sender发送的数据来自哪里？答案是：共享内存publication。

也就是说，sender一直在检查publication，如果有数据，就以UDP报文的形式发送出去。

（5）receiver负责从Media接收UDP报文

同样的问题：receiver接收到的数据存哪里去了？答案是：共享内存image。

即，receiver每收到一个UDP报文，就会写入image，写满后rotate一下，继续写，甭管有没有人要，反正放这里了。

这里留下两个问题：谁在往publication里写入数据？谁在从image中读取数据？

## 5.3 Client

图的左右两侧为客户端（client），可以理解为它们并不是aeron组件，虽然aeron提供了JAVA版和C++版客户端，可以封装进Application中。

客户端主要的功能有三个：

（1）通过Log Buffer共享内存，向Media Driver写入数据（publish）

（2）通过Log Buffer共享内存，从Media Driver读出数据（subscribe）

（3）通过Ring Buffer共享内存，向Media Driver发送指令（command），以及接受反馈（response）

注意，publish操纵的Log Buffer共享内存称之为publication，subscribe操作的Log Buffer共享内存称之为image，指令与反馈操作的Ring Buffer共享内存是CNC（command and control）。
这是三个不同的共享内存，彼此功能独立，严格区分，各司其职。

这张图蛊惑之处主要在客户端（client），以下分析纯属个人意见，并不强求大家认同：

（1）客户端（client）的publisher、subscriber、conductor并不精确。

client的conductor功效并不等同于Media Driver中的conductor，不是一回事。client conducotr主要是把控制指令写入共享内存CNC中，并从CNC中读取反馈。
因此，client只要按aeron要求提供写入、读取的功能即可与Media Driver的conductor正常交互，其实现既可利用aeron提供的不同语言版本的接口，也可自己实现。

client中的publisher和subscriber存在类似的问题，分别是与共享内存publication和image进行读写交互。从pub-sub模型来看，通常，一方pub，多方sub，说的是应用层参与方的事。
而client在此处仅能表达与共享内存的交互，不够精确。

个人认为应该将client中的publisher和subscriber换成另外两个角色：writer和reader。publisher和subscriber则留在应用层。
应用层建立好pub-sub关系后，具体数据流为：

应用层publisher将数据交给client writer => client writer将数据写入共享内存publication => Media Driver的sender从publication取出数据，放到Media上，发往远端的subscriber。这是pub数据流。

远端Media Driver的receiver从Media接收数据，放进共享内存image => client reader从共享内存image总读取数据，交给应用层。 这是sub数据流。

如果把应用层和客户端看作是一回事，即都是client，那么这张图没有毛病，只是理解上会断片。

        
（2）客户端（client）同时存在publisher、subscriber不准确。

客户端同时画上publisher和subscriber能理解，协议嘛，收发两端都要有。在图的左右两侧都画上客户端，每个客户端都画上publisher和subscriber，这就不理解了。这是要告诉读者，客户端必须同时支持publisher和subscriber？实际应用场景显然不是，通常是一端需要publisher，另一端需要subscriber就够了。把图右侧的Media Driver和Client去掉，理解上一点毛病都没有，协议完整。

无论如何，读者理解了原理，这张图怎么看都行，它只是对初学者会造成一些不必要的理解上的麻烦。

从Media Driver角度看，sender和receiver与UDP通讯的关系：sender是UDP的发送端，receiver是接收端，receiver负责监听指定的udp端口。

从应用角度看，publisher和subscriber与UDP通讯的关系：publisher为应用层数据发送端，对应Media Driver的sender，subscriber为应用层数据接收端，对应Media Driver的receiver，因此可以简单理解为，publisher往特定的UDP通道上发送数据，subscriber在特定的UDP通道上接收数据。

理解就好了，注意所站的角度。

根据前面的一系列探究，我们可以延伸断定：

（1）Media Driver不支持relay（中继）能力，因为发送的数据来自publication，接收的数据写入image，二者严格分开。

（2）Media Driver不可能是一个自治的系统，因为传输数据来源于client，去向也是client。

好，再来看如果只有一个Media Driver，publisher和subscriber都与它交互，那么sender是如何把数据交给receiver的？

答案是对于UPD Media而言走UDP通道：sender把数据放到UDP通道，receiver从UDP通道上接收数据，不论单播还是组播，因为sender和receiver在Media Driver内没有交互。


# 6 Ring Buffer

Ring Buffer（环形缓冲，也称为Circular Buffer）结构，是一个FIFO的环形队列。

Media Driver的conductor和client之间通讯通过共享内存CNC的Ring Buffer完成。CNC的结构如下图所示：

![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/CNC.png)

可见通讯是双向的：CNC有两个Ring Buffer，一个是client到conductor方向，多生产者单消费者（mpsc）模型；一个是conductor到client方向，广播方式，单生产者多消费者（spmc）模型。

特别指出，aeron的作者在实现Ring Buffer时很务实，充满实战既视感，毫不学究。

从网上盗来一图如下。Ring Buffer有一个写指针和一个读指针，顺序读写。关键不同点在遇到这段内存的尾部和头部时的处理。


![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/Circular%20Buffer.jpg)

对标准的Ring Buffer而言，遇到头尾，数据逻辑上顺序连接，实际上一段数据在这段内存的尾部，一段数据在这段内存的头部，不连续，读写数据需要两次操作。
aeron的实现是，遇到尾部空间不足，则直接跳过，从头部开始，读写一次操作完成。

这儿有一版很好的Ring Buffer开源实现，供读者参考：[Ring Buffer](https://github.com/VladimirTyrin/RingBuffer.git)


# 7 Log Buffer

aeron的Log Buffer结构，用于conductor与client之间交换数据时的共享内存publication和image中。其结构如下图所示：

![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/publication%20log%20buffer%E7%BB%93%E6%9E%84.png)


Log Buffer主要由三部分组成：3个partition，1个Meta Data、1个Default Frame Header。

Log Buffer Partition存的是数据部，由若干定长的Term组成。

Log Buffer Meta Data用来指示每个Partition存的是哪个term的哪部分数据。

对于publication，Log Buffer是多生产者、单消费者模型（mpsc）；对于image而言，Log Buffer是单生产者、多生产者模型（spmc）。

aeron处理的是数据流，因此Log Buffer数据的读写是顺序的。数据流被切成固定大小的Term，放进Log Buffer的3个partion中。
每个Term有一个标识（term id），标识从初始值开始，顺序增长，aeron根据该标识，对3个partition采用Round Robbin算法，
比如Term A放进partion[0]，那么Term A+1放进partion[1]， Term A+2放进partion[2]，Term A+3放进partion[0]，依次类推。
这样做的好处是可以降低读写竞争概率，比如receiver将Term A放进partion[0]后，reader才可以从这里读，而此时receiver可以将Term A+1放进partion[1]，对不同partition的内存屏障大概率不会产生竞争。


# 8 Frame

Frame（帧）是sender和receiver之间交互的唯一方式，无论它们是否属于同一个Media Driver。

aeron定义了六种Frame： Setup Frame、Data Frame、NAK Frame，Status Frame，RTT Frame， Error Frame。

其中，Data Frame根据标志和帧长的不同，又包含PAD Frame和Heartbeat Frame。

一个Frame就是一条aeron定义的Message（消息），包含头部和数据部，有明确的边界。其定义可参考[Protocol Specification](https://github.com/real-logic/aeron/wiki/Protocol-Specification)


## 8.1 Setup Frame

Setup Frame的目的是为了在sender和receiver之间建立起一个双向通讯（一对UDP通讯），又称之为一个session（会话）。

如果sender和receiver之间采用UDP单播通讯，那么建立的基本过程是，sender发送Setup Frame，receiver回Status Frame，如下图所示：


![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/%E5%8D%95%E6%92%AD%E4%BA%A4%E4%BA%92%E6%B5%81%E7%A8%8B.png)


如果sender和receiver之间采用UDP组播通讯，那么建立的基本过程是：receiver从数据组播通道上“听”到Data Frame后，向控制组播通道上发送携带setup标志的Status Frame，
sender收到该帧后，在控制组播通道上发送Setup Frame，然后receiver在单播通道上回Status Frame。如下图所示：

![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/%E7%BB%84%E6%92%AD%E4%BA%A4%E4%BA%92%E6%B5%81%E7%A8%8B.png)


## 8.2 Status Frame

Status Frame由receiver发送给sender，告诉对方，我方的数据收到了多少（收到什么位置了），我方的接收窗口多大等信息，sender据此来做拥塞控制和流控。


## 8.3 Data Frame

Data Frame顾名思义，是用来传输应用层数据的，携带的信息包括，本次传输的数据是从什么位置开始的，长度是多少等。

通常Data Frame在数据通道上传输，但是Heartbeat Frame在控制通道上传输。


## 8.4 PAD Frame

PAD Frame是一个特殊的Data Frame：有长度，没数据（无payload）。

这是因为应用层要传输的数据的长度未必正好是Term的整数倍，最后一段数据在一个Data Frame中传递完后，不足一个Term的部分，用PAD Frame传递，指示receiver数据传完了。


## 8.5 Heartbeat Frame

Heartbeat Frame（心跳帧）也是一个特殊的Data Frame：没长度（为0），没数据（无payload）。

aeron会检查流（stream）的活性，如果一段时间没有数据传输，aeron就会停止流，删除对应的Log Buffer等。

因此，在应用层无数据传输，又要保持流（stream）不被删，就需要发送Heartbeat Frame。

技巧性地，在aeron的实现中，如果要关闭流，直接停止心跳，或者让aeron以为心跳停止即可。


## 8.6 NAK Frame

NAK Frame是实现UDP可靠传输的重要手段之一。receiver通过NAK Frame明确告诉sender，哪个数据帧需要重传，即需要重传的数据从什么位置开始，长度是多少等。


## 8.7 RTT Frame

RTT Frame用来测量sender和receiver之间的往返延迟。该帧在JAVA版本中支持，在C版本中尚未支持。


## 8.8 Error Frame

Error Frame用来指示错误信息的帧。


# 9 Term

aeron将应用层收发数据看做数据流，并将其按固定长度切成连续的Term，并唯一顺序编号。

aeron为了简化后续的各种换算与定位，约定此固定长度必须为2的N次方。

应用层数据中的一个字节在数据流中的位置，姑且称之为packet position。另一方面，该字节在编号为term id、偏移量为term offset的Term中。

假设Term的起始编号为initial term id；假设Term的固定长度为term buffer length，为2的position bits to shift次方。

那么，Term中的位置到数据流中的位置的换算关系如下：


    packet position = （term id - initial term id） * （term buffer length） + （term offset）
                    =  (（term id - initial term id） <<（position bits to shift） ) + （term offset） 
                    = （term id - initial term id） |  （term offset）

    注：低位部占（position bits to shift）个比特
    
反过来，数据流中的位置到Term中的位置的换算关系如下：

    term offset = （packet position）的低位部（position bits to shift）个比特 =  （packet position） & mask， 这里 mask = （1 << （position bits to shift）） - 1
    term id     = （packet position）的高位部 + （initial term id） = （（packet position） >> （position bits to shift）） + （initial term id）

可见，Term长度固定为2的N次方后，换算关系可采用位操作进行。

根据term id到Log Buffer的partition之间的映射关系，字节也可以对应到partition，不再累述。

# 10 Fragment

应用程序数据（APDU，Application Data Unit），被切成多个Fragment，每个Fragment作为payload放到一个Frame里传输。

Term与Fragment的关系：

 一个Term由多个Fragment组成，即发送端将Term拆成多个Fragment，接收端将多个Fragment拼装成Term。

一个Frame只能承载一个Term中的一段数据（Fragment）。

Term是固定长度，Fragment没有长度要求，可以根据拥塞控制、流控、窗口大小动态调整。所以，Fragment更多的只是个概念，形而上，来描述Frame的payload罢了。

# 11 Channel

Channel用来描述一个Media上的一个数据流（stream），aeron中用URI来标识Channel。通俗地讲，Channel核心是二元组（media，endpoint）。

典型URI举例：

    aeron:udp?endpoint=192.168.0.1:40456

这里，  aeron:udp是一种media，endpoint由ip地址和端口号构成。

更详尽的URI定义与例子，请参阅[Channel Configuration](https://github.com/real-logic/aeron/wiki/Channel-Configuration)


# 12 Stream

一个Channel上可以承载多个stream（数据流），用不同的stream id来标识。


# 13 Correlation Id

aeron有不少标识，这里特别提到Correlation Id，是因为该标识与Client（这里不区分Client和Application）高度相关。

通常，Client生成Correlation Id，在指令中携带，传递给Media Driver，后者在反馈中携带，回送给Client。换言之，这原本是Client用来区分和匹配（指令，反馈）对的。

但是， aeron目前的实现中，将Correlation Id挪作它用了。比如，在create publication时，Media Driver将Correction Id直接作为publication的Registration Id，并将该Registration Id回送给Client留作纪念，用来标识所创建的publication。

这下麻烦了！Media Driver可能会面对多个Client，如果两个Client用同一个Correlation Id创建publication，这不冲突了吗？

为了缓解这个问题，aeron把球抛给了client：各位client，请务必保证Correlation Id全局唯一！

aeron设计上的一个缺陷，导致在这儿挖了个坑，各位小心点，绕道走就是。


# 14 IPC

在Channel二元组（media, endpoint）中，如果media是aeron:udp，那么就是UDP Media；如果media是aeron:ipc，那么就是IPC Media。

如果共享内存用来在同一台设备上的进程或线程间通讯，那么就可以用IPC Media来提高性能，获得更高的吞吐量、更低的延迟。


# 15 Congestion Control & Flow Control

aeron的C版这块实现基本为零，非常简陋：不能调整窗口大小。JAVA版这块我还没研究。

我认为，可靠UDP传输最有价值的，大概就是在这里了，目测aeron owner不会很快放出来。各位想在aeron上玩点大的，就在这儿下功夫吧！


# 16 JAVA、C++、C版本异同

无论是什么语言的版本，aeron都是在搞同一件事： 操作共享内存。

aeron最早出的是JAVA版，也是目前为止功能最全的版本。C版是将aeron的理念重新实现一遍，aeron协议完全一致。

剩下的，就是各路神仙封装出来的client：JAVA版本client，C++版本client，.NET版本client。

这些client都是用来操作Media Driver面向client端的共享内存的：CNC，publication，image。

由于Media Driver中runner的概念，导致Media Driver事实上既可以独立部署，也可以嵌套在Client或Application中，
于是有好事之徒，在Media Driver之上又包装一层，加上自己的指令集、流程控制，形成新的Media Driver。下面的Archive就是一例。


# 17 Archive

Archive的应用场景是录播（Recording）、重放（Replay），也就是，Media Driver将收到的数据，不仅放入image，还要额外落盘到文件，重放时从落盘文件中取出数据，放到指定的Channel上。

Aeron社区目前提供了Archive Media Driver和Archive Client。

Archive既可以单机单Media Driver部署，也可以单机双Media Driver部署，也可以双机双Media Driver部署。本质上，你只要能满足archive定义的指令集合处理流程即可。
双Media Driver部署时，一个用Archive Media Driver，一个用普通的Media Driver就够了，不必非要用两个Archive Media Driver。

下图是不同部署方式的原理图，供参考，不再累述原理。

单机单Media Driver：

![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/archive%E5%8E%9F%E7%90%86-%E5%90%8C%E6%9C%BA%E9%83%A8%E7%BD%B2.png)


单机双Media Driver，或双机双Media Driver：

![](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/aeron/archive%E5%8F%8C%E6%9C%BA%E9%83%A8%E7%BD%B2.png)


