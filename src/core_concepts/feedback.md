# Feedback

反馈是一个实体，它将被测程序的执行结果分类为有趣或不有趣。
通常情况下，如果一个执行是有趣的，那么用于输入目标程序的相应输入就会被添加到一个语料库中。

大多数时候，反馈的概念与观察者有很深的联系，但它们是不同的概念。

反馈，在大多数情况下，处理一个或多个观察者报告的信息，以决定执行是否有趣。
有趣 "的概念是抽象的，但通常它与新颖性搜索有关（即有趣的输入是那些达到控制流图中以前未见过的边缘的输入）。

举个例子，给定一个报告所有内存分配大小的观察者，可以用一个最大化反馈来最大化这些大小，以运动内存消耗方面的病态输入。

In terms of code, the library offers the [`Feedback`](https://docs.rs/libafl/0/libafl/feedbacks/trait.Feedback.html) and the [`FeedbackState`](https://docs.rs/libafl/0/libafl/feedbacks/trait.FeedbackState.html) traits.
第一个用于实现漏斗，在给定最后一次执行的观察者的状态时，告诉他们这次执行是否有趣。第二个是与 "反馈 "联系在一起的，它是反馈希望在模糊器的状态中坚持的数据的状态，例如，在基于边缘覆盖率的反馈的情况下，持有到目前为止看到的所有边缘的累积图。

多个反馈可以结合成布尔公式，例如，如果一个执行触发了新的代码路径，或者执行时间比平均执行时间短，就可以认为它是有趣的。[`feedback_or`](https://docs.rs/libafl/0/libafl/macro.feedback_or.html).

TODO目标反馈和快速反馈逻辑运算符
