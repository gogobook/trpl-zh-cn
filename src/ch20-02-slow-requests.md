## 慢請求如何影響吞吐率

> [ch20-02-slow-requests.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch20-02-slow-requests.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

目前 server 會依次處理每一個請求。這對於向我們這樣並不期望有非常大量請求的服務來說是可行的，不過隨著程序變得更複雜，這樣的串行處理並不是最優的。

因為當前的程序順序處理處理連接，在完成第一個連接的處理之前不會處理第二個連接。如果一個請求花費很長時間來處理，這段時間接收的請求則不得不等待這個長請求結束，即便這些新請求可以很快就處理完。讓我們實際嘗試一下。

### 在當前 server 實現中模擬慢請求

讓我們看看一個花費很長時間處理的請求對當前的 server 實現有何影響。示例 20-10 展示了對另一個請求的響應代碼，`/sleep`，它會使 server 在響應之前休眠五秒。這將模擬一個慢請求以便體現出 server 在串行的處理請求。

<span class="filename">文件名: src/main.rs</span>

```rust
use std::thread;
use std::time::Duration;
# use std::io::prelude::*;
# use std::net::TcpStream;
# use std::fs::File;
// ...snip...

fn handle_connection(mut stream: TcpStream) {
#     let mut buffer = [0; 512];
#     stream.read(&mut buffer).unwrap();
    // ...snip...

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

    // ...snip...
}
```

<span class="caption">示例 20-10：通過識別 `/sleep` 並休眠五秒來模擬慢請求</span>

這段代碼有些凌亂，不過對於模擬的目的來說已經足夠！這裡創建了第二個請求 `sleep`，我們會識別其數據。在 `if` 塊之後增加了一個 `else if` 來檢查 `/sleep` 請求，當發現這個請求時，在渲染歡迎頁面之前會先休眠五秒。

現在就可以真切的看出我們的 server 有多麼的原始；真實的庫將會以更簡潔的方式處理多請求識別問題。

使用 `cargo run` 啟動 server，並接著打開兩個瀏覽器窗口：一個請求 `http://localhost:8080/` 而另一個請求 `http://localhost:8080/sleep`。如果像之前一樣多次請求 `/`，會發現響應的比較快速。不過如果請求`/sleep` 之後在請求 `/`，就會看到 `/` 會等待直到 `sleep` 休眠完五秒之後才出現。

這裡有多種辦法來改變我們的 web server 使其避免所有請求都排在慢請求之後；其一便是實現一個線程池。

### 使用線程池改善吞吐量

**線程池**（*thread pool*）是一組預先分配的用來處理任務的線程。當程序收到一個新任務，線程池中的一個線程會被分配任務並開始處理。其餘的線程則可用於處理在第一個線程處理任務的同時處理其他接收到的任務。當第一個線程處理完任務時，它會返回空閒線程池中等待處理新任務。

線程池允許我們並發處理連接：可以在老連接處理完之前就開始處理新連接。這增加了 server 的吞吐量。

如下是我們將要實現的：不再等待每個請求處理完才開始下一個，我們將每個連接的處理發送給不同的線程。這些線程來此程序啟動時分配的四個線程的線程池。限制較少的線程數的原因是如果為每個新來的請求都創建一個新線程，則千萬級的請求就造成災難，他們會用盡服務器的資源並導致所有請求的處理都被終止。

不同於分配無限的線程，線程池中將有固定數量的等待線程。當新進請求時，將請求發送到線程池中做處理。線程池會維護一個接收請求的隊列。每一個線程會從隊列中取出一個請求，處理請求，接著向對隊列索取另一個請求。通過這種設計，則可以並發處理 `N` 個請求，其中 `N` 為線程數。這仍然意味著 `N` 個慢請求會阻塞隊列中的請求，不過確實將能夠處理的慢請求數量從一增加到了 `N`。

這個設計是多種改善 web server 吞吐量的方法之一。不過本書並不是有關 web server 的，所以這一種方法是我們將要涉及的。其他的方法有 fork/join 模型和單線程異步 I/O 模型。如果你對這個主題感興趣，則可以閱讀更多關於其他解決方案的內容並嘗試用 Rust 實現他們；對於一個像 Rust 這樣的底層語言，所有這些方法都是可能的。