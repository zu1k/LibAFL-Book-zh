# 在 `no_std` 环境中使用 LibAFL

可以在 `no_std` 环境中使用 LibAFL，例如自定义平台，如微控制器、内核、管理程序等等。

你可以简单地将 LibAFL 添加到你的 `Cargo.toml` 文件中:

```toml
libafl = { path = "path/to/libafl/", default-features = false}
```

然后构建你的项目，例如为 `aarch64-unknown-none` 使用:

```sh
cargo build --no-default-features --target aarch64-unknown-none
```

## 使用自定义时间戳

LibAFL 对 `no_std` 的最小输入量是一个单调增长的时间戳。
为此，在你项目的任何地方，你需要实现 `external_current_millis` 函数，它以毫秒为单位返回当前时间。

// 假设这是一个来自自定义stdlib的时钟源，你想使用它，它以秒为单位返回当前时间。

```c
int my_real_seconds(void)
{
    return *CLOCK;
}
```

而在这里我们在 Rust 中使用它，`external_current_millis` 会被LibAFL调用。
注意，它需要是 `no_mangle`，以便在链接时被 LibAFL 接受。

```rust,ignore
#[no_mangle]
pub extern "C" fn external_current_millis() -> u64 {
    unsafe { my_real_seconds()*1000 }
}
```
