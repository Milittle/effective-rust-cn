# Introduction
```
"The code is more what you'd call 'guidelines' than actual rules." – Hector Barbossa
```

斯科特·迈耶斯的《Effective C++》因为引入了一种新式的编程书籍风格而取得非凡的成功，这种风格专注于一系列从C++软件开发实际经验中学到的指导原则的集合。值得注意的是，这些指导原则不仅解释它们存在的必要性而且还允许读者自己决定他们特定的场景是否需要打破这些指导规则。

《Effective C++》的第一版出版于1992年，那时的C++语言虽然年轻，但已经是一种包含许多潜在问题的微妙语言；拥有一个指导其不同特性相互作用的指南是必不可少的。

相比之下，Rust也是一门年轻的语言，但它几乎不含潜在的问题。其类型系统的强度一致性意味着，如果一个Rust程序编译通过了，它就已经有相当大的可能性会正常工作——这种现象以前只在更学术性、不那么容易感知到的语言中观察到，比如Haskell。

Rust安全性——包括类型安全和内存安全 —— 虽然有代价。Rust因其学习曲线陡峭而闻名，入门者必须经历与借用检查器斗争、重新设计数据结构以及对生命周期感到困惑的入门仪式。一个编译通过的Rust程序可能很有可能直接工作，但让它编译通过内心所遭受的斗争是真实的——即使Rust编译器提供的错误诊断非常有帮助。

因此，这本书的目标读者与其他《Effective <Language>》系列书籍略有不同；这里有更多的项目涵盖了Rust的新概念，尽管官方文档已经包含了这些主题的良好介绍。这些项目的标题如“理解……”和“熟悉……”。

Rust的安全性也导致了完全没有标题为“永不……”的项目。如果你真的不应该做某事，编译器通常会阻止你这么做。（译者备注：这就是Rust编译器的自信，不该干的，不会让你干）

尽管如此，文本仍然假定读者对语言的基础有所了解。它还假设使用的是2018版的Rust，并使用稳定的工具链。

用于代码片段和错误消息的具体rustc版本是1.60。Rust现在已经足够稳定（并有足够的向后兼容保证），代码片段不太可能需要为后续版本做出更改，但错误消息可能会因你的特定编译器版本而有所不同。

文本还有许多参考和与C++的比较，因为这可能是最接近的等效语言（特别是具有C++11的移动语义），也是Rust新手最可能遇到的前一种语言。

所有的项目被分为如下六个片段：
* Types(类型)：围绕Rust核心系统的建议
* Concepts(概念)：Rust设计的核心概念
* Dependencies(依赖)：关于如何使用Rust的包生态系统的建议（更多的是对cargo的理解和使用）
* Tools(工具)：关于如何通过不仅仅是使用Rust编译器来改进代码库的建议。
* Asynchronous Rust(异步Rust)：和Rust async机制的工作方式的建议
* Beyond Standard Rust(额外的Rust)：当你需要在Rust的标准、安全环境之外工作时的建议。（更多的是unsafe的一些建议）

虽然“概念”部分可以说比“类型”部分更为基础，但它故意放在第二位，以便从头到尾阅读的读者可以先建立一些信心。（其实我们懂作者这句话，从我的理解，应该是第二部分也给一点简单的东西，好让有继续读下去的信心）