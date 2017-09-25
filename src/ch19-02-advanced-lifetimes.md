## 高級生命週期

> [ch19-02-advanced-lifetimes.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch19-02-advanced-lifetimes.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

回顧第十章，我們學習了怎樣使用生命週期參數來註解引用來幫助 Rust 理解不同引用的生命週期如何相互聯繫。見識到了大部分情況 Rust 允許我們省略生命週期，不過每一個引用都有一個生命週期。這裡有三個生命週期的高級特徵我們還未講到：**生命週期子類型**（*lifetime subtyping*），**生命週期 bound**（*lifetime bounds*），以及**trait 物件生命週期**（*trait object lifetimes*）。

### 生命週期子類型

想像一下我們想要編寫一個解析器。為此，會有一個儲存了需要解析的字符串的引用的結構體，我們稱之為結構體 `Context`。解析器將會解析字符串並返回成功或失敗。解析器需要借用 `Context` 來進行解析。其實現看起來像列表 19-12 中的代碼，它還不能編譯，因為目前我們去掉了生命週期註解：

```rust
struct Context(&str);

struct Parser {
    context: &Context,
}

impl Parser {
    fn parse(&self) -> Result<(), &str> {
        Err(&self.context.0[1..])
    }
}
```

<span class="caption">列表 19-12：定義結構體 `Context` 來存放一個字符串 slice，結構體 `Parser` 包含一個 `Context` 實例和一個 `parse` 方法，它總是返回一個引用了字符串 slice 的錯誤</span>

為了簡單起見，`parse` 方法返回 `Result<(), &str>`。也就是說，成功時不做任何操作，失敗時則返回字符串 slice 沒有正確解析的部分。真實的實現將會包含比這更多的錯誤信息，也將會在解析成功時返回創建的結果，不過我們將去掉這些部分的實現，因為他們與這個例子的生命週期部分並不相關。我們還定義了 `parse` 總是在第一個字節之後返回錯誤。注意如果第一個字節並不位於一個有效的字符範圍內（比如 Unicode）將會 panic；我們有一次簡化了例子以專注於涉及到的生命週期。

那麼我們如何為 `Context` 中的字符串 slice 和 `Parser` 中 `Context` 的引用放入生命週期參數呢？最直接的方法是在每處都使用相同的生命週期，如列表 19-13 所示：

```rust
struct Context<'a>(&'a str);

struct Parser<'a> {
    context: &'a Context<'a>,
}

impl<'a> Parser<'a> {
    fn parse(&self) -> Result<(), &str> {
        Err(&self.context.0[1..])
    }
}
```

<span class="caption">列表 19-13：將所有 `Context` 和 `Parser` 的引用標註為相同的生命週期參數</span>

這次可以編譯了。接下來，在列表 19-14 中，讓我們編寫一個抓取 `Context` 的實例，使用 `Parser` 來解析其內容，並返回 `parse` 的返回值的函數。這還不能運行：

```rust
fn parse_context(context: Context) -> Result<(), &str> {
    Parser { context: &context }.parse()
}
```

<span class="caption">列表 19-14：一個增加抓取 `Context` 並使用 `Parser` 的函數 `parse_context` 的嘗試</span>

當嘗試編譯這段額外帶有 `parse_context` 函數的代碼時會得到兩個相當冗長的錯誤：

```
error: borrowed value does not live long enough
  --> <anon>:16:5
   |
16 |     Parser { context: &context }.parse()
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ does not live long enough
17 | }
   | - temporary value only lives until here
   |
note: borrowed value must be valid for the anonymous lifetime #1 defined on the
body at 15:55...
  --> <anon>:15:56
   |
15 |   fn parse_context(context: Context) -> Result<(), &str> {
   |  ________________________________________________________^
16 | |     Parser { context: &context }.parse()
17 | | }
   | |_^

error: `context` does not live long enough
  --> <anon>:16:24
   |
16 |     Parser { context: &context }.parse()
   |                        ^^^^^^^ does not live long enough
17 | }
   | - borrowed value only lives until here
   |
note: borrowed value must be valid for the anonymous lifetime #1 defined on the
body at 15:55...
  --> <anon>:15:56
   |
15 |   fn parse_context(context: Context) -> Result<(), &str> {
   |  ________________________________________________________^
16 | |     Parser { context: &context }.parse()
17 | | }
   | |_^
```

這些錯誤表明我們創建的兩個 `Parser` 實例和 `context` 參數從 `Parser` 被創建開始一直存活到 `parse_context` 函數結束，不過他們都需要在整個函數的生命週期中都有效。

換句話說，`Parser` 和 `context` 需要比整個函數**長壽**（*outlive*）並在函數開始之前和結束之後都有效以確保代碼中的所有引用始終是有效的。雖然兩個我們創建的 `Parser` 和 `context` 參數在函數的結尾就離開了作用域（因為 `parse_context` 抓取了 `context` 的所有權）。

讓我們再次看看列表 19-13 中的定義，特別是 `parse` 方法的簽名：

```rust
    fn parse(&self) -> Result<(), &str> {
```

還記得（生命週期）省略規則嗎？如果標註了引用生命週期，簽名看起來應該是這樣：

```rust
    fn parse<'a>(&'a self) -> Result<(), &'a str> {
```

正是如此，`parse` 返回值的錯誤部分的生命週期與 `Parser` 實例的生命週期（`parse` 方法簽名中的 `&self`）相綁定。這就可以理解了，因為返回的字符串 slice 引用了 `Parser` 存放的 `Context` 實例中的字符串 slice，同時在 `Parser` 結構體的定義中我們指定了 `Parser` 中存放的 `Context` 引用的生命週期和 `Context` 中存放的字符串 slice 的生命週期應該一致。

問題是 `parse_context` 函數返回 `parse` 返回值，所以 `parse_context` 返回值的生命週期也與 `Parser` 的生命週期相聯繫。不過 `parse_context` 函數中創建的 `Parser` 實例並不能存活到函數結束之後（它是臨時的），同時 `context` 將會在函數的結尾離開作用域（`parse_context` 抓取了它的所有權）。

不允許一個在函數結尾離開作用域的值的引用。Rust 認為這是我們想要做的，因為我們將所有生命週期用相同的生命週期參數標記。這告訴了 Rust `Context` 中存放的字符串 slice 的生命週期與 `Parser` 中存放的 `Context` 引用的生命週期一致。

`parse_context` 函數並不知道 `parse` 函數裡面是什麼，返回的字符串 slice 將比 `Context` 和 `Parser` 都存活的更久，因此 `parse_context` 返回的引用指向字符串 slice，而不是 `Context` 或 `Parser`。

通過瞭解 `parse` 實現所做的工作，可以知道 `parse` 的返回值（的生命週期）與 `Parser` 相聯繫的唯一理由是它引用了 `Parser` 的 `Context`，也就是引用了這個字符串 slice，這正是 `parse_context` 所需要關心的生命週期。需要一個方法來告訴 Rust `Context` 中的字符串 slice 與 `Parser` 中 `Context` 的引用有著不同的生命週期，而且 `parse_context` 返回值與 `Context` 中字符串 slice 的生命週期相聯繫。

我們只能嘗試像列表 19-15 那樣給予 `Parser` 和 `Context` 不同的生命週期參數。這裡選擇了生命週期參數名 `'s` 和 `'c` 是為了使得 `Context` 中字符串 slice 與 `Parser` 中 `Context` 引用的生命週期顯得更明了（英文首字母）。注意這並不能完全解決問題，不過這是一個開始，我們將看看為什麼這還不足以能夠編譯代碼。

```rust
struct Context<'s>(&'s str);

struct Parser<'c, 's> {
    context: &'c Context<'s>,
}

impl<'c, 's> Parser<'c, 's> {
    fn parse(&self) -> Result<(), &'s str> {
        Err(&self.context.0[1..])
    }
}

fn parse_context(context: Context) -> Result<(), &str> {
    Parser { context: &context }.parse()
}
```

<span class="caption">列表 19-15：為字符串 slice 和 `Context` 的引用指定不同的生命週期參數</span>

這裡在與列表 19-13 完全相同的地方標註了引用的生命週期，不過根據引用是字符串 slice 或 `Context` 與否使用了不同的參數。另外還在 `parse` 返回值的字符串 slice 部分增加了註解來表明它與 `Context` 中字符串 slice 的生命週期相關聯。

這裡是現在得到的錯誤：

```
error[E0491]: in type `&'c Context<'s>`, reference has a longer lifetime than the data it references
 --> src/main.rs:4:5
  |
4 |     context: &'c Context<'s>,
  |     ^^^^^^^^^^^^^^^^^^^^^^^^
  |
note: the pointer is valid for the lifetime 'c as defined on the struct at 3:0
 --> src/main.rs:3:1
  |
3 | / struct Parser<'c, 's> {
4 | |     context: &'c Context<'s>,
5 | | }
  | |_^
note: but the referenced data is only valid for the lifetime 's as defined on the struct at 3:0
 --> src/main.rs:3:1
  |
3 | / struct Parser<'c, 's> {
4 | |     context: &'c Context<'s>,
5 | | }
  | |_^
```

Rust 並不知道 `'c` 與 `'s` 之間的任何聯繫。為了保證有效性，`Context`中引用的帶有生命週期 `'s` 的數據需要遵守它比帶有生命週期 `'c` 的 `Context` 的引用存活得更久的保證。如果 `'s` 不比 `'c` 更長久，那麼 `Context` 的引用可能不再有效。

這就引出了本部分的要點：Rust 有一個叫做**生命週期子類型**的功能，這是一個指定一個生命週期不會短於另一個的方法。在聲明生命週期參數的尖括號中，可以照常聲明一個生命週期 `'a`，並通過語法 `'b: 'a` 聲明一個不短於 `'a` 的生命週期 `'b`。

在 `Parser` 的定義中，為了表明 `'s`（字符串 slice 的生命週期）保證至少與 `'c`（`Context` 引用的生命週期）一樣長，需將生命週期聲明改為如此：

```rust
# struct Context<'a>(&'a str);
#
struct Parser<'c, 's: 'c> {
    context: &'c Context<'s>,
}
```

現在 `Parser` 中 `Context` 的引用與 `Context` 中字符串 slice 就有了不同的生命週期，並且保證了字符串 slice 的生命週期比 `Context` 引用的要長。

這是一個非常冗長的例子，不過正如本章的開頭所提到的，這類功能是很小眾的。你並不會經常需要這個語法，不過當出現類似這樣的情形時，卻還是有地方可以參考的。

### 生命週期 bound

在第十章，我們討論了如何在泛型類型上使用 trait bound。也可以像泛型那樣為生命週期參數增加限制，這被稱為**生命週期 bound**。例如，考慮一下一個封裝了引用的類型。回憶一下第十五章的 `RefCell<T>` 類型：其 `borrow` 和 `borrow_mut` 方法分別返回 `Ref` 和 `RefMut` 類型。這些類型是引用的封裝，他們在運行時記錄檢查借用規則。`Ref` 結構體的定義如列表 19-16 所示，現在還不帶有生命週期 bound：

```rust
struct Ref<'a, T>(&'a T);
```

<span class="caption">列表 19-16：定義結構體來封裝泛型的引用；開始時沒有生命週期 bound</span>

若不限制生命週期 `'a` 為與泛型參數 `T` 有關，會得到一個錯誤因為 Rust 不知道泛型 `T` 會存活多久：

```
error[E0309]: the parameter type `T` may not live long enough
 --> <anon>:1:19
  |
1 | struct Ref<'a, T>(&'a T);
  |                   ^^^^^^
  |
  = help: consider adding an explicit lifetime bound `T: 'a`...
note: ...so that the reference type `&'a T` does not outlive the data it points at
 --> <anon>:1:19
  |
1 | struct Ref<'a, T>(&'a T);
  |                   ^^^^^^
```

因為 `T` 可以是任意類型，`T` 自身也可能是一個引用，或者是一個存放了一個或多個引用的類型，而他們各自可能有著不同的生命週期。Rust 不能確認 `T` 會與 `'a` 存活的一樣久。

幸運的是，Rust 提供了這個情況下如何指定生命週期 bound 的有用建議：

```
consider adding an explicit lifetime bound `T: 'a` so that the reference type
`&'a T` does not outlive the data it points at.
```

列表 19-17 展示了按照這個建議，在聲明泛型 `T` 時指定生命週期 bound。現在代碼可以編譯了，因為 `T: 'a` 指定了 `T` 可以為任意類型，不過如果它包含任何引用的話，其生命週期必須至少與 `'a` 一樣長：

```rust
struct Ref<'a, T: 'a>(&'a T);
```

<span class="caption">列表19-17：為 `T` 增加生命週期 bound 來指定 `T` 中的任何引用需至少與 `'a` 存活的一樣久</span>

我們可以選擇不同的方法來解決這個問題，如列表 19-18 中展示的 `StaticRef` 結構體定義所示，通過在 `T` 上增加 `'static` 生命週期 bound。這意味著如果 `T` 包含任何引用，他們必須有 `'static` 生命週期：

```rust
struct StaticRef<T: 'static>(&'static T);
```

<span class="caption">列表 19-18：在 `T` 上增加 `'static` 生命週期 bound 來限制 `T` 為只擁有 `'static` 引用或沒有引用的類型</span>

沒有任何引用的類型被算作 `T: 'static`。因為 `'static` 意味著引用必須同整個程序存活的一樣長，一個不包含引用的類型滿足所有引用都與程序存活的一樣長的標準（因為他們沒有引用）。可以這樣理解：如果借用檢查器關心的是引用是否存活的夠久，那麼沒有引用的類型與有永遠存在的引用的類型並沒有真正的區別；對於確定引用是否比其所引用的值存活得較短的目的來說兩者是一樣的。

### trait 物件生命週期

在第十七章，我們學習了 trait 物件，其中介紹了可以把一個 trait 放在一個引用後面來進行動態分發。然而，我們並沒有討論如果 trait 物件中實現 trait 的類型帶有生命週期時會發生什麼。考慮一下 19-19，這裡有 trait `Foo`，和帶有一個實現了 trait `Foo` 的引用（因此還有其生命週期參數）的結構體 `Bar`，我們希望使用 `Bar` 的實例作為 trait 物件 `Box<Foo>`：

```rust
trait Foo { }

struct Bar<'a> {
    x: &'a i32,
}

impl<'a> Foo for Bar<'a> { }

let num = 5;

let obj = Box::new(Bar { x: &num }) as Box<Foo>;
```

<span class="caption">列表 19-19：使用一個帶有生命週期的類型作為 trait 物件</span>

這些代碼能沒有任何錯誤的編譯，即便並沒有明確指出 `obj` 中涉及的任何生命週期。這是因為有如下生命週期與 trait 物件必須遵守的規則：

* trait 物件的預設生命週期是 `'static`。
* 如果有 `&'a X` 或 `&'a mut X`，則預設（生命週期）是 `'a`。
* 如果只有 `T: 'a`， 則預設是 `'a`。
* 如果有多個類似 `T: 'a` 的從句，則沒有預設值；必須明確指定。

當必須明確指定時，可以為像 `Box<Foo>` 這樣的 trait 物件增加生命週期 bound，根據需要使用語法 `Box<Foo + 'a>` 或 `Box<Foo + 'static>`。正如其他的 bound，這意味著任何 `Foo` trait 的實現如果在內部包含有引用, 就必須在 trait 物件 bounds 中為那些引用指定生命週期。

接下來，讓我們看看一些其他處理 trait 的功能吧！