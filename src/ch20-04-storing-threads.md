## 創建線程池並儲存線程

> [ch20-04-storing-threads.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch20-04-storing-threads.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

之前的警告是因為在 `new` 和 `execute` 中沒有對參數做任何操作。讓我們用期望的實際行為實現他們。

### 驗證池中的線程數

以考慮 `new` 作為開始。之前提到使用無符號類型作為 `size` 參數的類型，因為為負的線程數沒有意義。然而，零個線程同樣沒有意義，不過零是一個完全有效的 `u32` 值。讓我們在返回 `ThreadPool` 之前檢查 `size`  是否大於零，並使用 `assert!` 巨集在得到零時 panic，如列表 20-13 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub struct ThreadPool;
impl ThreadPool {
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
    pub fn new(size: u32) -> ThreadPool {
        assert!(size > 0);

        ThreadPool
    }

    // ...snip...
}
```

<span class="caption">列表 20-13：實現 `ThreadPool::new` 在 `size` 為零時 panic</span>

趁著這個機會我們用文檔註釋為 `ThreadPool` 增加了一些文檔。注意這裡遵循了良好的文檔實踐並增加了一個部分提示函數會 panic 的情況，正如第十四章所討論的。嘗試運行 `cargo doc --open` 並點擊 `ThreadPool` 結構體來查看生成的 `new` 的文檔看起來如何！

相比像這裡使用 `assert!` 巨集，也可以讓 `new` 像之前 I/O 項目中列表 12-9 中 `Config::new` 那樣返回一個 `Result`，不過在這裡我們選擇創建一個沒有任何線程的線程池應該是要給不可恢復的錯誤。如果你想做的更好，嘗試編寫一個採用如下籤名的 `new` 版本來感受一下兩者的區別：

```rust
fn new(size: u32) -> Result<ThreadPool, PoolCreationError> {
```

### 在線程池中儲存線程

現在有了一個有效的線程池線程數，就可以實際創建這些線程並在返回之前將他們儲存在 `ThreadPool` 結構體中。

這引出了另一個問題：如何「儲存」一個線程？讓我們再看看 `thread::spawn` 的簽名：

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
    where
        F: FnOnce() -> T + Send + 'static,
        T: Send + 'static
```

`spawn` 返回 `JoinHandle<T>`，其中 `T` 是閉包返回的類型。嘗試使用 `JoinHandle` 來看看會發生什麼。在我們的情況中，傳遞給線程池的閉包會處理連接並不返回任何值，所以 `T` 將會是單元類型 `()`。

這還不能編譯，不過考慮一下列表 20-14 所示的代碼。我們改變了 `ThreadPool` 的定義來存放一個 `thread::JoinHandle<()>` 的 vector 實例，使用 `size` 容量來初始化，並設置一個 `for` 循環了來運行創建線程的代碼，並返回包含這些線程的 `ThreadPool` 實例：


<span class="filename">文件名: src/lib.rs</span>

```rust
use std::thread;

pub struct ThreadPool {
    threads: Vec<thread::JoinHandle<()>>,
}

impl ThreadPool {
    // ...snip...
    pub fn new(size: u32) -> ThreadPool {
        assert!(size > 0);

        let mut threads = Vec::with_capacity(size);

        for _ in 0..size {
            // create some threads and store them in the vector
        }

        ThreadPool {
            threads
        }
    }

    // ...snip...
}
```

<span class="caption">列表 20-14：為 `ThreadPool` 創建一個 vector 來存放線程</span>

這裡將 `std::thread` 引入庫 crate 的作用域，因為使用了 `thread::JoinHandle` 作為 `ThreadPool` 中 vector 元素的類型。

在得到了有效的數量之後，就可以新建一個存放 `size` 個元素的 vector。本書還未使用過 `with_capacity`；它與 `Vec::new` 做了同樣的工作，不過有一個重要的區別：它為 vector 預先分配空間。因為已經知道了 vector 中需要 `size` 個元素，預先進行分配比僅僅 `Vec::new` 要稍微有效率一些，因為 `Vec::new` 隨著插入元素而重新改變大小。因為一開始就用所需的確定大小來創建 vector，為其增減元素時不會改變底層 vector 的大小。

如果代碼能夠工作就應是如此效果，不過他們還不能工作！如果檢查他們，會得到一個錯誤：

```
$ cargo check
   Compiling hello v0.1.0 (file:///projects/hello)
error[E0308]: mismatched types
  --> src\main.rs:70:46
   |
70 |         let mut threads = Vec::with_capacity(size);
   |                                              ^^^^ expected usize, found u32

error: aborting due to previous error
```

`size` 是 `u32`，不過 `Vec::with_capacity` 需要一個 `usize`。這裡有兩個選擇：可以改變函數簽名，或者可以將 `u32` 轉換為 `usize`。如果你還記得定義 `new` 時，並沒有仔細考慮有意義的數值類型，只是隨便選了一個。現在來進行一些思考吧。考慮到 `size` 是 vector 的長度，`usize` 就很有道理了。甚至他們的名字都很類似！改變 `new` 的簽名，這會使列表 20-14 的代碼能夠編譯：

```rust
fn new(size: usize) -> ThreadPool {
```

如果再次運行 `cargo check`，會得到一些警告，不過應該能成功編譯。

列表 20-14 的 `for` 循環中留下了一個關於創建線程的註釋。如何實際創建線程呢？這是一個難題。這些線程應該做什麼呢？這裡並不知道他們需要做什麼，因為 `execute` 方法抓取閉包並傳遞給了線程池。

讓我們稍微重構一下：不再儲存一個 `JoinHandle<()>` 實例的 vector，將創建一下新的結構體來代表 *worker* 的概念。worker 會接收 `execute` 方法，並會處理實際的閉包調用。另外儲存固定 `size` 數量的還不知道將要執行什麼閉包的 `Worker` 實例，也可以為每一個 worker 設置一個 `id`，這樣就可以在日誌和調試中區別線程池中的不同 worker。

讓我們做出如下修改：

1. 定義 `Worker` 結構體存放 `id` 和 `JoinHandle<()>`
2. 修改 `ThreadPool` 存放一個 `Worker` 實例的 vector
3. 定義 `Worker::new` 函數，它抓取一個 `id` 數字並返回一個帶有 `id` 和用空閉包分配的線程的 `Worker` 實例，之後會修復這些
4. 在 `ThreadPool::new` 中，使用 `for` 循環來計數生成 `id`，使用這個 `id` 新建 `Worker`，並儲存進 vector 中

如果你渴望挑戰，在查看列表 20-15 中的代碼之前嘗試自己實現這些修改。

準備好了嗎？列表 20-15 就是一個做出了這些修改的例子：

<span class="filename">文件名: src/lib.rs</span>

```rust
use std::thread;

pub struct ThreadPool {
    workers: Vec<Worker>,
}

impl ThreadPool {
    // ...snip...
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id));
        }

        ThreadPool {
            workers
        }
    }
    // ...snip...
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize) -> Worker {
        let thread = thread::spawn(|| {});

        Worker {
            id,
            thread,
        }
    }
}
```

<span class="caption">列表 20-15：修改 `ThreadPool` 存放 `Worker` 實例而不是直接存放線程</span>

這裡選擇將 `ThreadPool` 中字段名從 `threads` 改為 `workers`，因為我們改變了存放內容為 `Worker` 而不是 `JoinHandle<()>`。使用 `for` 循環中的計數作為 `Worker::new` 的參數，並將每一個新建的 `Worker` 儲存在叫做 `workers` 的 vector 中。

`Worker` 結構體和其 `new` 函數是私有的，因為外部代碼（比如 *src/bin/main.rs* 中的 server）並不需要 `ThreadPool` 中使用 `Worker` 結構體的實現細節。`Worker::new` 函數使用 `id` 參數並儲存了使用一個空閉包創建的 `JoinHandle<()>`。

這段代碼能夠編譯並用指定給 `ThreadPool::new` 的參數創建儲存了一系列的 `Worker` 實例，不過**仍然**沒有處理 `execute` 中得到的閉包。讓我們聊聊接下來怎麼做。