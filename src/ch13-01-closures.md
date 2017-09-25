## 閉包：可以捕獲環境的匿名函數

> [ch13-01-closures.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch13-01-closures.md)
> <br>
> commit 56352c28cf3fe0402fa5a7cba73890e314d720eb

Rust 的閉包是可以保存進變數或作為參數傳遞給其他函數的匿名函數。可以在一個地方創建閉包，然後在以不同的上下文中執行閉包運算。不同於函數，閉包允許捕獲調用者作用域中的值。我們將展示閉包的這些功能如何復用代碼和自定義行為。

### 使用閉包創建行為的抽象

讓我們看看一個展示儲存閉包並在之後執行如何有用的情形的例子。其間我們會討論閉包的語法、類型推斷和 trait。

這個假想的情況如下：我們在一個通過 app 生成自定義健身計畫的初創企業工作。其後端使用 Rust 編寫，而生成健身計畫的算法需要考慮很多不同的因素，比如用戶的年齡、身體質量指數（Body Mass Index）、用戶喜好、最近的健身活動和用戶指定的強度係數。本例中實際的算法並不重要。我們只希望在需要時調用算法，並且只希望調用一次，這樣就不會讓用戶等得太久。這裡將通過調用 `simulated_expensive_calculation` 函數來模擬調用假象的算法，如列表 13-1 所示，它會打印出 `calculating slowly...`，等待兩秒，並接著返回傳遞給它的數字：

<span class="filename">文件名: src/main.rs</span>

```rust
use std::thread;
use std::time::Duration;

fn simulated_expensive_calculation(intensity: i32) -> i32 {
    println!("calculating slowly...");
    thread::sleep(Duration::from_secs(2));
    intensity
}
```

<span class="caption">列表 13-1：一個用來代替假象計算的函數，它大約會執行兩秒</span>

接下來，`main` 函數中將會包含本例的健身 app 中的重要部分。這代表當用戶請求健身計畫時 app 會調用的代碼。因為與 app 前端的交互與閉包的使用並不相關，所以我們將硬編碼代表程序輸入的值並打印輸出。

程序的輸入有：

- 一個來自用戶的 `intensity` 數字，請求健身計畫時指定，它代表用戶喜好低強度還是高強度健身
- 一個會在健身計畫中生成變化的隨機數

程序的輸出將會是建議的鍛鍊計畫。

列表 13-2 展示了我們將要使用的 `main` 函數。處於簡單考慮這裡硬編碼了 `simulated_user_specified_value` 變數的值為 10 和 `simulated_random_number` 變數的值為 7；一個實際的程序會從 app 前端獲取強度係數並使用 `rand` crate 來生成隨機數，正如第二章的猜猜看遊戲所做的那樣。`main` 函數使用模擬的輸入值調用 `generate_workout` 函數：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let simulated_user_specified_value = 10;
    let simulated_random_number = 7;

    generate_workout(simulated_user_specified_value, simulated_random_number);
}
# fn generate_workout(intensity: i32, random_number: i32) {}
```

<span class="caption">列表 13-2：`main` 函數包含了用於 `generate_workout` 函數的模擬用戶輸入和模擬隨機數輸入</span>

這就是我們的執行上下文。列表 13-3 中的 `generate_workout` 函數包含我們最關心的 app 業務邏輯。本例中餘下的代碼修改都將在這個函數中：

<span class="filename">文件名: src/main.rs</span>

```rust
# use std::thread;
# use std::time::Duration;
#
# fn simulated_expensive_calculation(num: i32) -> i32 {
#     println!("calculating slowly...");
#     thread::sleep(Duration::from_secs(2));
#     num
# }
#
fn generate_workout(intensity: i32, random_number: i32) {
    if intensity < 25 {
        println!(
            "Today, do {} pushups!",
            simulated_expensive_calculation(intensity)
        );
        println!(
            "Next, do {} situps!",
            simulated_expensive_calculation(intensity)
        );
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!(
                "Today, run for {} minutes!",
                simulated_expensive_calculation(intensity)
            )
        }
    }
}
```

<span class="caption">列表 13-3：程序的業務邏輯，它根據輸入並調用 `simulated_expensive_calculation` 函數來打印出健身計畫</span>

列表 13-3 中的代碼有多處慢計算函數的調用。第一個 `if` 塊調用了 `simulated_expensive_calculation` 兩次，外部 `else` 中的 `if` 完全沒有調用它，`else` 中的 `else` 調用了它一次。

`generate_workout` 函數的合意的行為是首先檢查用戶需要低強度（由小於 25 的係數代表）鍛鍊還是高強度（25 或以上）鍛鍊。低強度鍛鍊計畫會根據由 `simulated_expensive_calculation` 函數所模擬的複雜算法建議一定數量的俯臥撐和仰臥起坐，此函數需要強度係數作為輸入。

如果用戶需要高強度鍛鍊，這裡有一些額外的邏輯：如果 app 生成的隨機數剛好是 3，app 相反會建議用戶稍做休息並補充水分。如果不是，則用戶會從複雜算法中得到數分鐘跑步的高強度鍛鍊計畫。

數據科學部門的同學告知我們必須對調用算法的方式做出一些改變。為了簡化做出這些改變的更新，我們將重構代碼來只調用 `simulated_expensive_calculation` 一次。同時還希望去掉目前多餘的連續兩次函數調用，並不希望在計算過程中增加任何其他此函數的調用。也就是說，我們不希望在完全無需其結果的情況調用函數，不過最終仍然需要調用函數一次。

有多種方法可以重構此程序。我們首先嘗試的是將重複的慢計算函數調用提取到一個變數中，如列表 13-4 所示：

<span class="filename">文件名: src/main.rs</span>

```rust
# use std::thread;
# use std::time::Duration;
#
# fn simulated_expensive_calculation(num: i32) -> i32 {
#     println!("calculating slowly...");
#     thread::sleep(Duration::from_secs(2));
#     num
# }
#
fn generate_workout(intensity: i32, random_number: i32) {
    let expensive_result =
        simulated_expensive_calculation(intensity);

    if intensity < 25 {
        println!(
            "Today, do {} pushups!",
            expensive_result
        );
        println!(
            "Next, do {} situps!",
            expensive_result
        );
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!(
                "Today, run for {} minutes!",
                expensive_result
            )
        }
    }
}
```

<span class="caption">列表 13-4：將 `simulated_expensive_calculation` 調用提取到一個位置，位於 `if` 塊之前並將結果儲存在變數 `expensive_result` 中</span>

這個修改統一了 `simulated_expensive_calculation` 調用並解決了第一個 `if` 塊中不必要的兩次調用函數的問題。不幸的是，現在所有的情況下都需要調用函數並等待結果，而內部 `if` 塊完全不需要其結果。

我們希望能夠在程序的一個位置指定某些代碼，並只在程序的某處實際需要結果的時候執行這些代碼。這正是閉包的用武之地！

### 閉包儲存了之後會執行的代碼

不同於總是在 `if` 塊之前調用 `simulated_expensive_calculation` 函數並儲存其結果，我們可以定義一個閉包並將其儲存在變數中，如列表 13-5 所示。實際上可以選擇將整個 `simulated_expensive_calculation` 函數體移動到這裡引入的閉包中：

<span class="filename">文件名: src/main.rs</span>

```rust
# use std::thread;
# use std::time::Duration;
#
let expensive_closure = |num| {
    println!("calculating slowly...");
    thread::sleep(Duration::from_secs(2));
    num
};
# expensive_closure(5);
```

<span class="caption">列表 13-5：使用慢計算的函數體定義一個閉包並儲存到變數 `expensive_closure` 中</span>

閉包定義是 `expensive_closure` 賦值的 `=` 之後的部分。閉包的定義以一對豎線（`|`）開始。在豎線中指定閉包的參數；之所以選擇這個語法是因為它與 Smalltalk 和 Ruby 的閉包定義類似。這個閉包有一個參數 `num`；如果有多於一個參數，可以使用逗號分隔，比如 `|param1, param2|`。

參數之後是存放閉包體的大括號。如果閉包體只有一行則大括號是可以省略的。大括號之後，需要用於 `let` 語句的分號。閉包體的最後一行（`num`）返回的值將是調用閉包時返回的值，因為最後一行沒有分號，正如函數體中的一樣。

注意這個 `let` 語句意味著 `expensive_closure` 包含一個匿名函數的 **定義**，不是調用匿名函數的 **返回值**。回憶一下使用閉包的原因是我們需要在一個位置定義代碼，儲存代碼，並在之後的位置實際調用它；期望調用的代碼現在儲存在 `expensive_closure` 中。

現在我們定義了閉包，可以改變 `if` 塊中的代碼來調用閉包以執行代碼並獲取結果值。調用閉包看起來非常類似調用函數；指定存放閉包定義的變數名並後跟包含期望使用的參數的括號，如列表 13-6 所示：

<span class="filename">文件名: src/main.rs</span>

```rust
# use std::thread;
# use std::time::Duration;
#
fn generate_workout(intensity: i32, random_number: i32) {
    let expensive_closure = |num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    };

    if intensity < 25 {
        println!(
            "Today, do {} pushups!",
            expensive_closure(intensity)
        );
        println!(
            "Next, do {} situps!",
            expensive_closure(intensity)
        );
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!(
                "Today, run for {} minutes!",
                expensive_closure(intensity)
            )
        }
    }
}
```

<span class="caption">列表 13-6：調用定義的 `expensive_closure`</span>

現在我們達成了將滿計算統一到一個地方的目標，並只會在需要結果的時候執行改代碼。然而，我們又重新引入了列表 13-3 中的問題：仍然在第一個 `if` 塊中調用了閉包兩次，這會調用慢計算兩次並使用戶多等待一倍的時間。可以通過在 `if` 塊中創建一個本地變數存放閉包調用的結果來解決這個問題，不過正因為使用了閉包還有另一個解決方案。稍後會回到這個方案上；首先討論一下為何閉包定義中和所涉及的 trait 中沒有類型註解。

### 閉包類型推斷和註解

閉包與由 `fn` 函數定義的函數有一些區別。第一是不要求像 `fn` 函數那樣在參數和返回值上註明類型。

函數中需要類型註解是因為他們是暴露給用戶的顯式接口的一部分。嚴格的定義這些接口對於保證所有人都認同函數使用和返回值的類型來說是很重要的。但是閉包並不用於這樣暴露在外的接口：他們儲存在變數中並被使用，不用命名他們或暴露給庫的用戶調用。

另外，閉包通常很短並只與對應相對任意的場景較小的上下文中。在這些有限制的上下文中，編譯器能可靠的推斷參數和返回值的類型，類似於它是如何能夠推斷大部分變數的類型一樣。強制在這些小的匿名函數中註明類型是很惱人的，並且與編譯器已知的信息存在大量的重複。

類似於變數，如果相比嚴格的必要性你更希望增加明確性並變得更囉嗦，可以選擇增加類型註解；為列表 13-4 中定義的閉包標註類型將看起來像列表 13-7 中的定義：

<span class="filename">文件名: src/main.rs</span>

```rust
# use std::thread;
# use std::time::Duration;
#
let expensive_closure = |num: i32| -> i32 {
    println!("calculating slowly...");
    thread::sleep(Duration::from_secs(2));
    num
};
```

<span class="caption">列表 13-7：為閉包的參數和返回值增加可選的類型註解</span>

有了類型註解閉包的語法就更類似函數了。如下是一個對其參數加一的函數的定義與擁有相同行為閉包語法的縱向對比。這裡增加了一些空格來對其相應部分。這展示了閉包語法如何類似於函數語法，除了使用豎線而不是括號以及幾個可選的語法：

```rust
fn  add_one_v1   (x: i32) -> i32 { x + 1 }
let add_one_v2 = |x: i32| -> i32 { x + 1 };
let add_one_v3 = |x|             { x + 1 };
let add_one_v4 = |x|               x + 1  ;
```

第一行展示了一個函數定義，而第二行展示了一個完整標註的閉包定義。第三行閉包定義中省略了類型註解，而第四行去掉了可選的大括號，因為閉包體只有一行。

閉包定義會為每個參數和返回值推斷一個具體類型。例如，列表 13-8 中展示了僅僅將參數作為返回值的簡短的閉包定義。除了作為示例的目的這個閉包並不是很實用。注意其定義並沒有增加任何類型註解：如果嘗試調用閉包兩次，第一次使用 `String` 類型作為參數而第一次使用 `i32`，則會得到一個錯誤：

<span class="filename">文件名: src/main.rs</span>

```rust
let example_closure = |x| x;

let s = example_closure(String::from("hello"));
let n = example_closure(5);
```

<span class="caption">列表 13-8：嘗試調用一個被推斷為兩個不同類型的閉包</span>

編譯器給出如下錯誤：

```text
error[E0308]: mismatched types
 --> src/main.rs
  |
  | let n = example_closure(5);
  |                         ^ expected struct `std::string::String`, found
  integral variable
  |
  = note: expected type `std::string::String`
             found type `{integer}`
```

第一次使用 `String` 值調用 `example_closure` 時，編譯器推斷 `x` 和此閉包返回值的類型為 `String`。接著這些類型被鎖定進閉包 `example_closure` 中，如果嘗試對同一閉包使用不同類型則會得到類型錯誤。

### 使用帶有泛型和 `Fn` trait 的閉包

回到我們的健身計畫生成 app ，在列表 13-6 中的代碼仍然調用了多於需要的慢計算閉包。在全部代碼中的每一個需要多個慢計算閉包結果的地方，可以將將結果保存進變數以供復用，這樣就可以使用變數而不是再次調用閉包。但是這樣就會有很多重複的保存結果變數的地方。

然而，因為擁有一個慢計算的閉包，我們還可以採取另一個解決方案。可以創建一個存放閉包和調用閉包結果的結構體。該結構體只會在需要結果時執行閉包，並會緩存結果值，這樣餘下的代碼就不必再負責保存結果並可以復用該值。你可能見過這種模式被稱 *memoization* 或 *lazy evaluation*。

為了讓結構體存放閉包，我們需要能夠指定閉包的類型。每一個閉包實例有其自己獨有的匿名類型：也就是說，即便兩個閉包有著相同的簽名，他們的類型仍然可以被認為是不同。為了定義使用閉包的結構體、枚舉或函數參數，需要像第十章討論的那樣使用泛型和 trait bound。

`Fn` 系列 trait 由標準庫提供。所有的閉包都實現了 trait `Fn`、`FnMut` 或 `FnOnce` 中的一個。在下一部分捕獲環境部分我們會討論這些 trait 的區別；在這個例子中可以使用 `Fn` trait。

為了滿足 `Fn` trait bound 我們增加了代表閉包所必須的參數和返回值類型的類型。在這個例子中，閉包有一個 `i32` 的參數並返回一個 `i32`，這樣所指定的 trait bound 就是 `Fn(i32) -> i32`。

列表 13-9 展示了存放了閉包和一個 Option 結果值的 `Cacher` 結構體的定義：

<span class="filename">文件名: src/main.rs</span>

```rust
struct Cacher<T>
    where T: Fn(i32) -> i32
{
    calculation: T,
    value: Option<i32>,
}
```

<span class="caption">列表 13-9：定義一個 `Cacher` 結構體來在 `calculation` 中存放閉包並在 `value` 中存放 Option 值</span>

結果提 `Cacher` 有一個泛型  `T` 的字段 `calculation`。`T` 的 trait bound 指定了 `T` 是一個使用 `Fn` 的閉包。任何我們希望儲存到 `Cacher` 實例的 `calculation` 字段的閉包必須有一個 `i32` 參數（由 `Fn` 之後的括號的內容指定）並必須返回一個 `i32`（由 `->` 之後的內容）。

`value` 是 `Option<i32>` 類型的。在執行閉包之前，`value` 將是 `None`。如果使用 `Cacher` 的代碼請求閉包的結果，這時會執行閉包並將結果儲存在 `value` 字段的 `Some` 成員中。接著如果代碼再次請求閉包的結果，這時不再執行閉包，而是會返回存放在 `Some` 成員中的結果。

剛才討論的油管 `value` 字段邏輯定義於列表 13-10：

<span class="filename">文件名: src/main.rs</span>

```rust
# struct Cacher<T>
#     where T: Fn(i32) -> i32
# {
#     calculation: T,
#     value: Option<i32>,
# }
#
impl<T> Cacher<T>
    where T: Fn(i32) -> i32
{
    fn new(calculation: T) -> Cacher<T> {
        Cacher {
            calculation,
            value: None,
        }
    }

    fn value(&mut self, arg: i32) -> i32 {
        match self.value {
            Some(v) => v,
            None => {
                let v = (self.calculation)(arg);
                self.value = Some(v);
                v
            },
        }
    }
}
```

<span class="caption">列表 13-10：一個 `Cacher` 的關聯函數 `new` 和管理緩存邏輯的 `value` 方法的實現</span>

`Cacher` 結構體的字段是私有的，因為我們希望 `Cacher` 管理這些值而不是任由調用代碼潛在的直接改變他們。`Cacher::new` 函數獲取一個泛型參數 `T`，它定義於 `impl` 塊上下文中並與 `Cacher`  結構體有著相同的 trait bound。`Cacher::new` 返回一個在 `calculation` 字段中存放了指定閉包和在 `value` 字段中存放了 `None` 值的 `Cacher` 實例，因為我們還未執行閉包。

當調用代碼需要閉包的執行結果時，不同於直接調用閉包，它會調用 `value` 方法。這個方法會檢查 `self.value` 是否已經有了一個 `Some` 的結果值；如果有，它返回 `Some` 中的值並不會再次執行閉包。

如果 `self.value` 是 `None`，則會調用 `self.calculation` 中儲存的閉包，將結果保存到 `self.value` 以便將來使用，並同時返回結果值。

列表 13-11 展示了如何在列表 13-6 的 `generate_workout` 函數中利用 `Cacher` 結構體：

<span class="filename">文件名: src/main.rs</span>

```rust
# use std::thread;
# use std::time::Duration;
#
# struct Cacher<T>
#     where T: Fn(i32) -> i32
# {
#     calculation: T,
#     value: Option<i32>,
# }
#
# impl<T> Cacher<T>
#     where T: Fn(i32) -> i32
# {
#     fn new(calculation: T) -> Cacher<T> {
#         Cacher {
#             calculation,
#             value: None,
#         }
#     }
#
#     fn value(&mut self, arg: i32) -> i32 {
#         match self.value {
#             Some(v) => v,
#             None => {
#                 let v = (self.calculation)(arg);
#                 self.value = Some(v);
#                 v
#             },
#         }
#     }
# }
#
fn generate_workout(intensity: i32, random_number: i32) {
    let mut expensive_result = Cacher::new(|num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        num
    });

    if intensity < 25 {
        println!(
            "Today, do {} pushups!",
            expensive_result.value(intensity)
        );
        println!(
            "Next, do {} situps!",
            expensive_result.value(intensity)
        );
    } else {
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!(
                "Today, run for {} minutes!",
                expensive_result.value(intensity)
            )
        }
    }
}
```

<span class="caption">列表 13-11：在 `generate_workout` 函數中利用 `Cacher` 結構體來抽象出緩存邏輯</span>

不同於直接將閉包保存進一個變數，我們保存一個新的 `Cacher` 實例來存放閉包。接著，在每一個需要結果的地方，調用 `Cacher` 實例的 `value` 方法。可以調用 `value` 方法任意多次，或者一次也不調用，而慢計算最多只會運行一次。嘗試使用列表 13-2 中的 `main` 函數來運行這段程序，並將 `simulated_user_specified_value` 和 `simulated_random_number` 變數中的值來驗證在所有情況下在多個 `if` 和 `else` 塊中，閉包打印的 `calculating slowly...` 只會在需要時出現並只會出現一次。

`Cacher` 負責確保不會調用超過所需的慢計算所需的邏輯，這樣 `generate_workout` 就可以專注業務邏輯了。值緩存是一種更加廣泛的實用行為，我們可能希望在代碼中的其他閉包中也使用他們。然而，目前 `Cacher` 的實現存在一些小問題，這使得在不同上下文中復用變得很困難。

第一個問題是 `Cacher` 實例假設對於 `value` 方法的任何 `arg` 參數值總是會返回相同的值。也就是說，這個 `Cacher` 的測試會失敗：

```rust
#[test]
fn call_with_different_values() {
    let mut c = Cacher::new(|a| a);

    let v1 = c.value(1);
    let v2 = c.value(2);

    assert_eq!(v2, 2);
}
```

這個測試使用返回傳遞給它的值的閉包創建了一個新的 `Cacher` 實例。使用為 1 的 `arg` 和為 2 的 `arg` 調用 `Cacher` 實例的 `value` 方法，同時我們期望使用為 2 的 `arg` 調用 `value` 會返回 2。

使用列表 13-9 和列表 13-10 的 `Cacher` 實現運行測試，它會在 `assert_eq!` 失敗並顯示如下信息：

```text
thread 'call_with_different_arg_values' panicked at 'assertion failed:
`(left == right)` (left: `1`, right: `2`)', src/main.rs
```

這裡的問題是第一次使用 1 調用 `c.value`，`Cacher` 實例將 `Some(1)` 保存進 `self.value`。在這之後，無論傳遞什麼值調用 `value`，它總是會返回 1。

嘗試修改 `Cacher` 存放一個哈希 map 而不是單獨一個值。哈希 map 的 key 將是傳遞進來的 `arg` 值，而 value 則是對應 key 調用閉包的結果值。相比之前檢查 `self.value` 是 `Some` 還是 `None` 值，現在 `value` 會在哈希 map 中尋找 `arg`，如果存在就返回它。如果不存在，`Cacher` 會調用閉包並將結果值保存在哈希 map 對應 `arg` 值的位置。

當前 `Cacher` 實現的另一個問題是它的應用被限制為只接受獲取一個 `i32` 值並返回一個 `i32` 值的閉包。比如說，我們可能需要能夠緩存一個獲取字符串 slice 並返回 `usize` 值的閉包的結果。請嘗試引入更多泛型參數來增加 `Cacher` 功能的靈活性。

### 閉包會捕獲其環境

在健身計畫生成器的例子中，我們只將閉包作為內聯匿名函數來使用。不過閉包還有另一個函數所沒有的功能：他們可以捕獲其環境並訪問定義他們的作用域的變數。

列表 13-12 有一個儲存在 `equal_to_x` 變數中閉包的例子，它使用了閉包環境中的變數 `x`：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let x = 4;

    let equal_to_x = |z| z == x;

    let y = 4;

    assert!(equal_to_x(y));
}
```

<span class="caption">列表 13-12：一個引用了其周圍作用域中變數的閉包示例</span>

這裡，即便 `x` 並不是 `equal_to_x` 的一個參數，`equal_to_x` 閉包也被允許使用變數 `x`，因為它與 `equal_to_x` 定義於相同的作用域。

函數則不能做到同樣的事，讓我們試試會發生什麼：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let x = 4;

    fn equal_to_x(z: i32) -> bool { z == x }

    let y = 4;

    assert!(equal_to_x(y));
}
```

這會得到一個錯誤：

```text
error[E0434]: can't capture dynamic environment in a fn item; use the || { ... }
closure form instead
 -->
  |
4 |     fn equal_to_x(z: i32) -> bool { z == x }
  |                                          ^
```

編譯器甚至會提示我們這只能用於閉包！

當閉包從環境中捕獲一個值，閉包會在閉包體中儲存這個值以供使用。這會使用內存並產生額外的開銷，當執行不會捕獲環境的更通用的代碼場景中我們不希望有這些開銷。因為函數從未允許捕獲環境，定義和使用函數也就從不會有這些額外開銷。

閉包可以通過三種方式捕獲其環境，他們直接對應函數的三種獲取參數的方式：獲取所有權，不可變借用和可變借用。這三種捕獲值的方式被編碼為如下三個 `Fn` trait：

* `FnOnce` 消費從周圍作用域捕獲的變數（閉包周圍的作用域被稱為其 **環境**，*environment*）。為了消費捕獲到的變數，閉包必須獲取其所有權並在定義閉包時將其移動進閉包。其名稱的 `Once` 部分是因為閉包不能多次獲取相同變數的所有權，所以它只能被調用一次。
* `Fn` 從其環境不可變的借用值
* `FnMut` 可變的借用值所以可以改變其環境

當創建一個閉包時，Rust 根據其如何使用環境中變數來推斷我們希望如何引用環境。在列表 13-12 中，`equal_to_x` 閉包不可變的借用了 `x`（所以 `equal_to_x` 使用 `Fn` trait），因為閉包體只需要讀取 `x` 的值。

如果我們希望強制閉包獲取其使用的環境值的所有權，可以在參數列表前使用 `move` 關鍵字。這在將閉包傳遞給新線程以便將數據移動到新線程中時最為實用。第十六章討論並發時會展示更多 `move` 閉包的例子，不過現在這裡修改了列表 13-12 中的代碼（作為演示），在閉包定義中增加 `move` 關鍵字並使用 vector 代替整型，因為整型可以被拷貝而不是移動：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let x = vec![1, 2, 3];

    let equal_to_x = move |z| z == x;

    println!("can't use x here: {:?}", x);

    let y = vec![1, 2, 3];

    assert!(equal_to_x(y));
}
```

這個例子並不能編譯：

```text
error[E0382]: use of moved value: `x`
 --> src/main.rs:6:40
  |
4 |     let equal_to_x = move |z| z == x;
  |                      -------- value moved (into closure) here
5 |
6 |     println!("can't use x here: {:?}", x);
  |                                        ^ value used here after move
  |
  = note: move occurs because `x` has type `std::vec::Vec<i32>`, which does not
    implement the `Copy` trait
```

`x` 被移動進了閉包，因為閉包使用 `move` 關鍵字定義。接著閉包獲取了 `x` 的所有權，同時 `main` 就不再允許使用它了。去掉 `println!` 即可修復問題。

大部分需要指定一個 `Fn` trait bound 的時候，可以從 `Fn` 開始，編譯器會根據閉包體中的情況告訴你是否需要 `FnMut` 或 `FnOnce`。

為了展示閉包作為函數參數時捕獲其環境的作用，讓我們移動到下一個主題：迭代器。