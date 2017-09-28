## 所有可能會用到模式的位置

> [ch18-01-all-the-places-for-patterns.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch18-01-all-the-places-for-patterns.md)
> <br>
> commit 4ca9e513e532a4d229ab5af7dfcc567129623bf4

模式出現在 Rust 的很多地方。你已經在不經意間使用了很多模式！本部分是一個所有有效模式位置的參考。

### `match` 分支

如第六章所討論的，一個模式常用的位置是 `match` 表達式的分支。在形式上 `match` 表達式由 `match` 關鍵字、用於匹配的值和一個或多個分支構成。這些分支包含一個模式和在值匹配分支的模式時運行的表達式：

```
match VALUE {
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
    PATTERN => EXPRESSION,
}
```

#### 窮盡性和預設模式 `_`

`match` 表達式必須是窮盡的。當我們把所有分支的模式都放在一起，`match` 表達式所有可能的值都應該被考慮到。一個確保覆蓋每個可能值的方法是在最後一個分支使用捕獲所有的模式，比如一個變數名。一個匹配任何值的名稱永遠也不會失敗，因此可以覆蓋之前分支模式匹配剩下的情況。

這有一個額外的模式經常被用於結尾的分支：`_`。它匹配所有情況，不過它從不綁定任何變數。這在例如只希望在某些模式下運行代碼而忽略其他值的時候很有用。

### `if let` 表達式

第六章討論過了 `if let` 表達式，以及它是如何成為編寫等同於只關心一個情況的 `match` 語句的簡寫的。`if let` 可以對應一個可選的 `else` 和代碼在 `if let` 中的模式不匹配時運行。

示例 18-1 展示了甚至可以組合併匹配 `if let`、`else if` 和 `else if let`。這些代碼展示了一系列針對不同條件的檢查來決定背景顏色應該是什麼。為了達到這個例子的目的，我們創建了硬編碼值的變數，在真實程序中則可能由詢問用戶獲得。如果用戶指定了中意的顏色，我們將使用它作為背景顏色。如果今天是星期二，背景顏色將是綠色。如果用戶指定了他們的年齡字符串並能夠成功將其解析為數字的話，我們將根據這個數字使用紫色或者橙色。最後，如果沒有一個條件符合，背景顏色將是藍色：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let favorite_color: Option<&str> = None;
    let is_tuesday = false;
    let age: Result<u8, _> = "34".parse();

    if let Some(color) = favorite_color {
        println!("Using your favorite color, {}, as the background", color);
    } else if is_tuesday {
        println!("Tuesday is green day!");
    } else if let Ok(age) = age {
        if age > 30 {
            println!("Using purple as the background color");
        } else {
            println!("Using orange as the background color");
        }
    } else {
        println!("Using blue as the background color");
    }
}
```

<span class="caption">示例 18-1: 結合 `if let`、`else if`、`else if let` 和 `else`</span>

這個條件結構允許我們支持複雜的需求。使用這裡硬編碼的值，例子會打印出 `Using purple as the background color`。

注意 `if let` 也可以像 `match` 分支那樣引入覆蓋變數：`if let Ok(age) = age` 引入了一個新的覆蓋變數 `age`，它包含 `Ok` 成員中的值。這也意味著 `if age > 30` 條件需要位於這個代碼塊內部；不能將兩個條件組合為 `if let Ok(age) = age && age > 30`，因為我們希望與 30 進行比較的被覆蓋的 `age` 直到大括號開始的新作用域才是有效的。

另外注意這樣有很多情況的條件並沒有 `match` 表達式強大，因為其窮盡性沒有為編譯器所檢查。如果去掉最後的 `else` 塊而遺漏處理一些情況，編譯器也不會報錯。這個例子可能過於複雜以致難以重寫為一個可讀的 `match`，所以需要額外注意處理了所有的情況，因為編譯器不會為我們檢查窮盡性。

### `while let`

一個與 `if let` 類似的結構體是 `while let`：它允許只要模式匹配就一直進行 `while` 循環。示例 18-2 展示了一個使用 `while let` 的例子，它使用 vector 作為棧並打以先進後出的方式打印出 vector 中的值：

```rust
let mut stack = Vec::new();

stack.push(1);
stack.push(2);
stack.push(3);

while let Some(top) = stack.pop() {
    println!("{}", top);
}
```

<span class="caption">示例 18-2: 使用 `while let` 循環只要 `stack.pop()` 返回 `Some`就打印出其值</span>

這個例子會打印出 3、2 和 1。`pop` 方法取出 vector 的最後一個元素並返回`Some(value)`，如果 vector 是空的，它返回 `None`。`while` 循環只要 `pop` 返回 `Some` 就會一直運行其塊中的代碼。一旦其返回 `None`，`while`循環停止。我們可以使用 `while let` 來彈出棧中的每一個元素。

### `for` 循環

`for` 循環，如同第三章所講的，是 Rust 中最常見的循環結構。那一章所沒有講到的是 `for` 可以抓取一個模式。示例 18-3 中展示了如何使用 `for` 循環來解構一個元組。`enumerate` 方法適配一個迭代器來產生元組，其包含值和值的索引：

```rust
let v = vec![1, 2, 3];

for (index, value) in v.iter().enumerate() {
    println!("{} is at index {}", value, index);
}
```

<span class="caption">示例 18-3: 在 `for` 循環中使用模式來解構 `enumerate` 返回的元組</span>

這會打印出：

```
1 is at index 0
2 is at index 1
3 is at index 2
```

第一個 `enumerate` 調用會產生元組 `(0, 1)`。當這個匹配模式 `(index, value)`，`index` 將會是 0 而 `value` 將會是 1。

### `let` 語句

`match` 和 `if let` 都是本書之前明確討論過的使用模式的位置，不過他們不是僅有的**使用過**模式的地方。例如，考慮一下這個直白的 `let` 變數賦值：

```rust
let x = 5;
```

本書進行了不下百次這樣的操作。你可能沒有發覺，不過你這正是在使用模式！`let` 語句更為正式的樣子如下：

```
let PATTERN = EXPRESSION;
```

我們見過的像 `let x = 5;` 這樣的語句中變數名位於 `PATTERN` 位置；變數名不過是形式特別樸素的模式。

通過 `let`，我們將表達式與模式比較，並為任何找到的名稱賦值。所以例如 `let x = 5;` 的情況，`x` 是一個模式代表「將匹配到的值綁定到變數 x」。同時因為名稱 `x` 是整個模式，這個模式實際上等於「將任何值綁定到變數 `x`，不過它是什麼」。

為了更清楚的理解 `let` 的模式匹配的方面，考慮示例 18-4 中使用 `let` 和模式解構一個元組：

```rust
let (x, y, z) = (1, 2, 3);
```

<span class="caption">示例 18-4: 使用模式解構元組並一次創建三個變數</span>

這裡有一個元組與模式匹配。Rust 會比較值 `(1, 2, 3)` 與模式 `(x, y, z)` 並發現值匹配這個模式。在這個例子中，將會把 `1` 綁定到 `x`，`2` 綁定到 `y`， `3` 綁定到 `z`。你可以將這個元組模式看作是將三個獨立的變數模式結合在一起。

在第十六章中我們見過另一個解構元組的例子，示例 16-6 中，那裡解構 `mpsc::channel()` 的返回值為 `tx`（發送者）和 `rx`（接收者）。

### 函數參數

類似於 `let`，函數參數也可以是模式。示例 18-5 中的代碼聲明了一個叫做 `foo` 的函數，它抓取一個 `i32` 類型的參數 `x`，這看起來應該很熟悉：

```rust
fn foo(x: i32) {
    // code goes here
}
```

<span class="caption">示例 18-5: 在參數中使用模式的函數簽名</span>

`x` 部分就是一個模式！類似於之前對 `let` 所做的，可以在函數參數中匹配元組。示例 18-6 展示了如何可以將傳遞給函數的元組拆分為值：

<span class="filename">文件名: src/main.rs</span>

```rust
fn print_coordinates(&(x, y): &(i32, i32)) {
    println!("Current location: ({}, {})", x, y);
}

fn main() {
    let point = (3, 5);
    print_coordinates(&point);
}
```

<span class="caption">示例 18-6: 一個在參數中解構元組的函數</span>

這會打印出 `Current location: (3, 5)`。當傳遞值 `&(3, 5)` 給 `print_coordinates` 時，這個值會匹配模式 `&(x, y)`，`x` 得到了值 3，而 `y`得到了值 5。

因為如第十三章所講閉包類似於函數，也可以在閉包參數中使用模式。

在這些可以使用模式的位置中的一個區別是，對於 `for` 循環、`let` 和函數參數，其模式必須是 *irrefutable* 的。接下來讓我們討論這個。