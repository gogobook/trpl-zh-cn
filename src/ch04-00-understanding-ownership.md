# 了解所有權

> [ch04-00-understanding-ownership.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch04-00-understanding-ownership.md)
> <br>
> commit 4f2dc564851dc04b271a2260c834643dfd86c724

所有權（系統）是 Rust 最獨特的功能，它使得 Rust 可以無需垃圾回收（garbage collector）就能保障內存安全。因此，理解 Rust 中所有權如何工作是十分重要的。本章我們將講到所有權以及相關功能：借用、slice 以及 Rust 如何在內存中佈局數據。