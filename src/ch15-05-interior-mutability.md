## `RefCell<T>`和內部可變性模式

> [ch15-05-interior-mutability.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch15-05-interior-mutability.md)
> <br>
> commit 3f2a1bd8dbb19cc48b210fc4fb35c305c8d81b56

**內部可變性**（*Interior mutability*）是 Rust 中的一個設計模式，它允許你即使在有不可變引用時改變數據，這通常是借用規則所不允許。內部可變性模式涉及到在數據結構中使用`unsafe`代碼來模糊 Rust 通常的可變性和借用規則。我們還未講到不安全代碼；第十九章會學習他們。內部可變性模式用於當你可以確保代碼在運行時也會遵守借用規則，哪怕編譯器也不能保證的情況。引入的`unsafe`代碼將被封裝進安全的 API 中，而外部類型仍然是不可變的。

讓我們通過遵循內部可變性模式的`RefCell<T>`類型來開始探索。

###  `RefCell<T>`擁有內部可變性

不同於`Rc<T>`，`RefCell<T>`代表其數據的唯一的所有權。那麼是什麼讓`RefCell<T>`不同於像`Box<T>`這樣的類型呢？回憶一下第四章所學的借用規則：

1. 在任意給定時間，**只能**擁有如下中的一個：
  * 一個可變引用。
  * 任意屬性的不可變引用。
2. 引用必須總是有效的。

對於引用和`Box<T>`，借用規則的不可變性作用於編譯時。對於`RefCell<T>`，這些不可變性作用於**運行時**。對於引用，如果違反這些規則，會得到一個編譯錯誤。而對於`RefCell<T>`，違反這些規則會`panic!`。

Rust 編譯器執行的靜態分析天生是保守的。代碼的一些屬性則不可能通過分析代碼發現：其中最著名的就是停機問題（停機問題），這超出了本書的範疇，不過如果你感興趣的話這是一個值得研究的有趣主題。

因為一些分析是不可能的，Rust 編譯器在其不確定的時候甚至都不嘗試猜測，所以說它是保守的而且有時會拒絕事實上不會違反 Rust 保證的正確的程序。換句話說，如果 Rust 接受不正確的程序，那麼人們也就不會相信 Rust 所做的保證了。如果 Rust 拒絕正確的程序，會給程式設計師帶來不便，但不會帶來災難。`RefCell<T>`正是用於當你知道代碼遵守借用規則，而編譯器不能理解的時候。

類似於`Rc<T>`，`RefCell<T>`只能用於單線程場景。在並發章節會介紹如何在多線程程序中使用`RefCell<T>`的功能。現在所有你需要知道的就是如果嘗試在多線程上下文中使用`RefCell<T>`，會得到一個編譯錯誤。

對於引用，可以使用`&`和`&mut`語法來分別創建不可變和可變的引用。不過對於`RefCell<T>`，我們使用`borrow`和`borrow_mut`方法，它是`RefCell<T>`擁有的安全 API 的一部分。`borrow`返回`Ref`類型的智能指針，而`borrow_mut`返回`RefMut`類型的智能指針。這兩個類型實現了`Deref`所以可以被當作常規引用處理。`Ref`和`RefMut`動態的借用所有權，而他們的`Drop`實現也動態的釋放借用。

列表 15-14 展示了如何使用`RefCell<T>`來使函數不可變的和可變的借用它的參數。注意`data`變數使用`let data`而不是`let mut data`來聲明為不可變的，而`a_fn_that_mutably_borrows`則允許可變的借用數據並修改它！

<span class="filename">Filename: src/main.rs</span>

```rust
use std::cell::RefCell;

fn a_fn_that_immutably_borrows(a: &i32) {
    println!("a is {}", a);
}

fn a_fn_that_mutably_borrows(b: &mut i32) {
    *b += 1;
}

fn demo(r: &RefCell<i32>) {
    a_fn_that_immutably_borrows(&r.borrow());
    a_fn_that_mutably_borrows(&mut r.borrow_mut());
    a_fn_that_immutably_borrows(&r.borrow());
}

fn main() {
    let data = RefCell::new(5);
    demo(&data);
}
```

<span class="caption">Listing 15-14: Using `RefCell<T>`, `borrow`, and
`borrow_mut`</span>

這個例子打印出：

```
a is 5
a is 6
```

在`main`函數中，我們新聲明了一個包含值 5 的`RefCell<T>`，並儲存在變數`data`中，聲明時並沒有使用`mut`關鍵字。接著使用`data`的一個不可變引用來調用`demo`函數：對於`main`函數而言`data`是不可變的！

在`demo`函數中，通過調用`borrow`方法來抓取到`RefCell<T>`中值的不可變引用，並使用這個不可變引用調用了`a_fn_that_immutably_borrows`函數。更為有趣的是，可以通過`borrow_mut`方法來抓取`RefCell<T>`中值的**可變**引用，而`a_fn_that_mutably_borrows`函數就允許修改這個值。可以看到下一次調用`a_fn_that_immutably_borrows`時打印出的值是 6 而不是 5。

### `RefCell<T>`在運行時檢查借用規則

回憶一下第四章因為借用規則，嘗試使用常規引用在同一作用域中創建兩個可變引用的代碼無法編譯：

```rust
let mut s = String::from("hello");

let r1 = &mut s;
let r2 = &mut s;
```

這會得到一個編譯錯誤：

```
error[E0499]: cannot borrow `s` as mutable more than once at a time
 -->
  |
5 |     let r1 = &mut s;
  |                   - first mutable borrow occurs here
6 |     let r2 = &mut s;
  |                   ^ second mutable borrow occurs here
7 | }
  | - first borrow ends here
```

與此相反，使用`RefCell<T>`並在同一作用域調用兩次`borrow_mut`的代碼是**可以**編譯的，不過它會在運行時 panic。如下代碼：

```rust,should_panic
use std::cell::RefCell;

fn main() {
    let s = RefCell::new(String::from("hello"));

    let r1 = s.borrow_mut();
    let r2 = s.borrow_mut();
}
```

能夠編譯不過在`cargo run`運行時會出現如下錯誤：

```
    Finished dev [unoptimized + debuginfo] target(s) in 0.83 secs
     Running `target/debug/refcell`
thread 'main' panicked at 'already borrowed: BorrowMutError',
/stable-dist-rustc/build/src/libcore/result.rs:868
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

這個運行時`BorrowMutError`類似於編譯錯誤：它表明我們已經可變得借用過一次`s`了，所以不允許再次借用它。我們並沒有繞過借用規則，只是選擇讓 Rust 在運行時而不是編譯時執行他們。你可以選擇在任何時候任何地方使用`RefCell<T>`，不過除了不得不編寫很多`RefCell`之外，最終還是可能會發現其中的問題（可能是在生產環境而不是開發環境）。另外，在運行時檢查借用規則有性能懲罰。

### 結合`Rc<T>`和`RefCell<T>`來擁有多個可變數據所有者

那麼為什麼要權衡考慮選擇引入`RefCell<T>`呢？好吧，還記得我們說過`Rc<T>`只能擁有一個`T`的不可變引用嗎？考慮到`RefCell<T>`是不可變的，但是擁有內部可變性，可以將`Rc<T>`與`RefCell<T>`結合來創造一個既有引用計數又可變的類型。列表 15-15 展示了一個這麼做的例子，再次回到列表 15-5 中的 cons list。在這個例子中，不同於在 cons list 中儲存`i32`值，我們儲存一個`Rc<RefCell<i32>>`值。希望儲存這個類型是因為其可以擁有不屬於列表一部分的這個值的所有者（`Rc<T>`提供的多個所有者功能），而且還可以改變內部的`i32`值（`RefCell<T>`提供的內部可變性功能）：

<span class="filename">Filename: src/main.rs</span>

```rust
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use List::{Cons, Nil};
use std::rc::Rc;
use std::cell::RefCell;

fn main() {
    let value = Rc::new(RefCell::new(5));

    let a = Cons(value.clone(), Rc::new(Nil));
    let shared_list = Rc::new(a);

    let b = Cons(Rc::new(RefCell::new(6)), shared_list.clone());
    let c = Cons(Rc::new(RefCell::new(10)), shared_list.clone());

    *value.borrow_mut() += 10;

    println!("shared_list after = {:?}", shared_list);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);
}
```

<span class="caption">Listing 15-15: Using `Rc<RefCell<i32>>` to create a
`List` that we can mutate</span>

我們創建了一個值，它是`Rc<RefCell<i32>>`的實例。將其儲存在變數`value`中因為我們希望之後能直接訪問它。接著在`a`中創建了一個擁有存放了`value`值的`Cons`成員的`List`，而且`value`需要被克隆因為我們希望除了`a`之外還擁有`value`的所有權。接著將`a`封裝進`Rc<T>`中這樣就可以創建都引用`a`的有著不同開頭的列表`b`和`c`，類似列表 15-12 中所做的那樣。

一旦創建了`shared_list`、`b`和`c`，接下來就可以通過解引用`Rc<T>`和對`RefCell`調用`borrow_mut`來將 10 與 5 相加了。

當打印出`shared_list`、`b`和`c`時，可以看到他們都擁有被修改的值 15：

```
shared_list after = Cons(RefCell { value: 15 }, Nil)
b after = Cons(RefCell { value: 6 }, Cons(RefCell { value: 15 }, Nil))
c after = Cons(RefCell { value: 10 }, Cons(RefCell { value: 15 }, Nil))
```

這是非常巧妙的！通過使用`RefCell<T>`，我們可以擁有一個表面上不可變的`List`，不過可以使用`RefCell<T>`中提供內部可變性的方法來在需要時修改數據。`RefCell<T>`的運行時借用規則檢查也確實保護我們免於出現數據競爭，而且我們也決定犧牲一些速度來換取數據結構的靈活性。

`RefCell<T>`並不是標準庫中唯一提供內部可變性的類型。`Cell<T>`有點類似，不過不同於`RefCell<T>`那樣提供內部值的引用，其值被拷貝進和拷貝出`Cell<T>`。`Mutex<T>`提供線程間安全的內部可變性，下一章並發會討論它的應用。請查看標準庫來抓取更多細節和不同類型的區別。