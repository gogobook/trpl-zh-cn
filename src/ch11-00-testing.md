# 测试

> [ch11-00-testing.md](https://github.com/rust-lang/book/blob/master/second-edition/src/ch11-00-testing.md)
> <br>
> commit 1047433147b27d19e5acc068a0ebca5782d64f99

> Program testing can be a very effective way to show the presence of bugs, but it is hopelessly inadequate for showing their absence.
>
> Edsger W. Dijkstra, "The Humble Programmer" (1972)
>
> 软件测试是证明 bug 存在的有效方法，而证明它们不存在时则显得令人绝望的不足。
>
> Edsger W. Dijkstra，【谦卑的程序员】（1972）

程序的正确性意味着代码如我们期望的那样运行。Rust 是一个非常注重正确性的编程语言，不过正确性是一个难以证明的复杂主题。Rust 的类型系统在此问题上下了很大的功夫，不过它不可能捕获所有种类的错误。为此，Rust 也在语言本身包含了编写软件测试的支持。

例如，我们可以编写一个叫做 `add_two` 的将传递给它的值加二的函数。它的签名有一个整型参数并返回一个整型值。当实现和编译这个函数时，Rust 会进行所有目前我们已经见过的的类型检查和借用检查，例如，这些检查会确保我们不会传递 `String` 或无效的引用给这个函数。Rust 所 **不能** 检查的是这个函数是否会准确的完成我们期望的工作：返回参数加二后的值，而不是比如说参数加 10 或减 50 的值！这也就是测试出场的地方。

我们可以编写测试断言，比如说，当传递 `3` 给 `add_two` 函数时，应该得到 `5`。当对代码进行修改时可以运行测试来确保任何现存的正确行为没有被改变。

测试是一项复杂的技能，而且我们也不能期望在一本书的一个章节中就涉及到编写好的测试的所有内容，所以这里仅仅讨论 Rust 测试功能的机制。我们会讲到编写测试时会用到的注解和宏，Rust 提供用来运行测试的默认行为和选项，以及如何将测试组织成单元测试和集成测试。