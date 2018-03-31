目录：

1 基础知识

1.1 并发与并行与分布

1.2 NUMA

1.3 加速比

1.4 学习曲线

1.5 哲学问题

2 seastar底层机制

2.1  POSIX线程机制（posix thread）

2.2 协程机制（coroutine）

2.3 任务机制（task）

2.4  reactor机制（engine）

2.5  /p/f/c/t

3 seastar源码阅读（一）

3.1 入口

3.2 执行

3.3 出口

3.4 并行

3.5 串行（序关系）

3.6 线程

3.7 信号量（semaphore）

3.8 管道（pipe）

3.9 应用组件隔离 （isolation of application components）

3.10 sharding内存分配

3.11 ready futures

3.12 future和promise

3.13 lambda

4 seastar源码阅读（二）

4.1 captured state

4.2 unique_ptr

4.3 生命周期管理

4.4 循环

4.5 shared-nothing

4.6 network stack

4.7 日志（log）

5 seastar源码阅读（三）

5.1 futhre和promise

5.2 future链式操作

5.3 ready future

5.4 执行但不运行

6 核间消息传递

6.1 入口与流程

6.2 消息发送

6.3 消息处理

7 四个poller

7.1 epoll poller

7.2 smp poller

7.3 io poller 和 aio poller

8 空闲休眠机制

# 1 基础知识

## 1.1 并发与并行与分布

并发、并行与分布这三个概念都与时间、空间和待处理的事情有关系，差别不大。
Erlang 之父 Joe Armstrong 用一张图解释了并发与并行的区别：

![图一](https://github.com/chaoyongzhou/Knowledge-Sharing/blob/master/seastar/concurrence_parallel.jpg)


分布主要是强调处理事情所在的地域不同，防止单点故障。

什么叫地域不同？两个咖啡机分别在北京办公室和上海办公室，这是地域上的不同；两个咖啡机在同一张桌子上的两端，难道就是地域上相同？

所以，很难对这三个概念下无可挑剔的定义。我们不必太较真，根据不同场景，来理解并行与分布即可。

## 1.2 NUMA

NUMA通过提供分离的存储器给各个处理器，避免当多个处理器访问同一个存储器产生性能损失。

NUMA把一台计算机分成多个节点(node)，每个节点内部拥有多个CPU，节点内部使用共有的内存控制器，节点之间是通过互联模块进行连接和信息交互。

因此节点的所有内存对于本节点所有的CPU都是等同的，对于其他节点中的所有CPU都不同。

因此每个CPU可以访问整个系统内存，但是访问本地节点的内存速度
最快(不经过互联模块),访问非本地节点的内存速度较慢(需要经过互联模块)，即CPU访问内存的速度与节点的距离有关。

## 1.3 加速比

加速比是同一个任务在单处理器系统和并行处理器系统中运行消耗的时间的比率，用来衡量并行系统或程序并行化的性能和效果。

线性加速比，简言之，是指核数翻倍，性能翻倍。
能达到线性加速比的应用，被认为是性能极高的应用。通常因为核数的增加，应用内部的交互损耗加大，导致达不到线性加速比。

但是，线性加速比并不是所有应用的极限，有人找到案例确实能做到超线性加速比，比如核数增多，因为缓存命中率高，数据的存取反而更快，从而达到超线性加速比。

线性加速比可以认为是实际应用中能达到的最好效果。

## 1.4 学习曲线

并行计算的学习曲线非常陡峭，主要是三个方面的问题：

### 1.4.1 并行思维

人类的大脑天然是单核的，同一个时刻只能想一件事情，并不擅长从并行角度来思考问题。
举个看代码的例子：你觉得它似乎从该点已经返回了，其实它还没开始执行：-）

通常我们看到的是代码的顺序，不是执行顺序。

并行思维需要我们反复练习，直至成为习惯。

### 1.4.2 生命周期

变量在并行环境下，总是要考虑它的全生命周期。

由于并行执行的时机、时长、顺序不可控，导致“预测”变量的生命周期变成不可能。所以，我们尽量要使用保守的方式控制变量生命周期（或者资源的生命周期），即资源的隔离性。

追踪变量的生命周期时，也要不断变换视角，从不同的时间、空间、时序上来看待它。

### 1.4.3 时序问题
一个完全并行的环境，实际是一个失控的环境，任何执行顺序都可能发生。

并行之中掺杂“串行”才是实际应用追求的目标。“串行”就是一种序关系，规定了“只有先执行谁，才能执行谁”。但是什么时候执行，甚至由谁来执行，则是并行的、不可控的。

### 1.5 哲学问题

在并行和分布的世界里，经常要关注一个问题：我站在哪里？要到哪里去？

# 2. seastar底层机制

## 2.1  POSIX线程机制（posix thread）
在同一个进程中，不同的posix thread拥有彼此独立的堆栈空间（即函数调用空间），接受操作系统的调度。

多个posix thread同时在多个物理核上运行，是真正意义上（物理意义上）的并行。

创建posix thread pool主要目的是为了绑定不同物理核、拥有独立的堆栈空间，资源隔离，通过并发达到很高的执行效率。

## 2.2 协程机制（coroutine）

coroutine没有标准定义，它代表的是一种宏观上的、在用户空间的并行调度机制（伪并行）。

coroutine主要是针对应用层逻辑而言，为了达到业务层伪并行目的而产生的。

coroutine和posix thread的不同点：

* coroutine在用户空间由应用层调度；posix thread由操作系统调度。

* 同一个posix thread内的多个coroutine运行在与posix thread相同的核上，不可能占用多个核，因此实际上是串行执行；多个posix thread可以被操作系统调度到多个核上并行执行。

* coroutine的切换由应用层显示调用接口实现，是语句级，可控程度高；posix thread的切换由操作系统分配的时间片决定，是CPU指令级，不可控。

从目前实现coroutine的技术来看，可以分为两类：带独立堆栈空间的coroutine技术方案和不带独立堆栈空间的coroutine技术方案。

	带独立堆栈空间的coroutine技术方案

比如POSIX提供的ucontext。该方案的优势在于，coroutine执行过程中，可以从任意一点切出（yield），切出前将当前堆栈位置保存下来，后面再切回（resume），恢复堆栈，继续向后执行。

    不带独立堆栈空间的coroutine技术方案

比如lua coroutine。该方案的优势在于，coroutine的切出切入非常轻，并不保存和恢复完整的堆栈，仅仅保存和恢复返回地址。这一点同样也成为一大劣势：切出只能发生在一个函数的return处，不能做到任一点切出切入。

早期的coroutine技术方案是coco lua，应用于游戏领域。

**如果每个业务处理都是短时的、无逻辑分叉的，那么coroutine没有必要，直接放posix thread上跑即可。**

如果业务逻辑复杂 ，那么选择coroutine可以降低逻辑表达的复杂度。

coroutine需要消耗一定的调度资源（比如CPU）和系统资源（比如内存），需要根据业务特点酌情选择。

seastar中的coroutine定义为seastar::thread，创建方法有：*seastar::thread(), seastar::async()*。

seastar并未事先创建coroutine pool，而是按需创建。

## 2.3 任务机制（task）

task通常是应用层看到的，能调度到核上执行的最小单元，要占用一定的cpu时间。通常具有代码执行地址、参数和状态机（可选）三个属性。

seastar的主循环面向task，seastar应用层的概念，比如future, continuation，event等，最终被封装成task，接受主循环调度。

task被加载到posix thread上执行。

seastar抽象出future，用来创建task，连接后继task（future/continuation），实现序关系。

seastar对task的流转，利用三个queue，建立了一个简单的状态机：

（1） 新创建的task，先扔进\_task\_queues中，这是vector类型。

（2） 如果\_task\_queues非空，则将整个\_task\_queues挂载到\_activating\_task\_queues中，这是list类型。

（3） 然后将\_activating\_task\_queues中的queue依据优先级，移至\_active\_task\_queues中，这是list类型。

（4） 主循环体每个loop，从\_active\_task\_queues中取出一部分queue来，依次执行queue中的task。如果一个queue在本次loop中不能执行完所有task，则再把该queue重新挂回到\_active\_task\_queues中。

## 2.4  reactor机制（engine）

建立（event, handler）映射，负责event截获（poll）与处理。

reactor建立一组poller，每个poller负责一类事件，比如，网络IO事件，信号，DMA aio事件，系统调用事件（如超时）等。

seastar将event的处理（handler）封装进了task，在task层统一，接收统一调度。

每个posix thread拥有一个reactor（即引擎engine）。posix thread的主体循环在reactor::run。 

## 2.5  /p/f/c/t

简单形象点理解： 

task = {func, params} => posix thread加载到核上执行

future = { task } => 生成任务、控制任务的生命周期

continuation = {next task}，严格的序关系。仅当前序执行完毕，返回future 
(ready future)，才会触发后序continuation的执行

promise = 在前一个future执行完毕返回值后，由promise将返回值交给(set_value)后面的continuation，其实就是promise负责将前一个的返回值作为后一个的输入参数。

> 问：future::get()是指什么？
> 
> 答：future-based blocking point。即get()操作在逻辑上阻塞，直至future执行完（成为ready future）。在ready前，future::then()不会被执行。通过这种方式，实现流程挂起，以及实现chaining continuations。

# 3 seastar源码阅读（一）

## 3.1 入口

seastar的入口归一到app\_template::run\_deprecated，主要完成初始化和创建第一个应用层future。

### 3.1.1 初始化

路径：app\_template::run\_deprecated => seastar::smp::configure

* 创建n-1个posix thread，0号posix thread无须创建，就是当前进程。

* 通过posix thread亲和性和CPU设置，使得每个posix thread绑定一个物理核。
                  注：posix thread绑定到哪些核上，可以通过配置指定。
* n个posix thread建立master-slave模型，其中0号posix thread为主，其余为从。

* 初始化时，通过bootstrap的wait接口建立同步机制，确保执行顺序。

* 初始化结束时，slave posix thread进入主循环，开始执行任务，master posix thread继续向后执行，直至加载用户定义的lambda

> 问：run_deprecated是在哪个posix thread上执行的？
> 
> 答：0号posix thread。

### 3.1.2 创建第一个应用层task，即lambda，封装成future

## 3.2 执行

0号posix thread完成初始化工作后，app\_template::run\_deprecated => 创建第一个应用层task => engine().run() => while(true)进入循环主体，开始执行任务。

run\_deprecated(lambda)，这里的lambda就是系统的初始输入任务。

## 3.3 出口

master posix thread主循环体（run_deprecated => engine().run() => while(true) ）当\_stopped被置位时退出。

即，master posix thread调用reactor::stop()接口时退出，或者slave posix thread通过调用reactor::exit()接口触发master posix thread执行退出动作。

如果应用的入口是app\_template::run（），则lambda执行完毕，应用立即强制结束，相当于单次执行。

如果应用的入口是app\_template::run_deprecated（），则维持主体循环，即无限循环中。

## 3.4 并行

> 问：任务如何从一个posix thread发往另一个posix thread？message passing如何进行？
> 
> 答：reactor中定义的smp:
> 
>    seastar::reactor::smp::submit\_to
>
>    seastar::reactor::smp::invoke\_on\_all
   
> 问：smp_message_queue::submit\_item是否posix thread安全？
> 
> 答：用boost::lockfree::spsc\_queue实现的，无锁，posix thread安全。

## 3.5 串行（序关系）

pipeline/workflow/fiber通过chaining continuations来表达严格的序关系，即仅当前一个continuation执行完毕，才可能让下一个continuation执行。

表达序关系的主要接口是future::then()。

对于最后一个future，为了确保异步操作最终被执行而且执行完毕，需要使用接口future::finally()。

*future::do\_for\_each*: 依次执行，即串行执行

## 3.6 线程

seastar中出现的线程有两类，澄清如下：

（1） posix thread

标准POSIX线程，创建于seastar初始化阶段。

（2） seastar::thread

协程（coroutine），拥有独立的堆栈空间。在posix thread中被创建，并在该posix thread中执行。

seastar的协程有两种实现方式：通过posix 协程接口，或通过long jump自己实现，由预编译宏ASAN_ENABLED控制选择。

为了区分，我们用posix thread和coroutine来表达以上两种不同的线程。

### 3.6.1 coroutine创建

接口：seastar::thread

### 3.6.2 coroutine结束

接口： seastar::thread::join()， 即等待coroutine执行完毕
        
注：coroutine创建和等待执行完毕，可以简化到一个接口：*seastar::async()*

### 3.6.3 coroutine控制

控制coroutine是指利用coroutine的挂起能力，设立阻塞点（blocking point），代码逻辑上看存在序关系，可读性更强。

设立阻塞点的方式有：

    seastar::future::get()

    seastar::future::wait()

    seastar::thread::yield()

举例一：

	seastar::future<> f() {
		return seastar::async([] {
		std::cout << "Hi.\n";
		for (int i = 1; i < 4; i++) {
		seastar::sleep(std::chrono::seconds(1)).get(); // 注意：get()是协程挂起点（阻塞点）。如果没有get（），就相当于抛出了一个任务后，马上跑到下一条语句执行了
		std::cout << i << "\n";
		}
		});
	}

举例二：

	seastar::future<seastar::sstring> read_file(sstring file_name) {
		return seastar::async([file_name] () { // lambda executed in a thread 【注：这儿是指coroutine，即协程，而不是posix thread】
		file f = seastar::open_file_dma(file_name).get0(); // get0() call "blocks"
		auto buf = f.dma_read(0, 512).get0(); // "block" again
		return seastar::sstring(buf.get(), buf.size());
		});
		};
		
相对而言，seastar::thread比future/continuation机制更重，需要独立的堆栈空间、需要消耗更多的调度资源。但是它更便捷，代码的可读性更强，逻辑更清晰，符合人类思维方式。

对于小业务、耗时极短、高频的任务适合future/continuation机制；对于耗时长、业务重、逻辑复杂的任务适合seastar::thread机制。

所谓耗时长短是相对于协程调度能力而言的。


## 3.7 信号量（semaphore）

这里的信号量是指seastar重新实现的信号量机制（seastar::semaphore），用来提供资源消耗限制机制。运用于如下三个方面：

### 3.7.1 并行度限制

限制同一核上最大可并发实例数。

举例:


	seastar::future<> g() {
	    static thread_local seastar::semaphore limit(100); // 注意，这儿声明为static thread_local的，对每次g()调用有效。 信号量上限为100个
	    return limit.wait(1).then([] { // wait表示需要1个信号量，如果信号量剩余1个或多个，则消费1个信号量，进入conintuation执行，否则等待
	        return slow(); // do the real work of g()
	    }).finally([] {
	        limit.signal(1); // 释放1个信号量
	    });
	}

一对操作接口： （*不建议使用，难点在于信号量归还*）

seastar::semaphore::wait()

seastar::semaphore::signal()

可以使用替代接口：*seastar::with_semaphore()*

举例：


	seastar::future<> g() {
	    static thread_local seastar::semaphore limit(100);
	    return seastar::with_semaphore(limit, 1, [] {
	        return slow(); // do the real work of g()
	    });
	}

注：仅当lambda执行完毕，with\_seamphore才会返回一个ready future，否则，逻辑上就是blocking的。

或者更通用（符合RAII）的接口：*seastar::get_units()*

举例：

	seastar::future<> g() {
	    static thread_local semaphore limit(100);
	    return seastar::get_units(limit, 1).then([] (auto units) { //消费1个单位的信号量，给units
	        return slow().finally([units = std::move(units)] {}); //当units对象销毁时，归还信号量
	    });
	}

### 3.7.2 资源消耗限制

	seastar::future<> using_lots_of_memory(size_t bytes) {
	    static thread_local seastar::semaphore limit(1000000); // limit to 1MB
	    return seastar::with_semaphore(limit, bytes, [bytes] {
	    // do something allocating 'bytes' bytes of memory
	   });
	}

注：资源消耗上限1MB，超出就抛异常。

### 3.7.3 循环的并行度限制


	thread_local seastar::semaphore limit(100); // 设定信号量上限100
	seastar::future<> f() {
		return seastar::do_with(seastar::gate(), [] (auto& gate) { // 在入口处、并行任务开始前，分配一个gate对象，初始计数器为0
			return seastar::do_for_each(boost::counting_iterator<int>(0),
			boost::counting_iterator<int>(456), [&gate] (int i) {
				return seastar::get_units(limit, 1).then([&gate] (auto units) {//消费1个单位的信号量，给units。消费成功，进入continuation
					gate.enter(); // 计数器 + 1，注意：一旦触发gate.close，gate.enter不能继续操纵计数器（抽象看就是不能再次进入），否则抛出异常future
					seastar::futurize_apply(slow).finally([&gate, units = std::move(units)] {
					gate.leave(); // 计数器 - 1
				});
			});
		}).finally([&gate] { //  这里一定要使用finally等待前面的continuation执行（resolve ）完，否则最后一波任务可能还未执行或者还未执行完毕，seastar::do_for_each就结束了。
		                             //  注：异步调用是抛出去就不管了，并不在乎是否执行完毕，需要finally来同步，即确保序关系
		                             // 只有此finally才能确保do_for_each中的每个操作都完成后才准备返回
			return gate.close();  // 返回future。计数器归零时，将future的返回值（tuple）交给promise，这样才可以进入continuation
		});
		});
	}

注： gate用来阻止某个操作的发生、等待进行中的操作完成。有两个操作接口：seastar::gate::enter(), seastar::gate::leave()
当gate结束时（gate.close()且计数器归零），调用g.enter()将抛出异常future，这样，g.enter()后面的操作不会被执行。

一个简洁的替代接口是：*seastar::with_gate()*，相当于enter和leave的组合。

## 3.8 管道（pipe）

这里的管道（pipe）是seastar重新实现的管道机制(seastar::pipe)，是两个fiber之间传输数据的机制，采用生产者-消费者模型。

管道限于单个读单个写，读写逻辑上（future-based）阻塞，即同一个官道上，不允许两个同时读，或者同时写。

*注：还没有找到例子*

## 3.9 应用组件隔离 （isolation of application components）

组件隔离主要是通过不同组件占用不同的CPU时长资源配额，从而获得不同的性能表现。（还有一种隔离方式是占用不同的disk I/O配额）


	seastar::future<> f() {
		return seastar::when_all_succeed(
		seastar::create_scheduling_group("loop1", 100), // CPU配额100
		seastar::create_scheduling_group("loop2", 100)).then( // CPU配额100. 两个配额相同，它们获得的CPU时间相同.如果这儿配额是200，那么它将获得2倍于上一个的CPU时间。
		[] (seastar::scheduling_group sg1, seastar::scheduling_group sg2) {
			return seastar::do_with(false, [sg1, sg2] (bool& stop) {
				seastar::sleep(std::chrono::seconds(10)).then([&stop] {
				stop = true;
				});
				return seastar::when_all_succeed(loop_in_sg(1, stop, sg1), 
				loop_in_sg(10, stop, sg2)).then(
					[] (long n1, long n2) {
						std::cout << "Counters: " << n1 << ", " << n2 << "\n";
					});
			});
		});
	}

> 问：CPU配额是一个相对的概念，那么当设定一个scheduling\_group时，它的配额又是和谁对比呢？
> 
> 答：（不清楚）

## 3.10 sharding内存分配

seastar重新实现了C++中的shared_ptr，即索引计数的共享内存（后续章节介绍）

make_shared： 索引计数的共享内存，支持多态

make\_lw_shared： 轻量级版索引计数的共享内存，不支持多态

注：4.3.3章节将继续介绍

## 3.11 ready futures

ready futures会在同一个主循环（loop）中被立即执行，只要本次loop的continuation数没有超过256个。

立即执行可以看做是一个优化手段，其副作用是可能导致loop被饿死：无法进入下一次loop。所以采用最大执行256个的限制策略。

make_ready_future<>()可以返回一个ready future。

## 3.12 future和promise

只有把future的返回值（tuple）交给promise （set\_value，参见gate.hh:96），才能进入continuation。如果没有set\_value，即使通过get_future()获得future，也是不能进入continuation的。

promise更像一个future的控制器，控制着future是否能够调用（invocate）future::then()。
（A promise<T> has a method future<T> get\_future() to returns a future, and a method set_value(T), to resolve this future. ）

注：promise特别像负责函数调用压栈的。

注：可以忘了promise这个概念，聚焦在future和continuations上。

## 3.13 lambda

lambda本质上是一个对象，拥有数据和代码。

注：如果在lambda中，标准C++接口抛出异常，lambda最终能返回一个future，因为lambda是被放进futurize_apply（）中执行的。 

# 4 seastar源码阅读（二）

这部分主要介绍生命周期管理。

## 4.1 captured state

lambda的captured state要么拥有索引，要么拥有拷贝。

在并行场景下，由于future并非总是立即ready的，亦即continuation并不总是立即执行，那么captured state就处于fly in air的中间态：前面已结束，后面未开始。

seastar利用了C++14的特性，即captured state支持std::move，可以将captured state拷贝到堆（heap）上，然后在continuation执行时，再拷贝过来，同时销毁堆上的captured state。这会带来一定的拷贝量和内存占用，相较而言还可以接受，但需要防止不良编码导致的内存占用过大问题，比如巨复杂的类型作为captured state。

std::move是变形版拷贝，让数据的ownership发生转移而已。

> 问：为什么seastar框架和模型必须要用C++ 14及以上编译器？

> 答：因为captured state中要用到std::move

## 4.2 unique_ptr

C++确保std::unique_ptr始终只有指针一个拷贝，比如

	int do_something(std::unique_ptr<T> obj) {...}  //[C++ 11] 在此scope内用完即毁

不过std::move可以让该指针下的数据的ownership发生转移（注意指针还是只有一个拷贝，这一点没变）

	seastar::future<int> slow_do_something(std::unique_ptr<T> obj) { //[C++ 11] 在此scope内用完即毁
		using namespace std::chrono_literals;
		return seastar::sleep(10ms).then([obj = std::move(obj)] () mutable {//[C++ 14] ownership转移到captured state
			return do_something(std::move(obj)); // [C++ 11] 在obj销毁前，ownership先转移一下。                                                              // 由于std::move会导致obj read-only，从而禁止std::move操作，所以前面还要加上mutable去掉read-only限制
		});
	}

注：对于复杂对象，moving obj (std::move)操作并不总是无害或者轻的。此时可以考虑使用unique_ptr，然后用std::move操作unique_ptr。

## 4.3 生命周期管理

### 4.3.1 向continuation传递ownership        

方式：std::move       

> 问：在chaining continuations中，其中一个continuation的captured state使用了std::move，那么ownership是什么时候发生转移的？   
    
> 答：应该是编译器在推导时安排好了何时发生转移。例如：

	seastar::future<> slow_op(std::vector<int> v) {    // v is not copied again, but instead moved:    
		return seastar::sleep(10ms).then([v = std::move(v)] { /* do something with v */ 
	});
}

执行到seastar::sleep()时，slow_op立即返回一个future（需要10ms后才会ready），在返回前captured state对象必定已经被创建，v的ownership已经转移到captured state对象中，此时then中的lambda还未执行。

### 4.3.2 调用者持有ownership

并行环境下，在向两个（或多个）不同的异步函数（或continuations）传递同一个对象时，std::move并不方便。接口s*eastar::do_with（*）提供了解决途径：do_with将给定的对象存放在堆（heap）上，调用lambda时使用对象的索引（reference to object）。

举例：

	seastar::future<> f() {    
		return seastar::do_with(T1(), T2(), [] (auto& obj1, auto& obj2) { // 注意必须要用&符，如果没有，C++会释放掉两个对象        
			return slow_op(obj1, obj2);    
		}
	}

错误举例：

	seastar::future<> slow_op(T obj); // WRONG: should be T&, not Tseastar::future<> f() {    
		return seastar::do_with(T(), [] (auto& obj) {        
			return slow_op(obj); // 注意：future并不是立即执行的，而是等待调度。                                         // 这里拷贝的是值，一个地址。在lambda被resolve后，obj被释放，地址失效，当slow_op被调度执行时，reference对应的内存已经被销毁了    
		}
	}

### 4.3.3 共享ownership

采用reference counting的共享内存方案，不可依然要小心使用。先举例。

错误使用：

	seastar::future<uint64_t> slow_size(file f) { //file采用共享内存reference counting方案    
		return seastar::sleep(10ms).then([f] {    
			return f.size(); // 此处有问题，因为f.size()是异步执行（not ready future）的，当其执行时，对象f已被销毁，这是由C++编译器决定的
		});
	}

正确使用：
	
	seastar::future<uint64_t> slow_size(file f) {
		return seastar::sleep(10ms).then([f] {
			return f.size().finally([f] {}); // finally延长了f的生命周期，所以在f.size()执行（resolve）时，f依然在生命周期内，未被销毁。finally其实啥也没做。
		});
	}

std::shared_ptr<T>是C++ 11提供的标准的创建reference-counted共享对象的方法，不过它是面向posix multiple threads的，偏重。seastar重新实现了一下：

seastar::shared\_ptr<T>，没有原子操作。

seastar::shared\_ptr<T>：支持多态

seastar::lw\_shared_ptr<T>：更轻量级，不支持多态

注：为了更高的性能，建议尽量选用seastar::lw\_shared_ptr<T>

### 4.3.4  对象存堆栈（stack）上

这是针对coroutine而言的，对象存在 coroutine的堆栈上。

举例：

	seastar::future<> slow_incr(int i) {
		return seastar::async([i] {    //直接开coroutine， i存在coroutine的堆栈里
		seastar::sleep(10ms).get(); //阻塞点（blocking point），从该点切出，10ms后再切回来，继续执行后面的语句
		// We get here after the 10ms of wait, i is still available.
			return i + 1;                      // 此时，i依然在coroutine的堆栈中
		});
	}

## 4.4 循环

循环方式有：

### 4.4.1 do_until（）
### 4.4.2  repeat（）
### 4.4.3  keep_doing（）
### 4.4.4  map_reduce（）
### 4.4.5  parallel\_for\_each（）         
开启一系列异步操作，然后等待它们执行完毕。

	seastar::future<> service_loop();
	seastar::future<> f() {
		return seastar::parallel_for_each(boost::irange<unsigned>(0, seastar::smp::count),[] (unsigned c) {
			return seastar::smp::submit_to(c, service_loop); // 提交到每个posix thread（即核）上执行
		});
	}

### 4.4.6 when_all（）          

等待一系列已存在的future执行完毕，每个future的（返回）类型可以不同。future个数在编译期间确定。

注意：when_all（）中的future仅接受右值（rvalue）返回。

	future<> f() {
		using namespace std::chrono_literals;
		future<int> slow_two = sleep(2s).then([] { return 2; });
		return when_all(
			sleep(1s),  // 返回future<>
			std::move(slow_two), // 返回future<int>
			make_ready_future<double>(3.5) // 返回future<double>
		).discard_result();
	}

when_all返回的是一个tuple: future<std::tuple<Futs...>>

when_all()需要根据返回tuple处理每个future的返回，包括异常。要等到每个future都resolve（返回），才会继续下一个continutation，即使某个future返回exception。

when\_all\_succeed()是when_all的简化版，各future的返回值依序传给conitinuation（无返回值的忽略，不占位）。如果有一个或多个future返回exception，则返回某个exception给continuation（注：怀疑是第一个或者最后一个返回的exception）。

when\_all\_succeed返回的是一个future.

## 4.5 shared-nothing

所谓shared-nothing，是指posix thread之间不共同拥有同一块内存（防止锁竞争），posix thread之间交换数据通过消息方式（smp::submit_to）。

所谓的每个posix thread拥有一块独立的内存空间，是指彼此拥有的内存空间（比如变量）的交集为空，每个posix thread拥有的内存空间未必连续。特此澄清，以防误解。

## 4.6 network stack

seastar的shard网络堆栈是指，每个posix thread处理一部分连接，该连接的整个生命周期都发生在接入的posix thread中。

举例：

	seastar::future<> service_loop();
	seastar::future<> f() {
		return seastar::parallel_for_each(boost::irange<unsigned>(0, seastar::smp::count),[] (unsigned c) {
			return seastar::smp::submit_to(c, service_loop);
		});
	}
	
	seastar::future<> service_loop() {
		return seastar::do_with(seastar::listen(seastar::make_ipv4_address({1234})),[] (auto& listener) {
			return seastar::keep_doing([&listener] () {
				return listener.accept().then([] (seastar::connected_socket s, seastar::socket_address a) {
					std::cout << "Accepted connection from " << a << "\n";
				});
			});
		});
	}
	
	seastar::future<> service_loop() {
		seastar::listen_options lo;
		lo.reuse_address = true;
		return seastar::do_with(seastar::listen(seastar::make_ipv4_address({1234}), lo),[] (auto& listener) {
			return seastar::keep_doing([&listener] () {
				return listener.accept().then([] (seastar::connected_socket s, seastar::socket_address a) {
					auto out = s.output();
					return seastar::do_with(std::move(s), std::move(out),[] (auto& s, auto& out) {
						return out.write(canned_response).then([&out] {
							return out.close();
						});
					});
				});
			});
		});
	}

每个posix thread上执行seastar::listen，携带同样的端口信息，如果没有设置端口复用，那么master posix thread开启监听，slave posix thread不开启监听。

源码分析如下：

（1） seastar在运行最初创建一个全局的network\_stack\_registrator对象：

	network_stack_registrator nsr_posix{"posix", 
		boost::program_options::options_description(),    
		[](boost::program_options::variables_map ops) {        
			return smp::main_thread() ? posix_network_stack::create(ops) : posix_ap_network_stack::create(ops);    
		},    
		true // 指示当前为缺省网络堆栈
	};

（注：seastar目前还不支持端口复用，见reactor::posix\_reuseport_detect()）

这里，工厂std::function<future<std::unique_ptr<network_stack>> (options opts)> factory定义为上面红色的lambda，其执行体表明：当前posix thread为主时，调用静态方法posix\_network\_stack::create， 为从时调用静态方法posix\_ap\_network\_stack::create，即走了不同的分支。

根据构造函数network\_stack\_registrator::network\_stack\_registrator的实现，

	network_stack_registrator::network_stack_registrator(sstring name,  
	      boost::program_options::options_description opts,        
	      std::function<future<std::unique_ptr<network_stack>>(options opts)> factory,
	      bool make_default) {
	      		network_stack_registry::register_stack(name, opts, factory, make_default);
			}

工厂factory传给了方法network\_stack\_registry::register_stack：

	void network_stack_registry::register_stack(sstring name,
	    boost::program_options::options_description opts, 
       std::function<future<std::unique_ptr<network_stack>> (options opts)> create, bool make_default) {
			_map()[name] = std::move(create);    
			options_description().add(opts);    
			if (make_default) {        
				_default() = name;    
			}
		}
	);

即name和factory存进了network\_stack\_registry的_map表中：{（name, factory）}

或者抽象点看，

main posix thread的表中存入（"posix", posix\_network\_stack::create）表项，

slave posix thread的表中存入（"posix", posix\_ap\_network\_stack::create）表项

（2）seastar创建每个posix thread后开始执行初始化配置，即调用reactor::configure：

	void reactor::configure(boost::program_options::variables_map vm) {    
		auto network_stack_ready = vm.count("network-stack")?
		network_stack_registry::create(sstring(vm["network-stack"].as<std::string>()), vm):
		network_stack_registry::create(vm);    
		network_stack_ready.then([this] (std::unique_ptr<network_stack> stack) {        
			_network_stack_ready_promise.set_value(std::move(stack));    
		});    
		......
	}

（2.1）首先判断是否配置指定了network-stack，如果没有指定，则使用缺省的，即（1）中的“posix”，调用network\_stack\_registry::create()创建ready future，这里的create就是个根据name（="posix"）去查（1）中的_map表，然后执行create。
也就是main posix thread调用posix\_network\_stack::create创建ready future，这个future绑定的网络堆栈是posix\_network\_stack:

	class posix_network_stack : public network_stack {
		private:    
		const bool _reuseport;public:    
		explicit posix_network_stack(boost::program_options::variables_map opts) : _reuseport(engine().posix_reuseport_available()) {}    
		virtual server_socket listen(socket_address sa, listen_options opts) override;    
		virtual ::seastar::socket socket() override;    
		virtual net::udp_channel make_udp_channel(ipv4_addr addr) override;    
		static future<std::unique_ptr<network_stack>> create(boost::program_options::variables_map opts) {        
			return make_ready_future<std::unique_ptr<network_stack>>(std::unique_ptr<network_stack>(new posix_network_stack(opts)));    
		}    
		virtual bool has_per_core_namespace() override { return _reuseport; };
	};
	
slave posix thread调用posix\_ap\_network\_stack::create创建ready future，这个future绑定的网络堆栈是posix\_api\_network\_stack（它又继承了 public posix\_network\_stack）

	class posix_ap_network_stack : public posix_network_stack {
		private:    const bool _reuseport;
		public:    posix_ap_network_stack(boost::program_options::variables_map opts) : posix_network_stack(std::move(opts)), _reuseport(engine().posix_reuseport_available()) {}    
		virtual server_socket listen(socket_address sa, listen_options opts) override;    
		static future<std::unique_ptr<network_stack>> create(boost::program_options::variables_map opts) {        
			return make_ready_future<std::unique_ptr<network_stack>>(std::unique_ptr<network_stack>(new posix_ap_network_stack(opts)));    
		}
	};

（2.2）然后，根据ready future的特性，network\_stack\_ready.then()被立即执行，将网络堆栈交给reactor::\_network\_stack\_ready\_promise.set_value()。根据promise的特性，set\_value（）后，promise的 future可以开始后序（continuation）的执行。

每个posix thread在执行完reactor::configure()之后（注：路径是app\_template::run\_deprecated => seastar::smp::configure => engine().configure(configuration)），进入reactor::run（），此处开始promoise的future的后序的执行：  

	_network_stack_ready_promise.get_future().then([this] (std::unique_ptr<network_stack> stack) {        
		_network_stack = std::move(stack);       
		.....    
	});
  
  也就是把网络堆栈交给了最终接收者：reactor::\_network\_stack

（3）我们回到seastar::listen（）的实现处：

	server_socketreactor::listen(socket_address sa, listen_options opt) {    
		return server_socket(_network_stack->listen(sa, opt));
	}

这里调用的是网络堆栈的listen，也就是说，

master posix thread调用的是posix\_network\_stack::listen，没有端口复用时，最终调用的是reactor::posix\_listen（）

slave posix thread调用的是posix\_ap\_network\_stack::listen，没有端口复用时，没有listen ！

## 4.7 日志（log）

日志记录应包括：精确到毫秒甚至微秒的时间戳、posix thread号或核号、级别（critical, error, warning,info,debug）、内容字符串

日志语句应可以根据预编译宏来控制控制开关

### 4.7.1 日志使用

seastar::logger seastar\_logger("seastar");

seastar_logger.error("Timer callback failed: {}", std::current\_exception());

seastar_logger.warn("double close() detected, contact support");

seastar_logger.error("{}: {}", message, eptr);

seastar::logger sched_logger("scheduler");

被封装到：

	template <typename... Args>
	void
	sched_print(const char* fmt, Args&&... args) {
	    if (sched_debug()) {
	        sched_logger.trace(fmt, std::forward<Args>(args)...);
	    }
	}

sched\_print("run\_some\_tasks: start");

sched\_print("running tq {} {}", (void*)tq, tq->_name);

### 4.7.2 日志输出：
log.cc:

std::atomic<bool> logger::_stdout = { true };

std::atomic<bool> logger::_syslog = { false };

布尔值用来控制，日志是否输出到标准设备，以及日志是否输出到系统日志。

当前seastar的缺省设定是输出到标准设备，不输出到系统日志。

### 4.7.3 日志级别：

std::atomic\<log_level> \_level = { log_level::info };

seastar::logger缺省日志级别为INFO，即仅输出ERROR, WARN, INFO日志， 不输出DEBUG和TRACE日志。

缺省日志级别可以通过接口logger::set\_level()重新设定。

### 4.7.4 注意事项

日志采用锁操作来保证日志信息的输出的完整性（原子输出），因此日志的开启验证影响性能，不适合生产环境。

# 5 seastar源码阅读（三）

## 5.1 future和promise

理解future和promise的关系，需要从两个接口入手： promise的接口get\_future和future的构造函数

	template <typename... T>
	inline
	future<T...>
	promise<T...>::get_future() noexcept {
	    assert(!_future && _state && !_task);
	    return future<T...>(this);
	}
	
	future::future(promise<T...>* pr) noexcept : _promise(pr) {
	    _promise->_future = this;
	}

promise::get\_future()接口创建并返回一个future对象，入参为当前promise对象。

future::future()接口将入参promise对象pr与成员对象\_promise绑定，同时，\_promise的成员对象\_future与当前的future对象绑定，即完成promise对象与future对象的相互绑定。

## 5.2 future链式操作

再来看future类的链式操作接口then()。该接口的使用方式通常形如：future1.then(lambda1).then(lambda2).then(lambda3)

根据future::then()的实现，

（1）如果该future对象是available的（即ready future），且无抢占发生，那么立即执行then（）接口参数中的lambda

（2）否则创建一个promise对象，调用promise::get\_future()，获得一个新的future对象（注：参见前面future和promise的关系分析），然后执行future::schedule()，最后返回新的future对象。

因此，形如future1.then(lambda).then(lambda).then(lambda)的链式操作等价于：

future2 = future1.then(lambda1)

future3 = future2.then(lambda2)

future4 = future3.then(lambda3)

再来看future::schedule()接口：

	void future::schedule(Func&& func) {
	    if (state()->available()) {
	        ::seastar::schedule(std::make_unique<continuation<Func, T...>>(std::move(func), std::move(*state())));
	    } else {
	        assert(_promise);
	        _promise->schedule(std::move(func));
	        _promise->_future = nullptr;
	        _promise = nullptr;
	    }
	}

首先判断该future对象是否为available的（即ready future），如果是，调用apply.hh中的apply()接口，立即执行func；
否则调用promise::schedule()接口，解除promise对象与future对象之间的相互绑定。

继续追踪promise::schedule()接口：

    template <typename Func>
    void promise::schedule(Func&& func) {
        auto tws = std::make_unique<continuation<Func, T...>>(std::move(func));
        _state = &tws->_state;
        _task = std::move(tws);
    }
    
该接口首先创建了一个continuation对象，入参为func。continuation继承continuation_base，continuation_base继承task。

注意：此时continuation_base::\_state并没有被初始化（赋值）。

随后，该接口将continuation_base::\_state短路（shortcut）连接到了promise::\_state，将continuation作为task移交给了promise::\_task。

一个task产生了！

> 问：continuation_bas::\_state是什么时候被赋值的呢？

> 答：这需要显示调用promise::set\_value() => future\_state::set()来完成赋值。比如在seastar的表达循环的接口中都显示调用了promise::set\_value()接口。

## 5.3 ready future

> 问：难道future必须要和promise相互绑定吗？
> 
> 答：不一定！future和promise相互绑定，是因为需要构建一个新的task对象，进而被主循环调度。

如果不需要构建新的对象，而是立即执行lambda，或什么都不执行而仅仅产生一个新的future以便链式操作，那么promise可以直接跳过。

为此，future使用了一个短路方式：futhre::\_local\_state，它是future_state类型，和continuation_base::\_state类型一致，作用一模一样：

	template <typename... T, typename... A>
	inline
	future<T...> future::make_ready_future(A&&... value) {
	    return future<T...>(ready_future_marker(), std::forward<A>(value)...);
	}
	
	template <typename... A>
	future(ready_future_marker, A&&... a) : _promise(nullptr) {
	    _local_state.set(std::forward<A>(a)...);
	}

也是调用future\_state::set()来给future::\_local\_state赋值。

这种不创建task、不和promise互相绑定的future称之为ready future。因为没有task，所以不能接受主循环调度，只能立即执行lambda或者空操作。

## 5.4 执行但不运行

我捏造这个词（执行但不运行）是想说明，在并行编程中经常会遇到的一个现象：从代码上看，一段代码或函数似乎执行了，但事实上它还没有执行。我们看到的只是代码的顺序，不是执行顺序。

还以前面提到的链式操作为例：future1.then(lambda1).then(lambda2).then(lambda3)

看起来好像是：我们先执行lambda1，然后执行lambda2，然后执行lambda3，而真实的运行环境却通常是这样的：

程序运行到此，先执行future1.then，将lambda1封装成一个task抛出，返回future2；

然后执行future2.then，将lambda2封装成一个task抛出，返回future3；

然后执行future3.then，将lambda3封装成一个task抛出，返回future4；

这行代码执行完，返回的是future4，而此时此刻lambda1，lambda2，lambda3一个都没执行！

为什么会这样呢？真实的原因是，所有的并行语言、框架都在尽力追求一种符合人类思维习惯的表达方式，尽可能用串行的语句来表达并行的逻辑（虽然上面算不上并行，仅仅是异步和序关系）。

上面链式操作真正想表达的是异步环境下的一种序关系：先执行lambda1，然后过一段时间，执行lambda2，再过一段时间，最后执行lambda3。

一般的并行表达，比上面的序关系表达还要复杂。我们需要严格分清什么是计算机的执行顺序，什么时候逻辑表达上的执行顺序，切不可混淆。

# 6 核间消息传递

## 6.1 入口与流程

应用层入口：smp::submit_to()

smp::submit\_to（）向指定核传递消息（提交一个回调，是一个临时对象），实际是通过smp\_message\_queue::submit()放到相应的队列中（核之间交互的收发队列，二维表），此刻，消息（or回调）已经被转换成一个future（参见reactor.hh:372 smp\_message\_queue::submit()的实现）

消息最终通过轮询完成传递：smp::poll\_queues()。它通过

async\_work\_item::process()构造future（即构造一个task，并提交，等待执行完毕） 

async\_work\_item::complete()（注：async\_work\_item继承自纯虚work\_item），调用promise::set\_value()完成前序future执行结果（result）与新的future的绑定，即ready future。
这个轮询是在reactor::run()主循环中被调用，每次循环至少调用一次（check\_for\_work => reactor::poll\_once, pure\_check\_for\_work => reactor::pure\_poll\_once）

消息的处理（即回调的执行）发生在reactor::run\_some\_tasks（）中。

## 6.2 消息发送

消息发送即消息提交，入口smp::submit\_to()，定义为：
	
	template <typename Func>
	static futurize_t<std::result_of_t<Func()>> smp::submit_to(unsigned t, Func&& func)

下面仅考虑不同核间消息发送。

smp拥有n x n的收发队列二维数组，n为核数，组成员类型为smp\_message\_queue：

	static smp_message_queue** _qs;

smp首先将消息提交给消息队列 smp::\_qs[ 目的核 ][ 当前核 ]，即发送地址为当前核，接收地址为目的核，提交方式为调用smp\_message\_queue::submit()接口：
	
	futurize_t<std::result_of_t<Func()>> smp_message_queue：submit(Func&& func) {
	    auto wi = std::make_unique<async_work_item<Func>>(*this, std::forward<Func>(func));
	    auto fut = wi->get_future();
	    submit_item(std::move(wi));
	    return fut;
	}
    
这个接口先将func封装进一个async\_work\_item的智能指针，然后调用smp\_message\_queue::submit\_item()将智能指针提交给临时（pending）队列
smp\_message\_queue::\_tx.a.pending\_fifo：

    std::deque<work_item*> pending_fifo
    
这是一个普通的双向队列。如果临时队列长度达到上限（水桶满），则批量移至
smp_message_queue::_pending：

	void smp_message_queue::submit_item(std::unique_ptr<smp_message_queue::work_item> item) {
	    _tx.a.pending_fifo.push_back(item.get());
	    item.release();
	    if (_tx.a.pending_fifo.size() >= batch_size) {
	        move_pending();
	    }
	}

	void smp_message_queue::move_pending() {
	    auto begin = _tx.a.pending_fifo.cbegin();
	    auto end = _tx.a.pending_fifo.cend();
	    end = _pending.push(begin, end);
	    if (begin == end) {
	        return;
	    }
	    auto nr = end - begin;
	    _pending.maybe_wakeup();
	    _tx.a.pending_fifo.erase(begin, end);
	    _current_queue_length += nr;
	    _last_snt_batch = nr;
	    _sent += nr;
	}

注意，这里水桶高度（smp\_message\_queue::batch\_size）设定为16。

特别地，smp\_message\_queue:\_pending是一个boost::lockfree::spsc\_queue队列，即lock-free，单生产者单消费者的队列。

smp\_message\_queue::move\_pending()是该队列的生产者。

## 6.3 消息处理

在消息的接收端，即接收核，需要遍历所有从其它核发送来的消息队列smp\_message\_queue，即smp::\_qs[ 当前核 ][ 发送核 ]。

如果有消息到达，则开始处理，入口为smp\_message\_queue::process\_incoming()

	size_t smp_message_queue::process_incoming() {
	    auto nr = process_queue<prefetch_cnt>(_pending, [this] (work_item* wi) {
	        wi->process();
	    });
	    _received += nr;
	    _last_rcv_batch = nr;
	    return nr;
	}

直接调用smp\_message\_queue::process\_queue()对boost::lockfree::spsc\_queue队列\_pending进行处理。

调用一次smp\_message\_queue::process\_queue()，最多处理128个核间消息（这里，消息从队列拷贝到本地内存，采用了预取技术），超过的部分留到下次处理。

	template<size_t PrefetchCnt, typename Func>
	size_t smp_message_queue::process_queue(lf_queue& q, Func process) {
	    // copy batch to local memory in order to minimize
	    // time in which cross-cpu data is accessed
	    work_item* items[queue_length + PrefetchCnt];
	    work_item* wi;
	    if (!q.pop(wi))
	        return 0;
	    // start prefecthing first item before popping the rest to overlap memory
	    // access with potential cache miss the second pop may cause
	    prefetch<2>(wi);
	    auto nr = q.pop(items);
	    std::fill(std::begin(items) + nr, std::begin(items) + nr + PrefetchCnt, nr ? items[nr - 1] : wi);
	    unsigned i = 0;
	    do {
	        prefetch_n<2>(std::begin(items) + i, std::begin(items) + i + PrefetchCnt);
	        process(wi);
	        wi = items[i++];
	    } while(i <= nr);
	
	    return nr + 1;
	}

# 7 四个poller

seastar维护四个poller：epoll poller（可选）、smp poller、io poller、aio poller

## 7.1 epoll poller

reactor::\_epoll\_poller在reactor::start\_epoll（）中初始化。


\_epoll\_poller的创建很奇怪，是被动地、当需要的时候才去创建。也就是说，如果seastar不负责网络IO，\_epoll\_poller不会被创建。

以发起网络连接为例，来看它的创建过程。

reactor::connect()在发起socket connect后，调用

pollable\_fd::writable() => 
reactor::writable() => 
reactor\_backend\_epoll::writeable() => 
reactor\_backend\_epoll::get\_epoll\_future()  =>
reactor::start\_epoll()  在这里\_epoll\_poller被创建！

	inline
	future<> pollable_fd::writeable() {
	    return engine().writeable(*_s);
	}
	
	future<> reactor::writeable(pollable_fd_state& fd) {
	    return _backend.writeable(fd);
	}
	
	future<> reactor_backend_epoll::writeable(pollable_fd_state& fd) {
	    return get_epoll_future(fd, &pollable_fd_state::pollout, EPOLLOUT);
	}
	
	void
	reactor::start_epoll() {
	    if (!_epoll_poller) {
	        _epoll_poller = poller(std::make_unique<epoll_pollfn>(*this));
	    }
	}

## 7.2 smp poller

主要负责核间消息传递

## 7.3 io poller 和 aio poller

这两个poller都是负责磁盘异步IO的，其中

io poller由事件触发，如果返回结果表明要重试，则压入重试队列。
aio poller主要负责批量处理磁盘异步IO请求，在主循环体中每次循环都会被调用到（reactor::poll\_once() => reactor::aio\_batch\_submit\_pollfn:poll() => reactor::flush\_pending\_aio()）
被调用时，依次处理reactor::\_pending\_aio队列中的批量磁盘异步IO请求，之后起coroutine，处理io poller残留的、需要重试的磁盘异步IO请求队列reactor::\_pending_aio_retry。

# 8 空闲休眠机制

所谓空闲休眠（idle-sleep）机制，是指系统在空闲情况下，应该让出CPU控制权，降低对CPU的占用率。如无此机制，系统CPU负载可能一直维持在100%，空耗CPU和电能。

必须指出，seastar当前版本对idle-sleep的实现并不完美。

seastar的idle-sleep机制最终是建立在epoll_pwait上的，该接口定义为：

	int epoll_pwait(int epfd, struct epoll_event *events, int maxevents, int timeout, const sigset_t *sigmask)

参数timeout表示，如果一直无事件发生，操作系统必须在多长时间内返回。单位是毫秒。

epoll\_wait/epoll\_pwait事实上留给了应用一个实现idle-sleep的技巧：如果无事件发生，应用可以在timeout时长内，将CPU还给操作系统，即应用处于真正的、不被执行的空闲状态。

seastar的idle-sleep机制在主循环中，每个loop都需要检查的：
	
	while (true) {
	        .... 
	        if (check_for_work()) {
	            if (idle) {
	                _total_idle += idle_end - idle_start;
	                account_idle(idle_end - idle_start);
	                idle_start = idle_end;
	                idle = false;                         // 如果有任务待执行，而idle标志曾被置位，则立即无条件复位。也就是退出idle态（这是应用层的idle态，而不是进程的）
	            }
	        } else {
	            idle_end = sched_clock::now(); // 如果没有任务，则更新一下idle计时器idle_end
	            if (!idle) {
	                idle_start = idle_end;
	                idle = true;  // 从非idle态，进入idle态，同时初始化idle计时器idle_start，即开始计时
	            }
	            bool go_to_sleep = true; // 是否允许进入休眠的开关，初始为开
	            try {
	                // we can't run check_for_work(), because that can run tasks in the context
	                // of the idle handler which change its state, without the idle handler expecting
	                // it.  So run pure_check_for_work() instead.
	                auto handler_result = _idle_cpu_handler(pure_check_for_work);
	                go_to_sleep = handler_result == idle_cpu_handler_result::no_more_work; // 再次更新一下休眠开关，根据当前实现，结果总是开
	            } catch (...) {
	                report_exception("Exception while running idle cpu handler", std::current_exception());
	            }
	            if (go_to_sleep) {
	                internal::cpu_relax();
	                if (idle_end - idle_start > _max_poll_time) {     // 如果idle计时器达到门限，也就是进入idle一段时间后，执行sleep操作
	                    // Turn off the task quota timer to avoid spurious wakeups
	                    struct itimerspec zero_itimerspec = {};
	                    _task_quota_timer.timerfd_settime(0, zero_itimerspec);
	                    auto start_sleep = sched_clock::now();
	                    sleep();           // 执行sleep
	                    // We may have slept for a while, so freshen idle_end
	                    idle_end = sched_clock::now();
	                    add_nonatomically(_stall_detector_missed_ticks, uint64_t((start_sleep - idle_end)/_task_quota));
	                    _task_quota_timer.timerfd_settime(0, task_quote_itimerspec);
	                }
	            } else {
	                // We previously ran pure_check_for_work(), might not actually have performed
	                // any work.
	                check_for_work();
	            }
	            t_run_completed = idle_end;
	        }
	    }
	
	void
	reactor::sleep() {
	    for (auto i = _pollers.begin(); i != _pollers.end(); ++i) {
	        auto ok = (*i)->try_enter_interrupt_mode();  // reactor::io_pollfn::try_enter_interrupt_mode这个poller有问题
	        if (!ok) { // 如果返回false，则进程不可能休眠，因为下面直接返回了！
	            while (i != _pollers.begin()) {
	                (*--i)->exit_interrupt_mode();
	            }
	            return;
	        }
	    }
	    wait_and_process(-1, &_active_sigmask);           // 通过epoll_pwait，将CPU交还给操作系统，即应用开始休眠，直至事件到达，或者信号触发，或者超时（这里不可能超时！）
	    for (auto i = _pollers.rbegin(); i != _pollers.rend(); ++i) {
	        (*i)->exit_interrupt_mode();
	    }
	}
	
为了满足seastar对idle-sleep机制的需求[seastar issue 65#](https://github.com/scylladb/seastar/issues/65)，seastar要求每个poller提供两个接口：try\_enter\_interrupt\_mode和exit\_interrupt_mode。
前者用来判断是否可以进入sleep（seastar名称上模仿了中断模式），后者用来执行唤醒后（即退出sleep后）的动作。

如果所有poller同意进入sleep，那通过epoll\_pwait开始休眠；否则，继续下次循环。

来看磁盘异步IO的接口：

    virtual bool reactor::io_pollfn::try_enter_interrupt_mode() override {
        // aio cannot generate events if there are no inflight aios;
        // but if we enabled _aio_eventfd, we can always enter
        return _r.my_io_queue->requests_currently_executing() > 0 || _r._aio_eventfd;  
    }
    virtual void reactor::io_pollfn::exit_interrupt_mode() override {
        // nothing to do
    }
    
注意，对此poller而言，如果我们没有开启磁盘异步IO，那么try\_enter\_interrupt\_mode总是返回false，也就是说seastar永远不会进入休眠！

这是缺陷之一。

缺陷之二在于，调用wait\_and\_process()时给的参数。第一个参数timeout = -1，最终作为epoll\_pwait的timeout参数。这将导致seastar被永远阻塞在此，直至事件发生，或者信号发生。
如果seastar的应用非网络事件驱动，没有网络IO发生，那将是灾难性的！