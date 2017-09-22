# 通用集合類型

> [ch08-00-common-collections.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch08-00-common-collections.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

Rust 標準庫中包含一系列被稱為 **集合**（*collections*）的非常有用的數據結構。大部分其他數據類型都代表一個特定的值，不過集合可以包含多個值。不同於內建的數組和元組類型，這些集合指向的數據是儲存在堆上的，這意味著數據的數量不必在編譯時就可知並且可以隨著程序的運行增長或縮小。每種集合都有著不同能力和代價，而為所處的場景選擇合適的集合則是你將要始終發展的技能。在這一章裡，我們將詳細的瞭解三個在 Rust 程序中被廣泛使用的集合：

* *vector* 允許我們一個挨著一個地儲存一系列數量可變的值
* **字符串**（*string*）是一個字符的集合。我們之前見過 `String` 類型，現在將詳細介紹它。
* **哈希 map**（*hash map*）允許我們將值與一個特定的鍵（key）相關聯。這是一個叫做 *map* 的更通用的數據結構的特定實現。

對於標準庫提供的其他類型的集合，請查看[文檔][collections]。

[collections]: https://doc.rust-lang.org/std/collections

我們將討論如何創建和更新 vector、字符串和哈希 map，以及他們有什麼不同。