## 設計線程池接口

> [ch20-03-designing-the-interface.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch20-03-designing-the-interface.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

讓我們討論一下線程池看起來怎樣。庫作者們經常會發現，當嘗試設計一些代碼時，首先編寫客戶端接口確實有助於指導代碼設計。以期望的調用方式來構建 API 代碼的結構，接著在這個結構之內實現功能，而不是先實現功能再設計公有 API。

類似於第十二章項目中使用的測試驅動開發。這裡將要使用編譯器驅動開發（Compiler Driven Development）。我們將編寫調用所期望的函數的代碼，接著依靠編譯器告訴我們接下來需要修改什麼。編譯器錯誤信息會指導我們的實現。

### 如果使用 `thread::spawn` 的代碼結構

首先，讓我們探索一下為每一個連接都創建一個線程看起來如何。這並不是最終方案，因為正如之前講到的它會潛在的分配無限的線程，不過這是一個開始。列表 20-11 展示了 `main` 的改變，它在 `for` 循環中為每一個流分配了一個新線程進行處理：

<span class="filename">Filename: src/main.rs</span>

```rust,no_run
# use std::thread;
# use std::io::prelude::*;
# use std::net::TcpListener;
# use std::net::TcpStream;
#
fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        thread::spawn(|| {
            handle_connection(stream);
        });
    }
}
# fn handle_connection(mut stream: TcpStream) {}
```

<span class="caption">列表 20-11：為每一個流新建一個線程</span>

正如第十六章講到的，`thread::spawn` 會創建一個新線程並運行閉包中的代碼。如果運行這段代碼並在兩個瀏覽器標籤頁中加載 `/sleep` 和 `/`，確實會發現 `/` 請求並沒有等待 `/sleep` 結束。不過正如之前提到的，這最終會使系統崩潰因為我們無限制的創建新線程。

### 為 `ThreadPool` 創建一個類似的接口

我們期望線程池以類似且熟悉的方式工作，以便從線程切換到線程池並不會對運行於線程池中的代碼做出較大的修改。列表 20-12 展示我們希望用來替換 `thread::spawn` 的 `ThreadPool` 結構體的假想接口：

<span class="filename">文件名: src/main.rs</span>

```rust,no_run
# use std::thread;
# use std::io::prelude::*;
# use std::net::TcpListener;
# use std::net::TcpStream;
# struct ThreadPool;
# impl ThreadPool {
#    fn new(size: u32) -> ThreadPool { ThreadPool }
#    fn execute<F>(&self, f: F)
#        where F: FnOnce() + Send + 'static {}
# }
#
fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    let pool = ThreadPool::new(4);

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }
}
# fn handle_connection(mut stream: TcpStream) {}
```

<span class="caption">列表 20-12：如何使用我們將要實現的 `ThreadPool`</span>

這裡使用 `ThreadPool::new` 來創建一個新的線程池，它有一個可配置的線程數的參數，在這裡是四。這樣在 `for` 循環中，`pool.execute` 將會以類似 `thread::spawn` 的方式工作。

### 採用編譯器驅動開發來驅動 API 的編譯

繼續並對列表 20-12 中的 *src/main.rs* 做出修改，並利用編譯器錯誤來驅動開發。下面是我們得到的第一個錯誤：

```
$ cargo check
   Compiling hello v0.1.0 (file:///projects/hello)
error[E0433]: failed to resolve. Use of undeclared type or module `ThreadPool`
  --> src\main.rs:10:16
   |
10 |     let pool = ThreadPool::new(4);
   |                ^^^^^^^^^^^^^^^ Use of undeclared type or module
   `ThreadPool`

error: aborting due to previous error
```

好的，我們需要一個 `ThreadPool`。將 `hello` crate 從二進制 crate 轉換為庫 crate 來存放 `ThreadPool` 實現，因為線程池實現與我們的 web server 的特定工作相獨立。一旦寫完了線程池庫，就可以在任何工作中使用這個功能，而不僅僅是處理網絡請求。

創建 *src/lib.rs* 文件，它包含了目前可用的最簡單的 `ThreadPool` 定義：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub struct ThreadPool;
```

接著創建一個新目錄，*src/bin*，並將二進制 crate 根文件從 *src/main.rs* 移動到 *src/bin/main.rs*。這使得庫 crate 成為 *hello* 目錄的主要 crate；不過仍然可以使用 `cargo run` 運行 *src/bin/main.rs* 二進制文件。移動了 *main.rs* 文件之後，修改文件開頭加入如下代碼來引入庫 crate 並將 `ThreadPool` 引入作用域：

<span class="filename">文件名: src/bin/main.rs</span>

```rust
extern crate hello;
use hello::ThreadPool;
```

再次嘗試運行來得到下一個需要解決的錯誤：

```
$ cargo check
   Compiling hello v0.1.0 (file:///projects/hello)
error: no associated item named `new` found for type `hello::ThreadPool` in the
current scope
  --> src\main.rs:13:16
   |
13 |     let pool = ThreadPool::new(4);
   |                ^^^^^^^^^^^^^^^
   |
```

好的，下一步是為 `ThreadPool` 創建一個叫做 `new` 的關聯函數。我們還知道 `new` 需要有一個參數可以接受 `4`，而且 `new` 應該返回 `ThreadPool` 實例。讓我們實現擁有此特徵的最小化 `new` 函數：

<span class="filename">文件夾: src/lib.rs</span>

```rust
pub struct ThreadPool;

impl ThreadPool {
    pub fn new(size: u32) -> ThreadPool {
        ThreadPool
    }
}
```

這裡的 `size` 參數是 `u32` 類型，因為我們知道為負的線程數沒有意義。`u32` 是一個很好的默認值。一旦真正實現了 `new`，我們將考慮實現需要選擇什麼類型，目前我們僅僅處理編譯器錯誤。

再次編譯檢查這段代碼：

```
$ cargo check
   Compiling hello v0.1.0 (file:///projects/hello)
warning: unused variable: `size`, #[warn(unused_variables)] on by default
 --> src/lib.rs:4:16
  |
4 |     pub fn new(size: u32) -> ThreadPool {
  |                ^^^^

error: no method named `execute` found for type `hello::ThreadPool` in the
current scope
  --> src/main.rs:18:14
   |
18 |         pool.execute(|| {
   |              ^^^^^^^
```

好的，一個警告和一個錯誤。暫時先忽略警告，錯誤是因為並沒有 `ThreadPool` 上的 `execute` 方法。讓我們來定義一個，它應該能接受一個閉包。如果你還記得第十三章，閉包作為參數時可以使用三個不同的 trait：`Fn`、`FnMut` 和 `FnOnce`。那麼應該用哪一種閉包呢？好吧，最終需要實現的類似於 `thread::spawn`；`thread::spawn` 的簽名在其參數中使用了何種 bound 呢？查看文檔會發現：

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
    where
        F: FnOnce() -> T + Send + 'static,
        T: Send + 'static
```

`F` 是這裡我們關心的參數；`T` 與返回值有關所以我們並不關心。考慮到 `spawn` 使用 `FnOnce` 作為 `F` 的 trait bound，這可能也是我們需要的，因為最終會將傳遞給 `execute` 的參數傳給 `spawn`。因為處理請求的線程只會執行閉包一次，這也進一步確認了 `FnOnce` 是我們需要的 trait。

`F` 還有 trait bound `Send` 和生命週期綁定 `'static`，這對我們的情況也是有意義的：需要 `Send` 來將閉包從一個線程轉移到另一個線程，而 `'static` 是因為並不知道線程會執行多久。讓我們編寫一個使用這些 bound 的泛型參數 `F` 的 `ThreadPool` 的 `execute` 方法：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub struct ThreadPool;
impl ThreadPool {
    // ...snip...

    pub fn execute<F>(&self, f: F)
        where
            F: FnOnce() + Send + 'static
    {

    }
}
```

`FnOnce` trait 仍然需要之後的 `()`，因為這裡的 `FnOnce` 代表一個沒有參數也沒有返回值的閉包。正如函數的定義，返回值類型可以從簽名中省略，不過即便沒有參數也需要括號。

因為我們仍在努力使接口能夠編譯，這裡增加了 `execute` 方法的最小化實現，它沒有做任何工作。再次進行檢查：

```
$ cargo check
   Compiling hello v0.1.0 (file:///projects/hello)
warning: unused variable: `size`, #[warn(unused_variables)] on by default
 --> src/lib.rs:4:16
  |
4 |     pub fn new(size: u32) -> ThreadPool {
  |                ^^^^

warning: unused variable: `f`, #[warn(unused_variables)] on by default
 --> src/lib.rs:8:30
  |
8 |     pub fn execute<F>(&self, f: F)
  |                              ^
```

現在就只有警告了！能夠編譯了！注意如果嘗試 `cargo run` 運行程序並在瀏覽器中發起請求，仍會在瀏覽器中出現在本章開始時那樣的錯誤。這個庫實際上還沒有調用傳遞給 `execute` 的閉包！

> 一個你可能聽說過的關於像 Haskell 和 Rust 這樣有嚴格編譯器的語言的說法是「如果代碼能夠編譯，它就能工作」。這是一個提醒大家的好時機，這只是一個說法和一種有時存在的感覺，實際上並不是完全正確的。我們的項目可以編譯，不過它絕對沒有做任何工作！如果構建一個真實且功能完整的項目，則需花費大量的時間來開始編寫單元測試來檢查代碼能否編譯**並且**擁有期望的行為。