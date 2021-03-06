[English version](README.md)

# 什么是RPC?

互联网上的机器大都通过[TCP/IP协议](http://en.wikipedia.org/wiki/Internet_protocol_suite)相互访问，但TCP/IP只是往远端发送了一段二进制数据，为了建立服务还有很多问题需要抽象：

- 数据以什么格式传输？不同机器间，网络间可能是不同的字节序，直接传输内存数据显然是不合适的；随着业务变化，数据字段往往要增加或删减，怎么兼容前后不同版本的格式？
- 一个TCP连接可以被多个请求复用以减少开销么？多个请求可以同时发往一个TCP连接么?
- 如何访问一个包含很多机器的集群？
- 连接断开时应该干什么？万一server不发送回复怎么办？

* ...

[RPC](http://en.wikipedia.org/wiki/Remote_procedure_call)可以解决这些问题，它把网络交互类比为“client访问server上的函数”：client向server发送request后开始等待，直到server收到、处理、回复client后，client又再度恢复并根据response做出反应。

![rpc.png](docs/images/rpc.png)

我们来看看上面的一些问题是如何解决的：

- RPC需要序列化，[protobuf](https://github.com/google/protobuf)在这方面做的很好。用户填写protobuf::Message类型的request，RPC结束后，从同为protobuf::Message类型的response中取出结果。protobuf有较好的前后兼容性，方便业务调整字段。http广泛使用[json](http://www.json.org/)作为序列化方法。
- 用户不需要关心连接是如何建立的，但可以选择不同的[连接方式](docs/cn/client.md#连接方式)：短连接，连接池，单连接。
- 一个集群中的所有机器一般通过名字服务被发现，可基于[DNS](https://en.wikipedia.org/wiki/Domain_Name_System), [ZooKeeper](https://zookeeper.apache.org/), [etcd](https://github.com/coreos/etcd)等实现。在百度内，我们使用BNS (Baidu Naming Service)。brpc也提供["list://"和"file://"](docs/cn/client.md#名字服务)。用户可以指定负载均衡算法，让RPC每次选出一台机器发送请求，包括: round-robin, randomized, [consistent-hashing](docs/cn/consistent_hashing.md)(murmurhash3 or md5)和 [locality-aware](docs/cn/lalb.md).
- 连接断开时按用户的配置进行重试。如果server没有在给定时间内返回response，那么client会返回超时错误。

# 哪里可以使用RPC?

几乎所有的网络交互。

RPC不是万能的抽象，否则我们也不需要TCP/IP这一层了。但是在我们绝大部分的网络交互中，RPC既能解决问题，又能隔离更底层的网络问题。

对于RPC常见的质疑有：

- 我的数据非常大，用protobuf序列化太慢了。首先这可能是个伪命题，你得用[profiler](docs/cn/cpu_profiler.md)证明慢了才是真的慢，其次很多协议支持携带二进制数据以绕过序列化。
- 我传输的是流数据，RPC表达不了。事实上brpc中很多协议支持传递流式数据，包括[http中的ProgressiveReader](docs/cn/http_client.md#持续下载), h2的streams, [streaming rpc](docs/cn/streaming_rpc.md), 和专门的流式协议RTMP。
- 我的场景不需要回复。简单推理可知，你的场景中请求可丢可不丢，可处理也可不处理，因为client总是无法感知，你真的确认这是OK的？即使场景真的不需要，我们仍然建议用最小的结构体回复，因为这不大会是瓶颈，并且追查复杂bug时可能是很有价值的线索。

# 什么是![brpc](docs/images/logo.png)?

百度内最常使用的工业级RPC框架, 有超过**600,000**个实例(不包含client)和**500**多种服务, 在百度内叫做"**baidu-rpc**". 目前只开源C++版本。

你可以使用它：

* 搭建一个能在**同端口**支持多协议的服务, 或访问各种服务
  * restful http/https, h2/h2c (与[grpc](https://github.com/grpc/grpc)兼容, 即将开源). 使用brpc的http实现比[libcurl](https://curl.haxx.se/libcurl/)方便多了。
  * [redis](docs/cn/redis_client.md)和[memcached](docs/cn/memcache_client.md), 线程安全，比官方client更方便。
  * [rtmp](https://github.com/brpc/brpc/blob/master/src/brpc/rtmp.h)/[flv](https://en.wikipedia.org/wiki/Flash_Video)/[hls](https://en.wikipedia.org/wiki/HTTP_Live_Streaming), 可用于搭建[直播服务](docs/cn/live_streaming.md).
  * hadoop_rpc(仍未开源)
  * 基于[openucx](https://github.com/openucx/ucx)支持[rdma](https://en.wikipedia.org/wiki/Remote_direct_memory_access)(即将开源)
  * 各种百度内使用的协议: [baidu_std](docs/cn/baidu_std.md), [streaming_rpc](docs/cn/streaming_rpc.md), hulu_pbrpc, [sofa_pbrpc](https://github.com/baidu/sofa-pbrpc), nova_pbrpc, public_pbrpc, ubrpc和使用nshead的各种协议.
  * 从其他语言通过HTTP+json访问基于protobuf的协议.
  * 基于工业级的[RAFT算法](https://raft.github.io)实现搭建[高可用](https://en.wikipedia.org/wiki/High_availability)分布式系统 (即将在[braft](https://github.com/brpc/braft)开源)
* 创建丰富的访问模式
  * 服务都能以[同步](docs/cn/server.md)或[异步](docs/cn/server.md#异步service)方式处理请求。
  * 通过[同步](docs/cn/client.md#同步访问)、[异步](docs/cn/client.md#异步访问)或[半同步](docs/cn/client.md#半同步)访问服务。
  * 使用[组合channels](docs/cn/combo_channel.md)声明式地简化复杂的分库或并发访问。
* [通过http](docs/cn/builtin_service.md)调试服务, 使用[cpu](docs/cn/cpu_profiler.md), [heap](docs/cn/heap_profiler.md), [contention](docs/cn/contention_profiler.md) profilers.
* 获得[更好的延时和吞吐](#更好的延时和吞吐).
* 把你组织中使用的协议快速地[加入brpc](docs/cn/new_protocol.md)，或定制各类组件, 包括[名字服务](docs/cn/load_balancing.md#名字服务) (dns, zk, etcd), [负载均衡](docs/cn/load_balancing.md#负载均衡) (rr, random, consistent hashing)

# brpc的优势

### 更友好的接口

只有三个(主要的)用户类: [Server](https://github.com/brpc/brpc/blob/master/src/brpc/server.h), [Channel](https://github.com/brpc/brpc/blob/master/src/brpc/channel.h), [Controller](https://github.com/brpc/brpc/blob/master/src/brpc/controller.h), 分别对应server端，client端，参数集合. 你不必推敲诸如"如何初始化XXXManager", "如何组合各种组件",  "XXXController的XXXContext间的关系是什么"。要做的很简单:

* 建服务? 包含[brpc/server.h](https://github.com/brpc/brpc/blob/master/src/brpc/server.h)并参考注释或[示例](https://github.com/brpc/brpc/blob/master/example/echo_c++/server.cpp).
* 访问服务? 包含[brpc/channel.h](https://github.com/brpc/brpc/blob/master/src/brpc/channel.h)并参考注释或[示例](https://github.com/brpc/brpc/blob/master/example/echo_c++/client.cpp).
* 调整参数? 看看[brpc/controller.h](https://github.com/brpc/brpc/blob/master/src/brpc/controller.h). 注意这个类是Server和Channel共用的，分成了三段，分别标记为Client-side, Server-side和Both-side methods。

我们尝试让事情变得更加简单，以名字服务为例，在其他RPC实现中，你也许需要复制一长段晦涩的代码才可使用，而在brpc中访问BNS可以这么写"bns://node-name"，DNS是`Init("http://domain-name", ...)`，本地文件列表是"file:///home/work/server.list"，相信不用解释，你也能明白这些代表什么。

### 使服务更加可靠

brpc在百度内被广泛使用:

* map-reduce服务和table存储
* 高性能计算和模型训练
* 各种索引和排序服务
* ….

它是一个经历过考验的实现。

brpc特别重视开发和维护效率, 你可以通过浏览器或curl[查看server内部状态](docs/cn/builtin_service.md), 分析在线服务的[cpu热点](docs/cn/cpu_profiler.md), [内存分配](docs/cn/heap_profiler.md)和[锁竞争](docs/cn/contention_profiler.md), 通过[bvar](docs/cn/bvar.md)统计各种指标并通过[/vars](docs/cn/vars.md)查看。

### 更好的延时和吞吐

虽然大部分RPC实现都声称“高性能”，但数字仅仅是数字，要在广泛的场景中做到高性能仍是困难的。为了统一百度内的通信架构，brpc在性能方面比其他RPC走得更深。

- 对不同客户端请求的读取和解析是完全并发的，用户也不用区分”IO线程“和”处理线程"。其他实现往往会区分“IO线程”和“处理线程”，并把[fd](http://en.wikipedia.org/wiki/File_descriptor)（对应一个客户端）散列到IO线程中去。当一个IO线程在读取其中的fd时，同一个线程中的fd都无法得到处理。当一些解析变慢时，比如特别大的protobuf message，同一个IO线程中的其他fd都遭殃了。虽然不同IO线程间的fd是并发的，但你不太可能开太多IO线程，因为这类线程的事情很少，大部分时候都是闲着的。如果有10个IO线程，一个fd能影响到的”其他fd“仍有相当大的比例（10个即10%，而工业级在线检索要求99.99%以上的可用性）。这个问题在fd没有均匀地分布在IO线程中，或在多租户(multi-tenacy)环境中会更加恶化。在brpc中，对不同fd的读取是完全并发的，对同一个fd中不同消息的解析也是并发的。解析一个特别大的protobuf message不会影响同一个客户端的其他消息，更不用提其他客户端的消息了。更多细节看[这里](docs/cn/io.md#收消息)。
- 对同一fd和不同fd的写出是高度并发的。当多个线程都要对一个fd写出时（常见于单连接），第一个线程会直接在原线程写出，其他线程会以[wait-free](http://en.wikipedia.org/wiki/Non-blocking_algorithm#Wait-freedom)的方式托付自己的写请求，多个线程在高度竞争下仍可以在1秒内对同一个fd写入500万个16字节的消息。更多细节看[这里](docs/cn/io.md#发消息)。
- 尽量少的锁。高QPS服务可以充分利用一台机器的CPU。比如为处理请求[创建bthread](docs/cn/memory_management.md), [设置超时](docs/cn/timer_keeping.md), 根据回复[找到RPC上下文](docs/cn/bthread_id.md), [记录性能计数器](docs/cn/bvar.md)都是高度并发的。即使服务的QPS超过50万，用户也很少在[contention profiler](docs/cn/contention_profiler.md))中看到框架造成的锁竞争。
- 服务器线程数自动调节。传统的服务器需要根据下游延时的调整自身的线程数，否则吞吐可能会受影响。在brpc中，每个请求均运行在新建立的[bthread](docs/cn/bthread.md)中，请求结束后线程就结束了，所以天然会根据负载自动调节线程数。

brpc和其他实现的性能对比见[这里](docs/cn/benchmark.md)。

# 试一下!

* 从[编译步骤](docs/cn/getting_started.md)开始.
* 玩一下[示例程序](https://github.com/brpc/brpc/tree/master/example/).
* 文档:
  * [性能测试](docs/cn/benchmark.md)
  * [bvar](docs/cn/bvar.md)
    * [bvar_c++](docs/cn/bvar_c++.md)
  * [bthread](docs/cn/bthread.md)
    * [bthread or not](docs/cn/bthread_or_not.md)
    * [thread-local](docs/cn/thread_local.md)
    * [Execution Queue](docs/cn/execution_queue.md)
  * Client
    * [基础功能](docs/cn/client.md)
    * [错误码](docs/cn/error_code.md)
    * [组合channels](docs/cn/combo_channel.md)
    * [访问HTTP](docs/cn/http_client.md)
    * [访问UB](docs/cn/ub_client.md)
    * [Streaming RPC](docs/cn/streaming_rpc.md)
    * [访问redis](docs/cn/redis_client.md)
    * [访问memcached](docs/cn/memcache_client.md)
    * [Backup request](docs/cn/backup_request.md)
    * [Dummy server](docs/cn/dummy_server.md)
  * Server
    * [基础功能](docs/cn/server.md)
    * [搭建HTTP服务](docs/cn/http_service.md)
    * [搭建Nshead服务](docs/cn/nshead_service.md)
    * [高效率排查server卡顿](docs/cn/server_debugging.md)
    * [推送](docs/cn/server_push.md)
    * [雪崩](docs/cn/avalanche.md)
    * [直播](docs/cn/live_streaming.md)
    * [json2pb](docs/cn/json2pb.md)
  * [内置服务](docs/cn/builtin_service.md)
    * [status](docs/cn/status.md)
    * [vars](docs/cn/vars.md)
    * [connections](docs/cn/connections.md)
    * [flags](docs/cn/flags.md)
    * [rpcz](docs/cn/rpcz.md)
    * [cpu_profiler](docs/cn/cpu_profiler.md)
    * [heap_profiler](docs/cn/heap_profiler.md)
    * [contention_profiler](docs/cn/contention_profiler.md)
  * 工具
    * [rpc_press](docs/cn/rpc_press.md)
    * [rpc_replay](docs/cn/rpc_replay.md)
    * [rpc_view](docs/cn/rpc_view.md)
    * [benchmark_http](docs/cn/benchmark_http.md)
    * [parallel_http](docs/cn/parallel_http.md)
  * 其他
    * [IOBuf](docs/cn/iobuf.md)
    * [Streaming Log](docs/cn/streaming_log.md)
    * [FlatMap](docs/cn/flatmap.md)
    * [brpc外功修炼宝典](docs/cn/brpc_intro.pptx)(新人培训材料)
  * 深入RPC
    * [New Protocol](docs/cn/new_protocol.md)
    * [Atomic instructions](docs/cn/atomic_instructions.md)
    * [IO](docs/cn/io.md)
    * [Threading Overview](docs/cn/threading_overview.md)
    * [Load Balancing](docs/cn/load_balancing.md)
    * [Locality-aware](docs/cn/lalb.md)
    * [Consistent Hashing](docs/cn/consistent_hashing.md)
    * [Memory Management](docs/cn/memory_management.md)
    * [Timer keeping](docs/cn/timer_keeping.md)
    * [bthread_id](docs/cn/bthread_id.md)
  * Use cases inside Baidu
    * [百度地图api入口](docs/cn/case_apicontrol.md)
    * [联盟DSP](docs/cn/case_baidu_dsp.md)
    * [ELF学习框架](docs/cn/case_elf.md)
    * [云平台代理服务](docs/cn/case_ubrpc.md)
