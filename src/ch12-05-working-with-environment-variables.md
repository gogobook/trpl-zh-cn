## 處理環境變數

> [ch12-05-working-with-environment-variables.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch12-05-working-with-environment-variables.md)
> <br>
> commit adababc48956f4d39c97c8b6fc14a104d90e20dc

我們將用一個額外的功能來改進我們的工具：一個通過環境變數啟用的大小寫不敏感搜索的選項。我們可以將其設計為一個命令行參數並要求用戶每次需要時都加上它，不過相反我們將使用環境變數。這允許用戶設置環境變數一次之後在整個終端會話中所有的搜索都將是大小寫不敏感的。

### 編寫一個大小寫不敏感 `search` 函數的失敗測試

首先，增加一個新函數，當設置了環境變數時會調用它。

這裡將繼續遵循上一部分開始使用的 TDD 過程，其第一步是再次編寫一個失敗測試。我們將為新的大小寫不敏感搜索函數新增一個測試函數，並將老的測試函數從 `one_result` 改名為 `case_sensitive` 來更清楚的表明這兩個測試的區別，如列表 12-20 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
#[cfg(test)]
mod test {
    use super::*;

    #[test]
    fn case_sensitive() {
        let query = "duct";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.
Duct tape.";

        assert_eq!(
            vec!["safe, fast, productive."],
            search(query, contents)
        );
    }

    #[test]
    fn case_insensitive() {
        let query = "rUsT";
        let contents = "\
Rust:
safe, fast, productive.
Pick three.
Trust me.";

        assert_eq!(
            vec!["Rust:", "Trust me."],
            search_case_insensitive(query, contents)
        );
    }
}
```

<span class="caption">列表 12-20：為準備添加的大小寫不敏感函數新增失敗測試</span>

注意我們也改變了老測試中 `contents` 的值。還新增了一個含有文本 "Duct tape" 的行，它有一個大寫的 D，這在大小寫敏感搜索時不應該匹配 "duct"。我們修改這個測試以確保不會意外破壞已經實現的大小寫敏感搜索功能；這個測試現在應該能通過並在處理大小寫不敏感搜索時應該能一直通過。

大小寫 **不敏感** 搜索的新測試使用 "rUsT" 作為其查詢字符串。在我們將要增加的 `search_case_insensitive` 函數中，「rUsT」 查詢應該包含 「Rust:」 包含一個大寫的 R 還有 「Trust me.」 這兩行，即便他們與查詢的大小寫都不同。這個測試現在會編譯失敗因為還沒有定義 `search_case_insensitive` 函數。請隨意增加一個總是返回空 vector 的骨架實現，正如列表 12-16 中 `search` 函數為了使測試編譯並失敗時所做的那樣。

### 實現 `search_case_insensitive` 函數

`search_case_insensitive` 函數，如列表 12-21 所示，將與 `search` 函數基本相同。唯一的區別是它會將 `query` 變數和每一 `line` 都變為小寫，這樣不管輸入參數是大寫還是小寫，在檢查該行是否包含查詢字符串時都會是小寫。

<span class="filename">文件名: src/lib.rs</span>

```rust
fn search_case_insensitive<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let query = query.to_lowercase();
    let mut results = Vec::new();

    for line in contents.lines() {
        if line.to_lowercase().contains(&query) {
            results.push(line);
        }
    }

    results
}
```

<span class="caption">列表 12-21：定義 `search_case_insensitive` 函數，它在比較查詢和每一行之前將他們都轉換為小寫</span>

首先我們將 `query` 字符串轉換為小寫，並將其覆蓋到同名的變數中。對查詢字符串調用 `to_lowercase` 是必需的，這樣不管用戶的查詢是 「rust」、「RUST」、「Rust」 或者 「rUsT」，我們都將其當作 「rust」 處理並對大小寫不敏感。

注意 `query` 現在是一個 `String` 而不是字符串 slice，因為調用 `to_lowercase` 是在創建新數據，而不是引用現有數據。如果查詢字符串是 「rUsT」，這個字符串 slice 並不包含可供我們使用的小寫的 「u」 或 「t」，所以必需分配一個包含 「rust」 的新 `String`。現在當我們將 `query` 作為一個參數傳遞給 `contains` 方法時，需要增加一個 & 因為 `contains` 的簽名被定義為抓取一個字符串 slice。

接下來在檢查每個 `line` 是否包含 `search` 之前增加了一個 `to_lowercase` 調用將他們都變為小寫。現在我們將 `line` 和 `query` 都轉換成了小寫，這樣就可以不管查詢的大小寫進行匹配了。

讓我們看看這個實現能否通過測試：

```text
running 2 tests
test test::case_insensitive ... ok
test test::case_sensitive ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured
```

好的！現在，讓我們在 `run` 函數中實際調用新 `search_case_insensitive` 函數。首先，我們將在 `Config` 結構體中增加一個配置項來切換大小寫敏感和大小寫不敏感搜索：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub struct Config {
    pub query: String,
    pub filename: String,
    pub case_sensitive: bool,
}
```

這裡增加了 `case_sensitive` 字符來存放一個布爾值。接著我們需要 `run` 函數檢查 `case_sensitive` 字段的值並使用它來決定是否調用 `search` 函數或 `search_case_insensitive` 函數，如列表 12-22 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
# use std::error::Error;
# use std::fs::File;
# use std::io::prelude::*;
#
# fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
#      vec![]
# }
#
# fn search_case_insensitive<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
#      vec![]
# }
#
# struct Config {
#     query: String,
#     filename: String,
#     case_sensitive: bool,
# }
#
pub fn run(config: Config) -> Result<(), Box<Error>>{
    let mut f = File::open(config.filename)?;

    let mut contents = String::new();
    f.read_to_string(&mut contents)?;

    let results = if config.case_sensitive {
        search(&config.query, &contents)
    } else {
        search_case_insensitive(&config.query, &contents)
    };

    for line in results {
        println!("{}", line);
    }

    Ok(())
}
```

<span class="caption">列表 12-22：根據 `config.case_sensitive` 的值調用 `search` 或 `search_case_insensitive`</span>

最後需要實際檢查環境變數。處理環境變數的函數位於標準庫的 `env` 模塊中，所以我們需要在 *src/lib.rs* 的開頭增加一個 `use std::env;` 行將這個模塊引入作用域中。接著在 `Config::new` 中使用 `env` 模塊的 `var` 方法來檢查一個叫做 `CASE_INSENSITIVE` 的環境變數，如列表 12-23 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
use std::env;
# struct Config {
#     query: String,
#     filename: String,
#     case_sensitive: bool,
# }

// ...snip...

impl Config {
    pub fn new(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }

        let query = args[1].clone();
        let filename = args[2].clone();

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();

        Ok(Config { query, filename, case_sensitive })
    }
}
```

<span class="caption">列表 12-23：檢查叫做 `CASE_INSENSITIVE` 的環境變數</span>

這裡創建了一個新變數 `case_sensitive`。為了設置它的值，需要調用 `env::var` 函數並傳遞我們需要尋找的環境變數名稱，`CASE_INSENSITIVE`。`env::var` 返回一個 `Result`，它在環境變數被設置時返回包含其值的 `Ok` 成員，並在環境變數未被設置時返回 `Err` 成員。

我們使用 `Result` 的 `is_err` 方法來檢查其是否是一個 error（也就是環境變數未被設置的情況），這也就意味著我們 **需要** 進行一個大小寫敏感搜索。如果`CASE_INSENSITIVE` 環境變數被設置為任何值，`is_err` 會返回 false 並將進行大小寫不敏感搜索。我們並不關心環境變數所設置的 **值**，只關心它是否被設置了，所以檢查 `is_err` 而不是 `unwrap`、`expect` 或任何我們已經見過的 `Result` 的方法。

我們將變數 `case_sensitive` 的值傳遞給 `Config` 實例，這樣 `run` 函數可以讀取其值並決定是否調用 `search` 或者列表 12-22 中實現的 `search_case_insensitive`。

讓我們試一試吧！首先不設置環境變數並使用查詢 「to」 運行程序，這應該會匹配任何全小寫的單詞 「to」 的行：

```text
$ cargo run to poem.txt
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/minigrep to poem.txt`
Are you nobody, too?
How dreary to be somebody!
```

看起來程序仍然能夠工作！現在將 `CASE_INSENSITIVE` 設置為 1 並仍使用相同的查詢 「to」，這回應該得到包含可能有大寫字母的 「to」 的行：

```text
$ CASE_INSENSITIVE=1 cargo run to poem.txt
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/minigrep to poem.txt`
Are you nobody, too?
How dreary to be somebody!
To tell your name the livelong day
To an admiring bog!
```

好極了，我們也得到了包含 「To」 的行！現在 `minigrep` 程序可以通過環境變數控制進行大小寫不敏感搜索了。現在你知道了如何管理由命令行參數或環境變數設置的選項了！

一些程序允許對相同配置同時使用參數 **和** 環境變數。在這種情況下，程序來決定參數和環境變數的優先級。作為一個留給你的測試，嘗試通過一個命令行參數或一個環境變數來控制大小寫不敏感搜索。並在運行程序時遇到矛盾值時決定命令行參數和環境變數的優先級。

`std::env` 模塊還包含了更多處理環境變數的實用功能；請查看官方文檔來瞭解其可用的功能。