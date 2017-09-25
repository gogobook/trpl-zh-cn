## `Drop` Trait 運行清理代碼

> [ch15-03-drop.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch15-03-drop.md)
> <br>
> commit 3f2a1bd8dbb19cc48b210fc4fb35c305c8d81b56

對於智能指針模式來說另一個重要的 trait 是`Drop`。`Drop`運行我們在值要離開作用域時執行一些代碼。智能指針在被丟棄時會執行一些重要的清理工作，比如釋放內存或減少引用計數。更一般的來講，數據類型可以管理多於內存的資源，比如文件或網絡連接，而使用`Drop`在代碼處理完他們之後釋放這些資源。我們在智能指針上下文中討論`Drop`是因為其功能幾乎總是用於實現智能指針。

在其他一些語言中，我們不得不記住在每次使用完智能指針實例後調用清理內存或資源的代碼。如果忘記的話，運行代碼的系統可能會因為負荷過重而崩潰。在 Rust 中，可以指定一些代碼應該在值離開作用域時被執行，而編譯器會自動插入這些代碼。這意味著無需記住在所有處理完這些類型實例後調用清理代碼，而仍然不會洩露資源！

指定在值離開作用域時應該執行的代碼的方式是實現`Drop` trait。`Drop` trait 要求我們實現一個叫做`drop`的方法，它抓取一個`self`的可變引用。

列表 15-8 展示了並沒有實際功能的結構體`CustomSmartPointer`，不過我們會在創建實例之後打印出`CustomSmartPointer created.`，而在實例離開作用域時打印出`Dropping CustomSmartPointer!`，這樣就能看出每一段代碼是何時被執行的。實際的項目中，我們應該在`drop`中清理任何智能指針運行所需要的資源，而不是這個例子中的`println!`語句：

<span class="filename">Filename: src/main.rs</span>

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer!");
    }
}

fn main() {
    let c = CustomSmartPointer { data: String::from("some data") };
    println!("CustomSmartPointer created.");
    println!("Wait for it...");
}
```

<span class="caption">Listing 15-8: A `CustomSmartPointer` struct that
implements the `Drop` trait, where we could put code that would clean up after
the `CustomSmartPointer`.</span>

`Drop` trait 位於 prelude 中，所以無需導入它。`drop`方法的實現調用了`println!`；這裡是你需要放入實際關閉套接字代碼的地方。在`main`函數中，我們創建一個`CustomSmartPointer`的新實例並打印出`CustomSmartPointer created.`以便在運行時知道代碼運行到此處。在`main`的結尾，`CustomSmartPointer`的實例會離開作用域。注意我們沒有顯式調用`drop`方法：

當運行這個程序，我們會看到：

```
CustomSmartPointer created.
Wait for it...
Dropping CustomSmartPointer!
```

被打印到屏幕上，它展示了 Rust 在實例離開作用域時自動調用了`drop`。

可以使用`std::mem::drop`函數來在值離開作用域之前丟棄它。這通常是不必要的；整個`Drop` trait 的要點在於它自動的幫我們處理清理工作。在第十六章講到並發時我們會看到一個需要在離開作用域之前丟棄值的例子。現在知道這是可能的即可，`std::mem::drop`位於 prelude 中所以可以如列表 15-9 所示直接調用`drop`：

<span class="filename">Filename: src/main.rs</span>

```rust
fn main() {
    let c = CustomSmartPointer { data: String::from("some data") };
    println!("CustomSmartPointer created.");
    drop(c);
    println!("Wait for it...");
}
```

<span class="caption">Listing 15-9: Calling `std::mem::drop` to explicitly drop
a value before it goes out of scope</span>

運行這段代碼會打印出如下內容，因為`Dropping CustomSmartPointer!`在`CustomSmartPointer created.`和`Wait for it...`之間被打印出來，表明析構代碼被執行了：

```
CustomSmartPointer created.
Dropping CustomSmartPointer!
Wait for it...
```

注意不允許直接調用我們定義的`drop`方法：如果將列表 15-9 中的`drop(c)`替換為`c.drop()`，會得到一個編譯錯誤表明`explicit destructor calls not allowed`。不允許直接調用`Drop::drop`的原因是 Rust 在值離開作用域時會自動插入`Drop::drop`，這樣就會丟棄值兩次。丟棄一個值兩次可能會造成錯誤或破壞內存，所以 Rust 就不允許這麼做。相應的可以調用`std::mem::drop`，它的定義是：

```rust
pub mod std {
    pub mod mem {
        pub fn drop<T>(x: T) { }
    }
}
```

這個函數對於`T`是泛型的，所以可以傳遞任何值。這個函數的函數體並沒有任何實際內容，所以它也不會利用其參數。這個空函數的作用在於`drop`抓取其參數的所有權，它意味著在這個函數結尾`x`離開作用域時`x`會被丟棄。

使用`Drop` trait 實現指定的代碼在很多方面都使得清理值變得方便和安全：比如可以使用它來創建我們自己的內存分配器！通過`Drop` trait 和 Rust 所有權系統，就無需擔心之後清理代碼，因為 Rust 會自動考慮這些問題。如果代碼在值仍被使用時就清理它會出現編譯錯誤，因為所有權系統確保了引用總是有效的，這也就保證了`drop`只會在值不再被使用時被調用一次。

現在我們學習了`Box<T>`和一些智能指針的特性，讓我們聊聊一些其他標準庫中定義的擁有各種實用功能的智能指針。