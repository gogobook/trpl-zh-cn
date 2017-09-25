## 運行測試

> [ch11-02-running-tests.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch11-02-running-tests.md)
> <br>
> commit db08b34db5f1c78b4866b391c802344ec94ecc38

就像 `cargo run` 會編譯代碼並運行生成的二進制文件一樣，`cargo test` 在測試模式下編譯代碼並運行生成的測試二進制文件。這裡有一些選項可以用來改變 `cargo test` 的預設行為。例如，`cargo test` 生成的二進制文件的預設行為是並行的運行所有測試，並捕獲測試運行過程中產生的輸出避免他們被顯示出來，使得閱讀測試結果相關的內容變得更容易。你可以指定命令行參數來改變這些預設行為。

這些選項的一部分可以傳遞給 `cargo test`，而另一些則需要傳遞給生成的測試二進制文件。為了分隔兩種類型的參數，首先列出傳遞給 `cargo test` 的參數，接著是分隔符 `--`，再之後是傳遞給測試二進制文件的參數。運行 `cargo test --help` 會告訴你 `cargo test` 的相關參數，而運行 `cargo test -- --help` 則會告訴你位於分隔符 `--` 之後的相關參數。

### 並行或連續的運行測試

當運行多個測試時，他們預設使用線程來並行的運行。這意味著測試會更快的運行完畢，所以可以更快的得到代碼能否工作的反饋。因為測試是在同時運行的，你應該小心測試不能相互依賴或依賴任何共享狀態，包括類似於當前工作目錄或者環境變數這樣的共享環境。

例如，每一個測試都運行一些代碼在硬盤上創建一個 `test-output.txt` 文件並寫入一些數據。接著每一個測試都讀取文件中的數據並斷言這個文件包含特定的值，而這個值在每個測試中都是不同的。因為所有測試都是同時運行的，一個測試可能會在另一個測試讀寫文件過程中覆蓋了文件。那麼第二個測試就會失敗，並不是因為代碼不正確，而是因為測試並行運行時相互干涉。一個解決方案是使每一個測試讀寫不同的文件；另一個是一次運行一個測試。

如果你不希望測試並行運行，或者想要更加精確的控制使用線程的數量，可以傳遞 `--test-threads` 參數和希望使用線程的數量給測試二進制文件。例如：

```text
$ cargo test -- --test-threads=1
```

這裡將測試線程設置為 1，告訴程序不要使用任何並行機制。這也會比並行運行花費更多時間，不過測試就不會在存在共享狀態時潛在的相互干涉了。

### 顯示函數輸出

如果測試通過了，Rust 的測試庫預設會捕獲打印到標準輸出的任何內容。例如，如果在測試中調用 `println!` 而測試通過了，我們將不會在終端看到 `println!` 的輸出：只會看到說明測試通過的行。如果測試失敗了，就會看到所有標準輸出和其他錯誤信息。

例如，列表 11-20 有一個無意義的函數它打印出其參數的值並接著返回 10。接著還有一個會通過的測試和一個會失敗的測試：

<span class="filename">文件名: src/lib.rs</span>

```rust
fn prints_and_returns_10(a: i32) -> i32 {
    println!("I got the value {}", a);
    10
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn this_test_will_pass() {
        let value = prints_and_returns_10(4);
        assert_eq!(10, value);
    }

    #[test]
    fn this_test_will_fail() {
        let value = prints_and_returns_10(8);
        assert_eq!(5, value);
    }
}
```

<span class="caption">列表 11-10：一個調用了 `println!` 的函數的測試</span>

運行 `cargo test` 將會看到這些測試的輸出：

```text
running 2 tests
test tests::this_test_will_pass ... ok
test tests::this_test_will_fail ... FAILED

failures:

---- tests::this_test_will_fail stdout ----
	I got the value 8
thread 'tests::this_test_will_fail' panicked at 'assertion failed: `(left ==
right)` (left: `5`, right: `10`)', src/lib.rs:19
note: Run with `RUST_BACKTRACE=1` for a backtrace.

failures:
    tests::this_test_will_fail

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured
```

注意輸出中哪裡也不會出現 `I got the value 4`，這是當測試通過時打印的內容。這些輸出被捕獲。失敗測試的輸出，`I got the value 8`，則出現在輸出的測試總結部分，同時也顯示了測試失敗的原因。

如果你希望也能看到通過的測試中打印的值，捕獲輸出的行為可以通過 `--nocapture` 參數來禁用：

```text
$ cargo test -- --nocapture
```

使用 `--nocapture` 參數再次運行列表 11-10 中的測試會顯示：

```text
running 2 tests
I got the value 4
I got the value 8
test tests::this_test_will_pass ... ok
thread 'tests::this_test_will_fail' panicked at 'assertion failed: `(left ==
right)` (left: `5`, right: `10`)', src/lib.rs:19
note: Run with `RUST_BACKTRACE=1` for a backtrace.
test tests::this_test_will_fail ... FAILED

failures:

failures:
    tests::this_test_will_fail

test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured
```

注意測試的輸出和測試結果的輸出是相互交叉的；這是由於上一部分講到的測試是並行運行的。嘗試一同使用`--test-threads=1`和`--nocapture`功能來看看輸出是什麼樣子！

### 通過名稱來運行測試的子集

有時運行整個測試集會耗費很長時間。如果你負責特定位置的代碼，你可能會希望只運行這些代碼相關的測試。可以向 `cargo test` 傳遞希望運行的測試的（部分）名稱作為參數來選擇運行哪些測試。

為了展示如何運行測試的子集，列表 11-11 使用 `add_two` 函數創建了三個測試來供我們選擇運行哪一個：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_two_and_two() {
        assert_eq!(4, add_two(2));
    }

    #[test]
    fn add_three_and_two() {
        assert_eq!(5, add_two(3));
    }

    #[test]
    fn one_hundred() {
        assert_eq!(102, add_two(100));
    }
}
```

<span class="caption">列表 11-11：不同名稱的三個測試</span>

如果沒有傳遞任何參數就運行測試，如你所見，所有測試都會並行運行：

```text
running 3 tests
test tests::add_two_and_two ... ok
test tests::add_three_and_two ... ok
test tests::one_hundred ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured
```

#### 運行單個測試

可以向 `cargo test` 傳遞任意測試的名稱來只運行這個測試：

```text
$ cargo test one_hundred
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running target/debug/deps/adder-06a75b4a1f2515e9

running 1 test
test tests::one_hundred ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

不能像這樣指定多個測試名稱，只有傳遞給 `cargo test` 的第一個值才會被使用。

#### 過濾運行多個測試

然而，可以指定測試的部分名稱，這樣任何名稱匹配這個值的測試會被運行。例如，因為頭兩個測試的名稱包含 `add`，可以通過 `cargo test add` 來運行這兩個測試：

```text
$ cargo test add
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running target/debug/deps/adder-06a75b4a1f2515e9

running 2 tests
test tests::add_two_and_two ... ok
test tests::add_three_and_two ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured
```

這運行了所有名字中帶有 `add` 的測試。同時注意測試所在的模塊作為測試名稱的一部分，所以可以通過模塊名來過濾運行一個模塊中的所有測試。

### 除非指定否則忽略某些測試

有時一些特定的測試執行起來是非常耗費時間的，所以在大多數運行 `cargo test` 的時候希望能排除他們。與其通過參數列舉出所有希望運行的測試，也可以使用 `ignore` 屬性來標記耗時的測試來排除他們：

<span class="filename">文件名: src/lib.rs</span>

```rust
#[test]
fn it_works() {
    assert!(true);
}

#[test]
#[ignore]
fn expensive_test() {
    // code that takes an hour to run
}
```

對想要排除的測試的 `#[test]` 之後增加了 `#[ignore]` 行。現在如果運行測試，就會發現 `it_works` 運行了，而 `expensive_test` 沒有運行：

```text
$ cargo test
   Compiling adder v0.1.0 (file:///projects/adder)
    Finished dev [unoptimized + debuginfo] target(s) in 0.24 secs
     Running target/debug/deps/adder-ce99bcc2479f4607

running 2 tests
test expensive_test ... ignored
test it_works ... ok

test result: ok. 1 passed; 0 failed; 1 ignored; 0 measured

   Doc-tests adder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

`expensive_test` 被列為 `ignored`，如果只希望運行被忽略的測試，可以使用 `cargo test -- --ignored` 來請求運行他們：

```text
$ cargo test -- --ignored
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running target/debug/deps/adder-ce99bcc2479f4607

running 1 test
test expensive_test ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

通過控制運行哪些測試，可以確保運行 `cargo test` 的結果是快速的。當某個時刻需要檢查 `ignored` 測試的結果而且你也有時間等待這個結果的話，可以選擇執行 `cargo test -- --ignored`。