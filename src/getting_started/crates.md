# Crates

LibAFL 是由不同的 crate 组成的。
crate 是 Rust 的 Cargo 构建系统中的一个独立的库，你可以通过把它添加到你的项目的 `Cargo.toml` 文件中来使用，比如: 

```toml
[dependencies]
libafl = { version = "*" }
```

对于 LibAFL 来说，每个 crate 都有其独立的用途，用户可能不需要在其项目中使用所有的 crate。

按照项目根目录下的文件夹的命名惯例，它们是:

### [`libafl`](https://github.com/AFLplusplus/LibAFL/tree/main/libafl)

这是 主crate，包含了构建模糊器所需的所有组件。

这个板块有许多 Feature标志，可以启用和禁用 LibAFL 的某些方面。
这些特性可以在 [LibAFL's `Cargo.toml`](https://github.com/AFLplusplus/LibAFL/blob/main/libafl/Cargo.toml) "`[features]`"下找到，并且通常在那里有注释说明。
一些值得注意的特性是。

- `std`: 启用代码中使用Rust标准库的部分。如果没有这个标志，LibAFL是 `no_std` 兼容的，这将禁用一系列功能，但允许我们在嵌入式环境中使用LibAFL，阅读 [no_std`部分](../advanced_features/no_std/no_std.md) 了解更多细节
- `derive`: 可以使用 LibAFL 的 libafl_derive 中定义的 `derive(...)` 宏
- `rand_trait`: 允许你在需要与Rust的 [`rand` crate](https://crates.io/crates/rand) 兼容的地方使用LibAFL的非常快速（*但不安全！*）的随机数发生器
- `llmp_bind_public`: 使 LibAFL 的 LLMP 绑定到一个公共的TCP端口，其他的fuzzers节点可以通过这个端口与这个实例通信
- `introspection`: 为LibAFL添加性能统计

你可以通过在你的 `Cargo.toml` 中为 LibAFL 使用 `features = ["feature1", "feature2", ...]` 来选择特性。
在这个列表中，默认情况下，`std`、`derive` 和 `rand_trait` 已经被设置。你可以通过在你的 `Cargo.toml` 中设置 `default-features = false` 来选择禁用它们。

### libafl_sugar

sugar crate 抽离了 LibAFL API 的大部分复杂性。
它的目标不是高灵活性，而是高层次和易于使用。
它不像从每个单独的组件中缝合你的模糊器那样灵活，但允许你用最少的代码行建立一个模糊器。
要看它的运行情况，请看一下[`libfuzzer_stb_image_sugar`示例模糊器](https://github.com/AFLplusplus/LibAFL/tree/main/fuzzers/libfuzzer_stb_image_sugar)。

### libafl_derive

这是一个与 `libafl` crate 配对的 proc-macro 板块。

目前，它只是暴露了 `derive(SerdeAny)` 宏，可以用来定义 Metadata 结构，详见关于 [Metadata](../design/metadata.md) 的部分。

### libafl_targets

这个板块提供了与目标交互的代码，并对其进行检测。
为了在编译时启用和禁用功能，使用功能标志来启用和禁用这些功能。

目前，支持的标志有：

- `pcguard_edges`: 定义了 SanitizerCoverage trace-pc-guard 钩子来跟踪地图中的执行边
- `pcguard_hitcounts`: 定义了 SanitizerCoverage trace-pc-guard 钩子，以追踪地图中已执行的边和hitcounts（如AFL）
- `libfuzzer`: 提供了一个 libFuzzer 风格的兼容层
- `value_profile`: 定义了 SanitizerCoverage trace-cmp 钩子，以跟踪地图中每个比较的匹配位

### libafl_cc

这是一个提供实用程序的库，用于包装编译器和创建源码级模糊器。

目前，只有Clang编译器被支持。
为了更深入地了解它，请看一下教程和例子。

### libafl_frida

这个库将 LibAFL 与 Frida 作为插桩分析的后端连接起来。

有了这个库，你就可以对 Linux/MacOS/Windows/Android 上的目标进行覆盖率采集。

此外，它还支持 CmpLog 和 AddressSanitizer 插桩和 aarch64 的运行时。

### libafl_qemu

这个库将 LibAFL 与 QEMU 用户模式连接起来，以模糊 ELF 跨平台二进制文件。

它可以在 Linux 上工作，并且可以在没有碰撞的情况下收集边缘覆盖率!
它还支持大量的钩子和插桩选项。
