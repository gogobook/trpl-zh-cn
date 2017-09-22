# 枚舉和模式匹配

> [ch06-00-enums.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch06-00-enums.md)
> <br>
> commit 4f2dc564851dc04b271a2260c834643dfd86c724

本章介紹 **枚舉**（*enumerations*），也被稱作 *enums*。枚舉允許你通過列舉可能的值來定義一個類型。首先，我們會定義並使用一個枚舉來展示它是如何連同數據一起編碼信息的。接下來，我們會探索一個特別有用的枚舉，叫做 `Option`，它代表一個值要麼是一些值要麼什麼都不是。然後會講到 `match` 表達式中的模式匹配如何使對枚舉不同的值運行不同的代碼變得容易。最後會涉及到 `if let`，另一個簡潔方便處理代碼中枚舉的結構。

枚舉是一個很多語言都有的功能，不過不同語言中的功能各不相同。Rust 的枚舉與像 F#、OCaml 和 Haskell 這樣的函數式編程語言中的 **代數數據類型**（*algebraic data types*）最為相似。