## Slices

> [ch04-03-slices.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch04-03-slices.md)
> <br>
> commit df9e3b922335ec2c76b6c1c4ede31b7742103c48

另一個沒有所有權的數據類型是 *slice*。slice 允許你引用集合中一段連續的元素序列，而不用引用整個集合。

這裡有一個小的編程問題：編寫一個獲取一個字符串並返回它在其中找到的第一個單詞的函數。如果函數沒有在字符串中找到一個空格，就意味著整個字符串是一個單詞，所以整個字符串都應該返回。

讓我們看看這個函數的簽名：

```rust
fn first_word(s: &String) -> ?
```

`first_word` 這個函數有一個參數 `&String`。因為我們不需要所有權，所以這沒有問題。不過應該返回什麼呢？我們並沒有一個真正獲取 **部分** 字符串的辦法。不過，我們可以返回單詞結尾的索引。讓我們試試如列表 4-10 所示的代碼：

<span class="filename">Filename: src/main.rs</span>

```rust
fn first_word(s: &String) -> usize {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return i;
        }
    }

    s.len()
}
```

<span class="caption">列表 4-10：`first_word` 函數返回 `String` 參數的一個字節索引值</span>

讓我們將代碼分解成小塊。因為需要一個元素一個元素的檢查 `String` 中的值是否是空格，需要用 `as_bytes` 方法將 `String` 轉化為字節數組：

```rust
let bytes = s.as_bytes();
```

接下來，使用 `iter` 方法在字節數組上創建一個迭代器：

```rust
for (i, &item) in bytes.iter().enumerate() {
```

第十三章將討論迭代器的更多細節。現在，只需知道 `iter` 方法返回集合中的每一個元素，而 `enumerate` 包裝 `iter` 的結果並返回一個元組，其中每一個元素是元組的一部分。返回元組的第一個元素是索引，第二個元素是集合中元素的引用。這比我們自己計算索引要方便一些。

因為 `enumerate` 方法返回一個元組，我們可以使用模式來解構它，就像 Rust 中其他地方一樣。所以在 `for` 循環中，我們指定了一個模式，其中 `i` 是元組中的索引而 `&item` 是單個字節。因為從 `.iter().enumerate()` 中獲取了集合元素的引用，我們在模式中使用了`&`。

我們通過字節的字面值來尋找代表空格的字節。如果找到了，返回它的位置。否則，使用 `s.len()` 返回字符串的長度：

```rust
    if item == b' ' {
        return i;
    }
}
s.len()
```

現在有了一個找到字符串中第一個單詞結尾索引的方法了，不過這有一個問題。我們返回了單獨一個 `usize`，不過它只在 `&String` 的上下文中才是一個有意義的數字。換句話說，因為它是一個與 `String` 相分離的值，無法保證將來它仍然有效。考慮一下列表 4-11 中使用了列表 4-10 中 `first_word` 函數的程序：

<span class="filename">文件名: src/main.rs</span>

```rust
# fn first_word(s: &String) -> usize {
#     let bytes = s.as_bytes();
#
#     for (i, &item) in bytes.iter().enumerate() {
#         if item == b' ' {
#             return i;
#         }
#     }
#
#     s.len()
# }
#
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s); // word will get the value 5.

    s.clear(); // This empties the String, making it equal to "".

    // word still has the value 5 here, but there's no more string that
    // we could meaningfully use the value 5 with. word is now totally invalid!
}
```

<span class="caption">列表 4-11：儲存 `first_word` 函數調用的返回值並接著改變 `String` 的內容</span>

這個程序編譯時沒有任何錯誤，而且在調用 `s.clear()` 之後使用 `word` 也不會出錯。這時 `word` 與 `s` 狀態就沒有聯繫了，所以 `word `仍然包含值 `5`。可以嘗試用值 `5` 來提取變數 `s` 的第一個單詞，不過這是有 bug 的，因為在我們將 `5` 保存到 `word` 之後 `s` 的內容已經改變。

不得不擔心 `word` 的索引與 `s` 中的數據不再同步是乏味且容易出錯的！如果編寫這麼一個 `second_word` 函數的話管理索引將更加容易出問題。它的簽名看起來像這樣：

```rust
fn second_word(s: &String) -> (usize, usize) {
```

現在我們跟蹤了一個開始索引 **和** 一個結尾索引，同時有了更多從數據的某個特定狀態計算而來的值，他們也完全沒有與這個狀態相關聯。現在有了三個飄忽不定的不相關變數都需要被同步。

幸運的是，Rust 為這個問題提供了一個解決方案：字符串 slice。

### 字符串 slice

**字符串 slice**（*string slice*）是 `String` 中一部分值的引用，它看起來像這樣：

```rust
let s = String::from("hello world");

let hello = &s[0..5];
let world = &s[6..11];
```

這類似於獲取整個 `String` 的引用不過帶有額外的 `[0..5]` 部分。不同於整個 `String` 的引用，這是一個包含 `String` 內部的一個位置和所需元素數量的引用。`start..end` 語法代表一個以 `start` 開頭並一直持續到但不包含 `end` 的 range。

使用一個由中括號中的 `[starting_index..ending_index]` 指定的 range 創建一個 slice，其中 `starting_index` 是包含在 slice 的第一個位置，`ending_index` 則是 slice 最後一個位置的後一個值。在其內部，slice 的數據結構儲存了開始位置和 slice 的長度，長度對應 `ending_index` 減去 `starting_index` 的值。所以對於 `let world = &s[6..11];` 的情況，`world` 將是一個包含指向 `s` 第 6 個字節的指針和長度值 5 的 slice。

圖 4-12 展示了一個圖例。

<img alt="world containing a pointer to the 6th byte of String s and a length 5" src="img/trpl04-06.svg" class="center" style="width: 50%;" />

<span class="caption">圖 4-12：引用了部分 `String` 的字符串 slice</span>

對於 Rust 的 `..` range 語法，如果想要從第一個索引（0）開始，可以不寫兩個點號之前的值。換句話說，如下兩個語句是相同的：

```rust
let s = String::from("hello");

let slice = &s[0..2];
let slice = &s[..2];
```

由此類推，如果 slice 包含 `String` 的最後一個字節，也可以捨棄尾部的數字。這意味著如下也是相同的：

```rust
let s = String::from("hello");

let len = s.len();

let slice = &s[3..len];
let slice = &s[3..];
```

也可以同時捨棄這兩個值來獲取一個整個字符串的 slice。所以如下亦是相同的：

```rust
let s = String::from("hello");

let len = s.len();

let slice = &s[0..len];
let slice = &s[..];
```

在記住所有這些知識後，讓我們重寫 `first_word` 來返回一個 slice。「字符串 slice」 的類型簽名寫作 `&str`：

<span class="filename">文件名: src/main.rs</span>

```rust
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

我們使用跟列表 4-10 相同的方式獲取單詞結尾的索引，通過尋找第一個出現的空格。當我們找到一個空格，我們返回一個索引，它使用字符串的開始和空格的索引來作為開始和結束的索引。

現在當調用 `first_word` 時，會返回一個單獨的與底層數據相聯繫的值。這個值由一個 slice 開始位置的引用和 slice 中元素的數量組成。

`second_word`函數也可以改為返回一個 slice：

```rust
fn second_word(s: &String) -> &str {
```

現在我們有了一個不易混雜的直觀的 API 了，因為編譯器會確保指向 `String` 的引用保持有效。還記得列表 4-11 程序中，那個當我們獲取第一個單詞結尾的索引不過接著就清除了字符串所以索引就無效了的 bug 嗎？那些代碼邏輯上是不正確的，不過卻沒有表現出任何直接的錯誤。問題會在之後嘗試對空字符串使用第一個單詞的索引時出現。slice 就不可能出現這種 bug 並讓我們更早的知道出問題了。使用 slice 版本的 `first_word` 會拋出一個編譯時錯誤：

<span class="filename">Filename: src/main.rs</span>

```rust
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s);

    s.clear(); // Error!
}
```

這裡是編譯錯誤：

```text
17:6 error: cannot borrow `s` as mutable because it is also borrowed as
            immutable [E0502]
    s.clear(); // Error!
    ^
15:29 note: previous borrow of `s` occurs here; the immutable borrow prevents
            subsequent moves or mutable borrows of `s` until the borrow ends
    let word = first_word(&s);
                           ^
18:2 note: previous borrow ends here
fn main() {

}
^
```

回憶一下借用規則，當擁有某值的不可變引用時，就不能再獲取一個可變引用。因為 `clear` 需要清空 `String`，它嘗試獲取一個可變引用，它失敗了。Rust 不僅使得我們的 API 簡單易用，也在編譯時就消除了一整類的錯誤！

#### 字符串字面值就是 slice

還記得我們講到過字符串字面值被儲存在二進制文件中嗎。現在知道 slice 了，我們就可以正確的理解字符串字面值了：

```rust
let s = "Hello, world!";
```

這裡 `s` 的類型是 `&str`：它是一個指向二進製程序特定位置的 slice。這也就是為什麼字符串字面值是不可變的；`&str` 是一個不可變引用。

#### 字符串 slice 作為參數

在知道了能夠獲取字面值和 `String` 的 slice 後引起了另一個對 `first_word` 的改進，這是它的簽名：

```rust
fn first_word(s: &String) -> &str {
```

相反一個更有經驗的 Rustacean 會寫下如下這一行，因為它使得可以對 `String` 和 `&str` 使用相同的函數：

```rust
fn first_word(s: &str) -> &str {
```

如果有一個字符串 slice，可以直接傳遞它。如果有一個 `String`，則可以傳遞整個 `String` 的 slice。定義一個獲取字符串 slice 而不是字符串引用的函數使得我們的 API 更加通用並且不會丟失任何功能：

<span class="filename">Filename: src/main.rs</span>

```rust
# fn first_word(s: &str) -> &str {
#     let bytes = s.as_bytes();
#
#     for (i, &item) in bytes.iter().enumerate() {
#         if item == b' ' {
#             return &s[0..i];
#         }
#     }
#
#     &s[..]
# }
fn main() {
    let my_string = String::from("hello world");

    // first_word works on slices of `String`s
    let word = first_word(&my_string[..]);

    let my_string_literal = "hello world";

    // first_word works on slices of string literals
    let word = first_word(&my_string_literal[..]);

    // since string literals *are* string slices already,
    // this works too, without the slice syntax!
    let word = first_word(my_string_literal);
}
```

### 其他 slice

字符串 slice，正如你想像的那樣，是針對字符串的。不過也有更通用的 slice 類型。考慮一下這個數組：

```rust
let a = [1, 2, 3, 4, 5];
```

就跟我們想要獲取字符串的一部分那樣，我們也會想要引用數組的一部分，而我們可以這樣做：

```rust
let a = [1, 2, 3, 4, 5];

let slice = &a[1..3];
```

這個 slice 的類型是 `&[i32]`。它跟以跟字符串 slice 一樣的方式工作，通過儲存第一個元素的引用和一個長度。你可以對其他所有類型的集合使用這類 slice。第八章講到 vector 時會詳細討論這些集合。

## 總結

所有權、借用和 slice 這些概念是 Rust 何以在編譯時保障內存安全的關鍵所在。Rust 像其他系統編程語言那樣給予你對內存使用的控制，但擁有數據所有者在離開作用域後自動清除其數據的功能意味著你無須額外編寫和調試相關的控制代碼。

所有權系統影響了 Rust 中其他很多部分如何工作，所以我們還會繼續講到這些概念，這將貫穿本書的餘下內容。讓我們開始下一個章節，來看看如何將多份數據組合進一個 `struct` 中。