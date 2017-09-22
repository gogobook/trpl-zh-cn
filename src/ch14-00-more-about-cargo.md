# 進一步認識 Cargo 和 Crates.io

> [ch14-00-more-about-cargo.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch14-00-more-about-cargo.md)
> <br>
> commit db6129a30d7c7baed34dd38dbc56f7ed8a66ae92

目前為止我們只使用過 Cargo 構建、運行和測試代碼的最基本功能，不過它還可以做到更多。這裡我們將瞭解一些 Cargo 其他更高級的功能，他們將展示如何：

* 使用發佈配置來自定義構建
* 將庫發佈到 crates.io
* 使用工作空間來組織更大的項目
* 從 crates.io 安裝二進制文件
* 使用自定義的命令來擴展 Cargo

相比本章能夠涉及的工作 Cargo 甚至還可以做到更多，關於其功能的全部解釋，請查看[文檔](http://doc.rust-lang.org/cargo/)