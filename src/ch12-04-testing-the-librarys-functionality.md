## 測試庫的功能

> [ch12-04-testing-the-librarys-functionality.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch12-04-testing-the-librarys-functionality.md)
> <br>
> commit 5908c59a5a4cc58fd863605b80b295a335c2cbdf

現在我們將邏輯提取到了 *src/lib.rs* 並將所有的參數解析和錯誤處理留在了 *src/main.rs* 中，為代碼的核心功能編寫測試將更加容易。我們可以直接使用多種參數調用函數並檢查返回值而無需從命令行運行二進制文件了。如果你願意的話，請自行為 `Config::new` 和 `run` 函數的功能編寫一些測試。

在這一部分，我們將遵循測試驅動開發（Test Driven Development, TTD）的模式來逐步增加 `minigrep` 的搜索邏輯。這是一個軟件開發技術，它遵循如下步驟：

1. 編寫一個會失敗的測試，並運行它以確保其因為你期望的原因失敗。
2. 編寫或修改剛好足夠的代碼來使得新的測試通過。
3. 重構剛剛增加或修改的代碼，並確保測試仍然能通過。
4. 重複上述步驟！

這只是眾多編寫軟件的方法之一，不過 TDD 有助於驅動代碼的設計。在編寫能使測試通過的代碼之前編寫測試有助於在開發過程中保持高測試覆蓋率。

我們將測試驅動實現實際在文件內容中搜索查詢字符串並返回匹配的行列表的功能。我們將在一個叫做 `search` 的函數中增加這些功能。

### 編寫失敗測試

首先，去掉 *src/lib.rs* 和 *src/main.rs* 中的`println!`語句，因為不再真正需要他們了。接著我們會像第十一章那樣增加一個 `test` 模組和一個測試函數。測試函數指定了 `search` 函數期望擁有的行為：它會抓取一個需要查詢的字符串和用來查詢的文本，並只會返回包含請求的文本行。列表 12-15 展示了這個測試：

<span class="filename">文件名: src/lib.rs</span>

```rust
# fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
#      vec![]
# }
#
#[cfg(test)]
mod test {
    use super::*;

    #[test]
    fn one_result() {
        let query = "duct";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.";

        assert_eq!(
            vec!["safe, fast, productive."],
            search(query, contents)
        );
    }
}
```

<span class="caption">列表 12-15：創建一個我們期望的 `search` 函數的失敗測試</span>

這裡選擇使用 "duct" 作為這個測試中需要搜索的字符串。用來搜索的文本有三行，其中只有一行包含 "duct"。我們斷言 `search` 函數的返回值只包含期望的那一行。

我們還不能運行這個測試並看到它失敗，因為它甚至都還不能編譯！我們將增加足夠的代碼來使其能夠編譯：一個總是會返回空 vector 的 `search` 函數定義，如列表 12-16 所示。一旦有了它，這個測試應該能夠編譯並因為空 vector 並不匹配一個包含一行 `"safe, fast, productive."` 的 vector 而失敗。

<span class="filename">文件名: src/lib.rs</span>

```
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    vec![]
}
```

<span class="caption">列表 12-16：剛好足夠使測試通過編譯的 `search` 函數定義</span>

注意需要在 `search` 的簽名中定義一個顯式生命週期 `'a` 並用於 `contents` 參數和返回值。回憶一下第十章中講到生命週期參數指定哪個參數的生命週期與返回值的生命週期相關聯。在這個例子中，我們表明返回的 vector 中應該包含引用參數 `contents`（而不是參數`query`） slice 的字符串 slice。

換句話說，我們告訴 Rust 函數 `search` 返回的數據將與 `search` 函數中的參數 `contents` 的數據存在的一樣久。這是非常重要的！為了使這個引用有效那麼 **被** slice 引用的數據也需要保持有效；如果編譯器認為我們是在創建 `query` 而不是 `contents` 的字符串 slice，那麼安全檢查將是不正確的。

如果嘗試不用生命週期編譯的話，我們將得到如下錯誤：

```text
error[E0106]: missing lifetime specifier
 --> src/lib.rs:5:47
  |
5 | fn search(query: &str, contents: &str) -> Vec<&str> {
  |                                               ^ expected lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the
  signature does not say whether it is borrowed from `query` or `contents`
```

Rust 不可能知道我們需要的是哪一個參數，所以需要告訴它。因為參數 `contents` 包含了所有的文本而且我們希望返回匹配的那部分文本，所以我們知道 `contents` 是應該要使用生命週期語法來與返回值相關聯的參數。

其他語言中並不需要你在函數簽名中將參數與返回值相關聯，所以這麼做可能仍然感覺有些陌生，隨著時間的推移會越來越容易。你可能想要將這個例子與第十章中生命週期語法部分做對比。

現在試嘗試運行測試：

```text
$ cargo test
...warnings...
    Finished dev [unoptimized + debuginfo] target(s) in 0.43 secs
     Running target/debug/deps/minigrep-abcabcabc

running 1 test
test test::one_result ... FAILED

failures:

---- test::one_result stdout ----
	thread 'test::one_result' panicked at 'assertion failed: `(left == right)`
(left: `["safe, fast, productive."]`, right: `[]`)', src/lib.rs:16
note: Run with `RUST_BACKTRACE=1` for a backtrace.


failures:
    test::one_result

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured

error: test failed
```

好的，測試失敗了，這正是我們所期望的。修改代碼來讓測試通過吧！

### 編寫使測試通過的代碼

目前測試之所以會失敗是因為我們總是返回一個空的 vector。為了修復並實現 `search`，我們的程序需要遵循如下步驟：

* 遍歷每一行文本。
* 查看這一行是否包含要搜索的字符串。
* 如果有，將這一行加入返回列表中。
* 如果沒有，什麼也不做。
* 返回匹配到的列表

讓我們一步一步的來，從遍歷每行開始。

#### 使用 `lines` 方法遍歷每一行

Rust 有一個有助於一行一行遍歷字符串的方法，出於方便它被命名為 `lines`，它如列表 12-17 這樣工作：

<span class="filename">Filename: src/lib.rs</span>

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    for line in contents.lines() {
        // do something with line
    }
}
```

<span class="caption">列表 12-17：遍歷 `contents` 的每一行</span>

`lines` 方法返回一個迭代器。第十三章會深入瞭解迭代器，不過我們已經在列表 3-6 中見過使用迭代器的方法，在那裡使用了一個 `for` 循環和迭代器在一個集合的每一項上運行了一些代碼。

#### 用查詢字符串搜索每一行

接下來將會增加檢查當前行是否包含查詢字符串的功能。幸運的是，字符串類型為此也有一個叫做 `contains` 的實用方法！如列表 12-18 所示在 `search` 函數中加入 `contains` 方法調用：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    for line in contents.lines() {
        if line.contains(query) {
            // do something with line
        }
    }
}
```

<span class="caption">列表 12-18：增加檢查文本行是否包含 `query` 中字符串的功能</span>

#### 存儲匹配的行

最後我們需要一個方法來存儲包含查詢字符串的行。為此可以在 `for` 循環之前創建一個可變的 vector 並調用 `push` 方法在 vector 中存放一個 `line`。在 `for` 循環之後，返回這個 vector，如列表 12-19 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();

    for line in contents.lines() {
        if line.contains(query) {
            results.push(line);
        }
    }

    results
}
```

<span class="caption">列表 12-19：儲存匹配的行以便可以返回他們</span>

現在 `search` 函數應該返回只包含 `query` 的那些行，而測試應該會通過。讓我們運行測試：

```text
$ cargo test
running 1 test
test test::one_result ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

測試通過了，很好，它可以工作了！

現在測試通過了，我們可以考慮一下重構 `search` 的實現並時刻保持測試通過來保持其功能不變的機會了。`search` 函數中的代碼並不壞，不過並沒有利用迭代器的一些實用功能。第十三章將回到這個例子並深入探索迭代器並看看如何改進代碼。

#### 在 `run` 函數中使用 `search` 函數

現在 `search` 函數是可以工作並測試通過了的，我們需要實際在 `run` 函數中調用 `search`。需要將 `config.query` 值和 `run` 從文件中讀取的 `contents` 傳遞給 `search` 函數。接著 `run` 會打印出 `search` 返回的每一行：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub fn run(config: Config) -> Result<(), Box<Error>> {
    let mut f = File::open(config.filename)?;

    let mut contents = String::new();
    f.read_to_string(&mut contents)?;

    for line in search(&config.query, &contents) {
        println!("{}", line);
    }

    Ok(())
}
```

這裡仍然使用了 `for` 循環抓取了 `search` 返回的每一行並打印出來。

現在整個程序應該可以工作了！讓我們試一試，首先使用一個只會在艾米莉‧狄金森的詩中返回一行的單詞 "frog"：

```text
$ cargo run frog poem.txt
   Compiling minigrep v0.1.0 (file:///projects/minigrep)
    Finished dev [unoptimized + debuginfo] target(s) in 0.38 secs
     Running `target/debug/minigrep frog poem.txt`
How public, like a frog
```

好的！接下來，像 "the" 這樣會匹配多行的單詞會怎麼樣呢：

```text
$ cargo run the poem.txt
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/minigrep the poem.txt`
Then there's a pair of us — don't tell!
To tell your name the livelong day
```

最後，讓我們確保搜索一個在詩中哪裡都沒有的單詞時不會得到任何行，比如 "monomorphization"：

```text
$ cargo run monomorphization poem.txt
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/minigrep monomorphization poem.txt`
```

非常好！我們創建了一個屬於自己的迷你版經典工具，並學習了很多如何組織程序的知識。我們還學習了一些文件輸入輸出、生命週期、測試和命令行解析的內容。

為了使這個項目章節更豐滿，我們將簡要的展示如何處理環境變數和打印到標準錯誤，這兩者在編寫命令行程序時都很有用。現在如果你希望的話請隨意移動到第十三章。