# 架构

LibAFL 的架构是围绕着一些实体建立的，以允许代码重用和低成本的抽象。

最初，我们开始考虑用一种面向对象的语言来实现 LibAFL，比如 C++。当我们了解到Rust时，我们立即改变了想法，因为我们意识到，虽然Rust允许某种OOP模式，但我们可以用一种更理智的方法来构建这个库，就像[本博文](https://kyren.github.io/2018/09/14/rustconf-talk.html) 中描述的关于Rust中的游戏设计。

LibAFL 的代码重用方式是基于组件而不是子类的，但库中仍有一些OOP模式。

考虑到类似的模糊器，你可以观察到大多数时候被修改的数据结构是与测试案例和模糊器全局状态有关的。

除了之前描述的实体外，我们引入了 [`Testcase`](https://docs.rs/libafl/0.6/libafl/corpus/testcase/struct.Testcase.html) 和 [`State`](https://docs.rs/libafl/0.6/libafl/state/struct.StdState.html) 实体。测试案例是存储在语料库中的输入及其元数据的容器 (因此，在实现中，语料库存储测试案例)，状态包含在运行模糊器时演变的所有元数据，包括语料库。

在实现中，状态只包含可序列化的自有对象，它本身也是可序列化的。有些模糊器可能希望在暂停时序列化其状态，或者在进行进程内模糊处理时，在崩溃时序列化，并在新进程中反序列化，以继续模糊处理，并保留所有元数据。

此外，我们将 `actions` 的实体，如 CorpusScheduler 和 Feedbacks，归入一个共同的地方，即 [`Fuzzer'](https://docs.rs/libafl/0.6.1/libafl/fuzzer/struct.StdFuzzer.html)。
