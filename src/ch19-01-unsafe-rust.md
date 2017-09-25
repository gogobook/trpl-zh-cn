## 不安全的Rust

在本書之前的章節, 我們討論了Rust代碼在編譯時會強制保證內存安全. 然而, Rust還有另一個隱藏的語言特性, 這就是不安全的Rust, 它不會擔保內存安全. 不安全的Rust和常規Rust代碼無異, 但是它會給你安全的Rust代碼不具備的超能力.

不安全的Rust之所以存在, 本質上是因為編譯器對代碼的靜態分析趨於保守. 代碼何時保證內存安全, 何時放權這種擔保呢? 把合法的代碼拒絕掉通常比接納非法的代碼要好一點. 有些時候你的代碼的確沒問題, 但是Rust卻不這樣認為! 這時你可以用不安全的代碼告訴編譯器, "相信我吧, 我知道我在做什麼." 這樣缺陷可能就在於你自己了; 如果你的不安全代碼發生了錯誤, 比如對null指針解引用就可能會引發內存出錯的大問題.

還有另一個Rust需要不安全代碼的原因: 底層電腦硬件固有的不安全性. 如果Rust不讓你執行不安全的操作, 那麼有些任務你就完成不了. 但是Rust需要你能夠做像直接與操作系統交互甚至是寫你自己的操作系統這樣的底層操作! 這也是Rust語言的一部分目標, 所以我們需要一些來做這些事情的方法.

### 不安全的神力

我們通過使用`unsafe`關鍵字開啟一個持有不安全代碼的代碼塊來切換到不安全的Rust. 你可以在不安全的Rust中進行四個安全的Rust做不到的操作. 我們把它們稱作"不安全的神力". 之前我們沒見過這幾個特性是因為它們只用在`unsafe`代碼塊中! 它們是: 

1. 解引用原生指針
2. 調用一個不安全的函數或方法
3. 訪問或修改一個不可變的靜態變數
4. 實現一個不安全的trait

記住這一點很重要, `unsafe`不會關掉借用檢查器也不會禁用其它的Rust安全性檢查: 如果你在不安全的代碼中用了引用, 它仍將會被檢查. `unsafe`關鍵字做的唯一的一件事是讓你存取編譯器因內存安全性而沒去檢查的上述四個特性.在一個unsafe代碼塊中你仍然會獲得某種程度的安全性! 此外, `unsafe`並不是說代碼塊中的代碼是危險的或者有內存安全性問題: 它只是表明作為程式設計師的你關掉了編譯器檢查, 你將確保`unsafe`代碼塊會擁有合理的內存.

人是會犯錯誤的, 錯誤總會發生. 在`unsafe`代碼塊中執行上述四個不安全的操作時, 如果你犯了錯誤並得到一個內存安全性的錯誤, 你必定會知道它與你使用不安全的代碼有關. 這樣就更容易處理內存安全性的bug, 因為Rust已經幫我們把其它的代碼做了檢查. 能縮小排查內存安全性bug的出現區域當然好, 所以儘量縮小你的不安全代碼的數量吧. 當修正內存安全問題時, `unsafe`代碼塊中的任何代碼都可能出錯: 所以讓`unsafe`代碼塊儘可能的小吧, 以後你需要排查的代碼也會少一些.

為了儘可能隔離不安全的代碼, 在安全的抽象中包含不安全的代碼並提供一個安全的API是一個好主意, 當我們學習不安全的函數和方法時我們會討論它. 標準庫中有些不安全的代碼被實現為安全的抽象, 它們中的部分已被審核過了. 當你或者你的用戶使用通過`unsafe`代碼實現的功能時, 因為使用一個安全的抽象是安全的, 這樣就可以避免到處都是`unsafe`字樣.

讓我們按順序依次介紹上述四個不安全的神力, 同時我們會見到一些抽象, 它們為不安全的代碼提供了安全的接口.

### 解引用原生指針

回到第4章, 我們在哪裡學習了引用. 我們知道編譯器會確保引用永遠合法. 不安全的Rust有兩個類似於引用的被稱為*原生指針*(*raw pointers*)的新類型. 和引用一樣, 我們可以有一個不可變的原生指針和一個可變的原生指針. 在原生指針的上下文中, "不可變"意味著指針不能直接被解引用和被賦值. 例19-1演示了如何通過引用來創建一個原生指針:

```rust
let mut num = 5;

let r1 = &num as *const i32;
let r2 = &mut num as *mut i32;
```

<span class="caption">例19-1: 通過引用創建原生指針</span>

上例中`*const T`類型是一個不可變的原生指針, `*mut T`是一個可變的原生指針. 我們通過使用`as`把一個可變的和一個不可變的引用轉換成它們對應的原生指針類型來創建原生指針. 與引用不同, 這些指針的合法性不能得到保證.

例19-2演示了如何創建一個指向內存中任意地址的原生指針. 試圖隨便訪問內存地址所帶來的結果是難以預測的: 也許在那個地址處有數據, 也許在那個地址處沒有任何數據, 編譯器也可能會優化代碼導致那塊內存不能訪問, 亦或你的程序可能會發生段錯誤. 雖然可以寫出下面的代碼, 但是通常找不到好的理由來這樣做:

```rust
let address = 0x012345;
let r = address as *const i32;
```

<span class="caption">例子19-2: 創建一個指向任意內存地址的原生指針</span>

注意在例19-1和19-2中沒有`unsafe`代碼塊. 你可以在安全代碼中*創建*原生指針
raw pointers in safe code, 但是你不能在安全代碼中*解引用*(*dereference*)原生指針來讀取被指針指向的數據. 如例19-3所示, 對原生指針解引用需要在`unsafe`代碼塊中使用解引用操作符`*`:

```rust
let mut num = 5;

let r1 = &num as *const i32;
let r2 = &mut num as *mut i32;

unsafe {
    println!("r1 is: {}", *r1);
    println!("r2 is: {}", *r2);
}
```

<span class="caption">例19-3: 在`unsafe`代碼塊中解引用原生指針</span>

創建一個指針不會造成任何危險; 只有在你訪問指針指向的值時可能出問題, 因為你可能會用它處理無效的值.

注意在19-1和19-3中我們創建的一個`*const i32`和一個`*mut i32`都指向同一個內存位置, 也就是`num`. 如果我們嘗試創建一個不可變的和可變的`num`的引用而不是原生指針, 這就不能被編譯, 因為我們不能在使用了不可變引用的同時再對同一個值進行可變引用. 通過原生指針, 我們能創建指向同一個內存位置的可變指針和不可變指針, 我們可以通過可變指針來改變數據, 但是要小心, 因為這可能會產生數據競爭!

既然存在這麼多的危險, 為什麼我們還要使用原生指針呢? 一個主要的原因是為了與C代碼交互, 在下一節的不安全函數裡我們將會看到. 另一個原因是創建一個借用檢查器理解不了的安全的抽象. 下面讓我們介紹不安全的函數, 然後再看一個使用了不安全代碼的安全的抽象的例子.

### 調用一個不安全的函數或方法

需要一個不安全的代碼塊的才能執行的第二個操作是調用不安全的函數. 不安全的函數和方法與常規的函數和方法看上去沒有什麼異樣, 只是他們前面有一個額外的`unsafe`關鍵字. 不安全的函數的函數體自然是`unsafe`的代碼塊. 下例是一個名叫`dangerous`的不安全的函數:

```rust
unsafe fn dangerous() {}

unsafe {
    dangerous();
}
```

如果不用`unsafe`代碼塊來調用`dangerous`, 我們將會得到下面的錯誤:

```text
error[E0133]: call to unsafe function requires unsafe function or block
 --> <anon>:4:5
  |
4 |     dangerous();
  |     ^^^^^^^^^^^ call to unsafe function
```

通過把對`dangerous`的調用放到`unsafe`代碼塊中, 我們表明我們已經閱讀了該函數的文檔, 我們明白如何正確的使用它, 並且我們已經驗證了調用的正確性.

#### 創建一個不安全的代碼上的安全的抽象

讓我們用標準庫中的某個函數比如`split_at_mut`來舉個例子, 然後來探討我們如何自己來實現它. 這個方法被定義在一個可變的切片(slice)上, 它通過參數指定的索引把一個切片分割成兩個, 如例19-4所示:

```rust
let mut v = vec![1, 2, 3, 4, 5, 6];

let r = &mut v[..];

let (a, b) = r.split_at_mut(3);

assert_eq!(a, &mut [1, 2, 3]);
assert_eq!(b, &mut [4, 5, 6]);
```

<span class="caption">例19-4: 使用安全的`split_at_mut`函數</span>

用安全的Rust代碼是不能實現這個函數的. 如果要試一下用安全的Rust來實現它可以參考例19-5. 簡單起見, 我們把`split_at_mut`實現成一個函數而不是一個方法, 這個函數只處理`i32`類型的切片而不是泛型類型`T`的切片:

```rust
fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();

    assert!(mid <= len);

    (&mut slice[..mid],
     &mut slice[mid..])
}
```

<span class="caption">例19-5: 嘗試用安全的Rust來實現`split_at_mut`</span>

該函數先取得切片(slice)的長度, 然後通過檢查參數是否小於或等於這個長度來斷言參數給定的索引位於切片(slice)當中. 這個斷言意外著如果我們傳入的索引比要分割的切片(slice)的長度大, 這個函數就會在使用這個索引前中斷(panic).

接著我們在一個元組中返回兩個可變的切片(slice): 一個從被分割的切片的頭部開始直到`mid`索引的前一個元素中止, 另一個從被分割的切片的`mid`索引開始直到被分割的切片的末尾結束.

如果我們編譯上面的代碼, 我們將得到一個錯誤:

```text
error[E0499]: cannot borrow `*slice` as mutable more than once at a time
 --> <anon>:6:11
  |
5 |     (&mut slice[..mid],
  |           ----- first mutable borrow occurs here
6 |      &mut slice[mid..])
  |           ^^^^^ second mutable borrow occurs here
7 | }
  | - first borrow ends here
```

Rust的借用檢查器不能理解為什麼我們要借用這個切片(slice)的不同部分; 它只知道我們對同一個切片借用了兩次. 借用一個切片(slice)的不同部分在功能上是沒問題的; 而且我們的兩個`&mut [i32]`也沒有重疊. 但是Rust並沒有聰明到能明白這一點. 當我們知道有些東西是可以的但是Rust卻不知道的時候就是時候使用不安全的代碼了.

例子19-6演示了如何用一個`unsafe`代碼塊、 一個原生指針和一個不安全的函數調用來實現`split_at_mut`:

```rust
use std::slice;

fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();
    let ptr = slice.as_mut_ptr();

    assert!(mid <= len);

    unsafe {
        (slice::from_raw_parts_mut(ptr, mid),
         slice::from_raw_parts_mut(ptr.offset(mid as isize), len - mid))
    }
}
```

<span class="caption">例19-6: 用不安全的代碼來實現`split_at_mut`</span>

回顧一下第4章, 切片(slice)是一個指向某個數據的指針和這個切片(slice)的長度. 我們經常用`len`方法來取得切片的長度; 也可以用`as_mut_ptr`方法來訪問切片的原生指針. 在這個例子裡, 因為我們有一個可變的`i32`類型的切片, `as_mut_ptr`返回一個`*mut i32`類型的原生指針, 我們把它存放在變數`ptr`裡.

對索引`mid`合法性的斷言上面已經介紹過了. 函數`slice::from_raw_parts_mut`的行為與`as_mut_ptr`和`len`方法相反: 它以一個原生指針和一個長度為參數並返回一個切片(slice). 我們調用`slice::from_raw_parts_mut`來創建一個從`ptr`開始且擁有`mid`個元素的切片. 然後我們以`mid`為參數調用`prt`上的`offset`方法來得到一個從索引`mid`開始的原生指針, 然後我們用這個原生指針和索引`mid`之後的元素個數為參數創建一個切片.

因為切片(slice)會被檢查, 所以一旦我們創建了它就可以安全使用. 函數`slice::from_raw_parts_mut`是一個不安全的函數因為它有一個原生指針參數, 而且它相信這個指針是有效的. 原生指針的`offset`方法也是不安全的, 因為它相信一個原生指針的位置偏移一些後也是一個有效的指針. 為了能調用`slice::from_raw_parts_mut`和`offset`, 我們把他們的調用放到一個`unsafe`代碼塊中, 我們可以通過查看代碼並添加`mid`不大於`len`的斷言來表明`unsafe`代碼塊中的原生指針是指向切片中的數據的有效指針. 這是一個`unsafe`恰當用法.

注意結果`split_at_mut`函數是安全的: 我們不用在它的前面添加`unsafe`關鍵字, 並且我們可以從安全的Rust代碼中調用它. 我們通過寫一個使用了`unsafe`代碼的函數來創建不安全代碼的安全抽象, 上例用一種安全的方式通過函數訪問的數據來創建有效的指針.

相反, 當使用切片(slice)時, 例19-7中`slice::from_raw_parts_mut`的用法很可能會崩潰. 下面的代碼用一個隨意的內存地址來創建一個有10000個元素的切片:

```rust
use std::slice;

let address = 0x012345;
let r = address as *mut i32;

let slice = unsafe {
    slice::from_raw_parts_mut(r, 10000)
};
```

<span class="caption">例19-7: 通過一個任意的內存位置來創建一個切片</span>

我們不能擁有任意地址的內存, 也不能保證這個代碼創建的切片會包含有效的`i32`類型的值. 試圖使用臆測是有效切片的`slice`的行為是難以預測的.

#### 調用外部代碼的`extern`函數是不安全的

有時, 你的Rust代碼需要與其它語言交互. Rust有一個`extern`關鍵字可以實現這個功能, 這有助於創建並使用*外部功能接口(Foreign Function Interface)* (FFI). 例19-8演示了如何與定義在一個非Rust語言編寫的外部庫中的`some_function`進行交互. 在Rust中調用`extern`聲明的代碼塊永遠都是不安全的:

<span class="filename">Filename: src/main.rs</span>

```rust
extern "C" {
    fn some_function();
}

fn main() {
    unsafe { some_function() };
}
```

<span class="caption">例19-8: 聲明並調用一個用其它語言寫成的函數</span>

在`extern "C"`代碼塊中, 我們列出了我們想調用的用其它語言實現的庫中定義的函數名和這個函數的特徵簽名.`"C"`定義了外部函數使用了哪種*應用程序接口(application binary interface)* (ABI). ABI定義了如何在彙編層調研這個函數. `"C"`是最常用的遵循C語言的ABI.

調用一個外部函數總是不安全的. 如果我們要調用其他語言, 這種語言卻不會遵循Rust的安全保證. 因為Rust不能檢查外部代碼是否是安全的, 我們只負責檢查外部代碼的安全性來表明我們已經用`unsafe`代碼塊來調用外部函數了.

<!-- PROD: START BOX -->

##### 通過其它語言來調用Rust函數

`extern`關鍵字也總是被用來創建一個允許被其他語言調用的Rust函數. 與`extern`代碼塊不同, 我們可以在`fn`關鍵字之前添加`extern`關鍵字並指定要使用的ABI. 我們也加入註解`#[no_mangle]`來告訴Rust編譯器不要取笑這個函數的名字. 一旦我們把下例的代碼編譯成一個共享庫並鏈接到C, 這個`call_from_c`函數就可以被C代碼訪問了:

```rust
#[no_mangle]
pub extern "C" fn call_from_c() {
    println!("Just called a Rust function from C!");
}
```

上例的`extern`不需要`unsafe`就可以直接用

<!-- PROD: END BOX -->

### 訪問或修改一個可變的靜態變數

目前為止本書還沒有*討論全局變數(global variables)*. 很多語言都支持全局變數, 當然Rust也不例外. 然而全局變數也有問題: 比如, 如果兩個線程訪問同一個可變的全局變數有可能會發生數據競爭.

全局變數在Rust中被稱為是*靜態(static)*變數. 例19-9中聲明並使用了一個字符串切片類型的靜態變數:

<span class="filename">Filename: src/main.rs</span>

```rust
static HELLO_WORLD: &str = "Hello, world!";

fn main() {
    println!("name is: {}", HELLO_WORLD);
}
```

<span class="caption">例19-9: 定義和使用一個不可變的靜態變數</span>

`static`變數類似於常量: 按照慣例它們的命名遵從`SCREAMING_SNAKE_CASE(用下劃線分割的全大寫字母)`風格, 我們也*必須*註明變數的類型, 本例中是`&'static str`. 只有定義為`'static`的生命期才可以被存儲在一個靜態變數中. 也正因為此, Rust編譯器自己就已經很清楚靜態變數的生命期了, 所以我們也不需要明確地註明它了. 訪問不可變的靜態變數是安全的. 因為靜態變數的值有一個固定的內存地址, 所以使用該值的時候總會得到同樣的數據. 另一方面, 當常量被使用時, 複製它們的數據也是被允許的.

靜態變數與常量的另一個不同是靜態變數可以是可變的. 訪問和修改可變的靜態變數都是不安全的. 例19-10演示了如何聲明、訪問和修改一個名叫`COUNTER`的可變的靜態變數:

<span class="filename">Filename: src/main.rs</span>

```rust
static mut COUNTER: u32 = 0;

fn add_to_count(inc: u32) {
    unsafe {
        COUNTER += inc;
    }
}

fn main() {
    add_to_count(3);

    unsafe {
        println!("COUNTER: {}", COUNTER);
    }
}
```

<span class="caption">例19-10: 讀取或修改一個可變的靜態變數是不安全的</span>

與常規變數一樣, 我們用`mut`關鍵字來表明這個靜態變數是可變的. 每次我們對`COUNTER`的讀寫都必須被放到一個`unsafe`代碼塊中. 上面的代碼編譯運行會打印`COUNTER: 3`, 這正如我們期望的那樣, 因為程序現在是一個單線程, 如果有多個線程訪問`COUNTER`就可能會導致數據競爭.

可全局訪問的可變數據難於管理也很難保證沒有數據競爭, 這也正是Rust認為可變的靜態變數是不安全的原因. 如果可能, 請使用在第16章中介紹的並發技術和線程安全的智能指針, 這樣可以讓編譯器從不同的線程檢查被訪問的數據是安全的.

### 實現一個不安全的Trait

最後, 當我們使用`unsafe`關鍵字時最後一個只在不安全的代碼中才能做的事是實現一個不安全的trait. 我們可以在`trait`之前添加一個`unsafe`關鍵字來聲明一個trait是不安全的, 以後實現這個trait的時候也必須標記一個`unsafe`關鍵字, 如19-11所示:

```rust
unsafe trait Foo {
    // methods go here
}

unsafe impl Foo for i32 {
    // method implementations go here
}
```

<span class="caption">例19-11: 定義並實現一個不安全的trait</span>

與不安全的函數類似, 一個不安全的trait中的方法也有一些編譯器無法驗證的盲點. 通過使用`unsafe impl`, 我們就是在說明我們來保證這些有疑慮的地方的安全.

舉個例子, 回想一下第16章中的`Sync`和`Send`這兩個標記trait, 如果我們的類型全部由`Send`和`Sync`類型組合而成, 編譯器會自動實現它們. 如果我們要實現的一個類型包含了不是`Send`或`Sync`的東西, 比如原生指針, 若是我們像把我們的類型標記成`Send`或`Sync`, 這就要求使用`unsafe`關鍵字. Rust不能驗證我們的類型能保證可以安全地跨線程發送或從多個線程訪問, 所以我們需要用`unsafe`關鍵字來表明我們會自己來做這些檢查.

使用`unsafe`來執行這四個動作之一是沒有問題的, 因為編譯器不能確保內存安全, 所以把
`unsafe`代碼寫正確也實屬不易. 當你需要使用`unsafe`代碼時, 你可以這樣做, 明確註明`unsafe`, 這樣如果出現問題可以更容易地追蹤問題的源頭.