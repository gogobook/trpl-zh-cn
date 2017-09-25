## `mod` 和文件系統

> [ch07-01-mod-and-the-filesystem.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch07-01-mod-and-the-filesystem.md)
> <br>
> commit c6a9e77a1b1ed367e0a6d5dcd222589ad392a8ac

我們將通過使用 Cargo 創建一個新項目來開始我們的模組之旅，不過不再創建一個二進制 crate，而是創建一個庫(譯註:或稱為"套件") crate：一個其他人可以作為依賴導入的項目。第二章猜猜看遊戲中作為依賴使用的 `rand` 就是這樣的 crate。

我們將創建一個提供一些通用網絡功能的項目的骨架結構；我們將專注於模組和函數的組織，而不擔心函數體中的具體代碼。這個項目叫做 `communicator`。Cargo 預設會創建一個庫 crate 除非指定其他項目類型，所以如果不像一直以來那樣加入 `--bin` 參數則項目將會是一個庫：

```text
$ cargo new communicator
$ cd communicator
```

注意 Cargo 生成了 *src/lib.rs* 而不是 *src/main.rs*。在 *src/lib.rs* 中我們會找到這些：

<span class="filename">文件名: src/lib.rs</span>

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
    }
}
```

Cargo 創建了一個空的測試來幫助我們開始庫項目，不像使用 `--bin` 參數那樣創建一個 「Hello, world!」 二進制項目。在本章之後的 「使用 `super` 訪問父模組」 部分會介紹 `#[]` 和 `mod tests` 語法，目前只需確保他們位於 *src/lib.rs* 底部即可。

因為沒有 *src/main.rs* 文件，所以沒有可供 Cargo 的 `cargo run` 執行的東西。因此，我們將使用 `cargo build` 命令只是編譯庫 crate 的代碼。

我們將學習根據編寫代碼的意圖來選擇不同的織庫項目代碼組織來適應多種場景。

### 模組定義

對於 `communicator` 網絡庫，首先要定義一個叫做 `network` 的模組，它包含一個叫做 `connect` 的函數定義。Rust 中所有模組的定義以關鍵字 `mod` 開始。在 *src/lib.rs* 文件的開頭在測試代碼的上面增加這些代碼：

<span class="filename">文件名: src/lib.rs</span>

```rust
mod network {
    fn connect() {
    }
}
```

`mod` 關鍵字的後面是模組的名字，`network`，接著是位於大括號中的代碼塊。代碼塊中的一切都位於 `network` 命名空間中。在這個例子中，只有一個函數，`connect`。如果想要在 `network` 模組外面的代碼中調用這個函數，需要指定模組名並使用命名空間語法 `::`，像這樣：`network::connect()`，而不是只是 `connect()`。

也可以在 *src/lib.rs* 文件中同時存在多個模組。例如，再擁有一個 `client` 模組，它也有一個叫做 `connect` 的函數，如列表 7-1 中所示那樣增加這個模組：

<span class="filename">文件名: src/lib.rs</span>

```rust
mod network {
    fn connect() {
    }
}

mod client {
    fn connect() {
    }
}
```

<span class="caption">列表 7-1：`network` 模組和 `client` 一同定義於 *src/lib.rs*</span>

現在我們有了 `network::connect` 函數和 `client::connect` 函數。他們可能有著完全不同的功能，同時他們也不會彼此衝突，因為他們位於不同的模組。

在這個例子中，因為我們構建的是一個庫，作為庫入口點的文件是 *src/lib.rs*。然而，對於創建模組來說，*src/lib.rs* 並沒有什麼特殊意義。也可以在二進制 crate 的 *src/main.rs* 中創建模組，正如在庫 crate 的 *src/lib.rs* 創建模組一樣。事實上，也可以將模組放入其他模組中。這有助於隨著模組的增長，將相關的功能組織在一起並又保持各自獨立。如何選擇組織代碼依賴於如何考慮代碼不同部分之間的關係。例如，對於庫的用戶來說，`client` 模組和它的函數 `connect` 可能放在 `network` 命名空間裡顯得更有道理，如列表 7-2 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
mod network {
    fn connect() {
    }

    mod client {
        fn connect() {
        }
    }
}
```

<span class="caption">列表 7-2：將 `client` 模組移動到 `network` 模組中</span>

在 *src/lib.rs* 文件中，將現有的 `mod network` 和 `mod client` 的定義替換為列表 7-2 中的定義，這裡將 `client` 模組作為 `network` 的一個內部模組。現在我們有了 `network::connect` 和 `network::client::connect` 函數：同樣的，這兩個 `connect` 函數也不相衝突，因為他們在不同的命名空間中。

這樣，模組之間形成了一個層次結構。*src/lib.rs* 的內容位於最頂層，而其子模組位於較低的層次。如下是列表 7-1 中的例子以層次的方式考慮的結構：

```text
communicator
 ├── network
 └── client
```

而這是列表 7-2 中例子的的層次結構：

```text
communicator
 └── network
     └── client
```

可以看到列表 7-2 中，`client` 是 `network` 的子模組，而不是它的同級模組。更為複雜的項目可以有很多的模組，所以他們需要符合邏輯地組合在一起以便記錄他們。在項目中 「符合邏輯」 的意義全憑你的理解和庫的用戶對你項目領域的認識。利用我們這裡講到的技術來創建同級模組和嵌套的模組，總有一個會是你會喜歡的結構。

### 將模組移動到其他文件

位於層級結構中的模組，非常類似計算機領域的另一個我們非常熟悉的結構：文件系統！我們可以利用 Rust 的模組系統連同多個文件一起分解 Rust 項目，這樣就不會是所有的內容都落到 *src/lib.rs* 或 *src/main.rs* 中了。為了舉例，我們將從列表 7-3 中的代碼開始：

<span class="filename">文件名: src/lib.rs</span>

```rust
mod client {
    fn connect() {
    }
}

mod network {
    fn connect() {
    }

    mod server {
        fn connect() {
        }
    }
}
```

<span class="caption">列表 7-3：三個模組，`client`、`network` 和 `network::server`，他們都定義於 *src/lib.rs*</span>

*src/lib.rs* 文件有如下層次結構：

```text
communicator
 ├── client
 └── network
     └── server
```

如果這些模組有很多函數，而這些函數又很長，將難以在文件中尋找我們需要的代碼。因為這些函數被嵌套進一個或多個模組中，同時函數中的代碼也會開始變長。這就有充分的理由將`client`、`network` 和 `server`每一個模組從 *src/lib.rs* 抽出並放入他們自己的文件中。

首先，將 `client` 模組的代碼替換為只有 `client` 模組聲明，這樣 *src/lib.rs* 看起來應該像這樣：

<span class="filename">文件名: src/lib.rs</span>

```rust
mod client;

mod network {
    fn connect() {
    }

    mod server {
        fn connect() {
        }
    }
}
```

這裡我們仍然 **聲明** 了 `client` 模組，不過將代碼塊替換為了分號，這告訴了 Rust 在 `client` 模組的作用域中尋找另一個定義代碼的位置。換句話說，`mod client;` 行意味著：

```rust
mod client {
    // contents of client.rs
}
```

那麼現在需要創建對應模組名的外部文件。在 *src/* 目錄創建一個 *client.rs* 文件，接著打開它並輸入如下內容，它是上一步被去掉的 `client` 模組中的 `connect` 函數：

<span class="filename">Filename: src/client.rs</span>

```rust
fn connect() {
}
```

注意這個文件中並不需要一個 `mod` 聲明；因為已經在 *src/lib.rs* 中已經使用 `mod` 聲明了 `client` 模組。這個文件僅僅提供 `client` 模組的 **內容**。如果在這裡加上一個 `mod client`，那麼就等於給 `client` 模組增加了一個叫做 `client` 的子模組了！

Rust 預設只知道 *src/lib.rs* 中的內容。如果想要對項目加入更多文件，我們需要在 *src/lib.rs* 中告訴 Rust 去尋找其他文件；這就是為什麼 `mod client` 需要被定義在 *src/lib.rs* 而不能在 *src/client.rs* 的原因。

現在，一切應該能成功編譯，雖然會有一些警告。記住使用 `cargo build` 而不是 `cargo run` 因為這是一個庫 crate 而不是二進制 crate：

```text
$ cargo build
   Compiling communicator v0.1.0 (file:///projects/communicator)

warning: function is never used: `connect`, #[warn(dead_code)] on by default
 --> src/client.rs:1:1
  |
1 | fn connect() {
  | ^

warning: function is never used: `connect`, #[warn(dead_code)] on by default
 --> src/lib.rs:4:5
  |
4 |     fn connect() {
  |     ^

warning: function is never used: `connect`, #[warn(dead_code)] on by default
 --> src/lib.rs:8:9
  |
8 |         fn connect() {
  |         ^
```

這些警告提醒我們有從未被使用的函數。目前不用擔心這些警告；在本章後面的 「使用 `pub` 控制可見性」 部分會解決他們。好消息是，他們僅僅是警告；我們的項目能夠被成功編譯。

下面使用相同的模式將 `network` 模組提取到自己的文件中。刪除 *src/lib.rs* 中 `network` 模組的內容並在聲明後加上一個分號，像這樣：

<span class="filename">Filename: src/lib.rs</span>

```rust
mod client;

mod network;
```

接著新建 *src/network.rs* 文件並輸入如下內容：

<span class="filename">Filename: src/network.rs</span>

```rust
fn connect() {
}

mod server {
    fn connect() {
    }
}
```

注意這個模組文件中我們也使用了一個 `mod` 聲明；這是因為我們希望 `server` 成為 `network` 的一個子模組。

現在再次運行 `cargo build`。成功！不過我們還需要再提取出另一個模組：`server`。因為這是一個子模組——也就是模組中的模組——目前的將模組提取到對應名字的文件中的策略就不管用了。如果我們仍這麼嘗試則會出現錯誤。對 *src/network.rs* 的第一個修改是用 `mod server;` 替換 `server` 模組的內容：

<span class="filename">Filename: src/network.rs</span>

```rust
fn connect() {
}

mod server;
```

接著創建 *src/server.rs* 文件並輸入需要提取的 `server` 模組的內容：

<span class="filename">Filename: src/server.rs</span>

```rust
fn connect() {
}
```

當嘗試運行 `cargo build` 時，會出現如列表 7-4 中所示的錯誤：

```text
$ cargo build
   Compiling communicator v0.1.0 (file:///projects/communicator)
error: cannot declare a new module at this location
 --> src/network.rs:4:5
  |
4 | mod server;
  |     ^^^^^^
  |
note: maybe move this module `network` to its own directory via `network/mod.rs`
 --> src/network.rs:4:5
  |
4 | mod server;
  |     ^^^^^^
note: ... or maybe `use` the module `server` instead of possibly redeclaring it
 --> src/network.rs:4:5
  |
4 | mod server;
  |     ^^^^^^
```

<span class="caption">列表 7-4：嘗試將 `server` 子模組提取到 *src/server.rs* 時出現的錯誤</span>

這個錯誤說明 「不能在這個位置新聲明一個模組」 並指出 *src/network.rs* 中的 `mod server;` 這一行。看來 *src/network.rs* 與 *src/lib.rs* 在某些方面是不同的；繼續閱讀以理解這是為什麼。

列表 7-4 中間的 note 事實上是非常有幫助的，因為它指出了一些我們還未講到的操作：

```text
note: maybe move this module `network` to its own directory via
`network/mod.rs`
```

我們可以按照記錄所建議的去操作，而不是繼續使用之前的與模組同名文件的模式：

1. 新建一個叫做 *network* 的 **目錄**，這是父模組的名字
2. 將 *src/network.rs* 移動到新建的 *network* 目錄中並重命名，現在它是 *src/network/mod.rs*
3. 將子模組文件 *src/server.rs* 移動到 *network* 目錄中

如下是執行這些步驟的命令：

```text
$ mkdir src/network
$ mv src/network.rs src/network/mod.rs
$ mv src/server.rs src/network
```

現在如果運行 `cargo build` 的話將順利編譯（雖然仍有警告）。現在模組的佈局看起來仍然與列表 7-3 中所有代碼都在 *src/lib.rs* 中時完全一樣：

```text
communicator
 ├── client
 └── network
     └── server
```

對應的文件佈局現在看起來像這樣：

```text
├── src
│   ├── client.rs
│   ├── lib.rs
│   └── network
│       ├── mod.rs
│       └── server.rs
```

那麼，當我們想要提取 `network::server` 模組時，為什麼也必須將 *src/network.rs* 文件改名成 *src/network/mod.rs* 文件呢，還有為什麼要將`network::server`的代碼放入 *network* 目錄的 *src/network/server.rs* 文件中，而不能將 `network::server` 模組提取到 *src/server.rs* 中呢？原因是如果 *server.rs* 文件在 *src* 目錄中那麼 Rust 就不能知道 `server` 應當是 `network` 的子模組。為了闡明這裡 Rust 的行為，讓我們考慮一下有著如下層級的另一個例子，它的所有定義都位於 *src/lib.rs* 中：

```text
communicator
 ├── client
 └── network
     └── client
```

在這個例子中，仍然有這三個模組，`client`、`network` 和 `network::client`。如果按照與上面最開始將模組提取到文件中相同的步驟來操作，對於 `client` 模組會創建 *src/client.rs*。對於 `network` 模組，會創建 *src/network.rs*。但是接下來不能將 `network::client` 模組提取到 *src/client.rs* 文件中，因為它已經存在了，對應頂層的 `client` 模組！如果將 `client` 和 `network::client` 的代碼都放入 *src/client.rs* 文件，Rust 將無從可知這些代碼是屬於 `client` 還是 `network::client` 的。

因此，一旦想要將 `network` 模組的子模組 `network::client` 提取到一個文件中，需要為 `network` 模組新建一個目錄替代 *src/network.rs* 文件。接著 `network` 模組的代碼將進入 *src/network/mod.rs* 文件，而子模組 `network::client` 將擁有其自己的文件 *src/network/client.rs*。現在頂層的 *src/client.rs* 中的代碼毫無疑問的都屬於 `client` 模組。

### 模組文件系統的規則

與文件系統相關的模組規則總結如下：

* 如果一個叫做 `foo` 的模組沒有子模組，應該將 `foo` 的聲明放入叫做 *foo.rs* 的文件中。
* 如果一個叫做 `foo` 的模組有子模組，應該將 `foo` 的聲明放入叫做 *foo/mod.rs* 的文件中。

這些規則適用於遞歸（嵌套），所以如果 `foo` 模組有一個子模組 `bar` 而 `bar` 沒有子模組，則 *src* 目錄中應該有如下文件：

```text
├── foo
│   ├── bar.rs (contains the declarations in `foo::bar`)
│   └── mod.rs (contains the declarations in `foo`, including `mod bar`)
```

模組自身則應該使用 `mod` 關鍵字定義於父模組的文件中。

接下來，我們討論一下 `pub` 關鍵字，併除掉那些警告！