## `panic!` 與不可恢復的錯誤

> [ch09-01-unrecoverable-errors-with-panic.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch09-01-unrecoverable-errors-with-panic.md)
> <br>
> commit 8d24b2a5e61b4eea109d26e38d2144408ae44e53

突然有一天，糟糕的事情發生了，而你對此束手無策。對於這種情況，Rust 有 `panic!`巨集。當執行這個巨集時，程序會打印出一個錯誤信息，展開並清理棧數據，然後接著退出。出現這種情況的場景通常是檢測到一些類型的 bug 而且程式設計師並不清楚該如何處理它。

> ### Panic 中的棧展開與終止
> 
> 當出現 `panic!` 時，程序預設會開始 **展開**（*unwinding*），這意味著 Rust 會回溯棧並清理它遇到的每一個函數的數據，不過這個回溯並清理的過程有很多工作。另一種選擇是直接 **終止**（*abort*），這會不清理數據就退出程序。那麼程序所使用的內存需要由作業系統來清理。如果你需要項目的最終二進制文件越小越好，可以由 panic 時展開切換為終止，通過在  *Cargo.toml* 的 `[profile]` 部分增加 `panic = 'abort'`。例如，如果你想要在發佈模式中 panic 時直接終止：
>
> ```toml
> [profile.release]
> panic = 'abort'
> ```

讓我們在一個簡單的程序中調用 `panic!`：

<span class="filename">文件名: src/main.rs</span>

```rust,should_panic
fn main() {
    panic!("crash and burn");
}
```

運行程序將會出現類似這樣的輸出：

```text
$ cargo run
   Compiling panic v0.1.0 (file:///projects/panic)
    Finished dev [unoptimized + debuginfo] target(s) in 0.25 secs
     Running `target/debug/panic`
thread 'main' panicked at 'crash and burn', src/main.rs:2
note: Run with `RUST_BACKTRACE=1` for a backtrace.
error: Process didn't exit successfully: `target/debug/panic` (exit code: 101)
```

最後三行包含 `panic!` 造成的錯誤信息。第一行顯示了 panic 提供的信息並指明了源碼中 panic 出現的位置：*src/main.rs:2* 表明這是 *src/main.rs* 文件的第二行。

在這個例子中，被指明的那一行是我們代碼的一部分，而且查看這一行的話就會發現 `panic!` 巨集的調用。換句話說，`panic!` 可能會出現在我們的代碼調用的代碼中。錯誤信息報告的文件名和行號可能指向別人代碼中的 `panic!` 巨集調用，而不是我們代碼中最終導致 `panic!` 的那一行。可以使用 `panic!` 被調用的函數的 backtrace 來尋找（我們代碼中出問題的地方）。

### 使用 `panic!` 的 backtrace

讓我們來看看另一個因為我們代碼中的 bug 引起的別的庫中 `panic!` 的例子，而不是直接的巨集調用：

<span class="filename">文件名: src/main.rs</span>

```rust,should_panic
fn main() {
    let v = vec![1, 2, 3];

    v[100];
}
```

這裡嘗試訪問 vector 的第一百個元素，不過它只有三個元素。這種情況下 Rust 會 panic。`[]` 應當返回一個元素，不過如果傳遞了一個無效索引，就沒有可供 Rust 返回的正確的元素。

這種情況下其他像 C 這樣語言會嘗直接試提供所要求的值，即便這可能不是你期望的：你會得到對任何應 vector 中這個元素的內存位置的值，甚至是這些內存並不屬於 vector 的情況。這被稱為 **緩衝區溢出**（*buffer overread*），並可能會導致安全漏洞，比如攻擊者可以像這樣操作索引來讀取儲存在數組後面不被允許的數據。

為了使程序遠離這類漏洞，如果嘗試讀取一個索引不存在的元素，Rust 會停止執行並拒絕繼續。嘗試運行上面的程序會出現如下：

```text
$ cargo run
   Compiling panic v0.1.0 (file:///projects/panic)
    Finished dev [unoptimized + debuginfo] target(s) in 0.27 secs
     Running `target/debug/panic`
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is
100', /stable-dist-rustc/build/src/libcollections/vec.rs:1362
note: Run with `RUST_BACKTRACE=1` for a backtrace.
error: Process didn't exit successfully: `target/debug/panic` (exit code: 101)
```

這指向了一個不是我們編寫的文件，*libcollections/vec.rs*。這是標準庫中 `Vec<T>` 的實現。這是當對 vector `v` 使用 `[]` 時 *libcollections/vec.rs* 中會執行的代碼，也是真正出現 `panic!` 的地方。

接下來的幾行提醒我們可以設置 `RUST_BACKTRACE` 環境變數來得到一個 backtrace 來調查究竟是什麼導致了錯誤。讓我們來試試看。示例 9-1 顯示了其輸出：

```text
$ RUST_BACKTRACE=1 cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `target/debug/panic`
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 100', /stable-dist-rustc/build/src/libcollections/vec.rs:1392
stack backtrace:
   1:     0x560ed90ec04c - std::sys::imp::backtrace::tracing::imp::write::hf33ae72d0baa11ed
                        at /stable-dist-rustc/build/src/libstd/sys/unix/backtrace/tracing/gcc_s.rs:42
   2:     0x560ed90ee03e - std::panicking::default_hook::{{closure}}::h59672b733cc6a455
                        at /stable-dist-rustc/build/src/libstd/panicking.rs:351
   3:     0x560ed90edc44 - std::panicking::default_hook::h1670459d2f3f8843
                        at /stable-dist-rustc/build/src/libstd/panicking.rs:367
   4:     0x560ed90ee41b - std::panicking::rust_panic_with_hook::hcf0ddb069e7abcd7
                        at /stable-dist-rustc/build/src/libstd/panicking.rs:555
   5:     0x560ed90ee2b4 - std::panicking::begin_panic::hd6eb68e27bdf6140
                        at /stable-dist-rustc/build/src/libstd/panicking.rs:517
   6:     0x560ed90ee1d9 - std::panicking::begin_panic_fmt::abcd5965948b877f8
                        at /stable-dist-rustc/build/src/libstd/panicking.rs:501
   7:     0x560ed90ee167 - rust_begin_unwind
                        at /stable-dist-rustc/build/src/libstd/panicking.rs:477
   8:     0x560ed911401d - core::panicking::panic_fmt::hc0f6d7b2c300cdd9
                        at /stable-dist-rustc/build/src/libcore/panicking.rs:69
   9:     0x560ed9113fc8 - core::panicking::panic_bounds_check::h02a4af86d01b3e96
                        at /stable-dist-rustc/build/src/libcore/panicking.rs:56
  10:     0x560ed90e71c5 - <collections::vec::Vec<T> as core::ops::Index<usize>>::index::h98abcd4e2a74c41
                        at /stable-dist-rustc/build/src/libcollections/vec.rs:1392
  11:     0x560ed90e727a - panic::main::h5d6b77c20526bc35
                        at /home/you/projects/panic/src/main.rs:4
  12:     0x560ed90f5d6a - __rust_maybe_catch_panic
                        at /stable-dist-rustc/build/src/libpanic_unwind/lib.rs:98
  13:     0x560ed90ee926 - std::rt::lang_start::hd7c880a37a646e81
                        at /stable-dist-rustc/build/src/libstd/panicking.rs:436
                        at /stable-dist-rustc/build/src/libstd/panic.rs:361
                        at /stable-dist-rustc/build/src/libstd/rt.rs:57
  14:     0x560ed90e7302 - main
  15:     0x7f0d53f16400 - __libc_start_main
  16:     0x560ed90e6659 - _start
  17:                0x0 - <unknown>
```

<span class="caption">示例 9-1：當設置 `RUST_BACKTRACE` 環境變數時 `panic!` 調用所生成的 backtrace 信息</span>

這裡有大量的輸出！backtrace 第 11 行指向了我們程序中引起錯誤的行：*src/main.rs* 的第四行。backtrace 是一個執行到目前位置所有被調用的函數的示例。Rust 的 backtrace 跟其他語言中的一樣：閱讀 backtrace 的關鍵是從頭開始讀直到發現你編寫的文件。這就是問題的發源地。這一行往上是你的代碼調用的代碼；往下則是調用你的代碼的代碼。這些行可能包含核心 Rust 代碼，標準庫代碼或用到的 crate 代碼。

如果你不希望我們的程序 panic，第一個提到我們編寫的代碼行的位置是你應該開始調查的，以便查明是什麼值如何在這個地方引起了 panic。在上面的例子中，我們故意編寫會 panic 的代碼來演示如何使用 backtrace，修復這個 panic 的方法就是不要嘗試在一個只包含三個項的 vector 中請求索引是 100 的元素。當將來你的代碼出現了 panic，你需要搞清楚在這特定的場景下代碼中執行了什麼操作和什麼值導致了 panic，以及應當如何處理才能避免這個問題。

本章的後面會再次回到 `panic!` 並講到何時應該何時不應該使用這個方式。接下來，我們來看看如何使用 `Result` 來從錯誤中恢復。