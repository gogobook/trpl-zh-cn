## 一個使用結構體的示例程序

> [ch05-02-example-structs.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch05-02-example-structs.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

為了理解何時會需要使用結構體，讓我們編寫一個計算長方形面積的程序。我們會從單獨的變數開始，接著重構程序直到使用結構體替代他們為止。

使用 Cargo 來創建一個叫做 *rectangles* 的新二進製程序，它會獲取一個長方形以像素為單位的長度和寬度並計算它的面積。列表 5-2 中是項目的 *src/main.rs* 文件中為此實現的一個小程序：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let length1 = 50;
    let width1 = 30;

    println!(
        "The area of the rectangle is {} square pixels.",
        area(length1, width1)
    );
}

fn area(length: u32, width: u32) -> u32 {
    length * width
}
```

<span class="caption">列表 5-8：通過指定長方形的長寬變數來計算長方形面積</span>

嘗試使用 `cargo run` 運行程序：

```text
The area of the rectangle is 1500 square pixels.
```

### 使用元組重構

雖然列表 5-8 可以運行，並調用 `area` 函數用長方形的每個維度來計算出面積，不過我們可以做的更好。長度和寬度是相關聯的，因為他們在一起才能定義一個長方形。

這個做法的問題突顯在 `area` 的簽名上：

```rust
fn area(length: u32, width: u32) -> u32 {
```

函數 `area` 本應該計算一個長方形的面積，不過函數卻有兩個參數。這兩個參數是相關聯的，不過程序自身卻哪裡也沒有表現出這一點。將長度和寬度組合在一起將更易懂也更易處理。第三章的 「將值組合進元組」 部分已經討論過了一種可行的方法：元組。列表 5-9 是另一個使用元組的版本：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let rect1 = (50, 30);

    println!(
        "The area of the rectangle is {} square pixels.",
        area(rect1)
    );
}

fn area(dimensions: (u32, u32)) -> u32 {
    dimensions.0 * dimensions.1
}
```

<span class="caption">列表 5-8：使用元組來指定長方形的長寬</span>

在某種程度上說這樣好一點了。元組幫助我們增加了一些結構性，現在在調用 `area` 的時候只用傳遞一個參數。不過在另一方面這個方法卻更不明確了：元組並沒有給出它元素的名稱，所以計算變得更費解了，因為不得不使用索引來獲取元組的每一部分：

在面積計算時混淆長寬並沒有什麼問題，不過當在屏幕上繪製長方形時就有問題了！我們將不得不記住元組索引 `0` 是 `length` 而 `1` 是 `width`。如果其他人要使用這些代碼，他們也不得不搞清楚並記住他們。容易忘記或者混淆這些值而造成錯誤，因為我們沒有表明代碼中數據的意義。

### 使用結構體重構：增加更多意義

現在引入結構體的時候了。我們可以將元組轉換為一個有整體名稱而且每個部分也有對應名字的數據類型，如列表 5-10 所示：

<span class="filename">文件名: src/main.rs</span>

```rust
struct Rectangle {
    length: u32,
    width: u32,
}

fn main() {
    let rect1 = Rectangle { length: 50, width: 30 };

    println!(
        "The area of the rectangle is {} square pixels.",
        area(&rect1)
    );
}

fn area(rectangle: &Rectangle) -> u32 {
    rectangle.length * rectangle.width
}
```

<span class="caption">列表 5-10：定義 `Rectangle` 結構體</span>

這裡我們定義了一個結構體並稱其為 `Rectangle`。在 `{}` 中定義了字段 `length` 和 `width`，都是 `u32` 類型的。接著在 `main` 中，我們創建了一個長度為 50 和寬度為 30 的 `Rectangle` 的具體實例。

函數 `area` 現在被定義為接收一個名叫 `rectangle` 的參數，它的類型是一個結構體 `Rectangle` 實例的不可變借用。第四章講到過，我們希望借用結構體而不是獲取它的所有權這樣 `main` 函數就可以保持 `rect1` 的所有權並繼續使用它，所以這就是為什麼在函數簽名和調用的地方會有 `&`。

`area` 函數訪問 `Rectangle` 的 `length` 和 `width` 字段。`area` 的簽名現在明確的表明了我們的意圖：通過其 `length` 和 `width` 字段，計算一個 `Rectangle` 的面積。這表明了長度和寬度是相互聯繫的，並為這些值提供了描述性的名稱而不是使用元組的索引值 `0` 和 `1` 。結構體勝在更清晰明了。

### 通過衍生 trait 增加實用功能

如果能夠在調試程序時打印出 `Rectangle` 實例來查看其所有字段的值就更好了。列表 5-11 像往常一樣使用 `println!` 巨集：

<span class="filename">文件名: src/main.rs</span>

```rust
struct Rectangle {
    length: u32,
    width: u32,
}

fn main() {
    let rect1 = Rectangle { length: 50, width: 30 };

    println!("rect1 is {}", rect1);
}
```

<span class="caption">列表 5-11：嘗試打印出 `Rectangle` 實例</span>

如果運行代碼，會出現帶有如下核心信息的錯誤：

```text
error[E0277]: the trait bound `Rectangle: std::fmt::Display` is not satisfied
```

`println!` 巨集能處理很多類型的格式，不過，`{}`，默認告訴 `println!` 使用被稱為 `Display` 的格式：直接提供給終端用戶查看的輸出。目前為止見過的基本類型都默認實現了 `Display`，因為它就是向用戶展示 `1` 或其他任何基本類型的唯一方式。不過對於結構體，`println!` 應該用來輸出的格式是不明確的，因為這有更多顯示的可能性：是否需要逗號？需要打印出結構體的 `{}` 嗎？所有字段都應該顯示嗎？因為這種不確定性，Rust 不嘗試猜測我們的意圖所以結構體並沒有提供一個 `Display` 實現。

但是如果我們繼續閱讀錯誤，將會發現這個有幫助的信息：

```text
note: `Rectangle` cannot be formatted with the default formatter; try using
`:?` instead if you are using a format string
```

讓我們來試試！現在 `println!` 巨集調用看起來像 `println!("rect1 is {:?}", rect1);` 這樣。在 `{}` 中加入 `:?` 指示符告訴 `println!` 我們想要使用叫做 `Debug` 的輸出格式。`Debug` 是一個 trait，它允許我們在調試代碼時以一種對開發者有幫助的方式打印出結構體。

讓我們試試運行這個變化。見鬼了！仍然能看到一個錯誤：

```text
error: the trait bound `Rectangle: std::fmt::Debug` is not satisfied
```

不過編譯器又一次給出了一個有幫助的信息！

```text
note: `Rectangle` cannot be formatted using `:?`; if it is defined in your
crate, add `#[derive(Debug)]` or manually implement it
```

Rust **確實** 包含了打印出調試信息的功能，不過我們必須為結構體顯式選擇這個功能。為此，在結構體定義之前加上 `#[derive(Debug)]` 註解，如列表 5-12 所示：

<span class="filename">文件名: src/main.rs</span>

```rust
#[derive(Debug)]
struct Rectangle {
    length: u32,
    width: u32,
}

fn main() {
    let rect1 = Rectangle { length: 50, width: 30 };

    println!("rect1 is {:?}", rect1);
}
```

<span class="caption">列表 5-12：增加註解來導出 `Debug` trait </span>

此時此刻運行程序，運行這個程序，不會有任何錯誤並會出現如下輸出：

```text
rect1 is Rectangle { length: 50, width: 30 }
```

好極了！這並不是最漂亮的輸出，不過它顯示這個實例的所有字段，毫無疑問這對調試有幫助。如果想要輸出再好看和易讀一點，可以將 `println!` 的字符串中的 `{:?}` 替換為 `{:#?} `，這對更大的結構體會有幫助。如果在這個例子中使用了 `{:#?}` 風格的話，輸出會看起來像這樣：

```text
rect1 is Rectangle {
    length: 50,
    width: 30
}
```

Rust 為我們提供了很多可以通過 `derive` 註解來使用的 trait，他們可以為我們的自定義類型增加有益的行為。這些 trait 和行為在附錄 C 中列出。第十章會涉及到如何通過自定義行為來實現這些 trait，同時還有如何創建你自己的 trait。

我們的 `area` 函數是非常特化的————它只是計算了長方形的面積。如果這個行為與 `Rectangle` 結構體再結合得更緊密一些就更好了，因為它不能用於其他類型。現在讓我們看看如何繼續重構這些代碼，來將 `area` 函數協調進 `Rectangle` 類型定義的`area` **方法** 中。