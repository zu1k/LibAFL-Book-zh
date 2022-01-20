# 突变器 Mutator

突变器(Mutator) 是一个接受一个或多个输入并生成一个新的派生输入的实体。

突变器可以被组成，它们通常与特定的输入类型相联系。

例如，可以有一个 Mutator 在输入上应用不止一种类型的突变。考虑一个字节流的通用突变器，比特翻转只是可能的突变之一，但不是唯一的突变，还有，例如，随机替换一个字节的块的拷贝。

在LibAFL中，[`Mutator`](https://docs.rs/libafl/0/libafl/mutators/trait.Mutator.html)是一个 `trait`。
