## 生命週期與引用有效性

> [ch10-03-lifetime-syntax.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch10-03-lifetime-syntax.md)
> <br>
> commit aa4be9389d18c31f587ebf75cbbb6af39ff4247d

當在第四章討論引用時，我們遺漏了一個重要的細節：Rust 中的每一個引用都有其 **生命週期**（*lifetime*），也就是引用保持有效的作用域。大部分時候生命週期是隱含並可以推斷的，正如大部分時候類型也是可以推斷的一樣。類似於當因為有多種可能類型的時候必須註明類型，也會出現引用的生命週期以一些不同方式相關聯的情況，所以 Rust 需要我們使用泛型生命週期參數來註明他們的關係，這樣就能確保運行時實際使用的引用絕對是有效的。

好吧，這有點不太尋常，而且也不同於其他語言中使用的工具。生命週期，從某種意義上說，是 Rust 最與眾不同的功能。

生命週期是一個很廣泛的話題，本章不可能涉及到它全部的內容，所以這裡我們會講到一些通常你可能會遇到的生命週期語法以便你熟悉這個概念。第十九章會包含生命週期所有功能的更高級的內容。

### 生命週期避免了懸垂引用

生命週期的主要目標是避免懸垂引用，它會導致程序引用了非預期引用的數據。考慮一下列表 10-18 中的程序，它有一個外部作用域和一個內部作用域，外部作用域聲明了一個沒有初值的變數 `r`，而內部作用域聲明了一個初值為 5 的變數`x`。在內部作用域中，我們嘗試將 `r` 的值設置為一個 `x` 的引用。接著在內部作用域結束後，嘗試打印出 `r` 的值：

```rust
{
    let r;

    {
        let x = 5;
        r = &x;
    }

    println!("r: {}", r);
}
```

<span class="caption">列表 10-18：嘗試使用離開作用域的值的引用</span>

> ### 未初始化變數不能被使用
>
> 接下來的一些例子中聲明了沒有初始值的變數，以便這些變數存在於外部作用域。這看起來好像和 Rust 不允許存在空值相衝突。然而這是可以的，如果我們嘗試在給它一個值之前使用這個變數，會出現一個編譯時錯誤。請自行嘗試！

當編譯這段代碼時會得到一個錯誤：

```text
error: `x` does not live long enough
   |
6  |         r = &x;
   |              - borrow occurs here
7  |     }
   |     ^ `x` dropped here while still borrowed
...
10 | }
   | - borrowed value needs to live until here
```

變數 `x` 並沒有 「存在的足夠久」。為什麼呢？好吧，`x` 在到達第 7 行的大括號的結束時就離開了作用域，這也是內部作用域的結尾。不過 `r` 在外部作用域也是有效的；作用域越大我們就說它 「存在的越久」。如果 Rust 允許這段代碼工作，`r`將會引用在`x`離開作用域時被釋放的內存，這時嘗試對`r`做任何操作都會不能正常工作。那麼 Rust 是如何決定這段代碼是不被允許的呢？

#### 借用檢查器

編譯器的這一部分叫做 **借用檢查器**（*borrow checker*），它比較作用域來確保所有的借用都是有效的。列表 10-19 展示了與列表 10-18 相同的例子不過帶有變數聲明週期的註釋：

```rust
{
    let r;         // -------+-- 'a
                   //        |
    {              //        |
        let x = 5; // -+-----+-- 'b
        r = &x;    //  |     |
    }              // -+     |
                   //        |
    println!("r: {}", r); // |
                   //        |
                   // -------+
}
```

<span class="caption">列表 10-19：`r` 和 `x` 的生命週期註解，分別叫做 `'a` 和 `'b`</span>

<!-- Just checking I'm reading this right: the inside block is the b lifetime,
correct? I want to leave a note for production, make sure we can make that
clear -->
<!-- Yes, the inside block for the `'b` lifetime starts with the `let x = 5;`
line and ends with the first closing curly brace on the 7th line. Do you think
the text art comments work or should we make an SVG diagram that has nicer
looking arrows and labels? /Carol -->

我們將 `r` 的生命週期標記為 `'a` 並將 `x` 的生命週期標記為 `'b`。如你所見，內部的 `'b` 塊要比外部的生命週期 `'a` 小得多。在編譯時，Rust 比較這兩個生命週期的大小，並發現 `r` 擁有聲明週期 `'a`，不過它引用了一個擁有生命週期 `'b` 的物件。程序被拒絕編譯，因為生命週期 `'b` 比生命週期 `'a` 要小：被引用的物件比它的引用者存活的時間更短。

讓我們看看列表 10-20 中這個並沒有產生懸垂引用且可以正確編譯的例子：

```rust
{
    let x = 5;            // -----+-- 'b
                          //      |
    let r = &x;           // --+--+-- 'a
                          //   |  |
    println!("r: {}", r); //   |  |
                          // --+  |
}                         // -----+
```

<span class="caption">列表 10-20：一個有效的引用，因為數據比引用有著更長的生命週期</span>

這裡 `x` 擁有生命週期 `'b`，比 `'a` 要大。這就意味著 `r` 可以引用 `x`：Rust 知道 `r` 中的引用在 `x` 有效的時候也總是有效的。

現在我們已經在一個具體的例子中展示了引用的聲明週期位於何處，並討論了 Rust 如何分析生命週期來保證引用總是有效的，接下來讓我們聊聊在函數的上下文中參數和返回值的泛型生命週期。

### 函數中的泛型生命週期

讓我們來編寫一個返回兩個字符串 slice 中較長者的函數。我們希望能夠通過傳遞兩個字符串 slice 來調用這個函數，並希望返回一個字符串 slice。一旦我們實現了 `longest` 函數，列表 10-21 中的代碼應該會打印出 `The longest string is abcd`：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";

    let result = longest(string1.as_str(), string2);
    println!("The longest string is {}", result);
}
```

<span class="caption">列表 10-21：`main` 函數調用 `longest` 函數來尋找兩個字符串 slice 中較長的一個</span>

注意函數期望抓取字符串 slice（如第四章所講到的這是引用）因為我們並不希望`longest` 函數抓取其參數的所有權。我們希望函數能夠接受 `String` 的 slice（也就是變量 `string1` 的類型）以及字符串字面值（也就是變量 `string2` 包含的值）。

<!-- why is `a` a slice and `b` a literal? You mean "a" from the string "abcd"? -->
<!-- I've changed the variable names to remove ambiguity between the variable
name `a` and the "a" from the string "abcd". `string1` is not a slice, it's a
`String`, but we're going to pass a slice that refers to that `String` to the
`longest` function (`string1.as_str()` creates a slice that references the
`String` stored in `string1`). We chose to have `string2` be a literal since
the reader might have code with both `String`s and string literals, and the way
most readers first get into problems with lifetimes is involving string slices,
so we wanted to demonstrate the flexibility of taking string slices as
arguments but the issues you might run into because string slices are
references.
All of the `String`/string slice/string literal concepts here are covered
thoroughly in Chapter 4, which is why we put two back references here (above
and below). If these topics are confusing you in this context, I'd be
interested to know if rereading Chapter 4 clears up that confusion.
/Carol -->

參考之前第四章中的 「字符串 slice 作為參數」 部分中更多關於為什麼上面例子中的參數正符合我們期望的討論。

如果嘗試像列表 10-22 中那樣實現 `longest` 函數，它並不能編譯：

<span class="filename">文件名: src/main.rs</span>

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

<span class="caption">列表 10-22：一個 `longest` 函數的實現，它返回兩個字符串 slice 中較長者，現在還不能編譯</span>

將會出現如下有關生命週期的錯誤：

```text
error[E0106]: missing lifetime specifier
   |
1  | fn longest(x: &str, y: &str) -> &str {
   |                                 ^ expected lifetime parameter
   |
   = help: this function's return type contains a borrowed value, but the
   signature does not say whether it is borrowed from `x` or `y`
```

提示文本告訴我們返回值需要一個泛型生命週期參數，因為 Rust 並不知道將要返回的引用是指向 `x` 或 `y`。事實上我們也不知道，因為函數體中 `if` 塊返回一個 `x` 的引用而 `else` 塊返回一個 `y` 的引用。

雖然我們定義了這個函數，但是並不知道傳遞給函數的具體值，所以也不知道到底是 `if` 還是 `else` 會被執行。我們也不知道傳入的引用的具體生命週期，所以也就不能像列表 10-19 和 10-20 那樣通過觀察作用域來確定返回的引用是否總是有效。借用檢查器自身同樣也無法確定，因為它不知道 `x` 和 `y` 的生命週期是如何與返回值的生命週期相關聯的。接下來我們將增加泛型生命週期參數來定義引用間的關係以便借用檢查器可以進行分析。

### 生命週期註解語法

生命週期註解並不改變任何引用的生命週期的長短。與當函數簽名中指定了泛型類型參數後就可以接受任何類型一樣，當指定了泛型生命週期後函數也能接受任何生命週期的引用。生命週期註解所做的就是將多個引用的生命週期聯繫起來。

生命週期註解有著一個不太常見的語法：生命週期參數名稱必須以撇號（`'`）開頭。生命週期參數的名稱通常全是小寫，而且類似於泛型類型，其名稱通常非常短。`'a` 是大多數人預設使用的名稱。生命週期參數註解位於引用的 `&` 之後，並有一個空格來將引用類型與生命週期註解分隔開。

這裡有一些例子：我們有一個沒有生命週期參數的 `i32` 的引用，一個有叫做 `'a` 的生命週期參數的 `i32` 的引用，和一個也有的生命週期參數 `'a` 的 `i32` 的可變引用：

```rust
&i32        // a reference
&'a i32     // a reference with an explicit lifetime
&'a mut i32 // a mutable reference with an explicit lifetime
```

生命週期註解本身沒有多少意義：生命週期註解告訴 Rust 多個引用的泛型生命週期參數如何相互聯繫。如果函數有一個生命週期 `'a` 的 `i32` 的引用的參數 `first`，還有另一個同樣是生命週期 `'a` 的 `i32` 的引用的參數 `second`，這兩個生命週期註解有相同的名稱意味著 `first` 和 `second` 必須與這相同的泛型生命週期存在得一樣久。

### 函數簽名中的生命週期註解

來看看我們編寫的 `longest` 函數的上下文中的生命週期。就像泛型類型參數，泛型生命週期參數需要聲明在函數名和參數列表間的尖括號中。這裡我們想要告訴 Rust 關於參數中的引用和返回值之間的限制是他們都必須擁有相同的生命週期，就像列表 10-23 中在每個引用中都加上了 `'a` 那樣：

<span class="filename">文件名: src/main.rs</span>

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

<span class="caption">列表 10-23：`longest` 函數定義指定了簽名中所有的引用必須有相同的生命週期 `'a`</span>

這段代碼能夠編譯並會產生我們想要使用列表 10-21 中的 `main` 函數得到的結果。

現在函數簽名表明對於某些生命週期 `'a`，函數會抓取兩個參數，他們都是與生命週期 `'a` 存在的一樣長的字符串 slice。函數會返回一個同樣也與生命週期 `'a` 存在的一樣長的字符串 slice。這就是我們告訴 Rust 需要其保證的協議。

通過在函數簽名中指定生命週期參數，不會改變任何參數或返回值的生命週期，不過我們說過任何不堅持這個協議的類型都將被借用檢查器拒絕。這個函數並不知道（或需要知道）`x` 和 `y` 具體會存在多久，不過只需要知道一些可以使用 `'a` 替代的作用域將會滿足這個簽名。

當在函數中使用生命週期註解時，這些註解出現在函數簽名中，而不存在於函數體中的任何代碼中。這是因為 Rust 能夠分析函數中代碼而不需要任何協助，不過當函數引用或被函數之外的代碼引用時，參數或返回值的生命週期可能在每次函數被調用時都不同。這可能會產生驚人的消耗並且對於 Rust 來說通常是不可能分析的。在這種情況下，我們需要自己標註生命週期。

當具體的引用被傳遞給 `longest` 時，被 `'a` 所替代的具體生命週期是 `x` 的作用域與 `y` 的作用域相重疊的那一部分。因為作用域總是嵌套的，所以換一種說法就是泛型生命週期 `'a` 的具體生命週期等同於 `x` 和 `y` 的生命週期中較小的那一個。因為我們用相同的生命週期參數標註了返回的引用值，所以返回的引用值就能保證在 `x` 和 `y` 中較短的那個生命週期結束之前保持有效。

讓我們如何通過傳遞擁有不同具體生命週期的引用來觀察他們是如何限制 `longest` 函數的使用的。列表 10-24 是一個應該在任何編程語言中都很直觀的例子：`string1` 直到外部作用域結束都是有效的，`string2` 則在內部作用域中是有效的，而 `result` 則引用了一些直到內部作用域結束都是有效的值。借用檢查器認可這些代碼；它能夠編譯和運行，並打印出 `The longest string is long string is long`：

<span class="filename">文件名: src/main.rs</span>

```rust
# fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
#     if x.len() > y.len() {
#         x
#     } else {
#         y
#     }
# }
#
fn main() {
    let string1 = String::from("long string is long");

    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is {}", result);
    }
}
```

<span class="caption">列表 10-24：通過擁有不同的具體生命週期的 `String` 值調用 `longest` 函數</span>

接下來，讓我們嘗試一個 `result` 的引用的生命週期必須比兩個參數的要短的例子。將 `result` 變數的聲明從內部作用域中移動出來，不過將 `result` 和 `string2` 變數的賦值語句一同放在內部作用域裡。接下來，我們將使用 `result` 的 `println!` 移動到內部作用域之外，就在其結束之後。注意列表 10-25 中的代碼不能編譯：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}
```

<span class="caption">列表 10-25：在 `string2` 離開作用域之後使用 `result` 的嘗試不能編譯</span>

如果嘗試編譯會出現如下錯誤：

```text
error: `string2` does not live long enough
   |
6  |         result = longest(string1.as_str(), string2.as_str());
   |                                            ------- borrow occurs here
7  |     }
   |     ^ `string2` dropped here while still borrowed
8  |     println!("The longest string is {}", result);
9  | }
   | - borrowed value needs to live until here
```

錯誤表明為了保證 `println!` 中的 `result` 是有效的，`string2` 需要直到外部作用域結束都是有效的。Rust 知道這些是因為（`longest`）函數的參數和返回值都使用了相同的生命週期參數 `'a`。

以我們的理解 `string1` 更長，因此 `result` 會包含指向 `string1` 的引用。因為 `string1` 尚未離開作用域，對於 `println!` 來說 `string1` 的引用仍然是有效的。然而，我們通過生命週期參數告訴 Rust 的是 `longest` 函數返回的引用的生命週期應該與傳入參數的生命週期中較短那個保持一致。因此，借用檢查器不允許列表 10-25 中的代碼，因為它可能會存在無效的引用。

請嘗試更多採用不同的值和不同生命週期的引用作為 `longest` 函數的參數和返回值的實驗。並在開始編譯前猜想你的實驗能否通過借用檢查器，接著編譯一下看看你的理解是否正確！

### 深入理解生命週期

指定生命週期參數的正確方式依賴函數具體的功能。例如，如果將 `longest` 函數的實現修改為總是返回第一個參數而不是最長的字符串 slice，就不需要為參數 `y` 指定一個生命週期。如下代碼將能夠編譯：

<span class="filename">文件名: src/main.rs</span>

```rust
fn longest<'a>(x: &'a str, y: &str) -> &'a str {
    x
}
```

在這個例子中，我們為參數 `x` 和返回值指定了生命週期參數 `'a`，不過沒有為參數 `y` 指定，因為 `y` 的生命週期與參數 `x` 和返回值的生命週期沒有任何關係。

當從函數返回一個引用，返回值的生命週期參數需要與一個參數的生命週期參數相匹配。如果返回的引用 **沒有** 指向任何一個參數，那麼唯一的可能就是它指向一個函數內部創建的值，它將會是一個懸垂引用，因為它將會在函數結束時離開作用域。嘗試考慮這個並不能編譯的 `longest` 函數實現：

<span class="filename">文件名: src/main.rs</span>

```rust
fn longest<'a>(x: &str, y: &str) -> &'a str {
    let result = String::from("really long string");
    result.as_str()
}
```

即便我們為返回值指定了生命週期參數 `'a`，這個實現卻編譯失敗了，因為返回值的生命週期與參數完全沒有關聯。這裡是會出現的錯誤信息：

```text
error: `result` does not live long enough
  |
3 |     result.as_str()
  |     ^^^^^^ does not live long enough
4 | }
  | - borrowed value only lives until here
  |
note: borrowed value must be valid for the lifetime 'a as defined on the block
at 1:44...
  |
1 | fn longest<'a>(x: &str, y: &str) -> &'a str {
  |                                             ^
```

出現的問題是 `result` 在 `longest` 函數的結尾將離開作用域並被清理，而我們嘗試從函數返回一個 `result` 的引用。無法指定生命週期參數來改變懸垂引用，而且 Rust 也不允許我們創建一個懸垂引用。在這種情況，最好的解決方案是返回一個有所有權的數據類型而不是一個引用，這樣函數調用者就需要負責清理這個值了。

從結果上看，生命週期語法是關於如何聯繫函數不同參數和返回值的生命週期的。一旦他們形成了某種聯繫，Rust 就有了足夠的信息來允許內存安全的操作並阻止會產生懸垂指針亦或是違反內存安全的行為。

### 結構體定義中的生命週期註解

目前為止，我們只定義過有所有權類型的結構體。也可以定義存放引用的結構體，不過需要為結構體定義中的每一個引用添加生命週期註解。列表 10-26 中有一個存放了一個字符串 slice 的結構體 `ImportantExcerpt`：

<span class="filename">文件名: src/main.rs</span>

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.')
        .next()
        .expect("Could not find a '.'");
    let i = ImportantExcerpt { part: first_sentence };
}
```

<span class="caption">列表 10-26：一個存放引用的結構體，所以其定義需要生命週期註解</span>

這個結構體有一個字段，`part`，它存放了一個字符串 slice，這是一個引用。類似於泛型參數類型，必須在結構體名稱後面的尖括號中聲明泛型生命週期參數，以便在結構體定義中使用生命週期參數。

這裡的 `main` 函數創建了一個 `ImportantExcerpt` 的實例，它存放了變數 `novel` 所擁有的 `String` 的第一個句子的引用。

### 生命週期省略

在這一部分，我們知道了每一個引用都有一個生命週期，而且需要為使用了引用的函數或結構體指定生命週期。然而，第四章的 「字符串 slice」 部分有一個函數，我們在列表 10-27 中再次展示出來，它沒有生命週期註解卻能成功編譯：

<span class="filename">文件名: src/lib.rs</span>

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

<span class="caption">列表 10-27：第四章定義了一個沒有使用生命週期註解的函數，即便其參數和返回值都是引用</span>

這個函數沒有生命週期註解卻能編譯是由於一些歷史原因：在早期 1.0 之前版本的 Rust 中，這的確是不能編譯的。每一個引用都必須有明確的生命週期。那時的函數簽名將會寫成這樣：

```rust
fn first_word<'a>(s: &'a str) -> &'a str {
```

在編寫了很多 Rust 代碼後，Rust 團隊發現在特定情況下 Rust 程式設計師們總是重複地編寫一模一樣的生命週期註解。這些場景是可預測的並且遵循幾個明確的模式。接著 Rust 團隊就把這些模式編碼進了 Rust 編譯器中，如此借用檢查器在這些情況下就能推斷出生命週期而不再強製程式設計師顯式的增加註解。

這裡我們提到一些 Rust 的歷史是因為更多的明確的模式被合併和添加到編譯器中是完全可能的。未來只會需要更少的生命週期註解。

被編碼進 Rust 引用分析的模式被稱為 **生命週期省略規則**（*lifetime elision rules*）。這並不是需要程式設計師遵守的規則；這些規則是一系列特定的場景，此時編譯器會考慮，如果代碼符合這些場景，就無需明確指定生命週期。

省略規則並不提供完整的推斷：如果 Rust 在明確遵守這些規則的前提下變數的生命週期仍然是模棱兩可的話，它不會猜測剩餘引用的生命週期應該是什麼。在這種情況，編譯器會給出一個錯誤，這可以通過增加對應引用之間相聯繫的生命週期註解來解決。

首先，介紹一些定義：函數或方法的參數的生命週期被稱為 **輸入生命週期**（*input lifetimes*），而返回值的生命週期被稱為 **輸出生命週期**（*output lifetimes*）。

現在介紹編譯器用於判斷引用何時不需要明確生命週期註解的規則。第一條規則適用於輸入生命週期，後兩條規則適用於輸出生命週期。如果編譯器檢查完這三條規則後仍然存在沒有計算出生命週期的引用，編譯器將會停止並生成錯誤。

1. 每一個是引用的參數都有它自己的生命週期參數。換句話說就是，有一個引用參數的函數有一個生命週期參數：`fn foo<'a>(x: &'a i32)`，有兩個引用參數的函數有兩個不同的生命週期參數，`fn foo<'a, 'b>(x: &'a i32, y: &'b i32)`，依此類推。

2. 如果只有一個輸入生命週期參數，那麼它被賦給所有輸出生命週期參數：`fn foo<'a>(x: &'a i32) -> &'a i32`。

3. 如果方法有多個輸入生命週期參數，不過其中之一因為方法的緣故為 `&self` 或 `&mut self`，那麼 `self` 的生命週期被賦給所有輸出生命週期參數。這使得方法寫起來更簡潔。

假設我們自己就是編譯器並來計算列表 10-25 `first_word` 函數的簽名中的引用的生命週期。開始時簽名中的引用並沒有關聯任何生命週期：

```rust
fn first_word(s: &str) -> &str {
```

接著我們（作為編譯器）應用第一條規則，也就是每個引用參數都有其自己的生命週期。我們像往常一樣稱之為 `'a`，所以現在簽名看起來像這樣：

```rust
fn first_word<'a>(s: &'a str) -> &str {
```

對於第二條規則，因為這裡正好只有一個輸入生命週期參數所以是適用的。第二條規則表明輸入參數的生命週期將被賦予輸出生命週期參數，所以現在簽名看起來像這樣：

```rust
fn first_word<'a>(s: &'a str) -> &'a str {
```

現在這個函數簽名中的所有引用都有了生命週期，而編譯器可以繼續它的分析而無須程式設計師標記這個函數簽名中的生命週期。

讓我們再看看另一個例子，這次我們從列表 10-22 中沒有生命週期參數的 `longest` 函數開始：

```rust
fn longest(x: &str, y: &str) -> &str {
```

再次假設我們自己就是編譯器並應用第一條規則：每個引用參數都有其自己的生命週期。這次有兩個參數，所以就有兩個生命週期：

```rust
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &str {
```

再來應用第二條規則，它並不適用因為存在多於一個輸入生命週期。再來看第三條規則，它同樣也不適用因為沒有 `self` 參數。然後我們就沒有更多規則了，不過還沒有計算出返回值的類型的生命週期。這就是為什麼在編譯列表 10-22 的代碼時會出現錯誤的原因：編譯器適用所有已知的生命週期省略規則，不過仍然不能計算出簽名中所有引用的生命週期。

因為第三條規則真正能夠適用的就只有方法簽名，現在就讓我們看看那種情況中的生命週期，並看看為什麼這條規則意味著我們經常不需要在方法簽名中標註生命週期。

### 方法定義中的生命週期註解

<!-- Is this different to the reference lifetime annotations, or just a
finalized explanation? -->
<!-- This is about lifetimes on references in method signatures, which is where
the 3rd lifetime elision rule kicks in. It can also be confusing where lifetime
parameters need to be declared and used since the lifetime parameters could go
with the struct's fields or with references passed into or returned from
methods. /Carol -->

當為帶有生命週期的結構體實現方法時，其語法依然類似列表 10-11 中展示的泛型類型參數的語法：聲明和使用生命週期參數的位置依賴於生命週期參數是否同結構體字段或方法參數和返回值相關

（實現方法時）結構體字段的生命週期必須總是在 `impl` 關鍵字之後聲明並在結構體名稱之後被使用，因為這些生命週期是結構體類型的一部分。

`impl` 塊裡的方法簽名中，引用可能與結構體字段中的引用相關聯，也可能是獨立的。另外，生命週期省略規則也經常讓我們無需在方法簽名中使用生命週期註解。讓我們看看一些使用列表 10-26 中定義的結構體 `ImportantExcerpt` 的例子。

首先，這裡有一個方法 `level`。其唯一的參數是 `self` 的引用，而且返回值只是一個 `i32`，並不引用任何值：

```rust
# struct ImportantExcerpt<'a> {
#     part: &'a str,
# }
#
impl<'a> ImportantExcerpt<'a> {
    fn level(&self) -> i32 {
        3
    }
}
```

`impl` 之後和類型名稱之後的生命週期參數是必要的，不過因為第一條生命週期規則我們並不必須標註 `self` 引用的生命週期。

這裡是一個適用於第三條生命週期省略規則的例子：

```rust
# struct ImportantExcerpt<'a> {
#     part: &'a str,
# }
#
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

這裡有兩個輸入生命週期，所以 Rust 應用第一條生命週期省略規則並給予 `&self` 和 `announcement` 他們各自的生命週期。接著，因為其中一個參數是 `&self`，返回值類型被賦予了 `&self` 的生命週期，這樣所有的生命週期都被計算出來了。

### 靜態生命週期

這裡有 **一種** 特殊的生命週期值得討論：`'static`。`'static` 生命週期存活於整個程序期間。所有的字符串字面值都擁有 `'static` 生命週期，我們也可以選擇像下面這樣標註出來：

```rust
let s: &'static str = "I have a static lifetime.";
```

這個字符串的文本被直接儲存在程序的二進制文件中而這個文件總是可用的。因此所有的字符串字面值都是 `'static` 的。

<!-- How would you add a static lifetime (below)? -->
<!-- Just like you'd specify any lifetime, see above where it shows `&'static str`. /Carol -->

你可能在錯誤信息的幫助文本中見過使用 `'static` 生命週期的建議，不過將引用指定為 `'static` 之前，思考一下這個引用是否真的在整個程序的生命週期裡都有效（或者哪怕你希望它一直有效，如果可能的話）。大部分情況，代碼中的問題是嘗試創建一個懸垂引用或者可用的生命週期不匹配，請解決這些問題而不是指定一個 `'static` 的生命週期。

### 結合泛型類型參數、trait bounds 和生命週期

讓我們簡單的看一下在同一函數中指定泛型類型參數、trait bounds 和生命週期的語法！

```rust
use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(x: &'a str, y: &'a str, ann: T) -> &'a str
    where T: Display
{
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

這個是列表 10-23 中那個返回兩個字符串 slice 中較長者的 `longest` 函數，不過帶有一個額外的參數 `ann`。`ann` 的類型是泛型 `T`，它可以被放入任何實現了 `where` 從句中指定的 `Display` trait 的類型。這個額外的參數會在函數比較字符串 slice 的長度之前被打印出來，這也就是為什麼 `Display` trait bound 是必須的。因為生命週期也是泛型，所以生命週期參數 `'a` 和泛型類型參數 `T` 都位於函數名後的同一尖括號列表中。

## 總結

這一章介紹了很多的內容！現在你知道了泛型類型參數、trait 和 trait bounds 以及泛型生命週期類型，你已經準備好編寫既不重複又能適用於多種場景的代碼了。泛型類型參數意味著代碼可以適用於不同的類型。trait 和 trait bounds 保證了即使類型是泛型的，這些類型也會擁有所需要的行為。由生命週期註解所指定的引用生命週期之間的關係保證了這些靈活多變的代碼不會出現懸垂引用。而所有的這一切發生在編譯時所以不會影響運行時效率！

你可能不會相信，這個領域還有更多需要學習的內容：第十七章會討論 trait 物件，這是另一種使用 trait 的方式。第十九章會涉及到生命週期註解更複雜的場景。第二十章講解一些高級的類型系統功能。不過接下來，讓我們聊聊如何在 Rust 中編寫測試，來確保代碼的所有功能能像我們希望的那樣工作！