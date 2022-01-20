# Crates

LibAFL是由不同的crate组成的。
crate是Rust的Cargo构建系统中的一个单独的库，你可以通过把它添加到你的项目的`Cargo.toml`中来使用，比如：

```toml
[dependencies]
libafl = { version = "*" }
```

对于 LibAFL 来说，每个 crate 都有其独立的用途，用户可能不需要在其项目中使用所有的 crate。
按照项目根目录下的文件夹的命名惯例，它们是。

### [`libafl`](https://github.com/AFLplusplus/LibAFL/tree/main/libafl)

这是个主箱，包含了构建模糊器所需的所有组件。

这个板块有许多特性标志，可以启用和禁用LibAFL的某些方面。
这些特性可以在 [LibAFL's `Cargo.toml`](https://github.com/AFLplusplus/LibAFL/blob/main/libafl/Cargo.toml) "`[特性]`"下找到，并且通常在那里有注释说明。
一些值得注意的特性是。

- `std`启用代码中使用Rust标准库的部分。如果没有这个标志，LibAFL是`no_std`兼容的。这将禁用一系列功能，但允许我们在嵌入式环境中使用LibAFL，阅读[no_std`部分](../advanced_features/no_std/no_std.md)了解更多细节。
- `derive`可以使用libafl的libafl_derive中定义的`derive(...)`宏。
- `rand_trait`允许你在需要与Rust的[`rand` crate](https://crates.io/crates/rand)兼容的地方使用LibAFL的非常快速（*但不安全！*）的随机数发生器。
- `llmp_bind_public`使LibAFL的LLMP绑定到一个公共的TCP端口，其他的fuzzers节点可以通过这个端口与这个实例通信。
- `introspection`为LibAFL添加性能统计。

你可以通过在你的`Cargo.toml`中为LibAFL使用`features = ["feature1", "feature2", ...]`来选择特性。
在这个列表中，默认情况下，`std`、`derive`和`rand_trait`已经被设置。
你可以通过在你的`Cargo.toml`中设置`default-features = false`来选择禁用它们。

### libafl_sugar

sugar crate 抽离了 LibAFL API 的大部分复杂性。
它的目标不是高灵活性，而是高层次和易于使用。
它不像从每个单独的组件中缝合你的模糊器那样灵活，但允许你用最少的代码行建立一个模糊器。
要看它的运行情况，请看一下[`libfuzzer_stb_image_sugar`示例模糊器](https://github.com/AFLplusplus/LibAFL/tree/main/fuzzers/libfuzzer_stb_image_sugar)。

### libafl_derive

这是一个与`libafl`板块配对的proc-macro板块。

目前，它只是暴露了`derive(SerdeAny)`宏，可以用来定义Metadata结构，详见关于[Metadata](../design/metadata.md)的部分。

### libafl_targets

这个板块提供了与目标交互的代码，并对其进行检测。
为了在编译时启用和禁用功能，使用功能标志来启用和禁用这些功能。

目前，支持的标志有

- `pcguard_edges`定义了SanitizerCoverage trace-pc-guard钩子来跟踪地图中的执行边。
- `pcguard_hitcounts`定义了SanitizerCoverage追踪-pc-guard的钩子，以追踪地图中已执行的边和hitcounts（如AFL）。
- `libfuzzer`提供了一个与libFuzzer风格线束的兼容层。
- `value_profile`定义了SanitizerCoverage trace-cmp钩子，以跟踪地图中每个比较的匹配位。

### libafl_cc

这是一个提供实用程序的库，用于包装编译器和创建源码级模糊器。

目前，只有Clang编译器被支持。
为了更深入地了解它，请看一下教程和例子。

### libafl_frida

这个库将LibAFL与Frida作为插桩分析的后端连接起来。

有了这个库，你就可以对Linux/MacOS/Windows/Android上的目标进行覆盖率采集。

此外，它还支持CmpLog和AddressSanitizer插桩和aarch64的运行时间。

### libafl_qemu

这个库将LibAFL与QEMU用户模式连接起来，以模糊ELF跨平台二进制文件。

它可以在Linux上工作，并且可以在没有碰撞的情况下收集边缘覆盖率!
它还支持大量的钩子和插桩选项。
