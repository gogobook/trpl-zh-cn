## `match` 控制流運算符

> [ch06-02-match.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch06-02-match.md)
> <br>
> commit 01dd4248621c2f510947592e47d16bdab9b14cf0

Rust 有一個叫做 `match` 的極為強大的控制流運算符，它允許我們將一個值與一系列的模式相比較並根據匹配的模式執行相應代碼。模式可由字面值、變量、通配符和許多其他內容構成；第十八章會涉及到所有不同種類的模式以及他們的作用。`match` 的力量來源於模式的表現力以及編譯器檢查，它確保了所有可能的情況都得到處理。

可以把 `match` 表達式想像成某種硬幣分類器：硬幣滑入有著不同大小孔洞的軌道，每一個硬幣都會掉入符合它大小的孔洞。同樣地，值也會檢查 `match` 的每一個模式，並且在遇到第一個 「符合」 的模式時，值會進入相關聯的代碼塊並在執行中被使用。

因為剛剛提到了硬幣，讓我們用他們來作為一個使用 `match` 的例子！我們可以編寫一個函數來獲取一個未知的（美帝）硬幣，並以一種類似驗鈔機的方式，確定它是何種硬幣並返回它的美分值，如列表 6-3 中所示：

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: Coin) -> u32 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```

<span class="caption">列表 6-3：一個枚舉和一個以枚舉成員作為模式的 `match` 表達式</span>

拆開 `value_in_cents` 函數中的 `match` 來看。首先，我們列出 `match` 關鍵字後跟一個表達式，在這個例子中是 `coin` 的值。這看起來非常像 `if` 使用的表達式，不過這裡有一個非常大的區別：對於 `if`，表達式必須返回一個布爾值。而這裡它可以是任何類型的。例子中的 `coin` 的類型是列表 6-3 中定義的 `Coin` 枚舉。

接下來是 `match` 的分支。一個分支有兩個部分：一個模式和一些代碼。第一個分支的模式是值 `Coin::Penny` 而之後的 `=>` 運算符將模式和將要運行的代碼分開。這裡的代碼就僅僅是值 `1`。每一個分支之間使用逗號分隔。

當 `match` 表達式執行時，它將結果值按順序與每一個分支的模式相比較，如果模式匹配了這個值，這個模式相關聯的代碼將被執行。如果模式並不匹配這個值，將繼續執行下一個分支，非常類似一個硬幣分類器。可以擁有任意多的分支：列表 6-3 中的 `match` 有四個分支。

每個分支相關聯的代碼是一個表達式，而表達式的結果值將作為整個 `match`  表達式的返回值。

如果分支代碼較短的話通常不使用大括號，正如列表 6-3 中的每個分支都只是返回一個值。如果想要在分支中運行多行代碼，可以使用大括號。例如，如下代碼在每次使用`Coin::Penny` 調用時都會打印出 「Lucky penny!」，同時仍然返回代碼塊最後的值，`1`：

```rust
# enum Coin {
#    Penny,
#    Nickel,
#    Dime,
#    Quarter,
# }
#
fn value_in_cents(coin: Coin) -> u32 {
    match coin {
        Coin::Penny => {
            println!("Lucky penny!");
            1
        },
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```

### 綁定值的模式

匹配分支的另一個有用的功能是可以綁定匹配的模式的部分值。這也就是如何從枚舉成員中提取值。

作為一個例子，讓我們修改枚舉的一個成員來存放數據。1999 年到 2008 年間，美帝在 25 美分的硬幣的一側為 50 個州的每一個都印刷了不同的設計。其他的硬幣都沒有這種區分州的設計，所以只有這些 25 美分硬幣有特殊的價值。可以將這些信息加入我們的 `enum`，通過改變 `Quarter` 成員來包含一個 `State` 值，列表 6-4 中完成了這些修改：

```rust
#[derive(Debug)] // So we can inspect the state in a minute
enum UsState {
    Alabama,
    Alaska,
    // ... etc
}

enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(UsState),
}
```

<span class="caption">列表 6-4：`Quarter` 成員也存放了一個 `UsState` 值的 `Coin` 枚舉</span>

想像一下我們的一個朋友嘗試收集所有 50 個州的 25 美分硬幣。在根據硬幣類型分類零錢的同時，也可以報告出每個 25 美分硬幣所對應的州名稱，這樣如果我們的朋友沒有的話，他可以把它加入收藏。

在這些代碼的匹配表達式中，我們在匹配 `Coin::Quarter` 成員的分支的模式中增加了一個叫做 `state` 的變量。當匹配到 `Coin::Quarter` 時，變量 `state` 將會綁定 25 美分硬幣所對應州的值。接著在那個分支的代碼中使用 `state`，如下：

```rust
# #[derive(Debug)]
# enum UsState {
#    Alabama,
#    Alaska,
# }
#
# enum Coin {
#    Penny,
#    Nickel,
#    Dime,
#    Quarter(UsState),
# }
#
fn value_in_cents(coin: Coin) -> u32 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("State quarter from {:?}!", state);
            25
        },
    }
}
```

如果調用 `value_in_cents(Coin::Quarter(UsState::Alaska))`，`coin` 將是 `Coin::Quarter(UsState::Alaska)`。當將值與每個分支相比較時，沒有分支會匹配，直到遇到 `Coin::Quarter(state)`。這時，`state` 綁定的將會是值 `UsState::Alaska`。接著就可以在 `println!` 表達式中使用這個綁定了，像這樣就可以獲取 `Coin` 枚舉的 `Quarter` 成員中內部的州的值。

### 匹配 `Option<T>`

在之前的部分在使用 `Option<T>` 時我們想要從 `Some` 中取出其內部的 `T` 值；也可以像處理 `Coin` 枚舉那樣使用 `match` 處理 `Option<T>`！與其直接比較硬幣，我們將比較 `Option<T>` 的成員，不過 `match` 表達式的工作方式保持不變。

比如我們想要編寫一個函數，它獲取一個 `Option<i32>` 並且如果其中有一個值，將其加一。如果其中沒有值，函數應該返回 `None` 值並不嘗試執行任何操作。

得益於 `match`，編寫這個函數非常簡單，它將看起來像列表 6-5 中這樣：

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
}

let five = Some(5);
let six = plus_one(five);
let none = plus_one(None);
```

<span class="caption">列表 6-5：一個在 `Option<i32>` 上使用 `match` 表達式的函數</span>

#### 匹配 `Some(T)`

讓我們更仔細的檢查 `plus_one` 的第一行操作。當調用 `plus_one(five)` 時，`plus_one` 函數體中的 `x` 將會是值 `Some(5)`。接著將其與每個分支比較。

```rust
None => None,
```

值 `Some(5)` 並不匹配模式 `None`，所以繼續進行下一個分支。

```rust
Some(i) => Some(i + 1),
```

`Some(5)` 與 `Some(i)` 匹配嗎？為什麼不呢！他們是相同的成員。`i` 綁定了 `Some` 中包含的值，所以 `i` 的值是 `5`。接著匹配分支的代碼被執行，所以我們將 `i` 的值加一併返回一個含有值 `6` 的新 `Some`。

#### 匹配 `None`

接著考慮下列表 6-5 中 `plus_one` 的第二個調用，這裡 `x` 是 `None`。我們進入 `match` 並與第一個分支相比較。

```rust
None => None,
```

匹配上了！這裡沒有值來加一，所以程序結束並返回 `=>` 右側的值 `None`，因為第一個分支就匹配到了，其他的分支將不再比較。

將 `match` 與枚舉相結合在很多場景中都是有用的。你會在 Rust 代碼中看到很多這樣的模式：`match` 一個枚舉，綁定其中的值到一個變量，接著根據其值執行代碼。這在一開始有點複雜，不過一旦習慣了，你會希望所有語言都擁有它！這一直是用戶的最愛。

### 匹配是窮盡的

`match` 還有另一方面需要討論。考慮一下 `plus_one` 函數的這個版本：

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        Some(i) => Some(i + 1),
    }
}
```

我們沒有處理 `None` 的情況，所以這些代碼會造成一個 bug。幸運的是，這是一個 Rust 知道如何處理的 bug。如果嘗試編譯這段代碼，會得到這個錯誤：

```text
error[E0004]: non-exhaustive patterns: `None` not covered
 -->
  |
6 |         match x {
  |               ^ pattern `None` not covered
```


Rust 知道我們沒有覆蓋所有可能的情況甚至知道那些模式被忘記了！Rust 中的匹配是 **窮盡的**（*exhaustive）：必須窮舉到最後的可能性來使代碼有效。特別的在這個 `Option<T>` 的例子中，Rust 防止我們忘記明確的處理 `None` 的情況，這使我們免於假設擁有一個實際上為空的值，這造成了之前提到過的價值億萬的錯誤。

### `_` 通配符

Rust 也提供了一個模式用於不想列舉出所有可能值的場景。例如，`u8` 可以擁有 0 到 255 的有效的值，如果我們只關心 1、3、5 和 7 這幾個值，就並不想必須列出 0、2、4、6、8、9 一直到 255 的值。所幸我們不必這麼做：可以使用特殊的模式 `_` 替代：

```rust
let some_u8_value = 0u8;
match some_u8_value {
    1 => println!("one"),
    3 => println!("three"),
    5 => println!("five"),
    7 => println!("seven"),
    _ => (),
}
```

`_` 模式會匹配所有的值。通過將其放置於其他分支之後，`_` 將會匹配所有之前沒有指定的可能的值。`()` 就是 unit 值，所以 `_` 的情況什麼也不會發生。因此，可以說我們想要對 `_` 通配符之前沒有列出的所有可能的值不做任何處理。

然而，`match` 在只關心 **一個** 情況的場景中可能就有點囉嗦了。為此 Rust 提供了`if let`。