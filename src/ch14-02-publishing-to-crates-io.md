## 將 crate 發佈到 Crates.io

> [ch14-02-publishing-to-crates-io.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch14-02-publishing-to-crates-io.md)
> <br>
> commit 56352c28cf3fe0402fa5a7cba73890e314d720eb

我們曾經在項目中增加 crates.io 上的包作為依賴，不過你也可以通過發佈自己的包來向它人分享代碼。Crates.io 用來分發包的源代碼，所以它主要託管開源代碼。

Rust 和 Cargo 有一些幫助它人更方便找到和使用你發佈的包的功能。我們將介紹一些這樣的功能，接著講到如何發佈一個包。


### 編寫有用的文檔註釋

準確的包文檔有助於其他用戶立即如何以及何時使用他們，所以花一些時間編寫文檔是值得的。第三章中我們討論了如何使用 `//` 註釋 Rust 代碼。Rust 也有特定的用於文檔的註釋類型，通常被稱為 **文檔註釋**（*documentation comments*），他們會生成 HTML 文檔。這些 HTML 展示公有 API 文檔註釋的內容，他們意在讓對庫感興趣的程式設計師理解如何 **使用** 這個 crate，而不是它是如何被 **實現** 的。

文檔註釋使用 `///` 而不是 `//` 並支持 Markdown 註解來格式化文本。文檔註釋就位於需要文檔的項的之前。示例 14-2 展示了一個 `my_crate` crate 中 `add_one` 函數的文檔註釋：

<span class="filename">文件名: src/lib.rs</span>

```rust
/// Adds one to the number given.
///
/// # Examples
///
/// ```
/// let five = 5;
///
/// assert_eq!(6, my_crate::add_one(5));
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

<span class="caption">示例 14-2：一個函數的文檔註釋</span>

這裡，我們提供了一個 `add_one` 函數工作的描述，接著開始了一個標題為 「Examples」 的部分，和展示如何使用 `add_one` 函數的代碼。可以運行 `cargo doc` 來生成這個文檔註釋的 HTML 文檔。這個命令運行由 Rust 分發的工具 `rustdoc` 並將生成的 HTML 文檔放入 *target/doc* 目錄。

為了方便起見，運行 `cargo doc --open` 會構建當前 crate 文檔（同時還有所有 ceate 依賴的文檔）的 HTML 並在瀏覽器中打開。導航到 `add_one` 函數將會發現文檔註釋的文本是如何渲染的，如圖 13-3 所示：

<img alt="`my_crate` 的 `add_one` 函數所渲染的文檔註釋 HTML" src="img/trpl14-03.png" class="center" />

<span class="caption">圖 14-3：`add_one` 函數的文檔註釋 HTML</span>

#### 常用（文檔註釋）部分

示例 14-2 中使用了 `# Examples` Markdown 標題在 HTML 中創建了一個以 「Examples」 為標題的部分。一些其他經常在文檔註釋中使用的部分有：

- Panics：這個函數可能會 `panic!` 的場景。並不希望程序崩潰的函數調用者應該確保他們不會在這些情況下調用此函數。
- Errors：如果這個函數返回 `Result`，此部分描述可能會出現何種錯誤以及什麼情況會造成這些錯誤，這有助於調用者編寫代碼來採用不同的方式處理不同的錯誤。
- Safety：如果這個函數使用 `unsafe` 代碼（這會在第十九章討論），這一部分應該會涉及到期望函數調用者支持的確保 `unsafe` 塊中代碼正常工作的不變條件（invariants）。

大部分文檔註釋不需要所有這些部分，不過這是一個提醒你檢查調用你代碼的人有興趣瞭解的內容的示例。

#### 文檔註釋作為測試

在文檔註釋中增加示例代碼塊是一個清楚的表明如何使用庫的方法，這麼做還有一個額外的好處：`cargo test` 也會像測試那樣運行文檔中的示例代碼！沒有什麼比有例子的文檔更好的了！也沒有什麼比不能正常工作的例子更糟的了，因為代碼在編寫文檔時已經改變。嘗試 `cargo test` 運行像示例 14-2 中 `add_one` 函數的文檔；應該在測試結果中看到像這樣的部分：

```text
   Doc-tests my_crate

running 1 test
test src/lib.rs - add_one (line 5) ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

現在嘗試改變函數或例子來使例子中的 `assert_eq!` 產生 panic。再次運行 `cargo test`，你將會看到文檔測試捕獲到了例子與代碼不再同步！

#### 註釋包含項的結構

還有另一種風格的文檔註釋，`//!`，這為包含註釋的項，而不是註釋之後的項增加文檔。這通常用於 crate 根文件或模組的根文件為 crate 或模組整體提供文檔。

作為一個例子，如果我們希望增加描述包含 `add_one` 函數的 `my_crate` crate 目的的文檔，可以在 *src/lib.rs* 開頭增加以 `//!` 開頭的註釋，如示例 14-4 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
//! # My Crate
//!
//! `my_crate` is a collection of utilities to make performing certain
//! calculations more convenient.

/// Adds one to the number given.
// ...snip...
```

<span class="caption">示例 14-4：`my_crate` crate 整體的文檔</span>

注意 `//!` 的最後一行之後沒有任何代碼。因為他們以 `//!` 開頭而不是 `///`，這是屬於包含此註釋的項而不是註釋之後項的文檔。在這個情況中，包含這個註釋的項是 *src/lib.rs* 文件，也就是 crate 根文件。這些註釋描述了整個 crate。

如果運行 `cargo doc --open`，將會發現這些註釋顯示在 `my_crate` 文檔的首頁，位於 crate 中公有項示例之上，如圖 14-5 所示：

<img alt="crate 整體註釋所渲染的 HTML 文檔" src="img/trpl14-05.png" class="center" />

<span class="caption">圖 14-5：包含 `my_crate` 整體描述的註釋所渲染的文檔</span>

位於項之中的文檔註釋對於描述 crate 和模組特別有用。使用他們描述其容器整體的目的來幫助 crate 用戶理解你的代碼組織。


### 使用 `pub use` 導出合適的公有 API

第七章介紹了如何使用 `mod` 關鍵字來將代碼組織進模組中，如何使用 `pub` 關鍵字將項變為公有，和如何使用 `use` 關鍵字將項引入作用域。然而對你開發來說很有道理的結果可能對用戶來說就不太方便了。你可能希望將結構組織進有多個層次的層級中，不過想要使用被定義在很深層級中的類型的人可能很難發現這些類型是否存在。他們也可能會厭煩 `use my_crate::some_module::another_module::UsefulType;` 而不是 `use my_crate::UsefulType;` 來使用類型。

公有 API 的結構是你發佈 crate 時主要需要考慮的。crate 用戶沒有你那麼熟悉其結構，並且如何模組層級過大他們可能會難以找到所需的部分。

好消息是，如果結果對於用戶來說 **不是** 很方便，你也無需重新安排內部組織：你可以選擇使用 `pub use` 重導出（re-export）項來使公有結構不同於私有結構。重導出抓取位於一個位置的公有項並將其公開到另一個位置，好像它就定義在這個新位置一樣。

例如，假設我們創建了一個模組化了充滿藝術化氣息的庫 `art`。在這個庫中是一個包含兩個枚舉 `PrimaryColor` 和 `SecondaryColor` 的模組 `kinds`，以及一個包含函數 `mix` 的模組 `utils`，如示例 14-6 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
//! # Art
//!
//! A library for modeling artistic concepts.

pub mod kinds {
    /// The primary colors according to the RYB color model.
    pub enum PrimaryColor {
        Red,
        Yellow,
        Blue,
    }

    /// The secondary colors according to the RYB color model.
    pub enum SecondaryColor {
        Orange,
        Green,
        Purple,
    }
}

pub mod utils {
    use kinds::*;

    /// Combines two primary colors in equal amounts to create
    /// a secondary color.
    pub fn mix(c1: PrimaryColor, c2: PrimaryColor) -> SecondaryColor {
        // ...snip...
#        SecondaryColor::Green
    }
}
```

<span class="caption">示例 14-6：一個庫 `art` 其組織包含 `kinds` 和 `utils` 模組</span>

`cargo doc` 所生成的 crate 文檔首頁如圖 14-7 所示：

<img alt="包含 `kinds` 和 `utils` 模組的 `art`" src="img/trpl14-07.png" class="center" />

<span class="caption">圖 14-7：包含 `kinds` 和 `utils` 模組的庫 `art` 的文檔首頁</span>

注意 `PrimaryColor` 和 `SecondaryColor` 類型沒有在首頁中列出，`mix` 函數也是。必須點擊 `kinds` 或 `utils` 才能看到他們。

另一個依賴這個庫的 crate 需要 `use` 語句來導入 `art` 中的項，這包含指定其當前定義的模組結構。示例 14-8 展示了一個使用 `art` crate 中 `PrimaryColor` 和 `mix` 項的 crate 的例子：

<span class="filename">文件名: src/main.rs</span>

```rust
extern crate art;

use art::kinds::PrimaryColor;
use art::utils::mix;

fn main() {
    let red = PrimaryColor::Red;
    let yellow = PrimaryColor::Yellow;
    mix(red, yellow);
}
```

<span class="caption">示例 14-8：一個通過導出內部結構使用 `art` crate 中項的 crate</span>

示例 14-8 中使用 `art` crate 代碼的作者不得不搞清楚 `PrimaryColor` 位於 `kinds` 模組而 `mix` 位於 `utils` 模組。`art` crate 的模組結構相比使用它的開發者來說對編寫它的開發者更有意義。其內部的 `kinds` 模組和 `utils` 模組的組織結構並沒有對嘗試理解如何使用它的人提供任何有價值的信息。`art` crate 的模組結構因不得不搞清楚所需的內容在何處和必須在 `use` 語句中指定模組名稱而顯得混亂和不便。

為了從公有 API 中去掉 crate 的內部組織，我們可以採用示例 14-6 中的 `art` crate 並增加 `pub use` 語句來重導出項到頂層結構，如示例 14-9 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust
//! # Art
//!
//! A library for modeling artistic concepts.

pub use kinds::PrimaryColor;
pub use kinds::SecondaryColor;
pub use utils::mix;

pub mod kinds {
    // ...snip...
}

pub mod utils {
    // ...snip...
}
```

<span class="caption">示例 14-9：增加 `pub use` 語句重導出項</span>

現在此 crate 由 `cargo doc` 生成的 API 文檔會在首頁列出重導出的項以及其鏈接，如圖 14-10 所示，這就使得這些類型易於查找。

<img alt="Rendered documentation for the `art` crate with the re-exports on the front page" src="img/trpl14-10.png" class="center" />

<span class="caption">圖 14-10：`art` 文檔的首頁，這裡列出了重導出的項</span>

`art` crate 的用戶仍然可以看見和選擇使用示例 14-8 中的內部結構，或者可以使用示例 14-9 中更為方便的結構，如示例 14-11 所示：

<span class="filename">文件名: src/main.rs</span>

```rust
extern crate art;

use art::PrimaryColor;
use art::mix;

fn main() {
    // ...snip...
}
```

<span class="caption">示例 14-11：一個使用 `art` crate</span>

對於有很多嵌套模組的情況，使用 `pub use` 將類型重導出到頂級結構對於使用 crate 的人來說將會是大為不同的體驗。

創建一個有用的公有 API 結構更像是一門藝術而非科學，你可以反覆檢視他們來找出最適合用戶的 API。選擇 `pub use` 提供了組織 crate 內部結構和與終端用戶體現解耦的靈活性。觀察一些你所安裝的 crate 的代碼來看看其內部結構是否不同於公有 API。

### 創建 Crates.io 賬號

在你可以發佈任何 crate 之前，需要在 crates.io 上註冊賬號並抓取一個 API token。為此，訪問位於 *https://crates.io* 的官網並使用 GitHub 賬號登陸————目前 GitHub 賬號是必須的，不過將來該網站可能會支持其他創建賬號的方法。一旦登陸之後，查看位於 *https://crates.io/me* 的賬戶設置頁面並抓取 API token。接著使用該 API token 運行 `cargo login` 命令，像這樣：

```text
$ cargo login abcdefghijklmnopqrstuvwxyz012345
```

這個命令會通知 Cargo 你的 API token 並將其儲存在本地的 *~/.cargo/config* 文件中。注意這個 token 是一個 **秘密**（**secret**）且不應該與其他人共享。如果因為任何原因與他人共享了這個信息，應該立即重新生成這個 token。

### 發佈新 crate 之前

有了賬號之後，比如說你已經有一個希望發佈的 crate。在發佈之前，你需要在 crate 的 *Cargo.toml* 文件的 `[package]` 部分增加一些本 crate 的元信息（metadata）。

首先 crate 需要一個唯一的名稱。雖然在本地開發 crate 時，可以使用任何你喜歡的名稱。不過 Crates.io 上的 crate 名稱遵守先到先得的分配原則。一旦某個 crate 名稱被使用，其他人就不能再發佈這個名稱的 crate 了。請在網站上搜索你希望使用的名稱來找出它是否已被使用。如果沒有，修改 *Cargo.toml* 中 `[package]` 裡的名稱為你希望用於發佈的名稱，像這樣：

```toml
[package]
name = "guessing_game"
```

即使你選擇了一個唯一的名稱，如果此時嘗試運行 `cargo publish` 發佈該 crate 的話，會得到一個一個警告接著是一個錯誤：

```text
$ cargo publish
    Updating registry `https://github.com/rust-lang/crates.io-index`
warning: manifest has no description, license, license-file, documentation,
homepage or repository.
...snip...
error: api errors: missing or empty metadata fields: description, license.
```

這是因為我們缺少一些關鍵信息：關於該 crate 用途的描述和用戶可能在何種條款下使用該 crate 的 license。為了修正這個錯誤，需要在 *Cargo.toml* 中引入這些信息。

描述通常是一兩句話，因為它會出現在 crate 的搜索結果中和 crate 頁面裡。對於 `license` 字段，你需要一個 **license 標識符值 **（*license identifier value*）。Linux 基金會位於 *http://spdx.org/licenses/* 的 Software Package Data Exchange (SPDX) 列出了可以使用的標識符。例如，為了指定 crate 使用 MIT License，增加 `MIT` 標識符：

```toml
[package]
name = "guessing_game"
license = "MIT"
```

如果你希望使用不存在於 SPDX 的 license，則需要將 license 文本放入一個文件，將該文件包含進項目中，接著使用 `license-file` 來指定文件名而不是使用 `license` 字段。

關於項目所適用的 license 指導超出了本書的範疇。很多 Rust 社區成員選擇與 Rust 自身相同的 license，這是一個雙許可的 `MIT/Apache-2.0`————這展示了也可以通過斜槓來分隔來指定多個 license 標識符。

那麼，有了唯一的名稱、版本號、由 `cargo new` 新建項目時增加的作者信息、描述和所選擇的 license，已經準備好發佈的項目的 *Cargo.toml* 文件可能看起來像這樣：

```toml
[package]
name = "guessing_game"
version = "0.1.0"
authors = ["Your Name <you@example.com>"]
description = "A fun game where you guess what number the computer has chosen."
license = "MIT/Apache-2.0"

[dependencies]
```

[Cargo 的文檔](http://doc.rust-lang.org/cargo/) 描述了其他可以指定的元信息，他們可以幫助你的 crate 更容易被發現和使用！

### 發佈到 Crates.io

現在我們創建了一個賬號，保存了 API token，為 crate 選擇了一個名字，並指定了所需的元數據，你已經準備好發佈了！發佈 crate 會上傳特定版本的 crate 到 crates.io 以供他人使用。

發佈 crate 時請多加小心，因為發佈是 **永久性的**（*permanent*）。對應版本不可能被覆蓋，其代碼也不可能被刪除。Crates.io 的一個主要目標是作為一個代碼的永久文檔服務器，這樣所有依賴 Crates.io 中 crate 的項目都能一直正常工作。允許刪除版本將不可能滿足這個目標。然而，可以被發布的版本號卻沒有限制。

讓我們再次運行`cargo publish`命令。這次它應該會成功：

```text
$ cargo publish
 Updating registry `https://github.com/rust-lang/crates.io-index`
Packaging guessing_game v0.1.0 (file:///projects/guessing_game)
Verifying guessing_game v0.1.0 (file:///projects/guessing_game)
Compiling guessing_game v0.1.0
(file:///projects/guessing_game/target/package/guessing_game-0.1.0)
 Finished dev [unoptimized + debuginfo] target(s) in 0.19 secs
Uploading guessing_game v0.1.0 (file:///projects/guessing_game)
```

恭喜！你現在向 Rust 社區分享了代碼，而且任何人都可以輕鬆的將你的 crate 加入他們項目的依賴。

### 發佈現存 crate 的新版本

當你修改了 crate 並準備好發佈新版本時，改變 *Cargo.toml* 中 `version` 所指定的值。請使用 [語義化版本規則][semver] 來根據修改的類型決定下一個版本呢號。接著運行 `cargo publish` 來上傳新版本。

[semver]: http://semver.org/

### 使用 `cargo yank` 從 Crates.io 撤回版本

雖然你不能刪除之前版本的 crate，但是可以阻止任何將來的項目將他們加入到依賴中。這在某個版本因為這樣或那樣的原因被破壞的情況很有用。對於這種情況，Cargo 支持 **撤回**（*yanking*）某個版本。

撤回某個版本會阻止新項目開始依賴此版本，不過所有現存此依賴的項目仍然能夠下載和依賴這個版本。從本質上說，撤回意味著所有帶有 *Cargo.lock* 的項目的依賴不會被破壞，同時任何新生成的 *Cargo.lock* 將不能使用被撤回的版本。

為了撤回一個 crate，運行 `cargo yank` 並指定希望撤回的版本：

```text
$ cargo yank --vers 1.0.1
```

也可以撤銷撤回操作，並允許項目可以再次開始依賴某個版本，通過在命令上增加 `--undo`：


```text
$ cargo yank --vers 1.0.1 --undo
```

撤回 **並沒有** 刪除任何代碼。舉例來說，撤回功能並不意在刪除不小心上傳的秘密信息。如果出現了這種情況，請立即重新設置這些秘密信息。