## 引用循環和內存洩漏是安全的

> [ch15-06-reference-cycles.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch15-06-reference-cycles.md)
> <br>
> commit 9430a3d28a2121a938d704ce48b15d21062f880e
<!-- 注意:本節仍在大改中，一切以英文版為準。 -->
我們討論過 Rust 做出的一些保證，例如永遠也不會遇到一個空值，而且數據競爭也會在編譯時被阻止。Rust 的內存安全保證也使其更難以製造從不被清理的內存，這被稱為**內存洩露**。然而 Rust 並不是**不可能**出現內存洩漏，避免內存洩露**並**不是 Rust 的保證之一。換句話說，內存洩露是安全的。

在使用`Rc<T>`和`RefCell<T>`時，有可能創建循環引用，這時各個項相互引用並形成環。這是不好的因為每一項的引用計數將永遠也到不了 0，其值也永遠也不會被丟棄。

### Creating a Reference Cycle

讓我們看一下引用循環是如何發生的，以及如何避免。
先從示例 15-20 中的`List` enum 和 `tail` 方法的定義開始。
<span class="filename">Filename: src/main.rs</span>

```rust
use std::rc::Rc;
use std::cell::RefCell;
use List::{Cons, Nil};

#[derive(Debug)]
enum List {
    Cons(i32, RefCell<Rc<List>>),
    Nil,
}

impl List {
    fn tail(&self) -> Option<&RefCell<Rc<List>>> {
        match *self {
            Cons(_, ref item) => Some(item),
            Nil => None,
        }
    }
}
```

<span class="caption">Listing 15-20: A cons list definition that holds a
`RefCell` so that we can modify what a `Cons` variant is referring to</span>

我們在此使用示例 15-6中 `List`定義的變化形，在`Cons` 變數的第二個元素現在是`RefCell<Rc<List>>` 

在示例 15-16 中，我們將使用示例 15-5 中`List`定義的另一個變體。我們將回到儲存`i32`值作為`Cons`成員的第一個元素。現在`Cons`成員的第二個元素是`RefCell<Rc<List>>`：這時就不能修改`i32`值了，但是能夠修改`Cons`成員指向的那個`List`。還需要增加一個`tail`方法來方便我們在擁有一個`Cons`成員時訪問第二個項：

<span class="filename">Filename: src/main.rs</span>

```rust
#[derive(Debug)]
enum List {
    Cons(i32, RefCell<Rc<List>>),
    Nil,
}

impl List {
    fn tail(&self) -> Option<&RefCell<Rc<List>>> {
        match *self {
            Cons(_, ref item) => Some(item),
            Nil => None,
        }
    }
}
```

<span class="caption">Listing 15-16: A cons list definition that holds a
`RefCell` so that we can modify what a `Cons` variant is referring to</span>

接下來，在示例 15-17 中，我們將在變數`a`中創建一個`List`值，其內部是一個`5, Nil`的示例。接著在變數`b`創建一個值 10 和指向`a`中示例的`List`值。最後修改`a`指向`b`而不是`Nil`，這會創建一個循環：

<span class="filename">Filename: src/main.rs</span>

```rust
# #[derive(Debug)]
# enum List {
#     Cons(i32, RefCell<Rc<List>>),
#     Nil,
# }
#
# impl List {
#     fn tail(&self) -> Option<&RefCell<Rc<List>>> {
#         match *self {
#             Cons(_, ref item) => Some(item),
#             Nil => None,
#         }
#     }
# }
#
use List::{Cons, Nil};
use std::rc::Rc;
use std::cell::RefCell;

fn main() {

    let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));

    println!("a initial rc count = {}", Rc::strong_count(&a));
    println!("a next item = {:?}", a.tail());

    let b = Rc::new(Cons(10, RefCell::new(a.clone())));

    println!("a rc count after b creation = {}", Rc::strong_count(&a));
    println!("b initial rc count = {}", Rc::strong_count(&b));
    println!("b next item = {:?}", b.tail());

    if let Some(ref link) = a.tail() {
        *link.borrow_mut() = b.clone();
    }

    println!("b rc count after changing a = {}", Rc::strong_count(&b));
    println!("a rc count after changing a = {}", Rc::strong_count(&a));

    // Uncomment the next line to see that we have a cycle; it will
    // overflow the stack
    // println!("a next item = {:?}", a.tail());
}
```

<span class="caption">Listing 15-17: Creating a reference cycle of two `List`
values pointing to each other</span>

使用`tail`方法來抓取`a`中`RefCell`的引用，並將其放入變數`link`中。接著對`RefCell`使用`borrow_mut`方法將其中的值從存放`Nil`值的`Rc`改為`b`中的`Rc`。這創建了一個看起來像圖 15-18 所示的引用循環：

<img alt="Reference cycle of lists" src="img/trpl15-04.svg" class="center" style="width: 50%;" />

<span class="caption">Figure 15-18: A reference cycle of lists `a` and `b`
pointing to each other</span>

如果你註釋掉最後的`println!`，Rust 會嘗試打印出`a`指向`b`指向`a`這樣的循環直到棧溢出。

觀察最後一個`println!`之前的打印結果，就會發現在將`a`改變為指向`b`之後`a`和`b`的引用計數都是 2。在`main`的結尾，Rust 首先會嘗試丟棄`b`，這會使`Rc`的引用計數減一，但是這個計數是 1 而不是 0，所以`Rc`在堆上的內存不會被丟棄。它只是會永遠的停留在 1 上。這個特定例子中，程序立馬就結束了，所以並不是一個問題，不過如果是一個更加複雜的程序，它在這個循環中分配了很多內存並佔有很長時間，這就是個問題了。這個程序會使用多於它所需要的內存，並有可能壓垮系統並造成沒有內存可供使用。

現在，如你所見，在 Rust 中創建引用循環是困難和繁瑣的。但並不是不可能：避免引用循環這種形式的內存洩漏並不是 Rust 的保證之一。如果你有包含`Rc<T>`的`RefCell<T>`值或類似的嵌套結合了內部可變性和引用計數的類型，請務必小心確保你沒有形成一個引用循環。在示例 15-14 的例子中，可能解決方式就是不要編寫像這樣可能造成引用循環的代碼，因為我們希望`Cons`成員擁有他們指向的示例。

舉例來說，對於像圖這樣的數據結構，為了創建父節點指向子節點的邊和以相反方向從子節點指向父節點的邊，有時需要創建這樣的引用循環。如果一個方向擁有所有權而另一個方向沒有，對於模擬這種數據關係的一種不會創建引用循環和內存洩露的方式是使用`Weak<T>`。接下來讓我們探索一下！

### 避免引用循環：將`Rc<T>`變為`Weak<T>`

Rust 標準庫中提供了`Weak<T>`，一個用於存在引用循環但只有一個方向有所有權的智能指針。我們已經展示過如何克隆`Rc<T>`來增加引用的`strong_count`；`Weak<T>`是一種引用`Rc<T>`但不增加`strong_count`的方式：相反它增加`Rc`引用的`weak_count`。當`Rc`離開作用域，其內部值會在`strong_count`為 0 的時候被丟棄，即便`weak_count`不為 0 。為了能夠從`Weak<T>`中抓取值，首先需要使用`upgrade`方法將其升級為`Option<Rc<T>>`。升級`Weak<T>`的結果在`Rc`還未被丟棄時是`Some`，而在`Rc`被丟棄時是`None`。因為`upgrade`返回一個`Option`，我們知道 Rust 會確保`Some`和`None`的情況都被處理並不會嘗試使用一個無效的指針。

不同於示例 15-17 中每個項只知道它的下一項，假如我們需要一個樹，它的項知道它的子項**和**父項。

讓我們從一個叫做`Node`的存放擁有所有權的`i32`值和其子`Node`值的引用的結構體開始：

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    children: RefCell<Vec<Rc<Node>>>,
}
```

我們希望能夠`Node`擁有其子節點，同時也希望變數可以擁有每個節點以便可以直接訪問他們。這就是為什麼`Vec`中的項是`Rc<Node>`值。我們也希望能夠修改其他節點的子節點，這就是為什麼`children`中`Vec`被放進了`RefCell`的原因。在示例 15-19 中創建了一個叫做`leaf`的帶有值 3 並沒有子節點的`Node`實例，和另一個帶有值 5 和以`leaf`作為子節點的實例`branch`：


<span class="filename">Filename: src/main.rs</span>

```rust
fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        children: RefCell::new(vec![]),
    });

    let branch = Rc::new(Node {
        value: 5,
        children: RefCell::new(vec![leaf.clone()]),
    });
}
```

<span class="caption">Listing 15-19: Creating a `leaf` node and a `branch` node
where `branch` has `leaf` as one of its children but `leaf` has no reference to
`branch`</span>

`leaf`中的`Node`現在有兩個所有者：`leaf`和`branch`，因為我們克隆了`leaf`中的`Rc`並儲存在了`branch`中。`branch`中的`Node`知道它與`leaf`相關聯因為`branch`在`branch.children`中有`leaf`的引用。然而，`leaf`並不知道它與`branch`相關聯，而我們希望`leaf`知道`branch`是其父節點。

為了做到這一點，需要在`Node`結構體定義中增加一個`parent`字段，不過`parent`的類型應該是什麼呢？我們知道它不能包含`Rc<T>`，因為這樣`leaf.parent`將會指向`branch`而`branch.children`會包含`leaf`的指針，這會形成引用循環。`leaf`和`branch`不會被丟棄因為他們總是引用對方且引用計數永遠也不會是零。

所以在`parent`的類型中是使用`Weak<T>`而不是`Rc`，具體來說是`RefCell<Weak<Node>>`：

<span class="filename">Filename: src/main.rs</span>

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}
```

這樣，一個節點就能夠在擁有父節點時指向它，而並不擁有其父節點。一個父節點哪怕在擁有指向它的子節點也會被丟棄，只要是其自身也沒有一個父節點就行。現在將`main`函數更新為如示例 15-20 所示：

<span class="filename">Filename: src/main.rs</span>

```rust
fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());

    let branch = Rc::new(Node {
        value: 5,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![leaf.clone()]),
    });

    *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());
}
```

<span class="caption">Listing 15-20: A `leaf` node and a `branch` node where
`leaf` has a `Weak` reference to its parent, `branch`</span>

創建`leaf`節點是類似的；因為它作為開始並沒有父節點，這裡創建了一個新的`Weak`引用實例。當嘗試通過`upgrade`方法抓取`leaf`父節點的引用時，會得到一個`None`值，如第一個`println!`輸出所示：

```
leaf parent = None
```

類似的，`branch`也有一個新的`Weak`引用，因為也沒有父節點。`leaf`仍然作為`branch`的一個子節點。一旦在`branch`中有了一個新的`Node`實例，就可以修改`leaf`將一個`branch`的`Weak`引用作為其父節點。這裡使用了`leaf`中`parent`字段裡的`RefCell`的`borrow_mut`方法，接著使用了`Rc::downgrade`函數來從`branch`中的`Rc`值創建了一個指向`branch`的`Weak`引用。

當再次打印出`leaf`的父節點時，這一次將會得到存放了`branch`的`Some`值。另外需要注意到這裡並沒有打印出類似示例 15-14 中那樣最終導致棧溢出的循環：`Weak`引用僅僅打印出`(Weak)`：

```
leaf parent = Some(Node { value: 5, parent: RefCell { value: (Weak) },
children: RefCell { value: [Node { value: 3, parent: RefCell { value: (Weak) },
children: RefCell { value: [] } }] } })
```

沒有無限的輸出（或直到棧溢出）的事實表明這裡並沒有引用循環。另一種證明的方式時觀察調用`Rc::strong_count`和`Rc::weak_count`的值。在示例 15-21 中，創建了一個新的內部作用域並將`branch`的創建放入其中，這樣可以觀察`branch`被創建時和離開作用域被丟棄時發生了什麼：

<span class="filename">Filename: src/main.rs</span>

```rust
fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf),
    );

    {
        let branch = Rc::new(Node {
            value: 5,
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(vec![leaf.clone()]),
        });
        *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

        println!(
            "branch strong = {}, weak = {}",
            Rc::strong_count(&branch),
            Rc::weak_count(&branch),
        );

        println!(
            "leaf strong = {}, weak = {}",
            Rc::strong_count(&leaf),
            Rc::weak_count(&leaf),
        );
    }

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());
    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf),
    );
}
```

<span class="caption">Listing 15-21: Creating `branch` in an inner scope and
examining strong and weak reference counts of `leaf` and `branch`</span>

創建`leaf`之後，強引用計數是 1 （用於`leaf`自身）而弱引用計數是 0。在內部作用域中，在創建`branch`和關聯`leaf`和`branch`之後，`branch`的強引用計數為 1（用於`branch`自身）而弱引用計數為 1（因為`leaf.parent`通過一個`Weak<T>`指向`branch`）。`leaf`的強引用計數為 2，因為`branch`現在有一個`leaf`克隆的`Rc`儲存在`branch.children`中。`leaf`的弱引用計數仍然為 0。

當內部作用域結束，`branch`離開作用域，其強引用計數減少為 0，所以其`Node`被丟棄。來自`leaf.parent`的弱引用計數 1 與`Node`是否被丟棄無關，所以並沒有產生內存洩露！

如果在內部作用域結束後嘗試訪問`leaf`的父節點，會像`leaf`擁有父節點之前一樣得到`None`值。在程序的末尾，`leaf`的強引用計數為 1 而弱引用計數為 0，因為現在`leaf`又是唯一指向其自己的值了。

所有這些管理計數和值是否應該被丟棄的邏輯都通過`Rc`和`Weak`和他們的`Drop` trait 實現來控制。通過在定義中指定從子節點到父節點的關係為一個`Weak<T>`引用，就能夠擁有父節點和子節點之間的雙向引用而不會造成引用循環和內存洩露。

## 總結

現在我們學習了如何選擇不同類型的智能指針來選擇不同的保證並與 Rust 的常規引用向取捨。`Box<T>`有一個已知的大小並指向分配在堆上的數據。`Rc<T>`記錄了堆上數據的引用數量這樣就可以擁有多個所有者。`RefCell<T>`和其內部可變性使其可以用於需要不可變類型，但希望在運行時而不是編譯時檢查借用規則的場景。

我們還介紹了提供了很多智能指針功能的 trait `Deref`和`Drop`。同時探索了形成引用循環和造成內存洩漏的可能性，以及如何使用`Weak<T>`避免引用循環。

如果本章內容引起了你的興趣並希望現在就實現你自己的智能指針的話，請閱讀 [The Nomicon] 來抓取更多有用的信息。

[The Nomicon]: https://doc.rust-lang.org/stable/nomicon/

接下來，讓我們談談 Rust 的並發。我們還會學習到一些新的對並發有幫助的智能指針。