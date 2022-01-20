# 执行器

在不同的fuzzers中，这种执行被测程序的概念意味着每次运行都是一样的。
例如，对于像libFuzzer这样的内存模糊器来说，执行是对一个甲骨文函数的调用，而对于像[kAFL](https://github.com/IntelLabs/kAFL)这样的基于管理程序的模糊器来说，每次运行都会从一个快照启动整个操作系统。

在我们的模型中，执行者是一个实体，它不仅定义了如何执行目标，还定义了所有与目标的单一运行有关的易失性操作。

因此，执行者负责告知程序模糊器要在运行中使用的输入，例如写到一个内存位置或作为参数传递给线束函数。

在我们的模型中，它还可以持有一组与每个执行程序相关的观察者。

在Rust中，我们将这个概念与[`Executor`](https://docs.rs/libafl/0/libafl/executors/trait.Executor.html)特质绑定。实现这个特性的结构如果想持有一组观察者，也必须实现[`HasObservers`](https://docs.rs/libafl/0/libafl/executors/trait.HasObservers.html)。

默认情况下，我们实现了一些常用的执行器，如[`InProcessExecutor`](https://docs.rs/libafl/0/libafl/executors/inprocess/struct.InProcessExecutor.html)，其目标是提供进程中崩溃检测的线程函数。另一个执行器是[`ForkserverExecutor`](https://docs.rs/libafl/0/libafl/executors/forkserver/struct.ForkserverExecutor.html)，它实现了一个类似于AFL的机制，用来生成子进程进行模糊处理。

在创建执行器时，一个常见的模式是包装现有的执行器，例如[`TimeoutExecutor`](https://docs.rs/libafl/0.6.1/libafl/executors/timeout/struct.TimeoutExecutor.html)包装了一个执行器，并在调用被包装的执行器的原始运行函数之前安装一个超时回调。

## InProcessExecutor

让我们从基本情况开始；`InProcessExecutor`。
这个执行器使用[_SanitizerCoverage_](https://clang.llvm.org/docs/SanitizerCoverage.html)作为其后端，你可以在`libafl_targets/src/sancov_pcguards`中找到相关代码。在这里，我们分配了一个名为 "EDGES_MAP "的地图，然后我们的编译器包装器编译线束，将覆盖率写入这个地图中。
当你想尽可能快地执行线束时，你很可能想使用这个`InprocessExecutor`。 
 这里需要注意的是，当你的线束有可能出现堆损坏的问题时，你要使用另一个分配器，这样损坏的堆就不会影响到模糊器本身。(例如，我们在一些模糊器中采用MiMalloc）。另外，你也可以用地址净化器来编译你的线束，以确保你能捕捉到这些堆的错误。

## ForkserverExecutor

接下来，我们来看看`ForkserverExecutor`。在这种情况下，是`afl-cc`（来自AFLplus/AFLplus）在编译线束代码，因此，我们不能再使用`EDGES_MAP`。希望我们有[_a way_](https://github.com/AFLplusplus/AFLplusplus/blob/2e15661f184c77ac1fbb6f868c894e946cbb7f17/instrumentation/afl-compiler-rt.o.c#L270)告诉forkserver哪张地图来记录覆盖率。

你可以从forkserver的例子中看到：

```rust,ignore
//Coverage map shared between observer and executor
let mut shmem = StdShMemProvider::new().unwrap().new_shmem(MAP_SIZE).unwrap();
//let the forkserver know the shmid
shmem.write_to_env("__AFL_SHM_ID").unwrap();
let mut shmem_buf = shmem.as_mut_slice();
```

这里我们建立一个共享内存区域；`shmem`，并将其写入环境变量`__AFL_SHM_ID`。然后，被检测的二进制文件或forkerver会找到这个共享内存区域（来自上述环境变量）来记录其覆盖范围。在你的fuzzer方面，你可以将这个shmem map传递给你的`Observer`，以获得与任何`Feedback`相结合的覆盖率反馈。

`ForkserverExecutor`的另一个特点是共享内存测试案例。在正常情况下，变异的输入是通过`.cur_input`文件在forkerver和被测二进制之间传递。你可以通过用共享内存传递输入来提高你的forkserver模糊器的性能。
参见AFL++的[_documentation_](https://github.com/AFLplusplus/AFLplusplus/blob/stable/instrumentation/README.persistent_mode.md#5-shared-memory-fuzzing)或`forkserver_simple/src/program.c`中的fuzzer例子以供参考。 

这很简单，当你调用`ForkserverExecutor::new()`并将`use_shmem_testcase`设为true时，`ForkserverExecutor`会将事情设置好，你的线束就可以从`__AFL_FUZZ_TESTCASE_BUF`获取输入。

## InprocessForkExecutor

最后，我们来谈谈`InProcessForkExecutor`。
`InProcessForkExecutor`与`InprocessExecutor`只有一个区别；它在运行线束之前进行分叉，仅此而已。 
但为什么我们要这样做呢？好吧，在某些情况下，你可能会发现你的线束非常不稳定，或者你的线束对全局状态造成了破坏。在这种情况下，你想在子进程中执行线束运行之前将其分叉，这样就不会破坏事情。 
然而，我们必须照顾到共享内存，是子进程在运行线束代码，并将覆盖范围写到地图上。 
我们必须使地图在父进程和子进程之间共享，所以我们将再次使用共享内存。你应该用`pointer_maps`（用于`libafl_targes`）功能来编译你的线束，这样，我们可以有一个指针；`EDGES_MAP_PTR`，可以指向任何覆盖图。
在你的fuzzer方面，你可以分配一个共享内存区域，让`EDGES_MAP_PTR`指向你的共享内存。

```rust,ignore
let mut shmem;
unsafe{
    shmem = StdShMemProvider::new().unwrap().new_shmem(MAX_EDGES_NUM).unwrap();
}
let shmem_buf = shmem.as_mut_slice();
unsafe{
    EDGES_PTR = shmem_buf.as_ptr();
}
```

同样，你可以把这个`shmem`地图传递给你的`Observer`和`Feedback`以获得覆盖率反馈。
