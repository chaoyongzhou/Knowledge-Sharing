# 2 网络连接处理 

现在搭建一个面向TCP连接的网络服务器示例。 示例的原始版本来自： [tutorial](https://github.com/scylladb/seastar/blob/9c4a3389c89ce0ecc55809366c15516f2d46516e/doc/tutorial.md)

改造如下：


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
	
	seastar::logger demo_logger("demo");
	
	int main(int argc, char** argv) {
	    seastar::app_template app;
	
	    try {
	        app.run(argc, argv, entrance);
	    } catch(...) {
	        std::cerr << "Couldn't start application: "
	                  << std::current_exception() << "\n";
	        return 1;
	    }
	    return 0;
	}
	
	
	seastar::future<> service_connection_handle(seastar::connected_socket s,
	                                    seastar::socket_address a) {
	    auto out = s.output();
	    auto in = s.input();
	    return do_with(std::move(s), std::move(out), std::move(in), [] (auto& s, auto& out, auto& in) {
	        return seastar::repeat([&out, &in] {
	            return in.read().then([&out] (auto buf) {  // 从连接中读取数据，并将所读数据返回客户端
	                if (buf) {
	                    demo_logger.debug("service_connection_handle: recv: '{}'", buf.get());
	                    return out.write(std::move(buf)).then([&out] {
	                        demo_logger.debug("service_connection_handle: write done");
	
	                        demo_logger.debug("service_connection_handle: flush");
	                        return out.flush();
	                    }).then([] {
	                        demo_logger.debug("service_connection_handle: send done");
	                        return seastar::stop_iteration::no;
	                    });
	                } else {
	                    demo_logger.debug("service_connection_handle: buf is null");
	                    return seastar::make_ready_future<seastar::stop_iteration>(
	                        seastar::stop_iteration::yes);
	                }
	            });
	        }).then([&out] {
	            return out.close();
	        });
	    });
	}
	
	seastar::future<> service_loop() {
	    seastar::listen_options lo;
	    lo.reuse_address = true;
	    return seastar::do_with(seastar::listen(seastar::make_ipv4_address({9090}), lo), [] (auto& listener) { // 建立网络监听，端口9090
	        return seastar::keep_doing([&listener] () {
	            demo_logger.debug("service_loop: keep_doing enter");
	            return listener.accept().then([] (seastar::connected_socket s, seastar::socket_address a) { // 当有新连接进来时，调用service_connection_handle处理
	                    demo_logger.debug("service_loop: accepted connection from {}", a);    
	                    // Note we ignore, not return, the future returned by
	                    // service_connection_handle(), so we do not wait for one
	                    // connection to be handled before accepting the next one.
	                    service_connection_handle(std::move(s), std::move(a)); // 注意此语句前没有return，即不阻塞
	            });
	        });
	    });
	}
	
	seastar::future<> entrance() {
	    demo_logger.set_level(log_level::debug);
	
	    return seastar::parallel_for_each(boost::irange<unsigned>(0, seastar::smp::count),
	            [] (unsigned c) {
	        return seastar::smp::submit_to(c, service_loop); // 所有核上运行service_loop
	    });
	}

这个服务器的功能是，在9090端口建立服务器监听。如果有客户端新连接接入，则从连接中读取数据，并将数据原封不动发回给客户端。

来看实现。

首先，通过核间消息传递，要求所有核并行运行service\_loop。

service\_loop阻塞在do\_with的lambda，lambda阻塞在keep\_doing的lambda，这是一个无限循环。无限循环中的lambda阻塞在listener accept位置，accept后的连接不阻塞，直接抛出任务去执行service\_connection\_handle。

根据seastar源码分析4.6 “network stack”，只有0核上真正执行了listen，即0核上监听。

开启四个窗口，执行同样的telnet命令：

	telnet 127.1 9090

运行结果：
	
	DEBUG 2018-04-08 15:30:23,421 [shard 0] demo - service_loop: keep_doing enter
	DEBUG 2018-04-08 15:30:23,422 [shard 1] demo - service_loop: keep_doing enter
	DEBUG 2018-04-08 15:30:23,422 [shard 2] demo - service_loop: keep_doing enter
	DEBUG 2018-04-08 15:30:23,422 [shard 3] demo - service_loop: keep_doing enter
	DEBUG 2018-04-08 15:30:28,316 [shard 0] demo - service_loop: accepted connection from 127.0.0.1:56826
	DEBUG 2018-04-08 15:30:28,316 [shard 0] demo - service_loop: keep_doing enter
	DEBUG 2018-04-08 15:30:31,149 [shard 1] demo - service_loop: accepted connection from 127.0.0.1:56828
	DEBUG 2018-04-08 15:30:31,149 [shard 1] demo - service_loop: keep_doing enter
	DEBUG 2018-04-08 15:30:40,956 [shard 2] demo - service_loop: accepted connection from 127.0.0.1:56830
	DEBUG 2018-04-08 15:30:40,957 [shard 2] demo - service_loop: keep_doing enter
	DEBUG 2018-04-08 15:30:45,949 [shard 3] demo - service_loop: accepted connection from 127.0.0.1:56832
	DEBUG 2018-04-08 15:30:45,949 [shard 3] demo - service_loop: keep_doing enter
	
accept发生在四个不同的核上？为什么不是都在0核上呢？

其实，accept确实发生在0核上，根据posix\_server\_socket\_impl::accept()接口的实现，它做了一个Round Robbin的负载均衡器，在0核上accept一个新的连接后，马上通过核间消息，转给了别的核来处理连接。

（注：怀疑posix\_server\_socket\_impl::accept()接口的实现有毛病，咋是递归的呢？）

service\_connection\_handle是一个无限循环：阻塞读数据，然后发送数据。这意味着，在service\_connection\_handle的调用点，如果加上return阻塞，那么listener.accept().then()被阻塞，无法进入seastar::keep\_doing的下一轮循环，
也就是不能调用listener.accept()接入新的连接请求。事实真的如此吗？

实测表明：不是！由于posix\_server\_socket\_impl::accept()接口的奇葩实现，导致新的连接能被接入，但是由于listener.accept().then()被前面连接阻塞，无法调用service\_connection\_handle来处理！（目测seastar这个accept实现有毛病）

无论如何，service\_connection\_handle调用点不能加阻塞。

现在，改造一下service\_connection\_handle：根据客户端不同的输入，指挥不同的核来处理，即消息分发。


	seastar::future<seastar::bool_class<seastar::stop_iteration_tag>>
	service_request_dispose(output_stream<char>&& out, temporary_buffer<char> buf) { // 这里必须要两个&符号，不懂为什么，反正只有这样才能编译过:-(
	    if (buf) {
	        char       *data = buf.get_write();
	        size_t      len  = buf.size();
	
	        /*trim tail spaces and LF*/
	        while(len && ::isspace(data[ len - 1 ])) { -- len; }
	        
	        demo_logger.debug("service_connection_handle: recv: '{}' [{}]", data, len);
	
	        if(1 == len && 0 == ::memcmp("a", data, len)) {  // 客户端输入a，则由核1打印输出
	            demo_logger.debug("service_request_dispose: submit 'a' to core 1");
	            return seastar::smp::submit_to(1, []{
	                demo_logger.debug("service_request_dispose: recv 'a'");
	                return seastar::stop_iteration::no;
	            });
	        }
	
	        if(1 == len && 0 == ::memcmp("b", data, len)) {  // 客户端输入b，则由核2打印输出
	            demo_logger.debug("service_request_dispose: submit 'b' to core 2");
	            return seastar::smp::submit_to(2, []{
	                demo_logger.debug("service_request_dispose: recv 'b'");
	                return seastar::stop_iteration::no;
	            });
	        }    
	
	        if(1 == len && 0 == ::memcmp("c", data, len)) {  // 客户端输入c，则由核3打印输出
	            demo_logger.debug("service_request_dispose: submit 'c' to core 3");
	            return seastar::smp::submit_to(3, []{
	                demo_logger.debug("service_request_dispose: recv 'c'");
	                return seastar::stop_iteration::no;
	            });
	        }
	
	        if(1 == len && 0 == ::memcmp("q", data, len)) { // 客户端输入q，则返回ready future，触发service_connection_handle中的out.close()，关断与客户端的连接
	            return seastar::make_ready_future<seastar::stop_iteration>(
	            seastar::stop_iteration::yes);
	        }
	
	        return out.write(std::move(buf)).then([&out] {
	            demo_logger.debug("service_connection_handle: write done");
	    
	            demo_logger.debug("service_connection_handle: flush");
	            return out.flush();
	        }).then([] {
	            demo_logger.debug("service_connection_handle: send done");
	            return seastar::stop_iteration::no;
	        });
	    } else {
	        demo_logger.debug("service_connection_handle: buf is null");
	        return seastar::make_ready_future<seastar::stop_iteration>(
	            seastar::stop_iteration::yes);
	    }
	}
	
	
	seastar::future<> service_connection_handle(seastar::connected_socket s,
	                                    seastar::socket_address a) {
	    auto out = s.output();
	    auto in = s.input();
	    return do_with(std::move(s), std::move(out), std::move(in), [] (auto& s, auto& out, auto& in) {
	        return seastar::repeat([&out, &in] {
	            return in.read().then([&out] (auto buf) {
	                return service_request_dispose(std::move(out), buf.share()); // 从客户端连接读取数据后，交给请求分发接口
	            });
	        }).then([&out] {
	            return out.close();
	        });
	    });
	}

运行结果：


	DEBUG 2018-04-08 16:10:44,333 [shard 0] demo - service_loop: keep_doing enter
	DEBUG 2018-04-08 16:10:44,334 [shard 2] demo - service_loop: keep_doing enter
	DEBUG 2018-04-08 16:10:44,334 [shard 1] demo - service_loop: keep_doing enter
	DEBUG 2018-04-08 16:10:44,335 [shard 3] demo - service_loop: keep_doing enter
	DEBUG 2018-04-08 16:10:49,719 [shard 0] demo - service_loop: accepted connection from 127.0.0.1:56844
	DEBUG 2018-04-08 16:10:49,720 [shard 0] demo - service_loop: keep_doing enter
	DEBUG 2018-04-08 16:10:53,716 [shard 0] demo - service_connection_handle: recv: 'a
	' [1]
	DEBUG 2018-04-08 16:10:53,716 [shard 0] demo - service_request_dispose: submit 'a' to core 1
	DEBUG 2018-04-08 16:10:53,716 [shard 1] demo - service_request_dispose: recv 'a'
	DEBUG 2018-04-08 16:10:58,611 [shard 0] demo - service_connection_handle: recv: 'b
	' [1]
	DEBUG 2018-04-08 16:10:58,612 [shard 0] demo - service_request_dispose: submit 'b' to core 2
	DEBUG 2018-04-08 16:10:58,613 [shard 2] demo - service_request_dispose: recv 'b'
	DEBUG 2018-04-08 16:11:00,915 [shard 0] demo - service_connection_handle: recv: 'c
	' [1]
	DEBUG 2018-04-08 16:11:00,915 [shard 0] demo - service_request_dispose: submit 'c' to core 3
	DEBUG 2018-04-08 16:11:00,915 [shard 3] demo - service_request_dispose: recv 'c'
	DEBUG 2018-04-08 16:11:02,818 [shard 0] demo - service_connection_handle: recv: 'q
	' [1]

从核0分发到不同的核上。
（注：还有一个未解问题，buff如何分发到不同核上。貌似这个temporary\_buffer不好使。）


