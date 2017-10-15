# 使用結構體組織相關聯的數據

> [ch05-00-structs.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch05-00-structs.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

`struct`，是 *structure* 的縮寫，是一個允許我們命名並將多個相關值包裝進一個有意義的組合的自定義類型。如果你有來自一個物件導向編程語言背景，`struct` 就像物件中的數據屬性（字段）。在本章中，我們會比較元組與結構體，展示如何使用結構體，並討論如何在結構體上定義方法和關聯函數來指定與結構體數據相關的行為。結構體和 **枚舉**（*enum*）（將在第六章講到）是為了充分利用 Rust 的編譯時類型檢查來在程序範圍內創建新類型的基本組件。
(譯註: 可能可以把結構體視為只有屬性的類別。)