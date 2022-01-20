# 设置

第一步是下载 LibAFL 和所有没有被 `cargo` 自动安装的依赖项。

> ### 命令行符号
>
> 在本章和全书中，我们展示了一些终端内容。终端输入的行都以`$`开头，但你不需要输入`$`字符；
> 它表示每个命令的开始。不以"$"开头的行通常显示的是 > 前一条命令的输出。
> 此外，PowerShell特定的例子将使用`>`而不是`$`。

虽然技术上你不需要安装 LibAFL，而是可以直接使用 crates.io 的版本，但我们还是建议下载或克隆 GitHub版本，这样你就可以得到示例模糊器、额外的实用程序和最新的补丁。

最简单的方法是使用 `git`: 

```sh
$ git clone git@github.com:AFLplusplus/LibAFL.git
```

你也可以在 类UNIX 的机器上，下载一个压缩的存档，然后用以下方法解压:

```sh
$ wget https://github.com/AFLplusplus/LibAFL/archive/main.tar.gz
$ tar xvf LibAFL-main.tar.gz
$ rm LibAFL-main.tar.gz
$ ls LibAFL-main # this is the extracted folder
```

## 安装 Clang

LibAFL 的外部依赖之一是 Clang C/C++ 编译器。
虽然大部分代码都是纯Rust语言编写，但我们仍然需要一个C语言编译器，因为稳定的Rust仍然不支持LibAFL某些部分可能需要的功能，比如弱连接以及LLVM内置链接。
对于这些部分，我们使用C语言来向我们的Rust代码库暴露缺少的功能。

此外，如果你想对 C/C++ 应用程序进行源码级模糊测试，你可能需要Clang及其插桩选项来编译被测程序。

在Linux上，你可以使用你的发行版的软件包管理器来获取Clang，但这些软件包并不总是最新的。
相反，我们建议使用来自 LLVM 的 Debian/Ubuntu 预编译包，这些包可以通过他们的 [官方仓库](https://apt.llvm.org/) 获得。

对于Microsoft Windows，你可以下载 LLVM 定期生成的 [安装包](https://llvm.org/builds/)。

尽管 Clang 是 MacOS 上的默认C编译器，但我们不鼓励使用苹果公司提供的编译器，而鼓励使用从 [Homebrew](https://brew.sh/) 安装的版本: `brew install llvm`。

另外，你也可以下载LLVM源码并自行构建，按照 [这里](https://clang.llvm.org/get_started.html) 的方法。

## 安装 Rust

如果你没有安装 Rust，你可以很容易地按照 [这里](https://www.rust-lang.org/tools/install) 描述的步骤来安装它。

请注意，Linux 发行版中的 Rust 版本可能已经过时，LibAFL 总是使用最新的**稳定**版本，可通过 `rustup upgrade` 获得。

我们建议先安装 Clang 和 LLVM，然后再安装 Rust。
