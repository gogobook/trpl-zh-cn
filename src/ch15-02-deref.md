## `Deref` Trait 允許通過引用訪問數據

> [ch15-02-deref.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch15-02-deref.md)
> <br>
> commit d06a6a181fd61704cbf7feb55bc61d518c6469f9

第一個智能指針相關的重要 trait 是 `Deref`，它允許我們重載 `*`，解引用運算符（不同於乘法運算符或全局引用運算符）。重載智能指針的 `*` 能使訪問其持有的數據更為方便，在本章結束前談到解引用強制多態（deref coercions）時我們會說明方便意味著什麼。

第八章的哈希 map 的 「根據舊值更新一個值」 部分簡要的提到瞭解引用運算符。當時有一個可變引用，而我們希望改變這個引用所指向的值。為此，首先我們必須解引用。這是另一個使用 `i32` 值引用的例子：

```rust
let mut x = 5;
{
    let y = &mut x;

    *y += 1
}

assert_eq!(6, x);
```

我們使用 `*y` 來訪問可變引用 `y` 所指向的數據，而不是可變引用本身。接著可以修改它的數據，在這裡是對其加一。

引用並不是智能指針，他們只是引用指向的一個值，所以這個解引用操作是很直接的。智能指針還會儲存指針或數據的元數據。當解引用一個智能指針時，我們只想要數據，而不需要元數據，因為解引用一個常規的引用只能給我們數據而不是元數據。我們希望能在使用常規引用的地方也能使用智能指針。為此，可以通過實現 `Deref` trait 來重載 `*` 運算符的行為。

代碼例 15-7 展示了一個定義為儲存 mp3 數據和元數據的結構體通過 `Deref` trait 來重載 `*` 的例子。`Mp3`，在某種意義上是一個智能指針：它擁有包含音頻的 `Vec<u8>` 數據。另外，它儲存了一些可選的元數據，在這個例子中是音頻數據中藝術家和歌曲的名稱。我們希望能夠方便的訪問音頻數據而不是元數據，所以需要實現 `Deref` trait 來返回音頻數據。實現 `Deref` trait 需要一個叫做 `deref` 的方法，它借用 `self` 並返回其內部數據：

<span class="filename">文件名: src/main.rs</span>

```rust
use std::ops::Deref;

struct Mp3 {
    audio: Vec<u8>,
    artist: Option<String>,
    title: Option<String>,
}

impl Deref for Mp3 {
    type Target = Vec<u8>;

    fn deref(&self) -> &Vec<u8> {
        &self.audio
    }
}

fn main() {
    let my_favorite_song = Mp3 {
        // we would read the actual audio data from an mp3 file
        audio: vec![1, 2, 3],
        artist: Some(String::from("Nirvana")),
        title: Some(String::from("Smells Like Teen Spirit")),
    };

    assert_eq!(vec![1, 2, 3], *my_favorite_song);
}
```

<span class="caption">代碼例 15-7：一個存放 mp3 文件數據和元數據的結構體上的 `Deref` trait 實現</span>

大部分代碼看起來都比較熟悉：一個結構體、一個 trait 實現、和一個創建了結構體實例的 main 函數。其中有一部分我們還未全面的講解：類似於第十三章學習迭代器 trait 時出現的 `type Item`，`type Target = T;` 語法用於定義關聯類型，第十九章會更詳細的介紹。不必過分擔心例子中的這一部分；它只是一個稍顯不同的定義泛型參數的方式。

在 `assert_eq!` 中，我們驗證 `vec![1, 2, 3]` 是否為 `Mp3` 實例 `*my_favorite_song` 解引用的值，結果正是如此，因為我們實現了 `deref` 方法來返回音頻數據。如果沒有為 `Mp3` 實現 `Deref` trait，Rust 將不會編譯 `*my_favorite_song`：會出現錯誤說 `Mp3` 類型不能被解引用。

沒有 `Deref` trait 的話，編譯器只能解引用 `&` 引用，而 `my_favorite_song` 並不是（它是一個 `Mp3` 結構體）。通過 `Deref` trait，編譯器知道實現了 `Deref` trait 的類型有一個返回引用的 `deref` 方法（在這個例子中，是 `&self.audio` 因為代碼例 15-7 中的 `deref` 的定義）。所以為了得到一個 `*` 可以解引用的 `&` 引用，編譯器將 `*my_favorite_song` 展開為如下：

```rust,ignore
*(my_favorite_song.deref())
```

其結果就是 `self.audio` 中的值。`deref` 返回一個引用並接下來必需解引用而不是直接返回值的原因是所有權：如果 `deref` 方法直接返回值而不是引用，其值將被移動出 `self`。和大部分使用解引用運算符的地方相同，這裡並不想獲取 `my_favorite_song.audio` 的所有權。

注意將 `*` 替換為 `deref` 調用和 `*` 調用的過程在每次使用 `*` 的時候都會發生一次。`*` 的替換並不會無限遞歸進行。最終的數據類型是 `Vec<u8>`，它與代碼例 15-7 中 `assert_eq!` 的 `vec![1, 2, 3]` 相匹配。

### 函數和方法的隱式解引用強制多態

Rust 傾向於偏愛明確而不是隱晦，不過一個情況下這並不成立，就是函數和方法的參數的 **解引用強制多態**（*deref coercions*）。解引用強制多態會自動的將指針或智能指針的引用轉換為指針內容的引用。解引用強制多態發生於當傳遞給函數的參數類型不同於函數簽名中定義參數類型的時候。解引用強制多態的加入使得 Rust 調用函數或方法時無需很多顯式使用 `&` 和 `*` 的引用和解引用。

使用代碼例 15-7 中的 `Mp3` 結構體，如下是一個獲取 `u8` slice 並壓縮 mp3 音頻數據的函數簽名：

```rust,ignore
fn compress_mp3(audio: &[u8]) -> Vec<u8> {
    // the actual implementation would go here
}
```

如果 Rust 沒有解引用強制多態，為了使用 `my_favorite_song` 中的音頻數據調用此函數，必須寫成：

```rust,ignore
compress_mp3(my_favorite_song.audio.as_slice())
```

也就是說，必須明確表明需要 `my_favorite_song` 中的 `audio` 字段而且我們希望有一個 slice 來引用這整個 `Vec<u8>`。如果有很多地方需要用相同的方式處理 `audio` 數據，那麼 `.audio.as_slice()` 就顯得冗長重複了。

然而，因為解引用強制多態和 `Mp3` 的 `Deref` trait 實現，我們可以使用如下代碼使用 `my_favorite_song` 中的數據調用這個函數：

```rust,ignore
let result = compress_mp3(&my_favorite_song);
```

只有 `&` 和實例，好的！我們可以把智能指針當成普通的引用那樣使用。也就是說解引用強制多態意味著 Rust 利用了 `Deref` 實現的優勢：Rust 知道 `Mp3` 實現了 `Deref` trait 並從 `deref` 方法返回 `&Vec<u8>`。它也知道標準庫實現了 `Vec<T>`的 `Deref` trait，其 `deref` 方法返回 `&[T]`（我們也可以通過查閱 `Vec<T>` 的 API 文檔來發現這一點）。所以，在編譯時，Rust 會發現它可以調用兩次 `Deref::deref` 來將 `&Mp3` 變成 `&Vec<u8>` 再變成 `&[T]` 來滿足 `compress_mp3` 的簽名。這意味著我們可以少寫一些代碼！Rust 會多次分析 `Deref::deref` 的返回值類型直到它滿足參數的類型，只要相關類型實現了 `Deref` trait。這些間接轉換在編譯時進行，所以利用解引用強制多態並沒有運行時懲罰！

類似於如何使用 `Deref` trait 重載 `&T` 的 `*` 運算符，`DerefMut` trait 用於重載 `&mut T` 的 `*` 運算符。

Rust 在發現類型和 trait 實現滿足三種情況時會進行解引用強制多態：

* 當 `T: Deref<Target=U>` 時從 `&T` 到 `&U`。
* 當 `T: DerefMut<Target=U>` 時從 `&mut T` 到 `&mut U`。
* 當 `T: Deref<Target=U>` 時從 `&mut T` 到 `&U`。

頭兩個情況除了可變性之外是相同的：如果有一個 `&T`，而 `T` 實現了返回 `U` 類型的 `Deref`，則可以直接得到 `&U`。對於可變引用也是一樣。最後一個有些微妙：如果有一個可變引用，它也可以強轉為一個不可變引用。反之則是 **不可能** 的：不可變引用永遠也不能強轉為可變引用。

`Deref` trait 對於智能指針模式十分重要的原因在於智能指針可以被看作普通引用並被用於期望使用普通引用的地方。例如，無需重新定義方法和函數來直接獲取智能指針。