# 生成器

生成器是一个旨在从头生成输入的组件。

通常情况下，随机发生器被用来生成随机输入。

生成器在传统上较少用于反馈驱动的模糊测试，但也有例外，如Nautilus，它使用语法生成器来创建初始语料库，并使用子树生成器作为其语法突变器的突变。

在代码中，[`Generator`](https://docs.rs/libafl/0/libafl/generators/trait.Generator.html)是一个`trait`。
