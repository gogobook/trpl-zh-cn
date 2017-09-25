## 接受命令行參數

> [ch12-01-accepting-command-line-arguments.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch12-01-accepting-command-line-arguments.md)
> <br>
> commit 50658e654fb6a9208b635179cdd79939aa0ab133

一如之前使用 `cargo new` 新建一個項目。我們稱之為 `minigrep` 以便與可能已經安裝在系統上的`grep`工具相區別：

```text
$ cargo new --bin minigrep
     Created binary (application) `minigrep` project
$ cd minigrep
```

第一個任務是讓 `minigrep` 能夠接受兩個命令行參數：文件名和要搜索的字符串。也就是說我們希望能夠使用 `cargo run`、要搜索的字符串和被搜索的文件的路徑來運行程序，像這樣：

```text
$ cargo run searchstring example-filename.txt
```

現在 `cargo new` 生成的程序忽略任何傳遞給它的參數。crates.io 上有一些現成的庫可以幫助我們接受命令行參數，不過因為正在學習，讓我們自己來實現一個。

### 讀取參數值

首先我們需要程序能夠獲取傳遞給它的命令行參數的值，為此需要一個 Rust 標準庫提供的函數：`std::env::args`。這個函數返回一個傳遞給程序的命令行參數的 **迭代器**（*iterator*）。我們還未討論到迭代器，第十三章會全面的介紹他們。但是對於我們現在的目的來說只需要明白兩點：迭代器生成一系列的值，可以在迭代器上調用 `collect` 方法將其轉換為一個 vector，比如包含所有迭代器產生元素的 vector。

讓我們嘗試一下：使用列表 12-1 中的代碼來讀取任何傳遞給 `minigrep` 的命令行參數並將其收集到一個 vector 中。

<span class="filename">文件名: src/main.rs</span>

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    println!("{:?}", args);
}
```

列表 12-1：將命令行參數收集到一個 vector 中並打印出來

首先使用 `use` 語句來將 `std::env` 模塊引入作用域以便可以使用它的 `args` 函數。注意 `std::env::args` 函數被嵌套進了兩層模塊中。正如第七章講到的，當所需函數嵌套了多於一層模塊時，通常將父模塊引入作用域，而不是其自身。這便於我們利用 `std::env` 中的其他函數。這比增加了 `use std::env::args;` 後僅僅使用 `args` 調用函數要更明確一些；這樣容易被錯認成一個定義於當前模塊的函數。

> ### `args` 函數和無效的 Unicode
>
> 注意 `std::env::args` 在其任何參數包含無效 Unicode 字符時會 panic。如果你需要接受包含無效 Unicode 字符的參數，使用 `std::env::args_os` 代替。這個函數返回 `OsString` 值而不是 `String` 值。這裡出於簡單考慮使用了 `std::env::args`，因為 `OsString` 值每個平台都不一樣而且比 `String` 值處理起來更複雜。

在 `main` 函數的第一行，我們調用了 `env::args`，並立即使用 `collect` 來創建了一個包含迭代器所有值的 vector。`collect` 可以被用來創建很多類型的集合，所以這裡顯式註明的 `args` 類型來指定我們需要一個字符串 vector。雖然在 Rust 中我們很少會需要註明類型，`collect` 就是一個經常需要註明類型的函數，因為 Rust 不能推斷出你想要什麼類型的集合。

最後，我們使用調試格式 `:?` 打印出 vector。讓我們嘗試不用參數運行代碼，接著用兩個參數：

```text
$ cargo run
["target/debug/minigrep"]

$ cargo run needle haystack
...snip...
["target/debug/minigrep", "needle", "haystack"]
```

你可能注意到了 vector 的第一個值是 `"target/debug/minigrep"`，它是我們二進制文件的名稱。這與 C 中的參數列表的行為相符合，並使得程序可以在執行過程中使用它的名字。能夠訪問程序名稱在需要在信息中打印時，或者需要根據執行程序所使用的命令行別名來改變程序行為時顯得很方便，不過考慮到本章的目的，我們將忽略它並只保存所需的兩個參數。

### 將參數值保存進變數

打印出參數 vector 中的值展示了程序可以訪問指定為命令行參數的值。現在需要將這兩個參數的值保存進變數這樣就可以在程序的餘下部分使用這些值。讓我們如列表 12-2 這樣做：

<span class="filename">文件名: src/main.rs</span>

```rust,should_panic
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();

    let query = &args[1];
    let filename = &args[2];

    println!("Searching for {}", query);
    println!("In file {}", filename);
}
```

列表 12-2：創建變數來存放查詢參數和文件名參數

正如之前打印出 vector 時所所看到的，程序的名稱佔據了 vector 的第一個值 `args[0]`，所以我們從索引 `1` 開始。`minigrep` 獲取的第一個參數是需要搜索的字符串，所以將其將第一個參數的引用存放在變數 `query` 中。第二個參數將是文件名，所以將第二個參數的引用放入變數 `filename` 中。

我們將臨時打印出出這些變數的值，再一次證明代碼如我們期望的那樣工作。讓我們使用參數 `test` 和 `sample.txt` 再次運行這個程序：

```text
$ cargo run test sample.txt
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/minigrep test sample.txt`
Searching for test
In file sample.txt
```

好的，它可以工作！我們將所需的參數值保存進了對應的變數中。之後會增加一些錯誤處理來應對類似用戶沒有提供參數的情況，不過現在我們將忽略他們並開始增加讀取文件功能。