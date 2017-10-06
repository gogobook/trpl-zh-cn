## 函數如何工作

> [ch03-03-how-functions-work.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch03-03-how-functions-work.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

函數在 Rust 代碼中應用廣泛。你已經見過一個語言中最重要的函數：`main` 函數，它是很多程序的入口點。你也見過了 `fn` 關鍵字，它用來聲明新函數。

Rust 代碼使用 *snake case* 作為函數和變數名稱的規範風格。在 snake case 中，所有字母都是小寫並使用下劃線分隔單詞。這裡是一個包含函數定義的程序的例子：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    println!("Hello, world!");

    another_function();
}

fn another_function() {
    println!("Another function.");
}
```

Rust 中的函數定義以 `fn` 開始並在函數名後跟一對括號。大括號告訴編譯器哪裡是函數體的開始和結尾。

可以使用定義過的函數名後跟括號來調用任意函數。因為 `another_function` 已經在程序中定義過了，它可以在 `main` 函數中被調用。注意，源碼中 `another_function` 在 `main` 函數 **之後** 被定義；也可以在其之前定義。Rust 不關心函數定義於何處，只要他們被定義了。

讓我們開始一個叫做 *functions* 的新二進制項目來進一步探索函數。將上面的 `another_function` 例子寫入 *src/main.rs* 中並運行。你應該會看到如下輸出：

```text
$ cargo run
   Compiling functions v0.1.0 (file:///projects/functions)
     Running `target/debug/functions`
Hello, world!
Another function.
```

代碼在 `main` 函數中按照他們出現的順序被執行。首先，打印 「Hello, world!」 信息，接著 `another_function` 被調用並打印它的信息。

### 函數參數

函數也可以被定義為擁有 **參數**（*parameters*），他們是作為函數簽名一部分的特殊變數。當函數擁有參數時，可以為這些參數提供具體的值。技術上講，這些具體值被稱為參數（ *arguments*），不過通常的習慣是傾向於在函數定義中的變數和調用函數時傳遞的具體值都可以用 「parameter」 和 「argument」 而不加區別。

如下被重寫的 `another_function` 版本展示了 Rust 中參數是什麼樣的：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    another_function(5);
}

fn another_function(x: i32) {
    println!("The value of x is: {}", x);
}
```

嘗試運行程序，將會得到如下輸出：

```text
$ cargo run
   Compiling functions v0.1.0 (file:///projects/functions)
     Running `target/debug/functions`
The value of x is: 5
```

`another_function` 的聲明有一個叫做 `x` 的參數。`x` 的類型被指定為 `i32`。當 `5` 被傳遞給 `another_function` 時，`println!` 巨集將 `5` 放入格式化字符串中大括號的位置。

在函數簽名中，**必須** 聲明每個參數的類型。這是 Rust 設計中一個經過慎重考慮的決定：要求在函數定義中提供類型註解意味著編譯器再也不需要在別的地方要求你註明類型就能知道你的意圖。

當一個函數有多個參數時，使用逗號隔開他們，像這樣：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    another_function(5, 6);
}

fn another_function(x: i32, y: i32) {
    println!("The value of x is: {}", x);
    println!("The value of y is: {}", y);
}
```

這個例子創建了一個有兩個參數的函數，都是 `i32` 類型的。函數打印出了這兩個參數的值。注意函數參數並不一定都是相同類型的，這個例子中他們只是碰巧相同罷了。

嘗試運行代碼。使用上面的例子替換當前 *function* 項目的 *src/main.rs* 文件，並 `cargo run` 運行它：

```text
$ cargo run
   Compiling functions v0.1.0 (file:///projects/functions)
     Running `target/debug/functions`
The value of x is: 5
The value of y is: 6
```

因為我們使用 `5` 作為 `x` 的值和 `6` 作為 `y` 的值來調用函數，這兩個字符串和他們的值並被打印出來。

### 函數體

函數體由一系列的語句和一個可選的表達式構成。目前為止，我們只涉及到了沒有結尾表達式的函數，不過我們見過表達式作為了語句的一部分。因為 Rust 是一個基於表達式（expression-based）的語言，這是一個需要理解的（不同於其他語言）重要區別。其他語言並沒有這樣的區別，所以讓我們看看語句與表達式有什麼區別以及他們是如何影響函數體的。

### 語句與表達式

我們已經用過語句與表達式了。**語句**（*Statements*）是執行一些操作但不返回值的指令。表達式（*Expressions*）計算並產生一個值。讓我們看看一些例子：

使用 `let` 關鍵字創建變數並綁定一個值是一個語句。在示例 3-3 中，`let y = 6;` 是一個語句：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let y = 6;
}
```

<span class="caption">示例 3-3：包含一個語句的 `main` 函數定義</span>

函數定義也是語句；上面整個例子本身就是一個語句。

語句並不返回值。因此，不能把`let`語句賦值給另一個變數，比如下面的例子嘗試做的：

<span class="filename">Filename: src/main.rs</span>

```rust
fn main() {
    let x = (let y = 6);
}
```

當運行這個程序，會得到如下錯誤：

```text
$ cargo run
   Compiling functions v0.1.0 (file:///projects/functions)
error: expected expression, found statement (`let`)
 --> src/main.rs:2:14
  |
2 |     let x = (let y = 6);
  |              ^^^
  |
  = note: variable declaration using `let` is a statement
```

`let y = 6` 語句並不返回值，所以並沒有 `x` 可以綁定的值。這與其他語言不同，例如 C 和 Ruby，他們的賦值語句返回所賦的值。在這些語言中，可以這麼寫 `x = y = 6` 這樣 `x` 和 `y` 的值都是 `6`；這在 Rust 中可不行。

表達式計算出一些值，而且他們組成了其餘大部分你將會編寫的 Rust 代碼。考慮一個簡單的數學運算，比如 `5 + 6`，這是一個表達式並計算出值 `11`。表達式可以是語句的一部分：在示例 3-3 中有這個語句 `let y = 6;`，`6` 是一個表達式它計算出的值是 `6`。函數調用是一個表達式。巨集調用是一個表達式。我們用來創新建作用域的大括號（代碼塊），`{}`，也是一個表達式，例如：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let x = 5;

    let y = {
        let x = 3;
        x + 1
    };

    println!("The value of y is: {}", y);
}
```

這個表達式：

```rust
{
    let x = 3;
    x + 1
}
```

這個代碼塊的值是 `4`。這個值作為 `let` 語句的一部分被綁定到 `y` 上。注意結尾沒有分號的那一行，與大部分我們見過的代碼行不同。表達式並不包含結尾的分號。如果在表達式的結尾加上分號，他就變成了語句，這也就使其不返回一個值。在接下來的探索中記住 **函數和表達式都返回值** 就行了。

### 函數的返回值

可以向調用它的代碼返回值。並不對返回值命名，不過會在一個箭頭（`->`）後聲明它的類型。在 Rust 中，函數的返回值等同於函數體最後一個表達式的值。這是一個有返回值的函數的例子：

<span class="filename">文件名: src/main.rs</span>

```rust
fn five() -> i32 {
    5
}

fn main() {
    let x = five();

    println!("The value of x is: {}", x);
}
```

在函數 `five` 中並沒有函數調用、巨集、甚至也沒有 `let` 語句————只有數字 `5` 自身。這在 Rust 中是一個完全有效的函數。注意函數的返回值類型也被指定了，就是 `-> i32`。嘗試運行代碼；輸出應該看起來像這樣：

```text
$ cargo run
   Compiling functions v0.1.0 (file:///projects/functions)
     Running `target/debug/functions`
The value of x is: 5
```

函數 `five` 的返回值是 `5`，也就是為什麼返回值類型是 `i32`。讓我們仔細檢查一下這段代碼。這有兩個重要的部分：首先，`let x = five();` 這一行表明我們使用函數的返回值來初始化了一個變數。因為函數 `five` 返回 `5`，這一行與如下這行相同：

```rust
let x = 5;
```

其次，函數 `five` 沒有參數並定義了返回值類型，不過函數體只有單單一個 `5` 也沒有分號，因為這是我們想要返回值的表達式。讓我們看看另一個例子：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let x = plus_one(5);

    println!("The value of x is: {}", x);
}

fn plus_one(x: i32) -> i32 {
    x + 1
}
```

運行代碼會打印出 `The value of x is: 6`。如果在包含 `x + 1` 的那一行的結尾加上一個分號，把它從表達式變成語句後會怎樣呢？

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let x = plus_one(5);

    println!("The value of x is: {}", x);
}

fn plus_one(x: i32) -> i32 {
    x + 1;
}
```

運行代碼會產生一個錯誤，如下：

```text
error[E0308]: mismatched types
 --> src/main.rs:7:28
  |
7 |   fn plus_one(x: i32) -> i32 {
  |  ____________________________^
8 | |     x + 1;
9 | | }
  | |_^ expected i32, found ()
  |
  = note: expected type `i32`
             found type `()`
help: consider removing this semicolon:
 --> src/main.rs:8:10
  |
8 |     x + 1;
  |          ^
```

主要的錯誤信息，「mismatched types,」（類型不匹配），揭示了代碼的核心問題。函數`plus_one` 的定義說明它要返回一個 `i32`，不過語句並不返回一個值，這由那個空元組 `()` 表明。因此，這個函數返回了空元組 `()`，這與函數定義相矛盾並導致一個錯誤。在輸出中，Rust 提供了一個可能會對修正問題有幫助的信息：它建議去掉分號，這會修復這個錯誤。