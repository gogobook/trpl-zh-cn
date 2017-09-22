## 使用消息傳遞在線程間傳送數據

> [ch16-02-message-passing.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch16-02-message-passing.md)
> <br>
> commit da15de39eaabd50100d6fa662c653169254d9175

最近人氣正在上升的一個並發方式是**消息傳遞**（*message passing*），這裡線程或 actor 通過發送包含數據的消息來溝通。這個思想來源於口號：

> Do not communicate by sharing memory; instead, share memory by
> communicating.
>
> 不要共享內存來通訊；而是要通訊來共享內存。
>
> --[Effective Go](http://golang.org/doc/effective_go.html)

實現這個目標的主要工具是**通道**（*channel*）。通道有兩部分組成，一個發送者（transmitter）和一個接收者（receiver）。代碼的一部分可以調用發送者和想要發送的數據，而另一部分代碼可以在接收的那一端收取消息。

我們將編寫一個例子使用一個線程生成值並向通道發送他們。主線程會接收這些值並打印出來。

首先，如列表 16-6 所示，先創建一個通道但不做任何事：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();
#     tx.send(()).unwrap();
}
```

<span class="caption">Listing 16-6: Creating a channel and assigning the two
halves to `tx` and `rx`</span>

`mpsc::channel`函數創建一個新的通道。`mpsc`是**多個生產者，單個消費者**（*multiple producer, single consumer*）的縮寫。簡而言之，可以有多個產生值的**發送端**，但只能有一個消費這些值的**接收端**。現在我們以一個單獨的生產者開始，不過一旦例子可以工作了就會增加多個生產者。

`mpsc::channel`返回一個元組：第一個元素是發送端，而第二個元素是接收端。由於歷史原因，很多人使用`tx`和`rx`作為**發送者**和**接收者**的縮寫，所以這就是我們將用來綁定這兩端變量的名字。這裡使用了一個`let`語句和模式來解構了元組。第十八章會討論`let`語句中的模式和解構。

讓我們將發送端移動到一個新建線程中並發送一個字符串，如列表 16-7 所示：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });
}
```

<span class="caption">Listing 16-7: Moving `tx` to a spawned thread and sending
"hi"</span>

正如上一部分那樣使用`thread::spawn`來創建一個新線程。並使用一個`move`閉包來將`tx`移動進閉包這樣新建線程就是其所有者。

通道的發送端有一個`send`方法用來獲取需要放入通道的值。`send`方法返回一個`Result<T, E>`類型，因為如果接收端被丟棄了，將沒有發送值的目標，所以發送操作會出錯。在這個例子中，我們簡單的調用`unwrap`來忽略錯誤，不過對於一個真實程序，需要合理的處理它。第九章是你複習正確錯誤處理策略的好地方。

在列表 16-8 中，讓我們在主線程中從通道的接收端獲取值：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

<span class="caption">Listing 16-8: Receiving the value "hi" in the main thread
and printing it out</span>

通道的接收端有兩個有用的方法：`recv`和`try_recv`。這裡，我們使用了`recv`，它是 *receive* 的縮寫。這個方法會阻塞執行直到從通道中接收一個值。一旦發送了一個值，`recv`會在一個`Result<T, E>`中返回它。當通道發送端關閉，`recv`會返回一個錯誤。`try_recv`不會阻塞；相反它立刻返回一個`Result<T, E>`。

如果運行列表 16-8 中的代碼，我們將會看到主線程打印出這個值：

```
Got: hi
```

### 通道與所有權如何交互

現在讓我們做一個試驗來看看通道與所有權如何在一起工作：我們將嘗試在新建線程中的通道中發送完`val`之後再使用它。嘗試編譯列表 16-9 中的代碼：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let val = String::from("hi");
        tx.send(val).unwrap();
        println!("val is {}", val);
    });

    let received = rx.recv().unwrap();
    println!("Got: {}", received);
}
```

<span class="caption">Listing 16-9: Attempting to use `val` after we have sent
it down the channel</span>

這裡嘗試在通過`tx.send`發送`val`到通道中之後將其打印出來。這是一個壞主意：一旦將值發送到另一個線程後，那個線程可能會在我們在此使用它之前就修改或者丟棄它。這會由於不一致或不存在的數據而導致錯誤或意外的結果。

嘗試編譯這些代碼，Rust 會報錯：

```
error[E0382]: use of moved value: `val`
  --> src/main.rs:10:31
   |
9  |         tx.send(val).unwrap();
   |                 --- value moved here
10 |         println!("val is {}", val);
   |                               ^^^ value used here after move
   |
   = note: move occurs because `val` has type `std::string::String`, which does
   not implement the `Copy` trait
```

我們的並發錯誤會造成一個編譯時錯誤！`send`獲取其參數的所有權並移動這個值歸接收者所有。這個意味著不可能意外的在發送後再次使用這個值；所有權系統檢查一切是否合乎規則。

在這一點上，消息傳遞非常類似於 Rust 的單所有權系統。消息傳遞的擁護者出於相似的原因支持消息傳遞，就像 Rustacean 們欣賞 Rust 的所有權一樣：單所有權意味著特定類型問題的消失。如果一次只有一個線程可以使用某些內存，就沒有出現數據競爭的機會。

### 發送多個值並觀察接收者的等待

列表 16-8 中的代碼可以編譯和運行，不過這並不是很有趣：通過它難以看出兩個獨立的線程在一個通道上相互通訊。列表 16-10 則有一些改進會證明這些代碼是並發執行的：新建線程現在會發送多個消息並在每個消息之間暫停一段時間。

<span class="filename">Filename: src/main.rs</span>

```rust
use std::thread;
use std::sync::mpsc;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::new(1, 0));
        }
    });

    for received in rx {
        println!("Got: {}", received);
    }
}
```

<span class="caption">Listing 16-10: Sending multiple messages and pausing
between each one</span>

這一次，在新建線程中有一個字符串 vector 希望發送到主線程。我們遍歷他們，單獨的發送每一個字符串並通過一個`Duration`值調用`thread::sleep`函數來暫停一秒。

在主線程中，不再顯式的調用`recv`函數：而是將`rx`當作一個迭代器。對於每一個接收到的值，我們將其打印出來。當通道被關閉時，迭代器也將結束。

當運行列表 16-10 中的代碼時，將看到如下輸出，每一行都會暫停一秒：

```
Got: hi
Got: from
Got: the
Got: thread
```

在主線程中並沒有任何暫停或位於`for`循環中用於等待的代碼，所以可以說主線程是在等待從新建線程中接收值。

### 通過克隆發送者來創建多個生產者

差不多在本部分的開頭，我們提到了`mpsc`是 *multiple producer, single consumer* 的縮寫。可以擴展列表 16-11 中的代碼來創建都向同一接收者發送值的多個線程。這可以通過克隆通道的發送端在來做到，如列表 16-11 所示：

<span class="filename">Filename: src/main.rs</span>

```rust
# use std::thread;
# use std::sync::mpsc;
# use std::time::Duration;
#
# fn main() {
// ...snip...
let (tx, rx) = mpsc::channel();

let tx1 = tx.clone();
thread::spawn(move || {
    let vals = vec![
        String::from("hi"),
        String::from("from"),
        String::from("the"),
        String::from("thread"),
    ];

    for val in vals {
        tx1.send(val).unwrap();
        thread::sleep(Duration::new(1, 0));
    }
});

thread::spawn(move || {
    let vals = vec![
        String::from("more"),
        String::from("messages"),
        String::from("for"),
        String::from("you"),
    ];

    for val in vals {
        tx.send(val).unwrap();
        thread::sleep(Duration::new(1, 0));
    }
});
// ...snip...
#
#     for received in rx {
#         println!("Got: {}", received);
#     }
# }
```

<span class="caption">Listing 16-11: Sending multiple messages and pausing
between each one</span>

這一次，在創建新線程之前，我們對通道的發送端調用了`clone`方法。這會給我們一個可以傳遞給第一個新建線程的發送端句柄。我們會將原始的通道發送端傳遞給第二個新建線程，這樣每個線程將向通道的接收端發送不同的消息。

如果運行這些代碼，你**可能**會看到這樣的輸出：

```
Got: hi
Got: more
Got: from
Got: messages
Got: for
Got: the
Got: thread
Got: you
```

雖然你可能會看到這些以不同的順序出現。這依賴於你的系統！這也就是並發既有趣又困難的原因。如果你拿`thread::sleep`做實驗，在不同的線程中提供不同的值，就會發現他們的運行更加不確定並每次都會產生不同的輸出。

現在我們見識過了通道如何工作，再看看共享內存並發吧。