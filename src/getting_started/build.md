# 构建 LibAFL

LibAFL 和大多数 Rust 项目一样，可以使用 `cargo` 从项目的根目录下构建: 

```sh
$ cargo build --release
```

请注意，`--release`在开发测试时是可选项，但使用Debug版本进行模糊测试可能会有10倍以上的性能损失，正式使用时你需要添加 `--release` 标志才能获得正常的速度。

LibAFL资源库是由多个板块组成的。
顶层的 [`Cargo.toml`](https://github.com/AFLplusplus/LibAFL/blob/main/Cargo.toml) 是将这些板块分组的工作区文件。
从根目录调用 `cargo build` 将编译工作区中的所有板块。

## 构建示例模糊器

对于有经验的 rustaceans 来说，最好的起点是阅读并改编示例模糊器。

我们将这些模糊程序放在 LibAFL 资源库的 [`./fuzzers`](https://github.com/AFLplusplus/LibAFL/tree/main/fuzzers) 目录下，该目录包含一组不属于工作区的crates。

这些示例模糊器中的每一个都使用了 LibAFL 的特定功能，有时与不同的插桩后端相结合（例如 [SanitizerCoverage](https://clang.llvm.org/docs/SanitizerCoverage.html), [Frida](https://frida.re/), ...）。

你可以使用这些 crates 作为例子，并作为具有类似功能集的自定义模糊器的骨架。
每个模糊器的目录中都有一个 `README.md` 文件，描述模糊器及其特性。

要建立一个例子的模糊器，你必须从其各自的文件夹（`fuzzers/[FUZZER_NAME]`）调用 `cargo build --release`。
