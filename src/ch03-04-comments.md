## 註釋

> [ch03-04-comments.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch03-04-comments.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

所有編程語言都力求使他們的代碼易於理解，不過有時需要提供額外的解釋。在這種情況下，程式設計師在源碼中留下記錄，或者 **註釋**（*comments*），編譯器會忽略他們不過其他閱讀代碼的人可能會用得上。

這是一個註釋的例子：

```rust
// Hello, world.
```

在 Rust 中，註釋必須以兩道斜槓開始並持續到本行的結尾。對於超過一行的註釋，需要在每一行都加上`//`，像這樣：

```rust
// So we're doing something complicated here, long enough that we need
// multiple lines of comments to do it! Whew! Hopefully, this comment will
// explain what's going on.
```

註釋也可以在放在包含代碼的行的末尾：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let lucky_number = 7; // I'm feeling lucky today.
}
```

不過你會經常看到他們被以這種格式使用，也就是位於它所解釋的代碼行的上面一行：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    // I'm feeling lucky today.
    let lucky_number = 7;
}
```

這就是註釋的全部。並沒有什麼特別複雜的。