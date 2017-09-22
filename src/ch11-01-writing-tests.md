## 編寫測試

> [ch11-01-writing-tests.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch11-01-writing-tests.md)
> <br>
> commit db08b34db5f1c78b4866b391c802344ec94ecc38

測試用來驗證非測試的代碼是否按照期望的方式運行的 Rust 函數。測試函數體通常包括一些設置，運行需要測試的代碼，接著斷言其結果是我們所期望的。讓我們看看 Rust 提供的專門用來編寫測試的功能：`test` 屬性、一些巨集和 `should_panic` 屬性。

### 測試函數剖析

作為最簡單例子，Rust 中的測試就是一個帶有 `test` 屬性註解的函數。屬性（attribute）是關於 Rust 代碼片段的元數據：第五章中結構體中用到的 `derive` 屬性就是一個例子。為了將一個函數變成測試函數，需要在 `fn` 行之前加上 `#[test]`。當使用 `cargo test` 命令運行測試函數時，Rust 會構建一個測試執行者二進制文件用來運行標記了 `test` 屬性的函數並報告每一個測試是通過還是失敗。

第七章當使用 Cargo 新建一個庫項目時，它會自動為我們生成一個測試模塊和一個測試函數。這有助於我們開始編寫測試，因為這樣每次開始新項目時不必去查找測試函數的具體結構和語法了。當然也可以額外增加任意多的測試函數以及測試模塊！

我們將先通過對自動生成的測試模板做一些試驗來探索測試如何工作方面的一些內容，而不實際測試任何代碼。接著會寫一些真實的測試來調用我們編寫的代碼並斷言他們的行為是否正確。

讓我們創建一個新的庫項目 `adder`：

```text
$ cargo new adder
     Created library `adder` project
$ cd adder
```

adder 庫中 `src/lib.rs` 的內容應該看起來像這樣：

<span class="filename">文件名: src/lib.rs</span>

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
    }
}
```

<span class="caption">列表 11-1：由 `cargo new` 自動生成的測試模塊和函數</span>

現在讓我們暫時忽略 `tests` 模塊和 `#[cfg(test)]` 註解並只關注函數來瞭解其如何工作。注意 `fn` 行之前的 `#[test]`：這個屬性表明這是一個測試函數，這樣測試執行者就知道將其作為測試處理。也可以在 `tests` 模塊中擁有非測試的函數來幫助我們建立通用場景或進行常見操作，所以需要使用 `#[test]` 屬性標明哪些函數是測試。

這個函數目前沒有任何內容，這意味著沒有代碼會使測試失敗；一個空的測試是可以通過的！讓我們運行一下看看它是否通過了。

`cargo test` 命令會運行項目中所有的測試，如列表 11-2 所示：

```text
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished dev [unoptimized + debuginfo] target(s) in 0.22 secs
     Running target/debug/deps/adder-ce99bcc2479f4607

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

<span class="caption">列表 11-2：運行自動生成測試的輸出</span>

Cargo 編譯並運行了測試。在 `Compiling`、`Finished` 和 `Running` 這幾行之後，可以看到 `running 1 test` 這一行。下一行顯示了生成的測試函數的名稱，它是 `it_works`，以及測試的運行結果，`ok`。接著可以看到全體測試運行結果的總結：`test result: ok.` 意味著所有測試都通過了。`1 passed; 0 failed` 表示通過或失敗的測試數量。

這裡並沒有任何被標記為忽略的測試，所以總結表明 `0 ignored`。在下一部分關於運行測試的不同方式中會討論忽略測試。`0 measured` 統計是針對測試性能的性能測試的。性能測試（benchmark tests）在編寫本書時，仍只能用於 Rust 開發版（nightly Rust）。請查看附錄 D 來瞭解更多 Rust 開發版的信息。

測試輸出中以 `Doc-tests adder` 開頭的這一部分是所有文檔測試的結果。現在並沒有任何文檔測試，不過 Rust 會編譯任何出現在 API 文檔中的代碼示例。這個功能幫助我們使文檔和代碼保持同步！在第十四章的 「文檔註釋」 部分會講到如何編寫文檔測試。現在我們將忽略 `Doc-tests` 部分的輸出。

讓我們改變測試的名稱並看看這如何改變測試的輸出。給 `it_works` 函數起個不同的名字，比如 `exploration`，像這樣：

<span class="filename">Filename: src/lib.rs</span>

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn exploration() {
    }
}
```

並再次運行 `cargo test`。現在輸出中將出現 `exploration` 而不是 `it_works`：

```text
running 1 test
test tests::exploration ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

讓我們增加另一個測試，不過這一次是一個會失敗的測試！當測試函數中出現 panic 時測試就失敗了。每一個測試都在一個新線程中運行，當主線程發現測試線程異常了，就將對應測試標記為失敗。第九章講到了最簡單的造成 panic 的方法：調用 `panic!` 巨集！寫入新函數後 `src/lib.rs` 現在看起來如列表 11-3 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn exploration() {
    }

    #[test]
    fn another() {
        panic!("Make this test fail");
    }
}
```

<span class="caption">列表 11-3：增加第二個測試：他會失敗因為調用了 `panic!` 巨集</span>


再次 `cargo test` 運行測試。輸出應該看起來像列表 11-4，它表明 `exploration` 測試通過了而 `another` 失敗了：

```text
running 2 tests
test tests::exploration ... ok
test tests::another ... FAILED

failures:

---- tests::another stdout ----
	thread 'tests::another' panicked at 'Make this test fail', src/lib.rs:9
note: Run with `RUST_BACKTRACE=1` for a backtrace.

failures:
    tests::another

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured

error: test failed
```

<span class="caption">列表 11-4：一個測試通過和一個測試失敗的測試結果</span>

`test tests::another` 這一行是 `FAILED` 而不是 `ok` 了。在單獨測試結果和總結之間多了兩個新的部分：第一個部分顯示了測試失敗的詳細原因。在這個例子中，`another` 因為 `panicked at 'Make this test fail'` 而失敗，這位於 *src/lib.rs* 的第 9 行。下一部分僅僅列出了所有失敗的測試，這在很有多測試和很多失敗測試的詳細輸出時很有幫助。可以使用失敗測試的名稱來只運行這個測試，這樣比較方便調試；下一部分會講到更多運行測試的方法。

最後是總結行：總體上講，一個測試結果是 `FAILED` 的。有一個測試通過和一個測試失敗。

現在我們見過不同場景中測試結果是什麼樣子的了，再來看看除 `panic!` 之外的一些在測試中有幫助的巨集吧。

### 使用 `assert!` 巨集來檢查結果

`assert!` 巨集由標準庫提供，在希望確保測試中一些條件為 `true` 時非常有用。需要向 `assert!` 巨集提供一個計算為布爾值的參數。如果值是 `true`，`assert!` 什麼也不做同時測試會通過。如果值為 `false`，`assert!` 調用 `panic!` 巨集，這會導致測試失敗。這是一個幫助我們檢查代碼是否以期望的方式運行的巨集。

回憶一下第五章中，列表 5-9 中有一個 `Rectangle` 結構體和一個 `can_hold` 方法，在列表 11-5 中再次使用他們。將他們放進 *src/lib.rs* 而不是 *src/main.rs* 並使用 `assert!` 巨集編寫一些測試。

<span class="filename">文件名: src/lib.rs</span>

```rust
#[derive(Debug)]
pub struct Rectangle {
    length: u32,
    width: u32,
}

impl Rectangle {
    pub fn can_hold(&self, other: &Rectangle) -> bool {
        self.length > other.length && self.width > other.width
    }
}
```

<span class="caption">列表 11-5：第五章中 `Rectangle` 結構體和其 `can_hold` 方法</span>

`can_hold` 方法返回一個布爾值，這意味著它完美符合 `assert!` 巨集的使用場景。在列表 11-6 中，讓我們編寫一個 `can_hold` 方法的測試來作為練習，這裡創建一個長為 8 寬為 7 的 `Rectangle` 實例，並假設它可以放得下另一個長為 5 寬為 1 的 `Rectangle` 實例：

<span class="filename">文件名: src/lib.rs</span>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn larger_can_hold_smaller() {
        let larger = Rectangle { length: 8, width: 7 };
        let smaller = Rectangle { length: 5, width: 1 };

        assert!(larger.can_hold(&smaller));
    }
}
```

<span class="caption">列表 11-6：一個 `can_hold` 的測試，檢查一個較大的矩形確實能放得下一個較小的矩形</span>

注意在 `tests` 模塊中新增加了一行：`use super::*;`。`tests` 是一個普通的模塊，它遵循第七章介紹的通常的可見性規則。因為這是一個內部模塊，需要將外部模塊中被測試的代碼引入到內部模塊的作用域中。這裡選擇使用全局導入使得外部模塊定義的所有內容在 `tests` 模塊中都是可用的。

我們將測試命名為 `larger_can_hold_smaller`，並創建所需的兩個 `Rectangle` 實例。接著調用 `assert!` 巨集並傳遞 `larger.can_hold(&smaller)` 調用的結果作為參數。這個表達式預期會返回 `true`，所以測試應該通過。讓我們拭目以待！

```text
running 1 test
test tests::larger_can_hold_smaller ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

它確實通過了！再來增加另一個測試，這一回斷言一個更小的矩形不能放下一個更大的矩形：

<span class="filename">文件名: src/lib.rs</span>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn larger_can_hold_smaller() {
        let larger = Rectangle { length: 8, width: 7 };
        let smaller = Rectangle { length: 5, width: 1 };

        assert!(larger.can_hold(&smaller));
    }

    #[test]
    fn smaller_cannot_hold_larger() {
        let larger = Rectangle { length: 8, width: 7 };
        let smaller = Rectangle { length: 5, width: 1 };

        assert!(!smaller.can_hold(&larger));
    }
}
```

因為這裡 `can_hold` 函數的正確結果是 `false`，我們需要將這個結果取反後傳遞給 `assert!` 巨集。這樣的話，測試就會通過而 `can_hold` 將返回`false`：

```text
running 2 tests
test tests::smaller_cannot_hold_larger ... ok
test tests::larger_can_hold_smaller ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured
```

兩個通過的測試！現在讓我們看看如果引入一個 bug 的話測試結果會發生什麼。將 `can_hold` 方法中比較長度時本應使用大於號的地方改成小於號：

```rust
#[derive(Debug)]
pub struct Rectangle {
    length: u32,
    width: u32,
}

impl Rectangle {
    pub fn can_hold(&self, other: &Rectangle) -> bool {
        self.length < other.length && self.width > other.width
    }
}
```

現在運行測試會產生：

```text
running 2 tests
test tests::smaller_cannot_hold_larger ... ok
test tests::larger_can_hold_smaller ... FAILED

failures:

---- tests::larger_can_hold_smaller stdout ----
	thread 'tests::larger_can_hold_smaller' panicked at 'assertion failed:
    larger.can_hold(&smaller)', src/lib.rs:22
note: Run with `RUST_BACKTRACE=1` for a backtrace.

failures:
    tests::larger_can_hold_smaller

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured
```

我們的測試捕獲了 bug！因為 `larger.length` 是 8 而 `smaller.length` 是 5，`can_hold` 中的長度比較現在因為 8 不小於 5 而返回 `false`。

### 使用 `assert_eq!` 和 `assert_ne!` 巨集來測試相等

測試功能的一個常用方法是將需要測試代碼的值與期望值做比較，並檢查是否相等。可以通過向 `assert!` 巨集傳遞一個使用 `==` 運算符的表達式來做到。不過這個操作實在是太常見了，以至於標註庫提供了一對巨集來方便處理這些操作：`assert_eq!` 和 `assert_ne!`。這兩個巨集分別比較兩個值是相等還是不相等。當斷言失敗時他們也會打印出這兩個值具體是什麼，以便於觀察測試 **為什麼** 失敗，而 `assert!` 只會打印出它從 `==` 表達式中得到了 `false` 值，而不是導致 `false` 的兩個值。

列表 11-7 中，讓我們編寫一個對其參數加二並返回結果的函數 `add_two`。接著使用 `assert_eq!` 巨集測試這個函數：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_adds_two() {
        assert_eq!(4, add_two(2));
    }
}
```

<span class="caption">列表 11-7：使用 `assert_eq!` 巨集測試 `add_two`</span>

測試通過了！

```text
running 1 test
test tests::it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

傳遞給 `assert_eq!` 巨集的第一個參數，4，等於調用 `add_two(2)` 的結果。我們將會看到這個測試的那一行說 `test tests::it_adds_two ... ok`，`ok` 表明測試通過了！

在代碼中引入一個 bug 來看看使用 `assert_eq!` 的測試失敗是什麼樣的。修改 `add_two` 函數的實現使其加 3：

```rust
pub fn add_two(a: i32) -> i32 {
    a + 3
}
```

再次運行測試：

```text
running 1 test
test tests::it_adds_two ... FAILED

failures:

---- tests::it_adds_two stdout ----
	thread 'tests::it_adds_two' panicked at 'assertion failed: `(left ==
    right)` (left: `4`, right: `5`)', src/lib.rs:11
note: Run with `RUST_BACKTRACE=1` for a backtrace.

failures:
    tests::it_adds_two

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured
```

測試捕獲到了 bug！`it_adds_two` 測試失敗並顯示信息 `` assertion failed: `(left == right)` (left: `4`, right: `5`) ``。這個信息有助於我們開始調試：它說 `assert_eq!` 的 `left` 參數是 4，而 `right` 參數，也就是 `add_two(2)` 的結果，是 5。

注意在一些語言和測試框架中，斷言兩個值相等的函數的參數叫做 `expected` 和 `actual`，而且指定參數的順序是需要注意的。然而在 Rust 中，他們則叫做 `left` 和 `right`，同時指定期望的值和被測試代碼產生的值的順序並不重要。這個測試中的斷言也可以寫成 `assert_eq!(add_two(2), 4)`，這時錯誤信息會變成 `` assertion failed: `(left == right)` (left: `5`, right: `4`) ``。

`assert_ne!` 巨集在傳遞給它的兩個值不相等時通過而在相等時失敗。這個巨集在代碼按照我們期望運行時不確定值 **會** 是什麼，不過知道他們絕對 **不會**是什麼的時候最有用處。例如，如果一個函數確定會以某種方式改變其輸出，不過這種方式由運行測試是星期幾來決定，這時最好的斷言可能就是函數的輸出不等於其輸入。

`assert_eq!` 和 `assert_ne!` 巨集在底層分別使用了 `==` 和 `!=`。當斷言失敗時，這些巨集會使用調試格式打印出其參數，這意味著被比較的值必需實現了 `PartialEq` 和 `Debug` trait。所有的基本類型和大部分標準庫類型都實現了這些 trait。對於自定義的結構體和枚舉，需要實現 `PartialEq` 才能斷言他們的值是否相等。需要實現 `Debug` 才能在斷言失敗時打印他們的值。因為這兩個 trait 都是可推導 trait，如第五章所提到的，通常可以直接在結構體或枚舉上添加 `#[derive(PartialEq, Debug)]` 註解。附錄 C 中有更多關於這些和其他可推導 trait 的詳細信息。

### 自定義錯誤信息

也可以向 `assert!`、`assert_eq!` 和 `assert_ne!` 巨集傳遞一個可選的參數來增加用於打印的自定義錯誤信息。任何在 `assert!` 必需的一個參數和 `assert_eq!` 和 `assert_ne!` 必需的兩個參數之後指定的參數都會傳遞給第八章講到的 `format!` 巨集，所以可以傳遞一個包含 `{}` 佔位符的格式字符串和放入佔位符的值。自定義信息有助於記錄斷言的意義，這樣到測試失敗時，就能更好的理解代碼出了什麼問題。

例如，比如說有一個根據人名進行問候的函數，而我們希望測試將傳遞給函數的人名顯示在輸出中：

<span class="filename">Filename: src/lib.rs</span>

```rust
pub fn greeting(name: &str) -> String {
    format!("Hello {}!", name)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn greeting_contains_name() {
        let result = greeting("Carol");
        assert!(result.contains("Carol"));
    }
}
```

這個程序的需求還沒有被確定，而我們非常確定問候開始的 `Hello` 文本不會改變。我們決定並不想在人名改變時
不得不更新測試，所以相比檢查 `greeting` 函數返回的確切的值，我們將僅僅斷言輸出的文本中包含輸入參數。

讓我們通過將 `greeting` 改為不包含 `name` 來在代碼中引入一個 bug 來測試失敗時是怎樣的，

```rust
pub fn greeting(name: &str) -> String {
    String::from("Hello!")
}
```

運行測試會產生：

```text
running 1 test
test tests::greeting_contains_name ... FAILED

failures:

---- tests::greeting_contains_name stdout ----
	thread 'tests::greeting_contains_name' panicked at 'assertion failed:
    result.contains("Carol")', src/lib.rs:12
note: Run with `RUST_BACKTRACE=1` for a backtrace.

failures:
    tests::greeting_contains_name
```

這僅僅告訴了我們斷言失敗了和失敗的行號。一個更有用的錯誤信息應該打印出從 `greeting` 函數得到的值。讓我們改變測試函數來使用一個由包含佔位符的格式字符串和從 `greeting` 函數取得的值組成的自定義錯誤信息：

```rust
#[test]
fn greeting_contains_name() {
    let result = greeting("Carol");
    assert!(
        result.contains("Carol"),
        "Greeting did not contain name, value was `{}`", result
    );
}
```

現在如果再次運行測試，將會看到更有價值的錯誤信息：

```text
---- tests::greeting_contains_name stdout ----
	thread 'tests::greeting_contains_name' panicked at 'Greeting did not contain
    name, value was `Hello`', src/lib.rs:12
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

可以在測試輸出中看到所取得的確切的值，這會幫助我們理解發生了什麼而不是期望發生什麼。

### 使用 `should_panic` 檢查 panic

除了檢查代碼是否返回期望的正確的值之外，檢查代碼是否按照期望處理錯誤情況也是很重要的。例如，考慮第九章列表 9-8 創建的 `Guess` 類型。其他使用 `Guess` 的代碼依賴於 `Guess` 實例只會包含 1 到 100 的值的保證。可以編寫一個測試來確保創建一個超出範圍的值的 `Guess` 實例會 panic。

可以通過對函數增加另一個屬性 `should_panic` 來實現這些。這個屬性在函數中的代碼 panic 時會通過，而在其中的代碼沒有 panic 時失敗。

列表 11-8 展示了如何編寫一個測試來檢查 `Guess::new` 按照我們的期望出現的錯誤情況：

<span class="filename">文件名: src/lib.rs</span>

```rust
struct Guess {
    value: u32,
}

impl Guess {
    pub fn new(value: u32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }

        Guess {
            value
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```

<span class="caption">列表 11-8：測試會造成 `panic!` 的條件</span>

`#[should_panic]` 屬性位於 `#[test]` 之後和對應的測試函數之前。讓我們看看測試通過時它時什麼樣子：

```text
running 1 test
test tests::greater_than_100 ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

看起來不錯！現在在代碼中引入 bug，移除 `new` 函數在值大於 100 時會 panic 的條件：

```rust
# struct Guess {
#     value: u32,
# }
#
impl Guess {
    pub fn new(value: u32) -> Guess {
        if value < 1  {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }

        Guess {
            value
        }
    }
}
```

如果運行列表 11-8 的測試，它會失敗：

```text
running 1 test
test tests::greater_than_100 ... FAILED

failures:

failures:
    tests::greater_than_100

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured
```

這回並沒有得到非常有用的信息，不過一旦我們觀察測試函數，會發現它標註了 `#[should_panic]`。這個錯誤意味著代碼中函數 `Guess::new(200)` 並沒有產生 panic。

然而 `should_panic` 測試可能是非常含糊不清的，因為他們只是告訴我們代碼並沒有產生 panic。`should_panic` 甚至在測試因為其他不同的原因而不是我們期望發生的情況而 panic 時也會通過。為了使 `should_panic` 測試更精確，可以給 `should_panic` 屬性增加一個可選的 `expected` 參數。測試工具會確保錯誤信息中包含其提供的文本。例如，考慮列表 11-9 中修改過的 `Guess`，這裡 `new` 函數根據其值是過大還或者過小而提供不同的 panic 信息：

<span class="filename">文件名: src/lib.rs</span>

```rust
struct Guess {
    value: u32,
}

impl Guess {
    pub fn new(value: u32) -> Guess {
        if value < 1 {
            panic!("Guess value must be greater than or equal to 1, got {}.",
                   value);
        } else if value > 100 {
            panic!("Guess value must be less than or equal to 100, got {}.",
                   value);
        }

        Guess {
            value
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected = "Guess value must be less than or equal to 100")]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```

<span class="caption">列表 11-9：一個會帶有特定錯誤信息的 `panic!` 條件的測試</span>

這個測試會通過，因為 `should_panic` 屬性中 `expected` 參數提供的值是 `Guess::new` 函數 panic 信息的子字符串。我們可以指定期望的整個 panic 信息，在這個例子中是 `Guess value must be less than or equal to 100, got 200.`。這依賴於 panic 有多獨特或動態，和你希望測試有多準確。在這個例子中，錯誤信息的子字符串足以確保函數在 `else if value > 100` 的情況下運行。

為了觀察帶有 `expected` 信息的 `should_panic` 測試失敗時會發生什麼，讓我們再次引入一個 bug，將 `if value < 1` 和 `else if value > 100` 的代碼塊對換：

```rust
if value < 1 {
    panic!("Guess value must be less than or equal to 100, got {}.", value);
} else if value > 100 {
    panic!("Guess value must be greater than or equal to 1, got {}.", value);
}
```

這一次運行 `should_panic` 測試，它會失敗：

```text
running 1 test
test tests::greater_than_100 ... FAILED

failures:

---- tests::greater_than_100 stdout ----
	thread 'tests::greater_than_100' panicked at 'Guess value must be greater
    than or equal to 1, got 200.', src/lib.rs:10
note: Run with `RUST_BACKTRACE=1` for a backtrace.
note: Panic did not include expected string 'Guess value must be less than or
equal to 100'

failures:
    tests::greater_than_100

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured
```

錯誤信息表明測試確實如期望 panic 了，不過 panic 信息是 `did not include expected string 'Guess value must be less than or equal to 100'`。可以看到我們得到的 panic 信息，在這個例子中是 `Guess value must be greater than or equal to 1, got 200.`。這樣就可以開始尋找 bug 在哪了！

現在我們講完了編寫測試的方法，讓我們看看運行測試時會發生什麼並討論可以用於 `cargo test` 的不同選項。