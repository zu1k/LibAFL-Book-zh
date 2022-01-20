# 摘要

[LibAFL模糊测试库](./libafl.md)

[简介](./introduction.md)

- [入门](./getting_started/getting_started.md)
  - [设置](./getting_started/setup.md)
  - [构建](./getting_started/build.md)
  - [Crates](./getting_started/crates.md)

- [简单的模糊器实例](./baby_fuzzer.md)

- [核心概念](./core_concepts/core_concepts.md)
  - [观察者 Observer](./core_concepts/observer.md)
  - [执行器 Executor](./core_concepts/executor.md)
  - [反馈 Feedback](./core_concepts/feedback.md)
  - [输入 Input](./core_concepts/input.md)
  - [语料库 Corpus](./core_concepts/corpus.md)
  - [突变器 Mutator](./core_concepts/mutator.md)
  - [生成器 Generator](./core_concepts/generator.md)
  - [阶段 Stage](./core_concepts/stage.md)

- [设计](./design/design.md)
  - [架构](./design/architecture.md)
  - [元数据](./design/metadata.md)

- [消息传递](./message_passing/message_passing.md)
  - [派生实例](./message_passing/spawn_instances.md)
  - [配置](./message_passing/configurations.md)

- [教程](./tutorial/tutorial.md)
  - [简介](./tutorial/intro.md)

- [高级功能](./advanced_features/advanced_features.md)
  - [Concolic Tracing和混合模糊](./advanced_features/concolic/concolic.md)
  - [LibAFL在 `no_std` 环境下 (内核、管理程序...) ](./advanced_features/no_std/no_std.md)
