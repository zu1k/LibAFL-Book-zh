# 元数据

LibAFL中的元数据是一个自包含的结构，持有与状态或测试案例相关的数据。

在代码方面，元数据可以被定义为一个在SerdeAny寄存器中注册的Rust结构。

```rust
extern crate libafl;
extern crate serde;

use libafl::SerdeAny;
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize, SerdeAny)]
pub struct MyMetadata {
    //...
}
```

这个结构必须是静态的，所以它不能持有对借用对象的引用。

作为`libafl_derive`中的proc-macro，用户可以使用`libafl::impl_serdeany!(MyMetadata);`来替代`derive(SerdeAny)`。

## 用法

元数据对象主要用于[`SerdeAnyMap`](https://docs.rs/libafl/0.5.0/libafl/bolts/serdeany/serdeany_registry/struct.SerdeAnyMap.html)和[`NamedSerdeAnyMap`](https://docs.rs/libafl/0.5.0/libafl/bolts/serdeany/serdeany_registry/struct.NamedSerdeAnyMap.html)中。

通过这些地图，用户可以按类型（和名称）检索实例。在内部，这些实例被存储为SerdeAny特质对象。

想拥有一套元数据的结构必须实现[`HasMetadata`](https://docs.rs/libafl/0.5.0/libafl/state/trait.HasMetadata.html)特质。

默认情况下，Testcase和State实现了它，并持有一个SerdeAnyMap测试案例。

## (De)Serialization

我们对存储状态的元数据感兴趣，以便在崩溃或模糊器停止的情况下不会丢失它们。要做到这一点，它们必须使用Serde进行序列化和非序列化。

由于元数据是作为特质对象存储在SerdeAnyMap中的，所以默认情况下它们不能用Serde进行反序列化。

为了解决这个问题，在LibAFL中，每个SerdeAny结构都必须在一个全局注册表中注册，该注册表可以跟踪类型并允许对注册的类型进行（反）序列化。

通常情况下，`impl_serdeany`宏为用户创建一个构造函数来填充注册表。然而，当在no_std模式下使用LibAFL时，这个操作必须在`main`函数的任何其他操作之前手动进行。

要做到这一点，开发者需要知道模糊器内部使用的每个元数据类型，并在`main`的开头为每个元数据调用`RegistryBuilder::register::<MyMetadata>()`。
