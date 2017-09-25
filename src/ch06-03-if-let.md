## `if let` 簡單控制流

> [ch06-03-if-let.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch06-03-if-let.md)
> <br>
> commit 3f2a1bd8dbb19cc48b210fc4fb35c305c8d81b56

`if let` 語法讓我們以一種不那麼冗長的方式結合 `if` 和 `let`，來處理匹配一個模式的值而忽略其他的值。考慮列表 6-6 中的程序，它匹配一個 `Option<u8>` 值並只希望當值為三時執行代碼：

```rust
let some_u8_value = Some(0u8);
match some_u8_value {
    Some(3) => println!("three"),
    _ => (),
}
```

<span class="caption">列表 6-6：`match` 只關心當值為 `Some(3)` 時執行代碼</span>

我們想要對 `Some(3)` 匹配進行操作不過不想處理任何其他 `Some<u8>` 值或 `None` 值。為了滿足 `match` 表達式（窮盡性）的要求，必須在處理完這唯一的成員後加上 `_ => ()`，這樣也要增加很多樣板代碼。

不過我們可以使用 `if let` 這種更短的方式編寫。如下代碼與列表 6-6 中的 `match` 行為一致：

```rust
# let some_u8_value = Some(0u8);
if let Some(3) = some_u8_value {
    println!("three");
}
```

`if let` 抓取通過 `=` 分隔的一個模式和一個表達式。它的工作方式與 `match` 相同，這裡的表達式對應 `match` 而模式則對應第一個分支。

使用 `if let` 意味著編寫更少代碼，更少的縮進和更少的樣板代碼。然而，這樣會失去 `match` 強制要求的窮盡性檢查。`match` 和 `if let` 之間的選擇依賴特定的環境以及增加簡潔度和失去窮盡性檢查的權衡取捨。

換句話說，可以認為 `if let` 是 `match` 的一個語法糖，它當值匹配某一模式時執行代碼而忽略所有其他值。

可以在 `if let` 中包含一個 `else`。`else` 塊中的代碼與 `match` 表達式中的 `_` 分支塊中的代碼相同，這樣的 `match` 表達式就等同於 `if let` 和 `else`。回憶一下列表 6-4 中 `Coin` 枚舉的定義，它的 `Quarter` 成員包含一個 `UsState` 值。如果想要計數所有不是 25 美分的硬幣的同時也報告 25 美分硬幣所屬的州，可以使用這樣一個 `match` 表達式：

```rust
# #[derive(Debug)]
# enum UsState {
#    Alabama,
#    Alaska,
# }
#
# enum Coin {
#    Penny,
#    Nickel,
#    Dime,
#    Quarter(UsState),
# }
# let coin = Coin::Penny;
let mut count = 0;
match coin {
    Coin::Quarter(state) => println!("State quarter from {:?}!", state),
    _ => count += 1,
}
```

或者可以使用這樣的 `if let` 和 `else` 表達式：

```rust
# #[derive(Debug)]
# enum UsState {
#    Alabama,
#    Alaska,
# }
#
# enum Coin {
#    Penny,
#    Nickel,
#    Dime,
#    Quarter(UsState),
# }
# let coin = Coin::Penny;
let mut count = 0;
if let Coin::Quarter(state) = coin {
    println!("State quarter from {:?}!", state);
} else {
    count += 1;
}
```

如果你的程序遇到一個使用 `match` 表達起來過於囉嗦的邏輯，記住 `if let` 也在你的 Rust 工具箱中。

## 總結

現在我們涉及到了如何使用枚舉來創建有一系列可列舉值的自定義類型。我們也展示了標準庫的 `Option<T>` 類型是如何幫助你利用類型系統來避免出錯的。當枚舉值包含數據時，你可以根據需要處理多少情況來選擇使用 `match` 或 `if let` 來抓取並使用這些值。

你的 Rust 程序現在能夠使用結構體和枚舉在自己的作用域內表現其內容了。在你的 API 中使用自定義類型保證了類型安全：編譯器會確保你的函數只會得到它期望的類型的值。

為了向你的用戶提供一個組織良好的 API，它使用起來很直觀並且只向用戶暴露他們確實需要的部分，那麼現在就讓我們轉向 Rust 的模塊系統吧。