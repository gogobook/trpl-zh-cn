## 使用通道向線程發送請求

> [ch20-05-sending-requests-via-channels.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch20-05-sending-requests-via-channels.md)
> <br>
> commit 2e269ff82193fd65df8a87c06561d74b51ac02f7

下一個需要解決的問題是（線程中的）閉包完全沒有做任何工作。我們一直在繞過抓取 `execute` 方法中實際期望執行的閉包的問題，不過看起來在創建 `ThreadPool` 時就需要知道實際的閉包。

不過考慮一下真正需要做的：我們希望剛創建的 `Worker` 結構體能夠從 `ThreadPool` 的隊列中抓取任務，並在線程中執行他們。

在第十六章中，我們學習了通道。通道是一個溝通兩個線程的良好手段，對於這個例子來說則是絕佳的。通道將充當任務隊列的作用，`execute` 將通過 `ThreadPool` 向其中線程正在尋找工作的 `Worker` 實例發送任務。如下是這個計畫：

1. `ThreadPool` 會創建一個通道並充當發送端。
2. 每個 `Worker` 將會充當通道的接收端。
3. 新建一個 `Job` 結構體來存放用於向通道中發送的閉包。
4. `ThreadPool` 的 `execute` 方法會在發送端發出期望執行的任務。
5. 在線程中，`Worker` 會遍歷通道的接收端並執行任何接收到的任務。

讓我們以在 `ThreadPool::new` 中創建通道並讓 `ThreadPool` 實例充當發送端開始，如示例 20-16 所示。`Job` 是將在通道中發出的類型；目前它是一個沒有任何內容的結構體：

<span class="filename">文件名: src/lib.rs</span>

```rust
# use std::thread;
// ...snip...
use std::sync::mpsc;

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

struct Job;

impl ThreadPool {
    // ...snip...
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id));
        }

        ThreadPool {
            workers,
            sender,
        }
    }
    // ...snip...
}
#
# struct Worker {
#     id: usize,
#     thread: thread::JoinHandle<()>,
# }
#
# impl Worker {
#     fn new(id: usize) -> Worker {
#         let thread = thread::spawn(|| {});
#
#         Worker {
#             id,
#             thread,
#         }
#     }
# }
```

<span class="caption">示例 20-16：修改 `ThreadPool` 來儲存一個發送 `Job` 實例的通道發送端</span>

在 `ThreadPool::new` 中，新建了一個通道，並接著讓線程池在接收端等待。這段代碼能夠編譯，不過仍有警告。

在線程池創建每個 worker 時將通道的接收端傳遞給他們。須知我們希望在 worker 所分配的線程中使用通道的接收端，所以將在閉包中引用 `receiver` 參數。示例 20-17 中展示的代碼還不能編譯：

<span class="filename">文件名: src/lib.rs</span>

```rust
impl ThreadPool {
    // ...snip...
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, receiver));
        }

        ThreadPool {
            workers,
            sender,
        }
    }
    // ...snip...
}

// ...snip...

impl Worker {
    fn new(id: usize, receiver: mpsc::Receiver<Job>) -> Worker {
        let thread = thread::spawn(|| {
            receiver;
        });

        Worker {
            id,
            thread,
        }
    }
}
```

<span class="caption">示例 20-17：將通道的接收端傳遞給 worker</span>

這是一些小而直觀的修改：將通道的接收端傳遞進了 `Worker::new`，並接著在閉包中使用他們。

如果嘗試檢查代碼，會得到這個錯誤：

```
$ cargo check
   Compiling hello v0.1.0 (file:///projects/hello)
error[E0382]: use of moved value: `receiver`
  --> src/lib.rs:27:42
   |
27 |             workers.push(Worker::new(id, receiver));
   |                                          ^^^^^^^^ value moved here in
   previous iteration of loop
   |
   = note: move occurs because `receiver` has type
   `std::sync::mpsc::Receiver<Job>`, which does not implement the `Copy` trait
```

這些代碼還不能編譯的原因如上因為它嘗試將 `receiver` 傳遞給多個 `Worker` 實例。回憶第十六章，Rust 所提供的通道實現是多**生產者**，單**消費者**的，所以不能簡單的克隆通道的消費端來解決問題。即便可以我們也不希望克隆消費端；在所有的 worker 中共享單一 `receiver` 才是我們希望的在線程間分發任務的機制。

另外，從通道隊列中取出任務涉及到修改 `receiver`，所以這些線程需要一個能安全的共享和修改 `receiver` 的方式。如果修改不是線程安全的，則可能遇到競爭狀態，例如兩個線程因同時在隊列中取出相同的任務並執行了相同的工作。

所以回憶一下第十六章討論的線程安全智能指針，為了在多個線程間共享所有權並允許線程修改其值，需要使用 `Arc<Mutex<T>>`。`Arc` 使得多個 worker 擁有接收端，而 `Mutex` 則確保一次只有一個 worker 能從接收端得到任務。示例 20-18 展示了所做的修改：

<span class="filename">文件名: src/lib.rs</span>

```rust
# use std::thread;
# use std::sync::mpsc;
use std::sync::Arc;
use std::sync::Mutex;

// ...snip...

# pub struct ThreadPool {
#     workers: Vec<Worker>,
#     sender: mpsc::Sender<Job>,
# }
# struct Job;
#
impl ThreadPool {
    // ...snip...
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, receiver.clone()));
        }

        ThreadPool {
            workers,
            sender,
        }
    }

    // ...snip...
}
# struct Worker {
#     id: usize,
#     thread: thread::JoinHandle<()>,
# }
#
impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        // ...snip...
#         let thread = thread::spawn(|| {
#            receiver;
#         });
#
#         Worker {
#             id,
#             thread,
#         }
    }
}
```

<span class="caption">示例 20-18：使用 `Arc` 和 `Mutex` 在 worker 間共享通道的接收端</span>

在 `ThreadPool::new` 中，將通道的接收端放入一個 `Arc` 和一個 `Mutex` 中。對於每一個新 worker，則克隆 `Arc` 來增加引用計數，如此這些 worker 就可以共享接收端的所有權了。

通過這些修改，代碼可以編譯了！我們做到了！

最好讓我們實現 `ThreadPool` 上的 `execute` 方法。同時也要修改 `Job` 結構體：它將不再是結構體，`Job` 將是一個有著 `execute` 接收到的閉包類型的 trait 物件的類型別名。我們討論過類型別名如何將長的類型變短，現在就這種情況！看一看示例 20-19：

<span class="filename">文件名: src/lib.rs</span>

```rust
// ...snip...
# pub struct ThreadPool {
#     workers: Vec<Worker>,
#     sender: mpsc::Sender<Job>,
# }
# use std::sync::mpsc;
# struct Worker {}

type Job = Box<FnOnce() + Send + 'static>;

impl ThreadPool {
    // ...snip...

    pub fn execute<F>(&self, f: F)
        where
            F: FnOnce() + Send + 'static
    {
        let job = Box::new(f);

        self.sender.send(job).unwrap();
    }
}

// ...snip...
```

<span class="caption">示例 20-19：為存放每一個閉包的 `Box` 創建一個 `Job` 類型別名，接著在通道中發出</span>

在使用 `execute` 得到的閉包新建 `Job` 實例之後，將這些任務從通道的發送端發出。這裡調用 `send` 上的 `unwrap`，因為如果接收端停止接收新消息則發送可能會失敗，這可能發生於我們停止了所有的執行線程。不過目前這是不可能的，因為只要線程池存在他們就會一直執行。使用 `unwrap` 是因為我們知道失敗不可能發生，即便編譯器不這麼認為，正如第九章討論的這是 `unwrap` 的一個恰當用法。

那我們結束了嗎？不完全是！在 worker 中，傳遞給 `thread::spawn` 的閉包仍然還只是**引用**了通道的接收端。但是我們需要閉包一直循環，向通道的接收端請求任務，並在得到任務時執行他們。如示例 20-20 對 `Worker::new` 做出修改：

<span class="filename">文件名: src/lib.rs</span>

```rust
// ...snip...

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || {
            loop {
                let job = receiver.lock().unwrap().recv().unwrap();

                println!("Worker {} got a job; executing.", id);

                (*job)();
            }
        });

        Worker {
            id,
            thread,
        }
    }
}
```

<span class="caption">示例 20-20： 在 worker 線程中接收並執行任務</span>

這裡，首先在 `receiver` 上調用了 `lock` 來抓取互斥器，接著 `unwrap` 在出現任何錯誤時 panic。如果互斥器處於一種叫做**被污染**（*poisoned*）的狀態時抓取鎖肯能會失敗，這可能發生於其他線程在持有鎖時 panic 了並沒有釋放鎖。如果當前線程因為這個原因不能得到所，調用 `unwrap` 使其 panic 也是正確的行為。如果你覺得有意義的話請隨意將 `unwrap` 改為帶有錯誤信息的 `expect`。

如果鎖定了互斥器，接著調用 `recv` 從通道中接收 `Job`。最後的 `unwrap` 也繞過了一些錯誤，`recv` 在通道的發送端關閉時會返回 `Err`，類似於 `send` 在接收端關閉時返回 `Err` 一樣。

調用 `recv` 的代碼塊；也就是說，它還沒有任務，這個線程會等待直到有可用的任務。`Mutex<T>` 確保一次只有一個 `Worker` 線程嘗試請求任務。

理論上這段代碼應該能夠編譯。不幸的是，Rust 編譯器仍不夠完美，會給出如下錯誤：

```
error[E0161]: cannot move a value of type std::ops::FnOnce() +
std::marker::Send: the size of std::ops::FnOnce() + std::marker::Send cannot be
statically determined
  --> src/lib.rs:63:17
   |
63 |                 (*job)();
   |                 ^^^^^^
```

這個錯誤非常的神秘，因為這個問題本身就很神秘。為了調用儲存在 `Box<T>` （這正是 `Job` 別名的類型）中的 `FnOnce` 閉包，該閉包需要能將自己移動出 `Box<T>`，因為當調用這個閉包時，它抓取 `self` 的所有權。通常來說，將值移動出 `Box<T>` 是不被允許的，因為 Rust 不知道 `Box<T>` 中的值將會有多大；回憶第十五章能夠正常使用 `Box<T>` 是因為我們將未知大小的值儲存進 `Box<T>` 從而得到已知大小的值。

第十七章曾見過，示例 17-15 中有使用了 `self: Box<Self>` 語法的方法，它抓取了儲存在 `Box<T>` 中的 `Self` 值的所有權。這正是我們希望做的，然而不幸的是 Rust 調用閉包的那部分實現並沒有使用 `self: Box<Self>`。所以這裡 Rust 也不知道它可以使用 `self: Box<Self>` 來抓取閉包的所有權並將閉包移動出 `Box<T>`。

將來示例 20-20 中的代碼應該能夠正常工作。Rust 仍在努力改進提升編譯器。有很多像你一樣的人正在修復這個以及其他問題！當你結束了本書的閱讀，我們希望看到你也成為他們中的一員。

不過目前讓我們繞過這個問題。所幸有一個技巧可以顯式的告訴 Rust 我們處於可以抓取使用 `self: Box<Self>` 的 `Box<T>` 中值的所有權的狀態，而一旦抓取了閉包的所有權就可以調用它了。這涉及到定義一個新 trait，它帶有一個在簽名中使用 `self: Box<Self>` 的方法 `call_box`，為任何實現了 `FnOnce()` 的類型定義這個 trait，修改類型別名來使用這個新 trait，並修改 `Worker` 使用 `call_box` 方法。這些修改如示例 20-21 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
trait FnBox {
    fn call_box(self: Box<Self>);
}

impl<F: FnOnce()> FnBox for F {
    fn call_box(self: Box<F>) {
        (*self)()
    }
}

type Job = Box<FnBox + Send + 'static>;

// ...snip...

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || {
            loop {
                let job = receiver.lock().unwrap().recv().unwrap();

                println!("Worker {} got a job; executing.", id);

                job.call_box();
            }
        });

        Worker {
            id,
            thread,
        }
    }
}
```

<span class="caption">示例 20-21：新增一個 trait `FnBox` 來繞過當前 `Box<FnOnce()>` 的限制</span>

首先，新建了一個叫做 `FnBox` 的 trait。這個 trait 有一個方法 `call_box`，它類似於其他 `Fn*` trait 中的 `call` 方法，除了它抓取 `self: Box<Self>` 以便抓取 `self` 的所有權並將值從 `Box<T>` 中移動出來。

現在我們希望 `Job` 類型別名是任何實現了新 trait `FnBox` 的 `Box`，而不是 `FnOnce()`。這允許我們在得到 `Job` 值時使用 `Worker` 中的 `call_box`。因為我們為任何 `FnOnce()` 閉包都實現了 `FnBox` trait，無需對實際在通道中發出的值做任何修改。

最後，對於 `Worker::new` 的線程中所運行的閉包，調用 `call_box` 而不是直接執行閉包。現在 Rust 就能夠理解我們的行為是正確的了。

這是非常狡猾且複雜的手段。無需過分擔心他們並不是非常有道理；總有一天，這一切將是毫無必要的。

通過這些技巧，線程池處於可以運行的狀態了！執行 `cargo run` 並發起一些請求：

```
$ cargo run
   Compiling hello v0.1.0 (file:///projects/hello)
warning: field is never used: `workers`
 --> src/lib.rs:7:5
  |
7 |     workers: Vec<Worker>,
  |     ^^^^^^^^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `id`
  --> src/lib.rs:61:5
   |
61 |     id: usize,
   |     ^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `thread`
  --> src/lib.rs:62:5
   |
62 |     thread: thread::JoinHandle<()>,
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.99 secs
     Running `target/debug/hello`
     Worker 0 got a job; executing.
Worker 2 got a job; executing.
Worker 1 got a job; executing.
Worker 3 got a job; executing.
Worker 0 got a job; executing.
Worker 2 got a job; executing.
Worker 1 got a job; executing.
Worker 3 got a job; executing.
Worker 0 got a job; executing.
Worker 2 got a job; executing.
```

成功了！現在我們有了一個可以異步執行連接的線程池！它絕不會創建超過四個線程，所以當 server 收到大量請求時系統也不會負擔過重。如果請求 `/sleep`，server 也能夠通過另外一個線程處理其他請求。

那麼這些警告怎麼辦呢？難道我們沒有使用 `workers`、`id` 和 `thread` 字段嗎？好吧，目前我們用了所有這些字段存放了一些數據，不過當設置線程池並開始執行代碼在通道中向線程發送任務時，我們並沒有對數據**進行**任何實際的操作。但是如果不存放這些值，他們將會離開作用域：比如，如果不將 `Vec<Worker>` 值作為 `ThreadPool` 的一部分返回，這個 vector 在 `ThreadPool::new` 的結尾就會被清理。

那麼這些警告有錯嗎？從某種角度上講是的，這些警告是錯誤的，因為我們使用這些字段儲存一直需要的數據。從另一種角度來說也不對：使用過後我們也沒有做任何操作清理線程池，僅僅通過 <span class="keystroke">ctrl-C</span> 來停止程序並讓作業系統為我們清理。下面讓我們實現 graceful shutdown 來清理所創建的一切。