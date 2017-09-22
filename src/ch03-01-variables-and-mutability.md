## 變量和可變性

> [ch03-01-variables-and-mutability.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch03-01-variables-and-mutability.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

第二章中提到過，變量默認是 **不可變**（*immutable*）的。這是利用 Rust 安全和簡單並發的優勢編寫代碼一大助力。不過，變量仍然有可變的選項。讓我們探討一下 Rust 擁抱不可變性的原因及方法，以及何時你不想使用不可變性。

當變量不可變時，意味著一旦值被綁定上一個名稱，你就不能改變這個值。作為說明，通過 `cargo new --bin variables` 在 *projects* 目錄生成一個叫做 *variables* 的新項目。

接著，在新建的 *variables* 目錄，打開 *src/main.rs* 並替換其代碼為如下：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let x = 5;
    println!("The value of x is: {}", x);
    x = 6;
    println!("The value of x is: {}", x);
}
```

保存並使用 `cargo run` 運行程序。應該會看到一個錯誤信息，如下輸出所示：

```text
error[E0384]: re-assignment of immutable variable `x`
 --> src/main.rs:4:5
  |
2 |     let x = 5;
  |         - first assignment to `x`
3 |     println!("The value of x is: {}", x);
4 |     x = 6;
  |     ^^^^^ re-assignment of immutable variable
```

這個例子展示了編譯器如何幫助你找出程序中的錯誤。雖然編譯錯誤令人沮喪，那也不過是說程序不能安全的完成你想讓它完成的工作；而 **不能** 說明你是不是一個好程序員！有經驗的 Rustacean 們一樣會遇到編譯錯誤。這些錯誤給出的原因是 `對不可變變量重新賦值`（`re-assignment of immutable variable`），因為我們嘗試對不可變變量 `x` 賦第二個值。

嘗試去改變預設為不可變的值，產生編譯錯誤是很重要的，因為這種情況可能導致 bug：如果代碼的一部分假設一個值永遠也不會改變，而另一部分代碼改變了它，第一部分代碼就有可能以不可預料的方式運行。不得不承認這種 bug 難以跟蹤，尤其是第二部分代碼只是 **有時** 改變其值。

Rust 編譯器保證，如果聲明一個值不會變，它就真的不會變。這意味著當閱讀和編寫代碼時，不需要記住如何以及哪裡可能會被改變，從而使得代碼易於推導。

不過可變性也是非常有用的。變量只是默認不可變；可以通過在變量名之前加 `mut` 來使其可變。除了使值可以改變之外，它向讀者表明了其他代碼將會改變這個變量的意圖。

例如，改變 *src/main.rs* 並替換其代碼為如下：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let mut x = 5;
    println!("The value of x is: {}", x);
    x = 6;
    println!("The value of x is: {}", x);
}
```

當運行這個程序，出現如下：

```text
$ cargo run
   Compiling variables v0.1.0 (file:///projects/variables)
     Running `target/debug/variables`
The value of x is: 5
The value of x is: 6
```

通過 `mut`，允許把綁定到 `x` 的值從 `5` 改成 `6`。在一些情況下，你會想要一個變量可變，因為相對只有不可變的風格更容易編寫。

除了避免 bug 外，還有多處需要權衡取捨。例如，使用大型數據結構時，適當地使變量可變，可能比複製和返回新分配的實例更快。對於較小的數據結構，總是創建新實例，採用更偏向函數式的風格編程，可能會使代碼更易理解，為可讀性而遭受性能懲罰或許值得。

### 變量和常量的區別

不允許改變值的變量，可能會使你想起另一個大部分編程語言都有的概念：**常量**（*constants*）。類似於不可變變量，常量也是綁定到一個名稱的不允許改變的值，不過常量與變量還是有一些區別。

首先，不允許對常量使用 `mut`：常量不光默認不能變，它總是不能變。

聲明常量使用 `const` 關鍵字而不是 `let`，而且 *必須* 註明值的類型。在下一部分，「數據類型」，涉及到類型和類型註解，現在無需關心這些細節，記住總是標註類型即可。

常量可以在任何作用域聲明，包括全局作用域，這在一個值需要被很多部分的代碼用到時很有用。

最後一個區別是常量只能用於常量表達式，而不能作為函數調用的結果，或任何其他只在運行時計算的值。

這是一個常量聲明的例子，它的名稱是 `MAX_POINTS`，值是 100,000。（Rust 的常量使用下劃線分隔的大寫字母的命名規範）：

```rust
const MAX_POINTS: u32 = 100_000;
```

常量在整個程序生命週期中都有效，位於它聲明的作用域之中。這使得常量可以作為多處代碼使用的全局範圍的值，例如一個遊戲中所有玩家可以獲取的最高分或者光速。

將用於整個程序的硬編碼的值聲明為常量對後來的維護者瞭解值的意義很用幫助。它也能將硬編碼的值彙總一處，為將來可能的修改提供方便。

### 隱藏（Shadowing）

如第二章 「猜猜看遊戲」 所講的，我們可以定義一個與之前變量重名的新變量，而新變量會 **隱藏** 之前的變量。Rustacean 們稱之為第一個變量被第二個 **隱藏** 了，這意味著使用這個變量時會看到第二個值。可以用相同變量名稱來隱藏它自己，以及重複使用 `let` 關鍵字來多次隱藏，如下所示：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let x = 5;

    let x = x + 1;

    let x = x * 2;

    println!("The value of x is: {}", x);
}
```

這個程序首先將 `x` 綁定到值 `5` 上。接著通過 `let x =` 隱藏 `x`，獲取原始值並加 `1` 這樣 `x` 的值就變成 `6` 了。第三個 `let` 語句也隱藏了 `x`，獲取之前的值並乘以 `2`，`x` 最終的值是 `12`。運行這個程序，它會有如下輸出：

```text
$ cargo run
   Compiling variables v0.1.0 (file:///projects/variables)
     Running `target/debug/variables`
The value of x is: 12
```

這與將變量聲明為 `mut` 是有區別的。因為除非再次使用 `let` 關鍵字，不小心嘗試對變量重新賦值會導致編譯時錯誤。我們可以用這個值進行一些計算，不過計算完之後變量仍然是不變的。

`mut` 與隱藏的另一個區別是，當再次使用 `let` 時，實際上創建了一個新變量，我們可以改變值的類型，從而復用這個名字。例如，假設程序請求用戶輸入空格來提供文本間隔，然而我們真正需要的是將輸入存儲成數字（多少個空格）：

```rust
let spaces = "   ";
let spaces = spaces.len();
```

這裡允許第一個 `spaces` 變量是字符串類型，而第二個 `spaces` 變量，它是一個恰巧與第一個變量同名的嶄新變量，是數字類型。隱藏使我們不必使用不同的名字，如 `spaces_str` 和 `spaces_num`；相反，我們可以復用 `spaces` 這個更簡單的名字。然而，如果嘗試使用 `mut`，如下所示：

```rust
let mut spaces = "   ";
spaces = spaces.len();
```

會導致一個編譯錯誤，因為改變一個變量的類型是不被允許的：

```text
error[E0308]: mismatched types
 --> src/main.rs:3:14
  |
3 |     spaces = spaces.len();
  |              ^^^^^^^^^^^^ expected &str, found usize
  |
  = note: expected type `&str`
             found type `usize`
```

現在我們探索了變量如何工作，讓我們看看更多的數據類型。