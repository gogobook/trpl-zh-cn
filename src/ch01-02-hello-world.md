## Hello, World!

> [ch01-02-hello-world.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch01-02-hello-world.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

現在安裝好了 Rust，讓我們來編寫第一個程序。當學習一門新語言的時候，使用該語言在屏幕上打印 「Hello, world!」 是一項傳統，我們將遵循這個傳統。

> 注意：本書假設你熟悉基本的命令行操作。Rust 對於你的編輯器、工具，以及代碼位於何處並沒有特定的要求，如果相比命令行你更傾向於 IDE，請隨意使用合意的 IDE。

### 創建項目目錄

首先，創建一個存放 Rust 代碼的目錄。Rust 並不關心代碼的位置，不過在本書中，我們建議你在 home 目錄中創建一個 *projects* 目錄，並把你的所有項目放在這。打開一個終端，輸入如下命令為這個項目創建一個文件夾：

Linux 和 Mac：

```text
$ mkdir ~/projects
$ cd ~/projects
$ mkdir hello_world
$ cd hello_world
```

Windows 的 cmd：

```cmd
> mkdir %USERPROFILE%\projects
> cd %USERPROFILE%\projects
> mkdir hello_world
> cd hello_world
```

Windows 的 PowerShell：

```powershell
> mkdir $env:USERPROFILE\projects
> cd $env:USERPROFILE\projects
> mkdir hello_world
> cd hello_world
```

### 編寫並運行 Rust 程序

接下來，新建一個叫做 *main.rs* 的源文件。Rust 源代碼總是以 *.rs* 後綴結尾。如果文件名包含多個單詞，使用下劃線分隔它們。例如 *my_program.rs*，而不是 *myprogram.rs*。

現在打開剛創建的 *main.rs* 文件，輸入如下代碼：

<span class="filename">文件名: main.rs</span>

```rust
fn main() {
    println!("Hello, world!");
}
```

保存文件，並回到終端窗口。在 Linux 或 OSX 上，輸入如下命令：

```text
$ rustc main.rs
$ ./main
Hello, world!
```

在 Windows 上，運行 `.\main.exe`，而不是`./main`。不管使用何種系統，應該在終端看到 `Hello, world!` 字樣。如果你做到了，恭喜你！你已經正式編寫了一個 Rust 程序。現在你成為了一名 Rust 程式設計師！歡迎！

### 分析 Rust 程序

現在，讓我們回過頭來仔細看看 「Hello, world!」 程序到底發生了什麼。這是拼圖的第一片：

```rust
fn main() {

}
```

這幾行定義了一個 Rust **函數**。`main` 函數是特殊的：它是每個可執行的 Rust 程序首先執行的。第一行代碼表示 「我聲明了一個叫做 `main` 的函數，它沒有參數也沒有返回值。」 如果有參數的話，他們的名稱應該出現在括號中，`(`和`)`之間。

還須注意涵數體被包裹在花括號中，`{`和`}` 之間。Rust 要求所有函數體都要用花括號包裹起來（譯者註：有些語言，當函數體只有一行時可以省略花括號，但 Rust 中是不行的）。一般來說，將左花括號與函數聲明置於同一行並以空格分隔，是良好的代碼風格。

在 `main()` 函數中：

```rust
    println!("Hello, world!");
```

這行代碼完成這個小程序的所有工作：在屏幕上打印文本。這裡有很多細節需要注意。首先 Rust 使用 4 個空格的縮進風格，而不是 1 個製表符（tab）。

第二個重要的部分是 `println!()`。這給稱為 Rust **巨集**，Rust 元編程（metaprogramming）的關鍵所在。如果是調用函數，則應看起來像這樣：`println`（沒有`!`）。我們將在附錄 E 中更加詳細的討論巨集，現在你只需記住，當看到符號 `!` 的時候，調用的是巨集而不是普通函數。

接下來，`"Hello, world!"` 是一個 **字符串**。我們把這個字符串作為一個參數傳遞給 `println!`，它負責在屏幕上打印這個字符串。輕鬆加愉快！(⊙o⊙)

該行以分號結尾（`;`）。`;` 代表一個表達式的結束和下一個表達式的開始。大部分 Rust 代碼行以 `;` 結尾。

### 編譯和運行是兩個步驟

「編寫並運行 Rust 程序」 部分中展示了如何運行新創建的程序。現在我們將拆分並檢查每一步操作。

在運行一個 Rust 程序之前必須先進行編譯。可以通過 `rustc` 命令來使用 Rust 編譯器，並傳遞源文件的名字給它，如下：

```text
$ rustc main.rs
```

如果你有 C 或 C++ 背景，就會發現這與 `gcc` 和 `clang` 類似。編譯成功後，Rust 應該會輸出一個二進制可執行文件，在 Linux 或 OSX 上在 shell 中可以通過 `ls` 命令看到如下內容：

```text
$ ls
main  main.rs
```

在 Windows 上，輸入：

```cmd
> dir /B %= the /B option says to only show the file names =%
main.exe
main.rs
```

這表示我們有兩個文件：*.rs* 後綴的源文件，和可執行文件（在 Windows下是 *main.exe*，其它平台是 *main*）。餘下需要做的就是運行 *main* 或 *main.exe* 文件，如下：

```text
$ ./main  # or .\main.exe on Windows
```

如果 *main.rs* 是 「Hello, world!」 程序，它將會在終端上打印 `Hello, world!`。

來自 Ruby、Python 或 JavaScript 這樣的動態類型語言背景的同學，可能不太習慣將編譯和執行分為兩個單獨的步驟。Rust 是一種 **預編譯靜態類型語言**（*ahead-of-time compiled language*），這意味著你可以編譯程序並將其交與他人，他們不需要安裝 Rust 即可運行。相反如果你給他們一個 `.rb`、`.py` 或 `.js` 文件，他們需要先分別安裝 Ruby，Python，JavaScript 實現（運行時環境，VM），不過你只需要一句命令就可以編譯和執行程序。這一切都是語言設計上的權衡取捨。

使用 `rustc` 編譯簡單程序是沒問題的，不過隨著項目的增長，你可能需要控制你項目的方方面面，並且更容易地將代碼分享給其它人或項目。接下來，我們要介紹一個叫做 Cargo 的工具，它會幫助你編寫真實世界中的 Rust 程序。

## Hello, Cargo!

Cargo 是 Rust 的構建系統和包管理工具，同時 Rustacean 們使用 Cargo 來管理他們的 Rust 項目，它使得很多任務變得更輕鬆。例如，Cargo 負責構建代碼、下載依賴庫並編譯他們。我們把代碼需要的庫叫做 **依賴**（*dependencies*）。

最簡單的 Rust 程序，比如我們剛剛編寫的，並沒有任何依賴，所以我們只使用了 Cargo 構建代碼的功能。隨著編寫的程序更加複雜，你會想要添加依賴，如果你使用 Cargo 開始的話，這將會變得簡單許多。

由於絕大部分 Rust 項目使用 Cargo，本書接下來的部分將假設你使用它。如果使用之前介紹的官方安裝包的話，則自帶了 Cargo。如果通過其他方式安裝的話，可以在終端輸入如下命令檢查是否安裝了 Cargo：

```text
$ cargo --version
```

如果出現了版本號，一切 OK！如果出現類似 `command not found` 的錯誤，你應該查看相應安裝文檔以確定如何單獨安裝 Cargo。

### 使用 Cargo 創建項目

讓我們使用 Cargo 來創建一個新項目，然後看看與上面的 `hello_world` 項目有什麼不同。回到 projects 目錄（或者任何你放置代碼的目錄）：

Linux 和 Mac:

```text
$ cd ~/projects
```

Windows:

```cmd
> cd %USERPROFILE%\projects
```

並在任何操作系統下運行：

```text
$ cargo new hello_cargo --bin
$ cd hello_cargo
```

我們向 `cargo new` 傳遞了 `--bin`，因為我們的目標是生成一個可執行程序，而不是一個庫。可執行程序是二進制可執行文件，通常就叫做 **二進制文件**（*binaries*）。項目的名稱被定為 `hello_cargo`，同時 Cargo 在一個同名目錄中創建它的文件，接著我們可以進入查看。

如果列出 *hello_cargo* 目錄中的文件，將會看到 Cargo 生成了一個文件和一個目錄：一個 *Cargo.toml* 文件和一個 *src* 目錄，*main.rs* 文件位於 *src* 目錄中。它也在 *hello_cargo* 目錄初始化了一個 git 倉庫，以及一個 *.gitignore* 文件；你可以通過`--vcs`參數切換到其它版本控制系統（VCS），或者不使用 VCS。

使用文本編輯器（工具請隨意）打開 *Cargo.toml* 文件。它應該看起來像這樣：

<span class="filename">文件名: Cargo.toml</span>

```toml
[package]
name = "hello_cargo"
version = "0.1.0"
authors = ["Your Name <you@example.com>"]

[dependencies]
```

這個文件使用 [*TOML*][toml]<!-- ignore --> (Tom's Obvious, Minimal Language) 格式。TOML 類似於 INI，不過有一些額外的改進之處，並且被用作 Cargo 的配置文件的格式。

[toml]: https://github.com/toml-lang/toml

第一行，`[package]`，是一個段落標題，表明下面的語句用來配置一個包。隨著我們在這個文件增加更多的信息，還將增加其他段落。

接下來的三行設置了三個 Cargo 所需的配置，他們告訴 Cargo 需要編譯這個項目：名稱、版本和作者。Cargo 從環境中獲取你的名稱和 email 信息。如果不正確，請修改並保存此文件。

最後一行，`[dependencies]`，是項目依賴的 *crates* 列表（我們這樣稱呼 Rust 代碼包）段落的開始，這樣 Cargo 就知道下載和編譯它們了。這個項目並不需要任何其他的 crate，不過在下一章猜猜看教程會用得上。

現在看看 *src/main.rs*：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    println!("Hello, world!");
}
```

Cargo 為你生成了一個 「Hello World!」，正如我們之前編寫的那個！目前為止，之前項目與 Cargo 生成項目的區別有：

- 代碼位於 *src* 目錄
- 項目根目錄包含一個 *Cargo.toml* 配置文件

Cargo 期望源文件位於 *src* 目錄，將項目根目錄留給 README、license 信息、配置文件和其他跟代碼無關的文件。這樣，Cargo 幫助你保持項目乾淨整潔，一切井井有條。

如果沒有用 Cargo 創建項目，比如 *hello_world* 目錄中的項目，可以通過將代碼放入 *src* 目錄，並創建一個合適的 *Cargo.toml*，將其轉化為一個 Cargo 項目。

### 構建並運行 Cargo 項目

現在讓我們看看通過 Cargo 構建和運行 Hello World 程序有什麼不同。為此，輸入如下命令：

```text
$ cargo build
   Compiling hello_cargo v0.1.0 (file:///projects/hello_cargo)
```

這應該會創建 *target/debug/hello_cargo*（或者在 Windows 上是 *target\debug\hello_cargo.exe*）可執行文件，可以通過這個命令運行：

```text
$ ./target/debug/hello_cargo # or .\target\debug\hello_cargo.exe on Windows
Hello, world!
```

好的！如果一切順利，`Hello, world!`應該再次打印在終端上。

首次運行 `cargo build` 的時候，Cargo 會在項目根目錄創建一個新文件，*Cargo.lock*，它看起來像這樣：

<span class="filename">文件名: Cargo.lock</span>

```toml
[root]
name = "hello_cargo"
version = "0.1.0"
```

Cargo 使用 *Cargo.lock* 來記錄程序的依賴。這個項目並沒有依賴，所以其內容比較少。事實上，你自己永遠也不需要碰這個文件，讓 Cargo 處理它就行了。

我們剛剛使用 `cargo build` 構建了項目並使用 `./target/debug/hello_cargo` 運行了程序，也可以使用 `cargo run` 同時編譯並運行：

```text
$ cargo run
     Running `target/debug/hello_cargo`
Hello, world!
```

注意這一次並沒有出現 Cargo 正在編譯 `hello_cargo` 的輸出。Cargo 發現文件並沒有被改變，就直接運行了二進制文件。如果修改了源文件的話，Cargo 會在運行之前重新構建項目，並會出現像這樣的輸出：

```text
$ cargo run
   Compiling hello_cargo v0.1.0 (file:///projects/hello_cargo)
     Running `target/debug/hello_cargo`
Hello, world!
```

所以現在又出現更多的不同：

- 使用 `cargo build` 構建項目（或使用 `cargo run` 一步構建並運行），而不是使用`rustc`
- 有別於將構建結果放在與源碼相同的目錄，Cargo 會將其放到 *target/debug* 目錄。

Cargo 的另一個優點是，不管你使用什麼操作系統其命令都是一樣的，所以本書之後將不再為 Linux 和 Mac 以及 Windows 提供相應的命令。

### 發佈（release）構建

當項目最終準備好發佈了，可以使用 `cargo build --release` 來優化編譯項目。這會在 *target/release* 而不是  *target/debug* 下生成可執行文件。這些優化可以讓 Rust 代碼運行的更快，不過打開他們也需要更長的編譯時間。這也就是為什麼會有兩種兩種不同的配置：一種為了開發，你需要經常快速重新構建；另一種構建給用戶的最終程序，他們不會重新構建，並且希望程序運行得越快越好。如果你在測試代碼的運行時間，請確保運行 `cargo build --release` 並使用 *target/release* 下的可執行文件進行測試。

### 把 Cargo 當作習慣

對於簡單項目， Cargo 並不比 `rustc` 提供了更多的優勢，不過隨著開發的深入終將證明其價值。對於擁有多個 crate 的複雜項目，讓 Cargo 來協調構建將更簡單。有了 Cargo，只需運行`cargo build`，然後一切將有序運行。即便這個項目很簡單，它現在也使用了很多你之後的 Rust 生涯將會用得上的實用工具。其實你可以使用下面的命令開始任何你想要從事的項目：

```text
$ git clone someurl.com/someproject
$ cd someproject
$ cargo build
```

> 注意：如果想要瞭解 Cargo 更多的細節，請閱讀官方的 [Cargo guide]，它覆蓋了 Cargo 所有的功能。

[Cargo guide]: http://doc.crates.io/guide.html