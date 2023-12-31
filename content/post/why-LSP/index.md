---
title: 为什么是 LSP ？
slug: why-lsp
date: 2023-9-22 22:04:00+0800
categories:
    - 翻译作品
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

LSP ([language server protocol](https://microsoft.github.io/language-server-protocol/)) 在今天十分流行。
至于为什么会出现这种情况，有一个标准解释。你可能见过这张图片：


![LSP-MxN](LSP-MxN.png)


我认为这种对 LSP 受欢迎程度的标准解释是错误的。在这篇文章中，我提出了一个替代图片。

# 标准解释
解释如下：


首先，有 `M` 个编辑器和 `N` 种语言。如果你想在特定的编辑器中支持特定的语言，需要为此编写一个专用的插件。
这意味着要做 `M * N` 的工作，正如图片的左边生动地展示的那样。
LSP 所做的是通过提供一个常见的细腰[^1]将其削减为 `M + N`，如图片右侧所示。

[^1]: 译注：细腰（thin waist）是一种对用于解决计算机系统中互操作问题的概念或是接口的比喻说法。

# 为什么这个解释错了？

我们最好用图片来说明这个解释的问题。简而言之，上面的图片没有按比例绘制。
这里有一个更好的说明，以 `rust-analyzer` + `VS Code` 的组合是如何协同工作为例：


![ra-code](ra-code.png)


左边的（大）球是 rust-analyzer —— 一个语言服务器。
右边同样大小的球是 `VS Code` —— 一个编辑器。
中心的小球是将它们粘合在一起的代码，*包括* LSP 实现。


这部分代码相较而言非常小。语言服务器或编辑器背后的代码库则十分庞大。


如果标准理论正确，那么，在 LSP 出现之前，我们已经生活在一个某些语言在某些编辑器中具有极好的 IDE 支持的世界里。
例如，IntelliJ 之于 Java、Emacs 之于 C++、Vim 之于 C# 等等。
但我对那个时代的记忆截然不同。要想获得像样的 IDE 支持，要么使用 JetBrains 家的 IDE（IntelliJ 或 ReSharper） 所支持的语言，要么另寻他路。


只有一个编辑器可以提供有意义的语义化 IDE 支持。

# 另一种说法
我想说，过去 IDE 支持如此糟糕的原因并非如此。
与其说 `M*N` 太大，不如说它太小，因为 `N` 是零，`M` 只是略大于零。


我会先从 `N` 开始 —— 语言服务器的数量，这是我相对熟悉的方面。
在 LSP 出现之前，根本没有太多像语言服务器一样工作的东西。主要原因是构建一个语言服务器很困难。


语言服务器本身的复杂性相当高。众所周知，编译器很复杂，而语言服务器则是编译器*加上点其他东西*。


*首先*，就像编译器一样，语言服务器需要完全理解语言本身，它需要能够区分有效程序和无效程序。
然而，对于无效程序，编译器通常只是发出错误消息并立即退出，但语言服务器必须尽可能地分析*任何*无效程序。
与编译器相比，处理不完整和无效的程序是语言服务器遇到的第一个复杂问题。


*其次*，虽然编译器通常只是一个将源文本转换为机器代码的单纯程序，但语言服务器必须与用户不断修改的代码库协作。
它是一个具有时间维度的编译器，而处理状态随时间的演变是编程中最困难的问题之一。


*第三*，编译器通常是为最大化吞吐量而优化的，而语言服务器的目标却是最小化延迟（同时不完全放弃吞吐量）。
添加对延迟的要求并不意味着您需要更加努力地进行优化。
相反，这通常意味着您需要彻底改变体系结构，以把延迟控制在可接受的范围内。


而这将我们带入了与语言服务器相关的意外复杂度集群中。众所周知，如何编写编译器是个常识。
并非所有人都曾阅读过“龙书”（我没有认真阅读过解析相关章节），但人人皆知其中包含着一切答案。
因此，大多数现有编译器最终看起来都像一个典型的编译器。
而且，当编译器作者开始考虑 IDE 支持时，第一个想法可能是“嗯，IDE 有点像编译器，我们有编译器，所以问题解决了，对吧？”。
这种想法完全错误 —— IDE 内部与编译器非常不同，可是这个事实直到最近还鲜为人知。


语言服务器是[“从不重写”](https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/)规则的反例。
大多数备受好评的语言服务器都是编译器的重写或替代实现。
IntelliJ 和 Eclipse 都编写了自己的编译器，而不是在 IDE 中重用 javac 。
为了给 C# 提供足够 IDE 支持，微软将他们的 C# 编译器重写为交互式自托管编译器（Roslyn 项目）。
尽管 Dart 是一种从头开始、相对现代的语言，但最终却有*三种*实现（主机 AOT 编译器、主机 IDE 编译器（dart-analyzer）和在设备上的 JIT 编译器）。
Rust 从两个方向展开探索 —— rustc 的增量演变（RLS）和从头开始的实现（rust-analyzer），最终 rust-analyzer 大获全胜。

我知道的两个例外是 C++ 和 OCaml。奇怪的是，两者都需要前向声明和头文件，我不认为这是巧合。
有关详细信息，请参阅 [响应式 IDE 的三种体系结构](https://rust-analyzer.github.io/blog/2020/07/20/three-architectures-for-responsive-ide.html) 一文。


总之，在语言服务器方面，事情处于一个糟糕的平衡状态。
实现语言服务器完全可能，但需要些打破传统的方法，而想要做到这一点十分困难。


我不太确定编辑器方面发生了什么。尽管如此，我还是要声明一点，我们没有能适任 IDE 的编辑器。


IDE 包括许多语义方面的功能。最典型的例子当然是补全。
如果要实现 `VS Code` 的自定义补全，需要实现 [CompletionItemProvider](https://code.visualstudio.com/api/references/vscode-api#CompletionItemProvider) 接口：
```typescript
interface CompletionItemProvider {
    provideCompletionItems(
        document: TextDocument,
        position: Position,
    ): CompletionItem[]
}
```


这意味着，在 `VS Code` 中，代码补全（以及数十个其他与 IDE 相关功能）是编辑器一等概念，具有统一的用户 UI 和开发者 API 。


这与 Emacs 和 Vim 形成鲜明对比，因为它们没有合适的补全接口用于编辑器扩展。
相反，他们暴露了低级的光标和屏幕操作 API，然后人们在此基础上实现了相互冲突的补全框架！


这还只是代码补全！还有参数信息、嵌入提示、路径导航[^2]、扩展选择、辅助、符号搜索、查找用法（给我打住:)）要怎么办？
[^2]: https://zh.wikipedia.org/wiki/%E9%9D%A2%E5%8C%85%E5%B1%91%E5%AF%BC%E8%88%AA

简而言之，体面的 IDE 支持问题不在于 `N*M` ，而在于双边市场不充分平衡。


语言提供者不愿意创建语言服务器，因为这本身很难，需求又很低（等同于没有来自其他语言的竞争）。而且，即使创建了一个语言服务器，也会发现十几个编辑器完全没有准备好为智能服务器提供支持。


从编辑器的角度来看，想要添加 IDE 支持所需的高级 API 的动机很小，因为这些 API 没有潜在提供者。

# 为什么 LSP 很棒
这就是为什么我认为 LSP 很棒！


我不认为这是一个很大的技术创新（显然，您会想要将一个与语言无关的编辑器和一个特定于语言的服务器分开）。
我认为这是一个相当糟糕（又名“足够好”）的技术实现（请继续关注“为什么 LSP 很烂？”帖子？）。
*但是*它把我们从一个语言没有对应的 IDE 很正常，甚至没有人会考虑语言服务器的世界，带到了一个如果语言没有可用的补全和转到定义的就看起来不专业的世界。


值得注意的是，微软解决了双边市场问题，他们是两种语言（C# 和 TypeScript）和编辑器（VS Code 和 Visual Studio）的供应商，不过IDE领域通常输给了竞争对手（JetBrains）。
虽然我可能会对 LSP 的特定技术细节吹毛求疵 ，但我绝对钦佩他们在这一特定领域的战略眼光。

他们：
- 建立了一个基于 web 技术的编辑器。
- 将 web 开发支持确定为与 JetBrains 竞争的一大优势（在 IDE 中支持 JS 几乎是不可能的）。
- 构建了一种语言（！！！），使为 web 开发提供 IDE 支持变得可行。
- 构建了一个具有前瞻性架构的 IDE 平台（请继续关注我解释为什么 vscode.d.ts 是卓越的技术杰作的帖子）。
- 推出 LSP 以免费增加其平台在其他领域的价值（将整个世界推向一个明显更好的 IDE 平衡作为附带利益）。
- 现在，如果我们真的不再在本地机器上编辑、构建和运行代码，code spaces 将成为“远程优先开发”的主导者。


不过，说实话，我仍然希望最终的赢家是 JetBrains，因为他们将 Kotlin 作为适用于任何平台的通用语言:-)
虽然微软充分利用了当今占主导地位的技术（TypeScript 和 Electron），但 JetBrains 试图自下而上地解决问题（Kotlin 和 Compose）。

# 关于 `M * N` 的更多信息
现在我要强调的是，它*真的*不是 `M*N`:）

*首先*，`M*N` 的论点忽略了这样一个事实，即这是一个令人尴尬的平行问题。
语言设计者不需要为所有编辑器编写插件，编辑器也不需要添加对所有语言的特殊支持。
相反，一种语言应该实现一个使用某种协议的服务器，编辑器需要实现语言无关的 API 来提供补全等功能，如果语言和编辑器都并不深奥，那么对两者都感兴趣的人只需要写一点胶水代码就可以将两者绑定在一起！
rust analyzer 的 VS Code 插件有 3.2k 行代码，neovim 插件是 2.3k 行，Emacs 插件则是 1.2k 行。这三个插件都是由不同的人独立开发的。
去中心化与开源化的开发极具魅力！如果插件要支持自定义协议而不是 LSP（前提是编辑器内部支持高级 IDE API），我预期只需在上面数据的基础上添加 2k 行，而这仍然在业余爱好者的闲暇时间范围内。


*其次*，对于 `M*N` 优化，您应该希望协议实现是从一些机器可读的实现中生成的。
但在最新版本之前，LSP 规范的真实来源是一个非正式的 markdown 文档。
每种语言和客户端都在想出自己的方法来从中提取协议，许多人（包括 rust-analyzer）只是手动同步更改，其中有相当多的重复。


*第三*，如果 `M*N` 是一个问题，那么每个编辑器应该只能看到一个 LSP 实现。但事实上，LSP 在 Emacs 上有两种相互竞争的实现（lsp-mode 和 eglot）。
我不骗你，rust analyzer 手册中包含了与 vim 的 6 个不同 LSP 客户端集成的说明。
呼应第一点，这是开源的！工作量的总量几乎无关紧要，重要的是完成任务的协调程度。


*第四*，微软本身并没有试图利用 `M+N` 。`VS Code`中*没有*通用的 LSP 实现。
相反，每种语言都需要有一个具有完全独立 LSP 实现的专用插件。

# 我呼吁
* 每个人

请索求更好的 IDE 支持！
我认为今天我们已经跨过了 IDE 支持的一般可用性的门槛，但除了基本功能之外，我们还可以做很多事情。
在理想的情况下，应该可以使用今天可以用于检查编辑器缓冲区内容的相同简单的 API 来检查光标处表达式的每个小语义细节。

* 文本编辑器作者

留意 `VS Code` 的架构。虽然 electron  提供的用户体验值得商榷，但其内部架构中却蕴含有许多智慧。编辑器的 API 着眼于与显示方式无关的高级特性。
基本的 IDE 功能应该是一流的扩展点，它不应该被每个插件的作者重新发明。特别是，[辅助/代码操作/💡](https://rust-analyzer.github.io/blog/2020/09/28/how-to-make-a-light-bulb.html) 已经成为一流的用户体验的概念。
这是 IDE 唯一也是最重要的在改善用户体验上的创新。尽管已经出现很久了，但这并不是所有编辑器的标准接口，我感到非常荒谬。


但不要把 LSP *本身*作为一流的概念。
令人惊讶的是，VSCode 对 LSP *一无所知*。它只提供了一系列扩展点，而丝毫不关心它们是如何实现的。
LSP 实现只是一个库，由特定于语言的插件使用。例如，`VS Code` 的 Rust 和 C++ 扩展在运行时不共享相同的 LSP 实现，使得内存中有两个 LSP 库的副本！


此外，尝试利用开源的力量。
不要强制集中所有 LSP 实现！让不同的人可以独立地为您的编辑器提供完美的 Go 支持和完美的 Rust 支持。`VS Code` 是一个可能的模范，它具有市场和分布式、独立的插件。
不过，只要语言能有独立的维护者，将工作组织成一个共享的 仓库/源代码 树或许是可行的。

* 语言服务器作者

你们做得很好！所有语言的 IDE 支持质量都在迅速提高，尽管我觉得这不过是一条漫长道路的开端。
需要记住一点，LSP 是*一个*关于语言语义信息的接口，但它并不*唯一*，也许会有更好的东西出现。
即使在今天，LSP 的局限性也阻碍了很多实用功能推出。因此，请尝试将 LSP 视为序列化格式，而不是内部数据模型。
同时，尝试写更多关于如何实现语言服务器的内容————我觉得这方面的知识依然不够。


以上！

附言：如果您有机会从使用 rust-analyzer 中受益，请考虑赞助[Ferrous Systems Open Source Collective for rust analyzer](https://opencollective.com/rust-analyzer)，以支持其开发！

# 翻译信息
本文翻译自 (https://matklad.github.io/2022/04/25/why-lsp.html#Why-LSP)