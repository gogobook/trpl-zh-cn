## 共享狀態並發

> [ch16-03-shared-state.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch16-03-shared-state.md)
> <br>
> commit 9df612e93e038b05fc959db393c15a5402033f47

雖然消息傳遞是一個很好的處理並發的方式，但並不是唯一的一個。再次考慮一下它的口號：

> Do not communicate by sharing memory; instead, share memory by
> communicating.
>
> 不要共享內存來通訊；而是要通訊來共享內存。

那麼「共享內存來通訊」是怎樣的呢？共享內存並發有點像多所有權：多個線程可以同時訪問相同的內存位置。第十五章介紹了智能指針如何使得多所有權成為可能，然而這會增加額外的複雜性，因為需要以某種方式管理這些不同的所有者。

不過 Rust 的類型系統和所有權可以很好的幫助我們，正確的管理它們。以共享內存中更常見的並發原語：互斥器（mutexes）為例，讓我們看看具體的情況。

### 互斥器一次只允許一個線程訪問數據

**互斥器**（*mutex*）是一種用於共享內存的並發原語。它是「mutual exclusion」的縮寫，也就是說，任意時間，它只允許一個線程訪問某些數據。互斥器以難以使用著稱，因為你不得不記住：

1. 在使用數據之前嘗試抓取鎖。
2. 處理完被互斥器所保護的數據之後，必須解鎖數據，這樣其他線程才能夠抓取鎖。

現實中也有互斥器的例子，想像一下在一個會議中，只有一個麥克風。如果一個成員要發言，他必須請求使用麥克風。一旦得到了麥克風，他可以暢所欲言，然後將麥克風交給下一個希望講話的成員。如果成員在沒有麥克風的時候就開始叫喊，或者在其他成員發言結束之前就拿走麥克風，是很不合適的。如果這個共享的麥克風因為此類原因而出現問題，會議將無法正常進行。

正確的管理互斥器異常複雜，這也是許多人之所以熱衷於通道的原因。然而，在 Rust 中，得益於類型系統和所有權，我們不會在鎖和解鎖上出錯。

### `Mutex<T>`的 API

讓我們看看代碼例 16-12 中使用互斥器的例子，現在不涉及多線程：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);

    {
        let mut num = m.lock().unwrap();
        *num = 6;
    }

    println!("m = {:?}", m);
}
```

<span class="caption">Listing 16-12: Exploring the API of `Mutex<T>` in a
single threaded context for simplicity</span>

像很多類型一樣，我們使用關聯函數 `new` 來創建一個 `Mutex<T>`。使用`lock`方法抓取鎖，以訪問互斥器中的數據。這個調用會阻塞，直到我們擁有鎖為止。如果另一個線程擁有鎖，並且那個線程 panic 了，則這個調用會失敗。類似於代碼例 16-6 那樣，我們暫時使用 `unwrap()` 進行錯誤處理，或者使用第九章中提及的更好的工具。


一旦抓取了鎖，就可以將返回值（在這裡是`num`）作為一個數據的可變引用使用了。觀察 Rust 類型系統如何保證使用值之前必須抓取鎖：`Mutex<i32>`並不是一個`i32`，所以**必須**抓取鎖才能使用這個`i32`值。我們是不會忘記這麼做的，因為類型系統不允許。


你也許會懷疑，`Mutex<T>`是一個智能指針？是的！更準確的說，`lock`調用返回一個叫做`MutexGuard`的智能指針。類似我們在第十五章見過的智能指針，它實現了`Deref`來指向其內部數據。另外`MutexGuard`有一個用來釋放鎖的`Drop`實現。這樣就不會忘記釋放鎖了。這在`MutexGuard`離開作用域時會自動發生，例如它發生於代碼例 16-12 中內部作用域的結尾。接著可以打印出互斥器的值並發現能夠將其內部的`i32`改為 6。

#### 在線程間共享`Mutex<T>`

現在讓我們嘗試使用`Mutex<T>`在多個線程間共享值。我們將啟動十個線程，並在各個線程中對同一個計數器值加一，這樣計數器將從 0 變為 10。注意，接下來的幾個例子會出現編譯錯誤，而我們將通過這些錯誤來學習如何使用
`Mutex<T>`，以及 Rust 又是如何輔助我們以確保正確。代碼例 16-13 是最開始的例子：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Mutex::new(0);
    let mut handles = vec![];

    for _ in 0..10 {
        let handle = thread::spawn(|| {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

<span class="caption">Listing 16-13: The start of a program having 10 threads
each increment a counter guarded by a `Mutex<T>`</span>

這裡創建了一個 `counter` 變數來存放內含 `i32` 的 `Mutex<T>`，類似代碼例 16-12 那樣。接下來使用 range 創建了 10 個線程。使用了 `thread::spawn` 並對所有線程使用了相同的閉包：他們每一個都將調用 `lock` 方法來抓取 `Mutex<T>` 上的鎖，接著將互斥器中的值加一。當一個線程結束執行，`num` 會離開閉包作用域並釋放鎖，這樣另一個線程就可以抓取它了。

在主線程中，我們像代碼例 16-2 那樣收集了所有的 join 句柄，調用它們的 `join` 方法來確保所有線程都會結束。之後，主線程會抓取鎖並打印出程序的結果。

之前提示過這個例子不能編譯，讓我們看看為什麼！

```
error[E0373]: closure may outlive the current function, but it borrows
`counter`, which is owned by the current function
  -->
   |
9  |         let handle = thread::spawn(|| {
   |                                    ^^ may outlive borrowed value `counter`
10 |             let mut num = counter.lock().unwrap();
   |                           ------- `counter` is borrowed here
   |
help: to force the closure to take ownership of `counter` (and any other
referenced variables), use the `move` keyword, as shown:
   |         let handle = thread::spawn(move || {
```

這類似於代碼例 16-5 中解決了的問題。考慮到啟動了多個線程，Rust 無法知道這些線程會運行多久，而在每一個線程嘗試借用 `counter` 時它是否仍然有效。幫助信息提醒了我們如何解決它：可以使用 `move` 來給予每個線程其所有權。嘗試在閉包上做一點改動：

```rust
thread::spawn(move || {
```

再次編譯。這回出現了一個不同的錯誤！

```
error[E0382]: capture of moved value: `counter`
  -->
   |
9  |         let handle = thread::spawn(move || {
   |                                    ------- value moved (into closure) here
10 |             let mut num = counter.lock().unwrap();
   |                           ^^^^^^^ value captured here after move
   |
   = note: move occurs because `counter` has type `std::sync::Mutex<i32>`,
   which does not implement the `Copy` trait

error[E0382]: use of moved value: `counter`
  -->
   |
9  |         let handle = thread::spawn(move || {
   |                                    ------- value moved (into closure) here
...
21 |     println!("Result: {}", *counter.lock().unwrap());
   |                             ^^^^^^^ value used here after move
   |
   = note: move occurs because `counter` has type `std::sync::Mutex<i32>`,
   which does not implement the `Copy` trait

error: aborting due to 2 previous errors
```

`move` 並沒有像代碼例 16-5 中那樣解決問題。為什麼呢？錯誤信息有點難懂，因為它表明 `counter` 被移動進了閉包，接著它在調用 `lock` 時被捕獲。這似乎是我們希望的，然而不被允許。

讓我們推理一下。這次不再使用 `for` 循環創建 10 個線程，只創建兩個線程，看看會發生什麼。將代碼例 16-13 中第一個`for`循環替換為如下代碼：

```rust
let handle = thread::spawn(move || {
    let mut num = counter.lock().unwrap();

    *num += 1;
});
handles.push(handle);

let handle2 = thread::spawn(move || {
    let mut num2 = counter.lock().unwrap();

    *num2 += 1;
});
handles.push(handle2);
```

這裡創建了兩個線程，並將第二個線程所用的變數改名為 `handle2` 和 `num2`。我們簡化了例子，看是否能理解錯誤信息。此次編譯給出如下信息：

```text
error[E0382]: capture of moved value: `counter`
  -->
   |
8  |     let handle = thread::spawn(move || {
   |                                ------- value moved (into closure) here
...
16 |         let mut num = counter.lock().unwrap();
   |                       ^^^^^^^ value captured here after move
   |
   = note: move occurs because `counter` has type `std::sync::Mutex<i32>`,
   which does not implement the `Copy` trait

error[E0382]: use of moved value: `counter`
  -->
   |
8  |     let handle = thread::spawn(move || {
   |                                ------- value moved (into closure) here
...
26 |     println!("Result: {}", *counter.lock().unwrap());
   |                             ^^^^^^^ value used here after move
   |
   = note: move occurs because `counter` has type `std::sync::Mutex<i32>`,
   which does not implement the `Copy` trait

error: aborting due to 2 previous errors
```

啊哈！第一個錯誤信息中說，`counter` 被移動進了 `handle` 所代表線程的閉包中。因此我們無法在第二個線程中對其調用 `lock`，並將結果儲存在 `num2` 中時捕獲`counter`！所以 Rust 告訴我們不能將 `counter` 的所有權移動到多個線程中。這在之前很難看出，因為我們在循環中創建了多個線程，而 Rust 無法在每次迭代中指明不同的線程（沒有臨時變數 `num2`）。

#### 多線程和多所有權

在第十五章中，我們通過使用智能指針 `Rc<T>` 來創建引用計數的值，以便擁有多所有權。同時第十五章提到了 `Rc<T>` 只能在單線程環境中使用，不過還是在這裡試用 `Rc<T>` 看看會發生什麼。代碼例 16-14 將 `Mutex<T>` 裝進了 `Rc<T>` 中，並在移入線程之前克隆了 `Rc<T>`。再用循環來創建線程，保留閉包中的 `move` 關鍵字：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::rc::Rc;
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Rc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
    	let counter = counter.clone();
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

<span class="caption">Listing 16-14: Attempting to use `Rc<T>` to allow
multiple threads to own the `Mutex<T>`</span>

再一次編譯並...出現了不同的錯誤！編譯器真是教會了我們很多！

```
error[E0277]: the trait bound `std::rc::Rc<std::sync::Mutex<i32>>:
std::marker::Send` is not satisfied
  -->
   |
11 |         let handle = thread::spawn(move || {
   |                      ^^^^^^^^^^^^^ the trait `std::marker::Send` is not
   implemented for `std::rc::Rc<std::sync::Mutex<i32>>`
   |
   = note: `std::rc::Rc<std::sync::Mutex<i32>>` cannot be sent between threads
   safely
   = note: required because it appears within the type
   `[closure@src/main.rs:11:36: 15:10
   counter:std::rc::Rc<std::sync::Mutex<i32>>]`
   = note: required by `std::thread::spawn`
```

哇哦，太長不看！說重點：第一個提示表明 `Rc<Mutex<i32>>` 不能安全的在線程間傳遞。理由也在錯誤信息中，「不滿足 `Send` trait bound」（`the trait bound Send is not satisfied`）。下一部分將會討論 `Send`，它是確保許多用在多線程中的類型，能夠適合併發環境的 trait 之一。

不幸的是，`Rc<T>` 並不能安全的在線程間共享。當 `Rc<T>` 管理引用計數時，它必須在每一個 `clone` 調用時增加計數，並在每一個克隆被丟棄時減少計數。`Rc<T>` 並沒有使用任何並發原語，來確保改變計數的操作不會被其他線程打斷。在計數出錯時可能會導致詭異的 bug，比如可能會造成內存洩漏，或在使用結束之前就丟棄一個值。如果有一個類型與 `Rc<T>` 相似，又以一種線程安全的方式改變引用計數，會怎麼樣呢？

#### 原子引用計數 `Arc<T>`

答案是肯定的，確實有一個類似`Rc<T>`並可以安全的用於並發環境的類型：`Arc<T>`。字母「a」代表**原子性**（*atomic*），所以這是一個**原子引用計數**（*atomically reference counted*）類型。原子性是另一類這裡還未涉及到的並發原語；請查看標準庫中`std::sync::atomic`的文檔來抓取更多細節。其中的要點就是：原子性類型工作起來類似原始類型，不過可以安全的在線程間共享。

為什麼不是所有的原始類型都是原子性的？為什麼不是所有標準庫中的類型都預設使用`Arc<T>`實現？線程安全帶來性能懲罰，我們希望只在必要時才為此買單。如果只是在單線程中對值進行操作，原子性提供的保證並無必要，代碼可以因此運行的更快。

回到之前的例子：`Arc<T>`和`Rc<T>`除了`Arc<T>`內部的原子性之外沒有區別。其 API 也相同，所以可以修改`use`行和`new`調用。代碼例 16-15 中的代碼最終可以編譯和運行：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::sync::{Mutex, Arc};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
    	let counter = counter.clone();
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();

            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

<span class="caption">Listing 16-15: Using an `Arc<T>` to wrap the `Mutex<T>`
to be able to share ownership across multiple threads</span>

這會打印出：

```
Result: 10
```

成功了！我們從 0 數到了 10，這可能並不是很顯眼，不過一路上我們學習了很多關於`Mutex<T>`和線程安全的內容！這個例子中構建的結構可以用於比增加計數更為複雜的操作。能夠被分解為獨立部分的計算可以像這樣被分散到多個線程中，並可以使用`Mutex<T>`來允許每個線程在他們自己的部分更新最終的結果。

你可能注意到了，因為`counter`是不可變的，不過可以抓取其內部值的可變引用，這意味著`Mutex<T>`提供了內部可變性，就像`Cell`系列類型那樣。正如第十五章中使用`RefCell<T>`可以改變`Rc<T>`中的內容那樣，同樣的可以使用`Mutex<T>`來改變`Arc<T>`中的內容。

回憶一下`Rc<T>`並沒有避免所有可能的問題：我們也討論了當兩個`Rc<T>`相互引用時的引用循環的可能性，這可能造成內存洩露。`Mutex<T>`有一個類似的 Rust 同樣也不能避免的問題：死鎖。**死鎖**（*deadlock*）是一個場景中操作需要鎖定兩個資源，而兩個線程分別擁有一個鎖並永遠相互等待的問題。如果你對這個主題感興趣，嘗試編寫一個帶有死鎖的 Rust 程序，接著研究任何其他語言中使用互斥器的死鎖規避策略並嘗試在 Rust 中實現他們。標準庫中`Mutex<T>`和`MutexGuard`的 API 文檔會提供有用的信息。

Rust 的類型系統和所有權規則，確保了線程在更新共享值時擁有獨佔的訪問權限，所以線程不會以不可預測的方式覆蓋彼此的操作。雖然為了使一切正確運行而在編譯器上花了一些時間，但是我們節省了未來的時間，尤其是線程以特定順序執行才會出現的詭異錯誤難以重現。

接下來，為了豐富本章的內容，讓我們討論一下`Send`和`Sync` trait 以及如何對自定義類型使用他們。