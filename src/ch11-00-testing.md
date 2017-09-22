# 測試

> [ch11-00-testing.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch11-00-testing.md)
> <br>
> commit 1047433147b27d19e5acc068a0ebca5782d64f99

> Program testing can be a very effective way to show the presence of bugs, but it is hopelessly inadequate for showing their absence.
>
> Edsger W. Dijkstra, "The Humble Programmer" (1972)
>
> 軟件測試是證明 bug 存在的有效方法，而證明它們不存在時則顯得令人絕望的不足。
>
> Edsger W. Dijkstra，【謙卑的程序員】（1972）

程序的正確性意味著代碼如我們期望的那樣運行。Rust 是一個非常注重正確性的編程語言，不過正確性是一個難以證明的複雜主題。Rust 的類型系統在此問題上下了很大的功夫，不過它不可能捕獲所有種類的錯誤。為此，Rust 也在語言本身包含了編寫軟件測試的支持。

例如，我們可以編寫一個叫做 `add_two` 的將傳遞給它的值加二的函數。它的簽名有一個整型參數並返回一個整型值。當實現和編譯這個函數時，Rust 會進行所有目前我們已經見過的的類型檢查和借用檢查，例如，這些檢查會確保我們不會傳遞 `String` 或無效的引用給這個函數。Rust 所 **不能** 檢查的是這個函數是否會準確的完成我們期望的工作：返回參數加二後的值，而不是比如說參數加 10 或減 50 的值！這也就是測試出場的地方。

我們可以編寫測試斷言，比如說，當傳遞 `3` 給 `add_two` 函數時，應該得到 `5`。當對代碼進行修改時可以運行測試來確保任何現存的正確行為沒有被改變。

測試是一項複雜的技能，而且我們也不能期望在一本書的一個章節中就涉及到編寫好的測試的所有內容，所以這裡僅僅討論 Rust 測試功能的機制。我們會講到編寫測試時會用到的註解和巨集，Rust 提供用來運行測試的默認行為和選項，以及如何將測試組織成單元測試和集成測試。