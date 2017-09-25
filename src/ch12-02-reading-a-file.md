## 讀取文件

> [ch12-02-reading-a-file.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch12-02-reading-a-file.md)
> <br>
> commit b693c8400817f1022820fd63e3529cbecc35070c

接下來我們將讀取由命令行文件名參數指定的文件。首先，需要一個用來測試的示例文件——用來確保 `minigrep` 正常工作的最好的文件是擁有少量文本和多個行且有一些重複單詞的文件。列表 12-3 是一首艾米莉‧狄金森（Emily Dickinson）的詩，它正適合這個工作！在項目根目錄創建一個文件 `poem.txt`，並輸入詩 "I'm nobody! Who are you?"：

<span class="filename">文件名: poem.txt</span>

```text
I'm nobody! Who are you?
Are you nobody, too?
Then there's a pair of us — don't tell!
They'd banish us, you know.

How dreary to be somebody!
How public, like a frog
To tell your name the livelong day
To an admiring bog!
```

<span class="caption">列表 12-3：艾米莉‧狄金森的詩 「I'm nobody! Who are you?」，一個好的測試用例</span>

創建完這個文件之後，修改 *src/main.rs* 並增加如列表 12-4 所示的打開文件的代碼：

<span class="filename">文件名: src/main.rs</span>

```rust,should_panic
use std::env;
use std::fs::File;
use std::io::prelude::*;

fn main() {
#     let args: Vec<String> = env::args().collect();
#
#     let query = &args[1];
#     let filename = &args[2];
#
#     println!("Searching for {}", query);
    // ...snip...
    println!("In file {}", filename);

    let mut f = File::open(filename).expect("file not found");

    let mut contents = String::new();
    f.read_to_string(&mut contents)
        .expect("something went wrong reading the file");

    println!("With text:\n{}", contents);
}
```

<span class="caption">列表 12-4：讀取第二個參數所指定的文件內容</span>

首先，我們增加了更多的 `use` 語句來引入標準庫中的相關部分：需要 `std::fs::File` 來處理文件，而 `std::io::prelude::*` 則包含許多對於 I/O 包括文件 I/O 有幫助的 trait。類似於 Rust 有一個通用的 prelude 來自動引入特定內容，`std::io` 也有其自己的 prelude 來引入處理 I/O 時所需的通用內容。不同於預設的 prelude，必須顯式 `use` 位於 `std::io` 中的 prelude。

在 `main` 中，我們增加了三點內容：第一，通過傳遞變數 `filename` 的值調用 `File::open` 函數來抓取文件的可變句柄。創建了叫做 `contents` 的變數並將其設置為一個可變的，空的`String`。它將會存放之後讀取的文件的內容。第三，對文件句柄調用 `read_to_string` 並傳遞 `contents` 的可變引用作為參數。

在這些代碼之後，我們再次增加了臨時的 `println!` 打印出讀取文件後 `contents` 的值，這樣就可以檢查目前為止的程序能否工作。

嘗試運行這些代碼，隨意指定一個字符串作為第一個命令行參數（因為還未實現搜索功能的部分）而將 *poem.txt* 文件將作為第二個參數：

```text
$ cargo run the poem.txt
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/minigrep the poem.txt`
Searching for the
In file poem.txt
With text:
I'm nobody! Who are you?
Are you nobody, too?
Then there's a pair of us — don't tell!
They'd banish us, you know.

How dreary to be somebody!
How public, like a frog
To tell your name the livelong day
To an admiring bog!
```

好的！代碼讀取並打印出了文件的內容。雖然它還有一些瑕疵：`main` 函數有著多個職能，通常函數只負責一個功能的話會更簡潔並易於維護。另一個問題是沒有儘可能的處理錯誤。雖然我們的程序還很小，這些瑕疵並不是什麼大問題，不過隨著程序功能的豐富，將會越來越難以用簡單的方法修復他們。在開發程序時，及早開始重構是一個最佳實踐，因為重構少量代碼時要容易的多，所以讓我們現在就開始吧。