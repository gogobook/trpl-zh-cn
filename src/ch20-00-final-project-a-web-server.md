# 最後的項目: 構建多線程 web server

> [ch20-00-final-project-a-web-server.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch20-00-final-project-a-web-server.md)
> <br>
> commit 08e50d5e147ad290d88efd5c58365000723626df

這是一次漫長的旅途，不過我們做到了！這一章便是本書的結束。離別是如此甜蜜的悲傷。不過在我們結束之前，再來一起構建另一個項目，來展示最後幾章所學，同時複習更早的章節。

下面就是我們將要做的：一個簡單的 web server：

![hello from rust](img/trpl20-01.png)

為此我們將：

1. 學習一些 TCP 與 HTTP 知識
2. 在套接字（socket）上監聽 TCP 請求
3. 解析少量的 HTTP 請求
4. 創建一個合適的 HTTP 響應
5. 通過線程池改善 server 的吞吐量

在開始之前，需要提到一點：如果你曾在生產環境中編寫過這樣的代碼，還有很多更好的做法。特別需要指出的是，crates.io 上提供了很多更完整健壯的 web server 和 線程池實現，要比我們編寫的好很多。

然而，本章的目的在於學習，而不是走捷徑。因為 Rust 是一個系統編程語言，能夠選擇處理什麼層次的抽象。我們能夠選擇比其他語言可能或可用的層次更低的層次。所以我們將自己編寫一個基礎的 HTTP server 和線程池，以便學習將來可能用到的 crate 背後的通用理念和技術。