## 發佈配置

> [ch14-01-release-profiles.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch14-01-release-profiles.md)
> <br>
> commit db6129a30d7c7baed34dd38dbc56f7ed8a66ae92

在 Rust 中 **發佈配置**（*release profiles*）是預定義的、可定製的帶有不同選項的配置，他們允許程序員更多的控制代碼編譯的多種選項。每一個配置都彼此相互獨立。

Cargo 定義了四種有著良好默認值的可用於各自使用場景的配置。Cargo 根據運行的命令來選擇不同的配置。不同命令所對應的配置如表格 14-1 所示：

| 命令                 | 配置   |
|-------------------------|-----------|
| `cargo build`           | `dev`     |
| `cargo build --release` | `release` |
| `cargo test`            | `test`    |
| `cargo doc`             | `doc`     |

<span class="caption">表格 14-1：運行不同 Cargo 命令所使用的配置</span>

這可能很熟悉，他們出現在構建的輸出中，他們展示了構建中所使用的配置：

```text
$ cargo build
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
$ cargo build --release
    Finished release [optimized] target(s) in 0.0 secs
```

這裡的 「dev」 和 「release」 提示表明編譯器在使用不同的配置。

### 定製發佈配置

Cargo 對每一個配置都有默認設置，當項目的 *Cargo.toml* 文件的 `[profile.*]` 部分沒有指定時使用。通過增加任何希望定製的配置對應的 `[profile.*]` 部分，我們可以選擇覆蓋任意默認設置的子集。例如，如下是 `dev` 和 `release` 配置的 `opt-level` 設置的默認值：

```toml
[profile.dev]
opt-level = 0

[profile.release]
opt-level = 3
```

`opt-level` 設置控制 Rust 會對代碼進行何種程度的優化。這個配置的值從 0 到 3。越高的優化級別需要更多的時間編譯，所以如果你在進行開發並經常編譯，可能會希望在犧牲一些代碼性能的情況下編譯得快一些。這就是為什麼 `dev` 的 `opt-level` 默認為 `0`。當你準備發佈時，花費更多時間在編譯上則更好。只需要在發佈模式編譯一次，而編譯出來的程序則會運行很多次，所以發佈模式用更長的編譯時間換取運行更快的代碼。這正是為什麼 `release` 配置的 `opt-level` 默認為 `3`。

我們可以選擇通過在 *Cargo.toml* 增加不同的值來覆蓋任何默認設置。比如，如果我們想要在開發配置中使用級別 1 的優化，則可以在 *Cargo.toml* 中增加這兩行：

<span class="filename">文件名: Cargo.toml</span>

```toml
[profile.dev]
opt-level = 1
```

這會覆蓋默認的設置 `0`。現在運行 `cargo build` 時，Cargo 將會使用 `dev` 的默認配置加上定製的 `opt-level`。因為 `opt-level` 設置為 `1`，Cargo 會比默認進行更多的優化，但是沒有發佈構建那麼多。

對於每個配置的設置和其默認值的完整列表，請查看[Cargo 的文檔][cargodoc]。

[cargodoc]: http://doc.crates.io/