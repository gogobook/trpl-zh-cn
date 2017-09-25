## 迭代器

> [ch13-02-iterators.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch13-02-iterators.md)
> <br>
> commit 40910f557c328858f230123d1234c1cb3029dda3

迭代器模式允許你對一個項的序列進行某些處理。**迭代器**（*iterator*）負責遍歷序列中的每一項和決定序列何時結束的邏輯。當使用迭代器時，我們無需重新實現這些邏輯。

在 Rust 中，迭代器是 **惰性的**（*lazy*），這意味著直到調用方法消費迭代器之前它都不會有效果。例如，列表 13-13 中的代碼通過調用定義於 `Vec` 上的 `iter` 方法在一個 vector `v1` 上創建了一個迭代器。這段代碼本身沒有任何用處：

```rust
let v1 = vec![1, 2, 3];

let v1_iter = v1.iter();
```

<span class="caption">列表 13-13：創建了迭代器</span>

創建迭代器之後，可以選擇用多種方式利用它。在列表 3-6 中，我們實際上使用了迭代器和 `for` 循環在每一個項上執行了一些代碼，直到現在我們才解釋了 `iter` 調用做了什麼。列表 13-14 中的例子將迭代器的創建和 `for` 循環中的使用分開。迭代器被儲存在 `v1_iter` 變數中，而這時沒有進行迭代。一旦 `for` 循環開始使用 `v1_iter`，接著迭代器中的每一個元素被用於循環的一次迭代，這會打印出其每一個值：

```rust
let v1 = vec![1, 2, 3];

let v1_iter = v1.iter();

for val in v1_iter {
    println!("Got: {}", val);
}
```

<span class="caption">列表 13-14：在一個 `for` 循環中使用迭代器</span>

在標準庫中沒有提供迭代器的語言中，我們可能會使用一個從 0 開始的索引變數，使用這個變數索引 vector 中的值，並循環增加其值直到達到 vector 的元素數量。迭代器為我們處理了所有這些邏輯，這減少了重複代碼並潛在的消除了混亂。另外，迭代器的實現方式提供了對多種不同的序列使用相同邏輯的靈活性，而不僅僅是像 vector 這樣可索引的數據結構.讓我們看看迭代器是如何做到這些的。

### `Iterator` trait 和 `next` 方法

迭代器都實現了一個叫做 `Iterator` 的定義於標準庫的 trait。這個 trait 的定義看起來像：

```rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // methods with default implementations elided
}
```

這裡有一下我們還未講到的新語法：`type Item` 和 `Self::Item`，他們定義了 trait 的 **關聯類型**（*associated type*）。第十九章會深入講解關聯類型，不過現在只需知道這段代碼表明實現 `Iterator` trait 要求同時定義一個 `Item` 類型，這個 `Item` 類型被用作 `next` 方法的返回值類型。換句話說，`Item` 類型將是迭代器返回元素的類型。

`next` 是 `Iterator` 實現者被要求定義的唯一方法。`next` 一次返回迭代器中的一個項，封裝在 `Some` 中，當迭代器結束時，它返回 `None`。如果你希望的話可以直接調用迭代器的 `next` 方法；列表 13-15 有一個測試展示了重複調用由 vector 創建的迭代器的 `next` 方法所得到的值：

<span class="filename">文件名: src/lib.rs</span>

```rust,test_harness
#[test]
fn iterator_demonstration() {
    let v1 = vec![1, 2, 3];

    let mut v1_iter = v1.iter();

    assert_eq!(v1_iter.next(), Some(&1));
    assert_eq!(v1_iter.next(), Some(&2));
    assert_eq!(v1_iter.next(), Some(&3));
    assert_eq!(v1_iter.next(), None);
}
```

<span class="caption">列表 13-15：在迭代器上（直接）調用 `next` 方法</span>

注意 `v1_iter` 需要是可變的：在迭代器上調用 `next` 方法改變了迭代器中用來記錄序列位置的狀態。換句話說，代碼 **消費**（consume）了，或使用了迭代器。每一個 `next` 調用都會從迭代器中吃掉一個項。使用 `for` 循環時無需使 `v1_iter` 可變因為 `for` 循環會抓取 `v1_iter` 的所有權並在後台使 `v1_iter` 可變。

另外需要注意到從 `next` 調用中得到的值是 vector 的不可變引用。`iter` 方法生成一個不可變引用的迭代器。如果我們需要一個抓取 `v1` 所有權並返回擁有所有權的迭代器，則可以調用 `into_iter` 而不是 `iter`。類似的，如果我們希望迭代可變引用，則可以調用 `iter_mut` 而不是 `iter`。

### `Iterator` trait 中消費迭代器的方法

`Iterator` trait 有一系列不同的由標準庫提供默認實現的方法；你可以在 `Iterator` trait 的標準庫 API 文檔中找到所有這些方法。一些方法在其定義中調用了 `next` 方法，這也就是為什麼在實現 `Iterator` trait 時要求實現 `next` 方法的原因。

這些調用 `next` 方法的方法被稱為 **消費適配器**（*consuming adaptors*），因為調用他們會消耗迭代器。一個消費適配器的例子是 `sum` 方法。這個方法抓取迭代器的所有權並反覆調用 `next` 來遍歷迭代器，因而會消費迭代器。當其遍歷每一個項時，它將每一個項加總到一個總和並在迭代完成時返回總和。列表 13-16 有一個展示 `sum` 方法使用的測試：

<span class="filename">文件名: src/lib.rs</span>

```rust
#[test]
fn iterator_sum() {
    let v1 = vec![1, 2, 3];

    let v1_iter = v1.iter();

    let total: i32 = v1_iter.sum();

    assert_eq!(total, 6);
}
```

<span class="caption">列表 13-16：調用 `sum` 方法抓取迭代器所有項的總和</span>

調用 `sum` 之後不再允許使用 `v1_iter` 因為調用 `sum` 時它會抓取迭代器的所有權。

### `Iterator` trait 中產生其他迭代器的方法

`Iterator` trait 中定義的另一類方法會產生其他的迭代器。這些方法被稱為 **迭代器適配器**（*iterator adaptors*），他們允許我們將當前迭代器變為不同類型的迭代器。可以鏈式調用多個迭代器適配器。不過因為所有的迭代器都是惰性的，必須調用一個消費適配器方法以便抓取迭代器適配器調用的結果。列表 13-17 展示了一個調用迭代器適配器方法 `map` 的例子，它會抓取一個 `map` 會在每一個項上調用的閉包來產生一個新迭代器，它的每一項為 vector 中每一項加一。不過這些代碼會產生一個警告：

<span class="filename">文件名: src/main.rs</span>

```rust
let v1: Vec<i32> = vec![1, 2, 3];

v1.iter().map(|x| x + 1);
```

<span class="caption">列表 13-17：調用迭代器適配器 `map` 來創建一個新迭代器</span>

得到的警告是：

```text
warning: unused result which must be used: iterator adaptors are lazy and do
nothing unless consumed
 --> src/main.rs:4:1
  |
4 | v1.iter().map(|x| x + 1);
  | ^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: #[warn(unused_must_use)] on by default
```

列表 13-17 中的代碼實際上並沒有做任何事；所指定的閉包從未被調用過。警告提醒了我們為什麼：迭代器適配器是惰性的，而這裡我們可能意在消費迭代器。

為了修復這個警告並消費迭代器抓取有用的結果，我們將使用第十二章簡要講到的 `collect` 方法。這個方法消費迭代器並將結果收集到一個數據結構中。在列表 13-18 中，我們將遍歷由 `map` 調用生成的迭代器的結果收集到一個 vector 中，它將會含有原始 vector 中每個元素加一的結果：

<span class="filename">文件名: src/main.rs</span>

```rust
let v1: Vec<i32> = vec![1, 2, 3];

let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();

assert_eq!(v2, vec![2, 3, 4]);
```

<span class="caption">列表 13-18：調用 `map` 方法創建一個新迭代器，接著調用 `collect` 方法消費新迭代器並創建一個 vector</span>

因為 `map` 抓取一個閉包，可以指定任何希望在遍歷的每個元素上執行的操作。這是一個展示如何使用閉包來自定義行為同時又復用 `Iterator` trait 提供的迭代行為的絕佳例子。

### 使用閉包抓取環境與迭代器

現在我們介紹了迭代器，讓我們展示一個通過使用 `filter` 迭代器適配器和捕獲環境的閉包的常規用例。迭代器的 `filter` 方法抓取一個使用迭代器的每一個項並返回布爾值的閉包。如果閉包返回 `true`，其值將會包含在 `filter` 提供的新迭代器中。如果閉包返回 `false`，其值不會包含在結果迭代器中。列表 13-19 展示了使用 `filter` 和一個捕獲環境中變數 `shoe_size` 的閉包，這樣閉包就可以遍歷一個 `Shoe` 結構體集合以便只返回指定大小的鞋子：

<span class="filename">文件名: src/lib.rs</span>

```rust,test_harness
#[derive(PartialEq, Debug)]
struct Shoe {
    size: i32,
    style: String,
}

fn shoes_in_my_size(shoes: Vec<Shoe>, shoe_size: i32) -> Vec<Shoe> {
    shoes.into_iter()
        .filter(|s| s.size == shoe_size)
        .collect()
}

#[test]
fn filters_by_size() {
    let shoes = vec![
        Shoe { size: 10, style: String::from("sneaker") },
        Shoe { size: 13, style: String::from("sandal") },
        Shoe { size: 10, style: String::from("boot") },
    ];

    let in_my_size = shoes_in_my_size(shoes, 10);

    assert_eq!(
        in_my_size,
        vec![
            Shoe { size: 10, style: String::from("sneaker") },
            Shoe { size: 10, style: String::from("boot") },
        ]
    );
}
```

<span class="caption">列表 13-19：使用 `filter` 方法和一個捕獲 `shoe_size` 的閉包</span>

`shoes_in_my_size` 函數抓取一個鞋子 vector 的所有權和一個鞋子大小作為參數。它返回一個只包含指定大小鞋子的 vector。在 `shoes_in_my_size` 函數體中調用了 `into_iter` 來創建一個抓取 vector 所有權的迭代器。接著調用 `filter` 將這個迭代器適配成只含有閉包返回 `true` 元素的新迭代器。我們指定的閉包從環境中捕獲了 `shoe_size` 變數並使用其值與每一隻鞋的大小作比較，只保留指定大小的鞋子。最終，調用 `collect` 將迭代器適配器返回的值收集進一個 vector 並返回。

這個測試展示當調用 `shoes_in_my_size` 時，我們只會得到與指定值相同大小的鞋子。

### 實現 `Iterator` trait 來創建自定義迭代器

我們已經展示了可以通過在 vector 上調用 `iter`、`into_iter` 或 `iter_mut` 來創建一個迭代器。也可以用標準庫中其他的集合類型創建迭代器，比如哈希 map。另外，可以實現 `Iterator` trait 來創建任何我們希望的迭代器。正如之前提到的，定義中唯一要求提供的方法就是 `next` 方法。一旦定義了它，就可以使用所有其他由 `Iterator` trait 提供的擁有默認實現的方法來創建自定義迭代器了！

我們將要創建的迭代器只會從 1 數到 5。首先，我們會創建一個結構體來存放一些值，接著實現 `Iterator` trait 將這個結構體放入迭代器中並在此實現中使用其值。

列表 13-20 有一個 `Counter` 結構體定義和一個創建 `Counter` 實例的關聯函數 `new`：


<span class="filename">文件名: src/lib.rs</span>

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}
```

<span class="caption">列表 13-20：定義 `Counter` 結構體和一個創建 `count` 初值為 0 的 `Counter` 實例的 `new` 函數</span>

`Counter` 結構體有一個字段 `count`。這個字段存放一個 `u32` 值，它會記錄處理 1 到 5 的迭代過程中的位置。`count` 是私有的因為我們希望 `Counter` 的實現來管理這個值。`new` 函數通過總是從為 0 的 `count` 字段開始新實例來確保我們需要的行為。

接下來將為 `Counter` 類型實現 `Iterator` trait，通過定義 `next` 方法來指定使用迭代器時的行為，如列表 13-21 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
# struct Counter {
#     count: u32,
# }
#
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;

        if self.count < 6 {
            Some(self.count)
        } else {
            None
        }
    }
}
```

<span class="caption">列表 13-21：在 `Counter` 結構體上實現 `Iterator` trait</span>

這裡將迭代器的關聯類型 `Item` 設置為 `u32`，意味著迭代器會返回 `u32` 值集合。再一次，這裡仍無需擔心關聯類型，第十九章會講到。我們希望迭代器對其內部狀態加一，這也就是為何將 `count` 初始化為 0：我們希望迭代器首先返回 1。如果 `count` 值小於 6，`next` 會返回封裝在 `Some` 中的當前值，不過如果 `count` 大於或等於 6，迭代器會返回 `None`。

#### 使用 `Counter` 迭代器的 `next` 方法

一旦實現了 `Iterator` trait，我們就有了一個迭代器！列表 13-22 展示了一個測試用來演示現在我們可以使用 `Counter` 結構體的迭代器功能，通過直接調用 `next` 方法，正如列表 13-15 中從 vector 創建的迭代器那樣：

<span class="filename">文件名: src/lib.rs</span>

```rust
# struct Counter {
#     count: u32,
# }
#
# impl Iterator for Counter {
#     type Item = u32;
#
#     fn next(&mut self) -> Option<Self::Item> {
#         self.count += 1;
#
#         if self.count < 6 {
#             Some(self.count)
#         } else {
#             None
#         }
#     }
# }
#
#[test]
fn calling_next_directly() {
    let mut counter = Counter::new();

    assert_eq!(counter.next(), Some(1));
    assert_eq!(counter.next(), Some(2));
    assert_eq!(counter.next(), Some(3));
    assert_eq!(counter.next(), Some(4));
    assert_eq!(counter.next(), Some(5));
    assert_eq!(counter.next(), None);
}
```

<span class="caption">列表 13-22：測試 `next` 方法實現的功能</span>

這個測試在 `counter` 變數中新建了一個 `Counter` 實例並接著反覆調用 `next` 方法，來驗證我們實現的行為符合這個迭代器返回從 1 到 5 的值的預期。

#### 使用自定義迭代器中其他 `Iterator` trait 方法

通過定義 `next` 方法實現 `Iterator` trait，我們現在就可以使用任何標準庫定義的擁有默認實現的 `Iterator` trait 方法了，因為他們都使用了 `next` 方法的功能。

例如，出於某種原因我們希望抓取 `Counter` 實例產生的值，將這些值與另一個 `Counter` 實例在省略了第一個值之後產生的值配對，將每一對值相乘，只保留那些可以被三整除的結果，然後將所有保留的結果相加，這可以如列表 13-23 中的測試這樣做：

<span class="filename">文件名: src/lib.rs</span>

```rust
# struct Counter {
#     count: u32,
# }
#
# impl Counter {
#     fn new() -> Counter {
#         Counter { count: 0 }
#     }
# }
#
# impl Iterator for Counter {
#     // Our iterator will produce u32s
#     type Item = u32;
#
#     fn next(&mut self) -> Option<Self::Item> {
#         // increment our count. This is why we started at zero.
#         self.count += 1;
#
#         // check to see if we've finished counting or not.
#         if self.count < 6 {
#             Some(self.count)
#         } else {
#             None
#         }
#     }
# }
#
#[test]
fn using_other_iterator_trait_methods() {
    let sum: u32 = Counter::new().zip(Counter::new().skip(1))
                                 .map(|(a, b)| a * b)
                                 .filter(|x| x % 3 == 0)
                                 .sum();
    assert_eq!(18, sum);
}
```

<span class="caption">列表 13-23：使用自定義的 `Counter` 迭代器的多種方法</span>

注意 `zip` 只產生4對值；理論上第五對值 `(5, None)` 從未被產生，因為 `zip` 在任一輸入迭代器返回 `None` 時也返回 `None`。

所有這些方法調用都是可能的，因為我們通過指定 `next` 如何工作來實現 `Iterator` trait 而標準庫則提供其他調用 `next` 的默認方法實現。