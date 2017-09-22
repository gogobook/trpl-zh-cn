## Refutability(可反駁性): 模式是否會匹配失效

匹配模式有兩種形式: refutable(可反駁)和irrefutable(不可反駁). 對任意可能的值進行匹配都不會失效的模式被稱為是*irrefutable*(不可反駁)的, 而對某些可能的值進行匹配會失效的模式被稱為是*refutable*(可反駁)的.
`let`語句、 函數參數和`for`循環被約束為只接受*irrefutable*模式, 因為如果模式匹配失效程序就不會正確運行. `if let`和`while let`表達式被約束為只接受*refutable*模式, 因為它們需要處理可能存在的匹配失效的情況, 並且如果模式匹配永不失效, 那它們就派不上用場了.

通常, 你不用關心*refutable*和*irrefutable*模式的區別, 當你看見它出現在了錯誤消息中時, 你只要瞭解*可反駁性*(refutability)的概念即可. 如果你得到一個涉及到可反駁性概念的錯誤消息, 根據你的代碼行為的意圖, 你只需改變匹配模式或者是改變你構造模式的方法即可.

讓我們來看幾個例子. 在本章的前面部分, 我們提到`let x = 5;`. 這裡`x`就是一個我們被允許使用*irrefutable*的模式: 因為它不可能匹配失效. 相反, 如果用`let`來匹配一個枚舉的變體, 比如像**例18-7**中列出的那樣從`Option<T>`枚舉中只匹配`Some<T>`這個值:

```rust
let Some(x) = some_option_value;
```

<span class="caption">例18-7: 試試用一個有`let`的*refutable*模式</span>

如果`some_option_value`的值是`None`, `some_option_value`將不會匹配模式`Some(x)`. 模式`Some(x)`是可反駁的(refutable), 因為存在一個使它匹配失效的值. 如果`some_option_value`的值是`None`, 那麼`let`語句就不會產生任何效果. 因此Rust會在編譯時會報*期望irrefutable模式但是卻得到了一個refutable模式*的錯誤:

```text
error[E0005]: refutable pattern in local binding: `None` not covered
 --> <anon>:3:5
  |
3 | let Some(x) = some_option_value;
  |     ^^^^^^^ pattern `None` not covered
```

因為我們沒有(也不能)覆蓋到模式`Some(x)`的每一個可能的值, 所以Rust會報錯.

如果我們採用*refutable*模式, 使用`if let`而不是`let`. 這樣當模式不匹配時, 在花括號中的代碼將不執行, 這段代碼只有在值匹配模式的時候才會執行, 也只在此時才有意義. 例18-8顯示了如何修正在例18-7中用`Some(x)`來匹配`some_option_value`的代碼. 因為這個例子使用了`if let`, 因此使用*refutable*模式的`Some(x)`就沒問題了:

```rust
# let some_option_value: Option<i32> = None;
if let Some(x) = some_option_value {
    println!("{}", x);
}
```

<span class="caption">例18-8: 使用`if let`和一個有*refutable*模式的代碼塊來代替`let`</span>

此外, 如果我們給`if let`一個絕對會匹配的*irrefutable*模式, 比如在例18-9中顯示的`x`:

```rust
if let x = 5 {
    println!("{}", x);
};
```

<span class="caption">例18-9: 嘗試把一個*irrefutable*模式用到`if let`上</span>

Rust將會抱怨把`if let`和一個*irrefutable*模式一起使用沒有意義:

```text
error[E0162]: irrefutable if-let pattern
 --> <anon>:2:8
  |
2 | if let x = 5 {
  |        ^ irrefutable pattern
```

一般來說, 多數匹配使用*refutable*模式, 除非是那種可以匹配任意值的情況使用*irrefutable*模式. `match`操作符中如果只有一個*irrefutable*模式分支也沒有什麼問題, 但這就沒什麼特別的用處, 此時可以用一個更簡單的`let`語句來替換. 不管是把表達式關聯到`let`語句亦或是關聯到只有一個*irrefutable*模式分支的`match`操作, 代碼都肯定會運行, 如果它們的表達式一樣的話最終的結果也相同.

目前我們已經討論了所有可以使用模式的地方, 也介紹了*refutable*模式和*irrefutable*模式的不同, 下面讓我們一起去把可以用來創建模式的語法過目一遍吧.
