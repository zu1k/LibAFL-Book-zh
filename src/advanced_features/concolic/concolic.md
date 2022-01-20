# 并行追踪和混合模糊测试

LibAFL支持基于 [SymCC](https://github.com/eurecom-s3/symcc) 仪表编译器的协程跟踪。

对于那些没有经验的人来说，下面将尝试用一个例子从头开始描述协程跟踪。
然后，我们将讨论SymCC和LibAFL协程跟踪的关系。
最后，我们将通过使用LibAFL构建一个基本的混合模糊器。

## 同期追踪的例子

假设你想对以下程序进行模糊处理。

```rust
fn target(input: &[u8]) -> i32 {
    match &input {
        // fictitious crashing input
        &[1, 3, 3, 7] => 1337,
        // standard error handling code
        &[] => -1,
        // representative of normal execution
        _ => 0 
    }
}
```

一个简单的覆盖率最大化的模糊器在某种程度上随机地产生新的输入，将很难找到触发虚构的崩溃输入的一个输入。
许多技术被提出来，以使模糊处理不那么随机，而是更直接地试图改变输入，以翻转特定的分支，例如参与崩溃上述程序的分支。

并行追踪允许我们以 **分析的方式而不是** 随机的方式 (即猜测) 构建一个输入，以锻炼程序中的一个新路径 (比如例子中的崩溃路径) 。
原则上，协程跟踪的工作方式是观察程序执行过程中所有依赖输入的执行指令。
为了理解这一点，我们将以上述程序为例进行说明。

首先，我们将程序简化为简单的if-then-else-statements。

```rust
fn target(input: &[u8]) -> i32 {
    if input.len() == 4 {
        if input[0] == 1 {
            if input[1] == 3 {
                if input[2] == 3 {
                    if input[3] == 7 {
                        return 1337;
                    } else {
                        return 0;
                    }
                } else {
                    return 0;
                }
            } else {
                return 0;
            }
        } else {
            return 0;
        }
    } else {
        if input.len() == 0 {
            return -1;
        } else {
            return 0;
        }
    }
}
```
Next, we'll trace the program on the input `[]`.
The trace would look like this:
```rust,ignore
Branch { // if input.len() == 4
    condition: Equals { 
        left: Variable { name: "input_len" }, 
        right: Integer { value: 4 } 
    }, 
    taken: false // This condition turned out to be false...
}
Branch { // if input.len() == 0
    condition: Equals { 
        left: Variable { name: "input_len" }, 
        right: Integer { value: 0 } 
    }, 
    taken: true // This condition turned out to be true!
}
```

利用这个跟踪，我们可以很容易地推断出，我们可以通过一个长度为4的输入或者一个非零长度的输入来迫使程序采取不同的路径。
我们通过否定每个分支条件并分析解决所产生的 "表达式 "来做到这一点。
事实上，我们可以为任何计算创建这些表达式，并把它们交给 [SMT](https://en.wikipedia.org/wiki/Satisfiability_modulo_theories)-Solver，它将生成一个满足表达式的输入 (只要这种输入存在) 。

在混合模糊计算中，我们将这种追踪+求解的方法与更传统的模糊计算技术相结合。

## LibAFL、SymCC和SymQEMU中的协程跟踪

LibAFL中的协程跟踪支持是通过SymCC实现的。
SymCC是clang的一个编译器插件，可以用来替代普通的C或C++编译器。
SymCC将对编译后的代码进行回调，使之成为一个可以由用户提供的运行时。
这些回调允许运行时构建一个类似于前面例子的跟踪。

### SymCC和它的运行时

SymCC有两个运行时:

 * 一个 "简单的 "运行时，它试图用 [Z3](https://github.com/Z3Prover/z3/wiki) 来解决它遇到的任何分支，以及
 * 一个基于 [QSym](https://github.com/sslab-gatech/qsym) 的运行时，它对表达式进行了更多的过滤，也使用Z3进行求解。

然而，与LibAFL的集成需要你使用 [`symcc_runtime`](https://docs.rs/symcc_runtime/0.1/symcc_runtime) crate进行 **BYORT** (_bring your own runtime_) 。
这个工具箱允许你轻松地从内置的构建模块中建立一个自定义的运行时，或者以完全的灵活性创建全新的运行时。
查看 `symcc_runtime` 文档，了解更多关于如何建立你自己的运行时的信息。

### SymQEMU

[SymQEMU](https://github.com/eurecom-s3/symqemu) 是SymCC的一个兄弟项目。
它不是在编译时对目标进行检测，而是通过动态二进制翻译插入检测，建立在 [`QEMU`](https://www.qemu.org) 仿真栈之上。
这意味着使用SymQEMU，任何 (x86) 二进制文件都可以被追踪，而不需要提前建立插桩。
`symcc_runtime` 工具箱支持这种使用情况，用 `symcc_runtime` 构建的运行时也可用于SymQEMU。

## LibAFL中的混合型模糊处理

LibAFL资源库中包含了一个[混合模糊器实例](https://github.com/AFLplusplus/LibAFL/tree/main/fuzzers/libfuzzer_stb_image_concolic)。

使用LibAFL构建一个混合型模糊器主要有三个步骤。
1. 建立一个运行时间。
2. 选择一个工具化的方法和
3. 构建模糊器。

请注意，这些步骤的顺序是很重要的。
例如，在用SymCC进行插桩分析之前，我们需要先准备好运行时间。

### 建立一个运行时
使用`symcc_runtime`板块可以很容易地构建一个自定义运行时。
注意，自定义运行时是一个单独的共享对象文件，这意味着我们需要一个单独的crate用于我们的运行时。
请查看[混合模糊器的运行时间示例](https://github.com/AFLplusplus/LibAFL/tree/main/fuzzers/libfuzzer_stb_image_concolic/runtime)和[`symcc_runtime` docs](https://docs.rs/symcc_runtime/0.1/symcc_runtime)以获得灵感。

### 工具化

在LibAFL中，有两种主要的工具化方法来使用协程跟踪。

* 使用**编译时**插桩化的目标与**SymCC**。
这只有在目标的源代码可用，并且目标很容易使用SymCC编译器包装器构建的情况下才有效。
* 使用**SymQEMU**在**运行时动态地检测目标。
这避免了一个单独的插桩化目标与协程跟踪插桩化，而且不需要源代码。
然而，应该注意的是，生成的表达式的 "质量 "可能会大大降低，而且SymQEMU通常比SymCC生成的表达式要多得多，而且明显更曲折。
因此，建议尽可能使用SymCC而不是SymQEMU。

#### 使用SymCC

在使用SymCC进行模糊测试之前，需要对目标进行检测。
具体如何做并不重要。
然而，SymCC编译器需要知道它应该检测的运行时间的位置。
这可以通过将 `SYMCC_RUNTIME_DIR` 环境变量设置为包含运行时的目录来实现 (通常是你的运行时板块的 `target/(debug|release)` 文件夹) 。

混合模糊器的例子在其 [`build.rs`构建脚本](https://github.com/AFLplusplus/LibAFL/blob/main/fuzzers/libfuzzer_stb_image_concolic/fuzzer/build.rs#L50) 中检测目标。
它通过克隆和构建SymCC的副本来实现，然后使用这个版本来检测目标。
[`symcc_libafl` crate](https://docs.rs/symcc_libafl) 包含用于克隆和构建SymCC的辅助函数。

在尝试构建SymCC之前，请确保你满足SymCC的 [构建要求](https://github.com/eurecom-s3/symcc#readme)。

#### 使用SymQEMU

根据SymQEMU的 [构建说明](https://github.com/eurecom-s3/symqemu#readme) 来构建它。
默认情况下，SymQEMU会在一个同级目录下寻找运行时。
由于我们没有运行时，我们需要让它知道你的运行时的路径，将 `--symcc-build` 脚本的参数设置为你的运行时的路径。

### 构建模糊器

无论采用哪种方法，现在模糊器和被检测目标之间的接口应该是一致的。
使用SymCC和SymQEMU的唯一区别应该是代表目标的二进制文件。
在SymCC的情况下，它将是用插桩构建的二进制文件，而在SymQEMU的情况下，它将是模拟器的二进制文件 (例如: `x86_64-linux-user/symqemu-x86_64`) ，后面是你的非插桩化的目标二进制文件和论据。

你可以使用 [`CommandExecutor`](https://docs.rs/libafl/0.6.0/libafl/executors/command/struct.CommandExecutor.html) 来执行你的目标 ([example](https://github.com/AFLplusplus/LibAFL/blob/main/fuzzers/libfuzzer_stb_image_concolic/fuzzer/src/main.rs#L244)) 。
在配置命令时，如果你的目标从文件中读取输入 (而不是标准输入) ，请确保传递 `SYMCC_INPUT_FILE` 环境变量的输入文件路径。

#### 序列化和解算

虽然完全可以建立一个自定义的运行时，在目标进程的背景下执行混合模糊的求解步骤，但LibAFL协程跟踪支持的预期用途是使用 [`TracingRuntime`](https://docs.rs/symcc_runtime/0.1/symcc_runtime/tracing/struct.TracingRuntime.html) 对 (过滤和预处理的) 分支条件进行序列化。
这个序列化的表述可以在模糊程序中被反序列化，以便使用 [`ConcolicObserver`](https://docs.rs/libafl/0.6.0/libafl/observers/concolic/struct.ConcolicObserver.html) 包裹在 [`ConcolicTracingStage`](https://docs.rs/libafl/0.6.0/libafl/stages/concolic/struct.ConcolicTracingStage.html) 中进行解决，它将在每个 [`TestCase`](https://docs.rs/libafl/0.6.0/libafl/corpus/testcase/struct.Testcase.html) 中附加一个 [`ConcolicMetadata`](https://docs.rs/libafl/0.6.0/libafl/observers/concolic/struct.ConcolicMetadata.html)。

`ConcolicMetadata'可以用来重放协程跟踪，并使用SMT-Solver进行解决。
然而，大多数涉及协程跟踪的用例都需要围绕他们想要解决的分支定义一些策略。
[`SimpleConcolicMutationalStage`](https://docs.rs/libafl/0.6.0//libafl/stages/concolic/struct.SimpleConcolicMutationalStage.html) 可用于测试目的。
它将尝试解决所有分支，就像SymCC的原始简单后端一样，使用Z3。

### 示例

这个例子说明了如何使用 [`ConcolicTracingStage`和`SimpleConcolicMutationalStage`](https://github.com/AFLplusplus/LibAFL/blob/main/fuzzers/libfuzzer_stb_image_concolic/fuzzer/src/main.rs#L203) 来建立一个基本的混合模糊器。

