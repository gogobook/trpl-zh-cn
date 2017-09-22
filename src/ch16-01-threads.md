## 使用線程同時運行代碼

> [ch16-01-threads.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch16-01-threads.md)
> <br>
> commit 55b294f20fc846a13a9be623bf322d8b364cee77

在今天使用的大部分操作系統中，當程序執行時，操作系統運行代碼的上下文稱為**進程**（*process*）。操作系統可以運行很多進程，而操作系統也管理這些進程使得多個程序可以在電腦上同時運行。

我們可以將每個進程運行一個程序的概念再往下抽象一層：程序也可以在其上下文中同時運行獨立的部分。這個功能叫做**線程**（*thread*）。

將程序需要執行的計算拆分到多個線程中可以提高性能，因為程序可以在同時進行很多工作。不過使用線程會增加程序複雜性。因為線程是同時運行的，所以無法預先保證不同線程中的代碼的執行順序。這可能會由於線程以不一致的順序訪問數據或資源而導致競爭狀態，或由於兩個線程相互阻止對方繼續運行而造成死鎖，以及僅僅出現於特定場景並難以穩定重現的 bug。Rust 減少了這些或那些使用線程的負面影響，不過在多線程上下文中編程，相比只期望在單個線程中運行的程序，仍然要採用不同的思考方式和代碼結構。

編程語言有一些不同的方法來實現線程。很多操作系統提供了創建新線程的 API。另外，很多編程語言提供了自己的特殊的線程實現。編程語言提供的線程有時被稱作**輕量級**（*lightweight*）或**綠色**（*green*）線程。這些語言將一系列綠色線程放入不同數量的操作系統線程中執行。因為這個原因，語言調用操作系統 API 創建線程的模型有時被稱為 *1:1*，一個 OS 線程對應一個語言線程。綠色線程模型被稱為 *M:N* 模型，`M`個綠色線程對應`N`個 OS 線程，這裡`M`和`N`不必相同。

每一個模型都有其自己的優勢和取捨。對於 Rust 來說最重要的取捨是運行時支持。**運行時**是一個令人迷惑的概念；在不同上下文中它可能有不同的含義。這裡其代表二進制文件中包含的語言自身的代碼。對於一些語言，這些代碼是龐大的，另一些則很小。通俗的說，「沒有運行時」通常被人們用來指代「小運行時」，因為任何非彙編語言都存在一定數量的運行時。更小的運行時擁有更少的功能不過其優勢在於更小的二進制輸出。更小的二進制文件更容易在更多上下文中與其他語言結合。雖然很多語言覺得增加運行時來換取更多功能沒有什麼問題，但是 Rust 需要做到幾乎沒有運行時，同時為了保持高性能必需能夠調用 C 語言，這點也是不能妥協的。

綠色線程模型功能要求更大的運行時來管理這些線程。為此，Rust 標準庫只提供了 1:1 線程模型實現。因為 Rust 是這麼一個底層語言，所以有相應的 crate 實現了 M:N 線程模型，如果你寧願犧牲性能來換取例如更好的線程運行控制和更低的上下文切換成本。

現在我們明白了 Rust 中的線程是如何定義的，讓我們開始探索如何使用標準庫提供的線程相關的 API吧。

### 使用`spawn`創建新線程

為了創建一個新線程，調用`thread::spawn`函數並傳遞一個閉包（第十三章學習了閉包），它包含希望在新線程運行的代碼。列表 16-1 中的例子在新線程中打印了一些文本而其餘的文本在主線程中打印：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::thread;

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
    }
}
```

<span class="caption">Listing 16-1: Creating a new thread to print one thing
while the main thread is printing something else</span>


注意這個函數編寫的方式，當主線程結束時，它也會停止新線程。這個程序的輸出每次可能都略微不同，不過它大體上看起來像這樣：

```text
hi number 1 from the main thread!
hi number 1 from the spawned thread!
hi number 2 from the main thread!
hi number 2 from the spawned thread!
hi number 3 from the main thread!
hi number 3 from the spawned thread!
hi number 4 from the main thread!
hi number 4 from the spawned thread!
hi number 5 from the spawned thread!
```

這些線程可能會輪流運行，不過並不保證如此。在這裡，主線程先行打印，即便新創建線程的打印語句位於程序的開頭。甚至即便我們告訴新建的線程打印直到`i`等於 9 ，它在主線程結束之前也只打印到了 5。如果你只看到了一個線程，或沒有出現重疊打印的現象，嘗試增加 range 的數值來增加線程暫停並切換到其他線程運行的機會。

#### 使用`join`等待所有線程結束

由於主線程先於新建線程結束，不僅列表 16-1 中的代碼大部分時候不能保證新建線程執行完畢，甚至不能實際保證新建線程會被執行！可以通過保存`thread::spawn`的返回值來解決這個問題，這是一個`JoinHandle`。這看起來如列表 16-2 所示：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
        }
    });

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
    }

    handle.join();
}
```

<span class="caption">Listing 16-2: Saving a `JoinHandle` from `thread::spawn`
to guarantee the thread is run to completion</span>

`JoinHandle`是一個擁有所有權的值，它可以等待一個線程結束，這也正是`join`方法所做的。通過調用這個句柄的`join`，當前線程會阻塞直到句柄所代表的線程結束。因為我們將`join`調用放在了主線程的`for`循環之後，運行這個例子將產生類似這樣的輸出：

```
hi number 1 from the main thread!
hi number 2 from the main thread!
hi number 1 from the spawned thread!
hi number 3 from the main thread!
hi number 2 from the spawned thread!
hi number 4 from the main thread!
hi number 3 from the spawned thread!
hi number 4 from the spawned thread!
hi number 5 from the spawned thread!
hi number 6 from the spawned thread!
hi number 7 from the spawned thread!
hi number 8 from the spawned thread!
hi number 9 from the spawned thread!
```

這兩個線程仍然會交替執行，不過主線程會由於`handle.join()`調用會等待直到新建線程執行完畢。

如果將`handle.join()`放在主線程的`for`循環之前，像這樣：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {} from the spawned thread!", i);
        }
    });

    handle.join();

    for i in 1..5 {
        println!("hi number {} from the main thread!", i);
    }
}
```

主線程會等待直到新建線程執行完畢之後才開始執行`for`循環，所以輸出將不會交替出現：

```
hi number 1 from the spawned thread!
hi number 2 from the spawned thread!
hi number 3 from the spawned thread!
hi number 4 from the spawned thread!
hi number 5 from the spawned thread!
hi number 6 from the spawned thread!
hi number 7 from the spawned thread!
hi number 8 from the spawned thread!
hi number 9 from the spawned thread!
hi number 1 from the main thread!
hi number 2 from the main thread!
hi number 3 from the main thread!
hi number 4 from the main thread!
```

稍微考慮一下將`join`放置與何處會影響線程是否同時運行。

### 線程和`move`閉包

第十三章有一個我們沒有講到的閉包功能，它經常用於`thread::spawn`：`move`閉包。第十三章中講到：

> 獲取他們環境中值的閉包主要用於開始新線程的場景

現在我們正在創建新線程，所以讓我們討論一下獲取環境值的閉包吧！

注意列表 16-1 中傳遞給`thread::spawn`的閉包並沒有任何參數：並沒有在新建線程代碼中使用任何主線程的數據。為了在新建線程中使用來自於主線程的數據，需要新建線程的閉包獲取它需要的值。列表 16-3 展示了一個嘗試在主線程中創建一個 vector 並用於新建線程的例子，不過這麼寫還不能工作：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(|| {
        println!("Here's a vector: {:?}", v);
    });

    handle.join();
}
```

<span class="caption">Listing 16-3: Attempting to use a vector created by the
main thread from another thread</span>

閉包使用了`v`，所以閉包會獲取`v`並使其成為閉包環境的一部分。因為`thread::spawn`在一個新線程中運行這個閉包，所以可以在新線程中訪問`v`。

然而當編譯這個例子時，會得到如下錯誤：

```
error[E0373]: closure may outlive the current function, but it borrows `v`,
which is owned by the current function
 -->
  |
6 |     let handle = thread::spawn(|| {
  |                                ^^ may outlive borrowed value `v`
7 |         println!("Here's a vector: {:?}", v);
  |                                           - `v` is borrowed here
  |
help: to force the closure to take ownership of `v` (and any other referenced
variables), use the `move` keyword, as shown:
  |     let handle = thread::spawn(move || {
```

當在閉包環境中獲取某些值時，Rust 會嘗試推斷如何獲取它。`println!`只需要`v`的一個引用，所以閉包嘗試借用`v`。但是這有一個問題：我們並不知道新建線程會運行多久，所以無法知道`v`是否一直時有效的。

考慮一下列表 16-4 中的代碼，它展示了一個`v`的引用很有可能不再有效的場景：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(|| {
        println!("Here's a vector: {:?}", v);
    });

    drop(v); // oh no!

    handle.join();
}
```

<span class="caption">Listing 16-4: A thread with a closure that attempts to
capture a reference to `v` from a main thread that drops `v`</span>

這些代碼可以運行，而新建線程則可能直接就出錯了並完全沒有機會運行。新建線程內部有一個`v`的引用，不過主線程仍在執行：它立刻丟棄了`v`，使用了第十五章提到的顯式丟棄其參數的`drop`函數。接著，新建線程開始執行，現在`v`是無效的了，所以它的引用也就是無效的。噢，這太糟了！

為了修復這個問題，我們可以聽取錯誤信息的建議：

```
help: to force the closure to take ownership of `v` (and any other referenced
variables), use the `move` keyword, as shown:
  |     let handle = thread::spawn(move || {
```

通過在閉包之前增加`move`關鍵字，我們強制閉包獲取它使用的值的所有權，而不是引用借用。列表 16-5 中展示的對列表 16-3 代碼的修改可以按照我們的預期編譯並運行：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("Here's a vector: {:?}", v);
    });

    handle.join();
}
```

<span class="caption">Listing 16-5: Using the `move` keyword to force a closure
to take ownership of the values it uses</span>

那麼列表 16-4 中那個主線程調用了`drop`的代碼該怎麼辦呢？如果在閉包上增加了`move`，就將`v`移動到了閉包的環境中，我們將不能對其調用`drop`了。相反會出現這個編譯時錯誤：

```
error[E0382]: use of moved value: `v`
  -->
   |
6  |     let handle = thread::spawn(move || {
   |                                ------- value moved (into closure) here
...
10 |     drop(v); // oh no!
   |          ^ value used here after move
   |
   = note: move occurs because `v` has type `std::vec::Vec<i32>`, which does
   not implement the `Copy` trait
```

Rust 的所有權規則又一次幫助了我們！

現在我們有一個線程和線程 API 的基本瞭解，讓我們討論一下使用線程實際可以**做**什麼吧。