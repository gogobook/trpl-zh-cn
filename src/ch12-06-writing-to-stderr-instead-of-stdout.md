## 輸出到`stderr`而不是`stdout`

> [ch12-06-writing-to-stderr-instead-of-stdout.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch12-06-writing-to-stderr-instead-of-stdout.md)
> <br>
> commit 7db14aa689553706198ffcb11a8c60b478e752fe

目前為止，我們將所有的輸出都 `println!` 到了終端。大部分終端都提供了兩種輸出：**標準輸出**（*standard output*）對應大部分信息（有時在代碼中使用縮寫 `stdout`），**標準錯誤**（*standard error*）則用於錯誤信息（`stderr`）。這種區別允許用戶選擇將程序正常輸出定向到一個文件中並仍將錯誤信息打印到屏幕上。

但是 `println!` 函數只能夠打印到標準輸出，所以我們必需使用其他方法來打印到標準錯誤。

### 檢查錯誤應該寫入何處

首先，讓我們觀察一下目前 `minigrep` 打印的所有內容都被寫入了標準輸出，包括應該被寫入標準錯誤的錯誤信息。可以通過將標準輸出流重定向到一個文件同時有意產生一個錯誤來做到這一點。我們沒有重定向標準錯誤流，所以任何發送到標準錯誤的內容將會繼續顯示在屏幕上。命令行程序被期望將錯誤信息發送到標準錯誤流，這樣即便選擇將標準輸出流重定向到文件中時仍然能看到錯誤信息。目前我們的程序並不符合期望；我們將看到相反它將錯誤信息輸出保存到了文件中。

展示這種行為的方式是通過 `>` 和文件名 *output.txt* 來與運行程序，這個文件是期望重定向標準輸出流的位置。並不傳遞任何參數這樣應該會產生一個錯誤：

```text
$ cargo run > output.txt
```

`>` 語法告訴 shell 將標準輸出的內容寫入到 *output.txt* 文件中而不是屏幕上。我們並沒有看到期望的錯誤信息打印到屏幕上，所以這意味著它一定被寫入了文件中。讓我們看看 *output.txt* 包含什麼：

```text
Problem parsing arguments: not enough arguments
```

是的，錯誤信息被打印到了標準輸出中。像這樣的錯誤信息被打印到標準錯誤中將有用的多，並在重定向標準輸出時只將成功運行的信息寫入文件。我們將改變他們。

### 將錯誤打印到標準錯誤

讓我們如示例 12-24 所示的代碼改變錯誤信息是如何被打印的。得益於本章早些時候的重構，所有打印錯誤信息的代碼都位於 `main` 一個函數中。標準庫提供了 `eprintln!` 巨集來打印到標準錯誤流，所以將兩個調用 `println!` 打印錯誤信息的維持替換為 `eprintln!`：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args).unwrap_or_else(|err| {
        eprintln!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    if let Err(e) = minigrep::run(config) {
        eprintln!("Application error: {}", e);

        process::exit(1);
    }
}
```

<span class="caption">示例 12-24：使用 `eprintln!` 將錯誤信息寫入標準錯誤而不是標準輸出</span>

將 `println!` 改為 `eprintln!` 之後，讓我們再次嘗試用同樣的方式運行程序，不使用任何參數並通過 `>` 重定向標準輸出：

```text
$ cargo run > output.txt
Problem parsing arguments: not enough arguments
```

現在我們看到了屏幕上的錯誤信息，同時 `output.txt` 裡什麼也沒有，這也就是命令行程序所期望的行為。

如果使用不會造成錯誤的參數再次運行程序，不過仍然將標準輸出重定向到一個文件：

```text
$ cargo run to poem.txt > output.txt
```

我們並不會在終端看到任何輸出，同時 `output.txt` 將會包含其結果：

<span class="filename">文件名: output.txt</span>

```text
Are you nobody, too?
How dreary to be somebody!
```

這一部分展示了現在我們適當的使用成功時產生的標準輸出和錯誤時產生的標準錯誤。

## 總結

在這一章中，我們回顧了目前為止的一些主要章節並涉及了如何在 Rust 環境中進行常規的 I/O 操作。通過使用命令行參數、文件、環境變數和打印錯誤的 `eprintln!` 巨集，現在你已經準備好編寫命令行程序了。通過結合前幾章的知識，你的代碼將會是組織良好的，並能有效的將數據存儲到合適的數據結構中、更好的處理錯誤，並且還是經過良好測試的。

接下來，讓我們探索如何利用一些 Rust 中受函數式編程語言影響的功能：閉包和迭代器。