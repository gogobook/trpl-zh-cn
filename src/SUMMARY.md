# Rust 程序設計語言

## 入門指南

- [介紹](ch01-00-introduction.md)
    - [安裝](ch01-01-installation.md)
    - [Hello, World!](ch01-02-hello-world.md)

- [猜猜看教程](ch02-00-guessing-game-tutorial.md)

- [通用編程概念](ch03-00-common-programming-concepts.md)
    - [變數和可變性](ch03-01-variables-and-mutability.md)
    - [數據類型](ch03-02-data-types.md)
    - [函數如何工作](ch03-03-how-functions-work.md)
    - [註釋](ch03-04-comments.md)
    - [控制流](ch03-05-control-flow.md)

- [了解所有權](ch04-00-understanding-ownership.md)
    - [什麼是所有權](ch04-01-what-is-ownership.md)
    - [引用 & 借用](ch04-02-references-and-borrowing.md)
    - [Slices](ch04-03-slices.md)

- [使用結構體組織相關聯的數據](ch05-00-structs.md)
    - [定義並實例化結構體](ch05-01-defining-structs.md)
    - [一個使用結構體的示例程序](ch05-02-example-structs.md)
    - [方法語法](ch05-03-method-syntax.md)

- [枚舉和模式匹配](ch06-00-enums.md)
    - [定義枚舉](ch06-01-defining-an-enum.md)
    - [`match` 控制流運算符](ch06-02-match.md)
    - [`if let` 簡單控制流](ch06-03-if-let.md)

## 基本 Rust 技能

- [模組](ch07-00-modules.md)
    - [`mod`和文件系統](ch07-01-mod-and-the-filesystem.md)
    - [使用`pub`控制可見性](ch07-02-controlling-visibility-with-pub.md)
    - [使用`use`導入命名](ch07-03-importing-names-with-use.md)

- [通用集合類型](ch08-00-common-collections.md)
    - [vector](ch08-01-vectors.md)
    - [字符串](ch08-02-strings.md)
    - [哈希 map](ch08-03-hash-maps.md)

- [錯誤處理](ch09-00-error-handling.md)
    - [`panic!`與不可恢復的錯誤](ch09-01-unrecoverable-errors-with-panic.md)
    - [`Result`與可恢復的錯誤](ch09-02-recoverable-errors-with-result.md)
    - [`panic!`還是不`panic!`](ch09-03-to-panic-or-not-to-panic.md)

- [泛型、trait 和生命週期](ch10-00-generics.md)
    - [泛型數據類型](ch10-01-syntax.md)
    - [trait：定義共享的行為](ch10-02-traits.md)
    - [生命週期與引用有效性](ch10-03-lifetime-syntax.md)

- [測試](ch11-00-testing.md)
    - [編寫測試](ch11-01-writing-tests.md)
    - [運行測試](ch11-02-running-tests.md)
    - [測試的組織結構](ch11-03-test-organization.md)

- [一個 I/O 項目](ch12-00-an-io-project.md)
    - [接受命令行參數](ch12-01-accepting-command-line-arguments.md)
    - [讀取文件](ch12-02-reading-a-file.md)
    - [增強錯誤處理和模組化](ch12-03-improving-error-handling-and-modularity.md)
    - [測試庫的功能](ch12-04-testing-the-librarys-functionality.md)
    - [處理環境變數](ch12-05-working-with-environment-variables.md)
    - [輸出到`stderr`而不是`stdout`](ch12-06-writing-to-stderr-instead-of-stdout.md)

## Rust 編程思想

- [Rust 中的函數式語言功能](ch13-00-functional-features.md)
    - [閉包](ch13-01-closures.md)
    - [迭代器](ch13-02-iterators.md)
    - [改進 I/O 項目](ch13-03-improving-our-io-project.md)
    - [性能](ch13-04-performance.md)

- [更多關於 Cargo 和 Crates.io](ch14-00-more-about-cargo.md)
    - [發佈配置](ch14-01-release-profiles.md)
    - [將 crate 發佈到 Crates.io](ch14-02-publishing-to-crates-io.md)
    - [Cargo 工作空間](ch14-03-cargo-workspaces.md)
    - [使用`cargo install`從 Crates.io 安裝文件](ch14-04-installing-binaries.md)
    - [Cargo 自定義擴展命令](ch14-05-extending-cargo.md)

- [智能指針](ch15-00-smart-pointers.md)
    - [`Box<T>`Box<T>在堆上存儲數據，並且可確定大小](ch15-01-box.md)
    - [`Deref` Trait 允許通過引用訪問數據](ch15-02-deref.md)
    - [`Drop` Trait 運行清理代碼](ch15-03-drop.md)
    - [`Rc<T>` 引用計數智能指針](ch15-04-rc.md)
    - [`RefCell<T>`和內部可變性模式](ch15-05-interior-mutability.md)
    - [引用循環和內存洩漏是安全的](ch15-06-reference-cycles.md)

- [無畏並發](ch16-00-concurrency.md)
    - [線程](ch16-01-threads.md)
    - [消息傳遞](ch16-02-message-passing.md)
    - [共享狀態](ch16-03-shared-state.md)
    - [可擴展的並發：`Sync`和`Send`](ch16-04-extensible-concurrency-sync-and-send.md)

- [物件導向](ch17-00-oop.md)	
	- [什麼是物件導向？](ch17-01-what-is-oo.md)
	- [為使用不同類型的值而設計的 trait 物件](ch17-02-trait-objects.md)
    - [物件導向設計模式的實現](ch17-03-oo-design-patterns.md)
	
## 高級主題

- [模式用來匹配值的結構](ch18-00-patterns.md)
    - [所有可能會用到模式的位置](ch18-01-all-the-places-for-patterns.md)
    - [refutable：何時模式可能會匹配失敗](ch18-02-refutability.md)
    - [模式的全部語法](ch18-03-pattern-syntax.md)

- [高級特徵](ch19-00-advanced-features.md)
    - [不安全的 Rust](ch19-01-unsafe-rust.md)
    - [高級生命週期](ch19-02-advanced-lifetimes.md)
    - [高級 trait](ch19-03-advanced-traits.md)
    - [高級類型](ch19-04-advanced-types.md)
    - [高級函數與閉包](ch19-05-advanced-functions-and-closures.md)

- [最後的項目: 構建多線程 web server](ch20-00-final-project-a-web-server.md)
    - [單線程 web server](ch20-01-single-threaded.md)
    - [慢請求如何影響吞吐率](ch20-02-slow-requests.md)
    - [設計線程池接口](ch20-03-designing-the-interface.md)
    - [創建線程池並儲存線程](ch20-04-storing-threads.md)
    - [使用通道向線程發送請求](ch20-05-sending-requests-via-channels.md)
    - [Graceful Shutdown 與清理](ch20-06-graceful-shutdown-and-cleanup.md)

- [附錄](appendix-00.md)
    - [A - 關鍵字](appendix-01-keywords.md)
    - [B - 運算符](appendix-02-operators.md)
    - [C - 可導出的 trait]()
    - [D - Rust 開發版]()
    - [E - 巨集]()
    - [F - 本書翻譯]()
    - [G - 最新功能](appendix-07-newest-features.md)