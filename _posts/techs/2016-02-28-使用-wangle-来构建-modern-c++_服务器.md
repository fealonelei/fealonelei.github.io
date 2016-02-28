---
layout: post
title: 使用 Wangle 来构建 modern C++ 服务器
category: 开发
tags: C++ server wangle
keywords: modern C++, Wangle, high performance server
description:
---


>原文地址： <https://medium.com/hacker-daily/writing-high-performance-servers-in-modern-c-7cd00926828#.j5bditvpp> 

首先，感谢我第一篇博客的反馈 -- [Starting a tech startup with C++](https://medium.com/swlh/starting-a-tech-startup-with-c-6b5d5856e6de#.lfean82zv) 

>我将展示如何用 48 行代码构建一个 C++ 高性能异步服务器。

我在之前的文章里提到我能够用 Facebook 的 [Wangle](https://github.com/facebook/wangle) 在一天内构建一个数据库引擎的原型。这篇文章将讲讲我怎么做到。在本文结尾，你能够完成一个基于 wangle 的高性能 C++ 服务器。这篇文章也将成为 Wangle 的 README.md 里面的教程。<br />
我将展示怎么使用 modern C++ 写一个 echo server，（echo server）在分布式系统中相当于 “hello world”. 服务器响应每条 message 的方式是将该 message 原封返回。我们也将写一个客户端来向 echo server 发送 messages. 可以在[这里](https://github.com/facebook/wangle/tree/master/wangle/example/echo)找到源代码。<br />
Wangle 是构建异步，事件驱动的 modern C++ 服务的 client/server 应用框架。 Wangle 的基本抽象是 _Pipeline_ . 一旦完全理解了这种抽象，你讲能够写出各种复杂的 modern C++ 服务。另一个重要的抽象是 _Service_ , Service 是 pipeline 的高级版本。但是，本文不讨论 Service. <br />

***

## Pipeline 
Pipeline 是 Wangle 里最重要的也是最强大的抽象。Pipeline 为自定义 requests 和 responses 如何被 Services 处理提供了无与伦比的灵活性。 <br />
Pipeline 是一系列的 request/response handlers. 我认为这里的 Pipeline 就像制造业工厂的生产线一样。在一条按顺序工作的生产线上，每个工人收到一个物件，完成一个工序，把它给到下一个工人那里，直到完成制造。这个比喻其实不恰当，它只能朝一个方向前进而不能反向将制成品变回原材料，而 _Pipeline_ 可以这么反方向做。 <br />
一个 Wangle handler 可以处理下游（处理响应）也可以处理上游（处理请求）事件。只要将所有的 handlers 连在一起，那就可以非常灵活地将原始数据流（raw data stream）转换成需要的信息类型（message type， 比如 class）或者反方向，将需要的信息类型转换成原始数据流。 <br />
举个🌰， 在 echo service 的 pipeline 里，我们可以建立一个包含下面几个 handler 的 pipeline.         

1. Handler 1
上游（Upstream）：从 socket 里接收一个原始的二进制数据，将其读进一个零拷贝字节的缓冲器，然后发送到 handler 2.    <br />
下游（Downstream）：接收零拷贝字节的缓冲器的数据，将其内容从 socket 发送。

2. Handler 2
上游：接收来自 handler 1 的零拷贝字节的缓冲器，将字节缓冲器解码成 string，并将 string 发送到 handler 3. <br />
下游：接收来自 handler 3 的 string， 将 std::string 编码成零拷贝字节的缓冲器，并发送到 handler 1.

3. Handler 3 
上游：接收来自 handler 2 的 string，并通过 Pipeline 发送回客户端。这里 handler 3 将 string 发送回 handler 2.
下游：从上游接收 std::string 并传递给 handler 2.

这里有一个重要的地方就是每一个 handler 能且只能做一件事。如果你有一个 handler 处理多于一个函数，像直接将原始数据流解码成 string，那么你需要将其分解成几个 handler. 这对于最大化可维护性和灵活性很重要。 <br /> 
handler 是**线程不安全**的，所以千万不要加任何不由 mutex/atomic lock 等守护的共享态。如果想要线程安全的容器，请使用 [Folly](https://github.com/facebook/folly) 无锁数据结构，它是 Wangle 的依赖，而且速度超快，引入它也是很简单的。 <br />
现在不要担心 --- 当看到实际代码时，它自然就清晰多了。

***

## echo server 
下面展示怎么用 Wangle 来写第一个 C++ echo server. 请先安装 Wangle. 不过 Wangle 不能在 OSX 上 build，所以我推荐使用带图形界面的 Ubuntu 14.04 来安装 Wangle. <br />
这里是 echo handler，接收 string，通过 stdout 打印，并在 Pipeline 里发送回下游。加入一个行定界符很重要，因为我们的 Pipeline 将使用行解码器（line decoder）--- 将字节缓冲器分解成行分割的字节缓冲器。

**EchoHandler.cpp** 

	// the main logic of our echo server; receives a string and writes it straight
	// back
	class EchoHandler : public HandlerAdapter<std::string> {
	 public:
	  virtual void read(Context* ctx, std::string msg) override {
	    std::cout << "handling " << msg << std::endl;
	    write(ctx, msg + "\r\n");
	  }
	};

这是 Pipeline 里的 final handler. 我们需要新建一个 _PipelineFactory_ ,并在这里定义如何处理请求和响应。

**EchoPipelineFactory.cpp**

	// where we define the chain of handlers for each messeage received
	class EchoPipelineFactory : public PipelineFactory<EchoPipeline> {
	 public:
	  EchoPipeline::Ptr newPipeline(std::shared_ptr<AsyncTransportWrapper> sock) {
	    auto pipeline = EchoPipeline::create();
	    pipeline->addBack(AsyncSocketHandler(sock));
	    pipeline->addBack(LineBasedFrameDecoder(8192));
	    pipeline->addBack(StringCodec());
	    pipeline->addBack(EchoHandler());
	    pipeline->finalize();
	    return pipeline;
	  }
	};

**一定要严格按照上面的顺序来将几个 handler 加入到 pipeline** <br />
这里的 Pipeline 有 4 个 handlers ：

1. AsyncSocketHandler:
上游：从 socket 中读取原始数据流，并将其变成零拷贝字节缓冲器。
下游：将零拷贝字节缓冲器的内容从下属的 socket 发出。

2. LineBasedFrameDecoder:
上游：接收零拷贝字节缓冲器，并在行尾分割。
上游：将零拷贝字节缓冲器发送给 _AsyncSocketHandler_

3. StringCodec:
上游：接收零拷贝字节缓冲器并解码成 std::string 发送给 EchoHandler.
下游：接收 std::string 并编码成零拷贝字节缓冲器，发送给 LineBasedFrameDecoder.

4. EchoHandler:
上游：接收 std::stirng 并将其写入到 pipeline，Pipeline 将把 message 发送到下游。
下游：接收 std::string 并转发给 StringCodec.

那么现在要做就是将 Pipeline factory 塞进 _ServerBootStrap_， 好了。绑定端口并等它停止。

**EchoServer.cpp**

	
	#include <gflags/gflags.h>
	
	#include <wangle/bootstrap/ServerBootstrap.h>
	#include <wangle/channel/AsyncSocketHandler.h>
	#include <wangle/codec/LineBasedFrameDecoder.h>
	#include <wangle/codec/StringCodec.h>
	
	using namespace folly;
	using namespace wangle;
	
	DEFINE_int32(port, 8080, "echo server port");
	
	typedef Pipeline<IOBufQueue&, std::string> EchoPipeline;
	
	// the main logic of our echo server; receives a string and writes it straight
	// back
	class EchoHandler : public HandlerAdapter<std::string> {
	 public:
	  virtual void read(Context* ctx, std::string msg) override {
	    std::cout << "handling " << msg << std::endl;
	    write(ctx, msg + "\r\n");
	  }
	};
	
	// where we define the chain of handlers for each messeage received
	class EchoPipelineFactory : public PipelineFactory<EchoPipeline> {
	 public:
	  EchoPipeline::Ptr newPipeline(std::shared_ptr<AsyncTransportWrapper> sock) {
	    auto pipeline = EchoPipeline::create();
	    pipeline->addBack(AsyncSocketHandler(sock));
	    pipeline->addBack(LineBasedFrameDecoder(8192));
	    pipeline->addBack(StringCodec());
	    pipeline->addBack(EchoHandler());
	    pipeline->finalize();
	    return pipeline;
	  }
	};
	
	int main(int argc, char** argv) {
	  google::ParseCommandLineFlags(&argc, &argv, true);
	
	  ServerBootstrap<EchoPipeline> server;
	  server.childPipeline(std::make_shared<EchoPipelineFactory>());
	  server.bind(FLAGS_port);
	  server.waitForStop();
	
	  return 0;
	}

这样使用 48 行代码就完成了一个高性能异步的 C++ 服务器。

***

## Echo Client 
echo client 和 echo server 的代码很相似。

**EchoClientHandler.cpp**

	// the handler for receiving messages back from the server
	class EchoHandler : public HandlerAdapter<std::string> {
	 public:
	  virtual void read(Context* ctx, std::string msg) override {
	    std::cout << "received back: " << msg;
	  }
	  virtual void readException(Context* ctx, exception_wrapper e) override {
	    std::cout << exceptionStr(e) << std::endl;
	    close(ctx);
	  }
	  virtual void readEOF(Context* ctx) override {
	    std::cout << "EOF received :(" << std::endl;
	    close(ctx);
	  }
	};

这里重写 _readException_ 和 _readEOF_ 函数。还有一些其他的函数可以重写。如果需要处理特定的事件，重写相应的虚函数就行了。 <br />
这里是 client 的 Pipeline factory. 这里和 EventBaseHandler 分离出的 server 端的 pipeline factory 基本一致， EventBaseHandler 从事件循环线程中处理写入数据。

**EchoClientPipeline.cpp**

	// chains the handlers together to define the response pipeline
	class EchoPipelineFactory : public PipelineFactory<EchoPipeline> {
	 public:
	  EchoPipeline::Ptr newPipeline(std::shared_ptr<AsyncTransportWrapper> sock) {
	    auto pipeline = EchoPipeline::create();
	    pipeline->addBack(AsyncSocketHandler(sock));
	    pipeline->addBack(
	        EventBaseHandler()); // ensure we can write from any thread
	    pipeline->addBack(LineBasedFrameDecoder(8192, false));
	    pipeline->addBack(StringCodec());
	    pipeline->addBack(EchoHandler());
	    pipeline->finalize();
	    return pipeline;
	  }
	};

完整的 client 端的代码如下：

**EchoClient.cpp**

	
	#include <gflags/gflags.h>
	#include <iostream>
	
	#include <wangle/bootstrap/ClientBootstrap.h>
	#include <wangle/channel/AsyncSocketHandler.h>
	#include <wangle/channel/EventBaseHandler.h>
	#include <wangle/codec/LineBasedFrameDecoder.h>
	#include <wangle/codec/StringCodec.h>
	
	using namespace folly;
	using namespace wangle;
	
	DEFINE_int32(port, 8080, "echo server port");
	DEFINE_string(host, "::1", "echo server address");
	
	typedef Pipeline<folly::IOBufQueue&, std::string> EchoPipeline;
	
	// the handler for receiving messages back from the server
	class EchoHandler : public HandlerAdapter<std::string> {
	 public:
	  virtual void read(Context* ctx, std::string msg) override {
	    std::cout << "received back: " << msg;
	  }
	  virtual void readException(Context* ctx, exception_wrapper e) override {
	    std::cout << exceptionStr(e) << std::endl;
	    close(ctx);
	  }
	  virtual void readEOF(Context* ctx) override {
	    std::cout << "EOF received :(" << std::endl;
	    close(ctx);
	  }
	};
	
	// chains the handlers together to define the response pipeline
	class EchoPipelineFactory : public PipelineFactory<EchoPipeline> {
	 public:
	  EchoPipeline::Ptr newPipeline(std::shared_ptr<AsyncTransportWrapper> sock) {
	    auto pipeline = EchoPipeline::create();
	    pipeline->addBack(AsyncSocketHandler(sock));
	    pipeline->addBack(
	        EventBaseHandler()); // ensure we can write from any thread
	    pipeline->addBack(LineBasedFrameDecoder(8192, false));
	    pipeline->addBack(StringCodec());
	    pipeline->addBack(EchoHandler());
	    pipeline->finalize();
	    return pipeline;
	  }
	};
	
	int main(int argc, char** argv) {
	  google::ParseCommandLineFlags(&argc, &argv, true);
	
	  ClientBootstrap<EchoPipeline> client;
	  client.group(std::make_shared<wangle::IOThreadPoolExecutor>(1));
	  client.pipelineFactory(std::make_shared<EchoPipelineFactory>());
	  auto pipeline = client.connect(SocketAddress(FLAGS_host, FLAGS_port)).get();
	
	  try {
	    while (true) {
	      std::string line;
	      std::getline(std::cin, line);
	      if (line == "") {
	        break;
	      }
	
	      pipeline->write(line + "\r\n").get();
	      if (line == "bye") {
	        pipeline->close();
	        break;
	      }
	    }
	  } catch (const std::exception& e) {
	    std::cout << exceptionStr(e) << std::endl;
	  }
	
	  return 0;
	}

它在一个 loop 中读取 stdin 的输入并写入到 Pipeline, 在响应被处理前它都是阻塞的。 <br /> 

***

## Summary 
上述就是使用 Wangle 来写一个基本的高性能 modern C++ 服务器。你应该了解 Wangle 的基础并有信心来用 C++ 写自己的服务。强烈推荐深入理解 Wangle 里的 Service 抽象并用来构建复杂的 Server.
