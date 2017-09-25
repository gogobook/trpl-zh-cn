## 所有的模式語法

通過本書我們已領略過一些不同類型模式的例子. 本節會列出所有在模式中有效的語法並且會闡述你為什麼可能會用到它們中的每一個.

### 字面量

我們在第6章已經見過, 你可以直接匹配字面量:

```rust
let x = 1;

match x {
    1 => println!("one"),
    2 => println!("two"),
    3 => println!("three"),
    _ => println!("anything"),
}
```

這段代碼會打印`one`因為`x`的值是1.

### 命名變數

命名變數是可匹配任何值的`irrefutable`(不可反駁)模式.

與所有變數一樣, 模式中聲明的變數會屏蔽`match`表達式外層的同名變數, 因為一個`match`表達式會開啟一個新的作用域. 在列表18-10中, 我們聲明了一個值為`Some(5)`的變數`x`和一個值為`10`的變數`y`. 然後是一個值`x`上的`match`表達式. 看一看匹配分支的模式和結尾的`println!`, 你可以在繼續閱讀或運行代碼前猜一猜什麼會被打印出來:

<span class="filename">Filename: src/main.rs</span>

```rust
fn main() {
    let x = Some(5);
    let y = 10;

    match x {
        Some(50) => println!("Got 50"),
        Some(y) => println!("Matched, y = {:?}", y),
        _ => println!("Default case, x = {:?}", x),
    }

    println!("at the end: x = {:?}, y = {:?}", x, y);
}
```

<span class="caption">列表18-10: 引入了一個陰影變數`y`的`match`語句</span>

<!-- NEXT PARAGRAPH WRAPPED WEIRD INTENTIONALLY SEE #199 -->

讓我們看看當`match`語句運行的時候發生了什麼. 第一個匹配分支是模式`Some(50)`, `x`中的值(`Some(5)`)不匹配`Some(50)`, 所以我們繼續. 在第二個匹配分支中, 模式`Some(y)`引入了一個可以匹配在`Some`裡的任意值的新變數`y`. 因為我們位於`match`表達式裡面的新作用域中, 所以`y`就是一個新變數而不是在開頭被聲明的其值為10的變數`y`. 這個新的`y`綁定將會匹配在`Some`中的任意值, 這裡也就是`x`中的值, 因為`y`綁定到`Some`中的值是`x`, 這裡是5, 所以我們就執行了這個分支中的表達式並打印出`Matched, y = 5`.

如果`x`的值是`None`而不是`Some(5)`, 我們將會匹配下劃線因為其它兩個分支的模式將不會被匹配. 在這個匹配分支(下劃線)的表達式裡, 因為我們沒有在分支的模式中引入變數`x`, 所以這個`x`仍然是`match`作用域外部的那個沒被屏蔽的`x`. 在這個假想的例子中, `match`表達式將會打印出`Default case, x =
None`.

一旦`match`表達式執行完畢, 它的作用域也就結束了, 同時`match`內部的`y`也就結束了. 最後的`println!`會打印`at the end: x = Some(5), y = 10`.

為了讓`match`表達式能比較外部變數`x`和`y`的值而不是內部引入的陰影變數`x`和`y`, 我們需要使用一個有條件的匹配守衛(guard). 我們將在本節的後面討論匹配守衛.

### 多種模式

只有在`match`表達式中, 你可以通過`|`符號匹配多個模式, 它代表*或*(*or*)的意思:

```rust
let x = 1;

match x {
    1 | 2 => println!("one or two"),
    3 => println!("three"),
    _ => println!("anything"),
}
```

上面的代碼會打印`one or two`.

### 通過`...`匹配值的範圍

你可以用`...`匹配一個值包含的範圍:

```rust
let x = 5;

match x {
    1 ... 5 => println!("one through five"),
    _ => println!("something else"),
}
```

上面的代碼中, 如果`x`是1、 2、 3、 4或5, 第一個分支就會匹配.

範圍只能是數字或`char`類型的值. 下面是一個使用`char`類型值範圍的例子:

```rust
let x = 'c';

match x {
    'a' ... 'j' => println!("early ASCII letter"),
    'k' ... 'z' => println!("late ASCII letter"),
    _ => println!("something else"),
}
```

上面的代碼會打印`early ASCII letter`.

### 解構並提取值

模式可以用來*解構*(*destructure*)結構、枚舉、元組和引用. 解構意味著把一個值分解成它的組成部分. 例18-11中的結構`Point`有兩個字段`x`和`y`, 我們可以通過一個模式和`let`語句來進行提取:

<span class="filename">Filename: src/main.rs</span>

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };

    let Point { x, y } = p;
    assert_eq!(0, x);
    assert_eq!(7, y);
}
```

<span class="caption">例18-11: 用結構的字段來解構</span>

上面的代碼創建了匹配`p`中的`x`和`y`字段的變數`x`和`y`. 變數的名字必須匹配使用了這個寫法中的字段. 如果我們想使用不同的變數名字, 我們可以在模式中使用`field_name: variable_name`. 在例18-12中, `a`會擁有`Point`實例的`x`字段的值, `b`會擁有`y`字段的值:

<span class="filename">Filename: src/main.rs</span>

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };

    let Point { x: a, y: b } = p;
    assert_eq!(0, a);
    assert_eq!(7, b);
}
```

<span class="caption">例18-12: 把結構解構到與字段不同名的變數中</span>

為了測試和使用一個值內部的某個屬性, 我們也可以用字面量來解構. 例18-13用一個`match`語句來判斷一個點是位於`x`(此時`y` = 0)軸上還是在`y`(此時`x` = 0)軸上或者不在兩個軸上面:

```rust
# struct Point {
#     x: i32,
#     y: i32,
# }
#
fn main() {
    let p = Point { x: 0, y: 7 };

    match p {
        Point { x, y: 0 } => println!("On the x axis at {}", x),
        Point { x: 0, y } => println!("On the y axis at {}", y),
        Point { x, y } => println!("On neither axis: ({}, {})", x, y),
    }
}
```

<span class="caption">例18-13: 解構和匹配一個模式中的字面量</span>

上面的代碼會打印`On the y axis at 7`, 因為`p`的`x`字段的值是0, 這正好匹配第二個分支.

在第6章中我們對枚舉進行瞭解構, 比如例6-5中, 我們用一個`match`表達式來解構一個`Option<i32>`, 其中被提取出來的一個值是`Some`內的變數.

當我們正匹配的值在一個包含了引用的模式裡面時, 為了把引用和值分割開我們可以在模式中指定一個`&`符號. 在迭代器對值的引用進行迭代時當我們想在閉包中使用值而不是引用的時侯這個符號在閉包裡特別有用. 例18-14演示了如何在一個向量裡迭代`Point`實例的引用, 為了能方便地對`x`和`y`的值進行計算還對引用的結構進行瞭解構:

```rust
# struct Point {
#     x: i32,
#     y: i32,
# }
#
let points = vec![
    Point { x: 0, y: 0 },
    Point { x: 1, y: 5 },
    Point { x: 10, y: -3 },
];
let sum_of_squares: i32 = points
    .iter()
    .map(|&Point {x, y}| x * x + y * y)
    .sum();
```

<span class="caption">例18-14: 把結構的引用解構到結構的字段值中</span>

因為`iter`會對向量裡面的項目的引用進行迭代, 如果我們在`map`裡的閉包的參數上忘了`&`符號, 我們將會得到下面的類型不匹配的錯誤:

```text
error[E0308]: mismatched types
  -->
   |
14 |         .map(|Point {x, y}| x * x + y * y)
   |               ^^^^^^^^^^^^ expected &Point, found struct `Point`
   |
   = note: expected type `&Point`
              found type `Point`
```

這個報錯提示Rust希望我們的閉包匹配參數匹配`&Point`, 但是我們卻試圖用一個`Point`的值的模式去匹配它, 而不是一個`Point`的引用.

我們可以用更複雜的方法來合成、匹配和嵌套解構模式: 下例中我們通過在一個元組中嵌套結構和元組來解構出所有的基礎類型的值:

```rust
# struct Point {
#     x: i32,
#     y: i32,
# }
#
let ((feet, inches), Point {x, y}) = ((3, 10), Point { x: 3, y: -10 });
```

這使得我們把複雜的類型提取成了它們的組成成分.

### 忽略模式中的值

有一些簡單的方法可以忽略模式中全部或部分值: 使用`_`模式, 在另一個模式中使用`_`模式, 使用一個以下劃線開始的名字, 或者使用`..`來忽略掉所有剩下的值. 下面讓我們來探索如何以及為什麼要這麼做.

#### 用`_`忽略整個值

我們已經見過了用下劃線作為通配符會匹配任意值, 但是它不會綁定值. 把下劃線模式用作`match`表達式的最後一個匹配分支特別有用, 我們可以在任意模式中使用它, 比如在例18-15中顯示的函數參數:

```rust
fn foo(_: i32) {
    // code goes here
}
```

<span class="caption">例18-15: 在一個函數簽名中使用`_`</span>

通常, 你應該把這種函數的參數聲明改成不用無用參數. 如果是要實現這樣一個有特定類型簽名的*trait*, 使用下劃線可以讓你忽略一個參數, 並且編譯器不會像使用命名參數那樣警告有未使用的函數參數.

#### 用一個嵌套的`_`忽略部分值

我們也可以在另一個模式中使用`_`來忽略部分值. 在例18-16中, 第一個`match`分支中的模式匹配了一個`Some`值, 但是卻通過下劃線忽略掉了`Some`變數中的值:

```rust
let x = Some(5);

match x {
    Some(_) => println!("got a Some and I don't care what's inside"),
    None => (),
}
```

<span class="caption">例18-16: 通過使用一個嵌套的下劃線忽略`Some`變數中的值</span>

當代碼關聯的`match`分支不需要使用被嵌套的全部變數時這很有用.

我們也可以在一個模式中多處使用下劃線, 在例18-17中我們將忽略掉一個五元元組中的第二和第四個值:

```rust
let numbers = (2, 4, 8, 16, 32);

match numbers {
    (first, _, third, _, fifth) => {
        println!("Some numbers: {}, {}, {}", first, third, fifth)
    },
}
```

<span class="caption">例18-17: 忽略元組中的多個部分</span>

上面的代碼將會打印出`Some numbers: 2, 8, 32`, 元組中的4和16會被忽略.

#### 通過在名字前以一個下劃線開頭來忽略不使用的變數

如果你創建了一個變數卻不使用它, Rust通常會給你一個警告, 因為這可能會是個bug. 如果你正在做原型或者剛開啟一個項目, 那麼你可能會創建一個暫時不用但是以後會使用的變數. 如果你面臨這個情況並且希望Rust不要對你警告未使用的變數, 你可以讓那個變數以一個下劃線開頭. 這和其它模式中的變數名沒什麼區別, 只是Rust不會警告你這個變數沒用被使用. 在例18-18中, 我們會得到一個沒用使用變數`y`的警告, 但是我們不會得到沒用使用變數`_x`的警告:

```rust
fn main() {
    let _x = 5;
    let y = 10;
}
```

<span class="caption">例18-18: 為了消除對未被使用變數的警告以一個下劃線開始來命名變數</span>

注意, 只使用`_`和使用一個以一個下劃線起頭的名字是有微妙的不同的: `_x`仍然會把值綁定到變數上但是`_`不會綁定值.

例18-19顯示了這種區別的主要地方: `s`將仍然被轉移到`_s`, 它會阻止我們繼續使用`s`:

```rust
let s = Some(String::from("Hello!"));

if let Some(_s) = s {
    println!("found a string");
}

println!("{:?}", s);
```

<span class="caption">例18-19: 以下劃線起頭的未被使用的變數仍然會綁定值, 它也會擁有值的所有權</span>

只使用下劃線本身卻不會綁定值. 例18-20在編譯時將不會報錯, 因為`s`不會被轉移到`_`:

```rust
let s = Some(String::from("Hello!"));

if let Some(_) = s {
    println!("found a string");
}

println!("{:?}", s);
```

<span class="caption">例18-20: 使用下劃線不會綁定值</span>

上面的代碼能很好的運行. 因為我們沒有把`s`綁定到其它地方, 它沒有被轉移.

#### 用`..`忽略剩餘的值

對於有多個字段的值而言, 我們可以只提取少數字段並使用`..`來代替下劃線, 這就避免了用`_`把剩餘的部分列出來的麻煩. `..`模式將忽略值中沒有被精確匹配值中的其它部分. 在例18-21中, 我們有一個持有三維空間坐標的`Point`結構. 在`match`表達式裡,
我們只想操作`x`坐標上的值並忽略`y`坐標和`z`坐標上的值:

```rust
struct Point {
    x: i32,
    y: i32,
    z: i32,
}

let origin = Point { x: 0, y: 0, z: 0 };

match origin {
    Point { x, .. } => println!("x is {}", x),
}
```

<span class="caption">例18-21: 通過用`..`來忽略除了`x`以外的所有其它`Point`的字段</span>

使用`..`比列出`y: _`和`z: _`寫起來更簡單. 當一個結構有很多字段但卻只需要使用少量字段時`..`模式就特別有用.

`..`將會囊括它能匹配的儘可能多的值. 例18-22顯示了一個在元組中使用`..`的情況:

```rust
fn main() {
    let numbers = (2, 4, 8, 16, 32);

    match numbers {
        (first, .., last) => {
            println!("Some numbers: {}, {}", first, last);
        },
    }
}
```

<span class="caption">例18-22: 用`..`匹配元組中的第一和最後一個值並忽略掉所有的其它值</span>

我們在這裡用`first`和`last`來匹配了第一和最後一個值. `..`將匹配並忽略中間的所有其它值.

然而使用`..`必須清晰明了. 例18-23中的代碼就不是很清晰, Rust看不出哪些值時我們想匹配的, 也看不出哪些值是我們想忽略的:

```rust
fn main() {
    let numbers = (2, 4, 8, 16, 32);

    match numbers {
        (.., second, ..) => {
            println!("Some numbers: {}", second)
        },
    }
}
```

<span class="caption">例18-23: 嘗試含混不清地使用`..`</span>

如果我們編譯上面的例子, 我們會得到下面的錯誤:

```text
error: `..` can only be used once per tuple or tuple struct pattern
 --> src/main.rs:5:22
  |
5 |         (.., second, ..) => {
  |                      ^^
```

上面的代碼中在一個值被匹配到`second`之前不可能知道元組中有多少值應該被忽略, 同樣在`second`被匹配後也不知道應該有多少值被忽略. 我們可以忽略2, 把`second`綁定到4, 然後忽略8、16和32, 或者我們也可以忽略2和4, 把`second`綁定到8, 然後再忽略16和32. 對Rust而言, 變數名`second`並不意味著某個確定的值, 因為像這樣在兩個地方使用`..`是含混不清的, 所以我們就得到了一個編譯錯誤.

### 用`ref`和`ref mut`在模式中創建引用

當你匹配一個模式時, 模式匹配的變數會被綁定到一個值. 也就是說你會把值轉移進`match`(或者是其它你使用了模式的地方), 這是所有權規則的作用. 例18-24提供了一個例子:

```rust
let robot_name = Some(String::from("Bors"));

match robot_name {
    Some(name) => println!("Found a name: {}", name),
    None => (),
}

println!("robot_name is: {:?}", robot_name);
```

<span class="caption">例18-24: 在一個匹配分支模式裡創建的變數會擁有值的所有權</span>

上例的代碼不能編譯通過, 因為`robot_name`中的值被轉移到了`match`中的`Some`的值所綁定的`name`裡了.

在模式中使用`&`會匹配已存在的引用中的值, 我們在"解構並提取值"這一節中已經見過了. 如果你想創建一個引用來借用模式中變數的值, 可以在新變數名前使用`ref`關鍵字, 比如例18-25:

```rust
let robot_name = Some(String::from("Bors"));

match robot_name {
    Some(ref name) => println!("Found a name: {}", name),
    None => (),
}

println!("robot_name is: {:?}", robot_name);
```

<span class="caption">例18-25: 創建一個引用這樣模式中的變數就不會擁有值的所有權</span>

上例可以編譯, 因為`robot_name`沒有被轉移到`Some(ref name)`匹配分支的`Some`變數中; 這個匹配分支只是持有`robot_name`中的數據, `robot_name`並沒被轉移.

如果要創建一個可變引用, 可以像例18-26那樣使用`ref mut`:

```rust
let mut robot_name = Some(String::from("Bors"));

match robot_name {
    Some(ref mut name) => *name = String::from("Another name"),
    None => (),
}

println!("robot_name is: {:?}", robot_name);
```

<span class="caption">例18-26: 在模式中使用`ref mut`來創建一個值的可變引用</span>

上例可以編譯並打印出`robot_name is: Some("Another name")`. 因為在匹配分支的代碼中`name`是一個可變引用, 為了能夠改變這個值, 我們需要用`*`操作符來對它解引用.

### 用了匹配守衛的額外條件

你可以通過在模式後面指定一個額外的`if`條件來往匹配分支中引入*匹配守衛*(*match guards*). 這個條件可以使用模式中創建的變數. 例18-27中的`match`表達式的第一個匹配分支就有一個匹配守衛:

```rust
let num = Some(4);

match num {
    Some(x) if x < 5 => println!("less than five: {}", x),
    Some(x) => println!("{}", x),
    None => (),
}
```

<span class="caption">例18-27: 往一個模式中加入匹配守衛</span>

上例會打印`less than five: 4`. 如果把`num`換成`Some(7)`, 上例將會打印`7`.  匹配守衛讓你能表達出模式不能給予你的更多的複雜的東西.

在例18-10中, 我們見過了模式中的陰影變數, 當一個值等於`match`外部的變數時我們不能用模式來表達出這種情況. 例18-28演示了我們如何用一個匹配守衛來解決這個問題:

```rust
fn main() {
    let x = Some(5);
    let y = 10;

    match x {
        Some(50) => println!("Got 50"),
        Some(n) if n == y => println!("Matched, n = {:?}", n),
        _ => println!("Default case, x = {:?}", x),
    }

    println!("at the end: x = {:?}, y = {:?}", x, y);
}
```

<span class="caption">例18-28: 用一個匹配守衛來測試與外部變數的相等性</span>

上例會打印出`Default case, x = Some(5)`. 因為第二個匹配分支沒有往模式中引入新變數`y`, 所以外部變數`y`就不會被遮掩, 這樣我們就可以在匹配守衛中直接使用外部變數`y`. 我們還把`x`解構到了內部變數`n`中, 這樣我們就可以在匹配守衛中比較`n`和`y`了.

如果你在由`|`指定的多模式中使用匹配守衛, 匹配守衛的條件就會應用到所有的模式上. 例18-29演示了在第一個匹配分支中的匹配守衛會在被匹配的全部三個模式的值上生效:

```rust
let x = 4;
let y = false;

match x {
    4 | 5 | 6 if y => println!("yes"),
    _ => println!("no"),
}
```

<span class="caption">例18-29: 用一個匹配守衛來合成多個模式</span>

上例會打印`no`因為條件`if`會應用到整個模式`4 | 5 |
6`上, 而不是只應用到最後一個值`6`上面. 換一種說法, 一個與模式關聯的匹配守衛的優先級是:

```text
(4 | 5 | 6) if y => ...
```

而不是:

```text
4 | 5 | (6 if y) => ...
```

### `@`綁定

為了既能測試一個模式的值又能創建一個綁定到值的變數, 我們可以使用`@`. 例18-30演示了在匹配分支中我們想測試一個`Message::Hello`的`id`字段是否位於`3...7`之間, 同時我們又想綁定這個值這樣我們可以在代碼中使用它:

```rust
enum Message {
    Hello { id: i32 },
}

let msg = Message::Hello { id: 5 };

match msg {
    Message::Hello { id: id @ 3...7 } => {
        println!("Found an id in range: {}", id)
    },
    Message::Hello { id: 10...12 } => {
        println!("Found an id in another range")
    },
    Message::Hello { id } => {
        println!("Found some other id: {}", id)
    },
}
```

<span class="caption">例18-30: 在測試模式中的值的時候用`@`符號來綁定值</span>

上例會打印`Found an id in range: 5`. 通過在範圍前指定`id @`, 我們就在測試模式的同時又捕獲了匹配範圍的值. 在第二個分支我們只有一個在模式中指定的範圍, 與這個分支關聯的代碼就不知道`id`是10還是11或12, 因為我們沒有把`id`的值保存在某個變數中: 我們只知道如果匹配分支代碼被執行這個值與範圍匹配. 在最後一個匹配分支中我們指定了一個無範圍的變數, 這個值就可以用在分支代碼中, 此時我們沒有對這個值進行任何其它的測試. 在一個模式中使用`@`讓我們可以測試模式中的值並把它保存在一個變數中.

## 總結

模式是Rust的一個很有用的特點, 它幫助區分不同類型的數據. 當被用在`match`語句中時, Rust確保你的模式覆蓋了每個可能的值. 在`let`語句和函數參數中的模式使得這些構造更加強大, 這些模式在賦值給變數的同時可以把值解構成更小的部分.

現在讓我們進入倒數第二章吧, 讓我們看一下Rust的某些高級特性.