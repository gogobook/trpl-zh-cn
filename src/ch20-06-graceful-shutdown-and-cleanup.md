## Graceful Shutdown 與清理

> [ch20-06-graceful-shutdown-and-cleanup.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch20-06-graceful-shutdown-and-cleanup.md)
> <br>
> commit 2e269ff82193fd65df8a87c06561d74b51ac02f7

列表 20-21 中的代碼如期通過使用線程池異步的響應請求。這裡有一些警告說存在一些字段並沒有直接被使用，這提醒了我們並沒有清理任何內容。當使用 <span class="keystroke">ctrl-C</span> 終止主線程，所有其他線程也會立刻停止，即便他們正在處理一個請求。

現在我們要為 `ThreadPool` 實現 `Drop` trait 對線程池中的每一個線程調用 `join`，這樣這些線程將會執行完他們的請求。接著會為 `ThreadPool` 實現一個方法來告訴線程他們應該停止接收新請求並結束。為了實踐這些代碼，修改 server 在 graceful Shutdown 之前只接受兩個請求。

現在開始為線程池實現 `Drop`。當線程池被丟棄時，應該 join 所有線程以確保他們完成其操作。列表 20-22 展示了 `Drop` 實現的第一次嘗試；這些代碼還不能夠編譯：

<span class="filename">文件名: src/lib.rs</span>

```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        for worker in &mut self.workers {
            println!("Shutting down worker {}", worker.id);

            worker.thread.join().unwrap();
        }
    }
}
```

<span class="caption">列表 20-22：當線程池離開作用域時 join 每個線程</span>

這裡遍歷線程池中的每個 `workers`，這裡使用了 `&mut` 因為 `self` 本身是一個可變引用而且也需要能夠修改 `worker`。當特定 worker 關閉時會打印出說明信息，接著在對應 worker 上調用 `join`。如果 `join` 失敗了，通過 `unwrap` 將錯誤變為 panic 從而無法進行 graceful Shutdown。

如下是嘗試編譯代碼時得到的錯誤：

```
error[E0507]: cannot move out of borrowed content
  --> src/lib.rs:65:13
   |
65 |             worker.thread.join().unwrap();
   |             ^^^^^^ cannot move out of borrowed content
```

因為我們只有每個 `worker` 的可變借用，並不能調用 `join`：`join` 獲取其參數的所有權。為瞭解決這個問題，需要一個方法將 `thread` 移動出擁有其所有權的 `Worker` 實例以便 `join` 可以消費這個線程。列表 17-15 中我們曾見過這麼做的方法：如果 `Worker` 存放的是 `Option<thread::JoinHandle<()>`，就可以在 `Option` 上調用 `take` 方法將值從 `Some` 成員中移動出來而對 `None` 成員不做處理。換句話說，正在運行的 `Worker` 的 `thread` 將是 `Some` 成員值，而當需要清理 worker 時，將 `Some` 替換為 `None`，這樣 worker 就沒有可以運行的線程了。

所以我們知道了需要更新 `Worker` 的定義為如下：

<span class="filename">文件名: src/lib.rs</span>

```rust
# use std::thread;
struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}
```

現在依靠編譯器來找出其他需要修改的地方。我們會得到兩個錯誤：

```
error: no method named `join` found for type
`std::option::Option<std::thread::JoinHandle<()>>` in the current scope
  --> src/lib.rs:65:27
   |
65 |             worker.thread.join().unwrap();
   |                           ^^^^

error[E0308]: mismatched types
  --> src/lib.rs:89:21
   |
89 |             thread,
   |             ^^^^^^ expected enum `std::option::Option`, found
   struct `std::thread::JoinHandle`
   |
   = note: expected type `std::option::Option<std::thread::JoinHandle<()>>`
              found type `std::thread::JoinHandle<_>`
```

第二個錯誤指向 `Worker::new` 結尾的代碼；當新建 `Worker` 時需要將 `thread` 值封裝進 `Some`：

<span class="filename">文件名: src/lib.rs</span>

```rust
impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        // ...snip...

        Worker {
            id,
            thread: Some(thread),
        }
    }
}
```

第一個錯誤有關 `Drop` 實現，而且我們提到過要調用 `Option` 上的 `take` 將 `thread` 移動出 `worker`。如下是代碼：

<span class="filename">文件名: src/lib.rs</span>

```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        for worker in &mut self.workers {
            println!("Shutting down worker {}", worker.id);

            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}
```

如第十七章我們見過的，`Option` 上的 `take` 方法會取出 `Some` 而留下 `None`。使用 `if let` 解構 `Some` 並得到線程，接著在線程上調用 `join`。如果 worker 的線程已然是 `None`，就知道此時這個 worker 已經清理了其線程且無需做任何操作。

有了這些修改，代碼就能編譯且沒有任何警告。不過也有壞消息，這些代碼還不能以我們期望的方式運行。問題的關鍵在於 `Worker` 中分配的線程所運行的閉包中的邏輯：調用 `join` 並不會關閉線程，因為他們一直 `loop` 來尋找任務。如果採用這個實現來嘗試丟棄 `ThreadPool` ，則主線程會永遠阻塞在等待第一個線程結束上。

為了修復這個問題，修改線程既監聽是否有 `Job` 運行也要監聽應該停止監聽並退出無限循環的信號。所以通道將發送這個枚舉的兩個成員之一而不再直接使用 `Job` 實例：

<span class="filename">文件名: src/lib.rs</span>

```rust
# struct Job;
enum Message {
    NewJob(Job),
    Terminate,
}
```

`Message` 枚舉要麼是存放了線程需要運行的 `Job` 的 `NewJob` 成員，要麼是會導致線程退出循環並終止的 `Terminate` 成員。

同時需要修改通道來使用 `Message` 類型值而不是 `Job`，如列表 20-23 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Message>,
}

// ...snip...

impl ThreadPool {
    // ...snip...
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();

        // ...snip...
    }

    pub fn execute<F>(&self, f: F)
        where
            F: FnOnce() + Send + 'static
    {
        let job = Box::new(f);

        self.sender.send(Message::NewJob(job)).unwrap();
    }
}

// ...snip...

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Message>>>) ->
        Worker {

        let thread = thread::spawn(move ||{
            loop {
                let message = receiver.lock().unwrap().recv().unwrap();

                match message {
                    Message::NewJob(job) => {
                        println!("Worker {} got a job; executing.", id);

                        job.call_box();
                    },
                    Message::Terminate => {
                        println!("Worker {} was told to terminate.", id);

                        break;
                    },
                }
            }
        });

        Worker {
            id,
            thread: Some(thread),
        }
    }
}
```

<span class="caption">列表 20-23：收發 `Message` 值並在 `Worker` 收到 `Message::Terminate` 時退出循環</span>

需要將 `ThreadPool` 定義、創建通道的 `ThreadPool::new` 和 `Worker::new` 簽名中的 `Job` 改為 `Message`。`ThreadPool` 的 `execute` 方法需要發送封裝進 `Message::NewJob` 成員的任務，當獲取到 `NewJob` 時會處理任務而收到 `Terminate` 成員時則會退出循環。

通過這些修改，代碼再次能夠編譯並按照期望的行為運行。不過還是會得到一個警告，因為並沒有在任何消息中使用 `Terminate` 成員。如列表 20-14 所示那樣修改 `Drop` 實現：

<span class="filename">文件名: src/lib.rs</span>

```rust
impl Drop for ThreadPool {
    fn drop(&mut self) {
        println!("Sending terminate message to all workers.");

        for _ in &mut self.workers {
            self.sender.send(Message::Terminate).unwrap();
        }

        println!("Shutting down all workers.");

        for worker in &mut self.workers {
            println!("Shutting down worker {}", worker.id);

            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}
```

<span class="caption">列表 20-24：在對每個 worker 線程調用 `join` 之前向 worker 發送 `Message::Terminate`</span>

現在遍歷了 worker 兩次，一次向每個 worker 發送一個 `Terminate` 消息，一個調用每個 worker 線程上的  `join`。如果嘗試在同一循環中發送消息並立即 join 線程，則無法保證當前迭代的 worker 是從通道收到終止消息的 worker。

為了更好的理解為什麼需要兩個分開的循環，想像一下只有兩個 worker 的場景。如果在一個循環中遍歷每個 worker，在第一次迭代中 `worker` 是第一個 worker，我們向通道發出終止消息並對第一個 worker 線程調用 `join`。如果第一個 worker 當時正忙於處理請求，則第二個 worker 會從通道接收這個終止消息並結束。而我們在等待第一個 worker 結束，不過它永遠也不會結束因為第二個線程取走了終止消息。現在我們就阻塞在了等待第一個 worker 結束，而無法發出第二條終止消息。死鎖！

為了避免此情況，首先從通道中取出所有的 `Terminate` 消息，接著 join 所有的線程。因為每個 worker 一旦收到終止消息即會停止從通道接收消息，我們就可以確保如果發送同 worker 數相同的終止消息，在 join 之前每個線程都會收到一個終止消息。

為了實踐這些代碼，如列表 20-25 所示修改 `main` 在 graceful Shutdown server 之前只接受兩個請求：

<span class="filename">文件名: src/bin/main.rs</span>

```rust
fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    let pool = ThreadPool::new(4);

    let mut counter = 0;

    for stream in listener.incoming() {
        if counter == 2 {
            println!("Shutting down.");
            break;
        }

        counter += 1;

        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }
}
```

<span class="caption">列表 20-25：在處理兩個請求之後通過退出循環來停止 server</span>

只處理兩次請求並不是生產環境的 web server 所期望的行為，不過這可以讓我們看清 graceful shutdown 和清理起作用了，因為不用再通過 <span class="keystroke">ctrl-C</span> 停止 server 了。

這裡還增加了一個 `counter` 變量在每次收到 TCP 流時遞增。如果計數到達 2，會停止處理請求並退出 `for` 循環。`ThreadPool` 會在 `main` 的結尾離開作用域，而且還會看到 `drop` 實現的運行。

使用 `cargo run` 啟動 server，並發起三個請求。第三個請求應該會失敗，而終端的輸出應該看起來像這樣：

```
$ cargo run
   Compiling hello v0.1.0 (file:///projects/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 1.0 secs
     Running `target/debug/hello`
Worker 0 got a job; executing.
Worker 3 got a job; executing.
Shutting down.
Sending terminate message to all workers.
Shutting down all workers.
Shutting down worker 0
Worker 1 was told to terminate.
Worker 2 was told to terminate.
Worker 0 was told to terminate.
Worker 3 was told to terminate.
Shutting down worker 1
Shutting down worker 2
Shutting down worker 3
```

當然，你可能會看到不同順序的輸出。可以從信息中看到服務是如何運行的： worker 0 和 worker 3 獲取了頭兩個請求，接著在第三個請求時，我們停止接收連接。當 `ThreadPool` 在 `main` 的結尾離開作用域時，其 `Drop` 實現開始工作，線程池通知所有線程終止。每個 worker 在收到終止消息時會打印出一個信息，接著線程池調用 `join` 來終止每一個 worker 線程。

這個特定的運行過程中一個有趣的地方在於：注意我們向通道中發出終止消息，而在任何線程收到消息之前，就嘗試 join worker 0 了。worker 0 還沒有收到終止消息，所以主線程阻塞直到 worker 0 結束。與此同時，每一個線程都收到了終止消息。一旦 worker 0 結束，主線程就等待其他 worker 結束，此時他們都已經收到終止消息並能夠停止了。

恭喜！現在我們完成了這個項目，也有了一個使用線程池異步響應請求的基礎 web server。我們能對 server 執行 graceful shutdown，它會清理線程池中的所有線程。如下是完整的代碼參考：

<span class="filename">Filename: src/bin/main.rs</span>

```rust
extern crate hello;
use hello::ThreadPool;

use std::io::prelude::*;
use std::net::TcpListener;
use std::net::TcpStream;
use std::fs::File;
use std::thread;
use std::time::Duration;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();
    let pool = ThreadPool::new(4);

    let mut counter = 0;

    for stream in listener.incoming() {
        if counter == 2 {
            println!("Shutting down.");
            break;
        }

        counter += 1;

        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }
}

fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 512];
    stream.read(&mut buffer).unwrap();

    let get = b"GET / HTTP/1.1\r\n";
    let sleep = b"GET /sleep HTTP/1.1\r\n";

    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK\r\n\r\n", "hello.html")
    } else if buffer.starts_with(sleep) {
        thread::sleep(Duration::from_secs(5));
        ("HTTP/1.1 200 OK\r\n\r\n", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND\r\n\r\n", "404.html")
    };

     let mut file = File::open(filename).unwrap();
     let mut contents = String::new();

     file.read_to_string(&mut contents).unwrap();

     let response = format!("{}{}", status_line, contents);

     stream.write(response.as_bytes()).unwrap();
     stream.flush().unwrap();
}
```

<span class="filename">Filename: src/lib.rs</span>

```rust
use std::thread;
use std::sync::mpsc;
use std::sync::Arc;
use std::sync::Mutex;

enum Message {
    NewJob(Job),
    Terminate,
}

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Message>,
}

trait FnBox {
    fn call_box(self: Box<Self>);
}

impl<F: FnOnce()> FnBox for F {
    fn call_box(self: Box<F>) {
        (*self)()
    }
}

type Job = Box<FnBox + Send + 'static>;

impl ThreadPool {
    /// Create a new ThreadPool.
    ///
    /// The size is the number of threads in the pool.
    ///
    /// # Panics
    ///
    /// The `new` function will panic if the size is zero.
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

    pub fn execute<F>(&self, f: F)
        where
            F: FnOnce() + Send + 'static
    {
        let job = Box::new(f);

        self.sender.send(Message::NewJob(job)).unwrap();
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        println!("Sending terminate message to all workers.");

        for _ in &mut self.workers {
            self.sender.send(Message::Terminate).unwrap();
        }

        println!("Shutting down all workers.");

        for worker in &mut self.workers {
            println!("Shutting down worker {}", worker.id);

            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}

struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Message>>>) ->
        Worker {

        let thread = thread::spawn(move ||{
            loop {
                let message = receiver.lock().unwrap().recv().unwrap();

                match message {
                    Message::NewJob(job) => {
                        println!("Worker {} got a job; executing.", id);

                        job.call_box();
                    },
                    Message::Terminate => {
                        println!("Worker {} was told to terminate.", id);

                        break;
                    },
                }
            }
        });

        Worker {
            id,
            thread: Some(thread),
        }
    }
}
```

這裡還有很多可以做的事！如果你希望繼續增強這個項目，如下是一些點子：

- 為 `ThreadPool` 和其公有方法增加更多文檔
- 為庫的功能增加測試
- 將 `unwrap` 調用改為更健壯的錯誤處理
- 使用 `ThreadPool` 進行其他不同於處理網絡請求的任務
- 在 crates.io 尋找一個線程池 crate 並使用它實現一個類似的 web server，將其 API 和魯棒性與我們的實現做對比

## 總結

好極了！你結束了本書的學習！由衷感謝你與我們一道加入這次 Rust 之旅。現在你已經準備好出發並實現自己的 Rust 項目或幫助他人了。請不要忘記我們的社區，這裡有其他 Rustaceans 正樂於幫助你迎接 Rust 之路上的任何挑戰。