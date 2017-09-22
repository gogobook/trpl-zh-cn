# 介紹

> [ch01-00-introduction.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch01-00-introduction.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

歡迎閱讀 「Rust 程序設計語言」，一本介紹 Rust 的書。Rust 是一門著眼於安全、速度和並發的編程語言。其程序設計兼顧底層語言的性能與控制，並不失高級語言強大的抽象能力。其特性適合那些有類 C 語言經驗，正在尋找更安全的替代品的開發者；同樣適合有著類 Python 語言背景，尋求在不犧牲表現力的前提下，編寫更高性能代碼的開發者。

Rust 編譯時執行絕大部分的安全檢查和內存管理決策，對運行時性能的影響微不足道。這使其在其他語言不擅長的應用場景中得以大顯身手：可預測時間和空間需求的程序，嵌入到其他語言中，以及編寫如設備驅動和操作系統這樣的底層代碼，。Rust 也很擅長 web 程序：它驅動著 Rust 包註冊網站（package
registry site），[crates.io]！我們由衷期待**你**使用 Rust 進行創作。

[crates.io]: https://crates.io/

本書的目標讀者至少應瞭解一門其它編程語言。閱讀本書之後，你應該能自如的編寫 Rust 程序。我們將通過短小精悍、前後呼應的例子來學習 Rust，並展示其多樣功能的使用方法，同時瞭解他們幕後如何運行。

## 為本書做出貢獻

本書是開源的。如果你發現任何錯誤，不要猶豫，[在 GitHub 上][on GitHub]發起 issue 或提交 pull request。請查看 [CONTRIBUTING.md] 獲取更多信息。

[on GitHub]: https://github.com/rust-lang/book
[CONTRIBUTING.md]: https://github.com/rust-lang/book/blob/master/CONTRIBUTING.md

> 譯者註：譯本的 [GitHub 倉庫][trpl-zh-cn]，同樣歡迎 Issue 和 PR :)

[trpl-zh-cn]: https://github.com/KaiserY/trpl-zh-cn