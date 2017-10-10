## 定義並實例化結構體

> [ch05-01-defining-structs.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch05-01-defining-structs.md)
> <br>
> commit 56352c28cf3fe0402fa5a7cba73890e314d720eb

我們在第三章討論過，結構體與元組類似。就像元組，結構體的每一部分可以是不同類型。不同於元組，需要命名各部分數據以便能清楚的表明其值的意義。由於有了這些名字使得結構體比元組更靈活：不需要依賴順序來指定或訪問實例中的值。

為了定義結構體，通過 `struct` 關鍵字並為整個結構體提供一個名字。結構體的名字需要描述它所組合的數據的意義。接著，在大括號中，定義每一部分數據的名字，他們被稱作 **字段**（*field*），並定義字段類型。例如，示例 5-1 展示了一個儲存用戶帳號信息的結構體：

```rust
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}
```

<span class="caption">示例 5-1：`User` 結構體定義</span>

一旦定義了結構體後為了使用它，通過為每個字段指定具體值來創建這個結構體的 **實例**。創建一個實例需要以結構體的名字開頭，接著在大括號中使用 `key: value` 對的形式提供字段，其中 key 是字段的名字而 value 是需要儲存在字段中的數據值。這時字段的順序並不必要與在結構體中聲明他們的順序一致。換句話說，結構體的定義就像一個這個類型的通用模板，而實例則會在這個模板中放入特定數據來創建這個類型的值。例如，可以像示例 5-2 這樣來聲明一個特定的用戶：

```rust
# struct User {
#     username: String,
#     email: String,
#     sign_in_count: u64,
#     active: bool,
# }
#
let user1 = User {
    email: String::from("someone@example.com"),
    username: String::from("someusername123"),
    active: true,
    sign_in_count: 1,
};
```

<span class="caption">示例 5-2：創建 `User` 結構體的實例</span>

為了從結構體中抓取某個值，可以使用點號。如果我們只想要用戶的郵箱地址，可以用 `user1.email`。要更改結構體中的值，如果結構體的實例是可變的，我們可以使用點號並對對應的字段賦值。示例 5-3 展示了如何改變一個可變的 `User` 實例 `email` 字段的值：

```rust
# struct User {
#     username: String,
#     email: String,
#     sign_in_count: u64,
#     active: bool,
# }
#
let mut user1 = User {
    email: String::from("someone@example.com"),
    username: String::from("someusername123"),
    active: true,
    sign_in_count: 1,
};

user1.email = String::from("anotheremail@example.com");
```

<span class="caption">示例 5-3：改變 `User` 結構體 `email` 字段的值</span>

與其他任何表達式一樣，我們可以在函數體的最後一個表達式構造一個結構體，從函數隱式的返回一個結構體的新實例。表 5-4 顯示了一個返回帶有給定的 `email` 與 `username` 的 `User` 結構體的實例的 `build_user` 函數。`active` 字段的值為 `true`，並且 `sign_in_count` 的值為 `1`。

```rust
# struct User {
#     username: String,
#     email: String,
#     sign_in_count: u64,
#     active: bool,
# }
#
fn build_user(email: String, username: String) -> User {
    User {
        email: email,
        username: username,
        active: true,
        sign_in_count: 1,
    }
}
```

<span class="caption">示例 5-4：`build_user` 函數抓取 email 和用戶名並返回 `User` 實例</span>

不過，重複 `email` 字段與 `email` 變數的名字，同樣的對於`username`，感覺有一點無趣。將函數參數起與結構體字段相同的名字是可以理解的，但是如果結構體有更多字段，重複他們是十分煩人的。幸運的是，這裡有一個方便的語法！

### 變數與字段同名時的字段初始化語法

如果有變數與字段同名的話，你可以使用 **字段初始化語法**（*field init shorthand*）。這可以讓創建新的結構體實例的函數更為簡練。

在示例 5-4 中，名為 `email` 與 `username` 的參數與結構體 `User` 的字段 `email` 和 `username` 同名。因為名字相同，我們可以寫出不重複 `email` 和 `username` 的 `build_user` 函數，如示例 5-5 所示。 這個版本的函數與示例 5-4 中代碼的行為完全相同。這個字段初始化語法可以讓這類代碼更簡潔，特別是當結構體有很多字段的時候。

```rust
# struct User {
#     username: String,
#     email: String,
#     sign_in_count: u64,
#     active: bool,
# }
#
fn build_user(email: String, username: String) -> User {
    User {
        email,
        username,
        active: true,
        sign_in_count: 1,
    }
}
```

<span class="caption">示例 5-5：`build_user` 函數使用了字段初始化語法，因為 `email` 和 `username` 參數與結構體字段同名</span>

### 使用結構體更新語法從其他物件創建物件

可以從老的物件創建新的物件常常是很有幫助的，即復用大部分老物件的值並只改變一部分。示例 5-6 展示了一個設置 `email` 與 `username` 的值但其餘字段使用與示例 5-2 中 `user1` 實例相同的值以創建新的 `User` 實例 `user2` 的例子：

```rust
# struct User {
#     username: String,
#     email: String,
#     sign_in_count: u64,
#     active: bool,
# }
#
# let user1 = User {
#     email: String::from("someone@example.com"),
#     username: String::from("someusername123"),
#     active: true,
#     sign_in_count: 1,
# };
#
let user2 = User {
    email: String::from("another@example.com"),
    username: String::from("anotherusername567"),
    active: user1.active,
    sign_in_count: user1.sign_in_count,
};
```

<span class="caption">示例 5-6：創建 `User` 新實例，`user2`，並將一些字段的值設置為 `user1` 同名字段的值</span>

**結構體更新語法**（*struct update syntax*）可以利用更少的代碼獲得與示例 5-6 相同的效果。結構體更新語法利用 `..` 以指定未顯式設置的字段應有與給定實例對應字段相同的值。示例 5-7 中的代碼同樣地創建了有著不同的 `email` 與 `username` 值但 `active` 和 `sign_in_count` 字段與 `user1` 相同的實例 `user2`：

```rust
# struct User {
#     username: String,
#     email: String,
#     sign_in_count: u64,
#     active: bool,
# }
#
# let user1 = User {
#     email: String::from("someone@example.com"),
#     username: String::from("someusername123"),
#     active: true,
#     sign_in_count: 1,
# };
#
let user2 = User {
    email: String::from("another@example.com"),
    username: String::from("anotherusername567"),
    ..user1
};
```

<span class="caption">示例 5-7：使用結構體更新語法為 `User` 實例設置新的 `email` 和 `username` 值，但使用 `user1` 變數中剩下字段的值</span>

### 使用沒有命名字段的元組結構體創建不同的類型

也可以定義與元組相像的結構體，稱為 **元組結構體**（*tuple structs*），有著結構體名稱提供的含義，但沒有具體的字段名只有字段的類型。元組結構體的定義仍然以`struct` 關鍵字與結構體名稱，接下來是元組的類型。如以下是命名為 `Color`  與`Point` 的元組結構體的定義與使用：

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

let black = Color(0, 0, 0);
let origin = Point(0, 0, 0);
```

注意 `black` 和 `origin` 變數有不同的類型，因為他們是不同的元組結構體的實例。我們定義的每一個結構體有著自己的類型，即使結構體中的字段有相同的類型。在其他方面，元組結構體類似我們在第三章提到的元組。

### 沒有任何字段的類單元結構體

我們也可以定義一個沒有任何字段的結構體！他們被稱為 **類單元結構體**（*unit-like structs*）因為他們類似於 `()`，即 unit 類型。類單元結構體常常在你想要在某個類型上實現 trait 但不需要在類型內存儲數據的時候發揮作用。我們將在第十章介紹 trait。

> ## 結構體數據的所有權
>
> 在示例 5-1 中的 `User` 結構體的定義中，我們使用了自身擁有所有權的 `String` 類型而不是 `&str` 字符串 slice 類型。這是一個有意而為之的選擇，因為我們想要這個結構體擁有它所有的數據，為此只要整個結構體是有效的話其數據也應該是有效的。
>
> 可以使結構體儲存被其他物件擁有的數據的引用，不過這麼做的話需要用上 **生命週期**（*lifetimes*），這是一個第十章會討論的 Rust 功能。生命週期確保結構體引用的數據有效性跟結構體本身保持一致。如果你嘗試在結構體中儲存一個引用而不指定生命週期，比如這樣：
>
> <span class="filename">文件名: src/main.rs</span>
>
> ```rust
> struct User {
>     username: &str,
>     email: &str,
>     sign_in_count: u64,
>     active: bool,
> }
>
> fn main() {
>     let user1 = User {
>         email: "someone@example.com",
>         username: "someusername123",
>         active: true,
>         sign_in_count: 1,
>     };
> }
> ```
>
> 編譯器會抱怨它需要生命週期標識符：
>
> ```text
> error[E0106]: missing lifetime specifier
>  -->
>   |
> 2 |     username: &str,
>   |               ^ expected lifetime parameter
>
> error[E0106]: missing lifetime specifier
>  -->
>   |
> 3 |     email: &str,
>   |            ^ expected lifetime parameter
> ```
>
> 第十章會講到如何修復這個問題以便在結構體中儲存引用，不過現在，通過從像 `&str` 這樣的引用切換到像 `String` 這類擁有所有權的類型來修改修改這個錯誤。