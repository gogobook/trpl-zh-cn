## 安裝

> [ch01-01-installation.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch01-01-installation.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

使用 Rust 的第一步是安裝。你需要網絡連接來執行本章的命令，因為將要從網上下載 Rust。

這裡將會展示很多使用終端的命令，這些命令均以 `$` 開頭。不需要真的輸入`$`，在這裡他們代表每行命令的起始。網上有很多教程和例子遵循這種慣例：`$` 代表以常規用戶身份運行命令，`#` 代表需要用管理員身份運行命令。沒有以 `$`（或 `#`）起始的行通常是之前命令的輸出。

### 在 Linux 或 Mac 上安裝

如果你使用 Linux 或 Mac，你需要做的全部就是打開一個終端並輸入：

```text
$ curl https://sh.rustup.rs -sSf | sh
```

這會下載一個腳本並開始安裝。可能會提示你輸入密碼，如果一切順利，將會出現如下內容：

```text
Rust is installed now. Great!
```

當然，如果你對於 `curl | sh` 這樣的模式心有疑慮，請隨意下載、檢查和運行這個腳本。

此安裝腳本自動將 Rust 加入系統 PATH 環境變數中，再次登陸時生效。如果你希望立刻（不重新登陸）就開始使用 Rust，在 shell 中運行如下命令：

```text
$ source $HOME/.cargo/env
```

或者，在 `~/.bash_profile` 文件中增加如下行：

```text
$ export PATH="$HOME/.cargo/bin:$PATH"
```

### 在 Windows 上安裝

如果你使用 Windows，前往 [https://rustup.rs](https://rustup.rs/)<!-- ignore -->，按說明下載 rustup-init.exe，運行並照其指示操作。

本書中其餘 Windows 相關的命令，假設你使用 `cmd` 作為 shell。如果你使用其它 shell，也許可以執行與 Linux 和 Mac 用戶相同的命令。如果不行，請查看該 shell 的文檔。

### 自定義安裝

無論出於何種理由，如果不願意使用 rustup.rs，請查看 [Rust 安裝頁面](https://www.rust-lang.org/install.html) 獲取其他選項。


### 更新

一旦安裝了 Rust，更新到最新版本是很簡單的。在 shell 中執行更新腳本：

```text
$ rustup update
```

### 卸載

卸載 Rust 與安裝一個簡單。在 shell 中執行卸載腳本:

```text
$ rustup self uninstall
```

### 故障排除（Troubleshooting）

安裝完 Rust 後，打開 shell 並執行：

```text
$ rustc --version
```

應該能看到類似這樣格式的版本號、提交哈希和提交日期，對應安裝時的最新穩定版：

```text
rustc x.y.z (abcabcabc yyyy-mm-dd)
```

如果出現這些內容，Rust 就安裝成功了！

恭喜入坑！（此處應該有掌聲！）

如果在 Windows 中使用出現問題，檢查 Rust（rustc，cargo 等）是否在 `%PATH%` 環境變數所包含的路徑中。

如果還是不能解決，有許多地方可以求助。最簡單的是 [irc.mozilla.org 上的 #rust IRC 頻道][irc]<!-- ignore --> ，可以使用 [Mibbit][mibbit] 來訪問它。然後就能和其他 Rustacean（Rust 用戶的稱號，有自嘲意味）聊天並尋求幫助。其它給力的資源包括[用戶論壇][users]和 [Stack Overflow][stackoverflow]。

[irc]: irc://irc.mozilla.org/#rust
[mibbit]: http://chat.mibbit.com/?server=irc.mozilla.org&channel=%23rust
[users]: https://users.rust-lang.org/
[stackoverflow]: http://stackoverflow.com/questions/tagged/rust

### 本地文檔

安裝程序也自帶一份文檔的本地拷貝，可以離線閱讀。運行 `rustup doc` 在瀏覽器中查看本地文檔。

任何時候，如果你拿不準標準庫中的類型或函數如何工作，請查看 API 文檔！