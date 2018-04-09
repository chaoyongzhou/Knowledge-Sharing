# 1 神奇的return

seastar对于future的return处理十分独特，我没弄明白这是不是C++自身的特性，但实验结果很清楚。

首先回顾一下seastar源码分析5.2节 “future链式操作”。我们说， future1.then(lambda1).then(lambda2).then(lambda3)，等价于，

	future2 = future1.then(lambda1)
	future3 = future2.then(lambda2)
	future4 = future3.then(lambda3)

以及5.4节“执行但不运行”，上述链式操作实际运行情况为：

先执行future1.then，将lambda1封装成一个task抛出，返回future2；

然后执行future2.then，将lambda2封装成一个task抛出，返回future3；

然后执行future3.then，将lambda3封装成一个task抛出，返回future4；

这行代码执行完，返回的是future4，而此时此刻lambda1，lambda2，lambda3一个都没执行！

好，我们来看示例：

	#include "core/app-template.hh"
	#include "core/reactor.hh"
	#include "core/sleep.hh"
	#include "core/thread.hh"
	#include "core/gate.hh"
	#include "core/temporary_buffer.hh"
	#include "core/iostream-impl.hh"
	
	#include "util/log.hh"
	
	#include <iostream>
	#include <boost/iterator/counting_iterator.hpp>
	
	using namespace seastar;
	
	using namespace std::chrono_literals;
	
	seastar::future<> entrance();
	
	seastar::logger demo_logger("demo");  // 定义日志
	
	int main(int argc, char** argv) {
	    seastar::app_template app;
	
	    try {
	        app.run(argc, argv, entrance); // run()接口表示entrance执行完毕，app结束
	        demo_logger.debug("main: done"); // app结束后，打印日志
	    } catch(...) {
	        std::cerr << "Couldn't start application: "
	                  << std::current_exception() << "\n";
	        return 1;
	    }
	    return 0;
	}
	
	seastar::future<> entrance() {
	    demo_logger.set_level(log_level::debug); // 设置日志级别为debug。注意：该设置不能发生在app.run()之前，因为configure还没执行。事实上，只能在这儿设置。
	
	    demo_logger.debug("entrance: enter"); // sleep前打印一条日志，表示已进入entrance
	    return seastar::sleep(5s).then([] { demo_logger.debug("entrance: done"); }); // sleep后打印一条entrance结束日志
	}
	
实际运行结果为
	
	DEBUG 2018-04-08 13:49:57,912 [shard 0] demo - entrance: enter
	DEBUG 2018-04-08 13:50:02,913 [shard 0] demo - entrance: done
	DEBUG 2018-04-08 13:50:02,957 [shard 0] demo - main: done

即，进入entrance，sleep 5秒后，打印entrance结束，回到main打印app结束。 

现在，修改entrance()接口如下：

	seastar::future<> entrance() {
	    demo_logger.set_level(log_level::debug);
	
	    demo_logger.debug("entrance: enter");
	    seastar::sleep(5s).then([] { demo_logger.debug("entrance: done"); }); // 注意，这条语句前的return删除了
	
	    demo_logger.debug("entrance: leave"); // 打印日志，表示准备离开entrance
	    return seastar::make_ready_future<>();   //返回一个ready future 
	}

改动之处在于，将return挪走了。运行结果为
	
	DEBUG 2018-04-08 13:57:34,165 [shard 0] demo - entrance: enter
	DEBUG 2018-04-08 13:57:34,165 [shard 0] demo - entrance: leave
	DEBUG 2018-04-08 13:57:34,195 [shard 0] demo - main: done

程序根本没有等到sleep结束就返回了。

为什么仅仅挪到了一下return的位置，程序运行结果差别这么大？

根据seastar源码分析，代码
	
	seastar::sleep(5s).then([] { demo_logger.debug("entrance: done"); });

是指，抛出一个sleep任务，抛出一个lambda（执行打印）任务。sleep任务结束后，开始执行lambda任务。这条语句执行完时，两个任务还没有开始“运行”。

那么，第二个程序的结果才是“符合预期的”：立即返回main结束！

为什么第一个程序非要等这两个任务运行完毕，才从entrance()返回？为什么第二个程序压根儿不等两个任务运行，就从entrance()返回了？

这就是神奇的return的功劳：如果需要return一个future，那么仅当其变成ready future时，才会执行return动作！

换言之，return是一个future阻塞点（ future-based blocking point）。

现在不难理解上面两个程序的运行结果差异了：第一个程序抛出任务，阻塞，等待任务执行完成后返回；第二个程序抛出任务后不管了，直接返回。

事实上，如果将app.run()改成app.run\_deprecated()接口，那么第二个程序会在sleep 5秒后，打印结果，即两个任务有机会运行。

继续改造entrance()，现在将sleep语句放进do_with里，类上，比较return点不同时的运行差异：

	seastar::future<> entrance() {
	    demo_logger.set_level(log_level::debug);
	
	    demo_logger.debug("entrance: enter");
	    
	    return seastar::do_with(5, [](auto& unused) {
	        return seastar::sleep(5s).then([] { demo_logger.debug("entrance: done"); }); // return点在此
	    }).then([]{ demo_logger.debug("entrance: do_with then"); }); // do_with返回时，打印一条日志
	}

运行结果：

	DEBUG 2018-04-08 14:20:22,282 [shard 0] demo - entrance: enter
	DEBUG 2018-04-08 14:20:27,282 [shard 0] demo - entrance: done
	DEBUG 2018-04-08 14:20:27,283 [shard 0] demo - entrance: do_with then
	DEBUG 2018-04-08 14:20:27,315 [shard 0] demo - main: done

挪动return的位置：

	seastar::future<> entrance() {
	    demo_logger.set_level(log_level::debug);
	
	    demo_logger.debug("entrance: enter");
	    
	    return seastar::do_with(5, [](auto& unused) {
	        seastar::sleep(5s).then([] { demo_logger.debug("entrance: done"); });
	        return seastar::make_ready_future<>(); // return点在此
	    }).then([]{ demo_logger.debug("entrance: do_with then"); });
	}

运行结果：

	EBUG 2018-04-08 14:22:10,312 [shard 0] demo - entrance: enter
	DEBUG 2018-04-08 14:22:10,312 [shard 0] demo - entrance: do_with then
	DEBUG 2018-04-08 14:22:10,347 [shard 0] demo - main: done

没等sleep结束就返回了。原理同上。

继续改造entrance()，这次挪动在do\_with处的return：
	
	seastar::future<> entrance() {
	    demo_logger.set_level(log_level::debug);
	
	    demo_logger.debug("entrance: enter");
	
	    seastar::do_with(5, [](auto& unused) {
	        return seastar::sleep(5s).then([] { demo_logger.debug("entrance: done"); });
	    }).then([]{ demo_logger.debug("entrance: do_with then"); });
	
	    demo_logger.debug("entrance: leave");
	    return seastar::make_ready_future<>();  // return点在此
	}

运行结果：

	DEBUG 2018-04-08 14:32:00,197 [shard 0] demo - entrance: enter
	DEBUG 2018-04-08 14:32:00,197 [shard 0] demo - entrance: leave
	DEBUG 2018-04-08 14:32:00,228 [shard 0] demo - main: done

do\_with内的lambda没有运行，其后的then也没有运行。也就是说，do\_with抛出两个任务后，没等返回的future变成ready，就跑去执行下一条语句了。
典型的发射后不管。可见，do\_with前的return遵循同样的规则：**return是阻塞点**。 

继续改造entrance，这次挪动do\_with内，sleep前的return:

	seastar::future<> entrance() {
	    demo_logger.set_level(log_level::debug);
	
	    demo_logger.debug("entrance: enter");
	
	    seastar::do_with(5, [](auto& unused) {
	        seastar::sleep(5s).then([] { demo_logger.debug("entrance: done"); });
	        return seastar::make_ready_future<>(); // return点在此
	    }).then([]{ demo_logger.debug("entrance: do_with then"); });
	
	    demo_logger.debug("entrance: leave");
	    return seastar::make_ready_future<>();
	}

运行结果：

	DEBUG 2018-04-08 14:40:03,987 [shard 0] demo - entrance: enter
	DEBUG 2018-04-08 14:40:03,987 [shard 0] demo - entrance: do_with then
	DEBUG 2018-04-08 14:40:03,987 [shard 0] demo - entrance: leave
	DEBUG 2018-04-08 14:40:04,020 [shard 0] demo - main: done

不是说好的do\_with前没有return，抛出两个任务后不管了吗？为什么do\_with后的then这个任务被执行了呢？

这是由seastar::do\_with()的实现决定的：do\_with被调用时，立即执行lambda，

(a) 如果执行结果返回一个ready future，则继续调用后面的then()接口，执行then()中的lambda；

(b) 如果执行结果返回的future不ready，那么就抛出任务，然后调用后面的then()接口， then()中的lambda只能作为前面抛出任务的continuation，不能被执行。

深刻理解阻塞点、执行序、运行时机，不仅对理解seastar很重要，也是理解并行计算的重要一环。