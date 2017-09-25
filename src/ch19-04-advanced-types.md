## 高級類型

> [ch19-04-advanced-types.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch19-04-advanced-types.md)
> <br>
> commit e084e1773667c8eae28d9aab6d4939348eec0092

Rust 的類型系統有一些我們曾經提到或用到但沒有討論過的功能。我們從有關 trait 的 newtype 模式開始討論；首先從一個關於為什麼 newtype 與類型一樣有用的更寬泛的討論開始。接著會轉向類型別名（type aliases），一個類似於 newtype 但有著稍微不同的語義的功能。我們還會討論 `!` 類型和動態大小類型。

### 為了類型安全和抽象而使用 newtype 模式

在「高級 trait」部分最後開始的 newtype 模式的討論中，我們以一個包含一個封裝了某類型的字段的元組結構體創建了一個新類型，這對於靜態的確保其值不被混淆也是有幫助的，並且它經常用來表示一個值的單元。實際上列表 19-26 中已有一個例子：`Millimeters` 和 `Meters` 結構體都將 `u32` 值封裝進了新類型。如果編寫了一個有 `Millimeters` 類型參數的函數，不小心使用 `Meters` 或普通的 `u32` 值來調用該函數的程序是不能編譯的。

另一個使用 newtype 模式的原因是用來抽象掉一些類型的實現細節：例如，封裝類型可以暴露出與直接使用其內部私有類型時所不同的 API，以便限制其功能。新類型也可以隱藏其內部的泛型類型。例如，可以提供一個封裝了 `HashMap<i32, String>` 的 `People` 類型，用來儲存人名以及相應的 ID。使用 `People` 的代碼只需與提供的公有 API 交互即可，比如向 `People` 集合增加名字字符串的方法，這樣這些代碼就無需知道在內部我們將一個 `i32` ID 賦予了這個名字了。newtype 模式是一種實現第十七章所討論的隱藏實現細節的封裝的輕量級方法。

### 類型別名用來創建同義類型

newtype 模式涉及到創建新結構體來作為新的、單獨的類型。Rust 還提供了聲明**類型別名**（*type alias*）的能力，使用 `type` 關鍵字來給予現有類型另一個名字。例如，可以像這樣創建 `i32` 的別名 `Kilometers`：

```rust
type Kilometers = i32;
```

這意味著 `Kilometers` 是 `i32` 的**同義詞**（*synonym*）；不同於列表 19-26 中創建的 `Millimeters` 和 `Meters` 類型。`Kilometers` 不是一個新的、單獨的類型。`Kilometers` 類型的值將被完全當作 `i32` 類型值來對待：

```rust
type Kilometers = i32;

let x: i32 = 5;
let y: Kilometers = 5;

println!("x + y = {}", x + y);
```

因為 `Kilometers` 是 `i32` 的別名，他們是同一類型。可以將 `i32` 與 `Kilometers` 相加，可以將 `Kilometers` 傳遞給抓取 `i32` 參數的函數。但無法獲得上一部分討論的 newtype 模式所提供的類型檢查的好處。

類型別名的主要用途是減少重複。例如，可能會有這樣很長的類型：

```rust
Box<FnOnce() + Send + 'static>
```

在函數簽名或類型註解中每次都書寫這個類型將是枯燥且易於出錯的。想像一下如列表 19-31 這樣全是如此代碼的項目：

```rust
let f: Box<FnOnce() + Send + 'static> = Box::new(|| println!("hi"));

fn takes_long_type(f: Box<FnOnce() + Send + 'static>) {
    // ...
}

fn returns_long_type() -> Box<FnOnce() + Send + 'static> {
    // ...
#     Box::new(|| ())
}
```

<span class="caption">列表 19-31：在大部分地方使用名稱很長的類型</span>

類型別名通過減少項目中重複代碼的數量來使其更加易於控制。這裡我們為這個冗長的類型引入了一個叫做 `Thunk` 的別名，這樣就可以如列表 19-32 所示將所有使用這個類型的地方替換為更短的 `Thunk`：

```rust
type Thunk = Box<FnOnce() + Send + 'static>;

let f: Thunk = Box::new(|| println!("hi"));

fn takes_long_type(f: Thunk) {
    // ...
}

fn returns_long_type() -> Thunk {
    // ...
#     Box::new(|| ())
}
```

<span class="caption">列表 19-32：引入類型別名 `Thunk` 來減少重複</span>

這樣就讀寫起來就容易多了！為類型別名選擇一個好名字也可以幫助你表達意圖（單詞 *thunk* 表示會在之後被計算的代碼，所以這是一個存放閉包的合適的名字）。

類型別名的另一個常用用法是與 `Result<T, E>` 結合。考慮一下標準庫中的 `std::io` 模組。I/O 操作通常會返回一個 `Result<T, E>`，因為這些操作可能會失敗。`std::io::Error` 結構體代表了所有可能的 I/O 錯誤。`std::io` 中大部分函數會返回 `Result<T, E>`，其中 `E` 是 `std::io::Error`，比如 `Write` trait 中的這些函數：

```rust
use std::io::Error;
use std::fmt;

pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize, Error>;
    fn flush(&mut self) -> Result<(), Error>;

    fn write_all(&mut self, buf: &[u8]) -> Result<(), Error>;
    fn write_fmt(&mut self, fmt: fmt::Arguments) -> Result<(), Error>;
}
```

這裡出現了很多的 `Result<..., Error>`。為此，`std::io` 有這個類型別名聲明：

```rust
type Result<T> = Result<T, std::io::Error>;
```

因為這位於 `std::io` 中，可用的完全限定的別名是`std::io::Result<T>`；也就是說，`Result<T, E>` 中 `E` 放入了 `std::io::Error`。`Write` trait 中的函數最終看起來像這樣：

```rust
pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;

    fn write_all(&mut self, buf: &[u8]) -> Result<()>;
    fn write_fmt(&mut self, fmt: Arguments) -> Result<()>;
}
```

類型別名在兩個方面有幫助：易於編寫**並**在整個 `std::io` 中提供了一致的接口。因為這是一個別名，它只是另一個 `Result<T, E>`，這意味著可以在其上使用 `Result<T, E>` 的任何方法，以及像 `?` 這樣的特殊語法。

### 從不返回的 `!`，never type

Rust 有一個叫做 `!` 的特殊類型。在類型理論術語中，它被稱為 *empty type*，因為它沒有值。我們更傾向於稱之為 *never type*。這個名字描述了它的作用：在函數從不返回的時候充當返回值。例如：

```rust
fn bar() -> ! {
```

這讀作「函數 `bar` 從不返回」，而從不返回的函數被稱為**發散函數**（*diverging functions*）。不能創建 `!` 類型的值，所以 `bar` 也不可能返回。一個不能創建值的類型有什麼用呢？如果你回想一下第二章，曾經有一些看起來像這樣的代碼，如列表 19-33 所重現的：

```rust
# let guess = "3";
# loop {
let guess: u32 = match guess.trim().parse() {
    Ok(num) => num,
    Err(_) => continue,
};
# break;
# }
```

<span class="caption">列表 19-33：`match` 語句和一個以 `continue` 結束的分支</span>

當時我們忽略了一些代碼細節。在第六章中，我們學習了 `match` 的分支必須返回相同的類型。如下代碼不能工作：

```rust
let guess = match guess.trim().parse()  {
    Ok(_) => 5,
    Err(_) => "hello",
}
```

這裡的 `guess` 會是什麼類型呢？它必須既是整型也是字符串，而 Rust 要求 `guess` 只能是一個類型。那麼 `continue` 返回了什麼呢？為什麼列表 19-33 中會允許一個分支返回 `u32` 而另一個分支卻以 `continue` 結束呢？

正如你可能猜到的，`continue` 的值是 `!`。也就是說，當 Rust 要計算 `guess` 的類型時，它查看這兩個分支。前者是 `u32` 值，而後者是 `!` 值。因為 `!` 並沒有一個值，Rust 認為這是可行的，並決定 `guess` 的類型是 `u32`。描述 `!` 的行為的正式方式是 never type 可以與其他任何類型聯合。允許 `match` 的分支以 `continue` 結束是因為 `continue` 並不真正返回一個值；相反它把控制權交回上層循環，所以在 `Err` 的情況，事實上並未對 `guess` 賦值。

never type 的另一個用途是 `panic!`。還記得 `Option<T>` 上的 `unwrap` 函數嗎？它產生一個值或 panic。這裡是它的定義：

```rust
impl<T> Option<T> {
    pub fn unwrap(self) -> T {
        match self {
            Some(val) => val,
            None => panic!("called `Option::unwrap()` on a `None` value"),
        }
    }
}
```

這裡與列表 19-33 中的 `match` 發生的相同的情況：我們知道 `val` 是 `T` 類型，`panic!` 是 `!` 類型，所以整個 `match` 表達式的結果是 `T` 類型。這能工作是因為 `panic!` 並不產生一個值：它終止程序。對於 `None` 的情況，`unwrap` 並不返回一個值，所以這些代碼是有效。

最後的表達式在 `loop` 中使用了 `!` 類型：

```rust
print!("forever ");

loop {
    print!("and ever ");
}
```

這裡，循環永遠也不結束，所以此表達式的值是 `!`。但是如果引入 `break` 這就不為真了，因為循環在執行到 `break` 後就會終止。

### 動態大小類型和 `Sized` trait

因為 Rust 需要知道類似內存佈局之類的信息，在其類型系統的一個特定的角落可能令人迷惑，這就是**動態大小類型**（*dynamically sized types*）的概念。這有時被稱為「DST」 或 「unsized types」，這些類型允許我們處理只有在運行時才知道大小的類型。

讓我們深入研究一個貫穿本書都在使用的動態大小類型的細節：`str`。沒錯，不是 `&str`，而是 `str` 本身。`str` 是一個 DST；直到運行時我們都不知道字符串有多長。因為不能知道大小，也就不能創建 `str` 類型的變數，也不能抓取 `str` 類型的參數。考慮一下這些代碼，他們不能工作：

```rust
let s1: str = "Hello there!";
let s2: str = "How's it going?";
```

這兩個 `str` 值需要有完全一樣的內存佈局，不過他們卻有不同的長度：`s1` 需要 12 字節來存儲，而 `s2` 需要 15 字節。這樣就是為為什麼不可能創建一個存放動態大小類型的變數。

那麼該怎麼辦呢？好吧，在這個例子中你已經知道了答案：`s1` 和 `s2` 的類型是 `&str` 而不是 `str`。如果你回想第四章，我們這樣描述 `&str`：

> ... 這是一個字符串內部位置和其所引用的元素的數量的引用。

所以雖然 `&T` 是一個儲存了 `T` 所在的內存位置的單個值，`&str` 則是**兩個**值：`str` 的地址和其長度。這樣，`&str` 就有了一個在編譯時可以知道的大小：它是 `usize` 長度的兩倍。也就是說，我們總是知道 `&str` 的大小，而無論其引用的字符串是多長。這裡是 Rust 中動態大小類型的常規用法：他們有一些額外的元信息來儲存動態信息的大小。這引出了動態大小類型的黃金規則：必須將動態大小類型的值置於某種指針之後。

<!-- Note for Carol: `Rc<str>` is only in an accepted RFC right now, check on
its progress and pull this out if it's not going to be stable by Oct -->

雖然我們總是說 `&str`，但是可以將 `str` 與所有類型的指針結合：比如 `Box<str>` 或 `Rc<str>`。事實上，之前已經見過了，不過是另一個動態大小類型：trait。每一個 trait 都是一個可以通過 trait 名稱來引用的動態大小類型。在第十七章中，我們提到了為了將 trait 用於 trait 物件，必須將他們放入指針之後，比如 `&Trait` 或 `Box<Trait>`（`Rc<Trait>` 也可以）。

#### `Sized` trait

為了處理 DST，Rust 有一個 trait 來決定一個類型的大小是否在編譯時可知，這就是 `Sized`。這個 trait 自動為編譯器在編譯時就知道大小的類型實現。另外，Rust 隱式的為每一個泛型函數增加了 `Sized` bound。也就是說，對於如下泛型函數定義：

```rust
fn generic<T>(t: T) {
```

實際上被當作如下處理：

```rust
fn generic<T: Sized>(t: T) {
```

泛型函數預設只能用於在編譯時已知大小的類型。然而可以使用如下特殊語法來放鬆這個限制：

```rust
fn generic<T: ?Sized>(t: &T) {
```

`?Sized` trait bound 與 `Sized` 相對；也就是說，它可以讀作「`T` 可能是也可能不是 `Sized` 的」。這個語法只能用於 `Sized` ，而不是其他 trait。

另外注意我們將 `t` 參數的類型從 `T` 變為了 `&T`：因為其類型可能不是 `Sized` 的，所以需要將其置於某種指針之後。在這個例子中選擇了引用。

接下來，讓我們討論一下函數和閉包！