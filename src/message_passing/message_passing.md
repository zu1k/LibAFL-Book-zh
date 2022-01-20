# Message Passing

LibAFL提供了一个标准的机制，用于在进程和机器上进行低开销的消息传递。
我们使用消息传递来通知其他连接的客户端/模糊器/节点关于新的测试案例、元数据和关于当前运行的统计数据。
根据个人需要，LibAFL也可以将测试案例的内容写到磁盘上，同时仍然使用事件来通知其他模糊器，使用一个 `OnDiskCorpus`.

在我们的测试中，消息传递可以很好地在多个运行中的模糊器实例之间分享新的测试案例和元数据，以进行多核模糊处理。
Specifically, it scales _a lot_ better than using memory locks on a shared corpus, and _a lot_ better than sharing the testcases via the filesystem, as AFL traditionally does.
Think "all cores are green" in `htop`, aka., no kernel interaction.

The `EventManager` interface is used to send Events over the wire using `Low Level Message Passing`, a custom message passing mechanism over shared memory or TCP.

## Low Level Message Passing (LLMP)

LibAFL comes with a reasonably lock-free message passing mechanism that scales well across cores and, using its *broker2broker* mechanism, even to connected machines via TCP.
Most example fuzzers use this mechanism, and it is the best `EventManager` if you want to fuzz on more than a single core.
In the following, we will describe the inner workings of `LLMP`.

`LLMP` has one `broker` process that can forward messages sent by any client process to all other clients.
The broker can also intercept and filter the messages it receives instead of forwarding them.
经纪人过滤的信息的一个常见用例是每个客户直接发送给经纪人的状态信息。
经纪人用这些信息来绘制一个简单的用户界面，其中有所有客户的最新信息，然而其他客户不需要接收这些信息。

### 通过共享内存的快速本地消息

在整个LibAFL中，我们使用了一个围绕不同操作系统的共享地图的包装器，称为`ShMem`。
共享地图，为了不与Rust的`map()`函数相冲突，被称为共享内存，是`LLMP`的骨干。
每个客户，通常是试图分享统计数据和新的测试案例的摸索者，都会映射一个输出的`ShMem`地图。
除了极少数的例外，只有这个客户写到这个地图上，因此，我们不会在竞赛条件下运行，可以不用锁。
代理人从所有客户的`ShMem`映射中读取。
它定期检查所有传入的客户端映射，然后将新消息转发到它的出站广播-`ShMem`，由所有连接的客户端映射。

为了发送新消息，客户端在其共享内存的末端放置一个新消息，然后更新一个静态字段来通知代理。
一旦传出的映射已满，发送者使用各自的`ShMemProvider'分配一个新的`ShMem'。
然后，它使用页面结束（`EOP`）消息发送所需信息，将连接进程中新分配的页面映射到旧的页面。
一旦接收者映射了新的页面，就把它标记为安全的，可以从发送进程中解除映射（如果我们在短时间内有超过一个EOP，就可以避免竞赛条件），然后继续从新的`ShMem`中读取。

The schema for client's maps to the broker is as follows:
```text
[client0]        [client1]    ...    [clientN]
  |                  |                 /
[client0_out] [client1_out] ... [clientN_out]
  |                 /                /
  |________________/                /
  |________________________________/
 \|/
[broker]
```

代理人在所有传入的地图上循环，并检查新的消息。
在`std`构建中，经纪人会在循环后睡眠几毫秒，因为我们不需要消息立即到达。
在经纪人收到来自客户端N的新消息后，（`clientN_out->current_id != last_message->message_id`）经纪人会将消息内容复制到自己的广播共享内存。

客户端定期地，例如在完成`n'次突变后，通过检查是否有新的消息进入（`current_broadcast_map->current_id != last_message->message_id`）。
虽然经纪人使用相同的EOP机制为其传出的地图映射新的`ShMem'，但它从不解除旧页面的映射。
这种额外的内存开销有一个很好的目的：通过保留所有的广播页面，我们确保新的客户可以在以后的时间点加入到模糊测试活动中来
他们只需要从头到尾重新阅读所有广播的信息。

所以传出的消息在传出的广播`Shmem`上是这样流动的。

```text
[broker]
  |
[current_broadcast_shmem]
  |
  |___________________________________
  |_________________                  \
  |                 \                  \
  |                  |                  |
 \|/                \|/                \|/
[client0]        [client1]    ...    [clientN]
```

要在LibAFL中使用`LLMP`，你通常要使用`LlmpEventManager`或其重启的变体。
如果使用 LibAFL 的 `Launcher`，它们是默认的。

如果你想使用`LLMP`的原始形式，没有任何`LibAFL`的抽象，看看[./libafl/examples](https://github.com/AFLplusplus/LibAFL/blob/main/libafl/examples/llmp_test/main.rs)中的`llmp_test`例子。
你可以使用`cargo run --example llmp_test`以适当的模式运行这个例子，正如其帮助输出所指出的。
首先，你必须使用`LlmpBroker::new()`创建一个经纪人。
然后，在其他线程中创建一些`LlmpClient`s`，并使用`LlmpBroker::register_client`在主线程中注册它们。
最后，调用`LlmpBroker::loop_forever()`。

### B2B: 通过TCP连接模糊器

对于 "broker2broker "的通信，所有的广播信息都通过网络套接字转发。
为了方便起见，我们在经纪人中产生了一个额外的客户线程，它可以像其他客户那样读取广播共享内存。
对于broker2的通信，这个b2b客户端监听来自其他远程broker的TCP连接。
它在任何时候都保持一个开放的套接字池，用于连接其他远程的b2b经纪商。
当在本地经纪商共享内存中收到一个新消息时，b2b客户端将通过TCP将其转发给所有连接的远程经纪商。
另外，经纪商可以从所有连接的（远程）经纪商那里接收消息，并通过客户端`ShMem`转发给本地经纪商。

作为附带说明，用于b2b通信的tcp监听器也用于新客户试图连接到本地经纪商时的初始握手，简单地交换初始`ShMem`描述。
