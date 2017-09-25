## 數據類型

> [ch03-02-data-types.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch03-02-data-types.md)
> <br>
> commit 1d4f34661d4b13ecda4885d496e1b84cdbcbed31

在 Rust 中，任何值都屬於一種明確的 **類型**（*type*），這告訴了 Rust 它被指定了何種數據，以便明確其處理方式。我們將分兩部分探討一些內建類型：標量（scalar）和組合（compound）。

Rust 是 **靜態類型**（*statically typed*）語言，也就是說在編譯時就必須知道所有變數的類型，這一認知將貫穿整個章節，請在頭腦中明確。通過值的形式及其使用方式，編譯器通常可以推斷出我們想要用的類型。多種類型均有可能時，比如第二章中使用 `parse` 將 `String` 轉換為數字時，必須增加類型註解，像這樣：

```rust
let guess: u32 = "42".parse().expect("Not a number!");
```

這裡如果不添加類型註解，Rust 會顯示如下錯誤，這說明編譯器需要更多信息，來瞭解我們想要的類型：

```text
error[E0282]: unable to infer enough type information about `_`
 --> src/main.rs:2:9
  |
2 |     let guess = "42".parse().expect("Not a number!");
  |         ^^^^^ cannot infer type for `_`
  |
  = note: type annotations or generic parameter binding required
```

在我們討論各種數據類型時，你會看到多樣的類型註解。

### 標量類型

**標量**（*scalar*）類型代表一個單獨的值。Rust 有四種基本的標量類型：整型、浮點型、布爾類型和字符類型。你可能在其他語言中見過他們，不過讓我們深入瞭解他們在 Rust 中時如何工作的。

#### 整型

**整數** 是一個沒有小數部分的數字。我們在這一章的前面使用過 `u32` 類型。該類型聲明指示，u32 關聯的值應該是一個佔據 32 比特位的無符號整數（因為這個`u`，與`i`代表的有符號相對）。表格 3-1 展示了 Rust 內建的整數類型。每一種變體的有符號和無符號列（例如，*i8*）可以用來聲明對應的整數值。

<span class="caption">表格 3-1: Rust 中的整型</span>

| 長度 | 有符號 | 無符號 |
|--------|--------|----------|
| 8-bit  | i8     | u8       |
| 16-bit | i16    | u16      |
| 32-bit | i32    | u32      |
| 64-bit | i64    | u64      |
| arch   | isize  | usize    |

每一種變體都可以是有符號或無符號的，並有一個明確的大小。有符號和無符號代表數字能否為負值；換句話說，數字是否需要有一個符號（有符號數），或者永遠為正而不需要符號（無符號數）。這有點像在紙上書寫數字：當需要考慮符號的時候，數字以加號或減號作為前綴；然而，可以安全地假設為正數時，加號前綴通常省略。有符號數以二進制補碼形式（two's complement representation）存儲（如果你不清楚這是什麼，可以在網上搜索；對其的解釋超出了本書的範疇）。

每一個有符號的變體可以儲存包含從 -(2<sup>n - 1</sup>) 到 2<sup>n - 1</sup> - 1 在內的數字，這裡 `n` 是變體使用的位數。所以 `i8` 可以儲存從 -(2<sup>7</sup>) 到 2<sup>7</sup> - 1 在內的數字，也就是從 -128 到 127。無符號的變體可以儲存從 0 到 2<sup>n</sup> - 1 的數字，所以 `u8` 可以儲存從 0 到 2<sup>8</sup> - 1 的數字，也就是從 0 到 255。

另外，`isize` 和 `usize` 類型依賴運行程序的計算機架構：64 位架構上他們是 64 位的， 32 位架構上他們是 32 位的。

可以使用表格 3-2 中的任何一種形式編寫數字字面值。注意除字節以外的其它字面值允許使用類型後綴，例如 `57u8`，允許使用 `_` 做為分隔符以方便讀數。

<span class="caption">表格 3-2: Rust 中的整型字面值</span>

| 數字字面值  | 例子       |
|------------------|---------------|
| Decimal          | `98_222`      |
| Hex              | `0xff`        |
| Octal            | `0o77`        |
| Binary           | `0b1111_0000` |
| Byte (`u8` only) | `b'A'`        |

那麼該使用哪種類型的數字呢？如果拿不定主意，Rust 的默認類型通常就很好，數字類型默認是 `i32`：它通常是最快的，甚至在 64 位系統上也是。`isize` 或 `usize` 主要作為集合的索引。

#### 浮點型

Rust 同樣有兩個主要的 **浮點數**（*floating-point numbers*）類型，`f32` 和 `f64`，它們是帶小數點的數字，分別佔 32 位和 64 位比特。默認類型是 `f64`，因為它與 `f32` 速度差不多，然而精度更高。在 32 位系統上也能夠使用 `f64`，不過比使用 `f32` 要慢。多數情況下，一開始以潛在的性能損耗換取更高的精度是合理的；如果覺得浮點數的大小是個麻煩，你應該以性能測試作為決策依據。

這是一個展示浮點數的實例：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let x = 2.0; // f64

    let y: f32 = 3.0; // f32
}
```

浮點數採用 IEEE-754 標準表示。`f32` 是單精度浮點數，`f64` 是雙精度浮點數。

#### 數字運算符

Rust 支持所有數字類型常見的基本數學運算操作：加法、減法、乘法、除法以及餘數。如下代碼展示了如何使用一個 `let` 語句來使用他們：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    // addition
    let sum = 5 + 10;

    // subtraction
    let difference = 95.5 - 4.3;

    // multiplication
    let product = 4 * 30;

    // division
    let quotient = 56.7 / 32.2;

    // remainder
    let remainder = 43 % 5;
}
```

這些語句中的每個表達式使用了一個數學運算符並計算出了一個值，他們綁定到了一個變數。附錄 B 包含了一個 Rust 提供的所有運算符的列表。

#### 布爾型

正如其他大部分編程語言一樣，Rust 中的布爾類型有兩個可能的值：`true` 和 `false`。Rust 中的布爾類型使用 `bool` 表示。例如：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let t = true;

    let f: bool = false; // with explicit type annotation
}
```

使用布爾值的主要場景是條件表達式，例如 `if` 表達式。在 「控制流」（「Control Flow」）部分將講到`if`表達式在 Rust 中如何工作。

#### 字符類型

目前為止只使用到了數字，不過 Rust 也支持字符。Rust 的 `char` 類型是大部分語言中基本字母字符類型，如下代碼展示了如何使用它：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
   let c = 'z';
   let z = 'ℤ';
   let heart_eyed_cat = '😻';
}
```

Rust 的 `char` 類型代表了一個 Unicode 標量值（Unicode Scalar Value），這意味著它可以比 ASCII 表示更多內容。拼音字母（Accented letters），中文/日文/漢語等象形文字，emoji（絵文字）以及零長度的空白字符對於 Rust `char`類型都是有效的。Unicode 標量值包含從 `U+0000` 到 `U+D7FF` 和 `U+E000` 到 `U+10FFFF` 之間的值。不過，「字符」 並不是一個 Unicode 中的概念，所以人直覺上的 「字符」 可能與 Rust 中的 `char` 並不符合。第八章的 「字符串」 部分將詳細討論這個主題。

### 組合類型

**組合類型**（*Compound types*）可以將多個其他類型的值組合成一個類型。Rust 有兩個原生的組合類型：元組（tuple）和陣列（array）。

#### 將值組合進元組

元組是一個將多個其他類型的值組合進一個組合類型的主要方式。

我們使用一個括號中的逗號分隔的值列表來創建一個元組。元組中的每一個位置都有一個類型，而且這些不同值的類型也不必是相同的。這個例子中使用了額外的可選類型註解：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);
}
```

`tup` 變數綁定了整個元組，因為元組被認為是一個單獨的組合元素。為了從元組中獲取單個的值，可以使用模式匹配（pattern matching）來解構（destructure）元組，像這樣：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let tup = (500, 6.4, 1);

    let (x, y, z) = tup;

    println!("The value of y is: {}", y);
}
```

程序首先創建了一個元組並綁定到 `tup` 變數上。接著使用了 `let` 和一個模式將 `tup` 分成了三個不同的變數，`x`、`y` 和 `z`。這叫做 *解構*（*destructuring*），因為它將一個元組拆成了三個部分。最後，程序打印出了 `y` 的值，也就是 `6.4`。

除了使用模式匹配解構之外，也可以使用點號（`.`）後跟值的索引來直接訪問他們。例如：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let x: (i32, f64, u8) = (500, 6.4, 1);

    let five_hundred = x.0;

    let six_point_four = x.1;

    let one = x.2;
}
```

這個程序創建了一個元組，`x`，並接著使用索引為每個元素創建新變數。跟大多數編程語言一樣，元組的第一個索引值是 0。

#### 陣列

另一個獲取一個多個值集合的方式是 **陣列**（*array*）。與元組不同，陣列中的每個元素的類型必須相同。Rust 中的陣列與一些其他語言中的陣列不同，因為 Rust 中的陣列是固定長度的：一旦聲明，他們的長度不能增長或縮小。

Rust 中陣列的值位於中括號中的逗號分隔的列表中：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let a = [1, 2, 3, 4, 5];
}
```

陣列在需要在棧（stack）而不是在堆（heap）上為數據分配空間時（第四章將討論棧與堆的更多內容），或者是想要確保總是有固定數量的元素時十分有用。雖然它並不如 vector 類型那麼靈活。vector 類型是標準庫提供的一個 **允許** 增長和縮小長度的類似陣列的集合類型。當不確定是應該使用陣列還是 vector 的時候，你可能應該使用 vector：第八章會詳細討論 vector。

一個你可能想要使用陣列而不是 vector 的例子是當程序需要知道一年中月份的名字時。程序不大可能回去增加或減少月份，這時你可以使用陣列因為我們知道它總是含有 12 個元素：

```rust
let months = ["January", "February", "March", "April", "May", "June", "July",
              "August", "September", "October", "November", "December"];
```

##### 訪問陣列元素

陣列是一整塊分配在棧上的內存。可以使用索引來訪問陣列的元素，像這樣：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let a = [1, 2, 3, 4, 5];

    let first = a[0];
    let second = a[1];
}
```

在這個例子中，叫做 `first` 的變數的值是 `1`，因為它是陣列索引 `[0]` 的值。`second` 將會是陣列索引 `[1]` 的值 `2`。

##### 無效的陣列元素訪問

如果我們訪問陣列結尾之後的元素會發生什麼呢？比如我們將上面的例子改為如下：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let a = [1, 2, 3, 4, 5];
    let index = 10;

    let element = a[index];

    println!("The value of element is: {}", element);
}
```

使用 `cargo run` 運行代碼後會產生如下結果：

```text
$ cargo run
   Compiling arrays v0.1.0 (file:///projects/arrays)
     Running `target/debug/arrays`
thread '<main>' panicked at 'index out of bounds: the len is 5 but the index is
 10', src/main.rs:6
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

編譯並沒有產生任何錯誤，不過程序會導致一個 **運行時**（*runtime*）錯誤並且不會成功退出。當嘗試用索引訪問一個元素時，Rust 會檢查指定的索引是否小於陣列的長度。如果索引超出了陣列長度，Rust 會 *panic*，這是 Rust 中的術語，它用於程序因為錯誤而退出的情況。

這是第一個在實戰中遇到的 Rust 安全原則的例子。在很多底層語言中，並沒有進行這類檢查，這樣當提供了一個不正確的索引時，就會訪問無效的內存。Rust 通過立即退出而不是允許內存訪問並繼續執行來使你免受這類錯誤困擾。第九章會討論更多 Rust 的錯誤處理。