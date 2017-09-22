## Cargo 自定義擴展命令

> [ch14-05-extending-cargo.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch14-05-extending-cargo.md)
> <br>
> commit 6e53771a409794d9933c2a31310d78149b7e0534

Cargo 被設計為可擴展的，通過新的子命令而無須修改 Cargo 自身。如果 `$PATH` 中有類似 `cargo-something` 的二進制文件，就可以通過 `cargo something` 來像 Cargo 子命令一樣運行它。像這樣的自定義命令也可以運行 `cargo --list` 來展示出來。能夠通過 `cargo install` 向 Cargo 安裝擴展並可以如內建 Cargo 工具那樣運行他們是 Cargo 設計上的一個非常方便的優點！

## 總結

通過 Cargo 和 crates.io 來分享代碼是使得 Rust 生態環境可以用於許多不同的任務的重要組成部分。Rust 的標準庫是小而穩定的，不過 crate 易於分享和使用，並採用一個不同語言自身的時間線來提供改進。不要羞於在 crates.io 上共享對你有用的代碼；因為它很有可能對別人也很有用！