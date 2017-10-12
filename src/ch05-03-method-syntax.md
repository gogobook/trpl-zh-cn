## 方法語法

> [ch05-03-method-syntax.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch05-03-method-syntax.md)
> <br>
> commit 44bf3afd93519f8b0f900f21a5f2344d36e13448

**方法** 與函數類似：他們使用 `fn` 關鍵和名字聲明，可以擁有參數和返回值，同時包含一些代碼會在某處被調用時執行。不過方法與函數是不同的，因為他們在結構體（或者枚舉或者 trait 物件，將分別在第六章和第十七章講解）的上下文中被定義，並且他們第一個參數總是` self`，它代表方法被調用的結構體的實例。

### 定義方法

讓我們將抓取一個 `Rectangle` 實例作為參數的 `area` 函數改寫成一個定義於 `Rectangle` 結構體上的 `area` 方法，如示例 5-13 所示：

<span class="filename">文件名: src/main.rs</span>

```rust
#[derive(Debug)]
struct Rectangle {
    length: u32,
    width: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.length * self.width
    }
}

fn main() {
    let rect1 = Rectangle { length: 50, width: 30 };

    println!(
        "The area of the rectangle is {} square pixels.",
        rect1.area()
    );
}
```

<span class="caption">示例 5-13：在 `Rectangle` 結構體上定義 `area` 方法</span>

為了使函數定義於 `Rectangle` 的上下文中，我們開始了一個 `impl` 塊（`impl` 是 *implementation* 的縮寫）。接著將函數移動到 `impl` 大括號中，並將簽名中的第一個（在這裡也是唯一一個）參數和函數體中其他地方的對應參數改成 `self`。然後在`main` 中將我們調用 `area` 方法並傳遞 `rect1` 作為參數的地方，改成使用 **方法語法**（*method syntax*）在 `Rectangle` 實例上調用 `area` 方法。方法語法抓取一個實例並加上一個點號後跟方法名、括號以及任何參數。

在 `area` 的簽名中，開始使用 `&self` 來替代 `rectangle: &Rectangle`，因為該方法位於 `impl Rectangle` 上下文中所以 Rust 知道 `self` 的類型是 `Rectangle`。注意仍然需要在 `self` 前面加上 `&`，就像 `&Rectangle` 一樣。方法可以選擇抓取 `self` 的所有權，像我們這裡一樣不可變的借用 `self`，或者可變的借用 `self`，就跟其他別的參數一樣。

這裡選擇 `&self` 跟在函數版本中使用 `&Rectangle` 出於同樣的理由：我們並不想抓取所有權，只希望能夠讀取結構體中的數據，而不是寫入。如果想要能夠在方法中改變調用方法的實例的話，需要將第一個參數改為 `&mut self`。通過僅僅使用 `self` 作為第一個參數來使方法抓取實例的所有權，不過這是很少見的；這種技術通常用在當方法將 `self` 轉換成別的實例的時候，這時我們想要防止調用者在轉換之後使用原始的實例。

使用方法而不是函數，除了使用了方法語法和不需要在每個函數簽名中重複` self` 類型之外，其主要好處在於組織性。我將某個類型實例能做的所有事情都一起放入 `impl` 塊中，而不是讓將來的用戶在我們的代碼中到處尋找 `Rectangle` 的功能。

> ### `->`運算符到哪去了？
>
> 像在 C++ 這樣的語言中，有兩個不同的運算符來調用方法：`.` 直接在物件上調用方法，而 `->` 在一個物件的指針上調用方法，這時需要先解引用（dereference）指針。換句話說，如果 `object` 是一個指針，那麼 `object->something()` 就像 `(*object).something()` 一樣。
>
> Rust 並沒有一個與 `->` 等效的運算符；相反，Rust 有一個叫 **自動引用和解引用**（*automatic referencing and dereferencing*）的功能。方法調用是 Rust 中少數幾個擁有這種行為的地方。
>
> 這是它如何工作的：當使用 `object.something()` 調用方法時，Rust 會自動增加 `&`、`&mut` 或 `*` 以便使 `object` 符合方法的簽名。也就是說，這些代碼是等同的：
>
> ```rust
> # #[derive(Debug,Copy,Clone)]
> # struct Point {
> #     x: f64,
> #     y: f64,
> # }
> #
> # impl Point {
> #    fn distance(&self, other: &Point) -> f64 {
> #        let x_squared = f64::powi(other.x - self.x, 2);
> #        let y_squared = f64::powi(other.y - self.y, 2);
> #
> #        f64::sqrt(x_squared + y_squared)
> #    }
> # }
> # let p1 = Point { x: 0.0, y: 0.0 };
> # let p2 = Point { x: 5.0, y: 6.5 };
> p1.distance(&p2);
> (&p1).distance(&p2);
> ```
>
> 第一行看起來簡潔的多。這種自動解引用的行為之所以能行得通是因為方法有一個明確的接收者————`self` 類型。在給出接收者和方法名的前提下，Rust 可以明確的計算出方法是僅僅讀取（`&self`），做出修改（`&mut self`）或者是抓取所有權（`self`）。Rust 這種使得借用對方法接收者來說是隱式的做法是其所有權系統程式設計師友好性實踐的一大部分。

### 帶有更多參數的方法

讓我們更多的實踐一下方法，通過為 `Rectangle` 結構體實現第二個方法。這回，我們讓一個 `Rectangle` 的實例抓取另一個 `Rectangle` 實例並返回 `self` 能否完全包含第二個長方形，如果能則返回 `true` ，若不能則返回 `false`。一旦定義了 `can_hold` 方法，就可以運行示例 5-14 中的代碼了：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let rect1 = Rectangle { length: 50, width: 30 };
    let rect2 = Rectangle { length: 40, width: 10 };
    let rect3 = Rectangle { length: 45, width: 60 };

    println!("Can rect1 hold rect2? {}", rect1.can_hold(&rect2));
    println!("Can rect1 hold rect3? {}", rect1.can_hold(&rect3));
}
```

<span class="caption">示例 5-14：展示還未實現的 `can_hold` 方法的應用</span>

我們希望看到如下輸出，因為 `rect2` 的長寬都小於 `rect1`，而 `rect3` 比 `rect1` 要寬：

```text
Can rect1 hold rect2? true
Can rect1 hold rect3? false
```

因為我們想定義一個方法，所以它應該位於 `impl Rectangle` 塊中。方法名是 `can_hold`，並且它會抓取另一個 `Rectangle` 的不可變借用作為參數。通過觀察調用位置的代碼可以看出參數是什麼類型的：`rect1.can_hold(&rect2)` 傳入了 `&rect2`，它是一個 `Rectangle` 的實例 `rect2` 的不可變借用。這是可以理解的，因為我們只需要讀取 `rect2`（而不是寫入，這意味著我們需要一個可變借用）而且希望 `main` 保持 `rect2` 的所有權這樣就可以在調用這個方法後繼續使用它。`can_hold` 的返回值是一個布爾值，其實現會分別檢查 `self` 的長寬是否都大於另一個 `Rectangle`。讓我們在示例 5-13 的 `impl` 塊中增加這個新方法，如示例 5-15 所示：

<span class="filename">文件名: src/main.rs</span>

```rust
# #[derive(Debug)]
# struct Rectangle {
#     length: u32,
#     width: u32,
# }
#
impl Rectangle {
    fn area(&self) -> u32 {
        self.length * self.width
    }

    fn can_hold(&self, other: &Rectangle) -> bool {
        self.length > other.length && self.width > other.width
    }
}
```

<span class="caption">示例 5-15：在 `Rectangle` 上實現 `can_hold` 方法，它抓取另一個 `Rectangle` 實例作為參數</span>

如果結合示例 5-14 的 `main` 函數來運行，就會看到想要得到的輸出。方法可以在 `self` 後增加多個參數，而且這些參數就像函數中的參數一樣工作。

### 關聯函數

`impl` 塊的另一個有用的功能是：允許在 `impl` 塊中定義 **不** 以 `self` 作為參數的函數。這被稱為 **關聯函數**（*associated functions*），因為他們與結構體相關聯。即便如此他們仍是函數而不是方法，因為他們並不作用於一個結構體的實例。你已經使用過一個關聯函數了：`String::from`。

關聯函數經常被用作返回一個結構體新實例的構造函數。例如我們可以提供一個關聯函數，它接受一個維度參數並且同時用來作為長和寬，這樣可以更輕鬆的創建一個正方形 `Rectangle` 而不必指定兩次同樣的值：

<span class="filename">Filename: src/main.rs</span>

```rust
# #[derive(Debug)]
# struct Rectangle {
#     length: u32,
#     width: u32,
# }
#
impl Rectangle {
    fn square(size: u32) -> Rectangle {
        Rectangle { length: size, width: size }
    }
}
```

使用結構體名和 `::` 語法來調用這個關聯函數：比如 `let sq = Rectangle::square(3);`。這個方法位於結構體的命名空間中：`::` 語法用於關聯函數和模組創建的命名空間，第七章會講到後者。

### 多個 `impl` 塊

每個結構體都允許擁有多個 `impl` 塊。例如，示例 5-15 等同於示例 5-16 的代碼，這裡每個方法有其自己的 `impl` 塊：

```rust
# #[derive(Debug)]
# struct Rectangle {
#     length: u32,
#     width: u32,
# }
#
impl Rectangle {
    fn area(&self) -> u32 {
        self.length * self.width
    }
}

impl Rectangle {
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.length > other.length && self.width > other.width
    }
}
```

<span class="caption">示例 5-16：使用多個 `impl` 塊重寫示例 5-15</span>

## 總結

結構體讓我們可以在自己的範圍內創建有意義的自定義類型。通過結構體，我們可以將相關聯的數據片段聯繫起來並命名他們來使得代碼更清晰。方法允許為結構體實例指定行為，而關聯函數將特定功能置於結構體的命名空間中並且無需一個實例。

結構體並不是創建自定義類型的唯一方法；讓我們轉向 Rust 的枚舉功能並為自己的工具箱再填一個工具。