## 控制流

> [ch03-05-control-flow.md](https://github.com/rust-lang/book/blob/master/src/ch03-05-control-flow.md)
> <br>
> commit 2e269ff82193fd65df8a87c06561d74b51ac02f7

通過條件是不是為真來決定是否執行某些代碼，或者根據條件是否為真來重複運行一段代碼是大部分編程語言的基本組成部分。Rust 代碼中最常見的用來控制執行流的結構是 `if` 表達式和循環。

### `if` 表達式

`if` 表達式允許根據條件執行不同的代碼分支。我們提供一個條件並表示 「如果符合這個條件，運行這段代碼。如果條件不滿足，不運行這段代碼。」

在 *projects* 目錄創建一個叫做 *branches* 的新項目來學習 `if` 表達式。在 *src/main.rs* 文件中，輸入如下內容：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let number = 3;

    if number < 5 {
        println!("condition was true");
    } else {
        println!("condition was false");
    }
}
```

<!-- NEXT PARAGRAPH WRAPPED WEIRD INTENTIONALLY SEE #199 -->

所有 `if` 表達式以 `if` 關鍵字開頭，它後跟一個條件。在這個例子中，條件檢查 `number` 是否有一個小於 5 的值。在條件為真時希望執行的代碼塊位於緊跟條件之後的大括號中。`if` 表達式中與條件關聯的代碼塊有時被叫做 *arms*，就像第二章 「比較猜測與秘密數字」 部分中討論到的 `match` 表達式中分支一樣。也可以包含一個可選的 `else` 表達式，這裡我們就這麼做了，來提供一個在條件為假時應當執行的代碼塊。如果不提供 `else` 表達式並且條件為假時，程序會直接忽略 `if` 代碼塊並繼續執行下面的代碼。

嘗試運行代碼，應該能看到如下輸出：

```text
$ cargo run
   Compiling branches v0.1.0 (file:///projects/branches)
     Running `target/debug/branches`
condition was true
```

嘗試改變 `number` 的值使條件為假時看看會發生什麼：

```rust
let number = 7;
```

再次運行程序並查看輸出：

```text
$ cargo run
   Compiling branches v0.1.0 (file:///projects/branches)
     Running `target/debug/branches`
condition was false
```

另外值得注意的是代碼中的條件 **必須** 是 `bool`。如果像看看條件不是 `bool` 值時會發生什麼，嘗試運行如下代碼：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let number = 3;

    if number {
        println!("number was three");
    }
}
```

這裡 `if` 條件的值是 `3`，Rust 拋出了一個錯誤：

```text
error[E0308]: mismatched types
 --> src/main.rs:4:8
  |
4 |     if number {
  |        ^^^^^^ expected bool, found integral variable
  |
  = note: expected type `bool`
             found type `{integer}`
```

這個錯誤表明 Rust 期望一個 `bool` 不過卻得到了一個整型。Rust 並不會嘗試自動地將非布爾值轉換為布爾值，不像例如 Ruby 和 JavaScript 這樣的語言。必須總是顯式地使用 `boolean` 作為 `if` 的條件。例如如果想 要`if` 代碼塊只在一個數字不等於 `0` 時執行，可以把 `if` 表達式修改為如下：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let number = 3;

    if number != 0 {
        println!("number was something other than zero");
    }
}
```

運行代碼會打印出 `number was something other than zero`。

#### 使用 `else if` 實現多重條件

可以將 `else if` 表達式與 `if` 和 `else` 組合來實現多重條件。例如：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let number = 6;

    if number % 4 == 0 {
        println!("number is divisible by 4");
    } else if number % 3 == 0 {
        println!("number is divisible by 3");
    } else if number % 2 == 0 {
        println!("number is divisible by 2");
    } else {
        println!("number is not divisible by 4, 3, or 2");
    }
}
```

這個程序有四個可能的執行路徑。運行後應該能看到如下輸出：

```text
$ cargo run
   Compiling branches v0.1.0 (file:///projects/branches)
     Running `target/debug/branches`
number is divisible by 3
```

當執行這個程序，它按順序檢查每個 `if` 表達式並執行第一個條件為真的代碼塊。注意即使 6 可以被 2 整除，也不會出現 `number is divisible by 2` 的輸出，更不會出現 `else` 塊中的 `number is not divisible by 4, 3, or 2`。原因是 Rust 只會執行第一個條件為真的代碼塊，並且它一旦找到一個以後，就不會檢查剩下的條件了。

使用過多的 `else if` 表達式會使代碼顯得雜亂無章，所以如果有多於一個 `else if`，最好重構代碼。為此第六章會介紹 Rust 中一個叫做 `match` 的強大的分支結構（branching construct）。

#### 在 `let` 語句中使用 `if`

因為 `if` 是一個表達式，我們可以在 `let` 語句的右側使用它，例如在列表 3-4 中：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let condition = true;
    let number = if condition {
        5
    } else {
        6
    };

    println!("The value of number is: {}", number);
}
```

<span class="caption">列表 3-4：將 `if` 的返回值賦值給一個變量</span>

`number` 變量將會綁定到基於 `if` 表達式結果的值。運行這段代碼看看會出現什麼：

```text
$ cargo run
   Compiling branches v0.1.0 (file:///projects/branches)
     Running `target/debug/branches`
The value of number is: 5
```

還記得代碼塊的值是其最後一個表達式的值，以及數字本身也是一個表達式嗎。在這個例子中，整個 `if` 表達式的值依賴哪個代碼塊被執行。這意味著 `if` 的每個分支的可能的返回值都必須是相同類型；在列表 3-4 中，`if` 分支和 `else` 分支的結果都是 `i32` 整型。不過如果像下面的例子那樣這些類型並不匹配會怎麼樣呢？

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let condition = true;

    let number = if condition {
        5
    } else {
        "six"
    };

    println!("The value of number is: {}", number);
}
```

當運行這段代碼，會得到一個錯誤。`if` 和 `else` 分支的值類型是不相容的，同時 Rust 也準確地表明了在程序中的何處發現的這個問題：

```text
error[E0308]: if and else have incompatible types
 --> src/main.rs:4:18
  |
4 |       let number = if condition {
  |  __________________^
5 | |         5
6 | |     } else {
7 | |         "six"
8 | |     };
  | |_____^ expected integral variable, found reference
  |
  = note: expected type `{integer}`
             found type `&'static str`
```

`if` 代碼塊的表達式返回一個整型，而 `else` 代碼塊返回一個字符串。這並不可行，因為變量必須只有一個類型。Rust 需要在編譯時就確切的知道 `number` 變量的類型，這樣它就可以在編譯時證明其他使用 `number` 變量的地方它的類型是有效的。Rust 並不能夠在 `number` 的類型只能在運行時確定的情況下工作；這樣會使編譯器變得更複雜而且只能為代碼提供更少的保障，因為它不得不記錄所有變量的多種可能的類型。

### 使用循環重複執行

多次執行同一段代碼是很常用的。為了這個功能，Rust 提供了多種 **循環**（*loops*）。一個循環執行循環體中的代碼直到結尾並緊接著從回到開頭繼續執行。為了實驗一下循環，讓我們創建一個叫做 *loops* 的新項目。

Rust 有三種循環類型：`loop`、`while` 和 `for`。讓我們每一個都試試。

#### 使用 `loop` 重複執行代碼

`loop` 關鍵字告訴 Rust 一遍又一遍的執行一段代碼直到你明確要求停止。

作為一個例子，將 *loops* 目錄中的 *src/main.rs* 文件修改為如下：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    loop {
        println!("again!");
    }
}
```

當執行這個程序，我們會看到 `again!` 被連續的打印直到我們手動停止程序。大部分終端都支持一個鍵盤快捷鍵，<span class="keystroke">ctrl-C</span>，來終止一個陷入無限循環的程序。嘗試一下：

```text
$ cargo run
   Compiling loops v0.1.0 (file:///projects/loops)
     Running `target/debug/loops`
again!
again!
again!
again!
^Cagain!
```

符號 `^C` 代表你在這按下了<span class="keystroke">ctrl-C</span>。在 `^C` 之後你可能看到 `again!` 也可能看不到，這依賴於在接收到終止信號時代碼執行到了循環的何處。

幸運的是，Rust 提供了另一個更可靠的方式來退出循環。可以使用 `break` 關鍵字來告訴程序何時停止執行循環。還記得我們在第二章猜猜看遊戲的 「猜測正確後退出」 部分使用過它來在用戶猜對數字贏得遊戲後退出程序嗎。

#### `while` 條件循環

在程序中計算循環的條件也很常見。當條件為真，執行循環。當條件不再為真，調用 `break`停止循環。這個循環類型可以通過組合 `loop`、`if`、`else` 和 `break`來實現；如果你喜歡的話，現在就可以在程序中試試。

然而，這個模式太常見了以至於 Rust 為此提供了一個內建的語言結構，它被稱為 `while` 循環。下面的例子使用了 `while`：程序循環三次，每次數字都減一。接著，在循環之後，打印出另一個信息並退出：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let mut number = 3;

    while number != 0 {
        println!("{}!", number);

        number = number - 1;
    }

    println!("LIFTOFF!!!");
}
```

這個結構消除了很多需要嵌套使用 `loop`、`if`、`else` 和 `break` 的代碼，這樣顯得更加清楚。當條件為真就執行，否則退出循環。

#### 使用 `for` 遍歷集合

可以使用 `while` 結構來遍歷一個元素集合，比如數組。如下：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let a = [10, 20, 30, 40, 50];
    let mut index = 0;

    while index < 5 {
        println!("the value is: {}", a[index]);

        index = index + 1;
    }
}
```

<span class="caption">列表 3-5：使用 `while` 循環遍歷集合中的每一個元素</span>

這裡代碼對數組中的元素進行計數。它從索引 `0` 開始，並接著循環直到遇到數組的最後一個索引（這時，`index < 5` 不再為真）。運行這段代碼會打印出數組中的每一個元素：

```text
$ cargo run
   Compiling loops v0.1.0 (file:///projects/loops)
     Running `target/debug/loops`
the value is: 10
the value is: 20
the value is: 30
the value is: 40
the value is: 50
```

所有數組中的五個元素都如期被打印出來。儘管 `index` 在某一時刻會到達值 `5`，不過循環在其嘗試從數組獲取第六個值（會越界）之前就停止了。

不過這個過程是容易出錯的；如果索引長度不正確會導致程序 panic。這也使程序更慢，因為編譯器增加了運行時代碼來對每次循環的每個元素進行條件檢查。

可以使用 `for` 循環來對一個集合的每個元素執行一些代碼，來作為一個更有效率的替代。`for` 循環看起來像這樣：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let a = [10, 20, 30, 40, 50];

    for element in a.iter() {
        println!("the value is: {}", element);
    }
}
```

<span class="caption">列表 3-6：使用 `for` 循環遍歷集合中的每一個元素</span>

當運行這段代碼，將看到與列表 3-5 一樣的輸出。更為重要的是，我們增強了代碼安全性並消除了出現可能會導致超出數組的結尾或遍歷長度不夠而缺少一些元素這類 bug 的機會。

例如，在列表 3-5 的代碼中，如果從數組 `a` 中移除一個元素但忘記更新條件為 `while index < 4`，代碼將會 panic。使用`for`循環的話，就不需要惦記著在更新數組元素數量時修改其他的代碼了。

`for` 循環的安全性和簡潔性使得它在成為 Rust 中使用最多的循環結構。即使是在想要循環執行代碼特定次數時，例如列表 3-5 中使用 `while` 循環的倒計時例子，大部分 Rustacean 也會使用 `for` 循環。這麼做的方式是使用 `Range`，它是標準庫提供的用來生成從一個數字開始到另一個數字結束的所有數字序列的類型。

下面是一個使用 `for` 循環來倒計時的例子，它還使用了一個我們還未講到的方法，`rev`，用來反轉 range：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    for number in (1..4).rev() {
        println!("{}!", number);
    }
    println!("LIFTOFF!!!");
}
```

這段代碼看起來更帥氣不是嗎？

## 總結

你做到了！這是一個相當可觀的章節：你學習了變量，標量和 `if` 表達式，還有循環！如果你想要實踐本章討論的概念，嘗試構建如下的程序：

* 相互轉換攝氏與華氏溫度
* 生成 n 階斐波那契數列
* 打印聖誕頌歌 「The Twelve Days of Christmas」 的歌詞，並利用歌曲中的重複部分（編寫循環）

當你準備好繼續的時候，讓我們討論一個其他語言中 *並不* 常見的概念：所有權（ownership）。