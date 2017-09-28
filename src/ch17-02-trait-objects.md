## 為使用不同類型的值而設計的 trait 物件

> [ch17-02-trait-objects.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch17-02-trait-objects.md)
> <br>
> commit 67876e3ef5323ce9d394f3ea6b08cb3d173d9ba9

 在第八章中，我們談到了 vector 只能存儲同種類型元素的侷限。在示例 8-1 中有一個例子，其中定義了一個擁有分別存放整型、浮點型和文本型成員的枚舉類型 `SpreadsheetCell`，使用這個枚舉的 vector 可以在每一個單元格（cell）中儲存不同類型的數據，並使得 vector 整體仍然代表一行（row）單元格。這當編譯代碼時就知道希望可以交替使用的類型為固定集合的情況下是可行的。

<!-- The code example I want to reference did not have a listing number; it's
the one with SpreadsheetCell. I will go back and add Listing 8-1 next time I
get Chapter 8 for editing. /Carol -->

有時，我們希望使用的類型的集合對於使用庫的程式設計師來說是可擴展的。例如，很多圖形用戶接口（GUI）工具有一個項目示例的概念，它通過遍歷示例並調用每一個項目的 `draw` 方法來將其繪製到屏幕上。我們將要創建一個叫做 `rust_gui` 的庫 crate，它含一個 GUI 庫的結構。這個 GUI 庫包含一些可供開發者使用的類型，比如 `Button` 或 `TextField`。使用 `rust_gui` 的程式設計師會想要創建更多可以繪製在屏幕上的類型：其中一些可能會增加一個 `Image`，而另一些可能會增加一個 `SelectBox`。本章節並不準備實現一個功能完善的 GUI 庫，不過會展示其中各個部分是如何結合在一起的。

編寫 `rust_gui` 庫時，我們並不知道其他程式設計師想要創建的全部類型，所以無法定義一個 `enum` 來包含所有這些類型。我們所要做的是使 `rust_gui` 能夠記錄一系列不同類型的值，並能夠對其中每一個值調用 `draw` 方法。 GUI 庫不需要知道當調用 `draw` 方法時具體會發生什麼，只需提供這些值可供調用的方法即可。

在擁有繼承的語言中，我們可能定義一個名為 `Component` 的類，該類上有一個 `draw` 方法。其他的類比如 `Button`、`Image` 和 `SelectBox` 會從 `Component` 派生並因此繼承 `draw` 方法。它們各自都可以覆蓋 `draw` 方法來定義自己的行為，但是框架會把所有這些類型當作是 `Component` 的實例，並在其上調用 `draw`。

### 定義通用行為的 trait

不過，在 Rust 中，我們可以定義一個 `Draw` trait，包含名為 `draw` 的方法。接著可以定義一個存放**trait 物件**（*trait
object*）的 vector，trait 物件是一個位於某些指針，比如 `&` 引用或 `Box<T>` 智能指針，之後的 trait。第十九章會講到為何 trait 物件必須位於指針之後的原因。

之前提到過，我們並不將結構體與枚舉稱之為「物件」，以便與其他語言中的物件相區別。結構體與枚舉和 `impl` 塊中的行為是分開的，不同於其他語言中將數據和行為組合進一個稱為物件的概念中。trait 物件將由指向具體物件的指針構成的數據和定義於 trait 中方法的行為結合在一起，從這種意義上說它**則**更類似其他語言中的物件。不過 trait 物件與其他語言中的物件是不同的，因為不能向 trait 物件增加數據。trait 物件並不像其他語言中的物件那麼通用：他們（trait 物件）的作用是允許對通用行為的抽象。

trait 物件定義了在給定情況下所需的行為。接著就可以在要使用具體類型或泛型的地方使用 trait 來作為 trait 物件。Rust 的類型系統會確保任何我們替換為 trait 物件的值都會實現了 trait 的方法。這樣就無需在編譯時就知道所有可能的類型，就能夠用同樣的方法處理所有的實例。示例 17-3 展示了如何定義一個帶有 `draw` 方法的 trait `Draw`：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub trait Draw {
    fn draw(&self);
}
```

<span class="caption">示例 17-3:`Draw` trait 的定義</span>

<!-- NEXT PARAGRAPH WRAPPED WEIRD INTENTIONALLY SEE #199 -->



因為第十章已經討論過如何定義 trait，這看起來應該比較眼熟。接下來就是新內容了：示例 17-4 有一個名為 `Screen` 的結構體定義，它存放了一個叫做 `components` 的 `Box<Draw>` 類型的 vector 。`Box<Draw>` 是一個 trait 物件：它是 `Box` 中任何實現了 `Draw` trait 的類型的替身。

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub trait Draw {
#     fn draw(&self);
# }
#
pub struct Screen {
    pub components: Vec<Box<Draw>>,
}
```

<span class="caption">示例 17-4: 一個 `Screen` 結構體的定義，它帶有一個字段`components`，其包含實現了 `Draw` trait 的 trait 物件的 vector</span>

在 `Screen` 結構體上，我們將定義一個 `run` 方法，該方法會對其 `components` 上的每一個元素調用 `draw` 方法，如示例 17-5 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub trait Draw {
#     fn draw(&self);
# }
#
# pub struct Screen {
#     pub components: Vec<Box<Draw>>,
# }
#
impl Screen {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

<span class="caption">示例 17-5:在 `Screen` 上實現一個 `run` 方法，該方法在每個 component 上調用 `draw` 方法</span>

這與定義使用了帶有 trait bound 的泛型類型參數的結構體不同。泛型類型參數一次只能替代一個具體的類型，而 trait 物件則允許在運行時替代多種具體類型。例如，可以像示例 17-6 那樣定義使用泛型和 trait bound 的結構體 `Screen`：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub trait Draw {
#     fn draw(&self);
# }
#
pub struct Screen<T: Draw> {
    pub components: Vec<T>,
}

impl<T> Screen<T>
    where T: Draw {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

<span class="caption">示例 17-6: 一種 `Screen` 結構體的替代實現，它的 `run` 方法使用泛型和 trait bound</span>

這只允許我們擁有一個包含全是 `Button` 類型或者全是 `TextField` 類型的 component 示例的 `Screen` 實例。如果只擁有相同類型的集合，那麼使用泛型和 trait bound 是更好的，因為在編譯時使用具體類型其定義是單態（monomorphized）的。

相反對於存放了 `Vec<Box<Draw>>` trait 物件的 component 示例的 `Screen` 定義，一個 `Screen` 實例可以存放一個既可以包含 `Box<Button>`，也可以包含 `Box<TextField>` 的 `Vec`。讓我們看看它是如何工作的，接著會講到其運行時性能影響。

### 來自我們或者庫使用者的 trait 實現

現在來增加一些實現了 `Draw` trait 的類型。我們將提供 `Button` 類型，再一次重申，真正實現 GUI 庫超出了本書的範疇，所以 `draw` 方法體中不會有任何有意義的實現。為了想像一下這個實現看起來像什麼，一個 `Button` 結構體可能會擁有 `width`、`height`和`label`字段，如示例 17-7 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub trait Draw {
#     fn draw(&self);
# }
#
pub struct Button {
    pub width: u32,
    pub height: u32,
    pub label: String,
}

impl Draw for Button {
    fn draw(&self) {
        // Code to actually draw a button
    }
}
```

<span class="caption">示例 17-7: 一個實現了`Draw` trait 的 `Button` 結構體</span>

在 `Button` 上的 `width`、`height` 和 `label` 字段會和其他組件不同，比如 `TextField` 可能有 `width`、`height`、`label` 以及 `placeholder` 字段。每一個我們希望能在屏幕上繪製的類型都會使用不同的代碼來實現 `Draw` trait 的 `draw` 方法，來定義如何繪製像這裡的 `Button` 類型（並不包含任何實際的 GUI 代碼，這超出了本章的範疇）。除了實現 `Draw` trait 之外，`Button` 還可能有另一個包含按鈕點擊如何響應的方法的 `impl` 塊。這類方法並不適用於像 `TextField` 這樣的類型。

一些庫的使用者決定實現一個包含 `width`、`height`和`options` 字段的結構體 `SelectBox`。並也為其實現了 `Draw` trait，如示例 17-8 所示：

<span class="filename">文件名: src/main.rs</span>

```rust
extern crate rust_gui;
use rust_gui::Draw;

struct SelectBox {
    width: u32,
    height: u32,
    options: Vec<String>,
}

impl Draw for SelectBox {
    fn draw(&self) {
        // Code to actually draw a select box
    }
}
```

<span class="caption">示例 17-8: 在另一個使用 `rust_gui` 的 crate 中，在 `SelectBox` 結構體上實現 `Draw` trait</span>

庫使用者現在可以在他們的 `main` 函數中創建一個 `Screen` 實例，並通過將 `SelectBox` 和 `Button` 放入 `Box<T>` 轉變為 trait 物件來將它們放入屏幕實例。接著可以調用 `Screen` 的 `run` 方法，它會調用每個組件的 `draw` 方法。示例 17-9 展示了這個實現：

<span class="filename">文件名: src/main.rs</span>

```rust
use rust_gui::{Screen, Button};

fn main() {
    let screen = Screen {
        components: vec![
            Box::new(SelectBox {
                width: 75,
                height: 10,
                options: vec![
                    String::from("Yes"),
                    String::from("Maybe"),
                    String::from("No")
                ],
            }),
            Box::new(Button {
                width: 50,
                height: 10,
                label: String::from("OK"),
            }),
        ],
    };

    screen.run();
}
```

<span class="caption">示例 17-9: 使用 trait 物件來存儲實現了相同 trait 的不同類型的值</span>

即使我們不知道何時何人會增加 `SelectBox` 類型，`Screen` 的實現能夠操作`SelectBox` 並繪製它，因為 `SelectBox` 實現了 `Draw` trait，這意味著它實現了 `draw` 方法。

只關心值所反映的信息而不是值的具體類型，這類似於動態類型語言中稱為**鴨子類型**（*duck typing*）的概念：如果它走起來像一隻鴨子，叫起來像一隻鴨子，那麼它就是一隻鴨子！在示例 17-5 中 `Screen` 上的 `run` 實現中，`run` 並不需要知道各個組件的具體類型是什麼。它並不檢查組件實例是 `Button` 或者是`SelectBox`，它只是調用組件上的 `draw` 方法。通過指定 `Box<Draw>` 作為 `components` vector 中值的類型，我們就定義了 `Screen` 需要可以在其上調用 `draw` 方法的值。

使用 trait 物件和 Rust 類型系統來使用鴨子類型的優勢是無需在運行時檢查一個值是否實現了特定方法或者擔心在調用時因為值沒有實現方法而產生錯誤。如果值沒有實現 trait 物件所需的 trait 則 Rust 不會編譯這些代碼。

例如，示例 17-10 展示了當創建一個使用 `String` 做為其組件的 `Screen` 時發生的情況：

<span class="filename">文件名: src/main.rs</span>

```rust
extern crate rust_gui;
use rust_gui::Draw;

fn main() {
    let screen = Screen {
        components: vec![
            Box::new(String::from("Hi")),
        ],
    };

    screen.run();
}
```

<span class="caption">示例 17-10: 嘗試使用一種沒有實現 trait 物件的 trait 的類型</span>

我們會遇到這個錯誤，因為 `String` 沒有實現 `Draw` trait：

```
error[E0277]: the trait bound `std::string::String: Draw` is not satisfied
  -->
   |
 4 |             Box::new(String::from("Hi")),
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `Draw` is not
   implemented for `std::string::String`
   |
   = note: required for the cast to the object type `Draw`
```

這告訴了我們，要麼是我們傳遞了並不希望傳遞給 `Screen` 的類型並應該提供其他類型，要麼應該在 `String` 上實現 `Draw` 以便 `Screen` 可以調用其上的 `draw`。

### trait 物件執行動態分發

回憶一下第十章討論過的，當對泛型使用 trait bound 時編譯器所進行單態化處理：編譯器為每一個被泛型類型參數代替的具體類型生成了非泛型的函數和方法實現。單態化所產生的代碼進行**靜態分發**（*static dispatch*）：當方法被調用時，伴隨方法調用的代碼在編譯時就被確定了，同時尋找這些代碼是非常快速的。

當使用 trait 物件時，編譯器並不進行單態化，因為並不知道所有可能會使用這些代碼的類型。相反，Rust 記錄當方法被調用時可能會用到的代碼，並在運行時計算出特定方法調用時所需的代碼。這被稱為**動態分發**（*dynamic dispatch*），進行這種代碼搜尋是有運行時開銷的。動態分發也阻止編譯有選擇的內聯方法的代碼，這會禁用一些優化。儘管在編寫和支持代碼的過程中確實獲得了額外的靈活性，但仍然需要權衡取捨。

### Trait 物件要求物件安全

<!-- Liz: we're conflicted on including this section. Not being able to use a
trait as a trait object because of object safety is something that
beginner/intermediate Rust developers run into sometimes, but explaining it
fully is long and complicated. Should we just cut this whole section? Leave it
(and finish the explanation of how to fix the error at the end)? Shorten it to
a quick caveat, that just says something like "Some traits can't be trait
objects. Clone is an example of one. You'll get errors that will let you know
if a trait can't be a trait object, look up object safety if you're interested
in the details"? Thanks! /Carol -->

不是所有的 trait 都可以被放進 trait 物件中；只有**物件安全**（*object safe*）的 trait 才可以。 一個 trait 只有同時滿足如下兩點時才被認為是物件安全的:

* trait 不要求 `Self` 是 `Sized` 的
* 所有的 trait 方法都是物件安全的

`Self` 關鍵字是我們要實現 trait 或方法的類型的別名。`Sized` 是一個類似第十六章中介紹的 `Send` 和 `Sync` 那樣的標記 trait。`Sized` 會自動為在編譯時有已知大小的類型實現，比如 `i32` 和引用。包括 slice （`[T]`）和 trait 物件這樣的沒有已知大小的類型則沒有。

`Sized` 是一個所有泛型參數類型預設的隱含 trait bound。Rust 中大部分實用的操作都要求類型是 `Sized` 的，所以將 `Sized` 作為預設 trait bound 要求，就可以不必在每一次使用泛型時編寫 `T: Sized` 了。然而，如果想要使用在 slice 上使用 trait，則需要去掉 `Sized` trait bound，可以通過指定 `T: ?Sized` 作為 trait bound 來做到這一點。

trait 有一個預設的 bound `Self: ?Sized`，這意味著他們可以在是或者不是 `Sized` 的類型上實現。如果創建了一個去掉了 `Self: ?Sized` bound 的 trait `Foo`，它可能看起來像這樣：

```rust
trait Foo: Sized {
    fn some_method(&self);
}
```

trait `Sized` 現在就是 trait `Foo` 的**父 trait**（*supertrait*）了，也就意味著 trait `Foo` 要求實現 `Foo` 的類型（也就是 `Self`）是 `Sized` 的。我們將在第十九章中更詳細的介紹父 trait。

像 `Foo` 這樣要求 `Self` 是 `Sized` 的 trait 不被允許成為 trait 物件的原因是，不可能為 trait 物件實現 `Foo` trait：trait 物件不是 `Sized` 的，但是 `Foo` 又要求 `Self` 是 `Sized` 的。一個類型不可能同時既是有確定大小的又是無確定大小的。

關於第二條物件安全要求說到 trait 的所有方法都必須是物件安全的，一個物件安全的方法滿足下列條件之一：

* 要求 `Self` 是 `Sized` 的，或者
* 滿足如下三點：
    * 必須不包含任何泛型類型參數
    * 其第一個參數必須是 `Self` 類型或者能解引用為 `Self` 的類型（也就是說它必須是一個方法而非關聯函數，並且以 `self`、`&self` 或 `&mut self` 作為第一個參數）
    * 必須不能在方法簽名中除第一個參數之外的地方使用 `Self`

雖然這些規則有一點形式化, 但是換個角度想一下：如果方法在它的簽名的其他什麼地方要求使用具體的 `Self` 類型，而一個物件又忘記了它具體的類型，這時方法就無法使用它遺忘的原始的具體類型了。當使用 trait 的泛型類型參數被放入具體類型參數時也是如此：這個具體的類型就成了實現該 trait 的類型的一部分。一旦這個類型因使用 trait 物件而被擦除掉了之後，就無法知道放入泛型類型參數的類型是什麼了。

一個 trait 的方法不是物件安全的例子是標準庫中的 `Clone` trait。`Clone` trait 的 `clone` 方法的參數簽名看起來像這樣：

```rust
pub trait Clone {
    fn clone(&self) -> Self;
}
```

`String` 實現了 `Clone` trait，當在 `String` 實例上調用 `clone` 方法時會得到一個 `String` 實例。類似的，當調用 `Vec` 實例的 `clone` 方法會得到一個 `Vec` 實例。`clone` 的簽名需要知道什麼類型會代替 `Self`，因為這是它的返回值。

如果嘗試在像示例 17-3 中 `Draw` 那樣的 trait 上實現 `Clone`，就無法知道 `Self` 將會是 `Button`、`SelectBox` 亦或是將來會實現 `Draw` trait 的其他什麼類型。

如果嘗試做一些違反有關 trait 物件但違反物件安全規則的事情，編譯器會提示你。例如，如果嘗試實現示例 17-4 中的 `Screen` 結構體來存放實現了 `Clone` trait 而不是 `Draw` trait 的類型，像這樣：

```rust
pub struct Screen {
    pub components: Vec<Box<Clone>>,
}
```

將會得到如下錯誤：

```
error[E0038]: the trait `std::clone::Clone` cannot be made into an object
 -->
  |
2 |     pub components: Vec<Box<Clone>>,
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `std::clone::Clone` cannot be
  made into an object
  |
  = note: the trait cannot require that `Self : Sized`
```

<!-- If we are including this section, we would explain how to fix this
problem. It involves adding another trait and implementing Clone manually for
that trait. Because this section is getting long, I stopped because it feels
like we're off in the weeds with an esoteric detail that not everyone will need
to know about. /Carol -->