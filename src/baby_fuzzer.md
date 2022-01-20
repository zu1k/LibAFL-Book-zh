# 一个简单的LibAFL模糊器

本章讨论了一个使用 LibAFL API 构建的极其简单的模糊器。
你将学习基本的实体，如 `State`、`Observer` 和 `Executor`。
虽然下面的章节会详细讨论 LibAFL 的组件，但在这里我们介绍基本原理。

我们将对一个简单的 Rust 函数进行模糊处理，该函数在某个条件下会出现panic。这个模糊器将是单线程的，并在崩溃后停止，就像libFuzzer通常做的那样。

你可以在 [`fuzzers/baby_fuzzer`](https://github.com/AFLplusplus/LibAFL/tree/main/fuzzers/baby_fuzzer) 中找到本教程的完整版本，作为一个模糊器的例子。

> ### 警告
>
> 这个示例模糊器对于任何现实世界的使用来说都是太天真了。
> 它的目的仅仅是为了展示库的主要组件，如果想更深入地了解如何构建一个自定义的模糊器，请直接阅读 [Tutorial chapter](./tutorial/intro.md)。

## 创建一个项目

我们使用 `cargo` 创建一个新的Rust项目，将 `LibAFL` 作为一个依赖项。

```sh
$ cargo new baby_fuzzer
$ cd baby_fuzzer
```

生成的 `Cargo.toml` 看起来像下面这样: 

```toml
[package]
name = "baby_fuzzer"
version = "0.1.0"
authors = ["Your Name <you@example.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
```

为了使用 LibAFl，我们必须在 `[dependencies]` 下增添其依赖 `libafl = { path = "path/to/libafl/" }`。
如果你愿意，你可以使用 crates.io 的 LibAFL 版本，在这种情况下，你必须使用 `libafl = "*"` 来获取最新的版本（或者将其设置为当前版本）。

由于我们要对Rust代码进行模糊处理，我们希望崩溃不会简单地导致程序退出，而是引发一个 `abort` ，然后可以被模糊器捕获。
为此，我们在 [profiles](https://doc.rust-lang.org/cargo/reference/profiles.html) 中指定 `panic = "abort"`。

除了这个设置之外，我们还为在发布模式下的编译添加了一些优化标志，最终的 `Cargo.toml` 应该类似于下面的样子: 

```toml
[package]
name = "baby_fuzzer"
version = "0.1.0"
authors = ["Your Name <you@example.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
libafl = { path = "path/to/libafl/" }

[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
lto = true
codegen-units = 1
opt-level = 3
debug = true
```

## 被测试的函数

打开 `src/main.rs`，我们有一个空的 `main` 函数。
首先，我们创建一个我们想要模糊处理的闭包。它接受一个缓冲区作为输入，如果它以 `abc` 开头，就会引起崩溃: 

```rust
extern crate libafl;
use libafl::inputs::{BytesInput, HasTargetBytes};

let mut harness = |input: &BytesInput| {
    let target = input.target_bytes();
    let buf = target.as_slice();
    if buf.len() > 0 && buf[0] == 'a' as u8 {
        if buf.len() > 1 && buf[1] == 'b' as u8 {
            if buf.len() > 2 && buf[2] == 'c' as u8 {
                panic!("=)");
            }
        }
    }
};
// To test the panic:
// let input = BytesInput::new("abc".as_bytes());
// harness(&input);
```

## 生成和运行一些测试

基于 LibAFL 的模糊测试器使用的主要组件之一是状态，这是一个在模糊测试过程中演变的数据容器。
包括所有的状态，如输入的语料库，当前的rng状态，以及测试案例和运行的潜在 Metadata。
在我们的 `main` 中，我们创建了一个基本的 State 实例，如下所示。

```rust,ignore
// create a State from scratch
let mut state = StdState::new(
    // RNG
    StdRand::with_seed(current_nanos()),
    // Corpus that will be evolved, we keep it in memory for performance
    InMemoryCorpus::new(),
    // Corpus in which we store solutions (crashes in this example),
    // on disk so the user can get them after stopping the fuzzer
    OnDiskCorpus::new(PathBuf::from("./crashes")).unwrap(),
    (),
);
```

它需要一个随机数发生器，这是模糊器状态的一部分，在这种情况下，我们使用默认的 `StdRand` ，但你可以选择一个不同的。我们用当前的纳秒数作为种子。

作为第二个参数，它需要一个实现语料库特性的实例，本例中是 `InMemoryCorpus`。语料库是由模糊器演化出的测试案例的容器，在这种情况下，我们把它全部放在内存中。

我们将在后面讨论最后一个参数。第三个参数是另一个语料库，在这种情况下，用来存储被视为模糊器 "solutions" 的测试案例。对于我们的目的，solutions是触发崩溃的输入。在这种情况下，我们想把它存储在磁盘的 `crashes` 目录下，这样我们就可以检查它。

另一个必要的组件是 `EventManager`。它处理一些事件，如在模糊处理过程中向语料库添加测试案例。对于我们的目的，我们使用最简单的一个，它只是用一个 `Monitor` 实例向用户显示这些事件的信息。

```rust,ignore
// The Monitor trait defines how the fuzzer stats are displayed to the user
let mon = SimpleMonitor::new(|s| println!("{}", s));

// The event manager handle the various events generated during the fuzzing loop
// such as the notification of the addition of a new item to the corpus
let mut mgr = SimpleEventManager::new(mon);
```

此外，我们还有 Fuzzer，一个包含一些改变状态的行动的实体。其中一个动作是使用 `CorpusScheduler` 为模糊器调度测试案例。
我们将其创建为 `QueueCorpusScheduler`，一个以先进先出方式向模糊器提供测试案例的调度器。

```rust,ignore
// A queue policy to get testcasess from the corpus
let scheduler = QueueCorpusScheduler::new();

// A fuzzer with feedbacks and a corpus scheduler
let mut fuzzer = StdFuzzer::new(scheduler, (), ());
```

最后，我们需要一个 `Executor`，它是负责运行我们被测试程序的实体。在这个例子中，我们想在进程中运行 `harness` 函数（例如，不分叉出一个子程序），因此我们使用 `InProcessExecutor`。

```rust,ignore
// Create the executor for an in-process function
let mut executor = InProcessExecutor::new(
    &mut harness,
    (),
    &mut fuzzer,
    &mut state,
    &mut mgr,
)
.expect("Failed to create the Executor");
```

它需要一个 `harness`、`state` 和 事件管理器 的引用。我们将在后面讨论第二个参数。
由于执行器期望线束返回一个 `ExitKind` 对象，我们在 `harness` 函数中添加 `ExitKind::Ok`。

现在我们有4个主要的实体，可以运行我们的测试，但我们仍然不能生成测试案例。

为此，我们使用一个生成器，`RandPrintablesGenerator`，它可以生成一串可打印的字节。

```rust,ignore
use libafl::generators::RandPrintablesGenerator;

// Generator of printable bytearrays of max size 32
let mut generator = RandPrintablesGenerator::new(32);

// Generate 8 initial inputs
state
    .generate_initial_inputs(&mut fuzzer, &mut executor, &mut generator, &mut mgr, 8)
    .expect("Failed to generate the initial corpus".into());
```

现在你可以在你的 `main.rs` 中添加必要的 `use` 指令，并编译模糊器。

```rust
extern crate libafl;

use std::path::PathBuf;
use libafl::{
    bolts::{current_nanos, rands::StdRand},
    corpus::{InMemoryCorpus, OnDiskCorpus, QueueCorpusScheduler},
    events::SimpleEventManager,
    executors::{inprocess::InProcessExecutor, ExitKind},
    fuzzer::StdFuzzer,
    generators::RandPrintablesGenerator,
    inputs::{BytesInput, HasTargetBytes},
    monitors::SimpleMonitor,
    state::StdState,
};
```

运行时，你应该看到类似的东西:

```sh
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.04s
     Running `target/debug/baby_fuzzer`
[LOG Debug]: Loaded 0 over 8 initial testcases
```

## 用反馈来进化语料库

现在你只是运行了8个随机生成的测试案例，但其中没有一个被存储在语料库中。如果你非常幸运，也许你偶然触发了崩溃，但你在 `crashes` 中没有看到任何保存的文件。

现在我们想把我们的简单模糊器变成一个基于反馈的模糊器，增加产生正确的输入来触发崩溃的机会。我们将根据达到崩溃所需的3个条件来实现一个简单的反馈。

要做到这一点，我们需要一种方法来跟踪一个条件是否被满足。为模糊器提供模糊运行属性信息的组件，即我们案例中的满足条件，是观察者。我们使用 `StdMapObserver` ，这是一个默认的观察者，它使用一个 map 来跟踪覆盖的元素。在我们的模糊器中，每个条件都被映射到这种 map 的一个条目。

我们将这样的 map 表示为一个 `static mut` 变量。
由于我们不依赖于任何插桩引擎，我们必须手动跟踪 map 中被测试函数的满足条件。

```rust
extern crate libafl;
use libafl::{
    inputs::{BytesInput, HasTargetBytes},
    executors::ExitKind,
};

// Coverage map with explicit assignments due to the lack of instrumentation
static mut SIGNALS: [u8; 16] = [0; 16];

fn signals_set(idx: usize) {
    unsafe { SIGNALS[idx] = 1 };
}

// The closure that we want to fuzz
let mut harness = |input: &BytesInput| {
    let target = input.target_bytes();
    let buf = target.as_slice();
    signals_set(0);
    if buf.len() > 0 && buf[0] == 'a' as u8 {
        signals_set(1);
        if buf.len() > 1 && buf[1] == 'b' as u8 {
            signals_set(2);
            if buf.len() > 2 && buf[2] == 'c' as u8 {
                panic!("=)");
            }
        }
    }
    ExitKind::Ok
};
```

观察者可以直接从 `SIGNALS` map 中创建，方法如下:

```rust,ignore
// Create an observation channel using the signals map
let observer = StdMapObserver::new("signals", unsafe { &mut SIGNALS });
```

观察者通常被保存在相应的执行器中，因为它们所记录的信息只对一次运行有效。然后我们必须修改我们的 `InProcessExecutor` 创建，以包括观察者，如下所示:

```rust,ignore
// Create the executor for an in-process function with just one observer
let mut executor = InProcessExecutor::new(
    &mut harness,
    tuple_list!(observer),
    &mut fuzzer,
    &mut state,
    &mut mgr,
)
.expect("Failed to create the Executor".into());
```

既然模糊器可以观察到哪个条件被满足，我们就需要一种方法，根据这种观察来评定一个输入是否有趣（即值得添加到语料库中）。这里有一个反馈的概念，反馈是状态的一部分，它提供了一种将输入及其相应的执行评为有趣的方式，在观察者中寻找信息。反馈可以在一个所谓的 `FeedbackState` 实例中保持到目前为止所看到的信息的累积状态，在我们的例子中，它保持了在以前的运行中满足的条件的集合。

我们使用 `MaxMapFeedback`，这个反馈在 `MapObserver` 的 map 上实现了新奇的搜索。基本上，如果观察者的 map 中有一个值大于迄今为止为同一条目记录的最大值，它就会将该输入评为有趣的输入，并更新其状态。

反馈也被用来决定一个输入是否是一个 "solutions"。做到这一点的反馈被称为目标反馈，当它将一个输入评为有趣时，它不会被保存到语料库中，而是被保存到解决方案中，在我们的例子中被写在 `crash` 文件夹中。我们使用 `CrashFeedback` 来告诉模糊器，如果一个输入导致程序崩溃，那就是我们的解决方案。

我们需要更新我们的状态创建，包括反馈状态和模糊器，包括反馈和目标。

```rust,ignore
extern crate libafl;
use libafl::{
    bolts::{current_nanos, rands::StdRand, tuples::tuple_list},
    corpus::{InMemoryCorpus, OnDiskCorpus},
    feedbacks::{MapFeedbackState, MaxMapFeedback, CrashFeedback},
    fuzzer::StdFuzzer,
    state::StdState,
    observers::StdMapObserver,
};

// The state of the edges feedback.
let feedback_state = MapFeedbackState::with_observer(&observer);

// Feedback to rate the interestingness of an input
let feedback = MaxMapFeedback::new(&feedback_state, &observer);

// A feedback to choose if an input is a solution or not
let objective = CrashFeedback::new();

// create a State from scratch
let mut state = StdState::new(
    // RNG
    StdRand::with_seed(current_nanos()),
    // Corpus that will be evolved, we keep it in memory for performance
    InMemoryCorpus::new(),
    // Corpus in which we store solutions (crashes in this example),
    // on disk so the user can get them after stopping the fuzzer
    OnDiskCorpus::new(PathBuf::from("./crashes")).unwrap(),
    // States of the feedbacks.
    // They are the data related to the feedbacks that you want to persist in the State.
    tuple_list!(feedback_state),
);

// ...

// A fuzzer with feedbacks and a corpus scheduler
let mut fuzzer = StdFuzzer::new(scheduler, feedback, objective);
```

## 实际的模糊处理

现在，在包括正确的 `use` 之后，我们可以运行这个程序了，但结果与之前的并没有什么不同，因为随机生成器并没有考虑到我们在语料库中保存的有趣内容。要做到这一点，我们需要插入一个 `Mutator`。

LibAFL 的另一个核心组件是状态，它是对来自语料库的单个输入所做的动作。例如，`MutationalStage` 对输入进行变异，并多次执行。

作为最后一步，我们创建了一个突变状态，它使用了一个受 AFL 的 havoc 突变器启发的突变器。

```rust,ignore
use libafl::{
    mutators::scheduled::{havoc_mutations, StdScheduledMutator},
    stages::mutational::StdMutationalStage,
    fuzzer::Fuzzer,
};

// ...

// Setup a mutational stage with a basic bytes mutator
let mutator = StdScheduledMutator::new(havoc_mutations());
let mut stages = tuple_list!(StdMutationalStage::new(mutator));

fuzzer
    .fuzz_loop(&mut stages, &mut executor, &mut state, &mut mgr)
    .expect("Error in the fuzzing loop");
```

`fuzz_loop` 将使用调度器为每个迭代向模糊器请求一个测试案例，然后它将调用状态。

加入这段代码后，我们就有了一个合适的模糊器，它可以在一秒钟内找到让函数崩溃的输入。

```text
$ cargo run
   Compiling baby_fuzzer v0.1.0 (/home/andrea/Desktop/baby_fuzzer)
    Finished dev [unoptimized + debuginfo] target(s) in 1.56s
     Running `target/debug/baby_fuzzer`
[New Testcase] clients: 1, corpus: 2, objectives: 0, executions: 1, exec/sec: 0
[LOG Debug]: Loaded 1 over 8 initial testcases
[New Testcase] clients: 1, corpus: 3, objectives: 0, executions: 804, exec/sec: 0
[New Testcase] clients: 1, corpus: 4, objectives: 0, executions: 1408, exec/sec: 0
thread 'main' panicked at '=)', src/main.rs:35:21
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
Crashed with SIGABRT
Child crashed!
[Objective] clients: 1, corpus: 4, objectives: 1, executions: 1408, exec/sec: 0
Waiting for broker...
Bye!
```

正如你所看到的，在崩溃信息之后，日志的 `objectives` 计数增加 1，你会在 `crashes/` 中找到崩溃的输入。
