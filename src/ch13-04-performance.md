## 性能對比：循環 VS 迭代器

> [ch13-04-performance.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch13-04-performance.md)
> <br>
> commit 40910f557c328858f230123d1234c1cb3029dda3

為了決定使用哪個實現，我們需要知道哪個版本的 `search` 函數更快：直接使用 `for` 循環的版本還是使用迭代器的版本。

我們運行了一個性能測試，通過將阿瑟‧柯南‧道爾的「福爾摩斯探案集」的全部內容加載進 `String` 並尋找其中的單詞 「the」。如下是 `for` 循環版本和迭代器版本的 `search` 函數的性能測試結果：

```text
test bench_search_for  ... bench:  19,620,300 ns/iter (+/- 915,700)
test bench_search_iter ... bench:  19,234,900 ns/iter (+/- 657,200)
```

結果迭代器版本還要稍微快一點！這裡我們將不會查看性能測試的代碼，我們的目的並不是為了證明他們是完全等同的，而是得出一個怎樣比較這兩種實現方式性能的基本思路。對於一個更全面的性能測試，將會檢查不同長度的文本、不同的搜索單詞、不同長度的單詞和所有其他的可變情況。這裡所要表達的是：迭代器，作為一個高級的抽象，被編譯成了與手寫的底層代碼大體一致性能代碼。迭代器是 Rust 的 **零成本抽象**（*zero-cost abstractions*）之一，它意味著抽象並不會強加運行時開銷，它與本賈尼‧斯特勞斯特盧普，C++ 的設計和實現者所定義的 **零開銷**（*zero-overhead*）如出一轍：

> In general, C++ implementations obey the zero-overhead principle: What you don't use, you don't pay for. And further: What you do use, you couldn't hand code any better.
>
> - Bjarne Stroustrup "Foundations of C++"
>
> 從整體來說，C++ 的實現遵循了零開銷原則：你不需要的，無需為他們買單。更有甚者的是：你需要的時候，也不可能找到其他更好的代碼了。
>
> - 本賈尼‧斯特勞斯特盧普 "Foundations of C++"

作為另一個例子，這裡有一些取自於音頻解碼器的代碼。這些代碼使用迭代器鏈來對作用域中的三個變數進行了某種數學計算：一個叫 `buffer` 的數據 slice、一個有 12 個元素的數組 `coefficients`、和一個代表位移位數的 `qlp_shift`。例子中聲明了這些變數但並沒有提供任何值；雖然這些代碼在其上下文之外沒有什麼意義，不過仍是一個簡明的現實中的例子，來展示 Rust 如何將高級概念轉換為底層代碼：

```rust
let buffer: &mut [i32];
let coefficients: [i64; 12];
let qlp_shift: i16;

for i in 12..buffer.len() {
    let prediction = coefficients.iter()
                                 .zip(&buffer[i - 12..i])
                                 .map(|(&c, &s)| c * s as i64)
                                 .sum::<i64>() >> qlp_shift;
    let delta = buffer[i];
    buffer[i] = prediction as i32 + delta;
}
```

為了計算 `prediction` 的值，這些代碼遍歷了 `coefficients` 中的 12 個值，使用 `zip` 方法將係數與 `buffer` 的前 12 個值組合在一起。接著將每一對值相乘，再將所有結果相加，然後將總和右移 `qlp_shift` 位。

像音頻解碼器這樣的程序通常最看重計算的性能。這裡，我們創建了一個迭代器，使用了兩個適配器，接著消費了其值。Rust 代碼將會被編譯為什麼樣的彙編代碼呢？好吧，在編寫本書的這個時候，它被編譯成與手寫的相同的彙編代碼。遍歷 `coefficients` 的值完全用不到循環：Rust 知道這裡會迭代 12 次，所以它「展開」（unroll）了循環。展開是一種移除循環的控制代碼開銷並替換為每個迭代中的重複代碼的優化。

所有的係數都被儲存在了寄存器中，這意味著訪問他們非常快。這裡也沒有運行時數組訪問邊界檢查。所有這些 Rust 能夠提供的優化使得結果代碼極為高效。

現在知道這些了，請放心大膽的使用迭代器和閉包吧！他們使得代碼看起來更高級，但並不為此引入運行時性能損失。

## 總結

閉包和迭代器是 Rust 受函數式編程語言觀念所啟發的功能。他們對 Rust 以底層的性能來明確的表達高級概念的能力有很大貢獻。閉包和迭代器的實現達到了不影響運行時性能的程度。這正是 Rust 竭力提供零成本抽象的目標的一部分。

現在我們改進了我們 I/O 項目的（代碼）表現力，讓我們看一看更多 `cargo` 的功能，他們將幫助我們準備好將項目分享給世界。