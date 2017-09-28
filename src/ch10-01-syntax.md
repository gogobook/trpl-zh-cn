## 泛型數據類型

> [ch10-01-syntax.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch10-01-syntax.md)
> <br>
> commit 56352c28cf3fe0402fa5a7cba73890e314d720eb

泛型用於通常我們放置類型的位置，比如函數簽名或結構體，允許我們創建可以代替許多具體數據類型的結構體定義。讓我們看看如何使用泛型定義函數、結構體、枚舉和方法，並且在本部分的結尾我們會討論泛型代碼的性能。

### 在函數定義中使用泛型

定義函數時可以在函數簽名的參數數據類型和返回值中使用泛型。以這種方式編寫的代碼將更靈活並能向函數調用者提供更多功能，同時不引入重複代碼。

回到 `largest` 函數上，示例 10-4 中展示了兩個提供了相同的尋找 slice 中最大值功能的函數。第一個是從示例 10-3 中提取的尋找 slice 中 `i32` 最大值的函數。第二個函數尋找 slice 中 `char` 的最大值：


<span class="filename">文件名: src/main.rs</span>

```rust
fn largest_i32(list: &[i32]) -> i32 {
    let mut largest = list[0];

    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn largest_char(list: &[char]) -> char {
    let mut largest = list[0];

    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest_i32(&number_list);
    println!("The largest number is {}", result);
#    assert_eq!(result, 100);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest_char(&char_list);
    println!("The largest char is {}", result);
#    assert_eq!(result, 'y');
}
```

<span class="caption">示例 10-4：兩個只在名稱和簽名中類型有所不同的函數</span>

這裡 `largest_i32` 和 `largest_char` 有著完全相同的函數體，所以如果能夠將這兩個函數變成一個來減少重複就太好了。所幸通過引入一個泛型參數就能實現！

為了參數化要定義的函數的簽名中的類型，我們需要像給函數的值參數起名那樣為這類型參數起一個名字。這裡選擇了名稱 `T`。任何標識符都可以作為類型參數名，選擇 `T` 是因為 Rust 的類型命名規範是大寫駱駝命名法（CamelCase）。另外泛型類型參數的規範也傾向於簡短，經常僅僅是一個字母。`T` 作為 「type」 的縮寫是大部分 Rust 程式設計師的首選。

當需要在函數體中使用一個參數時，必須在函數簽名中聲明這個參數以便編譯器能知道函數體中這個名稱的意義。同理，當在函數簽名中使用一個類型參數時，必須在使用它之前就聲明它。類型參數聲明位於函數名稱與參數示例中間的尖括號中。

我們將要定義的泛型版本的 `largest` 函數的簽名看起來像這樣：

```rust
fn largest<T>(list: &[T]) -> T {
```

這可以理解為：函數 `largest` 有泛型類型 `T`。它有一個參數 `list`，它的類型是一個 `T` 值的 slice。`largest` 函數將會返回一個與 `T` 相同類型的值。

示例 10-5 展示一個在簽名中使用了泛型的統一的 `largest` 函數定義，並向我們展示了如何對 `i32` 值的 slice 或 `char` 值的 slice 調用 `largest` 函數。注意這些代碼還不能編譯！

<span class="filename">文件名: src/main.rs</span>

```rust
fn largest<T>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
}
```

<span class="caption">示例 10-5：一個還不能編譯的使用泛型參數的 `largest` 函數定義</span>

如果現在就嘗試編譯這些代碼，會出現如下錯誤：

```text
error[E0369]: binary operation `>` cannot be applied to type `T`
  |
5 |         if item > largest {
  |            ^^^^
  |
note: an implementation of `std::cmp::PartialOrd` might be missing for `T`
```

註釋中提到了 `std::cmp::PartialOrd`，這是一個 *trait*。下一部分會講到 trait，不過簡單來說，這個錯誤表明 `largest` 的函數體不能適用於 `T` 的所有可能的類型；因為在函數體需要比較 `T` 類型的值，不過它只能用於我們知道如何排序的類型。標準庫中定義的 `std::cmp::PartialOrd` trait 可以實現類型的排序功能。在下一部分會再次回到 trait 並講解如何為泛型指定一個 trait，不過讓我們先把這個例子放在一邊並探索其他那些可以使用泛型類型參數的地方。

<!-- Liz: this is the reason we had the topics in the order we did in the first
draft of this chapter; it's hard to do anything interesting with generic types
in functions unless you also know about traits and trait bounds. I think this
ordering could work out okay, though, and keep a stronger thread with the
`longest` function going through the whole chapter, but we do pause with a
not-yet-compiling example here, which I know isn't ideal either. Let us know
what you think. /Carol -->

### 結構體定義中的泛型

同樣也可以使用 `<>` 語法來定義擁有一個或多個泛型參數類型字段的結構體。示例 10-6 展示了如何定義和使用一個可以存放任何類型的 `x` 和 `y` 坐標值的結構體 `Point`：

<span class="filename">文件名: src/main.rs</span>

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let integer = Point { x: 5, y: 10 };
    let float = Point { x: 1.0, y: 4.0 };
}
```

<span class="caption">示例 10-6：`Point` 結構體存放了兩個 `T` 類型的值 `x` 和 `y`</span>

其語法類似於函數定義中使用泛型。首先，必須在結構體名稱後面的尖括號中聲明泛型參數的名稱。接著在結構體定義中可以指定具體數據類型的位置使用泛型類型。

注意 `Point` 的定義中只使用了一個泛型類型，我們想要表達的是結構體 `Point` 對於一些類型 `T` 是泛型的，而且字段 `x` 和 `y` **都是** 相同類型的，無論它具體是何類型。如果嘗試創建一個有不同類型值的 `Point` 的實例，像示例 10-7 中的代碼就不能編譯：

<span class="filename">文件名: src/main.rs</span>

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let wont_work = Point { x: 5, y: 4.0 };
}
```

<span class="caption">示例 10-7：字段 `x` 和 `y` 必須是相同類型，因為他們都有相同的泛型類型 `T`</span>

嘗試編譯會得到如下錯誤：

```text
error[E0308]: mismatched types
 -->
  |
7 |     let wont_work = Point { x: 5, y: 4.0 };
  |                                      ^^^ expected integral variable, found
  floating-point variable
  |
  = note: expected type `{integer}`
  = note:    found type `{float}`
```

當我們將 5 賦值給 `x`，編譯器就知道這個 `Point` 實例的泛型類型 `T` 是一個整型。接著我們將 `y` 指定為 4.0，而它被定義為與 `x` 有著相同的類型，所以出現了類型不匹配的錯誤。

如果想要定義一個 `x` 和 `y` 可以有不同類型且仍然是泛型的 `Point` 結構體，我們可以使用多個泛型類型參數。在示例 10-8 中，我們修改 `Point` 的定義為擁有兩個泛型類型 `T` 和 `U`。其中字段 `x` 是 `T` 類型的，而字段 `y` 是 `U` 類型的：

<span class="filename">文件名: src/main.rs</span>

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

fn main() {
    let both_integer = Point { x: 5, y: 10 };
    let both_float = Point { x: 1.0, y: 4.0 };
    let integer_and_float = Point { x: 5, y: 4.0 };
}
```

<span class="caption">示例 10-8：使用兩個泛型的 `Point`，這樣 `x` 和 `y` 可能是不同類型</span>

現在所有這些 `Point` 實例都是被允許的了！你可以在定義中使用任意多的泛型類型參數，不過太多的話代碼將難以閱讀和理解。如果你處於一個需要很多泛型類型的位置，這可能是一個需要重新組織代碼並分隔成一些更小部分的信號。

### 枚舉定義中的泛型數據類型

類似於結構體，枚舉也可以在其成員中存放泛型數據類型。第六章我們使用過了標準庫提供的 `Option<T>` 枚舉，現在這個定義看起來就更容易理解了。讓我們再看看：

```rust
enum Option<T> {
    Some(T),
    None,
}
```

換句話說 `Option<T>` 是一個擁有泛型 `T` 的枚舉。它有兩個成員：`Some`，它存放了一個類型 `T` 的值，和不存在任何值的`None`。標準庫中只有這一個定義來支持創建任何具體類型的枚舉值。「一個可能的值」 是一個比具體類型的值更抽象的概念，而 Rust 允許我們不引入重複代碼就能表現抽象的概念。

枚舉也可以擁有多個泛型類型。第九章使用過的 `Result` 枚舉定義就是一個這樣的例子：

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`Result` 枚舉有兩個泛型類型，`T` 和 `E`。`Result` 有兩個成員：`Ok`，它存放一個類型 `T` 的值，而 `Err` 則存放一個類型 `E` 的值。這個定義使得 `Result` 枚舉能很方便的表達任何可能成功（返回 `T` 類型的值）也可能失敗（返回 `E` 類型的值）的操作。回憶一下示例 9-2 中打開一個文件的場景：當文件被成功打開 `T` 被放入了 `std::fs::File` 類型而當打開文件出現問題時 `E` 被放入了 `std::io::Error` 類型。

當發現代碼中有多個只有存放的值的類型有所不同的結構體或枚舉定義時，你就應該像之前的函數定義中那樣引入泛型類型來減少重複代碼。

### 方法定義中的枚舉數據類型

可以像第五章介紹的那樣來為其定義中帶有泛型的結構體或枚舉實現方法。示例 10-9 中展示了示例 10-6 中定義的結構體 `Point<T>`。接著我們在 `Point<T>` 上定義了一個叫做 `x` 的方法來返回字段 `x` 中數據的引用：

<span class="filename">文件名: src/main.rs</span>

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

fn main() {
    let p = Point { x: 5, y: 10 };

    println!("p.x = {}", p.x());
}
```

<span class="caption">示例 10-9：在 `Point<T>` 結構體上實現方法 `x`，它返回 `T` 類型的字段 `x` 的引用</span>

注意必須在 `impl` 後面聲明 `T`，這樣就可以在 `Point<T>` 上實現的方法中使用它了。在 `impl` 之後聲明泛型 `T` ，這樣 Rust 就知道 `Point` 的加括號中的類型是泛型而不是具體類型。例如，可以選擇為 `Point<f32>` 實例實現方法，而不是為泛型 `Point` 實例。示例 10-10 展示了一個沒有在 `impl` 之後（的尖括號）聲明泛型的例子，這裡使用了一個具體類型，`f32`：

```rust
# struct Point<T> {
#     x: T,
#     y: T,
# }
#
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

<span class="caption">示例 10-10：構建一個只用於擁有泛型參數 `T` 的結構體的具體類型的 `impl` 塊</span>

這段代碼意味著 `Point<f32>` 類型會有一個方法 `distance_from_origin`，而其他 `T` 不是 `f32` 類型的 `Point<T>` 實例則沒有定義此方法。這個方法計算點實例與另一個點坐標之間的距離，它使用了只能用於浮點型的數學運算符。

結構體定義中的泛型類型參數並不總是與結構體方法簽名中使用的泛型是同一類型。示例 10-11 中在示例 10-8 中的結構體 `Point<T, U>` 上定義了一個方法 `mixup`。這個方法抓取另一個 `Point` 作為參數，而它可能與調用 `mixup` 的 `self` 是不同的 `Point` 類型。這個方法用 `self` 的 `Point` 類型的 `x` 值（類型 `T`）和參數的 `Point` 類型的 `y` 值（類型 `W`）來創建一個新 `Point` 類型的實例：

<span class="filename">文件名: src/main.rs</span>

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 5, y: 10.4 };
    let p2 = Point { x: "Hello", y: 'c'};

    let p3 = p1.mixup(p2);

    println!("p3.x = {}, p3.y = {}", p3.x, p3.y);
}
```

<span class="caption">示例 10-11：方法使用了與結構體定義中不同類型的泛型</span>

在 `main` 函數中，定義了一個有 `i32` 類型的 `x`（其值為 `5`）和 `f64` 的 `y`（其值為 `10.4`）的 `Point`。`p2` 則是一個有著字符串 slice 類型的 `x`（其值為 `"Hello"`）和 `char` 類型的 `y`（其值為`c`）的 `Point`。在 `p1` 上以 `p2` 作為參數調用 `mixup` 會返回一個 `p3`，它會有一個 `i32` 類型的 `x`，因為 `x` 來自 `p1`，並擁有一個 `char` 類型的 `y`，因為 `y` 來自 `p2`。`println!` 會打印出 `p3.x = 5, p3.y = c`。

注意泛型參數 `T` 和 `U` 聲明於 `impl` 之後，因為他們與結構體定義相對應。而泛型參數 `V` 和 `W` 聲明於 `fn mixup` 之後，因為他們只是相對於方法本身的。

### 泛型代碼的性能

在閱讀本部分的內容的同時你可能會好奇使用泛型類型參數是否會有運行時消耗。好消息是：Rust 實現泛型泛型的方式意味著你的代碼使用泛型類型參數相比指定具體類型並沒有任何速度上的損失。

Rust 通過在編譯時進行泛型代碼的 **單態化**（*monomorphization*）來保證效率。單態化是一個將泛型代碼轉變為實際放入的具體類型的特定代碼的過程。

編譯器所做的工作正好與示例 10-5 中我們創建泛型函數的步驟相反。編譯器尋找所有泛型代碼被調用的位置並使用泛型代碼針對具體類型生成代碼。

讓我們看看一個使用標準庫中 `Option` 枚舉的例子：

```rust
let integer = Some(5);
let float = Some(5.0);
```

當 Rust 編譯這些代碼的時候，它會進行單態化。編譯器會讀取傳遞給 `Option` 的值並發現有兩種 `Option<T>`：一個對應 `i32` 另一個對應 `f64`。為此，它會將泛型定義 `Option<T>` 展開為 `Option_i32` 和 `Option_f64`，接著將泛型定義替換為這兩個具體的定義。

編譯器生成的單態化版本的代碼看起來像這樣，并包含將泛型 `Option` 替換為編譯器創建的具體定義後的用例代碼：

<span class="filename">文件名: src/main.rs</span>

```rust
enum Option_i32 {
    Some(i32),
    None,
}

enum Option_f64 {
    Some(f64),
    None,
}

fn main() {
    let integer = Option_i32::Some(5);
    let float = Option_f64::Some(5.0);
}
```

我們可以使用泛型來編寫不重複的代碼，而 Rust 將會為每一個實例編譯其特定類型的代碼。這意味著在使用泛型時沒有運行時開銷；當代碼運行，它的執行效率就跟好像手寫每個具體定義的重複代碼一樣。這個單態化過程正是 Rust 泛型在運行時極其高效的原因。