## 引用與借用

> [ch04-02-references-and-borrowing.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch04-02-references-and-borrowing.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

在上一部分的結尾處的使用元組的代碼是有問題的，我們需要將 `String` 返回給調用者函數這樣就可以在調用 `calculate_length` 後仍然可以使用 `String` 了，因為 `String` 先被移動到了 `calculate_length`。

下面是如何定義並使用一個（新的）`calculate_length` 函數，它以一個物件的 **引用** 作為參數而不是抓取值的所有權：

<span class="filename">Filename: src/main.rs</span>

```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

首先，注意變數聲明和函數返回值中的所有元組代碼都消失了。其次，注意我們傳遞 `&s1` 給 `calculate_length`，同時在函數定義中，我們抓取 `&String` 而不是 `String`。

這些 & 符號就是 **引用**，他們允許你使用值但不抓取它的所有權。圖 4-8 展示了一個圖解。

<img alt="&String s pointing at String s1" src="img/trpl04-05.svg" class="center" />

<span class="caption">圖 4-8：`&String s` 指向 `String s1`</span>

仔細看看這個函數調用：

```rust
# fn calculate_length(s: &String) -> usize {
#     s.len()
# }
let s1 = String::from("hello");

let len = calculate_length(&s1);
```

`&s1` 語法允許我們創建一個 **參考** 值 `s1` 的引用，但是並不擁有它。因為並不擁有這個值，當引用離開作用域它指向的值也不會被丟棄。

同理，函數簽名使用了 `&` 來表明參數 `s` 的類型是一個引用。讓我們增加一些解釋性的註解：

```rust
fn calculate_length(s: &String) -> usize { // s is a reference to a String
    s.len()
} // Here, s goes out of scope. But because it does not have ownership of what
  // it refers to, nothing happens.
```

變數 `s` 有效的作用域與函數參數的作用域一樣，不過當引用離開作用域後並不丟棄它指向的數據因為我們沒有所有權。函數使用引用而不是實際值作為參數意味著無需返回值來交還所有權，因為就不曾擁有所有權。

我們將抓取引用作為函數參數稱為 **借用**（*borrowing*）。正如現實生活中，如果一個人擁有某樣東西，你可以從他那裡借來。當你使用完畢，必須還回去。

如果我們嘗試修改借用的變數呢？嘗試列表 4-9 中的代碼。劇透：這行不通！

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let s = String::from("hello");

    change(&s);
}

fn change(some_string: &String) {
    some_string.push_str(", world");
}
```

<span class="caption">列表 4-9：嘗試修改借用的值</span>

這裡是錯誤：

```text
error: cannot borrow immutable borrowed content `*some_string` as mutable
 --> error.rs:8:5
  |
8 |     some_string.push_str(", world");
  |     ^^^^^^^^^^^
```

正如變數默認是不可變的，引用也一樣。不允許修改引用的值。

### 可變引用

可以通過一個小調整來修復在列表 4-9 代碼中的錯誤，在列表 4-9 的代碼中：

<span class="filename">文件名: src/main.rs</span>

```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

首先，必須將 `s` 改為 `mut`。然後必須創建一個可變引用 `&mut s` 和接受一個可變引用 `some_string: &mut String`。

不過可變引用有一個很大的限制：在特定作用域中的特定數據只能有一個可變引用，(譯註:且原本變數也不能使用)，否則失敗：

<span class="filename">文件名: src/main.rs</span>

```rust
let mut s = String::from("hello");

let r1 = &mut s;
let r2 = &mut s;
```

錯誤如下：

```text
error[E0499]: cannot borrow `s` as mutable more than once at a time
 --> borrow_twice.rs:5:19
  |
4 |     let r1 = &mut s;
  |                   - first mutable borrow occurs here
5 |     let r2 = &mut s;
  |                   ^ second mutable borrow occurs here
6 | }
  | - first borrow ends here
```

這個限制允許可變性，不過是以一種受限制的方式。新 Rustacean 們經常與此作鬥爭，因為大部分語言任何時候變數都是可變的。這個限制的好處是 Rust 可以在編譯時就避免數據競爭。

**數據競爭**（*data race*）是一種特定類型的競爭狀態，它可由這三個行為造成：

1. 兩個或更多指針同時訪問同一數據。
2. 至少有一個這樣的指針被用來寫入數據。
3. 不存在同步數據訪問的機制。

數據競爭會導致未定義行為，難以在運行時追蹤，並且難以診斷和修復；Rust 避免了這種情況的發生，因為它直接拒絕編譯存在數據競爭的代碼！

一如既往，可以使用大括號來創建一個新的作用域來允許擁有多個可變引用，只是不能 **同時** 擁有：

```rust
let mut s = String::from("hello");

{
    let r1 = &mut s;

} // r1 goes out of scope here, so we can make a new reference with no problems.

let r2 = &mut s;
```

當結合可變和不可變引用時有一個類似的規則存在。這些代碼會導致一個錯誤：

```rust
let mut s = String::from("hello");

let r1 = &s; // no problem
let r2 = &s; // no problem
let r3 = &mut s; // BIG PROBLEM
```

錯誤如下：

```text
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as
immutable
 --> borrow_thrice.rs:6:19
  |
4 |     let r1 = &s; // no problem
  |               - immutable borrow occurs here
5 |     let r2 = &s; // no problem
6 |     let r3 = &mut s; // BIG PROBLEM
  |                   ^ mutable borrow occurs here
7 | }
  | - immutable borrow ends here
```

哇哦！我們 **也** 不能在擁有不可變引用的同時擁有可變引用。不可變引用的用戶可不希望在它的眼皮底下值突然就被改變了！然而，多個不可變引用是沒有問題的因為沒有哪個只能讀取數據的人有能力影響其他人讀取到的數據。

即使這些錯誤有時是使人沮喪的。記住這是 Rust 編譯器在提早指出一個潛在的 bug（在編譯時而不是運行時）並明確告訴你問題在哪，而不是任由你去追蹤為何有時數據並不是你想像中的那樣。

### 懸垂引用

在存在指針的語言中，容易通過釋放內存時保留指向它的指針而錯誤地生成一個 **懸垂指針**（*dangling pointer*），所謂懸垂指針是其指向的內存可能已經被分配給其它持有者。相比之下，在 Rust 中編譯器確保引用永遠也不會變成懸垂狀態：當我們擁有一些數據的引用，編譯器確保數據不會在其引用之前離開作用域。

讓我們嘗試創建一個懸垂引用：

<span class="filename">Filename: src/main.rs</span>

```rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String {
    let s = String::from("hello");

    &s
}
```

這裡是錯誤：

```text
error[E0106]: missing lifetime specifier
 --> dangle.rs:5:16
  |
5 | fn dangle() -> &String {
  |                ^^^^^^^
  |
  = help: this function's return type contains a borrowed value, but there is no
    value for it to be borrowed from
  = help: consider giving it a 'static lifetime

error: aborting due to previous error
```

錯誤信息引用了一個我們還未涉及到的功能：**生命週期**（*lifetimes*）。第十章會詳細介紹生命週期。不過，如果你不理會生命週期的部分，錯誤信息確實包含了為什麼代碼是有問題的關鍵：

```text
this function's return type contains a borrowed value, but there is no value
for it to be borrowed from.
```

讓我們仔細看看我們的 `dangle` 代碼的每一步到底放生了什麼：

```rust
fn dangle() -> &String { // dangle returns a reference to a String

    let s = String::from("hello"); // s is a new String

    &s // we return a reference to the String, s
} // Here, s goes out of scope, and is dropped. Its memory goes away.
  // Danger!
```

因為 `s` 是在 `dangle` 創建的，當 `dangle` 的代碼執行完畢後，`s` 將被釋放。不過我們嘗試返回一個它的引用。這意味著這個引用會指向一個無效的 `String`！這可不對。Rust 不會允許我們這麼做的。

這裡的解決方法是直接返回 `String`：

```rust
fn no_dangle() -> String {
    let s = String::from("hello");

    s
}
```

這樣就可以沒有任何錯誤的運行了。所有權被移動出去，所以沒有值被釋放掉。

### 引用的規則

簡要的概括一下對引用的討論：

1. 在任意給定時間，**只能** 擁有如下中的一個：
  * 一個可變引用。
  * 任意數量的不可變引用。
2. 引用必須總是有效的。

接下來，我們來看看一種不同類型的引用：slice。

譯註:

```rust
fn manin() {
  let mut s1 = String::from("hello");
  let s2 = &mut s1;
  s2.push_str(", world!");
  println!("This is s2:{}", s2);
  println!("This is s1:{}", s1);
}
```
錯誤: [E0502]:cannot borrow `s1` as immutable because it is also borrowed as mutable

```rust
fn manin() {
  let mut s1 = String::from("hello");
  {
    let s2 = &mut s1;
    s2.push_str(", world!");
    println!("This is s2:{}", s2);
  }
  println!("This is s1:{}", s1);
}
```
正確:
```
This is s2:hello, world!
This is s1:hello, world!
```