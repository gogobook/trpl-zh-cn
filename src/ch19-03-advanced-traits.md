## 高級 trait

> [ch19-03-advanced-traits.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch19-03-advanced-traits.md)
> <br>
> commit f8727711388b28eb2f5c852dd83fdbe6d22ab9bb

第十章講到了 trait，不過就像生命週期，我們並沒有涉及所有的細節。現在我們更加瞭解 Rust 了，可以深入理解本質了。

### 關聯類型

**關聯類型**（*associated types*）是一個將類型佔位符與 trait 相關聯的方式，這樣 trait 的方法簽名中就可以使用這些佔位符類型。實現一個 trait 的人只需要針對專門的實現在這個類型的位置指定相應的類型即可。

本章描述的大部分內容都非常少見。關聯類型則比較適中；它們比本書其他的內容要少見，不過比本章中的很多內容要更常見。

一個帶有關聯類型的 trait 的例子是標準庫提供的 `Iterator` trait。它有一個叫做 `Item` 的關聯類型來替代遍歷的值的類型。第十三章曾提到過 `Iterator` trait 的定義如代碼例 19-20 所示：

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

<span class="caption">代碼例 19-20：`Iterator` trait 的定義中帶有關聯類型 `Item`</span>

這就是說 `Iterator` trait 有一個關聯類型 `Item`。`Item` 是一個佔位類型，同時 `next` 方法會返回 `Option<Self::Item>` 類型的值。這個 trait 的實現者會指定 `Item` 的具體類型，然而不管實現者指定何種類型, `next` 方法都會返回一個包含了這種類型值的 `Option`。

#### 關聯類型 vs 泛型

當在代碼例 13-6 中在 `Counter` 結構體上實現 `Iterator` trait 時，將 `Item` 的類型指定為 `u32`：

```rust
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
```

這感覺類似於泛型。那麼為什麼 `Iterator` trait 不像代碼例 19-21 那樣定義呢？

```rust
pub trait Iterator<T> {
    fn next(&mut self) -> Option<T>;
}
```

<span class="caption">代碼例 19-21：一個使用泛型的 `Iterator` trait 假象定義</span>

區別是在代碼例 19-21 的定義中，我們也可以實現 `Iterator<String> for Counter`，或者任何其他類型，這樣就可以有多個 `Counter` 的 `Iterator` 的實現。換句話說，當 trait 有泛型參數時，可以多次實現這個 trait，每次需改變泛型參數的具體類型。接著當使用 `Counter` 的 `next` 方法時，必須提供類型註解來表明希望使用 `Iterator` 的哪一個實現。

通過關聯類型，不能多次實現 trait。使用代碼例 19-20 中這個 `Iterator` 的具體定義，只能選擇一次 `Item` 會是什麼類型，因為只能有一個 `impl Iterator for Counter`。當調用 `Counter` 的 `next` 時不必每次指定我們需要 `u32` 值的迭代器。

當 trait 使用關聯類型時不必指定泛型參數的好處也在另外一些方面得到體現。考慮一下代碼例 19-22 中定義的兩個 trait。他們都必須處理一個包含一些節點和邊的圖結構。`GGraph` 定義為使用泛型，而 `AGraph` 定義為使用關聯類型：

```rust
trait GGraph<Node, Edge> {
    // methods would go here
}

trait AGraph {
    type Node;
    type Edge;

    // methods would go here
}
```

<span class="caption">代碼例 19-22：兩個圖 trait 定義，`GGraph` 使用泛型而 `AGraph` 使用關聯類型代表 `Node` 和 `Edge`</span>

比如說想要是實現一個計算任何實現了圖 trait 的類型中兩個節點之間距離的函數。對於使用泛型的 `GGraph` trait 來說，`distance` 函數的簽名看起來應該如代碼例 19-23 所示：

```rust
# trait GGraph<Node, Edge> {}
#
fn distance<N, E, G: GGraph<N, E>>(graph: &G, start: &N, end: &N) -> u32 {
#     0
}
```

<span class="caption">代碼例 19-23：`distance` 函數的簽名，它使用 `GGraph` trait 並必須指定所有的泛型參數</span>

函數需要指定泛型參數 `N`、`E` 和 `G`，其中 `G` 擁有以 `N` 類型作為 `Node` 和 `E` 類型作為 `Edge` 的 `GGraph` trait 作為 trait bound。即便 `distance` 函數無需指定邊的類型，我們也強制聲明了 `E` 參數，因為需要使用 `GGraph` trait, 而 `GGraph` 需要指定 `Edge` 的類型。

與此相對，代碼例 19-24 中的 `distance` 定義使用代碼例 19-22 中帶有關聯類型的 `AGraph` trait：

```rust
# trait AGraph {
#     type Node;
#     type Edge;
# }
#
fn distance<G: AGraph>(graph: &G, start: &G::Node, end: &G::Node) -> u32 {
#     0
}
```

<span class="caption">代碼例 19-24：`distance` 函數的簽名，它使用 trait `AGraph` 和關聯類型 `Node`</span>

這樣就清楚多了。只需指定一個泛型參數 `G`，帶有 `AGraph` trait bound。因為 `distance` 完全不需要使用 `Edge` 類型，無需每次都指定它。為了使用 `AGraph` 的關聯類型 `Node`，可以指定為 `G::Node`。

#### 帶有關聯類型的 trait 物件
**TODO** 待英文原版確認中

你可能會好奇為什麼不在代碼例 19-23 和 19-24 的 `distance` 函數中使用 trait 物件。當使用 trait 物件時使用泛型 `GGraph` trait 的 `distance` 函數的簽名確實更準確了一些：

```rust
# trait GGraph<Node, Edge> {}
#
fn distance<N, E>(graph: &GGraph<N, E>, start: &N, end: &N) -> u32 {
#     0
}
```

與代碼例 19-24 相比較可能更顯公平。不過依然需要指定 `Edge` 類型，這意味著代碼例 19-24 仍更為合適，因為無需指定並不需要的類型。

不可能改變代碼例 19-24 來對圖使用 trait 物件，因為這樣就無法引用 `AGraph` trait 中的關聯類型。

但是一般而言常見的情形是使用帶有關聯類型 trait 的 trait 物件；代碼例 19-25 展示了一個函數 `traverse` ，它無需在其他參數中使用關聯類型。然而這種情況必須指定關聯類型的具體類型。這裡選擇接受以 `usize` 作為 `Node` 和以兩個 `usize` 值的元組作為  `Edge` 的實現了 `AGraph` trait 的類型：

```rust
# trait AGraph {
#     type Node;
#     type Edge;
# }
#
fn traverse(graph: &AGraph<Node=usize, Edge=(usize, usize)>) {}
```

雖然 trait 物件意味著無需在編譯時就知道 `graph` 參數的具體類型，但是我們確實需要在 `traverse` 函數中通過具體的關聯類型來限制 `AGraph` trait 的使用。如果不提供這樣的限制，Rust 將不能計算出用哪個 `impl` 來匹配這個 trait 物件，因為關聯類型可以作為方法簽名的一部分，Rust 需要在虛函數表(vtable)中查找它們。

### 運算符重載和預設類型參數

`<PlaceholderType=ConcreteType>` 語法也可以以另一種方式使用：用來指定泛型的預設類型。這種情況的一個非常好的例子是用於運算符重載。

Rust 並不允許創建自定義運算符或重載任意運算符，不過 `std::ops` 中所列出的運算符和相應的 trait 可以通過實現運算符相關 trait 來重載。例如，代碼例 19-25 中展示了如何在 `Point` 結構體上實現 `Add` trait 來重載 `+` 運算符，這樣就可以將兩個 `Point` 實例相加了：

<span class="filename">文件名: src/main.rs</span>

```rust
use std::ops::Add;

#[derive(Debug,PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Point;

    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

fn main() {
    assert_eq!(Point { x: 1, y: 0 } + Point { x: 2, y: 3 },
               Point { x: 3, y: 3 });
}
```

<span class="caption">代碼例 19-25：實現 `Add` 來重載 `Point` 的 `+` 運算符</span>

這裡實現了 `add` 方法將兩個 `Point` 實例的 `x` 值和 `y` 值分別相加來創建一個新的 `Point`。`Add` trait 有一個叫做 `Output` 的關聯類型，它用來決定 `add` 方法的返回值類型。

讓我們更仔細的看看 `Add` trait。這裡是其定義：

```rust
trait Add<RHS=Self> {
    type Output;

    fn add(self, rhs: RHS) -> Self::Output;
}
```

這看來應該很熟悉；這是一個帶有一個方法和一個關聯類型的 trait。比較陌生的部分是尖括號中的 `RHS=Self`：這個語法叫做**預設類型參數**（*default type parameters*）。`RHS` 是一個泛型參數（「right hand side」 的縮寫），它用於 `add` 方法中的 `rhs` 參數。如果實現 `Add` trait 時不指定 `RHS` 的具體類型，`RHS` 的類型將是預設的 `Self` 類型（在其上實現 `Add` 的類型）。

讓我們看看另一個實現了 `Add` trait 的例子。想像一下我們擁有兩個存放不同的單元值的結構體，`Millimeters` 和 `Meters`。可以如代碼例 19-26 所示那樣用不同的方式為 `Millimeters` 實現 `Add` trait：

```rust
use std::ops::Add;

struct Millimeters(u32);
struct Meters(u32);

impl Add for Millimeters {
    type Output = Millimeters;

    fn add(self, other: Millimeters) -> Millimeters {
        Millimeters(self.0 + other.0)
    }
}

impl Add<Meters> for Millimeters {
    type Output = Millimeters;

    fn add(self, other: Meters) -> Millimeters {
        Millimeters(self.0 + (other.0 * 1000))
    }
}
```

<span class="caption">代碼例 19-26：在 `Millimeters` 上實現 `Add`，以能夠將`Millimeters` 與 `Millimeters` 相加和將 `Millimeters` 與 `Meters` 相加</span>

如果將 `Millimeters` 與其他 `Millimeters` 相加，則無需為 `Add` 參數化 `RHS` 類型，因為預設的 `Self` 正是我們希望的。如果希望實現 `Millimeters` 與 `Meters` 相加，那麼需要聲明為 `impl Add<Meters>` 來設定 `RHS` 類型參數的值。

預設參數類型主要用於如下兩個方面：

1. 擴展類型而不破壞現有代碼。
2. 允許以一種大部分用戶都不需要的方法進行自定義。

`Add` trait 就是第二個目的一個例子：大部分時候你會將兩個相似的類型相加。在 `Add` trait 定義中使用預設類型參數使得實現 trait 變得更容易，因為大部分時候無需指定這額外的參數。換句話說，這樣就去掉了一些實現的樣板代碼。

第一個目的是相似的，但過程是反過來的：因為現有 trait 實現並沒有指定類型參數，如果需要為現有 trait 增加類型參數，為其提供一個預設值將允許我們在不破壞現有實現代碼的基礎上擴展 trait 的功能。

### 完全限定語法與消歧義

Rust 既不能避免一個 trait 與另一個 trait 擁有相同名稱的方法，也不能阻止為同一類型同時實現這兩個 trait。甚至也可以直接在類型上實現相同名稱的方法！那麼為了能使用相同的名稱調用每一個方法，需要告訴 Rust 我們希望使用哪個方法。考慮一下代碼例 19-27 中的代碼，trait `Foo` 和 `Bar` 都擁有方法 `f`，並在結構體 `Baz` 上實現了這兩個 trait，結構體也有一個叫做 `f` 的方法：

<span class="filename">文件名: src/main.rs</span>

```rust
trait Foo {
    fn f(&self);
}

trait Bar {
    fn f(&self);
}

struct Baz;

impl Foo for Baz {
    fn f(&self) { println!("Baz's impl of Foo"); }
}

impl Bar for Baz {
    fn f(&self) { println!("Baz's impl of Bar"); }
}

impl Baz {
    fn f(&self) { println!("Baz's impl"); }
}

fn main() {
    let b = Baz;
    b.f();
}
```

<span class="caption">代碼例 19-27：實現兩個擁有相同名稱的方法的 trait，同時還有直接定義於結構體的（同名）方法</span>

對於 `Baz` 的 `Foo` trait 中方法 `f` 的實現，它打印出 `Baz's impl of Foo`。對於 `Baz` 的 `Bar` trait 中方法 `f` 的實現，它打印出 `Baz's impl of Bar`。直接定義於 `Baz` 的 `f` 實現打印出 `Baz's impl`。當調用 `b.f()` 時會發生什麼呢？在這個例子中，Rust 總是會使用直接定義於 `Baz` 的實現並打印出 `Baz's impl`。

為了能夠調用 `Foo` 和 `Baz` 中的 `f` 方法而不是直接定義於 `Baz` 的 `f` 實現，則需要使用**完全限定語法**（*fully qualified syntax*）來調用方法。它像這樣工作：對於任何類似如下的方法調用：

```rust
receiver.method(args);
```

可以像這樣使用完全限定的方法調用：

```rust
<Type as Trait>::method(receiver, args);
```

所以為了消歧義並能夠調用代碼例 19-27 中所有的 `f` 方法，需要在尖括號中指定每個希望 `Baz` 作為的 trait，接著使用雙冒號，接著傳遞 `Baz` 實例作為第一個參數並調用 `f` 方法。代碼例 19-28 展示了如何調用 `Foo` 中的 `f`，和 `Bar` 中與 `b` 中的 `f`：

<span class="filename">文件名: src/main.rs</span>

```rust
# trait Foo {
#     fn f(&self);
# }
# trait Bar {
#     fn f(&self);
# }
# struct Baz;
# impl Foo for Baz {
#     fn f(&self) { println!("Baz's impl of Foo"); }
# }
# impl Bar for Baz {
#     fn f(&self) { println!("Baz's impl of Bar"); }
# }
# impl Baz {
#     fn f(&self) { println!("Baz's impl"); }
# }
#
fn main() {
    let b = Baz;
    b.f();
    <Baz as Foo>::f(&b);
    <Baz as Bar>::f(&b);
}
```

<span class="caption">代碼例 19-28：使用完全限定語法調用作為`Foo` 和 `Bar` trait 一部分的 `f` 方法</span>

這會打印出：

```
Baz's impl
Baz's impl of Foo
Baz's impl of Bar
```

只在存在歧義時才需要 `Type as` 部分，只有需要 `Type as` 時才需要 `<>` 部分。所以如果在作用域中只有定義於 `Baz` 和 `Baz` 上實現的 `Foo` trait 的 `f` 方法的話，則可以使用 `Foo::f(&b)` 調用 `Foo` 中的 `f` 方法，因為無需與 `Bar` trait 相區別。

也可以使用 `Baz::f(&b)` 調用直接定義於 `Baz` 上的 `f` 方法，不過因為這個定義是在調用 `b.f()` 時預設使用的，並不要求調用此方法時使用完全限定的名稱。

### 父 trait 用於在另一個 trait 中使用某 trait 的功能

有時我們希望當實現某 trait 時依賴另一個 trait 也被實現，如此這個 trait 就可以使用其他 trait 的功能。這個所需的 trait 是我們實現的 trait 的**父（超） trait**（*supertrait*）。

例如，加入我們希望創建一個帶有 `outline_print` 方法的 trait `OutlinePrint`，它會打印出帶有星號框的值。也就是說，如果 `Point` 實現了 `Display` 並返回 `(x, y)`，調用以 1 作為 `x` 和 3 作為 `y` 的 `Point` 實例的 `outline_print` 會顯示如下：

```
**********
*        *
* (1, 3) *
*        *
**********
```

在 `outline_print` 的實現中，因為希望能夠使用 `Display` trait 的功能，則需要說明 `OutlinePrint` 只能用於同時也實現了 `Display` 並提供了 `OutlinePrint` 需要的功能的類型。可以在 trait 定義中指定 `OutlinePrint: Display` 來做到這一點。這類似於為 trait 增加 trait bound。代碼例 19-29 展示了一個 `OutlinePrint` trait 的實現：

```rust
use std::fmt;

trait OutlinePrint: fmt::Display {
    fn outline_print(&self) {
        let output = self.to_string();
        let len = output.len();
        println!("{}", "*".repeat(len + 4));
        println!("*{}*", " ".repeat(len + 2));
        println!("* {} *", output);
        println!("*{}*", " ".repeat(len + 2));
        println!("{}", "*".repeat(len + 4));
    }
}
```

<span class="caption">代碼例 19-29：實現 `OutlinePrint` trait，它要求來自 `Display` 的功能</span>

因為指定了 `OutlinePrint` 需要 `Display` trait，則可以在 `outline_print` 中使用 `to_string`（`to_string` 會為任何實現 `Display` 的類型自動實現）。如果不在 trait 名後增加 `: Display` 並嘗試在 `outline_print` 中使用 `to_string`，則會得到一個錯誤說在當前作用域中沒有找到用於 `&Self` 類型的方法 `to_string`。

如果嘗試在一個沒有實現 `Display` 的類型上實現 `OutlinePrint`，比如 `Point` 結構體：

```rust
# trait OutlinePrint {}
struct Point {
    x: i32,
    y: i32,
}

impl OutlinePrint for Point {}
```

則會得到一個錯誤說 `Display` 沒有被實現而 `Display` 被 `OutlinePrint` 所需要：

```
error[E0277]: the trait bound `Point: std::fmt::Display` is not satisfied
  --> src/main.rs:20:6
   |
20 | impl OutlinePrint for Point {}
   |      ^^^^^^^^^^^^ the trait `std::fmt::Display` is not implemented for
   `Point`
   |
   = note: `Point` cannot be formatted with the default formatter; try using
   `:?` instead if you are using a format string
   = note: required by `OutlinePrint`
```

一旦在 `Point` 上實現 `Display` 並滿足 `OutlinePrint` 要求的限制，比如這樣：

```rust
# struct Point {
#     x: i32,
#     y: i32,
# }
#
use std::fmt;

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
```

那麼在 `Point` 實現 `OutlinePrint` trait 將能成功編譯並可以在 `Point` 實例上調用 `outline_print` 來顯示位於星號框中的點的值。

### newtype 模式用以在外部類型上實現外部 trait

在第十章中，我們提到了孤兒規則（orphan rule），它說明只要 trait 或類型對於當前 crate 是本地的話就可以在此類型上實現該 trait。一個繞開這個限制的方法是使用**newtype 模式**（*newtype pattern*），它涉及到使用一個元組結構體來創建一個新類型，它帶有一個字段作為希望實現 trait 的類型的簡單封裝。接著這個封裝類型對於 crate 是本地的，這樣就可以在這個封裝上實現 trait。「Newtype」 是一個源自 Haskell 編程語言的概念。使用這個模式沒有運行時性能懲罰。這個封裝類型在編譯時被省略了。

例如，如果想要在 `Vec` 上實現 `Display`，可以創建一個包含 `Vec` 實例的 `Wrapper` 結構體。接著可以如代碼例 19-30 那樣在 `Wrapper` 上實現 `Display` 並使用 `Vec` 的值：

<span class="filename">文件名: src/main.rs</span>

```rust
use std::fmt;

struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

fn main() {
    let w = Wrapper(vec![String::from("hello"), String::from("world")]);
    println!("w = {}", w);
}
```

<span class="caption">代碼例 19-30：創建 `Wrapper` 類型封裝 `Vec<String>` 以便實現 `Display`</span>

`Display` 的實現使用 `self.0` 來訪問其內部的 `Vec`，接著就可以使用 `Wrapper` 中 `Display` 的功能了。

此方法的缺點是因為 `Wrapper` 是一個新類型，它沒有定義於其值之上的方法；必須直接在 `Wrapper` 上實現 `Vec` 的所有方法，如 `push`、`pop` 等等，並代理到 `self.0` 上以便可以將 `Wrapper` 完全當作 `Vec` 處理。如果希望新類型擁有其內部類型的每一個方法，為封裝類型實現第十五章講到的 `Deref` trait 並返回其內部類型是一種解決方案。如果不希望封裝類型擁有所有內部類型的方法，比如為了限制封裝類型的行為，則必須自行實現所需的方法。

上面便是 newtype 模式如何與 trait 結合使用的；還有一個不涉及 trait 的實用模式。現在讓我們將話題的焦點轉移到一些與 Rust 類型系統互動的高級方法上來吧。