## 什麼是所有權

> [ch04-01-what-is-ownership.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch04-01-what-is-ownership.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

Rust 的核心功能（之一）是 **所有權**（*ownership*）。雖然這個功能理解起來很直觀，不過它對rust 語言的其餘部分有著更深層的含義。

所有程序都必須管理他們運行時使用計算機內存的方式。一些語言中使用垃圾回收在程序運行過程中來時刻尋找不再被使用的內存；在另一些語言中，程式設計師必須親自分配和釋放內存。Rust 則選擇了第三種方式：內存被一個所有權系統管理，它擁有一系列的規則使編譯器在編譯時進行檢查。任何所有權系統的功能都不會導致運行時開銷。

因為所有權對很多程式設計師都是一個新概念，需要一些時間來適應。好消息是隨著你對 Rust 和所有權系統的規則越來越有經驗，你就越能自然地編寫出安全和高效的代碼。持之以恆！

當你理解了所有權系統，你就會對這個使 Rust 如此獨特的功能有一個堅實的基礎。在本章中，你將會通過一些例子來學習所有權，他們關注一個非常常見的數據結構：字符串。

<!-- PROD: START BOX -->

> ### 棧（Stack）與堆（Heap）
>
> 在很多語言中並不經常需要考慮到棧(譯註：可想成一疊盤子)與堆(譯註：像石子堆一樣，各堆間各自獨立)。不過在像 Rust 這樣的系統編程語言中，值是位於棧上還是堆上，更大的影響了語言的行為以及為何必須做出這樣的選擇。我們會在本章的稍後部分描述所有權與堆與棧相關的部分，所以這裡只是一個用來預熱的簡要解釋。
>
> 棧和堆都是代碼在運行時可供使用的部分內存，不過他們以不同的結構組成。棧以放入值的順序存儲並以相反順序取出值。這也被稱作 **後進先出**（*last in, first out*）。想像一下一疊盤子：當增加更多盤子時，把他們放在盤子堆的頂部，當需要盤子時，也從頂部拿走。不能從中間也不能從底部增加或拿走盤子！增加數據叫做 **進棧**（*pushing onto the stack*），而移出數據叫做 **出棧**（*popping off the stack*）。
>
> 操作棧是非常快的，因為它訪問數據的方式：永遠也不需要尋找一個位置放入新數據或者取出數據因為這個位置總是在棧頂。另一個使得棧快速的性質是棧中的所有數據都必須是一個已知的固定的大小。
>
> 相反對於在編譯時未知大小或大小可能變化的數據，可以把他們儲存在堆上。堆是缺乏組織的：當向堆放入數據時，我們請求一定大小的空間。操作系統在堆的某處找到一塊足夠大的空位，把它標記為已使用，並返回給我們一個它位置的指針。這個過程稱作 **在堆上分配內存**（*allocating on the heap*），並且有時這個過程就簡稱為 「分配」（allocating）。向棧中放入數據並不被認為是分配。因為指針是已知的固定大小的，我們可以將指針儲存在棧上，不過當需要實際數據時，必須訪問指針。
>
> 想像一下去餐館就坐吃飯。當進入時，你說明有幾個人，餐館員工會找到一個夠大的空桌子並領你們過去。如果有人來遲了，他們也可以通過詢問來找到你們坐在哪。
>
> 訪問堆上的數據要比訪問棧上的數據要慢因為必須通過指針來訪問。現代的處理器在內存中跳轉越少就越快（緩存）。繼續類比，假設有一台服務器來處理來自多個桌子的訂單。它在處理完一個桌子的所有訂單後再移動到下一個桌子是最有效率的。從桌子 A 獲取一個訂單，接著再從桌子 B 獲取一個訂單，然後再從桌子 A，然後再從桌子 B 這樣的流程會更加緩慢。出於同樣原因，處理器在處理的數據之間彼此較近的時候（比如在棧上）比較遠的時候（比如可能在堆上）能更好的工作。在堆上分配大量的空間也可能消耗時間。
>
> 當調用一個函數，傳遞給函數的值（包括可能指向堆上數據的指針）和函數的局部變數被壓入棧中。當函數結束時，這些值被移出棧。
>
> 記錄何處的代碼在使用堆上的什麼數據，最小化堆上的冗餘數據的數量以及清理堆上不再使用的數據以使不至於耗盡空間，這些所有的問題正是所有權系統要處理的。一旦理解了所有權，你就不需要經常考慮棧和堆了，不過理解如何管理堆內存可以幫助我們理解所有權為何存在以及為什麼以這種方式工作。

<!-- PROD: END BOX -->

### 所有權規則

首先，讓我們看一下所有權的規則。請記住它們，我們將講解一些它們的例子：

> 1. 每一個值都被它的 **所有者**（*owner*）變數擁有。
> 2. 值在任意時刻只能被一個所有者擁有。
> 3. 當所有者離開作用域，這個值將被丟棄。

### 變數作用域

我們已經在第二章完成過一個 Rust 程序的例子。現在我們已經掌握了基本語法，所以不會在之後的例子中附上 `fn main() {` 代碼了，所以如果你是一路跟過來的，必須手動將之後例子的代碼放入一個`main`函數中。為此，例子將顯得更加具體，使我們可以關注具體細節而不是樣板代碼。

作為所有權的第一個例子，我們看看一些變數的 **作用域**（*scope*）。作用域是一個項(原文:item) 在程序中有效的範圍。假如有一個這樣的變數：

```rust
let s = "hello";
```

變數`s`綁定到了一個字符串字面值，這個字符串值是硬編碼進我們程序代碼中的。這個變數從聲明的點開始直到當前 **作用域** 結束時都是有效的。列表 4-1 的註釋標明了變數`s`在哪裡是有效的：

```rust
{                      // s is not valid here, it's not yet declared
    let s = "hello";   // s is valid from this point forward

    // do stuff with s
}                      // this scope is now over, and s is no longer valid
```

<span class="caption">列表 4-1：一個變數和其有效的作用域</span>

換句話說，這裡有兩個重要的點：

1. 當`s` **進入作用域**，它就是有效的。
2. 這一直持續到它 **離開作用域** 為止。

目前為止，變數是否有效與作用域的關係跟其他編程語言是類似的。現在我們在此基礎上介紹 `String` 類型。

### `String` 類型

為了演示所有權的規則，我們需要一個比第三章講到的任何一個都要複雜的數據類型。之前出現的數據類型都是儲存在棧上的並且當離開作用域時被移出棧，不過我們需要尋找一個儲存在堆上的數據來探索 Rust 如何知道該在何時清理數據的。

這裡使用 `String` 作為例子並專注於 `String` 與所有權相關的部分。這些方面也同樣適用於其他標準庫提供的或你自己創建的複雜數據類型。在第八章會更深入地講解 `String`。

我們已經見過字符串字面值了，它被硬編碼進程序裡。字符串字面值是很方便的，不過他們並不總是適合所有需要使用文本的場景。原因之一就是他們是不可變的。另一個原因是不是所有字符串的值都能在編寫代碼時就知道：例如，如果想要獲取用戶輸入並儲存該怎麼辦呢？為此，Rust 有第二個字符串類型，`String`。這個類型儲存在堆上所以能夠儲存在編譯時未知大小的文本。可以用 `from` 從字符串字面值來創建 `String`，如下：

```rust
let s = String::from("hello");
```

這兩個冒號（`::`）運算符允許將特定的 `from` 函數置於 `String` 類型的命名空間（namespace）下而不需要使用類似 `string_from` 這樣的名字。在第五章的 「方法語法」（「Method Syntax」）部分會著重講解這個語法而且在第七章會講到模塊的命名空間。

這類字符串 *可以* 被修改：

```rust
let mut s = String::from("hello");

s.push_str(", world!"); // push_str() appends a literal to a String

println!("{}", s); // This will print `hello, world!`
```

那麼這裡有什麼區別呢？為什麼 `String` 可變而字面值卻不行呢？區別在於兩個類型對內存的處理上。

### 內存與分配

對於字符串字面值的情況，我們在編譯時就知道內容所以它直接被硬編碼進最終的可執行文件中，這使得字符串字面值快速和高效。不過這些屬性都只來源於其不可變性。不幸的是，我們不能為了每一個在編譯時未知大小的文本而將一塊內存放入二進制文件中而它的大小還可能隨著程序運行而改變。

對於 `String` 類型，為了支持一個可變，可增長的文本片段，需要在堆上分配一塊在編譯時未知大小的內存來存放內容。這意味著：

1. 內存必須在運行時向操作系統請求。
2. 需要一個當我們處理完 `String` 時將內存返回給操作系統的方法。

第一部分由我們完成：當調用 `String::from` 時，它的實現請求它需要的內存。這在編程語言中是非常通用的。

然而，第二部分實現起來就各有區別了。在有 **垃圾回收**（*garbage collector*，*GC*）的語言中， GC 用以記錄並清除不再使用的內存，而我們作為程式設計師，並不需要關心他們。沒有 GC 的話，識別出不再使用的內存並調用代碼顯式釋放就是我們程式設計師的責任了，正如請求內存的時候一樣。從歷史的角度上說正確處理內存回收曾經是一個困難的編程問題。如果忘記回收了會浪費內存。如果過早回收了，將會出現無效變數。如果重複回收，這也是個 bug。我們需要 `allocate` 和 `free` 一一對應。

Rust 採取了一個不同的策略：內存在擁有它的變數離開作用域後就被自動釋放。下面是列表 4-1 中作用域例子的一個使用 `String` 而不是字符串字面值的版本：

```rust
{
    let s = String::from("hello"); // s is valid from this point forward

    // do stuff with s
}                                  // this scope is now over, and s is no
                                   // longer valid
```

這裡是一個將 `String` 需要的內存返回給操作系統的很自然的位置：當 `s` 離開作用域的時候。當變數離開作用域，Rust 為其調用一個特殊的函數。這個函數叫做 `drop`，在這裡 `String` 的作者可以放置釋放內存的代碼。Rust 在結尾的 `}` 處自動調用 `drop`。

> 注意：在 C++ 中，這種 item 在生命週期結束時釋放資源的方法有時被稱作 **資源獲取即初始化**（*Resource Acquisition Is Initialization (RAII)*）。如果你使用過 RAII 模式的話應該對 Rust 的 `drop` 函數並不陌生。

這個模式對編寫 Rust 代碼的方式有著深遠的影響。它現在看起來很簡單，不過在更複雜的場景下代碼的行為可能是不可預測的，比如當有多個變數使用在堆上分配的內存時。現在讓我們探索一些這樣的場景。

#### 變數與數據交互的方式（一）：移動

Rust 中的多個變數以一種獨特的方式與同一數據交互。讓我們看看列表 4-2 中一個使用整型的例子：

```rust
let x = 5;
let y = x;
```

<span class="caption">列表 4-2：將變數 `x` 賦值給 `y`</span>

根據其他語言的經驗大致可以猜到這在幹什麼：「將 `5` 綁定到 `x`；接著生成一個值 `x` 的拷貝並綁定到 `y`」。現在有了兩個變數，`x` 和 `y`，都等於 `5`。這也正是事實上發生了的，因為正數是有已知固定大小的簡單值，所以這兩個 `5` 被放入了棧中。

現在看看這個 `String` 版本：

```rust
let s1 = String::from("hello");
let s2 = s1;
```

這看起來與上面的代碼非常類似，所以我們可能會假設他們的運行方式也是類似的：也就是說，第二行可能會生成一個 `s1` 的拷貝並綁定到 `s2` 上。不過，事實上並不完全是這樣。

為了更全面的解釋這個問題，讓我們看看圖 4-3 中 `String` 真正是什麼樣。`String` 由三部分組成，如圖左側所示：一個指向存放字符串內容內存的指針，一個長度，和一個容量。這一組數據儲存在棧上。右側則是堆上存放內容的內存部分。

<img alt="String in memory" src="img/trpl04-01.svg" class="center" style="width: 50%;" />

<span class="caption">圖 4-3：一個綁定到 `s1` 的擁有值 `"hello"` 的 `String` 的內存表現</span>

長度代表當前 `String` 的內容使用了多少字節的內存。容量是 `String` 從操作系統總共獲取了多少字節的內存。長度與容量的區別是很重要的，不過這在目前為止的場景中並不重要，所以可以暫時忽略容量。

當我們把 `s1` 賦值給 `s2`，`String` 的數據被覆制了，這意味著我們從棧上拷貝了它的指針、長度和容量。但我們並沒有複製堆上指針所指向的數據。換句話說，內存中數據的表現如圖 4-4 所示。

<img alt="s1 and s2 pointing to the same value" src="img/trpl04-02.svg" class="center" style="width: 50%;" />

<span class="caption">圖 4-4：變數 `s2` 的內存表現，它有一份 `s1` 指針、長度和容量的拷貝</span>

這個表現形式看起來 **並不像** 圖 4-5 中的那樣，它是如果 Rust 也拷貝了堆上的數據後內存看起來是怎麼樣的。如果 Rust 這麼做了，那麼操作 `s2 = s1` 在堆上數據比較大的時，可能會對運行時性能造成非常大的影響。

<img alt="s1 and s2 to two places" src="img/trpl04-03.svg" class="center" style="width: 50%;" />

<span class="caption">圖 4-5：另一個 `s2 = s1` 時可能的內存表現，如果 Rust 同時也拷貝了堆上的數據的話</span>

之前，我們提到過當變數離開作用域後 Rust 自動調用 `drop` 函數並清理變數的堆內存。不過圖 4-4 展示了兩個數據指針指向了同一位置。這就有了一個問題：當 `s2` 和 `s1` 離開作用域，他們都會嘗試釋放相同的內存。這是一個叫做 *double free* 的錯誤，也是之前提到過的內存安全性 bug 之一。兩次釋放（相同）內存會導致內存污染，它可能會導致潛在的安全漏洞。

為了確保內存安全，這種場景下 Rust 的處理有另一個細節值得注意。與其嘗試拷貝被分配的內存，Rust 則認為 `s1` 不再有效，因此 Rust 不需要在 `s1` 離開作用域後清理任何東西。看看在 `s2` 被創建之後嘗試使用 `s1` 會發生生麼：

```rust
let s1 = String::from("hello");
let s2 = s1;

println!("{}, world!", s1);
```

你會得到一個類似如下的錯誤，因為 Rust 禁止你使用無效的引用。

```text
error[E0382]: use of moved value: `s1`
 --> src/main.rs:4:27
  |
3 |     let s2 = s1;
  |         -- value moved here
4 |     println!("{}, world!", s1);
  |                            ^^ value used here after move
  |
  = note: move occurs because `s1` has type `std::string::String`,
which does not implement the `Copy` trait
```

如果你在其他語言中聽說過術語 「淺拷貝」（「shallow copy」）和 「深拷貝」（「deep copy」），那麼拷貝指針、長度和容量而不拷貝數據可能聽起來像淺拷貝。不過因為 Rust 同時使第一個變數無效化了，這個操作被稱為 **移動**（*move*），而不是淺拷貝。上面的例子可以解讀為 `s1` 被 **移動** 到了 `s2` 中。那麼具體發生了什麼如圖 4-6 所示。

<img alt="s1 moved to s2" src="img/trpl04-04.svg" class="center" style="width: 50%;" />

<span class="caption">圖 4-6：`s1` 無效化之後的內存表現</span>

這樣就解決了我們的麻煩！因為只有 `s2` 是有效的，當其離開作用域，它就釋放自己的內存，完畢。

另外，這裡還隱含了一個設計選擇：Rust 永遠也不會自動創建數據的 「深拷貝」。因此，任何 **自動** 的複製可以被認為對運行時性能影響較小。

#### 變數與數據交互的方式（二）：克隆

如果我們 **確實** 需要深度複製 `String` 中堆上的數據，而不僅僅是棧上的數據，可以使用一個叫做 `clone` 的通用函數。第五章會討論方法語法，不過因為方法在很多語言中是一個常見功能，所以之前你可能已經見過了。

這是一個實際使用 `clone` 方法的例子：

```rust
let s1 = String::from("hello");
let s2 = s1.clone();

println!("s1 = {}, s2 = {}", s1, s2);
```

這段代碼能正常運行，也是如何顯式產生圖 4-5 中行為的方式，這裡堆上的數據 **確實** 被覆制了。

當出現 `clone` 調用時，你知道一些特有的代碼被執行而且這些代碼可能相當消耗資源。它作為一個代表發生了不同的行為的可視化的標識。

#### 只在棧上的數據：拷貝

這裡還有一個沒有提到的小竅門。這些代碼使用了整型並且是有效的，他們是之前列表 4-2 中的一部分：

```rust
let x = 5;
let y = x;

println!("x = {}, y = {}", x, y);
```

他們似乎與我們剛剛學到的內容相牴觸：沒有調用 `clone`，不過 `x` 依然有效且沒有被移動到 `y` 中。

原因是像整型這樣的在編譯時已知大小的類型被整個儲存在棧上，所以拷貝其實際的值是快速的。這意味著沒有理由在創建變數 `y` 後使 `x` 無效。換句話說，這裡沒有深淺拷貝的區別，所以這裡調用 `clone` 並不會與通常的淺拷貝有什麼不同，我們可以不用管它。

Rust 有一個叫做 `Copy` trait 的特殊註解，可以用在類似整型這樣的儲存在棧上的類型（第十章詳細講解 trait）。如果一個類型擁有 `Copy` trait，一個舊的變數在（重新）賦值後仍然可用。Rust 不允許自身或其任何部分實現了 `Drop` trait 的類型使用`Copy` trait。如果我們對其值離開作用域時需要特殊處理的類型使用 `Copy` 註解，將會出現一個編譯時錯誤。關於如何為你的類型增加 `Copy` 註解，請閱讀附錄 C 中的可導出 trait。

那麼什麼類型是 `Copy` 的呢？可以查看給定類型的文檔來確認，不過作為一個通用的規則，任何簡單標量值的組合可以是 `Copy` 的，任何需要分配內存，或者本身就是某種形式資源的類型不會是 `Copy `的。如下是一些 `Copy` 的類型：

* 所有整數類型，比如 `u32`。
* 布爾類型，`bool`，它的值是 `true` 和 `false`。
* 所有浮點數類型，比如 `f64`。
* 元組，當且僅當其包含的類型也都是 `Copy` 的時候。`(i32, i32)` 是 `Copy` 的，不過 `(i32, String)` 就不是。

### 所有權與函數

將值傳遞給函數在語義上與給變數賦值相似。向函數傳遞值可能會移動或者複製，就像賦值語句一樣。列表 4-7 是一個帶有變數何時進入和離開作用域標註的例子：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let s = String::from("hello");  // s comes into scope.

    takes_ownership(s);             // s's value moves into the function...
                                    // ... and so is no longer valid here.
    let x = 5;                      // x comes into scope.

    makes_copy(x);                  // x would move into the function,
                                    // but i32 is Copy, so it's okay to still
                                    // use x afterward.

} // Here, x goes out of scope, then s. But since s's value was moved, nothing
  // special happens.

fn takes_ownership(some_string: String) { // some_string comes into scope.
    println!("{}", some_string);
} // Here, some_string goes out of scope and `drop` is called. The backing
  // memory is freed.

fn makes_copy(some_integer: i32) { // some_integer comes into scope.
    println!("{}", some_integer);
} // Here, some_integer goes out of scope. Nothing special happens.
```

<span class="caption">列表 4-7：帶有所有權和作用域標註的函數</span>

當嘗試在調用 `takes_ownership` 後使用 `s` 時，Rust 會拋出一個編譯時錯誤。這些靜態檢查使我們免於犯錯。試試在 `main` 函數中添加使用 `s` 和 `x` 的代碼來看看哪裡能使用他們，以及哪裡所有權規則會阻止我們這麼做。

### 返回值與作用域

返回值也可以轉移作用域。這裡是一個有與列表 4-7 中類似標註的例子：

<span class="filename">Filename: src/main.rs</span>

```rust
fn main() {
    let s1 = gives_ownership();         // gives_ownership moves its return
                                        // value into s1.

    let s2 = String::from("hello");     // s2 comes into scope.

    let s3 = takes_and_gives_back(s2);  // s2 is moved into
                                        // takes_and_gives_back, which also
                                        // moves its return value into s3.
} // Here, s3 goes out of scope and is dropped. s2 goes out of scope but was
  // moved, so nothing happens. s1 goes out of scope and is dropped.

fn gives_ownership() -> String {             // gives_ownership will move its
                                             // return value into the function
                                             // that calls it.

    let some_string = String::from("hello"); // some_string comes into scope.

    some_string                              // some_string is returned and
                                             // moves out to the calling
                                             // function.
}

// takes_and_gives_back will take a String and return one.
fn takes_and_gives_back(a_string: String) -> String { // a_string comes into
                                                      // scope.

    a_string  // a_string is returned and moves out to the calling function.
}
```

變數的所有權總是遵循相同的模式：將值賦值給另一個變數時移動它。當持有堆中數據值的變數離開作用域時，其值將通過 `drop` 被清理掉，除非數據被移動為另一個變數所有。

在每一個函數中都獲取並接著返回所有權是冗餘乏味的。如果我們想要函數使用一個值但不獲取所有權改怎麼辦呢？如果我們還要接著使用它的話，每次都傳遞出去再傳回來就有點煩人了，另外我們也可能想要返回函數體產生的任何（不止一個）數據。

使用元組來返回多個值是可能的，像這樣：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let s1 = String::from("hello");

    let (s2, len) = calculate_length(s1);

    println!("The length of '{}' is {}.", s2, len);
}

fn calculate_length(s: String) -> (String, usize) {
    let length = s.len(); // len() returns the length of a String.

    (s, length)
}
```

但是這不免有些形式主義，同時這離一個通用的觀點還有很長距離。幸運的是，Rust 對此提供了一個功能，叫做 **引用**（*references*）。