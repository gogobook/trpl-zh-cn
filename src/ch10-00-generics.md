# 泛型、trait 和生命週期

> [ch10-00-generics.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch10-00-generics.md)
> <br>
> commit f65676e17d7fc4c0c7cd7275a7bf15447364831a

每一個編程語言都有高效的處理重複概念的工具；在 Rust 中工具之一就是 **泛型**（*generics*）。泛型是具體類型或其他屬性的抽象替代。我們可以表達泛型的屬性，比如他們的行為或如何與其他泛型相關聯，而不需要在編寫和編譯代碼時知道他們在這裡實際上代表什麼。

同理為了編寫一份可以用於多種具體值的代碼，函數並不知道其參數為何值，這時就可以讓函數抓取泛型而不是像 `i32` 或 `String` 這樣的具體值。我們已經使用過第六章的 `Option<T>`，第八章的 `Vec<T>` 和 `HashMap<K, V>`，以及第九章的 `Result<T, E>` 這些泛型了。本章會探索如何使用泛型定義我們自己自己的類型、函數和方法！

首先，我們將回顧一下提取函數以減少代碼重複的機制。接著使用一個只在參數類型上不同的泛型函數來實現相同的功能。我們也會講到結構體和枚舉定義中的泛型。

之後，我們討論 *trait*，這是一個定義泛型行為的方法。trait 可以與泛型結合來將泛型限制為擁有特定行為的類型，而不是任意類型。

最後介紹 **生命週期**（*lifetimes*），它是一類允許我們向編譯器提供引用如何相互關聯的泛型。Rust 的生命週期功能允許在很多場景下借用值的同時仍然使編譯器能夠檢查這些引用的有效性。

## 提取函數來減少重複

在介紹泛型語法之前，首先來回顧一個不使用泛型的處理重複的技術：提取一個函數。當熟悉了這個技術以後，我們將使用相同的機制來提取一個泛型函數！如同你識別出可以提取到函數中重複代碼那樣，你也會開始識別出能夠使用泛型的重複代碼。

考慮一下這個尋找列表中最大值的小程序，如列表 10-1 所示：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let mut largest = number_list[0];

    for number in number_list {
        if number > largest {
            largest = number;
        }
    }

    println!("The largest number is {}", largest);
#  assert_eq!(largest, 100);
}
```

<span class="caption">列表 10-1：在一個數字列表中尋找最大值的函數</span>

這段代碼抓取一個整型列表，存放在變數 `number_list` 中。它將列表的第一項放入了變數 `largest` 中。接著遍歷了列表中的所有數字，如果當前值大於 `largest` 中儲存的值，將 `largest` 替換為這個值。如果當前值小於目前為止的最大值，`largest` 保持不變。當列表中所有值都被考慮到之後，`largest` 將會是最大值，在這裡也就是 100。

如果需要在兩個不同的列表中尋找最大值，我們可以重複列表 10-1 中的代碼，這樣程序中就會存在兩段相同邏輯的代碼，如列表 10-2 所示：

<span class="filename">Filename: src/main.rs</span>

```rust
fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let mut largest = number_list[0];

    for number in number_list {
        if number > largest {
            largest = number;
        }
    }

    println!("The largest number is {}", largest);

    let number_list = vec![102, 34, 6000, 89, 54, 2, 43, 8];

    let mut largest = number_list[0];

    for number in number_list {
        if number > largest {
            largest = number;
        }
    }

    println!("The largest number is {}", largest);
}
```

<span class="caption">列表 10-2：尋找 **兩個** 數字列表最大值的代碼</span>

雖然代碼能夠執行，但是重複的代碼是冗餘且容易出錯的，並且意味著當更新邏輯時需要修改多處地方的代碼。

<!-- Are we safe assuming the reader will be familiar with the term
"abstraction" in this context, or do we want to give a brief definition? -->
<!-- Yes, our audience will be familiar with this term. /Carol -->

為了消除重複，我們可以創建一層抽象，在這個例子中將表現為一個抓取任意整型列表作為參數並對其進行處理的函數。這將增加代碼的簡潔性並讓我們將表達和推導尋找列表中最大值的這個概念與使用這個概念的特定位置相互獨立。

在列表 10-3 的程序中將尋找最大值的代碼提取到了一個叫做 `largest` 的函數中。這個程序可以找出兩個不同數字列表的最大值，不過列表 10-1 中的代碼只存在於一個位置：

<span class="filename">文件名: src/main.rs</span>

```rust
fn largest(list: &[i32]) -> i32 {
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
#    assert_eq!(result, 100);

    let number_list = vec![102, 34, 6000, 89, 54, 2, 43, 8];

    let result = largest(&number_list);
    println!("The largest number is {}", result);
#    assert_eq!(result, 6000);
}
```

<span class="caption">列表 10-3：抽象後的尋找兩個數字列表最大值的代碼</span>

這個函數有一個參數 `list`，它代表會傳遞給函數的任何具體的 `i32`值的 slice。函數定義中的 `list` 代表任何 `&[i32]`。當調用 `largest` 函數時，其代碼實際上運行於我們傳遞的特定值上。

從列表 10-2 到列表 10-3 中涉及的機制經歷了如下幾步：

1. 我們注意到了重複代碼。
2. 我們將重複代碼提取到了一個函數中，並在函數簽名中指定了代碼中的輸入和返回值。
3. 我們將兩個具體的存在重複代碼的位置替換為了函數調用。

在不同的場景使用不同的方式，我們也可以利用相同的步驟和泛型來減少重複代碼。與函數體中現在作用於一個抽象的 `list` 而不是具體值一樣，使用泛型的代碼也會作用於抽象類型。支持泛型背後的概念與你已經瞭解的支持函數的概念是一樣的，不過是實現方式不同。

如果我們有兩個函數，一個尋找一個 `i32` 值的 slice 中的最大項而另一個尋找 `char` 值的 slice 中的最大項該怎麼辦？該如何消除重複呢？讓我們拭目以待！