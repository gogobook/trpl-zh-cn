## 高級函數與閉包

> [ch19-05-advanced-functions-and-closures.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch19-05-advanced-functions-and-closures.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

最後讓我們討論一些有關函數和閉包的高級功能：函數指針、發散函數和返回值閉包。

### 函數指針

我們討論過了如何向函數傳遞閉包，不過也可以向函數傳遞常規的函數！函數的類型是 `fn`，使用小寫的 「f」 以便不與 `Fn` 閉包 trait 向混淆。`fn` 被稱為**函數指針**（*function pointer*）。指定參數為函數指針的語法類似於閉包，如代碼例 19-34 所示：

<span class="filename">文件名: src/main.rs</span>

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

fn do_twice(f: fn(i32) -> i32, arg: i32) -> i32 {
    f(arg) + f(arg)
}

fn main() {
    let answer = do_twice(add_one, 5);

    println!("The answer is: {}", answer);
}
```

<span class="caption">代碼例 19-34：使用 `fn` 類型接受函數指針作為參數</span>

這會打印出 `The answer is: 12`。`do_twice` 中的 `f` 被指定為一個接受一個 `i32` 參數並返回 `i32` 的 `fn`。接著就可以在 `do_twice` 函數體中調用 `f`。在  `main` 中，可以將函數名 `add_one` 作為第一個參數傳遞給 `do_twice`。

不同於閉包，`fn` 是一個類型而不是一個 trait，所以直接指定 `fn` 作為參數而不是聲明一個帶有 `Fn` 作為 trait bound 的泛型參數。

函數指針實現了所有三個閉包 trait（`Fn`、`FnMut` 和 `FnOnce`），所以總是可以在調用期望閉包的函數時傳遞函數指針作為參數。傾向於編寫使用泛型和閉包 trait 的函數，這樣它就能接受函數或閉包作為參數。一個只期望接受 `fn` 的情況的例子是與不存在閉包的外部代碼交互時：C 語言的函數可以接受函數作為參數，但沒有閉包。

比如，如果希望使用 `map` 函數將一個數字 vector 轉換為一個字符串 vector，就可以使用閉包：

```rust
let list_of_numbers = vec![1, 2, 3];
let list_of_strings: Vec<String> = list_of_numbers
    .iter()
    .map(|i| i.to_string())
    .collect();
```

或者可以將函數作為 `map` 的參數來代替閉包：

```rust
let list_of_numbers = vec![1, 2, 3];
let list_of_strings: Vec<String> = list_of_numbers
    .iter()
    .map(ToString::to_string)
    .collect();
```

注意這裡必須使用「高級 trait」部分講到的完全限定語法，因為存在多個叫做 `to_string` 的函數；這裡使用定義於 `ToString` trait 的 `to_string` 函數，標準庫為所有實現了 `Display` 的類型實現了這個 trait。

一些人傾向於函數風格，一些人喜歡閉包。他們最終都會產生同樣的代碼，所以請使用你更明白的吧。

### 返回閉包

因為閉包以 trait 的形式體現，返回閉包就有點微妙了；不能直接這麼做。對於大部分需要返回 trait 的情況，可以使用是實現了期望返回的 trait 的具體類型替代函數的返回值。但是這不能用於閉包。他們沒有一個可返回的具體類型；例如不允許使用函數指針 `fn` 作為返回值類型。

這段代碼嘗試直接返回閉包，它並不能編譯：

```rust
fn returns_closure() -> Fn(i32) -> i32 {
    |x| x + 1
}
```

編譯器給出的錯誤是：

```
error[E0277]: the trait bound `std::ops::Fn(i32) -> i32 + 'static:
std::marker::Sized` is not satisfied
 --> <anon>:2:25
  |
2 | fn returns_closure() -> Fn(i32) -> i32 {
  |                         ^^^^^^^^^^^^^^ the trait `std::marker::Sized` is
  not implemented for `std::ops::Fn(i32) -> i32 + 'static`
  |
  = note: `std::ops::Fn(i32) -> i32 + 'static` does not have a constant size
  known at compile-time
  = note: the return type of a function must have a statically known size
```

又是 `Sized` trait！Rust 並不知道需要多少空間來儲存閉包。不過我們在上一部分見過這種情況的解決辦法：可以使用 trait 物件：

```rust
fn returns_closure() -> Box<Fn(i32) -> i32> {
    Box::new(|x| x + 1)
}
```

關於 trait 物件的更多內容，請參考第十八章。

## 總結

好的！現在我們學習了 Rust 並不常用但你可能用得著的功能。我們介紹了很多複雜的主題，這樣當你在錯誤信息提示或閱讀他人代碼時遇到他們時，至少可以說已經見過這些概念和語法了。

現在，讓我們再開始一個項目，將本書所學的所有內容付與實踐！