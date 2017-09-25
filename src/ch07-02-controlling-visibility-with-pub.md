## 使用 `pub` 控制可見性

> [ch07-02-controlling-visibility-with-pub.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch07-02-controlling-visibility-with-pub.md)
> <br>
> commit 0a4ed5875aeba78a81ae03ac73aeb84d2e2aca86

我們通過將 `network` 和 `network::server` 的代碼分別移動到 *src/network/mod.rs* 和 *src/network/server.rs* 文件中解決了列表 7-4 中出現的錯誤信息。現在，`cargo build` 能夠構建我們的項目，不過仍然有一些警告信息，表示 `client::connect`、`network::connect` 和`network::server::connect` 函數沒有被使用：

```text
warning: function is never used: `connect`, #[warn(dead_code)] on by default
src/client.rs:1:1
  |
1 | fn connect() {
  | ^

warning: function is never used: `connect`, #[warn(dead_code)] on by default
 --> src/network/mod.rs:1:1
  |
1 | fn connect() {
  | ^

warning: function is never used: `connect`, #[warn(dead_code)] on by default
 --> src/network/server.rs:1:1
  |
1 | fn connect() {
  | ^
```

那麼為什麼會出現這些錯誤信息呢？我們構建的是一個庫，它的函數的目的是被 **用戶** 使用，而不一定要被項目自身使用，所以不應該擔心這些 `connect` 函數是未使用的。創建他們的意義就在於被另一個項目而不是被我們自己使用。

為了理解為什麼這個程序出現了這些警告，嘗試作為另一個項目來使用這個 `connect` 庫，從外部調用他們。為此，通過創建一個包含這些代碼的 *src/main.rs* 文件，在與庫 crate 相同的目錄創建一個二進制 crate：

<span class="filename">文件名: src/main.rs</span>

```rust
extern crate communicator;

fn main() {
    communicator::client::connect();
}
```

使用 `extern crate` 指令將 `communicator` 庫 crate 引入到作用域，因為事實上我們的包現在包含 **兩個** crate。Cargo 認為 *src/main.rs* 是一個二進制 crate 的根文件，與現存的以 *src/lib.rs* 為根文件的庫 crate 相區分。這個模式在可執行項目中非常常見：大部分功能位於庫 crate 中，而二進制 crate 使用這個庫 crate。通過這種方式，其他程序也可以使用這個庫 crate，這是一個很好的關注分離（separation of concerns）。

從一個外部 crate 的視角觀察 `communicator` 庫的內部，我們創建的所有模塊都位於一個與 crate 同名的模塊內部，`communicator`。這個頂層的模塊被稱為 crate 的 **根模塊**（*root module*）。

另外注意到即便在項目的子模塊中使用外部 crate，`extern crate` 也應該位於根模塊（也就是 *src/main.rs* 或 *src/lib.rs*）。接著，在子模塊中，我們就可以像頂層模塊那樣引用外部 crate 中的項了。

我們的二進制 crate 如今正好調用了庫中 `client` 模塊的 `connect` 函數。然而，執行 `cargo build` 會在之前的警告之後出現一個錯誤：

```text
error: module `client` is private
 --> src/main.rs:4:5
  |
4 |     communicator::client::connect();
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

啊哈！這告訴了我們 `client` 模塊是私有的，這也正是那些警告的癥結所在。這也是我們第一次在 Rust 上下文中涉及到 **公有**（*public*）和 **私有**（*private*）的概念。Rust 所有代碼的默認狀態是私有的：除了自己之外別人不允許使用這些代碼。如果不在自己的項目中使用一個私有函數，因為程序自身是唯一允許使用這個函數的代碼，Rust 會警告說函數未被使用。

一旦我們指定一個像 `client::connect` 的函數為公有，不光二進制 crate 中的函數調用是允許的，函數未被使用的警告也會消失。將其標記為公有讓 Rust 知道了我們意在使函數在程序的外部被使用。現在這個可能的理論上的外部可用性使得 Rust 認為這個函數 「已經被使用」。因此。當某項被標記為公有，Rust 不再要求它在程序自身被使用並停止警告某項未被使用。

### 標記函數為公有

為了告訴 Rust 某項為公有，在想要標記為公有的項的聲明開頭加上 `pub` 關鍵字。現在我們將致力於修復 `client::connect` 未被使用的警告，以及二進制 crate 中 「模塊`client`是私有的」 的錯誤。像這樣修改 *src/lib.rs* 使 `client` 模塊公有：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub mod client;

mod network;
```

`pub` 寫在 `mod` 之前。再次嘗試構建：

```text
error: function `connect` is private
 --> src/main.rs:4:5
  |
4 |     communicator::client::connect();
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

非常好！另一個不同的錯誤！好的，不同的錯誤信息也是值得慶祝的（可能是程式設計師被黑的最慘的一次）。新錯誤表明 「函數 `connect` 是私有的」，那麼讓我們修改 *src/client.rs* 將 `client::connect` 也設為公有：

<span class="filename">文件名: src/client.rs</span>

```rust
pub fn connect() {
}
```

再一次運行 `cargo build`：

```text
warning: function is never used: `connect`, #[warn(dead_code)] on by default
 --> src/network/mod.rs:1:1
  |
1 | fn connect() {
  | ^

warning: function is never used: `connect`, #[warn(dead_code)] on by default
 --> src/network/server.rs:1:1
  |
1 | fn connect() {
  | ^
```

編譯通過了，關於 `client::connect` 未被使用的警告消失了！

未被使用的代碼並不總是意味著他們需要被設為公有的：如果你 **不** 希望這些函數成為公有 API 的一部分，未被使用的代碼警告可能是在警告你這些代碼不再需要並可以安全的刪除他們。這也可能是警告你出 bug 了，如果你剛剛不小心刪除了庫中所有這個函數的調用。

當然我們的情況是，**確實** 希望另外兩個函數也作為 crate 公有 API 的一部分，所以讓我們也將其標記為 `pub` 並去掉剩餘的警告。修改 *src/network/mod.rs* 為：

<span class="filename">文件名: src/network/mod.rs</span>

```rust
pub fn connect() {
}

mod server;
```

並編譯代碼：

```text
warning: function is never used: `connect`, #[warn(dead_code)] on by default
 --> src/network/mod.rs:1:1
  |
1 | pub fn connect() {
  | ^

warning: function is never used: `connect`, #[warn(dead_code)] on by default
 --> src/network/server.rs:1:1
  |
1 | fn connect() {
  | ^
```

恩，雖然將 `network::connect` 設為 `pub` 了我們仍然得到了一個未被使用函數的警告。這是因為模塊中的函數是公有的，不過函數所在的 `network` 模塊卻不是公有的。這回我們是自內向外修改庫文件的，而 `client::connect` 的時候是自外向內修改的。我們需要修改 *src/lib.rs* 讓 `network` 也是公有的：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub mod client;

pub mod network;
```

現在編譯的話，那個警告就消失了：

```text
warning: function is never used: `connect`, #[warn(dead_code)] on by default
 --> src/network/server.rs:1:1
  |
1 | fn connect() {
  | ^
```

只剩一個警告了！嘗試自食其力修改它吧！

### 私有性規則

總的來說，有如下項的可見性規則：

1. 如果一個項是公有的，它能被任何父模塊訪問
2. 如果一個項是私有的，它能被其直接父模塊及其任何子模塊訪問

### 私有性示例

讓我們看看更多例子作為練習。創建一個新的庫項目並在新項目的 *src/lib.rs* 輸入列表 7-5 中的代碼：

<span class="filename">文件名: src/lib.rs</span>

```rust
mod outermost {
    pub fn middle_function() {}

    fn middle_secret_function() {}

    mod inside {
        pub fn inner_function() {}

        fn secret_function() {}
    }
}

fn try_me() {
    outermost::middle_function();
    outermost::middle_secret_function();
    outermost::inside::inner_function();
    outermost::inside::secret_function();
}
```

<span class="caption">列表 7-5：私有和公有函數的例子，其中部分是不正確的</span>

在嘗試編譯這些代碼之前，猜測一下 `try_me` 函數的哪一行會出錯。接著編譯項目來看看是否猜對了，然後繼續閱讀後面關於錯誤的討論！

#### 檢查錯誤

`try_me` 函數位於項目的根模塊。叫做 `outermost` 的模塊是私有的，不過第二條私有性規則說明` try_me` 函數允許訪問 `outermost` 模塊，因為 `outermost` 位於當前（根）模塊，`try_me` 也是。

`outermost::middle_function` 的調用是正確的。因為 `middle_function` 是公有的，而 `try_me` 通過其父模塊 `outermost` 訪問 `middle_function`。根據上一段的規則我們可以確定這個模塊是可訪問的。

`outermost::middle_secret_function` 的調用會造成一個編譯錯誤。`middle_secret_function` 是私有的，所以第二條（私有性）規則生效了。根模塊既不是 `middle_secret_function` 的當前模塊（`outermost`是），也不是 `middle_secret_function` 當前模塊的子模塊。

叫做 `inside` 的模塊是私有的且沒有子模塊，所以它只能被當前模塊 `outermost` 訪問。這意味著 `try_me` 函數不允許調用 `outermost::inside::inner_function` 或 `outermost::inside::secret_function` 中的任何一個。

#### 修改錯誤

這裡有一些嘗試修復錯誤的代碼修改意見。在你嘗試他們之前，猜測一下他們哪個能修復錯誤，接著編譯查看你是否猜對了，並結合私有性規則理解為什麼。

* 如果 `inside` 模塊是公有的？
* 如果 `outermost` 是公有的而 `inside` 是私有的？
* 如果在 `inner_function` 函數體中調用 `::outermost::middle_secret_function()`？（開頭的兩個冒號意味著從根模塊開始引用模塊。）

請隨意設計更多的實驗並嘗試理解他們！

接下來，讓我們討論一下使用 `use` 關鍵字將模塊項目引入作用域。