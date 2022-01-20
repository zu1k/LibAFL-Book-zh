# 观察者

观察者，或称观察通道，是一个实体，它向模糊器提供在被测程序执行期间观察到的信息。

观察者所包含的信息在不同的执行过程中是不被保留的。

例如，在执行过程中填充的覆盖率共享图，以报告由AFL和HonggFuzz等模糊器使用的执行边缘，可以被认为是一个观察通道。
这种信息在不同的运行中并不保留，它是对程序的动态属性的观察。

In terms of code, in the library this entity is described by the [`Observer`](https://docs.rs/libafl/0/libafl/observers/trait.Observer.html) trait.

除了保存与目标的最后一次执行有关的易失性数据外，实现这一特性的结构可以定义一些执行钩子，在每个模糊情况前后执行。在这个钩子中，观察者可以修改模糊器的状态。
