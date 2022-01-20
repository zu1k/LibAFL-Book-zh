# 派生实例

多个模糊器实例可以通过不同的方式产生。

## 手动，通过一个TCP端口

做多线程的直接方法是使用`LlmpRestartingEventManager`，特别是使用`setup_restarting_mgr_std`。

它抽象化了所有讨厌的细节，如崩溃处理时的重启（针对内存模糊器）和多线程。
有了它，你手动启动的每个实例都会尝试连接到本地机器上的一个TCP端口。

如果这个端口还没有被绑定，这个实例就会成为代理，它自己会绑定到这个端口以等待新的客户。

如果该端口已经被绑定，EventManager将尝试连接到它。
该实例成为客户端，现在可以与所有其他节点通信。

手动启动节点的好处是，你可以有多个具有不同配置的节点，比如客户端在有ASAN和没有ASAN的情况下进行模糊处理。

虽然它被称为 "重启 "管理器，但它在Unix操作系统上使用`fork`作为优化，在Windows上只实际从头开始重启。

## Launcher

启动器是lazy模式启动多线程。

你可以使用`Launcher::builder`来创建一个产生多个节点的模糊器，所有这些节点都使用重新启动的事件管理器。

看一个例子:\r

```rust,ignore
    Launcher::builder()
        .configuration(EventConfig::from_name(&configuration))
        .shmem_provider(shmem_provider)
        .monitor(mon)
        .run_client(&mut run_client)
        .cores(cores)
        .broker_port(broker_port)
        .stdout_file(stdout_file)
        .remote_broker_addr(broker_addr)
        .build()
        .launch()
```

首先启动一个代理，然后根据传递给 `cores` 的值生成 `n` 个客户端。
这个值是一个字符串，表示要绑定的核心，例如， `0,2,5` 或 `0-3`。
对于每个客户端，`run_client'将被调用。
在Windows上，启动器将重新启动每个客户端，而在Unix上，它将使用`fork`。

## 其他方式

LlmpEvenManager系列是做产卵实例的最简单的方法，但对于不明显的目标，你可能需要想出其他的解决方案。

LLMP甚至在理论上是 `no_std` 兼容的，甚至完全不同的EventManagers也可以用于消息传递。

如果你遇到这种情况，请阅读当前的实现和/或与我们联系。