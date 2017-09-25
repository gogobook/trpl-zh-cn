# 高級特徵

> [ch19-00-advanced-features.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch19-00-advanced-features.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

我們已經走得很遠了！現在我們已經學習了 99% 的編寫 Rust 時需要瞭解的內容。在第二十章開始的新項目之前，讓我們聊聊你可能會遇到的最後 1% 的內容。你可以隨意跳過本章並在遇到這些問題時再回過頭來；這裡將要學習的特徵在某些非常特定的情況下非常有用。我們並不想我們不想捨棄這些特性，但你會發現不會經常用到他們。

本章將覆蓋如下內容：

* 不安全 Rust：用於當需要捨棄 Rust 的某些保證並告訴編譯器你將會負責維持這些保證
* 高級生命週期：用於負責情形的額外的生命週期語法
* 高級 trait：與 trait 相關的關聯類型，預設類型參數，完全限定語法（fully qualified syntax），超（父）trait（supertraits）和 newtype 模式
* 高級類型：關於 newtype 模式的更多內容，類型別名，「never」 類型和動態大小類型
* 高級函數和閉包：函數指針和返回閉包

對所有人而言，這都是一個介紹 Rust 迷人特性的寶典！讓我們翻開它吧！