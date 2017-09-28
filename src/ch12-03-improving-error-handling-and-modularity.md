## 重構改進模組性和錯誤處理

> [ch12-03-improving-error-handling-and-modularity.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch12-03-improving-error-handling-and-modularity.md)
> <br>
> commit 5908c59a5a4cc58fd863605b80b295a335c2cbdf

為了改善我們的程序這裡有四個問題需要修復，而且他們都與程序的組織方式和如何處理潛在錯誤有關。

第一，`main` 現在進行了兩個任務：它解析了參數並打開了文件。對於一個這樣的小函數，這並不是一個大問題。然而如果 `main` 中的功能持續增加，`main` 函數處理的獨立的任務也會增加。當函數承擔了更多責任，它就更難以推導，更難以測試，並且更難以在不破壞其他部分的情況下做出修改。最好能分離出功能以便每個函數就負責一個任務。

這同時也關係到第二個問題：`search` 和 `filename` 是程序中的配置變數，而像 `f` 和 `contents` 則用來執行程序邏輯。隨著 `main` 函數的增長，就需要引入更多的變數到作用域中，而當作用域中有更多的變數時，將更難以追蹤每個變數的目的。最好能將將配置變數組織進一個結構這樣就能使他們的目的更明確了。

第三個問題是如果打開文件失敗我們使用 `expect` 來打印出錯誤信息，不過這個錯誤信息只是說 `file not found`。除了缺少文件之外還有很多打開文件可能失敗的方式：例如，文件可能存在，不過可能沒有打開它的權限。如果我們現在就出於這種情況，打印出的 `file not found` 錯誤信息就給了用戶錯誤的建議！

第四，我們不停的使用 `expect` 來處理不同的錯誤，如果用戶沒有指定足夠的參數來運行程序，他們會從 Rust 得到 "index out of bounds" 錯誤，而這並不能明確的解釋問題。如果所有的錯誤處理都位於一處這樣將來的維護者在需要修改錯誤處理邏輯時就只需要考慮這一處代碼。將所有的錯誤處理都放在一處也有助於確保我們打印的錯誤信息對終端用戶來說是有意義的。

讓我們通過重構項目來解決這些問題。

### 二進制項目的關注分離

`main` 函數負責多個任務的組織問題在許多二進制項目中很常見。所以 Rust 社區開發出一類在 `main` 函數開始變得龐大時進行二進制程序的關注分離的指導性過程。這些過程有如下步驟：

1. 將程序拆分成 *main.rs* 和 *lib.rs* 並將程序的邏輯放入 *lib.rs* 中。
2. 當命令行解析邏輯比較小時，可以保留在 *main.rs* 中。
3. 當命令行解析開始變得複雜時，也同樣將其從 *main.rs* 提取到 *lib.rs*中。
4. 經過這些過程之後保留在 `main` 函數中的責任應該被限制為：
    * 使用參數值調用命令行解析邏輯
    * 設置任何其他的配置
    * 調用 *lib.rs* 中的 `run` 函數
    * 如果 `run` 返回錯誤，則處理這個錯誤

這個模式的一切就是為了關注分離：*main.rs* 處理程序運行，而 *lib.rs* 處理所有的真正的任務邏輯。因為不能直接測試 `main` 函數，這個結構通過將所有的程序邏輯移動到 *lib.rs* 的函數中使得我們可以測試他們。僅僅保留在 *main.rs* 中的代碼將足夠小以便閱讀就可以驗證其正確性。

### 提取參數解析器

首先，我們將解析參數的功能提取到一個 `main` 將會調用的函數中，為將命令行解析邏輯移動到 *src/lib.rs* 做準備。示例 12-5 中展示了新 `main` 函數的開頭，它調用了新函數 `parse_config`。目前它仍將定義在 *src/main.rs* 中：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let args: Vec<String> = env::args().collect();

    let (query, filename) = parse_config(&args);

    // ...snip...
}

fn parse_config(args: &[String]) -> (&str, &str) {
    let query = &args[1];
    let filename = &args[2];

    (query, filename)
}
```

<span class="caption">示例 12-5：從 `main` 中提取出 `parse_config` 函數</span>

我們仍然將命令行參數收集進一個 vector，不過不同於在`main`函數中將索引 1 的參數值賦值給變數 `query` 和將索引 2 的值賦值給變數 `filename`，我們將整個 vector 傳遞給 `parse_config` 函數。接著 `parse_config` 函數將包含決定哪個參數該放入哪個變數的邏輯，並將這些值返回到 `main`。仍然在 `main` 中創建變數 `query` 和 `filename`，不過 `main` 不再負責處理命令行參數與變數如何對應。

這對我們這小程序可能有點大材小用，不過我們將採用小的、增量的步驟進行重構。在做出這些改變之後，再次運行程序並驗證參數解析是否仍然正常。經常驗證你的進展是一個好習慣，這樣在遇到問題時能幫助你定位問題的成因。

### 組合配置值

我們可以採取另一個小的步驟來進一步改善這個函數。現在函數返回一個元組，不過立刻又就將元組拆成了獨立的部分。這是一個我們可能沒有進行正確抽象的信號。

另一個表明還有改進空間的跡像是 `parse_config` 名稱的 `config` 部分，它暗示了我們返回的兩個值是相關的並都是一個配置值的一部分。目前除了將這兩個值組合進元組之外並沒有表達這個數據結構的意義：我們可以將這兩個值放入一個結構體並給每個字段一個有意義的名字。這會讓未來的維護者更容易理解不同的值如何相互關聯以及他們的目的。

> 注意：一些同學將這種拒絕使用相對而言更為合適的復合類型而使用基本類型的模式稱為 **基本類型偏執**（*primitive obsession*）。

示例 12-6 展示了新定義的結構體 `Config`，它有字段 `query` 和 `filename`。我們也改變了 `parse_config` 函數來返回一個 `Config` 結構體的實例，並更新 `main` 來使用結構體字段而不是單獨的變數：

<span class="filename">文件名: src/main.rs</span>

```rust,should_panic
# use std::env;
# use std::fs::File;
#
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = parse_config(&args);

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    let mut f = File::open(config.filename).expect("file not found");

    // ...snip...
}

struct Config {
    query: String,
    filename: String,
}

fn parse_config(args: &[String]) -> Config {
    let query = args[1].clone();
    let filename = args[2].clone();

    Config { query, filename }
}
```

<span class="caption">示例 12-6：重構 `parse_config` 返回一個 `Config` 結構體實例</span>

`parse_config` 的簽名表明它現在返回一個 `Config` 值。在 `parse_config` 的函數體中，之前返回引用了 `args` 中 `String` 值的字符串 slice，現在我們選擇定義 `Config` 來包含擁有所有權的 `String` 值。`main` 中的 `args` 變數是參數值的所有者並只允許 `parse_config` 函數借用他們，這意味著如果 `Config` 嘗試抓取 `args` 中值的所有權將違反 Rust 的借用規則。

還有許多不同的方式可以處理 `String` 的數據，而最簡單但有些不太高效的方式是調用這些值的 `clone` 方法。這會生成 `Config` 實例可以擁有的數據的完整拷貝，不過會比儲存字符串數據的引用消耗更多的時間和內存。不過拷貝數據使得代碼顯得更加直白因為無需管理引用的生命週期，所以在這種情況下犧牲一小部分性能來換取簡潔性的取捨是值得的。

> #### 使用 `clone` 的權衡取捨
>
> 由於其運行時消耗，許多 Rustacean 之間有一個趨勢是傾向於避免使用 `clone` 來解決所有權問題。在關於迭代器的第十三章中，我們將會學習如何更有效率的處理這種情況，不過現在，複製一些字符串來取得進展是沒有問題的，因為只會進行一次這樣的拷貝，而且文件名和要搜索的字符串都比較短。在第一輪編寫時擁有一個可以工作但有點低效的程序要比嘗試過度優化代碼更好一些。隨著你對 Rust 更加熟練，將能更輕鬆的直奔合適的方法，不過現在調用 `clone` 是完全可以接受的。

我們更新 `main` 將 `parse_config` 返回的 `Config` 實例放入變數 `config` 中，並將之前分別使用 `search` 和 `filename` 變數的代碼更新為現在的使用 `Config` 結構體的字段的代碼。

現在代碼更明確的表現了我們的意圖，`query` 和 `filename` 是相關聯的並且他們的目的是配置程序如何工作的。任何使用這些值的代碼就知道在 `config` 實例中對應目的的字段名中尋找他們。

### 創建一個 `Config` 構造函數

目前為止，我們將負責解析命令行參數的邏輯從 `main` 提取到了 `parse_config` 函數中，這有助於我們看清值 `query` 和 `filename` 是相互關聯的並應該在代碼中表現這種關係。接著我們增加了 `Config` 結構體來描述 `query` 和 `filename` 的相關性，並能夠從 `parse_config` 函數中將這些值的名稱作為結構體字段名稱返回。

所以現在 `parse_config` 函數的目的是創建一個 `Config` 實例，我們可以將 `parse_config` 從一個普通函數變為一個叫做 `new` 的與結構體關聯的函數。做出這個改變使得代碼更符合習慣：可以像標準庫中的 `String` 調用 `String::new` 來創建一個該類型的實例那樣，將 `parse_config` 變為一個與 `Config` 關聯的 `new` 函數。示例 12-7 展示了需要做出的修改：


<span class="filename">文件名: src/main.rs</span>

```rust,should_panic
# use std::env;
#
fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args);

    // ...snip...
}

# struct Config {
#     query: String,
#     filename: String,
# }
#
// ...snip...

impl Config {
    fn new(args: &[String]) -> Config {
        let query = args[1].clone();
        let filename = args[2].clone();

        Config { query, filename }
    }
}
```

<span class="caption">示例 12-7：將 `parse_config` 變為 `Config::new`</span>

這裡將 `main` 中調用 `parse_config` 的地方更新為調用 `Config::new`。我們將 `parse_config` 的名字改為 `new` 並將其移動到 `impl` 塊中，這使得 `new` 函數與 `Config` 相關聯。再次嘗試編譯並確保它可以工作。

### 修復錯誤處理

現在我們開始修復錯誤處理。回憶一下之前提到過如果 `args` vector 包含少於 3 個項並嘗試訪問 vector 中索引 1 或 索引 2 的值會造成程序 panic。嘗試不帶任何參數運行程序；這將看起來像這樣：

```text
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/minigrep`
thread 'main' panicked at 'index out of bounds: the len is 1
but the index is 1', /stable-dist-rustc/build/src/libcollections/vec.rs:1307
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

`index out of bounds: the len is 1 but the index is 1` 是一個針對程式設計師的錯誤信息，然而這並不能真正幫助終端用戶理解發生了什麼和他們應該做什麼。現在就讓我們修復它吧。

### 改善錯誤信息

在示例 12-8 中，在 `new` 函數中增加了一個檢查在訪問索引 1 和 2 之前檢查 slice 是否足夠長。如果 slice 不夠長，我們使用一個更好的錯誤信息 panic 而不是 `index out of bounds` 信息：

<span class="filename">文件名: src/main.rs</span>

```rust
// ...snip...
fn new(args: &[String]) -> Config {
    if args.len() < 3 {
        panic!("not enough arguments");
    }
    // ...snip...
```

<span class="caption">示例 12-8：增加一個參數數量檢查</span>

這類似於示例 9-8 中的 `Guess::new` 函數，那裡如果 `value` 參數超出了有效值的範圍就調用 `panic!`。不同於檢查值的範圍，這裡檢查 `args` 的長度至少是 3，而函數的剩餘部分則可以在假設這個條件成立的基礎上運行。如果 
`args` 少於 3 個項，則這個條件將為真，並調用 `panic!` 立即終止程序。

有了 `new` 中這幾行額外的代碼，再次不帶任何參數運行程序並看看現在錯誤看起來像什麼：

```bash
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/minigrep`
thread 'main' panicked at 'not enough arguments', src/main.rs:29
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

這個輸出就好多了，現在有了一個合理的錯誤信息。然而，我們還有一堆額外的信息不希望提供給用戶。所以在這裡使用示例 9-8 中的技術可能不是最好的；無論如何 `panic!` 調用更適合程序上的問題而不是使用上的問題，正如第九章所講到的。相反我們可以使用那一章學習的另一個技術：返回一個可以表明成功或錯誤的 `Result`。

#### 從 `new` 中返回 `Result` 而不是調用 `panic!`

我們可以選擇返回一個 `Result` 值，它在成功時會包含一個 `Config` 的實例，而在錯誤時會描述問題。當 `Config::new` 與 `main` 交流時，可以使用 `Result` 類型來表明這裡存在問題。接著修改 `main` 將 `Err` 成員轉換為對用戶更友好的錯誤，而不是 `panic!` 調用產生的關於 `thread 'main'` 和 `RUST_BACKTRACE` 的文本。

示例 12-9 展示了為了返回 `Result` 在 `Config::new` 的返回值和函數體中所需的改變：

<span class="filename">文件名: src/main.rs</span>

```rust
impl Config {
    fn new(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let filename = args[2].clone();

        Ok(Config { query, filename })
    }
}
```

<span class="caption">示例 12-9：從 `Config::new` 中返回 `Result`</span>

現在 `new` 函數返回一個 `Result`，在成功時帶有一個 `Config` 實例而在出現錯誤時帶有一個 `&'static str`。回憶一下第十章 「靜態生命週期」 中講到 `&'static str` 是一個字符串字面值，也是目前的錯誤信息。

`new` 函數體中有兩處修改：當沒有足夠參數時不再調用 `panic!`，而是返回 `Err` 值。同時我們將 `Config` 返回值包裝進 `Ok` 成員中。這些修改使得函數符合其新的類型簽名。

通過讓 `Config::new` 返回一個 `Err` 值，這就允許 `main` 函數處理 `new` 函數返回的 `Result` 值並在出現錯誤的情況更明確的結束進程。

### `Config::new` 調用並處理錯誤

為了處理錯誤情況並打印一個對用戶友好的信息，我們需要像示例 12-10 那樣更新 `main` 函數來處理現在 `Config::new` 返回的 `Result`。另外還需要負責手動實現 `panic!` 的使用非零錯誤碼退出命令行工具的工作。非零的退出狀態是一個告訴調用程序的進程我們的程序以錯誤狀態退出的慣例信號。

<span class="filename">文件名: src/main.rs</span>

```rust
use std::process;

fn main() {
    let args: Vec<String> = env::args().collect();

    let config = Config::new(&args).unwrap_or_else(|err| {
        println!("Problem parsing arguments: {}", err);
        process::exit(1);
    });

    // ...snip...
```

<span class="caption">示例 12-10：如果新建 `Config` 失敗則使用錯誤碼退出</span>

在上面的示例中，使用了一個之前沒有涉及到的方法：`unwrap_or_else`，它定義於標準庫的 `Result<T, E>` 上。使用 `unwrap_or_else` 可以進行一些自定義的非 `panic!` 的錯誤處理。當 `Result` 是 `Ok` 時，這個方法的行為類似於 `unwrap`：它返回 `Ok` 內部封裝的值。然而，當其值是 `Err` 時，該方法會調用一個 **閉包**（*closure*），也就是一個我們定義的作為參數傳遞給 `unwrap_or_else` 的匿名函數。第十三章會更詳細的介紹閉包。現在你需要理解的是 `unwrap_or_else` 會將 `Err` 的內部值，也就是示例 12-9 中增加的 `not enough arguments` 靜態字符串的情況，傳遞給閉包中位於兩道豎線間的參數 `err`。閉包中的代碼在其運行時可以使用這個 `err` 值。

我們新增了一個 `use` 行來從標準庫中導入 `process`。在錯誤的情況閉包中將被運行的代碼只有兩行：我們打印出了 `err` 值，接著調用了 `std::process::exit`。`process::exit` 會立即停止程序並將傳遞給它的數字作為退出狀態碼。這類似於示例 12-8 中使用的基於 `panic!` 的錯誤處理，除了不會再得到所有的額外輸出了。讓我們試試：

```text
$ cargo run
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.48 secs
     Running `target/debug/minigrep`
Problem parsing arguments: not enough arguments
```

非常好！現在輸出對於用戶來說就友好多了。

### 從 `main` 提取邏輯

現在我們完成了配置解析的重構：讓我們轉向程序的邏輯。正如 「二進制項目的關注分離」 部分所展開的討論，我們將提取一個叫做 `run` 的函數來存放目前 `main `函數中不屬於設置配置或處理錯誤的所有邏輯。一旦完成這些，`main` 函數將簡明的足以通過觀察來驗證，而我們將能夠為所有其他邏輯編寫測試。

示例 12-11 展示了提取出來的 `run` 函數。目前我們只進行小的增量式的提取函數的改進。我們仍將在 *src/main.rs* 中定義這個函數：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    // ...snip...

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    run(config);
}

fn run(config: Config) {
    let mut f = File::open(config.filename).expect("file not found");

    let mut contents = String::new();
    f.read_to_string(&mut contents)
        .expect("something went wrong reading the file");

    println!("With text:\n{}", contents);
}

// ...snip...
```

<span class="caption">示例 12-11：提取 `run` 函數來包含剩餘的程序邏輯</span>

現在 `run` 函數包含了 `main` 中從讀取文件開始的剩餘的所有邏輯。`run` 函數抓取一個 `Config` 實例作為參數。

#### 從 `run` 函數中返回錯誤

通過將剩餘的邏輯分離進 `run` 函數而不是留在 `main` 中，就可以像示例 12-9 中的 `Config::new` 那樣改進錯誤處理。不再通過 `expect` 允許程序 panic，`run` 函數將會在出錯時返回一個 `Result<T, E>`。這讓我們進一步以一種對用戶友好的方式統一 `main` 中的錯誤處理。示例 12-12 展示了 `run` 簽名和函數體中的改變：

<span class="filename">文件名: src/main.rs</span>

```rust
use std::error::Error;

// ...snip...

fn run(config: Config) -> Result<(), Box<Error>> {
    let mut f = File::open(config.filename)?;

    let mut contents = String::new();
    f.read_to_string(&mut contents)?;

    println!("With text:\n{}", contents);

    Ok(())
}
```

<span class="caption">示例 12-12：修改 `run` 函數返回 `Result`</span>
 `Result<(), Box<Error>>`。之前這個函數返回 unit 類型 `()`，現在它仍然保持作為 `Ok` 時的返回值。

對於錯誤類型，使用了 **trait 物件** `Box<Error>`（在開頭使用了 `use` 語句將 `std::error::Error` 引入作用域）。第十七章會涉及 trait 物件。目前只需知道 `Box<Error>` 意味著函數會返回實現了 `Error` trait 的類型，不過無需指定具體將會返回的值的類型。這提供了在不同的錯誤場景可能有不同類型的錯誤返回值的靈活性。

第二個改變是去掉了 `expect` 調用並替換為第九章講到的 `?`。不同於遇到錯誤就 `panic!`，這會從函數中返回錯誤值並讓調用者來處理它。

第三個修改是現在成功時這個函數會返回一個 `Ok` 值。因為 `run` 函數簽名中聲明成功類型返回值是 `()`，這意味著需要將 unit 類型值包裝進 `Ok` 值中。`Ok(())` 一開始看起來有點奇怪，不過這樣使用 `()` 是表明我們調用 `run` 只是為了它的副作用的慣用方式；它並沒有返回什麼有意義的值。

上述代碼能夠編譯，不過會有一個警告：

```text
warning: unused result which must be used, #[warn(unused_must_use)] on by
default
  --> src/main.rs:39:5
   |
39 |     run(config);
   |     ^^^^^^^^^^^^
```

Rust 提示我們的代碼忽略了 `Result` 值，它可能表明這裡存在一個錯誤。雖然我們沒有檢查這裡是否有一個錯誤，而編譯器提醒我們這裡應該有一些錯誤處理代碼！現在就讓我們修正他們。

#### 處理 `main` 中 `run` 返回的錯誤

我們將檢查錯誤並使用類似示例 12-10 中 `Config::new` 處理錯誤的技術來處理他們，不過有一些細微的不同：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    // ...snip...

    println!("Searching for {}", config.query);
    println!("In file {}", config.filename);

    if let Err(e) = run(config) {
        println!("Application error: {}", e);

        process::exit(1);
    }
}
```

我們使用 `if let` 來檢查 `run` 是否返回一個 `Err` 值，不同於 `unwrap_or_else`，並在出錯時調用 `process::exit(1)`。`run` 並不返回像 `Config::new` 返回的 `Config` 實例那樣需要 `unwrap` 的值。因為 `run` 在成功時返回 `()`，而我們只關心發現一個錯誤，所以並不需要 `unwrap_or_else` 來返回未封裝的值，因為它只會是 `()`。

不過兩個例子中 `if let` 和 `unwrap_or_else` 的函數體都一樣：打印出錯誤並退出。

### 將代碼拆分到庫 crate

現在項目看起來好多了！現在我們將要拆分 *src/main.rs* 並將一些代碼放入 *src/lib.rs*，這樣就能測試他們並擁有一個擁有更少功能的 `main` 函數。

讓我們將如下代碼片段從 *src/main.rs* 移動到新文件 *src/lib.rs* 中：

- `run` 函數定義
- 相關的 `use` 語句
- `Config` 的定義
- `Config::new` 函數定義

現在 *src/lib.rs* 的內容應該看起來像示例 12-13（為了簡潔省略了函數體）：

<span class="filename">文件名: src/lib.rs</span>

```rust
use std::error::Error;
use std::fs::File;
use std::io::prelude::*;

pub struct Config {
    pub query: String,
    pub filename: String,
}

impl Config {
    pub fn new(args: &[String]) -> Result<Config, &'static str> {
        // ...snip...
    }
}

pub fn run(config: Config) -> Result<(), Box<Error>> {
    // ...snip...
}
```

<span class="caption">示例 12-13：將 `Config` 和 `run` 移動到 *src/lib.rs*</span>

這裡使用了公有的 `pub`：在 `Config`、其字段和其 `new`方法，以及 `run` 函數上。現在我們有了一個擁有可以測試的公有 API 的庫 crate 了。

現在需要在 *src/main.rs* 中使用 `extern crate greprs` 將移動到 *src/lib.rs* 的代碼引入二進制 crate 的作用域。接著我們將增加一個 `use greprs::Config` 行將 `Config` 類型引入作用域，並使用庫 crate 的名稱作為 `run` 函數的前綴，如示例 12-14 所示：

<span class="filename">Filename: src/main.rs</span>

```rust
extern crate minigrep;

use std::env;
use std::process;

use minigrep::Config;

fn main() {
    // ...snip...
    if let Err(e) = minigrep::run(config) {
        // ...snip...
    }
}
```

<span class="caption">示例 12-14：將 `minigrep` crate 引入 *src/main.rs* 的作用域</span>

為了將庫 crate 引入二進制 crate，我們使用 `extern crate minigrep`。接著增加 `use minigrep::Config` 將 `Config` 類型引入作用域，並使用 crate 名作為 `run` 函數的前綴。通過這些重構，所有功能應該能夠聯繫在一起並運行了。運行 `cargo run` 來確保一切都正確的銜接在一起。

哇哦！這可有很多的工作，不過我們為將來成功打下了基礎。現在處理錯誤將更容易，同時代碼也更加模組化。從現在開始幾乎所有的工作都將在 *src/lib.rs* 中進行。

讓我們利用這些新創建的模組的優勢來進行一些在舊代碼中難以展開的工作，他們在新代碼中卻很簡單：編寫測試！