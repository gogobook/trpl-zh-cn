## 面向物件設計模式的實現

> [ch17-03-oo-design-patterns.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch17-03-oo-design-patterns.md)
> <br>
> commit 67737ff868e3347588cc832eceb8fc237afc5895

讓我們看看一個狀態設計模式的例子以及如何在 Rust 中使用他們。**狀態模式**（*state pattern*）是指一個值有某些內部狀態，而它的行為隨著其內部狀態而改變。內部狀態由一系列繼承了共享功能的物件表現（我們使用結構體和 trait 因為 Rust 沒有物件和繼承）。每一個狀態物件負責它自身的行為和當需要改變為另一個狀態時的規則。持有任何一個這種狀態物件的值對於不同狀態的行為以及何時狀態轉移毫不知情。當將來需求改變時，無需改變值持有狀態或者使用值的代碼。我們只需更新某個狀態物件中的代碼來改變它的規則，或者是增加更多的狀態物件。

為了探索這個概念，我們將實現一個增量式的發布博文的工作流。這個我們希望發佈博文時所應遵守的工作流，一旦完成了它的實現，將為如下：

1. 博文從空白的草案開始。
2. 一旦草案完成，請求審核博文。
3. 一旦博文過審，它將被發表。
4. 只有被發表的博文的內容會被打印，這樣就不會意外打印出沒有被審核的博文的文本。

任何其他對博文的修改嘗試都是沒有作用的。例如，如果嘗試在請求審核之前通過一個草案博文，博文應該保持未發佈的狀態。

列表 17-11 展示這個工作流的代碼形式。這是一個我們將要在一個叫做 `blog` 的庫 crate 中實現的 API 的使用示例：

<span class="filename">文件名: src/main.rs</span>

```rust
extern crate blog;
use blog::Post;

fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");
    assert_eq!("", post.content());

    post.request_review();
    assert_eq!("", post.content());

    post.approve();
    assert_eq!("I ate a salad for lunch today", post.content());
}
```

<span class="caption">列表 17-11: 展示了 `blog` crate 期望行為的代碼</span>

我們希望能夠使用 `Post::new` 創建一個新的博文草案。接著希望能在草案階段為博文編寫一些文本。如果嘗試立即打印出博文的內容，將不會得到任何文本，因為博文仍然是草案。這裡增加的 `assert_eq!` 用於展示目的。斷言草案博文的 `content` 方法返回空字符串將能作為庫的一個非常好的單元測試，不過我們並不準備為這個例子編寫單元測試。

接下來，我們希望能夠請求審核博文，而在等待審核的階段 `content` 應該仍然返回空字符串，當博文審核通過，它應該被發表，這意味著當調用 `content` 時我們編寫的文本將被返回。

注意我們與 crate 交互的唯一的類型是 `Post`。博文可能處於的多種狀態（草案，等待審核和發佈）由 `Post` 內部管理。博文狀態依我們在`Post`調用的方法而改變，但不必直接管理狀態改變。這也意味著不會在狀態上犯錯，比如忘記了在發佈前請求審核。

### 定義 `Post` 並新建一個草案狀態的實例

讓我們開始實現這個庫吧！我們知道需要一個公有 `Post` 結構體來存放一些文本，所以讓我們從結構體的定義和一個創建 `Post` 實例的公有關聯函數 `new` 開始，如列表 17-12 所示。我們還需定義一個私有 trait `State`。`Post` 將在私有字段 `state` 中存放一個 `Option` 中的 trait 物件 `Box<State>`。稍後將會看到為何 `Option` 是必須的。`State` trait 定義了所有不同狀態的博文所共享的行為，同時 `Draft`、`PendingReview` 和 `Published` 狀態都會實現`State` 狀態。現在這個 trait 並沒有任何方法，同時開始將只定義`Draft`狀態因為這是我們希望開始的狀態：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub struct Post {
    state: Option<Box<State>>,
    content: String,
}

impl Post {
    pub fn new() -> Post {
        Post {
            state: Some(Box::new(Draft {})),
            content: String::new(),
        }
    }
}

trait State {}

struct Draft {}

impl State for Draft {}
```

<span class="caption">列表 17-12: `Post`結構體的定義和新建 `Post` 實例的 `new`函數，`State` trait 和實現了 `State` 的結構體 `Draft`</span>

當創建新的 `Post` 時，我們將其 `state` 字段設置為一個 `Some` 值，它存放了指向一個 `Draft` 結構體新實例的 `Box`。這確保了無論何時新建一個 `Post` 實例，它會從草案開始。因為 `Post` 的 `state` 字段是私有的，也就無法創建任何其他狀態的 `Post` 了！。

### 存放博文內容的文本

在 `Post::new` 函數中，我們設置 `content` 字段為新的空 `String`。在列表 17-11 中，展示了我們希望能夠調用一個叫做 `add_text` 的方法並向其傳遞一個 `&str` 來將文本增加到博文的內容中。選擇實現為一個方法而不是將 `content` 字段暴露為 `pub` 是因為我們希望能夠通過之後實現的一個方法來控制 `content` 字段如何被讀取。`add_text` 方法是非常直觀的，讓我們在列表 17-13 的 `impl Post` 塊中增加一個實現：


<span class="filename">文件名: src/lib.rs</span>

```rust
# pub struct Post {
#     content: String,
# }
#
impl Post {
    // ...snip...
    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }
}
```

<span class="caption">列表 17-13: 實現方法 `add_text` 來向博文的 `content` 增加文本</span>

`add_text` 抓取一個 `self` 的可變引用，因為需要改變調用 `add_text` 的 `Post`。接著調用 `content` 中的 `String` 的 `push_str` 並傳遞 `text` 參數來保存到 `content` 中。這不是狀態模式的一部分，因為它的行為並不依賴博文所處的狀態。`add_text` 方法完全不與 `state` 狀態交互，不過這是我們希望支持的行為的一部分。

### 博文草案的內容是空的

調用 `add_text` 並像博文增加一些內容之後，我們仍然希望 `content` 方法返回一個空字符串 slice，因為博文仍然處於草案狀態，如列表 17-11 的第 8 行所示。現在讓我們使用能滿足要求的最簡單的方式來實現 `content` 方法 總是返回一個空字符 slice。當實現了將博文狀態改為發佈的能力之後將改變這一做法。但是現在博文只能是草案狀態，這意味著其內容總是空的。列表 17-14 展示了這個佔位符實現：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub struct Post {
#     content: String,
# }
#
impl Post {
    // ...snip...
    pub fn content(&self) -> &str {
        ""
    }
}
```

<span class="caption">列表 17-14: 增加一個 `Post` 的 `content` 方法的佔位實現，它總是返回一個空字符串 slice</span>

通過增加這個 `content`方法，列表 17-11 中直到第 8 行的代碼能如期運行。

### 請求審核博文來改變其狀態

接下來是請求審核博文，這應當將其狀態由 `Draft` 改為 `PendingReview`。我們希望 `post` 有一個抓取 `self` 可變引用的公有方法 `request_review`。接著將調用內部存放的狀態的 `request_review` 方法，而這第二個 `request_review` 方法會消費當前的狀態並返回要一個狀態。為了能夠消費舊狀態，第二個 `request_review` 方法需要能夠抓取狀態值的所有權。這就是 `Option` 的作用：我們將 `take` 字段 `state` 中的 `Some` 值並留下一個 `None` 值，因為 Rust 並不允許結構體中有空字段。接著將博文的 `state` 設置為這個操作的結果。列表 17-15 展示了這些代碼：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub struct Post {
#     state: Option<Box<State>>,
#     content: String,
# }
#
impl Post {
    // ...snip...
    pub fn request_review(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.request_review())
        }
    }
}

trait State {
    fn request_review(self: Box<Self>) -> Box<State>;
}

struct Draft {}

impl State for Draft {
    fn request_review(self: Box<Self>) -> Box<State> {
        Box::new(PendingReview {})
    }
}

struct PendingReview {}

impl State for PendingReview {
    fn request_review(self: Box<Self>) -> Box<State> {
        self
    }
}
```

<span class="caption">列表 17-15: 實現 `Post` 和 `State` trait 的 `request_review` 方法</span>

這裡給 `State` trait 增加了 `request_review` 方法；所有實現了這個 trait 的類型現在都需要實現 `request_review` 方法。注意不用於使用`self`、 `&self` 或者 `&mut self` 作為方法的第一個參數，這裡使用了 `self: Box<Self>`。這個語法意味著這個方法調用只對這個類型的 `Box` 有效。這個語法抓取了 `Box<Self>` 的所有權，這是我們希望的，因為需要從老狀態轉換為新狀態，同時希望老狀態不再有效。

`Draft` 的方法 `request_review` 的實現返回一個新的，裝箱的 `PendingReview` 結構體的實例，這是新引入的用來代表博文處於等待審核狀態的類型。結構體 `PendingReview` 同樣也實現了 `request_review` 方法，不過它不進行任何狀態轉換。它返回自身，因為請求審核已經處於 `PendingReview` 狀態的博文應該保持 `PendingReview` 狀態。

現在能夠看出狀態模式的優勢了：`Post` 的 `request_review` 方法無論 `state` 是何值都是一樣的。每個狀態負責它自己的規則。

我們將繼續保持 `Post` 的 `content` 方法不變，返回一個空字符串 slice。現在可以擁有 `PendingReview` 狀態而不僅僅是 `Draft` 狀態的 `Post` 了，不過我們希望在 `PendingReview` 狀態下其也有相同的行為。現在列表 17-11 中直到 11 行的代碼是可以執行的！

### 批准博文並改變 `content` 的行為

`Post` 的 `approve` 方法將與 `request_review` 方法類似：它會將 `state` 設置為審核通過時應處於的狀態。我們需要為 `State` trait 增加 `approve` 方法，並需新增實現了 `State` 的結構體， `Published` 狀態。列表 17-16 展示了新增的代碼：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub struct Post {
#     state: Option<Box<State>>,
#     content: String,
# }
#
impl Post {
    // ...snip...
    pub fn approve(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.approve())
        }
    }
}

trait State {
    fn request_review(self: Box<Self>) -> Box<State>;
    fn approve(self: Box<Self>) -> Box<State>;
}

struct Draft {}

impl State for Draft {
#     fn request_review(self: Box<Self>) -> Box<State> {
#         Box::new(PendingReview {})
#     }
#
    // ...snip...
    fn approve(self: Box<Self>) -> Box<State> {
        self
    }
}

struct PendingReview {}

impl State for PendingReview {
#     fn request_review(self: Box<Self>) -> Box<State> {
#         Box::new(PendingReview {})
#     }
#
    // ...snip...
    fn approve(self: Box<Self>) -> Box<State> {
        Box::new(Published {})
    }
}

struct Published {}

impl State for Published {
    fn request_review(self: Box<Self>) -> Box<State> {
        self
    }

    fn approve(self: Box<Self>) -> Box<State> {
        self
    }
}
```

<span class="caption">列表 17-16: 為 `Post` 和 `State` trait 實現 `approve` 方法</span>

類似於 `request_review`，如果對 `Draft` 調用 `approve` 方法，並沒有任何效果，因為它會返回 `self`。當對 `PendingReview` 調用 `approve` 時，它返回一個新的、裝箱的 `Published` 結構體的實例。`Published` 結構體實現了 `State` trait，同時對於 `request_review` 和 `approve` 方法來說，它返回自身，因為在這兩種情況博文應該保持 `Published` 狀態。

現在更新 `Post` 的 `content` 方法：我們希望當博文處於 `Published` 時返回 `content` 字段的值，否則返回空字符串 slice。因為目標是將所有像這樣的規則保持在實現了 `State` 的結構體中，我們將調用 `state` 中的值的 `content` 方法並傳遞博文實例（也就是 `self`）作為參數。接著返回 `state` 值的 `content` 方法的返回值，如列表 17-17 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
# trait State {
#     fn content<'a>(&self, post: &'a Post) -> &'a str;
# }
# pub struct Post {
#     state: Option<Box<State>>,
#     content: String,
# }
#
impl Post {
    // ...snip...
    pub fn content(&self) -> &str {
        self.state.as_ref().unwrap().content(&self)
    }
    // ...snip...
}
```

<span class="caption">列表 17-17: 更新 `Post` 的 `content` 方法來委託調用 `State` 的`content` 方法</span>

這裡調用 `Option` 的 `as_ref`方法是因為需要 `Option` 中值的引用。接著調用 `unwrap` 方法，這裡我們知道永遠也不會 panic 因為 `Post` 的所有方法都確保在他們返回時 `state` 會有一個 `Some` 值。這就是一個第十二章討論過的我們知道 `None` 是不可能的而編譯器卻不能理解的情況。

`State` trait 的 `content` 方法是博文返回什麼內容的邏輯所在之處。我們將增加一個 `content` 方法的默認實現來返回一個空字符串 slice。這樣就無需為 `Draft` 和 `PendingReview` 結構體實現 `content` 了。`Published` 結構體會覆蓋 `content` 方法並會返回 `post.content` 的值，如列表 17-18 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub struct Post {
#     content: String
# }
trait State {
    // ...snip...
    fn content<'a>(&self, post: &'a Post) -> &'a str {
        ""
    }
}

// ...snip...
struct Published {}

impl State for Published {
    // ...snip...
    fn content<'a>(&self, post: &'a Post) -> &'a str {
        &post.content
    }
}
```

<span class="caption">列表 17-18: 為 `State` trait 增加 `content` 方法</span>

注意這個方法需要生命週期註解，如第十章所討論的。這裡抓取 `post` 的引用作為參數，並返回 `post` 一部分的引用，所以返回的引用的生命週期與 `post` 參數相關。

### 狀態模式的權衡取捨

我們展示了 Rust 是能夠實現面向物件的狀態模式的，以便能根據博文所處的狀態來封裝不同類型的行為。`Post` 的方法並不知道這些不同類型的行為。這種組織代碼的方式，為了找到所有已發佈的博文不同行為只需查看一處代碼：`Published` 的 `State` trait 的實現。

一個不使用狀態模式的替代實現可能會在 `Post` 的方法中，甚至於在使用 `Post` 的代碼中（在這裡是 `main` 中）用到 `match` 語句，來檢查博文狀態並在這裡改變其行為。這可能意味著需要查看很多位置來理解處於發佈狀態的博文的所有邏輯！這在增加更多狀態時會變得更糟：每一個 `match` 語句都會需要另一個分支。對於狀態模式來說，`Post` 的方法和使用 `Post` 的位置無需`match` 語句，同時增加新狀態只涉及到增加一個新 `struct` 和為其實現 trait 的方法。

這個實現易於增加更多功能。這裡是一些你可以嘗試對本部分代碼做出的修改，來親自體會一下使用狀態模式隨著時間的推移維護代碼是什麼感覺：

- 只允許博文處於 `Draft` 狀態時增加文本內容
- 增加 `reject` 方法將博文的狀態從 `PendingReview` 變回 `Draft`
- 在將狀態變為 `Published` 之前需要兩次 `approve` 調用

狀態模式的一個缺點是因為狀態實現了狀態之間的轉換，一些狀態會相互聯繫。如果在 `PendingReview` 和 `Published` 之間增加另一個狀態，比如 `Scheduled`，則不得不修改 `PendingReview` 中的代碼來轉移到 `Scheduled`。如果 `PendingReview` 無需因為新增的狀態而改變就更好了，不過這意味著切換到另一個設計模式。

這個 Rust 中的實現的缺點在於存在一些重複的邏輯。如果能夠為 `State` trait 中返回 `self` 的 `request_review` 和 `approve` 方法增加默認實現就好了，不過這會違反物件安全性，因為 trait 不知道 `self` 具體是什麼。我們希望能夠將 `State` 作為一個 trait 物件，所以需要這個方法是物件安全的。

另一個最好能去除的重複是 `Post` 中 `request_review` 和 `approve` 這兩個類似的實現。他們都委託調用了 `state` 字段中 `Option` 值的同一方法，並在結果中為 `state` 字段設置了新值。如果 `Post` 中的很多方法都遵循這個模式，我們可能會考慮定義一個巨集來消除重複（查看附錄 E 以瞭解巨集）。

這個完全按照面向物件語言的定義實現的面向物件模式的缺點在於沒有儘可能的利用 Rust 的優勢。讓我們看看一些代碼中可以做出的修改，來將無效的狀態和狀態轉移變為編譯時錯誤。

#### 將狀態和行為編碼為類型

我們將展示如何稍微反思狀態模式來進行一系列不同的權衡取捨。不同於完全封裝狀態和狀態轉移使得外部代碼對其毫不知情，我們將將狀態編碼進不同的類型。當狀態是類型時，Rust 的類型檢查就會使任何在只能使用發佈的博文的地方使用草案博文的嘗試變為編譯時錯誤。

讓我們考慮一下列表 17-11 中 `main` 的第一部分：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");
    assert_eq!("", post.content());
}
```

我們仍然希望使用 `Post::new` 創建一個新的草案博文，並仍然希望能夠增加博文的內容。不過不同於存在一個草案博文時返回空字符串的 `content` 方法，我們將使草案博文完全沒有 `content` 方法。這樣如果嘗試抓取草案博文的內容，將會得到一個方法不存在的編譯錯誤。這使得我們不可能在生產環境意外顯示出草案博文的內容，因為這樣的代碼甚至就不能編譯。列表 17-19 展示了 `Post` 結構體、`DraftPost` 結構體以及各自的方法的定義：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub struct Post {
    content: String,
}

pub struct DraftPost {
    content: String,
}

impl Post {
    pub fn new() -> DraftPost {
        DraftPost {
            content: String::new(),
        }
    }

    pub fn content(&self) -> &str {
       &self.content
    }
}

impl DraftPost {
    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }
}
```

<span class="caption">列表 17-19: 帶有 `content` 方法的 `Post` 和沒有 `content` 方法的 `DraftPost`</span>

`Post` 和 `DraftPost` 結構體都有一個私有的 `content` 字段來儲存博文的文本。這些結構體不再有 `state` 字段因為我們將類型編碼為結構體的類型。`Post` 將代表發佈的博文，它有一個返回 `content` 的 `content` 方法。

仍然有一個 `Post::new` 函數，不過不同於返回 `Post` 實例，它返回 `DraftPost` 的實例。現在不可能創建一個 `Post` 實例，因為 `content` 是私有的同時沒有任何函數返回 `Post`。`DraftPost` 上定義了一個 `add_text` 方法，這樣就可以像之前那樣向 `content` 增加文本，不過注意 `DraftPost` 並沒有定義 `content` 方法！所以所有博文都強制從草案開始，同時草案博文沒有任何可供展示的內容。任何繞過這些限制的嘗試都會產生編譯錯誤。

#### 實現狀態轉移為不同類型的轉移

那麼如何得到發佈的博文呢？我們希望強制的規則是草案博文在可以發佈之前必須被審核通過。等待審核狀態的博文應該仍然不會顯示任何內容。讓我們通過增加另一個結構體 `PendingReviewPost` 來實現這個限制，在 `DraftPost` 上定義 `request_review` 方法來返回 `PendingReviewPost`，並在 `PendingReviewPost` 上定義 `approve` 方法來返回 `Post`，如列表 17-20 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub struct Post {
#     content: String,
# }
#
# pub struct DraftPost {
#     content: String,
# }
#
impl DraftPost {
    // ...snip...

    pub fn request_review(self) -> PendingReviewPost {
        PendingReviewPost {
            content: self.content,
        }
    }
}

pub struct PendingReviewPost {
    content: String,
}

impl PendingReviewPost {
    pub fn approve(self) -> Post {
        Post {
            content: self.content,
        }
    }
}
```

<span class="caption">列表 17-20: `PendingReviewPost` 通過調用 `DraftPost` 的 `request_review` 創建，`approve` 方法將 `PendingReviewPost` 變為發佈的 `Post`</span>

`request_review` 和 `approve` 方法抓取 `self` 的所有權，因此會消費 `DraftPost` 和 `PendingReviewPost` 實例，並分別轉換為 `PendingReviewPost` 和 發佈的 `Post`。這樣在調用 `request_review` 之後就不會遺留任何 `DraftPost` 實例，後者同理。`PendingReviewPost` 並沒有定義 `content` 方法，所以類似 `DraftPost` 嘗試讀取它的內容是一個編譯錯誤。因為唯一得到定義了 `content` 方法的 `Post` 實例的途徑是調用 `PendingReviewPost` 的 `approve` 方法，而得到 `PendingReviewPost` 的唯一辦法是調用 `DraftPost` 的 `request_review` 方法，現在我們就將發博文的工作流編碼進了類型系統。

這也意味著不得不對 `main`做出一些小的修改。因為 `request_review` 和 `approve` 返回新實例而不是修改被調用的結構體，我們需要增加更多的 `let post = ` 覆蓋賦值來保存返回的實例。也不能再斷言草案和等待審核的博文的內容為空字符串了，我們也不再需要他們：不能編譯嘗試使用這些狀態下博文內容的代碼。更新後的 `main` 的代碼如列表 18-21 所示：

<span class="filename">Filename: src/main.rs</span>

```rust
extern crate blog;
use blog::Post;

fn main() {
    let mut post = Post::new();

    post.add_text("I ate a salad for lunch today");

    let post = post.request_review();

    let post = post.approve();

    assert_eq!("I ate a salad for lunch today", post.content());
}
```

<span class="caption">列表 17-21: `main` 中使用新的博文工作流實現的修改</span>

不得不修改 `main` 來重新賦值 `post` 使得這個實現不再完全遵守面向物件的狀態模式：狀態間的轉換不再完全封裝在 `Post` 實現中。然而，得益於類型系統和編譯時類型檢查我們得到了不可能擁有無效狀態的屬性！這確保了特定的 bug，比如顯示未發佈博文的內容，將在部署到生產環境之前被發現。

嘗試在這一部分開始所建議的增加額外需求的任務來體會使用這個版本的代碼是何感覺。

即便 Rust 能夠實現面向物件設計模式，也有其他像將狀態編碼進類型這樣的模式存在。這些模式有著不同於面向物件模式的權衡取捨。雖然你可能非常熟悉面向物件模式，重新思考這些問題來利用 Rust 提供的像在編譯時避免一些 bug 這樣有益功能。在 Rust 中面向物件模式並不總是最好的解決方案，因為 Rust 擁有像所有權這樣的面向物件語言所沒有的功能。

## 總結

閱讀本章後，不管你是否認為 Rust 是一個面向物件語言，現在你都見識了 trait 物件是一個 Rust 中抓取部分面向物件功能的方法。動態分發可以通過犧牲一些運行時性能來為你的代碼提供一些靈活性。這些靈活性可以用來實現有助於代碼可維護性的面向物件模式。Rust 也有像所有權這樣不同於面向物件語言的功能。面向物件模式並不總是利用 Rust 實力的最好方式。

接下來，讓我們看看另一個提供了多樣靈活性的Rust功能：模式。貫穿全書的模式, 我們已經和它們打過照面了，但並沒有見識過它們的全部本領。讓我們開始探索吧！