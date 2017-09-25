## 什麼是面向物件？

> [ch17-01-what-is-oo.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch17-01-what-is-oo.md)
> <br>
> commit 2a9b2a1b019ad6d4832ff3e56fbcba5be68b250e

關於一個語言被稱為面向物件所需的功能，在編程社區內並未達成一致意見。Rust 被很多不同的編程範式影響；我們探索了十三章提到的來自函數式編程的特性。面向物件編程語言所共享的一些特性往往是物件、封裝和繼承。讓我們看一下這每一個概念的含義以及 Rust 是否支持他們。

### 物件包含數據和行為

`Design Patterns: Elements of Reusable Object-Oriented Software`這本書被俗稱為`The Gang of Four book`，是面向物件編程模式的目錄。它這樣定義面向物件編程：

> Object-oriented programs are made up of objects. An *object* packages both
> data and the procedures that operate on that data. The procedures are
> typically called *methods* or *operations*.
>
> 面向物件的程序是由物件組成的。一個**物件**包含數據和操作這些數據的過程。這些過程通常被稱為**方法**或**操作**。

在這個定義下，Rust 是面向物件的：結構體和枚舉包含數據而 impl 塊提供了在結構體和枚舉之上的方法。雖然帶有方法的結構體和枚舉並不被**稱為**物件，但是他們提供了與物件相同的功能，參考 Gang of Four 中物件的定義。

### 隱藏了實現細節的封裝

另一個通常與面向物件編程相關的方面是**封裝**（*encapsulation*）的思想：物件的實現細節不能被使用物件的代碼獲取到。唯一與物件交互的方式是通過物件提供的公有 API；使用物件的代碼無法深入到物件內部並直接改變數據或者行為。封裝使得改變和重構物件的內部時無需改變使用物件的代碼。

就像我們在第七章討論的那樣，可以使用`pub`關鍵字來決定模塊、類型函數和方法是公有的，而默認情況下一切都是私有的。比如，我們可以定義一個包含一個`i32`類型的 vector 的結構體`AveragedCollection `。結構體也可以有一個字段，該字段保存了 vector 中所有值的平均值。這樣，希望知道結構體中的 vector 的平均值的人可以隨時獲取它，而無需自己計算。`AveragedCollection`會為我們緩存平均值結果。列表 17-1 有`AveragedCollection`結構體的定義：

<span class="filename">文件名: src/lib.rs</span>

```rust
pub struct AveragedCollection {
    list: Vec<i32>,
    average: f64,
}
```

<span class="caption">列表 17-1: `AveragedCollection`結構體維護了一個整型列表和集合中所有元素的平均值。</span>

注意，結構體自身被標記為`pub`，這樣其他代碼可以使用這個結構體，但是在結構體內部的字段仍然是私有的。這是非常重要的，因為我們希望保證變數被增加到列表或者被從列表刪除時，也會同時更新平均值。可以通過在結構體上實現`add`、`remove`和`average`方法來做到這一點，如列表 17-2 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
# pub struct AveragedCollection {
#     list: Vec<i32>,
#     average: f64,
# }
impl AveragedCollection {
    pub fn add(&mut self, value: i32) {
        self.list.push(value);
        self.update_average();
    }

    pub fn remove(&mut self) -> Option<i32> {
        let result = self.list.pop();
        match result {
            Some(value) => {
                self.update_average();
                Some(value)
            },
            None => None,
        }
    }

    pub fn average(&self) -> f64 {
        self.average
    }

    fn update_average(&mut self) {
        let total: i32 = self.list.iter().sum();
        self.average = total as f64 / self.list.len() as f64;
    }
}
```

<span class="caption">列表 17-2: 在`AveragedCollection`結構體上實現了`add`、`remove`和`average`公有方法</span>

公有方法`add`、`remove`和`average`是修改`AveragedCollection`實例的唯一方式。當使用`add`方法把一個元素加入到`list`或者使用`remove`方法來刪除它時，這些方法的實現同時會調用私有的`update_average`方法來更新`average`字段。因為`list`和`average`是私有的，沒有其他方式來使得外部的代碼直接向`list`增加或者刪除元素，直接操作`list`可能會引發`average`字段不同步。`average`方法返回`average`字段的值，這使得外部的代碼只能讀取`average`而不能修改它。

因為我們已經封裝好了`AveragedCollection`的實現細節，將來可以輕鬆改變類似數據結構這些方面的內容。例如，可以使用`HashSet`代替`Vec`作為`list`字段的類型。只要`add`、`remove`和`average`公有函數的簽名保持不變，使用`AveragedCollection`的代碼就無需改變。如果將`List`暴露給外部代碼時，未必都是這樣，因為`HashSet`和`Vec`使用不同的方法增加或移除項，所以如果要想直接修改`list`的話，外部的代碼可能不得不修改。

如果封裝是一個語言被認為是面向物件語言所必要的方面的話，那麼 Rust 就滿足這個要求。在代碼中不同的部分使用或者不使用`pub`決定了實現細節的封裝。

## 作為類型系統的繼承和作為代碼共享的繼承

**繼承**（*Inheritance*）是一個很多編程語言都提供的機制，一個物件可以定義為繼承另一個物件的定義，這使其可以獲得父物件的數據和行為，而不用重新定義。一些人定義面向物件語言時，認為繼承是一個特色。

如果一個語言必須有繼承才能被稱為面向物件語言的話，那麼 Rust 就不是面向物件的。無法定義一個結構體繼承自另外一個結構體，從而獲得父結構體的成員和方法。然而，如果你過去常常在你的編程工具箱使用繼承，根據你希望使用繼承的原因，Rust 也提供了其他的解決方案。

使用繼承有兩個主要的原因。第一個是為了重用代碼：一旦為一個類型實現了特定行為，繼承可以對一個不同的類型重用這個實現。相反 Rust 代碼可以使用默認 trait 方法實現來進行共享，在列表 10-14 中我們見過在`Summarizable` trait 上增加的`summary`方法的默認實現。任何實現了`Summarizable` trait 的類型都可以使用`summary`方法而無須進一步實現。這類似於父類有一個方法的實現，而通過繼承子類也擁有這個方法的實現。當實現`Summarizable` trait 時也可以選擇覆蓋`summary`的默認實現，這類似於子類覆蓋從父類繼承的方法實現。

第二個使用繼承的原因與類型系統有關：用來表現子類型可以在父類型被使用的地方使用。這也被稱為**多態**（*polymorphism*），意味著如果多種物件有一個相同的形態大小，它們可以替代使用。

<!-- PROD: START BOX -->

> 雖然很多人使用「多態」（"polymorphism"）來描述繼承，但是它實際上是一種特殊的多態，稱為「子類型多態」（"sub-type polymorphism"）。也有很多種其他形式的多態，在 Rust 中帶有泛型參數的 trait bound 也是多態，更具體的說是「參數多態」（"parametric polymorphism"）。不同類型多態的確切細節在這裡並不關鍵，所以不要過於擔心細節，只需要知道 Rust 有多種多態相關的特色就好，不同於很多其他 OOP 語言。

<!-- PROD: END BOX -->

為了支持這種模式，Rust 有 **trait 物件**（*trait objects*），這樣就可以使用任意類型的值，只要這個值實現了指定的 trait。

繼承最近在很多編程語言的設計方案中失寵了。使用繼承來實現代碼重用，會共享更多非必需的代碼。子類不應該總是共享其父類的所有特性，然而繼承意味著子類得到了其父類全部的數據和行為。這使得程序的設計更不靈活，並產生了無意義的方法調用或子類，以及由於方法並不適用於子類，卻必需從父類繼承而可能造成的錯誤。另外，某些語言只允許子類繼承一個父類，進一步限制了程序設計的靈活性。

因為這些原因，Rust 選擇了一個另外的途徑，使用 trait 物件替代繼承。讓我們看一下在 Rust 中 trait 物件是如何實現多態的。