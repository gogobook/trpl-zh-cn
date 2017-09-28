# 定義枚舉

> [ch06-01-defining-an-enum.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch06-01-defining-an-enum.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

讓我們通過一用代碼來表現的場景，來看看為什麼這裡枚舉是有用的而且比結構體更合適。比如我們要處理 IP 地址。目前被廣泛使用的兩個主要 IP 標準：IPv4（version four）和 IPv6（version six）。這是我們的程序只可能會遇到兩種 IP 地址：所以可以 **枚舉** 出所有可能的值，這也正是它名字的由來。

任何一個 IP 地址要麼是 IPv4 的要麼是 IPv6 的而不能兩者都是。IP 地址的這個特性使得枚舉數據結構非常適合這個場景，因為枚舉值只可能是其中一個成員。IPv4 和 IPv6 從根本上講仍是 IP 地址，所以當代碼在處理申請任何類型的 IP 地址的場景時應該把他們當作相同的類型。

可以通過在代碼中定義一個 `IpAddrKind` 枚舉來表現這個概念並列出可能的 IP 地址類型，`V4` 和 `V6`。這被稱為枚舉的 **成員**（*variants*）：

```rust
enum IpAddrKind {
    V4,
    V6,
}
```

現在 `IpAddrKind` 就是一個可以在代碼中使用的自定義類型了。

### 枚舉值

可以像這樣創建 `IpAddrKind` 兩個不同成員的實例：

```rust
# enum IpAddrKind {
#     V4,
#     V6,
# }
#
let four = IpAddrKind::V4;
let six = IpAddrKind::V6;
```

注意枚舉的成員位於其標識符的命名空間中，並使用兩個冒號分開。這麼設計的益處是現在 `IpAddrKind::V4` 和 `IpAddrKind::V6` 是相同類型的：`IpAddrKind`。例如，接著可以定義一個函數來抓取任何 `IpAddrKind`：

```rust
# enum IpAddrKind {
#     V4,
#     V6,
# }
#
fn route(ip_type: IpAddrKind) { }
```

現在可以使用任意成員來調用這個函數：

```rust
# enum IpAddrKind {
#     V4,
#     V6,
# }
#
# fn route(ip_type: IpAddrKind) { }
#
route(IpAddrKind::V4);
route(IpAddrKind::V6);
```

使用枚舉甚至還有更多優勢。進一步考慮一下我們的 IP 地址類型，目前沒有一個儲存實際 IP 地址 **數據** 的方法；只知道它是什麼 **類型** 的。考慮到已經在第五章學習過結構體了，你可能會像示例 6-1 那樣修改這個問題：

```rust
enum IpAddrKind {
    V4,
    V6,
}

struct IpAddr {
    kind: IpAddrKind,
    address: String,
}

let home = IpAddr {
    kind: IpAddrKind::V4,
    address: String::from("127.0.0.1"),
};

let loopback = IpAddr {
    kind: IpAddrKind::V6,
    address: String::from("::1"),
};
```

<span class="caption">示例 6-1：將 IP 地址的數據和 `IpAddrKind` 成員儲存在一個 `struct` 中</span>

這裡我們定義了一個有兩個字段的結構體 `IpAddr`：`kind` 字段是 `IpAddrKind`（之前定義的枚舉）類型的而 `address` 字段是 `String` 類型的。這裡有兩個結構體的實例。第一個，`home`，它的 `kind` 的值是 `IpAddrKind::V4` 與之相關聯的地址數據是 `127.0.0.1`。第二個實例，`loopback`，`kind` 的值是 `IpAddrKind` 的另一個成員，`V6`，關聯的地址是 `::1`。我們使用了一個結構體來將 `kind` 和 `address` 打包在一起，現在枚舉成員就與值相關聯了。

我們可以使用一種更簡潔的方式來表達相同的概念，僅僅使用枚舉並將數據直接放進每一個枚舉成員而不是將枚舉作為結構體的一部分。`IpAddr` 枚舉的新定義表明了 `V4` 和 `V6` 成員都關聯了 `String` 值：

```rust
enum IpAddr {
    V4(String),
    V6(String),
}

let home = IpAddr::V4(String::from("127.0.0.1"));

let loopback = IpAddr::V6(String::from("::1"));
```

我們直接將數據附加到枚舉的每個成員上，這樣就不需要一個額外的結構體了。

使用枚舉而不是結構體還有另外一個優勢：每個成員可以處理不同類型和數量的數據。IPv4 版本的 IP 地址總是含有四個值在 0 和 255 之間的數字部分。如果我們想要將 `V4` 地址儲存為四個 `u8` 值而 `V6` 地址仍然表現為一個 `String`，這就不能使用結構體了。枚舉可以輕易處理的這個情況：

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

let home = IpAddr::V4(127, 0, 0, 1);

let loopback = IpAddr::V6(String::from("::1"));
```

這些代碼展示了使用枚舉來儲存兩種不同 IP 地址的幾種可能的選擇。然而，事實證明儲存和編碼 IP 地址實在是太常見了[以致標準庫提供了一個可供使用的定義！][IpAddr]<!-- ignore -->讓我們看看標準庫如何定義 `IpAddr` 的：它正有著跟我們定義和使用的一樣的枚舉和成員，不過它將成員中的地址數據嵌入到了兩個不同形式的結構體中，他們對不同的成員的定義是不同的：

[IpAddr]: https://doc.rust-lang.org/std/net/enum.IpAddr.html

```rust
struct Ipv4Addr {
    // details elided
}

struct Ipv6Addr {
    // details elided
}

enum IpAddr {
    V4(Ipv4Addr),
    V6(Ipv6Addr),
}
```

這些代碼展示了可以將任意類型的數據放入枚舉成員中：例如字符串、數字類型或者結構體。甚至可以包含另一個枚舉！另外，標準庫中的類型通常並不比你設想出來的要複雜多少。

注意雖然標準庫中包含一個 `IpAddr` 的定義，仍然可以創建和使用我們自己的定義而不會有衝突，因為我們並沒有將標準庫中的定義引入作用域。第七章會講到如何導入類型。

來看看示例 6-2 中的另一個枚舉的例子：它的成員中內嵌了多種多樣的類型：

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}
```

<span class="caption">示例 6-2：一個 `Message` 枚舉，其每個成員都儲存了不同數量和類型的值</span>

這個枚舉有四個含有不同類型的成員：

* `Quit` 沒有關聯任何數據。
* `Move` 包含一個匿名結構體
* `Write` 包含單獨一個 `String`。
* `ChangeColor` 包含三個 `i32`。

定義一個像示例 6-2 中的枚舉類似於定義不同類型的結構體，除了枚舉不使用 `struct` 關鍵字而且所有成員都被組合在一起位於 `Message` 下之外。如下這些結構體可以包含與之前枚舉成員中相同的數據：

```rust
struct QuitMessage; // unit struct
struct MoveMessage {
    x: i32,
    y: i32,
}
struct WriteMessage(String); // tuple struct
struct ChangeColorMessage(i32, i32, i32); // tuple struct
```

不過如果我們使用不同的結構體，他們都有不同的類型，將不能輕易的定義一個抓取任何這些信息類型的函數，正如可以使用示例 6-2 中定義的 `Message` 枚舉那樣，因為他們是一個類型的。

枚舉和結構體還有另一個相似點：就像可以使用 `impl` 來為結構體定義方法那樣，也可以在枚舉上定義方法。這是一個定義於我們 `Message` 枚舉上的叫做 `call` 的方法：

```rust
# enum Message {
#     Quit,
#     Move { x: i32, y: i32 },
#     Write(String),
#     ChangeColor(i32, i32, i32),
# }
#
impl Message {
    fn call(&self) {
        // method body would be defined here
    }
}

let m = Message::Write(String::from("hello"));
m.call();
```

方法體使用了 `self` 來抓取調用方法的值。這個例子中，創建了一個擁有類型 `Message::Write("hello")` 的變數 `m`，而且這就是當 `m.call()` 運行時 `call` 方法中的 `self` 的值。

讓我們看看標準庫中的另一個非常常見且實用的枚舉：`Option`。

### `Option` 枚舉和其相對於空值的優勢

在之前的部分，我們看到了 `IpAddr` 枚舉如何利用 Rust 的類型系統編碼更多信息而不單單是程序中的數據。這一部分探索一個 `Option` 的案例分析，它是另一個標準庫定義的枚舉。`Option` 類型應用廣泛因為它編碼了一個非常普遍的場景，就是一個值可能是某個值或者什麼都不是。從類型系統的角度來表達這個概念就意味著編譯器需要檢查是否處理了所有應該處理的情況，這樣就可以避免在其他編程語言中非常常見的 bug。

編程語言的設計經常從其包含功能的角度考慮問題，但是從其所排除在外的功能的角度思考也很重要。Rust 並沒有很多其他語言中有的空值功能。**空值**（*Null* ）是一個值，它代表沒有值。在有空值的語言中，變數總是這兩種狀態之一：空值和非空值。

在 「Null References: The Billion Dollar Mistake」 中，Tony Hoare，null 的發明者，曾經說到：

> I call it my billion-dollar mistake. At that time, I was designing the first
> comprehensive type system for references in an object-oriented language. My
> goal was to ensure that all use of references should be absolutely safe, with
> checking performed automatically by the compiler. But I couldn't resist the
> temptation to put in a null reference, simply because it was so easy to
> implement. This has led to innumerable errors, vulnerabilities, and system
> crashes, which have probably caused a billion dollars of pain and damage in
> the last forty years.
>
> 我稱之為我的萬億美元錯誤。當時，我在為一個物件導向語言設計第一個綜合性的面向引用的類型系統。我的目標是通過編譯器的自動檢查來保證所有引用的應有都應該是絕對安全的。不過我未能抗拒引入一個空引用的誘惑，僅僅是因為它是這麼的容易實現。這引發了無數錯誤、漏洞和系統崩潰，在之後的四十多年中造成了數以萬計美元的苦痛和傷害。

空值的問題在於當你嘗試像一個非空值那樣使用一個空值，會出現某種形式的錯誤。因為空和非空的屬性是無處不在的，非常容易出現這類錯誤。

然而，空值嘗試表達的概念仍然是有意義的：空值是一個因為某種原因目前無效或缺失的值。

問題不在於具體的概念而在於特定的實現。為此，Rust 並沒有空值，不過它確實擁有一個可以編碼存在或不存在概念的枚舉。這個枚舉是 `Option<T>`，而且它[定義於標準庫中][option]<!-- ignore -->，如下:

[option]: https://doc.rust-lang.org/std/option/enum.Option.html

```rust
enum Option<T> {
    Some(T),
    None,
}
```

`Option<T>` 是如此有用以至於它甚至被包含在了 prelude 之中：不需要顯式導入它。另外，它的成員也是如此：可以不需要 `Option::` 前綴來直接使用 `Some` 和 `None`。即便如此 `Option<T>` 也仍是常規的枚舉，`Some(T)` 和 `None` 仍是 `Option<T>` 的成員。

`<T>` 語法是一個我們還未講到的 Rust 功能。它是一個泛型類型參數，第十章會更詳細的講解泛型。目前，所有你需要知道的就是 `<T>` 味著 `Option` 枚舉的 `Some` 成員可以包含任意類型的數據。這裡是一些包含數字類型和字符串類型 `Option` 值的例子：

```rust
let some_number = Some(5);
let some_string = Some("a string");

let absent_number: Option<i32> = None;
```

如果使用 `None` 而不是 `Some`，需要告訴 Rust `Option<T>` 是什麼類型的，因為編譯器只通過 `None` 值無法推斷出 `Some` 成員的類型。

當有一個 `Some` 值時，我們就知道存在一個值，而這個值保存在 `Some` 中。當有個`None` 值時，在某種意義上它跟空值是相同的意義：並沒有一個有效的值。那麼，`Option<T>` 為什麼就比空值要好呢？


簡而言之，因為 `Option<T>` 和 `T`（這裡 `T` 可以是任何類型）是不同的類型，編譯器不允許像一個被定義的有效的類型那樣使用 `Option<T>`。例如，這些代碼不能編譯，因為它嘗試將 `Option<i8>` 與 `i8` 相比：

```rust
let x: i8 = 5;
let y: Option<i8> = Some(5);

let sum = x + y;
```

如果運行這些代碼，將得到類似這樣的錯誤信息：

```text
error[E0277]: the trait bound `i8: std::ops::Add<std::option::Option<i8>>` is
not satisfied
 -->
  |
7 | let sum = x + y;
  |           ^^^^^
  |
```

哇哦！事實上，錯誤信息意味著 Rust 不知道該如何將 `Option<i8>` 與 `i8` 相加。當在 Rust 中擁有一個像 `i8` 這樣類型的值時，編譯器確保它總是有一個有效的值。我們可以自信使用而無需判空。只有當使用 `Option<i8>`（或者任何用到的類型）的時候需要擔心可能沒有一個值，而編譯器會確保我們在使用值之前處理為空的情況。

換句話說，在對 `Option<T>` 進行 `T` 的運算之前必須轉換為 `T`。通常這能幫助我們捕獲空值最常見的問題之一：假設某值不為空但實際上為空的情況。

無需擔心錯過存在非空值的假設讓我們對代碼更加有信心，為了擁有一個可能為空的值，必須顯式的將其放入對應類型的 `Option<T>` 中。接著，當使用這個值時，必須明確的處理值為空的情況。任何地方一個值不是 `Option<T>` 類型的話，**可以** 安全的假設它的值不為空。這是 Rust 的一個有意為之的設計選擇，來限制空值的氾濫和增加 Rust 代碼的安全性。

那麼當有一個 `Option<T>` 的值時，如何從 `Some` 成員中取出 `T` 的值來使用它呢？`Option<T>` 枚舉擁有大量用於各種情況的方法：你可以查看[相關代碼][docs]<!-- ignore -->。熟悉 `Option<T>` 的方法將對你的 Rust 之旅提供巨大的幫助。

[docs]: https://doc.rust-lang.org/std/option/enum.Option.html

總的來說，為了使用 `Option<T>` 值，需要編寫處理每個成員的代碼。我們想要一些代碼只當擁有 `Some(T)` 值時運行，這些代碼允許使用其中的 `T`。也希望一些代碼當在 `None` 值時運行，這些代碼並沒有一個可用的 `T` 值。`match` 表達式就是這麼一個處理枚舉的控制流結構：它會根據枚舉的成員運行不同的代碼，這些代碼可以使用匹配到的值中的數據。