## 測試的組織結構

> [ch11-03-test-organization.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch11-03-test-organization.md)
> <br>
> commit 0665bc5646339a6bcda21838f46d4357b9435e75

正如之前提到的，測試是一個很廣泛的學科，而且不同的開發者也採用不同的技術和組織。Rust 社區傾向於根據測試的兩個主要分類來考慮問題：**單元測試**（*unit tests*）與 **集成測試**（*integration tests*）。單元測試傾向於更小而更專注，在隔離的環境中一次測試一個模組，也可以測試私有接口。集成測試對於你的庫來說則完全是外部的。他們與其他用戶使用相同的方式使用你的代碼，他們只針對公有接口而且每個測試都會測試多個模組。

編寫這兩類測試對於從獨立和整體的角度保證你的庫符合期望是非常重要的。

### 單元測試

單元測試的目的是在與其他部分隔離的環境中測試每一個單元的代碼，以便於快速而準確的定位代碼位於何處和是否符合預期。單元測試位於 *src* 目錄中，與他們要測試的代碼存在於相同的文件中。傳統做法是在每個文件中創建包含測試函數的 `tests` 模組，並使用 `cfg(test)` 標註模組。

#### 測試模組和`cfg(test)`

測試模組的 `#[cfg(test)]` 註解告訴 Rust 只在執行 `cargo test` 時才編譯和運行測試代碼，而在運行 `cargo build` 時不這麼做。這在只希望構建庫的時候可以節省編譯時間，並能節省編譯產物的空間因為他們並沒有包含測試。我們將會看到因為集成測試位於另一個文件夾，他們並不需要 `#[cfg(test)]` 註解。但是因為單元測試位於與源碼相同的文件中，所以使用 `#[cfg(test)]` 來指定他們不應該被包含進編譯產物中。

還記得本章第一部分新建的 `adder` 項目嗎？Cargo 為我們生成了如下代碼：

<span class="filename">文件名: src/lib.rs</span>

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
    }
}
```

這裡自動生成了測試模組。`cfg` 屬性代表 *configuration* ，它告訴 Rust 其之後的項只被包含進特定配置中。在這個例子中，配置是 `test`，Rust 所提供的用於編譯和運行測試的配置。通過使用這個屬性，Cargo 只會在我們主動使用 `cargo test` 運行測試時才編譯測試代碼。除了標註為 `#[test]` 的函數之外，還包括測試模組中可能存在的幫助函數。

#### 測試私有函數

測試社區中一直存在關於是否應該對私有函數進行單元測試的論戰，而其他語言中難以甚至不可能測試私有函數。不過無論你堅持哪種測試意識形態，Rust 的私有性規則確實允許你測試私有函數，由於私有性規則。考慮列表 11-12 中帶有私有函數 `internal_adder` 的代碼：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub fn add_two(a: i32) -> i32 {
    internal_adder(a, 2)
}

fn internal_adder(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn internal() {
        assert_eq!(4, internal_adder(2, 2));
    }
}
```

<span class="caption">列表 11-12：測試私有函數</span>

注意 `internal_adder` 函數並沒有標記為 `pub`，不過因為測試也不過是 Rust 代碼同時 `tests` 也僅僅是另一個模組，我們完全可以在測試中導入和調用 `internal_adder`。如果你並不認為私有函數應該被測試，Rust 也不會強迫你這麼做。

### 集成測試

在 Rust 中，集成測試對於需要測試的庫來說是完全獨立。他們同其他代碼一樣使用庫文件，這意味著他們只能調用作為庫公有 API 的一部分函數。他們的目的是測試庫的多個部分能否一起正常工作。每個能單獨正確運行的代碼單元集成在一起也可能會出現問題，所以集成測試的覆蓋率也是很重要的。為了創建集成測試，首先需要一個 *tests* 目錄。

#### *tests* 目錄

為了編寫集成測試，需要在項目根目錄創建一個 *tests* 目錄，與 *src* 同級。Cargo 知道如何去尋找這個目錄中的集成測試文件。接著可以隨意在這個目錄中創建任意多的測試文件，Cargo 會將每一個文件當作單獨的 crate 來編譯。

讓我們試一試吧！保留列表 11-12 中 *src/lib.rs* 的代碼。創建一個 *tests* 目錄，新建一個文件 *tests/integration_test.rs*，並輸入列表 11-13 中的代碼。

<span class="filename">文件名: tests/integration_test.rs</span>

```rust
extern crate adder;

#[test]
fn it_adds_two() {
    assert_eq!(4, adder::add_two(2));
}
```

<span class="caption">列表 11-13：一個 `adder` crate 中函數的集成測試</span>

我們在頂部增加了 `extern crate adder`，這在單元測試中是不需要的。這是因為每一個 `tests` 目錄中的測試文件都是完全獨立的 crate，所以需要在每一個文件中導入庫。集成測試就像其他庫使用者那樣通過導入 crate 並只使用公有 API。

並不需要將 *tests/integration_test.rs* 中的任何代碼標註為 `#[cfg(test)]`。Cargo 對 `tests` 文件夾特殊處理並只會在運行 `cargo test` 時編譯這個目錄中的文件。現在就試試運行 `cargo test`：

```text
cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished dev [unoptimized + debuginfo] target(s) in 0.31 secs
     Running target/debug/deps/adder-abcabcabc

running 1 test
test tests::internal ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

     Running target/debug/deps/integration_test-ce99bcc2479f4607

running 1 test
test it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

現在有了三個部分的輸出：單元測試、集成測試和文檔測試。第一部分單元測試與我們之前見過的一樣：每一個單元測試一行（列表 11-12 中有一個叫做 `internal` 的測試），接著是一個單元測試的總結行。

集成測試部分以行 `Running target/debug/deps/integration-test-ce99bcc2479f4607`（輸出最後的哈希值可能不同）開頭。接著是每一個集成測試中的測試函數一行，以及一個就在 `Doc-tests adder` 部分開始之前的集成測試的總結行。

注意在任意 *src* 文件中增加更多單元測試函數會增加更多單元測試部分的測試結果行。在我們創建的集成測試文件中增加更多測試函數會增加更多集成測試部分的行。每一個集成測試文件有其自己的部分，所以如果在 *tests* 目錄中增加更多文件，這裡就會有更多集成測試部分。

我們仍然可以通過指定測試函數的名稱作為 `cargo test` 的參數來運行特定集成測試。為了運行某個特定集成測試文件中的所有測試，使用 `cargo test` 的 `--test` 後跟文件的名稱：

```text
$ cargo test --test integration_test
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running target/debug/integration_test-952a27e0126bb565

running 1 test
test it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

這些只是 *tests* 目錄中我們指定的文件中的測試。

#### 集成測試中的子模組

隨著集成測試的增加，你可能希望在 `tests` 目錄增加更多文件，例如根據測試的功能來將測試分組。正如我們之前提到的，每一個 *tests* 目錄中的文件都被編譯為單獨的 crate。

將每個集成測試文件當作其自己的 crate 來對待有助於創建更類似與終端用戶使用 crate 那樣的單獨的作用域。然而，這意味著考慮到第七章學習的如何將代碼分隔進模組和文件的知識，*tests* 目錄中的文件不能像 *src* 中的文件那樣共享相同的行為，。

對於 *tests* 目錄中不同文件的行為，通常在如果有一系列有助於多個集成測試文件的幫助函數，而你嘗試遵循第七章的步驟將他們提取到一個通用的模組中時顯得很明顯。例如，如果我們創建了 *tests/common.rs* 並將 `setup` 函數放入其中，這裡將放入一些希望能夠在多個測試文件的多個測試函數中調用的代碼：

<span class="filename">文件名: tests/common.rs</span>

```rust
pub fn setup() {
    // setup code specific to your library's tests would go here
}
```

如果再次運行測試，將會在測試結果中看到一個對應 *common.rs* 文件的新部分，即便這個文件並沒有包含任何測試函數，或者沒有任何地方調用了 `setup` 函數：

```text
running 1 test
test tests::internal ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

     Running target/debug/deps/common-b8b07b6f1be2db70

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured

     Running target/debug/deps/integration_test-d993c68b431d39df

running 1 test
test it_adds_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

`common` 出現在測試結果中並顯示 `running 0 tests`，這不是我們想要的；我們只是希望能夠在其他集成測試文件中分享一些代碼罷了。

為了使 `common` 不出現在測試輸出中，需要使用第七章學習到的另一個將代碼提取到文件的方式：不再創建 *tests/common.rs*，而是創建 *tests/common/mod.rs*。當將 `setup` 代碼移動到 *tests/common/mod.rs* 並去掉 *tests/common.rs* 文件之後，測試輸出中將不會出現這一部分。*tests* 目錄中的子目錄不會被作為單獨的 crate 編譯或作為一部分出現在測試輸出中。

一旦擁有了 *tests/common/mod.rs*，就可以將其作為模組來在任何集成測試文件中使用。這裡是一個 *tests/integration_test.rs* 中調用 `setup` 函數的 `it_adds_two` 測試的例子：

<span class="filename">文件名: tests/integration_test.rs</span>

```rust
extern crate adder;

mod common;

#[test]
fn it_adds_two() {
    common::setup();
    assert_eq!(4, adder::add_two(2));
}
```

注意 `mod common;` 聲明與第七章中的模組聲明相同。接著在測試函數中就可以調用 `common::setup()` 了。

#### 二進制 crate 的集成測試

如果項目是二進制 crate 並且只包含 *src/main.rs* 而沒有 *src/lib.rs*，這樣就不可能在 *tests* 創建集成測試並使用 `extern crate` 導入 *src/main.rs* 中的函數了。只有庫 crate 向其他 crate 暴露了可供調用和使用的函數；二進制 crate 只意在單獨運行。

這也是 Rust 二進制項目明確採用 *src/main.rs* 調用 *src/lib.rs* 中邏輯這樣的結構的原因之一。通過這種結構，集成測試 **就可以** 使用 `extern crate` 測試庫 crate 中的主要功能了，而如果這些重要的功能沒有問題的話，*src/main.rs* 中的少量代碼也就會正常工作且不需要測試。

## 總結

Rust 的測試功能提供了一個如何確保即使函數做出改變也能繼續以指定方式運行的途徑。單元測試獨立的驗證庫的不同部分並能夠測試私有實現細節。集成測試則涉及多個部分結合起來工作時的用例，並像其他代外部碼那樣測試庫的公有 API。即使 Rust 的類型系統和所有權規則可以幫助避免一些 bug，不過測試對於減少代碼是否符合期望相關的邏輯 bug 仍然是很重要的。

接下來讓我們結合本章所學和其他之前章節的知識，在下一章一起編寫一個項目！