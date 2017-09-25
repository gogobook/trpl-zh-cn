## 導入命名

> [ch07-03-importing-names-with-use.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch07-03-importing-names-with-use.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

我們已經講到了如何使用模組名稱作為調用的一部分，來調用模組中的函數，如列表 7-6 中所示的 `nested_modules` 函數調用。

<span class="filename">文件名: src/main.rs</span>

```rust
pub mod a {
    pub mod series {
        pub mod of {
            pub fn nested_modules() {}
        }
    }
}

fn main() {
    a::series::of::nested_modules();
}
```

<span class="caption">列表 7-6：通過完全指定模組中的路徑來調用函數</span>

如你所見，指定函數的完全限定名稱可能會非常冗長。所幸 Rust 有一個關鍵字使得這些調用顯得更簡潔。

### 使用 `use` 的簡單導入

Rust 的 `use` 關鍵字的工作是縮短冗長的函數調用，通過將想要調用的函數所在的模組引入到作用域中。這是一個將 `a::series::of` 模組導入一個二進制 crate 的根作用域的例子：

<span class="filename">文件名: src/main.rs</span>

```rust
pub mod a {
    pub mod series {
        pub mod of {
            pub fn nested_modules() {}
        }
    }
}

use a::series::of;

fn main() {
    of::nested_modules();
}
```

`use a::series::of;` 這一行的意思是每當想要引用 `of` 模組時，不必使用完整的 `a::series::of` 路徑，可以直接使用 `of`。

`use` 關鍵字只將指定的模組引入作用域；它並不會將其子模組也引入。這就是為什麼想要調用 `nested_modules` 函數時仍然必須寫成 `of::nested_modules`。

也可以將函數本身引入到作用域中，通過如下在 `use` 中指定函數的方式：

```rust
pub mod a {
    pub mod series {
        pub mod of {
            pub fn nested_modules() {}
        }
    }
}

use a::series::of::nested_modules;

fn main() {
    nested_modules();
}
```

這使得我們可以忽略所有的模組並直接引用函數。

因為枚舉也像模組一樣組成了某種命名空間，也可以使用 `use` 來導入枚舉的成員。對於任何類型的 `use` 語句，如果從一個命名空間導入多個項，可以使用大括號和逗號來列舉他們，像這樣：

```rust
enum TrafficLight {
    Red,
    Yellow,
    Green,
}

use TrafficLight::{Red, Yellow};

fn main() {
    let red = Red;
    let yellow = Yellow;
    let green = TrafficLight::Green;
}
```

### 使用 `*` 的全局引用導入

為了一次導入某個命名空間的所有項，可以使用 `*` 語法。例如：

```rust
enum TrafficLight {
    Red,
    Yellow,
    Green,
}

use TrafficLight::*;

fn main() {
    let red = Red;
    let yellow = Yellow;
    let green = Green;
}
```

`*` 被稱為 **全局導入**（*glob*），它會導入命名空間中所有可見的項。全局導入應該保守的使用：他們是方便的，但是也可能會引入多於你預期的內容從而導致命名衝突。

### 使用 `super` 訪問父模組

正如我們已經知道的，當創建一個庫 crate 時，Cargo 會生成一個 `tests` 模組。現在讓我們來深入瞭解一下。在 `communicator` 項目中，打開 *src/lib.rs*。

<span class="filename">文件名: src/lib.rs</span>

```rust
pub mod client;

pub mod network;

#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
    }
}
```

第十一章會更詳細的解釋測試，不過其部分內容現在應該可以理解了：有一個叫做 `tests` 的模組緊鄰其他模組，同時包含一個叫做 `it_works` 的函數。即便存在一些特殊註解，`tests` 也不過是另外一個模組！所以我們的模組層次結構看起來像這樣：

```text
communicator
 ├── client
 ├── network
 |   └── client
 └── tests
```

測試是為了檢驗庫中的代碼而存在的，所以讓我們嘗試在 `it_works` 函數中調用 `client::connect` 函數，即便現在不準備測試任何功能：

<span class="filename">文件名: src/lib.rs</span>

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        client::connect();
    }
}
```

使用 `cargo test` 命令運行測試：

```text
$ cargo test
   Compiling communicator v0.1.0 (file:///projects/communicator)
error[E0433]: failed to resolve. Use of undeclared type or module `client`
 --> src/lib.rs:9:9
  |
9 |         client::connect();
  |         ^^^^^^^^^^^^^^^ Use of undeclared type or module `client`
```

編譯失敗了，不過為什麼呢？並不需要像 *src/main.rs* 那樣將 `communicator::` 置於函數前，因為這裡肯定是在 `communicator` 庫 crate 之內的。之所以失敗的原因是路徑是相對於當前模組的，在這裡就是 `tests`。唯一的例外就是 `use` 語句，它預設是相對於 crate 根模組的。我們的 `tests` 模組需要 `client` 模組位於其作用域中！

那麼如何在模組層次結構中回退一級模組，以便在 `tests` 模組中能夠調用 `client::connect`函數呢？在 `tests` 模組中，要麼可以在開頭使用雙冒號來讓 Rust 知道我們想要從根模組開始並列出整個路徑：

```rust
::client::connect();
```

要麼可以使用 `super` 在層級中抓取當前模組的上一級模組：

```rust
super::client::connect();
```

在這個例子中這兩個選擇看不出有多麼大的區別，不過隨著模組層次的更加深入，每次都從根模組開始就會顯得很長了。在這些情況下，使用 `super` 來抓取當前模組的同級模組是一個好的捷徑。再加上，如果在代碼中的很多地方指定了從根開始的路徑，那麼當通過移動子樹或到其他位置來重新排列模組時，最終就需要更新很多地方的路徑，這就非常乏味無趣了。

在每一個測試中總是不得不編寫 `super::` 也會顯得很惱人，不過你已經見過解決這個問題的利器了：`use`！`super::` 的功能改變了提供給 `use` 的路徑，使其不再相對於根模組而是相對於父模組。

為此，特別是在 `tests` 模組，`use super::something` 是常用的手段。所以現在的測試看起來像這樣：

<span class="filename">文件名: src/lib.rs</span>

```rust
#[cfg(test)]
mod tests {
    use super::client;

    #[test]
    fn it_works() {
        client::connect();
    }
}
```

如果再次運行`cargo test`，測試將會通過而且測試結果輸出的第一部分將會是：

```text
$ cargo test
   Compiling communicator v0.1.0 (file:///projects/communicator)
     Running target/debug/communicator-92007ddb5330fa5a

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

## 總結

現在你掌握了組織代碼的核心科技！利用他們將相關的代碼組合在一起、防止代碼文件過長並將一個整潔的公有 API 展現給庫的用戶。

接下來，讓我們看看一些標準庫提供的集合數據類型，你可以利用他們編寫出漂亮整潔的代碼。