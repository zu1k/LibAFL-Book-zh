# 构建LibAFL

LibAFL和大多数Rust项目一样，可以使用`cargo`从项目的根目录下构建。

```sh
$ cargo build --release
```

请注意，`--release`标志在开发中是可选的，但你需要添加它才能以适当的速度进行模糊测试。
对于Debug构建来说，10倍以上的减速并不罕见。

LibAFL资源库是由多个板块组成的。
顶层的`Cargo.toml`](https://github.com/AFLplusplus/LibAFL/blob/main/Cargo.toml)是将这些板块分组的工作区文件。
从根目录调用`cargo build`将编译工作区中的所有板块。

## 构建示例模糊器

对于有经验的rustaceans来说，最好的起点是阅读并改编示例模糊器。

我们将这些模糊程序放在LibAFL资源库的[`./fuzzers`](https://github.com/AFLplusplus/LibAFL/tree/main/fuzzers)目录下。
该目录包含一组不属于工作区的板条箱。

这些示例模糊器中的每一个都使用了 LibAFL 的特定功能，有时与不同的插桩后端相结合（例如 [SanitizerCoverage](https://clang.llvm.org/docs/SanitizerCoverage.html), [Frida](https://frida.re/), ...）。

你可以使用这些箱子作为例子，并作为具有类似功能集的自定义模糊器的骨架。
每个模糊器的目录中都有一个`README.md`文件，描述模糊器及其特性。

要建立一个例子的模糊器，你必须从其各自的文件夹（`fuzzers/[FUZZER_NAME]`）调用`cargo build --release`。
