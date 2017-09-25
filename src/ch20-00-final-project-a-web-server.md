## 單線程 web server

> [ch20-01-single-threaded.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch20-01-single-threaded.md)
> <br>
> commit 2e269ff82193fd65df8a87c06561d74b51ac02f7

首先讓我們創建一個可運行的單線程 web server。我們將處理 TCP 和 HTTP 請求和響應的原始字節來從 server 向瀏覽器發送 HTML。首先先快速瞭解一下涉及到的協議。

**超文本傳輸協議**（*Hypertext Transfer Protocol*，*HTTP*）驅動著現在的互聯網，它構建於 **傳輸控制協議**（*Transmission Control Protocol*，*TCP*）的基礎上。這裡並不會過多的涉及細節，只做簡單的概括：TCP 是一個底層協議，HTTP 是 TCP 之上的高級協議。這兩個都是一種被稱為 **請求-響應協議**（*request-response protocol*）的協議，也就是說，有 **客戶端**（*client*）來初始化請求，並有 **服務端**（*server*）監聽請求並向客戶端提供響應。請求與響應的內容由協議本身定義。

TCP 描述了信息如何從一個 server 到另一個的底層細節，不過它並不指定信息是什麼；它僅僅是一堆 0 和 1。HTTP 構建於 TCP 之上，它定義了請求和響應的內容。為此，技術上講可將 HTTP 用於其他協議之上，不過對於絕大部分情況，HTTP 通過 TCP 傳輸。

所以我們的 web server 所需做的第一件事便是能夠監聽 TCP 連接。標準庫有 `std::net` 模塊處理這些功能。現在創建一個新項目：

```
$ cargo new hello --bin
     Created binary (application) `hello` project
$ cd hello
```

並在 `src/main.rs` 放入列表 20-1 中的代碼作為開始。這段代碼會在地址 `127.0.0.1:8080` 上監聽傳入的 TCP 流。當抓取到傳入的流，它會打印出 `Connection established!`：

<span class="filename">文件名: src/main.rs</span>

```rust
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        println!("Connection established!");
    }
}
```

<span class="caption">列表 20-1：監聽傳入的流並在接收到流時打印信息</span>

`TcpListener` 用於監聽 TCP 連接。我們選擇監聽地址 `127.0.0.1:8080`。冒號之前的部分是一個代表本機的 IP 地址，而 `8080` 是端口。選擇這個端口是因為通常 HTTP 監聽 80 端口，不過連接 80 端口需要管理員權限。普通用戶可以監聽大於 1024 的端口；8080 端口易於記憶因為它重複了 HTTP 的 80 端口兩次。

`bind` 函數類似於 `new` 因為它返回一個新的 `TcpListener` 實例，不過 `bind` 是一個符合這個領域術語的描述性名稱。在網絡領域，人們通常說「綁定到一個端口」（「binding to a port」），所以標準庫中將創建新 `TcpListener` 的函數定義為 `bind`。

`bind` 函數返回 `Result<T, E>`。綁定可能會失敗，例如，如果不是管理員嘗試連接 80 端口。另一個綁定會失敗的情況是兩個程序監聽相同的端口，這可能發生於運行兩個本程序的實例時。因為我們編寫的是一個基礎的 server，將不會擔心處理這類錯誤，`unwrap` 使得出現這些情況時直接停止程序。

`TcpListener` 的 `incoming` 方法返回一個迭代器，它提供了一系列的流（更準確的說是 `TcpStream` 類型的流）。**流**（*stream*）代表一個客戶端和服務端之間打開的連接。**連接**（*connection*）代表客戶端連接服務端、服務端生成響應以及服務端關閉連接的全部請求 / 響應過程。為此，`TcpStream` 允許我們讀取它來查看客戶端發送了什麼，並可以編寫響應。所以這個 `for` 循環會依次處理每個連接並產生一系列的流供我們處理。

目前為止，處理流意味著調用 `unwrap` 在出現任何錯誤時終止程序，接著打印信息。因為我們實際上沒有遍歷連接，而是遍歷 **連接嘗試**（*connection attempts*），所以可能出現錯誤。連接可能會因為很多原因不能成功，大部分是操作系統相關的。例如，很多系統限制同時打開的連接數；新連接嘗試產生錯誤，直到一些打開的連接關閉為止。

讓我們試試這段代碼！首先在終端執行 `cargo run`，接著在瀏覽器中加載 `127.0.0.1:8080`。瀏覽器會顯示出看起來類似於「連接重置」（「Connection reset」）的錯誤信息，因為目前並沒響應任何數據。但是如果我們觀察終端，會發現當瀏覽器連接 server 時會打印出一系列的信息！

```
     Running `target/debug/hello`
Connection established!
Connection established!
Connection established!
```

對於一次瀏覽器請求會打印出多條信息；這些連接可能是瀏覽器請求頁面和請求出現在瀏覽器 tab 頁中的 `favicon.ico`，或者瀏覽器可能重試了連接。瀏覽器期望使用 HTTP 交互，不過我們並未恢復任何內容。當 `stream` 在循環的結尾離開作用域並被丟棄，其連接將被關閉，作為 `TcpStream` 的 `drop` 實現的一部分。瀏覽器有時通過重連來處理關閉的連接，因為這些問題可能是暫時的。現在重要的是我們成功的處理了 TCP 連接！

記得當運行完特定版本的代碼後使用 <span class="keystroke">ctrl-C</span> 來停止程序，並在做出最新的代碼修改之後執行 `cargo run` 重啟服務。

### 讀取請求

下面讀取瀏覽器的請求！因為我們增加了出於處理連接目的的功能。開始一個新函數來將設置 server 和連接，與處理每個請求分離，踐行關注分離原則。在這個新的 `handle_connection` 函數中，從 `stream` 中讀取數據並打印出來以便觀察瀏覽器發送過來的數據。將代碼修改為如列表 20-2 所示：

<span class="filename">文件名: src/main.rs</span>

```rust,no_run
use std::io::prelude::*;
use std::net::TcpListener;
use std::net::TcpStream;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:8080").unwrap();

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        handle_connection(stream);
    }
}

fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 512];

    stream.read(&mut buffer).unwrap();

    println!("Request: {}", String::from_utf8_lossy(&buffer[..]));
}
```

<span class="caption">列表 20-2：讀取 `TcpStream` 並打印數據</span>

在開頭增加 `std::io::prelude` 以便將讀寫流所需的 trait 引入作用域。相比在 `main` 的 `for` 中在抓取到連接時打印信息，現在調用新的 `handle_connection` 函數並向其傳遞 `stream`。

在 `handle_connection` 中，通過 `mut` 關鍵字將 `stream` 參數變為可變。我們將從流中讀取數據，所以它需要是可修改的。

接下來，需要實際讀取流。這裡分兩步進行：首先，在棧上聲明一個 `buffer` 來存放讀取到的數據。這裡創建了一個 512 字節的緩衝區，它足以存放基本請求的數據。這對於本章的目的來說是足夠的。如果希望處理任意大小的請求，管理所需的緩衝區將更複雜，不過現在一切從簡。接著將緩衝區傳遞給 `stream.read` ，它會從 `TcpStream` 中讀取字節並放入緩衝區中。

接下來將緩衝區中的字節轉換為字符串並打印出來。`String::from_utf8_lossy` 函數抓取一個 `&[u8]` 並產生一個 `String`。函數名的 「lossy」 部分來源於當其遇到無效的 UTF-8 序列時的行為：它使用  �，`U+FFFD REPLACEMENT CHARACTER`，來代替無效序列。你可能會在緩衝區的剩餘部分看到這些替代字符，因為他們沒有被請求數據填滿。

讓我們試一試！啟動程序並再次在瀏覽器中發起請求。注意瀏覽器中仍然會出現錯誤頁面，不過終端中程序的輸出現在看起來像這樣：

```
$ cargo run
   Compiling hello v0.1.0 (file:///projects/hello)
    Finished dev [unoptimized + debuginfo] target(s) in 0.42 secs
     Running `target/debug/hello`
Request: GET / HTTP/1.1
Host: 127.0.0.1:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64; rv:52.0) Gecko/20100101
Firefox/52.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
Upgrade-Insecure-Requests: 1
������������������������������������
```

根據使用的瀏覽器不同可能會出現稍微不同的數據。也可能會看到請求重複出現。現在我們打印出了請求數據，可以通過觀察 `Request: GET` 之後的路徑來解釋為何會從瀏覽器得到多個連接。如果重複的連接都是請求 `/`，就知道了瀏覽器嘗試重複抓取 `/` 因為它沒有得到響應。

拆開請求數據來理解瀏覽器向我們請求了什麼。HTTP 是一個基於文本的協議，而一個請求有如下格式：

```
Method Request-URI HTTP-Version CRLF
headers CRLF
message-body
```

第一行叫做 **請求行**（*request line*），它存放了客戶端請求了什麼的信息。請求行的第一部分是 *method*，比如 `GET` 或 `POST`，這描述了客戶端如何進行請求。

接著是請求的 *URI*，它代表 **統一資源標識符**（*Uniform Resource Identifier*），URI 大體上類似，但也不完全類似於 URL（**統一資源定位符**，*Uniform Resource Locators*），我們通常將其稱為輸入到瀏覽器中的地址。HTTP 規範使用術語 URI，而 URI 和 URL 之間的區別對於本章的目的來說並不重要，所以心理上將 URL 替換為 URI 即可。

接下來，是客戶端使用的 HTTP 版本，接著請求行以一個 CRLF 序列結尾。CRLF 序列也可以寫作 `\r\n`：`\r` 是 **回車**（*carriage return*）而 `\n` 是 **換行**（*line feed*）。這些術語來自打字機時代！CRLF 序列將請求行與其他請求數據分開。

看看代碼打印出的請求行的數據：

```text
GET / HTTP/1.1
```

`GET` 是 method，`/` 是請求 URI，而 `HTTP/1.1` 是版本。

從 `Host:` 開始的其餘的行是 headers；`GET` 請求沒有 body。

如果你希望的話，嘗試用不同的瀏覽器發送請求，或請求不同的地址，比如 `127.0.0.1:8080/test`，來觀察請求數據如何變化。

現在我們知道了瀏覽器請求了什麼。讓我們返回一些數據！

### 編寫響應

現在向瀏覽器返回數據以響應請求。響應有如下格式：

```
HTTP-Version Status-Code Reason-Phrase CRLF
headers CRLF
message-body
```

第一行叫做 **狀態行**（*status line*），它包含響應的 HTTP 版本、一個數字狀態碼用以總結請求的結果和一個描述之前狀態碼的文本原因。CRLF 序列之後是任意 header，另一個 CRLF 序列，和響應的 body。

這裡是一個使用 HTTP 1.1 版本的響應例子，其狀態碼為 `200`，原因描述為 `OK`，沒有 header，也沒有 body：

```
HTTP/1.1 200 OK\r\n\r\n
```

這些文本是一個微型的成功 HTTP 響應。讓我們把這些寫入流！去掉打印請求信息的 `println!` 行，並在這裡增加如列表 20-3 所示的代碼：

<span class="filename">文件名: src/main.rs</span>

```rust
# use std::io::prelude::*;
# use std::net::TcpStream;
fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 512];

    stream.read(&mut buffer).unwrap();

    let response = "HTTP/1.1 200 OK\r\n\r\n";

    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
```

<span class="caption">列表 20-3：將一個微型成功 HTTP 響應寫入流</span>

新代碼中的第一行定義了變數 `response` 來存放將要返回的成功響應的數據。接著，在 `response` 上調用 `as_bytes`，因為 `stream` 的 `write` 方法抓取一個 `&[u8]` 並直接將這些字節發送給連接。

`write` 可能會失敗，所以 `write` 返回 `Result<T, E>`；我們繼續使用 `unwrap` 以繼續本章的核心內容而不是處理錯誤。最後，`flush` 會等待直到所有字節都被寫入連接中；`TcpStream` 包含一個內部緩衝區來最小化對底層操作系統的調用。

有了這些修改，運行我們的代碼並進行請求！我們不再向終端打印任何數據，所以不會再看到除了 Cargo 以外的任何輸出。不過當在瀏覽器中加載 `127.0.0.1:8080` 時，會得到一個空頁面而不是錯誤。太棒了！我們剛剛手寫了一個 HTTP 請求與響應。

### 返回真正的 HTML

讓我們不只是返回空頁面。在項目根目錄創建一個新文件，*hello.html*，也就是說，不是在 `src` 目錄。在此可以放入任何你期望的 HTML；列表 20-4 展示了本書作者改採用的：

<span class="filename">文件名: hello.html</span>

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Hello!</title>
  </head>
  <body>
    <h1>Hello!</h1>
    <p>Hi from Rust</p>
  </body>
</html>
```

<span class="caption">列表 20-4：一個簡單的 HTML 文件用來作為響應</span>

這是一個極小化的 HTML 5 文檔，它有一個標題和一小段文本。如列表 20-5 所示修改 `handle_connection` 來讀取 HTML 文件，將其加入到響應的 body 中，並發送：

<span class="filename">文件名: src/main.rs</span>

```rust
# use std::io::prelude::*;
# use std::net::TcpStream;
use std::fs::File;

// ...snip...

fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 512];
    stream.read(&mut buffer).unwrap();

    let mut file = File::open("hello.html").unwrap();

    let mut contents = String::new();
    file.read_to_string(&mut contents).unwrap();

    let response = format!("HTTP/1.1 200 OK\r\n\r\n{}", contents);

    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
```

<span class="caption">列表 20-5：將 *hello.html* 的內容作為響應 body 發送</span>

在開頭增加了一行來將標準庫中的 `File` 引入作用域，打開和讀取文件的代碼應該看起來很熟悉，因為第十二章 I/O 項目的列表 12-4 中讀取文件內容時出現過類似的代碼。

接下來，使用 `format!` 將文件內容加入到將要寫入流的成功響應的 body 中。

使用 `cargo run` 運行程序，在瀏覽器加載 `127.0.0.1:8080`，你應該會看到渲染出來的 HTML 文件！

注意目前忽略了 `buffer` 中的請求數據並無條件的發送了 HTML 文件的內容。嘗試在瀏覽器中請求 `127.0.0.1:8080/something-else` 也會得到同樣的 HTML。對於所有請求都發送相同的響應其作用是非常有限的，也不是大部分 server 所做的；讓我們檢查請求並只對格式良好（well-formed）的請求 `/` 發送 HTML 文件。

### 驗證請求並有選擇的響應

目前我們的 server 不管客戶端請求什麼都會返回相同的 HTML 文件。讓我們檢查瀏覽器是否請求 `/`， 並在其請求其他內容時返回錯誤。如列表 20-6 所示修改 `handle_connection` ，它增加了所需的那部分代碼。這一部分將接收到的請求的內容與已知的 `/` 請求的一部分做比較，並增加了 `if` 和 `else` 塊來加入處理不同請求的代碼：

<span class="filename">Filename: src/main.rs</span>

```rust
# use std::io::prelude::*;
# use std::net::TcpStream;
# use std::fs::File;
// ...snip...

fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 512];
    stream.read(&mut buffer).unwrap();

    let get = b"GET / HTTP/1.1\r\n";

    if buffer.starts_with(get) {
        let mut file = File::open("hello.html").unwrap();

        let mut contents = String::new();
        file.read_to_string(&mut contents).unwrap();

        let response = format!("HTTP/1.1 200 OK\r\n\r\n{}", contents);

        stream.write(response.as_bytes()).unwrap();
        stream.flush().unwrap();
    } else {
        // some other request
    };
}
```

<span class="caption">列表 20-6：將請求與期望的 `/` 請求內容做匹配，並設置對 `/` 和其他請求的條件化處理</span>

這裡在變數 `get` 中硬編碼了所需的請求相關的數據。因為我們從緩衝區中讀取原始字節，所以使用了字節字符串，使用 `b""` 使得 `get` 也是一個字節字符串。接著檢查 `buffer` 是否以 `get` 中的字節開頭。如果是，這就是一個格式良好的 `/` 請求，也就是 `if` 塊中期望處理的成功情況。`if` 塊中包含列表 20-5 中增加的返回 HTML 文件內容的代碼。

如果 `buffer` 不以 `get` 中的字節開頭，就說明是其他請求。對於所有其他請求都將使用 `else` 塊中增加的代碼來響應。

如果運行代碼並請求 `127.0.0.1:8080`，就會得到 *hello.html* 中的 HTML。如果進行其他請求，比如 `127.0.0.1:8080/something-else`，則會得到像運行列表 20-1 和 20-2 中代碼那樣的連接錯誤。

如列表 20-7 所示向 `else` 塊增加代碼來返回一個帶有 `404` 狀態碼的響應，這代表了所請求的內容沒有找到。接著也會返回一個 HTML 向瀏覽器終端用戶表明此意：

<span class="filename">Filename: src/main.rs</span>

```rust
# use std::io::prelude::*;
# use std::net::TcpStream;
# use std::fs::File;
# fn handle_connection(mut stream: TcpStream) {
# if true {
// ...snip...

} else {
    let header = "HTTP/1.1 404 NOT FOUND\r\n\r\n";
    let mut file = File::open("404.html").unwrap();
    let mut contents = String::new();

    file.read_to_string(&mut contents).unwrap();

    let response = format!("{}{}", header, contents);

    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
# }
```

<span class="caption">列表 20-7：對於任何不是 `/` 的請求返回 `404` 狀態碼的響應和錯誤頁面</span>

這裡，響應頭有狀態碼 `404` 和原因短語 `NOT FOUND`。仍然沒有任何 header，而其 body 將是 *404.html* 文件中的 HTML。也在 *hello.html* 同級目錄創建 *404.html* 文件作為錯誤頁面；這一次也可以隨意使用任何 HTML 或使用列表 20-8 中的示例 HTML：

<span class="filename">文件名: 404.html</span>

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Hello!</title>
  </head>
  <body>
    <h1>Oops!</h1>
    <p>Sorry, I don't know what you're asking for.</p>
  </body>
</html>
```

<span class="caption">列表 20-8：任何 `404` 響應所返回錯誤頁面內容樣例</span>

有了這些修改，再次運行 server。請求 `127.0.0.1:8080` 應該會返回 *hello.html*，而對於任何其他請求，比如 `127.0.0.1:8080/foo`，應該會返回 *404.html* 中的錯誤 HTML！

`if` 和 `else` 塊中的代碼有很多的重複：他們都讀取文件並將其內容寫入流。這兩個情況唯一的區別是狀態行和文件名。將這些區別分別提取到一行 `if` 和 `else` 中，對狀態行和文件名變數賦值；然後在讀取文件和寫入響應的代碼中無條件的使用這些變數。重構後代碼後的結果如列表 20-9 所示：

<span class="filename">文件名: src/main.rs</span>

```rust
# use std::io::prelude::*;
# use std::net::TcpStream;
# use std::fs::File;
// ...snip...

fn handle_connection(mut stream: TcpStream) {
#     let mut buffer = [0; 512];
#     stream.read(&mut buffer).unwrap();
#
#     let get = b"GET / HTTP/1.1\r\n";
    // ...snip...

   let (status_line, filename) = if buffer.starts_with(get) {
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

<span class="caption">列表 20-9：重構代碼使得 `if` 和 `else` 塊中只包含兩個情況所不同的代碼</span>

這裡，`if` 和 `else` 塊所做的唯一的事就是在一個元組中返回合適的狀態行和文件名的值；接著使用第十八章講到的使用模式的 `let` 語句通過解構元組的兩部分給 `filename` 和 `header` 賦值。

讀取文件和寫入響應的冗餘代碼現在位於 `if` 和 `else` 塊之外，並會使用變數 `status_line` 和 `filename`。這樣更易於觀察這兩種情況真正有何不同，並且如果需要改變如何讀取文件或寫入響應時只需要更新一處的代碼。列表 20-9 中代碼的行為與列表 20-8 完全一樣。

好極了！我們有了一個 40 行左右 Rust 代碼的小而簡單的 server，它對一個請求返回頁面內容而對所有其他請求返回 `404` 響應。

不過因為 server 運行於單線程中，它一次只能處理一個請求。讓我們模擬一些慢請求來看看這如何會稱為一個問題。