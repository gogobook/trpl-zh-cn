# 通用編程概念

> [ch03-00-common-programming-concepts.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch03-00-common-programming-concepts.md)
> <br>
> commit 04aa3a45eb72855b34213703718f50a12a3eeec8

本章涉及一些幾乎所有編程語言都有的概念，以及他們在 Rust 中是如何工作的。很多編程語言的核心概念都是共通的，本章中展示的概念都不是 Rust 特有的，不過我們會在 Rust 環境中討論他們，解釋他們的使用習慣。

具體的，我們將會學習變數，基本類型，函數，註釋和控制流。這些基礎知識將會出現在每一個 Rust 程序中，提早學習這些概念會使你擁有堅實的起步基礎。

> ### 關鍵字
>
> Rust 語言有一系列保留的 **關鍵字**（*keywords*），只能由語言本身使用，像大部分語言一樣。你不能使用這些關鍵字作為變數或函數的名稱，大部分關鍵字有特殊的意義，並被用來完成 Rust 程序中的各種任務；一些關鍵字目前沒有相應的功能，是為將來可能添加的功能保留的。可以在附錄 A 中找到關鍵字的列表。