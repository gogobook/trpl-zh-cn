## trait：定義共享的行為

> [ch10-02-traits.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch10-02-traits.md)
> <br>
> commit 1cbcc277af6931d3091fe46a8f379fefae7202db

trait 允許我們進行另一種抽象：他們讓我們可以抽象類型所通用的行為。*trait* 告訴 Rust 編譯器某個特定類型擁有可能與其他類型共享的功能。在使用泛型類型參數的場景中，可以使用 *trait bounds* 在編譯時指定泛型可以是任何實現了某個 trait 的類型，並由此在這個場景下擁有我們希望的功能。

> 注意：*trait* 類似於其他語言中的常被稱為 **接口**（*interfaces*）的功能，雖然有一些不同。

### 定義 trait

一個類型的行為由其可供調用的方法構成。如果可以對不同類型調用相同的方法的話，這些類型就可以共享相同的行為了。trait 定義是一種將方法簽名組合起來的方法，目的是定義一個實現某些目的所必需的行為的集合。

例如，這裡有多個存放了不同類型和屬性文本的結構體：結構體 `NewsArticle` 用於存放發生於世界各地的新聞故事，而結構體 `Tweet` 最多只能存放 140 個字符的內容，以及像是否轉推或是否是對推友的回覆這樣的元數據。

我們想要創建一個多媒體聚合庫用來顯示可能儲存在 `NewsArticle` 或 `Tweet` 實例中的數據的總結。每一個結構體都需要的行為是他們是能夠被總結的，這樣的話就可以調用實例的 `summary` 方法來請求總結。代碼例 10-12 中展示了一個表現這個概念的 `Summarizable` trait 的定義：

<span class="filename">文件名: lib.rs</span>

```rust
pub trait Summarizable {
    fn summary(&self) -> String;
}
```

<span class="caption">代碼例 10-12：`Summarizable` trait 定義，它包含由 `summary` 方法提供的行為</span>

使用 `trait` 關鍵字來聲明一個 trait，後面是 trait 的名字，在這個例子中是 `Summarizable`。在大括號中聲明描述實現這個 trait 的類型所需要的行為的方法簽名，在這個例子中是是 `fn summary(&self) -> String`。在方法簽名後跟分號，而不是在大括號中提供其實現。接著每一個實現這個 trait 的類型都需要提供其自定義行為的方法體，編譯器也會確保任何實現 `Summarizable` trait 的類型都擁有與這個簽名的定義完全一致的 `summary` 方法。

trait 體中可以有多個方法，一行一個方法簽名且都以分號結尾。

### 為類型實現 trait

現在我們定義了 `Summarizable` trait，接著就可以在多媒體聚合庫中需要擁有這個行為的類型上實現它了。代碼例 10-12 中展示了 `NewsArticle` 結構體上 `Summarizable` trait 的一個實現，它使用標題、作者和創建的位置作為 `summary` 的返回值。對於 `Tweet` 結構體，我們選擇將 `summary` 定義為用戶名後跟推文的全部文本作為返回值，並假設推文內容已經被限制為 140 字符以內。

<span class="filename">文件名: lib.rs</span>

```rust
# pub trait Summarizable {
#     fn summary(&self) -> String;
# }
#
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summarizable for NewsArticle {
    fn summary(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summarizable for Tweet {
    fn summary(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

<span class="caption">代碼例 10-13：在 `NewsArticle` 和 `Tweet` 類型上實現 `Summarizable` trait</span>

在類型上實現 trait 類似與實現與 trait 無關的方法。區別在於 `impl` 關鍵字之後，我們提供需要實現 trait 的名稱，接著是 `for` 和需要實現 trait 的類型的名稱。在 `impl` 塊中，使用 trait 定義中的方法簽名，不過不再後跟分號，而是需要在大括號中編寫函數體來為特定類型實現 trait 方法所擁有的行為。

一旦實現了 trait，我們就可以用與 `NewsArticle` 和 `Tweet` 實例的非 trait 方法一樣的方式調用 trait 方法了：

```rust
let tweet = Tweet {
    username: String::from("horse_ebooks"),
    content: String::from("of course, as you probably already know, people"),
    reply: false,
    retweet: false,
};

println!("1 new tweet: {}", tweet.summary());
```

這會打印出 `1 new tweet: horse_ebooks: of course, as you probably already know, people`。

注意因為代碼例 10-12 中我們在相同的 `lib.rs` 裡定義了 `Summarizable` trait 和 `NewsArticle` 與 `Tweet` 類型，所以他們是位於同一作用域的。如果這個 `lib.rs` 是對應 `aggregator` crate 的，而別人想要利用我們 crate 的功能外加為其 `WeatherForecast` 結構體實現 `Summarizable` trait，在實現 `Summarizable` trait 之前他們首先就需要將其導入其作用域中，如代碼例 10-14 所示：

<span class="filename">文件名: lib.rs</span>

```rust
extern crate aggregator;

use aggregator::Summarizable;

struct WeatherForecast {
    high_temp: f64,
    low_temp: f64,
    chance_of_precipitation: f64,
}

impl Summarizable for WeatherForecast {
    fn summary(&self) -> String {
        format!("The high will be {}, and the low will be {}. The chance of
        precipitation is {}%.", self.high_temp, self.low_temp,
        self.chance_of_precipitation)
    }
}
```

<span class="caption">代碼例 10-14：在另一個 crate 中將 `aggregator` crate 的 `Summarizable` trait 引入作用域</span>

另外這段代碼假設 `Summarizable` 是一個公有 trait，這是因為代碼例 10-12 中 `trait` 之前使用了 `pub` 關鍵字。

trait 實現的一個需要注意的限制是：只能在 trait 或對應類型位於我們 crate 本地的時候為其實現 trait。換句話說，不允許對外部類型實現外部 trait。例如，不能在 `Vec` 上實現 `Display` trait，因為 `Display` 和 `Vec` 都定義於標準庫中。允許在像 `Tweet` 這樣作為我們 `aggregator`crate 部分功能的自定義類型上實現標準庫中的 trait `Display`。也允許在 `aggregator`crate 中為 `Vec` 實現 `Summarizable`，因為 `Summarizable` 定義與此。這個限制是我們稱為 **孤兒規則**（*orphan rule*）的一部分，如果你感興趣的可以在類型理論中找到它。簡單來說，它被稱為 orphan rule 是因為其父類型不存在。沒有這條規則的話，兩個 crate 可以分別對相同類型是實現相同的 trait，因而這兩個實現會相互衝突：Rust 將無從得知應該使用哪一個。因為 Rust 強制執行 orphan rule，其他人編寫的代碼不會破壞你代碼，反之亦是如此。

### 預設實現

有時為 trait 中的某些或全部方法提供預設的行為，而不是在每個類型的每個實現中都定義自己的行為是很有用的。這樣當為某個特定類型實現 trait 時，可以選擇保留或重載每個方法的預設行為。

代碼例 10-15 中展示了如何為 `Summarize` trait 的 `summary` 方法指定一個預設的字符串值，而不是像代碼例 10-12 中那樣只是定義方法簽名：

<span class="filename">文件名: lib.rs</span>

```rust
pub trait Summarizable {
    fn summary(&self) -> String {
        String::from("(Read more...)")
    }
}
```

<span class="caption">代碼例 10-15：`Summarizable` trait 的定義，帶有一個 `summary` 方法的預設實現</span>

如果想要對 `NewsArticle` 實例使用這個預設實現，而不是像代碼例 10-13 中那樣定義一個自己的實現，則可以指定一個空的 `impl` 塊：

```rust
impl Summarizable for NewsArticle {}
```

即便選擇不再直接為 `NewsArticle` 定義 `summary` 方法了，因為 `summary` 方法有一個預設實現而且 `NewsArticle` 被指定為實現了 `Summarizable` trait，我們仍然可以對 `NewsArticle` 的實例調用 `summary` 方法：

```rust
let article = NewsArticle {
    headline: String::from("Penguins win the Stanley Cup Championship!"),
    location: String::from("Pittsburgh, PA, USA"),
    author: String::from("Iceburgh"),
    content: String::from("The Pittsburgh Penguins once again are the best
    hockey team in the NHL."),
};

println!("New article available! {}", article.summary());
```

這段代碼會打印 `New article available! (Read more...)`。

將 `Summarizable` trait 改變為擁有預設 `summary` 實現並不要求對代碼例 10-13 中 `Tweet` 和代碼例 10-14 中 `WeatherForecast` 的 `Summarizable` 實現做任何改變：重載一個預設實現的語法與實現沒有預設實現的 trait 方法時完全一樣的。

預設實現允許調用相同 trait 中的其他方法，哪怕這些方法沒有預設實現。通過這種方法，trait 可以實現很多有用的功能而只需實現一小部分特定內容。我們可以選擇讓`Summarizable` trait 也擁有一個要求實現的`author_summary` 方法，接著 `summary` 方法則提供預設實現並調用 `author_summary` 方法：

```rust
pub trait Summarizable {
    fn author_summary(&self) -> String;

    fn summary(&self) -> String {
        format!("(Read more from {}...)", self.author_summary())
    }
}
```

為了使用這個版本的 `Summarizable`，只需在實現 trait 時定義 `author_summary` 即可：

```rust
impl Summarizable for Tweet {
    fn author_summary(&self) -> String {
        format!("@{}", self.username)
    }
}
```

一旦定義了 `author_summary`，我們就可以對 `Tweet` 結構體的實例調用 `summary` 了，而 `summary` 的預設實現會調用我們提供的 `author_summary` 定義。

```rust
let tweet = Tweet {
    username: String::from("horse_ebooks"),
    content: String::from("of course, as you probably already know, people"),
    reply: false,
    retweet: false,
};

println!("1 new tweet: {}", tweet.summary());
```

這會打印出 `1 new tweet: (Read more from @horse_ebooks...)`。

注意在重載過的實現中調用預設實現是不可能的。

### trait bounds

現在我們定義了 trait 並在類型上實現了這些 trait，也可以對泛型類型參數使用 trait。我們可以限制泛型不再適用於任何類型，編譯器會確保其被限制為那些實現了特定 trait 的類型，由此泛型就會擁有我們希望其類型所擁有的功能。這被稱為指定泛型的 *trait bounds*。

例如在代碼例 10-13 中為 `NewsArticle` 和 `Tweet` 類型實現了 `Summarizable` trait。我們可以定義一個函數 `notify` 來調用 `summary` 方法，它擁有一個泛型類型 `T` 的參數 `item`。為了能夠在 `item` 上調用 `summary` 而不出現錯誤，我們可以在 `T` 上使用 trait bounds 來指定 `item` 必須是實現了 `Summarizable` trait 的類型：

```rust
pub fn notify<T: Summarizable>(item: T) {
    println!("Breaking news! {}", item.summary());
}
```

trait bounds 連同泛型類型參數聲明一同出現，位於尖括號中的冒號後面。由於 `T` 上的 trait bounds，我們可以傳遞任何 `NewsArticle` 或 `Tweet` 的實例來調用 `notify` 函數。代碼例 10-14 中使用我們 `aggregator` crate 的外部代碼也可以傳遞一個 `WeatherForecast` 的實例來調用 `notify` 函數，因為 `WeatherForecast` 同樣也實現了 `Summarizable`。使用任何其他類型，比如 `String` 或 `i32`，來調用 `notify` 的代碼將不能編譯，因為這些類型沒有實現 `Summarizable`。

可以通過 `+` 來為泛型指定多個 trait bounds。如果我們需要能夠在函數中使用 `T` 類型的顯示格式的同時也能使用 `summary` 方法，則可以使用 trait bounds `T: Summarizable + Display`。這意味著 `T` 可以是任何實現了 `Summarizable` 和 `Display` 的類型。

對於擁有多個泛型類型參數的函數，每一個泛型都可以有其自己的 trait bounds。在函數名和參數代碼例之間的尖括號中指定很多的 trait bound 信息將是難以閱讀的，所以有另外一個指定 trait bounds 的語法，它將其移動到函數簽名後的 `where` 從句中。所以相比這樣寫：

```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: T, u: U) -> i32 {
```

我們也可以使用 `where` 從句：

```rust
fn some_function<T, U>(t: T, u: U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{
```

這就顯得不那麼雜亂，同時也使這個函數看起來更像沒有很多 trait bounds 的函數。這時函數名、參數代碼例和返回值類型都離得很近。

### 使用 trait bounds 來修復 `largest` 函數

所以任何想要對泛型使用 trait 定義的行為的時候，都需要在泛型參數類型上指定 trait bounds。現在我們就可以修複代碼例 10-5 中那個使用泛型類型參數的 `largest` 函數定義了！當我們將其放置不管的時候，它會出現這個錯誤：

```text
error[E0369]: binary operation `>` cannot be applied to type `T`
  |
5 |         if item > largest {
  |            ^^^^
  |
note: an implementation of `std::cmp::PartialOrd` might be missing for `T`
```

在 `largest` 函數體中我們想要使用大於運算符比較兩個 `T` 類型的值。這個運算符被定義為標準庫中 trait `std::cmp::PartialOrd` 的一個預設方法。所以為了能夠使用大於運算符，需要在 `T` 的 trait bounds 中指定 `PartialOrd`，這樣 `largest` 函數可以用於任何可以比較大小的類型的 slice。因為 `PartialOrd` 位於 prelude 中所以並不需要手動將其引入作用域。

```rust
fn largest<T: PartialOrd>(list: &[T]) -> T {
```

但是如果編譯代碼的話，會出現不同的錯誤：

```text
error[E0508]: cannot move out of type `[T]`, a non-copy array
 --> src/main.rs:4:23
  |
4 |     let mut largest = list[0];
  |         -----------   ^^^^^^^ cannot move out of here
  |         |
  |         hint: to prevent move, use `ref largest` or `ref mut largest`

error[E0507]: cannot move out of borrowed content
 --> src/main.rs:6:9
  |
6 |     for &item in list.iter() {
  |         ^----
  |         ||
  |         |hint: to prevent move, use `ref item` or `ref mut item`
  |         cannot move out of borrowed content
```

錯誤的核心是 `cannot move out of type [T], a non-copy array`，對於非泛型版本的 `largest` 函數，我們只嘗試了尋找最大的 `i32` 和 `char`。正如第四章討論過的，像 `i32` 和 `char` 這樣的類型是已知大小的並可以儲存在棧上，所以他們實現了 `Copy` trait。當我們將 `largest` 函數改成使用泛型後，現在 `list` 參數的類型就有可能是沒有實現 `Copy` trait 的，這意味著我們可能不能將 `list[0]` 的值移動到 `largest` 變數中。

如果只想對實現了 `Copy` 的類型調用這些代碼，可以在 `T` 的 trait bounds 中增加 `Copy`！代碼例 10-16 中展示了一個可以編譯的泛型版本的 `largest` 函數的完整代碼，只要傳遞給 `largest` 的 slice 值的類型實現了 `PartialOrd` 和 `Copy` 這兩個 trait，例如 `i32` 和 `char`：

<span class="filename">文件名: src/main.rs</span>

```rust
use std::cmp::PartialOrd;

fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
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

<span class="caption">代碼例 10-16：一個可以用於任何實現了 `PartialOrd` 和 `Copy` trait 的泛型的 `largest` 函數</span>

如果並不希望限制 `largest` 函數只能用於實現了 `Copy` trait 的類型，我們可以在 `T` 的 trait bounds 中指定 `Clone` 而不是 `Copy`，並克隆 slice 的每一個值使得 `largest` 函數擁有其所有權。但是使用 `clone` 函數潛在意味著更多的堆分配，而且堆分配在涉及大量數據時可能會相當緩慢。另一種 `largest` 的實現方式是返回 slice 中一個 `T` 值的引用。如果我們將函數返回值從 `T` 改為 `&T` 並改變函數體使其能夠返回一個引用，我們將不需要任何 `Clone` 或 `Copy` 的 trait bounds 而且也不會有任何的堆分配。嘗試自己實現這種替代解決方式吧！

### 使用 trait bound 有條件的實現方法

通過使用帶有 trati bound 的泛型 `impl` 塊，可以有條件的只為實現了特定 trait 的類型實現方法。例如，代碼例 10-17 中的類型 `Pair<T>` 總是實現了 `new` 方法，不過只有 `Pair<T>` 內部的 `T` 實現了 `PartialOrd` trait 來允許比較和 `Display` trait 來啟用打印，才會實現 `cmp_display`：

```rust
use std::fmt::Display;

struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self {
            x,
            y,
        }
    }
}

impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {}", self.x);
        } else {
            println!("The largest member is y = {}", self.y);
        }
    }
}
```

<span class="caption">代碼例 10-17：根據 trait bound 在泛型上有條件的實現方法</span>

也可以對任何實現了特定 trait 的類型有條件的實現 trait。對任何滿足特定 trait bound 的類型實現 trait 被稱為 *blanket implementations*，他們被廣泛的用於 Rust 標準庫中。例如，標準庫為任何實現了 `Display` trait 的類型實現了 `ToString` trait。這個 `impl` 塊看起來像這樣：

```rust
impl<T: Display> ToString for T {
    // ...snip...
}
```

因為標準庫有了這些 blanket implementation，我們可以對任何實現了 `Display` trait 的類型調用由 `ToString` 定義的 `to_string` 方法。例如，可以將整型轉換為對應的 `String` 值，因為整型實現了 `Display`：

```rust
let s = 3.to_string();
```

blanket implementation 會出現在 trait 文檔的 「Implementers」 部分。

trait 和 trait bound 讓我們使用泛型類型參數來減少重複，並仍然能夠向編譯器明確指定泛型類型需要擁有哪些行為。因為我們向編譯器提供了 trait bound 信息，它就可以檢查代碼中所用到的具體類型是否提供了正確的行為。在動態類型語言中，如果我們嘗試調用一個類型並沒有實現的方法，會在運行時出現錯誤。Rust 將這些錯誤移動到了編譯時，甚至在代碼能夠運行之前就強迫我們修復錯誤。另外，我們也無需編寫運行時檢查行為的代碼，因為在編譯時就已經檢查過了，這樣相比其他那些不願放棄泛型靈活性的語言有更好的性能。

這裡還有一種泛型，我們一直在使用它甚至都沒有察覺它的存在，這就是 **生命週期**（*lifetimes*）。不同於其他泛型幫助我們確保類型擁有期望的行為，生命週期則有助於確保引用在我們需要他們的時候一直有效。讓我們學習生命週期是如何做到這些的。