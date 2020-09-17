# 标题：So you want to write a package manager

原文：https://medium.com/@sdboyer/so-you-want-to-write-a-package-manager-4ae9c17d9527

作者：[sam boyer](https://twitter.com/sdboyer)

翻译时间：2020-08-02

# 译注

本文作者 [sam boyer](https://twitter.com/sdboyer) 是 Go 的包管理器之一 [dep](https://github.com/golang/dep) 的 [主要作者](https://github.com/golang/dep/graphs/contributors)，dep 诞生于官方 Go Modules 机制之前，曾是最可能成为 Go 官方的包管理器工具。其中的争论可见 [这里](https://zhuanlan.zhihu.com/p/41627929)。

不过后来 Golang 主导者 [Russ Cox](https://twitter.com/_rsc?lang=en) 也在 Go Module  [相关文章](https://github.com/vikyd/note/blob/master/go_and_versioning/go_add_package_versioning.md#%E4%B8%8B%E4%B8%80%E6%AD%A5) 中感谢了 sam boyer。

本文写于 Go Module 机制正式发布之前，所以现在看来有些观点已经过时，但其关于包管理器的公共需求是值得了解的，也是翻译本文的主要原因。

> 本文图片直接引用于原文，查看可能需自带梯子


# 标题：如果你想写一个包管理器

原文发表时间：2016-02-12


# 目录

[TOC]


# 正文
你今天早上醒来，蹦下床，想到："你知道吗？我这一生太顺了，没体验过什么痛苦坎坷滋味。啊，我想到了！我可以去写个编程语言的包管理器来体验下人生的痛苦。"。

我也是！和你一样，正体验着这种痛苦遭遇。所以写下这篇所谓的 "包管理器设计指南"，希望能减缓你的发际线后移。

谁适合看本文：

- 想体验人生痛苦的你
- 希望优化现有包管理器的人们
- 好奇包管理器运作机制的人们
- 创建了新编程语言，但发现若无包管理器，新语言将一无是处，的人们

现在，我确实有一个实际的动机：目前 Go 语言社区真的需要一个合适的包管理器，而我正好在开发其中 [一个包管理器（dep）](https://github.com/golang/dep)。因此，我会以 Go 语言作为主要例子，文末也有 Go 语言相关的章节。

本文真正的目的：探讨通用包管理器的设计原则、领域限制，以及如何应用到不同的语言中。


# 包管理器是个大坑，你最好别踩
依赖包管理是一个不讨好的领域，真的不讨好。从表面看，这好像只是一个纯粹的技术问题，应采用纯粹的技术解决方案。于是，人们理所当然的以此方式对待包管理器。随着时间流逝，他们却得出以下无情的结论：

1. 软件是个可怕的东西
2. 人类也同样可怕
3. 有太多不确定的事物
4. 没任何事情能真正确定可运行
5. 可以证明：没任何事情能真正确定可运行
6. 我们的生活处于混乱与熵的漩涡中，毫无意义

如果你曾编写过任何软件，你可能会觉得上述顿悟似曾熟悉。这种死循环的经验告诉我们，这是一个信号，提醒我们应回过头重新评估我们的预期和假设。通常，事实会证明你在开始时并没有正确分析问题。又一个天气晴朗的好日子，事情又来了：那些认为依赖包管理是纯粹技术问题的人又理所当然或潜意识地寻找一些可以完整、正确地自动升级依赖包的方案。

请不要这样做，不然，你可能会很难受。

依赖包管理的结果是与人耦合的：
- 与他们所知道的知识相关
- 与他们的实际行动相关
- 与他们能否合理负责该事情相关

这些与人相关的东西，和计算机如何能正确识别源代码或其他东西一样重要。两边都是必需的，缺一不可。


呃，这样一想，好像我已实现自我超越了。



# 本文所说的 "包管理器"
我相信你知道如何上网，所以我猜你已经第一时间看过了维基百科中的 [前面两句话](https://en.wikipedia.org/wiki/Package_manager)。你现在已经是一个砖家了哈。所以你应已知这里讨论哪种包管理器了。否则，就会变成：A 君说 `嗨，传个球给我`，你说 `没问题`，然后把弓箭递了给我（译注：我不是 A 君呀），然而我想要的是一个条丝带，而此时 B 君跑去拿她的大提琴了（译注：乱套了）。请看以下定义：

- **操作系统的包管理器**（OS/system package manager，简称 **SPM**）：
  - 本文不讨论这个
- **编程语言的包管理器**（Language package manager，简称 **LPM**）：
  - 通常是一个交互式工具（如 Golang 的 `go get`），可以获取特定编程语言中特定的依赖包源码，并对其进行构建。有些方案做得不好：将获取到的源码到一个没注明版本号的全局位置（`GOPATH`，没错，说的就是你），还不知羞耻的告诉你这样做具有重要的意义。
- **项目/应用程序内的包管理器**（Project/application dependency manager，简称 **PDM**）：
  - 一个交互式的系统，用于管理某编程语言中一个项目的依赖包源码。这意味着包括：描述、获取、升级、按约定结构存放到磁盘、删除部分依赖包的源码等功能，提供一条龙服务，而非提供一个单一的命令就完事。PDM 的输出是可精确复现的，此输出应是一个自包含的源码树，并作为编译器或解释器的输入。或者你可将此理解为 [编译器的零级阶段](https://www.tutorialspoint.com/compiler_design/compiler_design_phases_of_compiler.htm)。

（译注：从使用者是谁的角度看）这三者的主要区别包括，有些是给用户安装软件使用的，有些是给开发者创建新软件用的。
- SPM 是供用户使用的系统
- PDM 是供开发者使用的系统
- LPM 通常兼具两者的功能（译注：可想象基于 `GOPATH` 模式的 `go get`，既下载源码可供开发者使用，又会编译到 `GOPATH/bin` 可供用户使用），但若 LPM 的背后没有 PDM 的支撑，天使也可能变恶魔。

另一方面，PDM 可在缺失 LPM 时活得不错，不过实际上通常会将两者捆绑在一起。但是，这种捆绑容易让旁观者搞混 LPM 和 PDM，甚至会完全忽略后者。

请不要这样做，不然，你可能会很难受。

PDM 几乎是整个技术栈的最底层。因为它可组成更上层的组件（而且 Go 语言现在迫切需要这种功能），这是本文的重点。不过好在可以轻松梳理出 PDM 的职责。此工具必须正确、容易、快速地完成以下步骤：

1. 开发者在开发过程中，从无数各式各样已存在的软件中，凭感觉挑出来其中一些 **认为** 可作为项目直接依赖的软件包。然后
2. 将这些直接依赖包转换为精确的、可递归遍历的依赖包版本列表（译注：间接依赖包之间可能会存在冲突）。这样一来，不管是何人用哪台机器，都可以
3. 从上述依赖包列表中创建或 **复现** 出源码树，从而
4. 得到一个独立的、自包含的项目 + 所有依赖包源码，最终作为编译器或解释器的输入

还记得前面说过，在依赖包管理这件事里，人与计算机同样重要吗？上述步骤中有 2 个加粗的词：**认为**、**复现**，分别代表了人和计算机在这些步骤中的作用。通过确定的算法来得到精确的输出结果，这是很自然的想法，而又因为人类在开发过程中的不确定性，导致了矛盾。这种不可避免的矛盾需要解决方案。如何提供这种方案，正是 PDM 要做的事。


# 我们碰到敌人了，敌人却是我们自己
PDM 的困难不在于算法方面。（译注：PDM 的）算法输出属于编译器或解释器的零级阶段，并且尽管其具体内容因不同编程语言而异，但每个算法依然能提供明确定义的目标。正如在计算领域的很多其他问题一样，真正的挑战在于如何以适合人类思维模型的方式向机器提出需求。

现在，我们讨论的是 PDM，也即表示，我们将会讨论与 FLOSS 世界的交互（译注：[FLOSS：Free/Libre and Open-Source Software](https://en.wikipedia.org/wiki/Free_and_open-source_software)，即开源世界）：从中拉取源码，或发布源码。（实际上，不止是 FLOSS，各公司内部的代码生态也一样。但为了方便，这里统称 FLOSS）我想说的是，与 FLOSS 交织在一起的思维模型大致是这样的：

- 我有一些正在编写、更新的软件，可称之为 "项目（project）"。当我在项目中工作时，此项目就是我在 FLOSS 世界中的家，其他因素的考虑都会围绕此家进行。
- 我知道应在项目中做什么，但必须承认，我对项目的理解也不很完整。
- 我知道我/我的团队需对项目最终负责，无论上游发生任何问题，都要确保我们创建的项目能达到预期。
- 我知道若我不引用任何依赖包，全部代码自己编写，将会耗费我更多时间，并且肯定难以比久经历练的外部依赖包更可靠。
- 我知道使用别人的代码肯定会对其产生依赖，至少会产生一些了解成本、维护整理成本。
- 我知道我难以彻底了解外部依赖包。
- 我不知道外部总共有多少依赖包，但肯定比我知道的多得多。其中有些可能与我相关，但绝大部分都与我无关，所以我需要一些搜索、评估成本才能得到想要的依赖包。
- 我已做好心理准备，外部的依赖包很多时候有不少坑，至少  [我是曾经被伤害过的](http://www.zazzle.com/hell_is_other_peoples_code_t_shirts-235847475911306715)。吃一堑长一智都需要一定的时间成本。

这里提到的点包括：时间、风险、不确定性。开发软件时，有太多未知的方向：

- 时间有限，你不可能看遍所有的依赖包
- 若使用了有坑的依赖包可能还会导致项目失败
- 不同项目的不确定性可能有多有少，但不可能被彻底消除

一直以来，这些都是很自然就能想到的限制。实际上，对于开源生态来说，有些要点是必需的。所以我们将其写成正式条款了（译注：为了免责）：

```
本软件是 "按原样" 提供的，不附带任何明示或暗示的保证，包括没有任何有关适销性、适用性、非侵权性的保证以及其他保证。
在任何情况下，作者或版权持有人，对任何权益追索、损害赔偿以及其他追责，都不负任何责任。
无论这些追责产生自合同、侵权，还是直接或间接来自于本软件以及与本软件使用或经营有关的情形。
```

上述是我摘自 [MIT 协议](https://opensource.org/licenses/MIT) 的部分。（译注：上述条款翻译来自 [这里](https://zhuanlan.zhihu.com/p/137297074)）

而且你也知道其中可能的风险问题，条文里也说得很清楚，但基本没人真正去读一下这些条文。

尽管我们明知编写软件时会面临很多未知的危险，但我们依然会去使用外部的依赖包。所以，问题就变成了：我们如何能在各种不确定性的世界中编写软件，且同时我们的项目（或说软件健壮性）所受到的风险影响更小？

我喜欢将此与公共卫生领域进行类比来寻找解决方案：减少伤害（harm reduction）。不久前，[Julia Evans](https://medium.com/u/152f65dab15?source=post_page-----4ae9c17d9527----------------------) 写了一篇名为 [减少对开发者的伤害（Harm Reduction for Developers）](https://jvns.ca/blog/2014/11/18/harm-reduction-for-developers/) 的好文。文章虽简短，但值得一读。我在这里划一下重点：

```
人们将要做一些有风险的事情时，我们应助其降低风险，而非让他别做。
```

这是我们为开发者创建一个实用工具时必须采取的思路。不是因为我们要愤世嫉俗地迎合那些最小公分母的爱冒险开发者。而是，呃，请看看那些充满不确定性的航天火箭发射塔！敢于冒险是人类发展的必要因素。PDM 应是这样一个工具：鼓励开发者通过广泛的实验来降低不确定性，但同时又尽可能保持项目稳定和合理发展。这就像火箭与火箭发射台之间的关系。

按以往习惯，我可能准备开始聊具体的案例了。但这次不着重列举具体案例，所以难免会在黑暗中摸索，甚至得出一些令人失望的、奇奇怪怪的、由委员会设计的折中方案。不管怎样，对于 PDM 工具，我还是认为能有更优的方案。这次，我不会从个例的角度探讨解决方案，而会从分布式系统的方向大致聊下，并将方案抽象到一些状态和协议中。



# 状态与协议
重点关注状态是有重要原因的。了解有哪些状态、状态何时会改变、为什么会改变、如何改变，都有助于让软件更稳定，远离混乱和熵的漩涡。如果我们可以约定一些状态和交互协议的通用规则，就可更容易直捣黄龙（之后，才列举具体案例）。话又说回来，即使是本文，也来来回回修订了好几遍，所以状态与协议的通用规则也不会一锤定音，会根据实际修订多次。


## 角色
在磁盘中，PDM 会在 4 种互相独立的状态中变换。根据不同的 PDM，这些状态的输入、输出各不相同：

![PDM 在磁盘的 4 种状态](https://miro.medium.com/max/611/1*jRJVOQnIOzEvDjuGd0o2OA.png)

> 磁盘上的状态对 PDM 来说很重要

- **项目代码（Project Code）**：是指正活跃开发的源代码，我们想通过 PDM 来管理其依赖包。现在已不是 1967 年那个古时候了，现在所有的项目代码都会有版本控制（译注：如 Git）。对于大部分 PDM 来说，项目的源码是一个仓库的所有代码，也有项目代码只是仓库中的一个子目录。
- **清单文件（Manifest file）**：是一个人类可读写的文件，不过通常也会有软件工具协助，用于列明要管理的依赖包。清单文件通常也会包含项目的其他元数据信息。有时还会包含 PDM 所绑定的 LPM 的指引。（清单文件是必须提交到仓库的，否则项目跑不起来）
- **锁文件（Lock file）**：一个由软件自动写入的文件，记录了所有可复现完整依赖包源码树的必要信息。通过遍历分析清单文件中所有的依赖项，最终凝固成本文件，里面都是不再变化的版本号。（本文件也必须提交到仓库。有机会的话后面再展开讲。）
- **依赖包代码（Dependency code）**（译注：后译为 **lock 文件**）：指 lock 文件中具体依赖包版本所对应的所有源码。存放在磁盘中，将作为编译器/解释器的输入。不会因为其他原因被修改。有时还可能会包含某些构建环境下的补充逻辑，如一些自动加载器。（这些依赖包代码无需提交到仓库）

不过，不是每个 PDM 都有上述几种状态。有些 PDM 虽然有这几种状态，但其实现方式有所不同。大部分新出现的 PDM 都会包含上述几种状态，但并没有完全达成共识。我想说的是，每个 PDM 都需要这 4 种状态才算功能齐全，并且这些状态互相分离才是最佳。


## 管道中的管道
约定了状态只是事情的一半，另一半是约定协议：状态之间该如何变换。好在协议比较直观。

![](https://miro.medium.com/max/683/1*_6xHKQxpQkVahkXXzEClsA.png)

译注：下面 4 点分别对应上图黄色背景文字：
1. 每天定时任务，或手动启动的任务
2. 通过代码静态分析生成或验证得到清单文件；通过工具的操作命令得到；通过用户手动编辑得到
3. 通过遍历分析清单文件得到
4. 通过遍历 lock 文件得到

状态之间大致形成了一条管道，其中每个阶段的输入都是其前面的状态（通常第 1 个转换比较混乱）。很难说这种线性的模型能为真正实现 PDM 的合理性带来多少正面影响，但不管怎样，它至少为 PDM 的逻辑提供了一个清晰的 "前进" 方向，消除了很多歧义的地方。更少的歧义，意味着使用工具时更简单、安全。

PDM 可被认为是更大管道中的一个组成部分。首先是编译器/解释器管道，这就是我之前将其称为 "编译器的零阶段" 的原因：

![编译器/编译器无需关心 PDM 的存在。其输入都是源码而已。](https://miro.medium.com/max/1000/1*V2xxqes9qQOUdLwmHA0FqA.png)

对于预编译的语言来说，PDM 的角色有点像预处理器：PDM 的最终结果是：项目本身代码 + 依赖包代码。当项目源码中看到类似 `include code X` 时，PDM 会对 `X` 分析，并获取项目作者真正想要的依赖包 `X` 的源码。

尽管机制有些不同，但 "零级" 的概念也适用于动态或 JIT 编译型的语言。除了在磁盘上存放代码外，PDM 通常还会覆盖或分析解析器的代码加载机制，以便于正确进行依赖分析。从这个角度看，PDM 正在构造一个供其自己使用的文件目录结构。可能有些吹毛求疵的人会说，这岂不是自我引用了。不过，真正重要的是，PDM 在解析器启动之前就应已准备好所需的文件结构了。所以说 PDM 依然是 "零级" 的。

另一个包含 PDM 的管道是版本控制。版本控制与编译器管道是完全正交的，如下图那样正交：

![源代码（以及随后 PDM 的动作）处于 X 轴。由版本控制系统管理的提交历史，处于 Y 轴](https://miro.medium.com/max/1000/1*6mQ-hX83Ow6EputaB5if5w.jpeg)

> 源码（以及 PDM 随后的操作）处于 X 轴。由版本控制系统组织的 commit 历史处于 Y 轴

注意到没？PDM 沿 X 轴向编译器提供代码，而版本控制则沿 Y 轴提供时间序列。这两者是正交的！

等等，X 轴上的是 PDM，Y 轴上的是时间。这好像与我们 [描述时空](https://en.wikipedia.org/wiki/Spacetime) 的欧几里得空间不谋而合。这是否意味着 PDM 是某种特别的拓扑流形函数？编译器的输出实质是一个时空连续体！？可能是吧！我不太清楚，我数学一般般。不管怎样，我们都得到了空间和时间，所以我称之为：每个代码仓库都有自己的小世界（或小宇宙）。

尽管此比喻可能不恰当，但依然有启发意义。每个项目仓库小世界都有自己的逻辑规则，所有规则都沿着各自独立的时间表演进。我虽然不知道如何以合理、共存的方式进行编排实际宇宙，但至少是 "相当困难"。因此，当我说 PDM 是对代码宇宙进行编排的工具时，意思是这项工作是极度有挑战的。只对代码进行 1 次编排是不够的，你还必须编排时间线。只有这样，代码宇宙才能共同成长和改变。

"编排宇宙" 后面会很有用，但 PDM 管道自身里头还有另一个需要注意的点。虽然 PDM 确实有 4 个离散的物理状态，但实际上只有 2 个概念起作用：

![项目代码、清单文件本质是用户的意图。lock 文件与依赖包源代码本质是 PDM 对用户意图的猜测尝试](https://miro.medium.com/max/679/1*RM550AcnpqSXolUYORKVxg.png)

> 项目代码与清单文件表达用户的意图。lock 文件和依赖包源码是 PDM 分析用户意图得到的结果

清单文件与项目代码，其组合方式可能不一样，但都是表达用户的意图。这说明有些事情用户不应该干预：不要手动搞乱 lock 文件或依赖包代码，同样的，也不要 [修改生成的代码](http://stackoverflow.com/a/740603/2498698)。相反，PDM 在生成 lock 文件和依赖代码树时也不应修改左侧的东西。一句话就是：人类管左边，机器管右边。

这样看来很美好很简洁。但我们来看看实际如何：

![](https://miro.medium.com/max/679/1*VBXNMeOORcgzfMMd7qufQQ.png)

> 别不信这是真的

对于很多软件来说，"一团糟" 可能是善意导致的。这方向被带偏了，导致人们希望在工具中删除（或至少不创建）那些鼓励导致混乱的功能。但请记住我们的目标：减少伤害。PDM 的职责不是阻止开发者去冒险，而应是让风险行为尽量安全可控。风险肯定会导致混乱，至少会有短暂的混乱，但这正是我们的事业，应接受这个事实。

可能 PDM 最容易搞错的方向是过于侧着某一边而伤害了另一边。PDM 必须保持平衡。若过于侧重右边，就会导致笨重而又紧张的混乱，就像 git 的子模块功能，开发者几乎都不用此功能。若过于侧重左边，就会误以为东西 "越简单" 生产效率越高，导致在不牢固的基础上进行的建设变成巨大的技术债务，这样的方式迟早（很快）会被淘汰。

请不要那样做，否则大家都很痛苦。


# 构造清单文件
![](https://miro.medium.com/max/598/1*pzAufPNA_YVNkenxHbzt6g.png)

> 也称为 "常规开发模式"

清单文件通常包含与特定语言相关的很多信息，而且如何从代码分析出清单文件，也与特定语言相关。根据输入的多少，可能将此称为 "世界 -> 清单文件" 会更准确。具体我们看下面一个真实的清单文件：

```toml
[package]
name = "iron"
version = "0.2.6"
description = "Extensible, Concurrency Focused Web Development in Rust."
readme = "README.md"
repository = "https://github.com/iron/iron"
documentation = "http://ironframework.io/doc/iron/"
license = "MIT"
authors = [
    "", # privacy: redacted
]

[lib]
name = "iron"
path = "src/lib.rs"

[features]
default = ["ssl"]
ssl = ["hyper/ssl"]

[dependencies]
typemap = "0.3"
url = "0.5"
plugin = "0.2*"
modifier = "0.1"
error = "0.1"
log = "0.3"
conduit-mime-types = "0.7"
lazy_static = "0.1"
num_cpus = "0.2"

[dependencies.hyper]
version = "0.7"
default-features = false

[dev-dependencies]
time = "*"
```

> [上述文件](https://gist.githubusercontent.com/sdboyer/cf32276135fbecdb7908/raw/2e249e2e05cb266b6d3563196193889ee7c40d54/Cargo.toml)，来自 Rust 语言 [iron crate](https://crates.io/crates/iron/) 项目的 Cargo 清单文件的修改版

由于 Go 目前并没有标准的清单文件格式（译注：本文写于 2016 年，现在 Go 已有 [go.mod](https://github.com/golang/go/wiki/Modules#gomod) 了），这里先用 Cargo（Rust）的清单格式作为示例。Cargo 就像很多其他包管理器那样，不只是 PDM，还是用户编译 Rust 程序的入口。清单文件包含的信息如下：
- `package` 部分：项目自身信息
- `features` 部分：参数化选项
- 各种 `dependencies` 部分：依赖包信息

下面我们将逐一介绍。


## 包管理中心
第 1 部分主要是 Rust 的包管理中心 [crates.io](https://crates.io/) 。是否使用包管理仓库可能是一个最重要的问题，大部分语言的 PDM 都有包管理中心：[Ruby](https://rubygems.org/)、[Clojure](https://clojars.org/)、[Dart](https://pub.dartlang.org/)、[js/node](https://www.npmjs.com/)、[js/browser](http://bower.io/search/)、[PHP](https://packagist.org/)、[Python](https://pypi.python.org/pypi)、[Erlang](https://hex.pm/)、[Haskell](https://hackage.haskell.org/) 等。当然，大部分语言（Go 语言是例外）别无选择，因为无法直接从源码仓库获取足够的信息检索依赖包。有些 PDM 仅依赖于包管理中心，而有些则同时也支持直接与源码仓库交互（如 GitHub、Bitbucket 等）。创建、托管、维护一项艰巨的任务。不过这与我无关！本文不是讨论如何构建高可用、高流量的数据存储服务。所以我可以愉快地跳过这个话题。

不过。。。

我虽然跳过了如何创建一个包管理中心的话题，但跳不过其中的功能角色。包管理中心可充当依赖包元数据的缓存，提供显著的性能优势。由于依赖包的元数据通常保存在清单文件中，而清单文件又必须存储在源码仓库中，所以若无包管理中心，则检索包的元数据通常需要 clone 源码仓库。但是，包管理中心可以提取及缓存这些元数据，并通过简单的 HTTP API 调用，快很多。

包管理中心也对元数据进行约束。例如 Cargo 的清单文件就有一个 `package.version` 字段：

```toml
[package]
version = "0.2.6"
```

仔细想想，这太不可思议了！版本号必须指向仓库的版本，但若将版本号写入到清单文件中，则同一个版本号可能对应着多个仓库版本，导致混乱。

Cargo 通过增加自身约束来解决此问题：发布一个版本到 `crates.io` 是 [永久不变的](https://doc.rust-lang.org/cargo/reference/publishing.html#publishing-crates)，所以其他提交是什么版本就无关重要了。从 crates 的角度看，只有第 1 个仓库版本对应真正的版本号。

其他包管理中心可能会对源码包的结构进行验证。若语言本身没有规定源码的目录结构，则 PDM 可能也会约定一种结构，而包管理中心则可强制执行此结构：npm 会自动寻找符合约定的 [模块对象](https://nodejs.org/api/modules.html#modules_the_module_object)；PHP 的 composer 会尝试 [验证是否兼容 autoloader PSR 规范](https://getcomposer.org/doc/01-basic-usage.md#autoloading)。（据我所知，我既不会这样做，也不确定这些方案是否可行）。而像 Go 这样的语言中，代码是自我描述的，基本没必要采用这些方案。


## 参数化
第 2 部分全部是关于参数化的。这里以 Rust 的细节为例，但也可引申到一般情况：可通过明确的参数定义，以修改项目输出的形式或实际逻辑行为。尽管有些方面与 PDM 的职责有重合，但这通常超出了 PDM 的职责范围。例如，Rust 的 [目标相关参数](http://doc.crates.io/manifest.html#the-[profile.*]-sections) 允许设置控制编译器优化级别的参数。编译器参数？纳尼？PDM 不应关心这些东西。

但是，某些类型的选项，如 Rust 的 `features` 字段，Go 的构建 tag，等等的构建参数（如 `test` vs `dev` vs `prod`）是可修改项目中使用的逻辑路径。这样的转变反过来可能会导致一些新的依赖关系，或消除了对其他依赖关系的需求。这种属于 PDM 的职责。

如果这种可配置性在你的语言环境中十分重要，则 "依赖" 这个概念可能需要些限制。另一方面，确保最小依赖集合只是（一般都是）减少了网络带宽。这是性能优化相关的话题，本小节可跳过，我们将在后面的 lock 文件一节继续讨论。

如果你不是精打细算的 Rust 粉丝或依赖包管理者，那 Cargo 的 `features` 字段可能就有点让人困惑了：到底该由谁在何时去对这些 feature 做选择？答案是：在 [顶层项目](http://doc.crates.io/manifest.html#usage-in-packages) 的清单文件中指定 feature。这就引出了一个我了解不多的一个大方向：`程序` 类型的项目 vs `库` 类型的项目。通常，`程序` 型的项目处于依赖链的顶端，又称顶层项目；而 `库` 程序则必须处于依赖链的下游。

`程序`/`库` 之间的区别有时并非那么的严格。实际上，要具体情况具体分析。 `库` 项目可能一般都处于依赖链的底端，但运行单元测试、性能测试时，它们就属于顶层项目。`程序` 项目通常处于顶层，但它们的子组件有时可以作为 `库` 使用，此时就属于依赖链底端了。有个比较好的区分方式是：一个项目可产生多少个可执行文件。

将 `程序` 和 `库` 统一成一个概念是好事，因为这实际认为使用相同的清单文件结构是合理的。而且清单文件应同时适用于底端（依赖包），也应适用于顶层（向主人提供元数据和选择的权限）。


## 依赖包
终于开始讨论 PDM 的正宗话题了！

每个依赖包的组成部分，至少包括一个唯一标识符、一个版本号。有些也可能包括一些参数和源码类型（如 原始 VCS vs 包管理中心）。清单文件中依赖包部分的修改必须是以下动作：

- 添加、删除依赖包
- 修改现有依赖包的版本
- 修改依赖包的源码类型参数

这些都是开发者实际需要完成的任务。删除功能可能太简单了，很多 PDM 都不提供这样的命令，认为你在清单文件删除一行就能完事。（但我认为 `rm` 这样的命令还是有必要的，原因后面我会细说）而添加依赖包，则通常不会比下面更难：

```
<tool> add <identifier>@<version>
```

或直接：

```
<tool> add <identifier>
```

这条命令通常暗示使用最新版本。

依赖包的唯一标识符的唯一硬性要求是：都处于同一个命名空间，PDM 可以从唯一标识符中解析出足够的信息（可能还需额外的字段如 `type` 字段，例如 `git` 或 `bzr`，又或者在包管理中心的话 `pkg`），来决定如何下载依赖包代码。基本上，它们都是一些特定领域的 URI，只要你避开了名称冲突，就没太大问题。

如果可以的话，建议将包的唯一标识符与代码里引用的名称一致（如 import、include 等的语句）。因为这样做有以下好处：

- 无需增加用户的记忆成本
- 无需增加静态分析器的内存消耗

而且，在语言限制允许的情况下，大部分 PDM 中可对包起一个别名。当下游正等待上游出一个补丁版本时，允许别名可算是提供了一种方便的 hack 方法来使用 fork 的补丁版本。当然，你也可以（译注：在清单文件中将依赖包名称）直接切换到真正的 fork 仓库。

再看看清单文件原本的目的：用于表达用户的意图。使用别名功能时，作者就可告诉其他人（特别是项目合作者，甚至用户），这只是一个临时 hack，所以预期不要太高。而且若觉得这样还不够清楚，还可添加注释来说明为什么会有这个别名。 

除了别名外，唯一标识符其实很简单。但是，版本不简单。


## 等等，我们要聊下版本
版本不是一个容易解决的问题。可能有些人觉得确实是，可能有些人觉得也没那么难。但我们应了解到底是什么原因导致版本问题变得困难（或其解决办法），这至关重要。

版本是一个很明确的东西，就是指一堆固定的代码。版本本身没什么疑问。关于版本，无法回答的问题之一，也是开发者最重要的问题之一是："如果将依赖包修改为不同的版本，我的代码还能正常工作吗？"。

尽管存在上述问题，但我们依旧会使用版本。其中的一个简单原因是 FLOSS 世界观的其中一项：

```
我知道存在一个限制边界，超过这个边界我可能就无法理解我所引用的依赖包代码了。
```

> 译注：人类不可能对每个依赖包的源码都能完全了解，精力是有限的

如果我们在使用或升级依赖包版本之前必须完全了解依赖包的源码，那我们将做不了任何事情。当然，若严格按照《A Software Engineering Best Practice》™ 做，可能整个软件行业都得废掉。这比误用版本导致软件不能运行的风险要大得多。（译注：所以我们不可能在使用依赖包前完全了解依赖包源码）

但是，人类在挑选依赖包版本时并没有太多明确的原因：因为机器不太懂如何挑选依赖包。也就是说，根据我们目前对计算机工作原理的了解，可认为目前不存在能够自动挑选出依赖包组合，且能让项目按预期工作的通用工具（即使是特定语言也不行）。当然，你可以写一些测试来验证能否工作，但别忘了 Dijkstra 限制：

```
根据经验的测试只能证明存在错误，不能证明没有缺陷。
```

在数学、计算机科学、类型理论方面比我厉害的人，可能可以恰当地解释这一点。但你也不了解我，所以这里说下我的想法：[类型系统](https://en.wikipedia.org/wiki/Hindley%E2%80%93Milner_type_system) 和 [按照合同的设计](http://c2.com/cgi/wiki?DesignByContract) 可以较好地确定兼容性，但若该语言是图灵完备的，则它们永远也达不到要求。它们最多能证明代码不兼容，当尝试更多组合时，就会因不断涌现更多的新边缘用例，而导致（译注：效率）递归下降。有位朋友曾将这种尝试称为 [哥德尔式打地鼠](https://en.wikipedia.org/wiki/G%C3%B6del%27s_incompleteness_theorems)。（友情提示：请不要入坑，哥德尔的连胜纪录长达 85 年）

这并不是什么深奥的理论，这只是一个简单的直觉：在判断 "我的代码与你的代码在一起能否正常运行？" 时，离不开人类的参与。机器虽然能帮忙，甚至能帮很多忙，例如做部分工作和输出结果，但机器不能做最终的决策。这就是我们需要版本的原因，也说明了版本相关工具的职责：帮助人类更好地做这些决策。

版本存在的唯一理由是用作代码作者向其用户传递原始信号的系统。你也可认为这是一个分享系统。若使用类似 [SemVer](http://semver.org/) 的版本方式，可能会对版本有以下建议：

- 通过两个版本之间的明确顺序关系，表示软件的演进过程
- 允许定义公开使用的版本的预备版本（如 `<1.0.0`，`pre-releases`、`alpha`/`beta`/`rc`）
- 允许表达互不兼容的版本关系
- 潜规则：如果作者发布了明确的版本，但你依然使用无版本号的 commit 版本，则容易出问题

若要让类似 SemVer 之类的版本规则能应用在你的语言中，重点在于如何针对对应语言做相应的适配，搞清楚哪些逻辑对应 SemVer 中的何种版本类型：主版本号（major）、次版本号（minor）、补丁版本号（patch）。[Rust 已先行采用 SemVer 了](http://doc.crates.io/manifest.html#the-package-section)， [Go 也是时候需要了](http://blog.merovius.de/2015/07/29/backwards-compatibility-in-go.html)，也应能实现。若没有约定的版本规则，每个人都凭感觉行事（或说是每个人搞出一套自己的规则，获得所谓的领土与防御），则当作者增加错误的版本号时，没有共同的沟通渠道，导致问题发生。这些约定的版本规则至关重要，不仅能解决当前的错误，而且是我们在使用版本时的有效沟通方式。

尽管版本号与代码之间并没有必然的对应关系，但至少有一种方法可以有助于软件日常运作：只需发布有限的几个公开的版本即可。因此，在集中注意力的情况下，[Linux 定律](https://en.wikipedia.org/wiki/Linus%27s_Law) 认为 bug 都将会被找出来和修复（并发布一个新的补丁版本）。这样一来，将精力集中到有限的特定版本上可降低这些版本的全局风险。这也就是 FLOSS 的另一个不确定性原则：

```
我必须为很可能发生的大多数事情做好准备。就像从谷壳中分离出小麦，也是需要一定时间的。
```

有版本的概念至少可以做到：如果此软件是有问题的，则肯定是因为你知道这个版本时有问题的，而非你随机从一堆 commit 中挑出一个后才发现有问题。这样既能节省时间，也能拯救项目于水火之中。



## 没有版本的版本
许多 PDM 还允许非标准 SemVer 的版本标识符。虽然这不是必须的，但有重要应用场景。有两种值得注意的其他版本标识符类型，且或多或少都与 VCS 仓库有关系：指定分支的标识符、指定某个 commit 的标识符。

使用何种版本标识符类型，在于你想如何依赖于上游库。也就是说：让别人的宇宙与你的宇宙协同运作的策略是什么？在我看来，有点像下面几个图：

![](https://miro.medium.com/max/370/1*UQYhXzJXvoFfH2Fr_KbkAw.jpeg) 

> ↑ 依赖于正式的版本

![](https://miro.medium.com/max/383/1*xFSWtpZd7YMLJXTClubYFA.jpeg) 

> ↑ 依赖于分支

![依赖于随便挑的 commit](https://miro.medium.com/max/690/1*V6vcc_Aw0w21KYPs-tJF8A.png) 

> ↑ 依赖于随便挑的 commit（译注：就像赌盘一样靠运气）


> 上述概念这些来自 iStockPhoto 的照片之一对应。哈

正式的版本标识符，为我提供了一个很好的、有序的软件包环境。而基于分支的版本标识符，就像把我搭在别人正在开的车上，说不定何时车上有人吐痰到我身上或断开绳子。基于 commit 的版本标识符仅当作者并未提供指导信息时才有用，此时你只能转动赌盘随机选中一个 commit 使用。一旦找到可运行的 commit，你应将此 commit 写入到清单文件中，告诉你的小伙伴们别再改此版本。

现在，任何选中的依赖包可能是没问题，也可能是有问题的。可能那些看似不错的软件包，实际说不定里面携带了 NSA（译注：美国国家安全局）的秘密后门。也可能前面开车拉着我的是一个训练有素的老司机。也可能赌盘桌上的是一个披着羊皮的狼？

无论如何，不同类型的版本标识符本质区别，在于其升级依赖包版本的过程不一样。纯粹基于仓库 commit 的版本标识符没有升级过程；基于分支的会一直跟随上游的变化而升级；而正式版本号的，特别是指定 [版本范围](https://docs.npmjs.com/misc/semver)，可让你按需控制（前提是上游的作者知道如何正确使用 SemVer）。

实际上，这又是对 FLOSS 世界观的扩展和探索：

```
我知道依赖别人的代码意味着我的项目将与其绑定，至少会需要些拉取和了解成本。
```

我们知道引入别人的代码会带来一定的风险和额外的开销成本。拥有一些明确约定的模式，至少可以在沟通时更畅通。并且我之所以关心这些问题，再说一次，是因为清单文件就是用户意图的表达：只需清单文件中的版本指定情况，即可清晰复现作者想要关联的依赖包代码。这样虽然并不能让上游代码变得更好，但遵循约定模式可减少沟通成本。如果你的 PDM 也能正常运作，则会减少获取依赖包过程的难度。


## 交换的原子
我曾经一直忽略了一个很重要的问题：项目（project）是什么？包（package）是什么？依赖项（dependency）是什么？当然，这些东西肯定是一堆代码，也基本都来自源码控制系统（译注：如 GitHub）。但到是底整个仓库，还是仓库的子集算是一个项目？所以，引出真正的问题："源码交换的原子是什么？"。

对于那些源码层级与文件系统无关联关系的语言，仓库就成为了语言缺失的天然代码边界。此时，仓库就是代码交换的原子，所以此时清单文件只能存放在仓库的根目录：

```sh
$ ls -a
.
..
.git
MANIFEST
<ur source code heeere>
```

而对于那些源码层级与文件系统有明确关联关系的语言，仓库就没有作为边界的价值了。（Go 就是最明显的例子，我将在后面 Go 小节详述）实际上，如果语言支持独立导入仓库的不同子集，若依然以整个仓库作为源码交换的原子就不太适合了。

对于依赖包使用者来说，有时只需仓库的某个子集或子集的不同版本，若依然必须拉取整个仓库，就显得比较麻烦。另外，对于依赖包作者来说，切分为多个子集会比较痛苦，因为作者会认为必须花额外的代价才能分解为多个子集。（切分为多个仓库是个麻烦事；我们只会在 [最后一代 DVCS（分布式版本控制系统）确认这样做是真的有用](https://bitquabit.com/post/unorthodocs-abandon-your-dvcs-and-return-to-sanity/) 时，才采取多仓库的方法。对于这种语言，最好能以仓库之外的方式作为代码交换的原子。如果你选择这条路，请记住以下几点：

- 清单文件（以及 lock 文件）与其旁边的代码有着特别有意义的关系。通常清单文件定义一个独立的 "原子"。
- 你的交互原子 **绝对**、**必须** 能有属于原子自己的时间线，并且你不能再依赖 VCS 提供原子的时间线了（译注：因为 VSC 的时间线都是整个仓库的，至少 commit 是对应整个仓库的）。没有时间线、没有自己的宇宙、没有自己的宇宙（译注：重要的事情说几遍）、没有 PDM、没有 PDM、没有什么是正常的了。
- 请记住：即使不算上时间线，软件本身已经足够困难了。时间线信息不应再放在源码里。没人喜欢在复杂的四维空间写真正的代码。

在清单文件中的版本与 [路径依赖](http://doc.crates.io/manifest.html#the-dependencies-section) 之间的联系，Cargo 貌似已找到了一种方法。


## 其他思考
下面是一些小想法，也会有些评论：

- 应选用一种首先人类友好，其次计算机友好的格式作为清单文件：TOML、YAML（其实 JSON 对人类不太友好）。这些格式是声明式和无状态的，所以简单。适当支持注释会很有用，因为清单文件是实验的容身之处，为与你合作的小伙伴留下实验原因的注释会很有帮助。
- [TIMTOWTDI（做一件事通常不止一种方法）](https://en.wikipedia.org/wiki/There%27s_more_than_one_way_to_do_it)，至少在 PDM 层次，是一个值得注意的主要问题。PDM 会进行内部自动化操作。除添加/删除/升级依赖包命令外，若 PDM 还有其他对清单文件的修改功能（[偶现的，不是必须的](http://faculty.salisbury.edu/~xswang/Research/Papers/SERelated/no-silver-bullet.pdf)），则应把这些动作放到其他命令中。
- 要决定是否提供包管理中心（大部分都会提供）。若提供的话，只要不影响、混淆 PDM 所需的依赖包信息，则应按需将尽量多的包管理中心信息放进清单文件。
- 避免将静态分析工具能明确得到的信息放到清单文件中，否则会有歧义。代码与清单文件有歧义是件很头痛的事。静态分析器并不是很难实现的工具。语言开发者或 PDM 开发者应实现静态分析工具，而非用户去重复填写。
- 应采用一种版本号命名模式（可以是 SemVer 或其他类似方案，并以 [全序关系](https://en.wikipedia.org/wiki/Total_order) 对其增强）。还应允许使用正式模式之外的命名，如：分支名、甚至 commit ID 等。
- 确定你的软件是否将 PDM 的行为与 LPM 等其他功能结合在一起（通常会）。若分离开，请写明分开后使用 PDM 所需的指引。
- 还有些其他类型约束（例如，所需编辑器/解释器的最低版本），也可放入清单文件中。没关系的。请记住，这是仅次于 PDM 的主要职责（尽管最终会交织）。
- 确定你的交换原子粒度。应针对你的语言语法做一个选择，但绝对要让原子有自己的时间线。



# 转化为 lock 文件
![](https://miro.medium.com/max/598/1*wRHhBbbZtGBfO3b-56JP6g.png)

> ↑ 箭头的转换过程见 [这里（你们靠得足够近了吗?）](https://www.youtube.com/watch?v=Uqduf3g__jE) （译注：作者在开玩笑）

从清单文件转化为 lock 文件，其实是开发过程中从初级毛坯状态固化为可靠、可复现构建的过程。不管开发者如何对依赖项进行疯狂的处理，lock 文件都可确保其他用户能复现其构建，完全没有思想负担。当我想到，要恰当处理这些滚烫热辣东西的应该是软件而非人类时，还是想到震惊的。

这种转换是我们为了减轻开发过程固有的风险的主要过程。这也是我们解决 FLOSS 世界观中特定问题的方式：

```
我知道为了确保我们创建的项目按预期工作，我/我的团队必须对项目负最终责任，不管上游发生什么问题。
```

现在，可能有些人会对精确可复现性的价值存怀疑。

以下是我曾见过的一些观点：

- "清单已经足够了"
- "在项目出问题前，可复现性一点都不重要"
- "npm 在 lock 文件推出前就已经流行很长一段时间了"

这些观点确实让我震惊，因为他们不是这么问："我是否需要可复现的构建？"、"若没有可复现的构建我是否会失去什么？"。顾名思义，可复现构建能让所有人最终受益。（除非你的乐趣是通过电子邮件发送构建结果的 tar 包，[SSH 进行每次细微的修改](http://www.quickmeme.com/img/9d/9daf428b4fcdc94c27fae2bd28107198595f299cdf03cff8fcd124832d57322b.jpg)）。唯一的问题是你在什么时候需要可复现的构建：a. 现在；b. 马上了；c. 我们一起合作看看怎么玩？我姑且认为你此时此刻并没编程，你现正在阅读本文，所以你变成了一个怪人，而我喜欢和怪人打交道。

可复现构建的唯一真正潜在缺点可能是会让工具变慢、变复杂（可能需增加额外的命令、参数），阻碍开发流程。这才是真正的问题，不过这种问题可能也只是目前的实现不够好而已，不代表可复现性本身的问题。实际上，这与 UX 原则有点像，PDM 应尽量 "苗条"：快、无感知、尽可能自动化。


## 算法
上述准则都只是大声喊 "需要算法"！实际上，lock 文件的生成必须完全自动化。这种算法可能会很复杂，但其基本步骤倒不难描述。

4 个步骤：

- 根据项目的清单文件，递归遍历所有依赖，构建出一个依赖包的图（可能是：有向无环图，还有各种标签）。
- 选择一个满足清单文件要求的 commit 对应的源码。
- 若存在依赖同一个依赖包的不同版本，请使用额外的策略来协调
- 凝固成最终的图（可能还需要每个包的元数据），并将其写入磁盘。叮！你的 lock 文件诞生了！

（作者注释：依赖包的版本选择问题是一个 [布尔可满足性问题](https://en.wikipedia.org/wiki/Boolean_satisfiability_problem)，它是 NP 困难的。上述步骤分解大体上还是有用的，但算法问题依然待解决。）

lock 文件所包含的完整的依赖包列表（即，在已计算的、完全解析完毕的依赖包图中的所有 [可达性依赖包](https://en.wikipedia.org/wiki/Reachability)）。"所有可达性" 是指清单文件中项目有下面 3 个直接依赖包：

![](https://miro.medium.com/max/275/1*w2cupzAkeHADLey0LtBb5A.png)

- 此项目直接依赖了 A、B、E
- A、B 都依赖了 C
- C 依赖了 D
- E 依赖了 F

我们在 lock 文件中直接列明了所有可达的依赖包：

![](https://miro.medium.com/max/412/1*VyMCI1xTDWytgzOQTBFjhQ.png)

> 此 lock 文件应包含所有依赖包：A、B、C、D、E、F

至于要包含多少元数据到 lock 文件，这要根据具体的语言而定。但必须包含：依赖包唯一标识符、来源信息、源码类型允许的离不可变版本（如 commit 哈希值）最近的版本编码。若你增加其他信息，如在磁盘的位置，没问题，只要你能保证推导出最终的源码时无歧义。

如果你对图算法很熟悉，那上述 4 个步骤中的第 一个和最后一个步骤或多或少会比较简单。遗憾的是，因我的图论能力有限，本文无法深入图论太多。有懂的小伙伴可以告诉我。

此外，中间两个步骤（实际都是选择一个版本，被拆成两个步骤）存在一些我们可以也应该面对的问题。第二个步骤比较简单。若已存在一个 lock 文件，请保持该版本记录，除非发生以下事情：

- 用户明确指定忽略 lock 文件
- 版本标识符是一个不稳定的版本号，如分支名
- 用户正在升级一个或多个依赖包
- 清单文件已被修改，且不再承认 lock 文件
- 判断版本冲突时不允许之前的版本

上述几点可避免不必要的修改：若清单文件接受 `1.0.7`、`1.0.8`、`1.0.9`，但 lock 文件记录的是 `1.0.8`，则后续的选择应注意到这一点，并继续使用 `1.0.8`。如果这样说能说明白，那就好！这是一个简单的示例，说明了清单文件与 lock 文件的基本关系。

这种思想是行之有效的，[Bundler](https://bundler.io/) 称之为 ["保守更新"](http://bundler.io/man/bundle-install.1.html#CONSERVATIVE-UPDATING)。[有些 PDM 建议不要](http://yehudakatz.com/2010/12/16/clarifying-the-roles-of-the-gemspec-and-gemfile/) 将 lock 文件提交到仓库中，至少 [是不一样的方式](https://getcomposer.org/doc/02-libraries.md#lock-file)。这其实是错失了一个机会。不管如何都将 lock 文件提交到仓库，因为 lock 文件可以简化用户的选择。而且，在计算顶层项目的时，可以很容易地解析并优先使用 lock 文件中的依赖包版本，而非使用满足清单文件要求的任意某个版本。依赖包中的版本选择的优先级会低于顶层项目 lock 文件的优先级（有的话）。下一节将会讨论依赖包版本冲突问题，但这些优先选项可以一定程度提高构建的稳定性。



## 钻石依赖、SemVer、忍耐，我要发飙了
第三个问题更难。当两个项目共享一个依赖项时，我们必须选择一种版本选择策略，正确的选择在很大程度上取决于语言特性。这也称为 "钻石依赖问题（diamond dependency problem）"，并且下面从图的一个子集看：

![](https://miro.medium.com/max/296/1*O_QKQh3E3Nb0kS1H8UTShQ.png)

> 蓝色框里面是一个快乐的钻石形状

上图并未列出版本号，所以看不出问题。但是，若 A 和 B 同时依赖了不同版本的 C，则会发现有冲突了，钻石被分割成下图：

![](https://miro.medium.com/max/230/1*7nHmFU4HILXsIMcTs14_fA.png)

> 钻石破裂了。同样值得注意的是：这个钻石只是一个图，但同时也是一棵树。你知道还有什么东西是树吗？文件系统。有没有感觉似曾相识？我感觉有。

有两类解决方案：

- 允许多个 C 的版本（重复）
- 或尝试调解此冲突（调解）

有些语言，譬如 Go，不允许使用前者。其他语言允许，但会有不同危险程度的副作用。不管哪种方法，本质都不正确。但是，若允许多个版本的 C，则永远不需要用户手动干预，对用户来说比较轻松简单。我们先来解决这个问题。

如果一个语言支持多个依赖包版本实例并存，则下一个问题是 "状态"：若依赖包实例可以操作全局状态，则多个依赖包实例的状态可能会有冲突。因此在 node.js 中，没有大量共享的状态，npm 只是以 "破裂钻石" 的树形式来避免所有可能的冲突。（尽管可通过 [deduping](https://docs.npmjs.com/cli/dedupe) 到达 "快乐钻石" 的目的，npm v3 默认使用此方法）。

另一方面，前端 JavaScript 由于拥有 DOM（全局状态共享的鼻祖），所以会让允许多版本并存有更大的风险。对于浏览器端包管理器 [bower](http://bower.io/) 来说，会基于调解（bower 将其称之为 "扁平化"）所有依赖包版本的方法，共享或不共享。（当然，前端 JavaScript 还有一个目标是尽量减少传输到客户端的代码大小）。

若你想要允许同一个依赖包的多个不同版本实例的方案，则需满足下面前提：
- 假设你的语言允许那样做
- 假设 [类型系统也不阻碍](http://www.well-typed.com/blog/2008/04/the-dreaded-diamond-dependency-problem/)
- 假设共享全局状态的风险可忽略不计
- 假设不考虑二进制程序/进程的膨胀大小、运行时的性能损失
- 假设对应的静态分析工具和移植工具都没问题

这种方案要求条件太多，但可能仍然比调解策略更好，因为大多数其他策略都需用户干预（俗称 [依赖地狱](https://en.wikipedia.org/wiki/Dependency_hell)），还因为有些逻辑本身存在可能令人难受的妥协之处。

假设 A -> C 和 B -> C 的关系都是使用指定的版本号，而非使用分支名或 commit ID，则调解策略包括：

- **[协商政策](http://blogs.forrester.com/f/b/users/JGOWNDER/highlander_0.jpg)**：分析 A -> C 的关系和 B -> C 的关系，看看 A 是否能安全地切换为使用 C-1.1.1，或看看 B 是否能安全地切换为使用 C-1.0.3。若都不能，看下面的 "实力政策"。
- **实力政策**：分析是否有 C 的其他 tag/release 版本同时满足 A、B 的要求。若不能，则看下面的 "变通政策"。
- **变通政策**：通过 fork 或 patch C 创建出一个新版本号来同时满足 A、B 的需求。至少，你会觉得这样可行，对吧？

等等！我忘说了一个捷径：

- **向朋友打电话咨询**：询问 A、B 的作者，看能否都同意使用 C 的某个版本。（若不同意，则回退到前面的 **协商策略**）

到目前为止，最后一种是最好的原始方法。与其让我花时间去解决 A、B、C 的冲突，还不如直接依赖 A、B 的作者所释放出来的信号，来寻找到 C 的最小不确定性版本：

![](https://miro.medium.com/max/179/1*Zaq1s63kPKG4c0Xo_mXQjQ.png)

- A 的清单文件指明了可使用 `>=1.0.2` 的任何补丁版本
- B 的清单文件指明了可使用 `>=1.0.0` 的任何次要、补丁版本

有多个可能的版本满足上述要求。但若 A 的 lock 文件指向了 `1.0.3`，则算法会选择 `1.0.3` 作为最终的选择。

现在的方案可能还不能实际跑起来。版本号毕竟只是一个粗糙的信号系统。1.x 可能表示很广的范围，并且可能 B 的作者选择这个版本号范围时比较随意。不管怎样，这已经是一个不错的开始了，因为：

- SemVer 范围只是表示建议的方案，并不代表我必须接受这些方案。
- PDM 工具可根据静态分析来进一步定义 SemVer 的匹配程度（若该语言的静态分析能提供一些有用信息的话）。
- 不管使用哪种调解方案，我都必须进行集成测试，以确保一切都符合我项目的特定需求。
- 所有的调解方案都是从可能的较大搜索空间（C 的所有可用的 commit 版本）中选择可接受的解决方案。虽然偶尔出现误报有点麻烦，但通过较小努力即可减少空间大小是很有用的。

最重要的是，假如我正在编写程序，发现 B 实际过于宽松，包含了一些让 C 不能工作的版本（或没包含一些能让 C 工作的版本），我可通过适当地修改 B 的清单文件来发布补丁版本。这些补丁版本公开记录了你这次的工作，可避免他人踩同样的坑。一个确定的 FLOSS 式的方案，解决了一个明显的 FLOSS 问题。



## 依赖包的参数化
在讨论清单文件时，我曾简短提了下：可能允许通过一定的参数化形式，来形成不同的依赖图。若你的语言能从中受益，则可能会让 lock 文件产生疑惑。

因为 lock 文件的目的在于完整地、无歧义地描述一个依赖包图。而参数化机制则会立马让 lock 文件的目的难以达成。较傻的方法是将每个参数组合在内存中都构造一个完整的图。假设每个参数都是一个二进制开关，则所需构造的图的数量会随参数的数量而呈指数增长（[2^N](https://en.wikipedia.org/wiki/Combination#Number_of_k-combinations_for_all_k)）！

上述方法不是 "很傻"，而是 "很傻很天真"。更好的方法可能是将参数所有可能的组合划分了几个集合。这些集合对应相同的依赖包版本输入，每个集合生成一个图。更更好的方法可能是先找到所有组合的公约最小依赖包版本组合子集，之后再在此子集之上构建剩余的参数集合组合。（再说一次，我算法能力有限，在这个问题上可能有我不知道的更优解。）

可能这听起来有些有趣，也可能你会觉得这有些胡言乱语。不管怎样，甚至不提供参数化也是一种选择。即使清单文件确实支持对依赖包进行参数化，你也可以直接获取所有内容，并且编译器会乐于自动忽略不需要的内容。并且，一旦你的 PDM 不小心变得很流行了，且收到会议邀请演讲，就会有人处理这些困难的部分，因为开源！

lock 文件的讨论基本到此为止。好了，下面准备讨论协议部分。




# 编译器，最初阶段：从 lock 文件到依赖包源码
![](https://miro.medium.com/max/598/1*hjmkjj5hFUBrIZa9RCcINg.png)

> 所有的转换过程都无需思考

若你的 PDM 严格生成 lock 文件，那最后一步可能相当于代码生成：

- 读取 lock 文件
- 通过网络获取所需的资源（当然也可能通过缓存获得）
- 按约定结构将资源存入磁盘

有两种封装代码的基本方法：

- 存放在源码树中（译注：即当前项目源码内）
- 放到一个另外的中央位置，并且依赖包的标识符、版本号均在路径中表示

如果后者可行会有一些优点，因为这样可向用户隐藏容纳该依赖包的源码树里其他无关的东西，而且用户一般也不需要查看。即使不能将中央位置直接用作编译器/解释器的输入（可能大部分语言都不能），但也可继续这么干，因为可用作缓存。磁盘不贵，也许你以后会找到一种直接使用它的方法。

若你的 PDM 属于后者，且无法集中存储，则必须将依赖包代码封装在其他位置。你甚至可能会认为，为了保证不被外部污染，唯一的 "其他位置" 可能是位于源码树下。（Go 的 vendor 目录模式很好满足了此需求）这意味着依赖包代码处于版本管理系统的管核范围内，立即产生了一个问题：是否应将依赖包源码也提交到仓库？

可能不用提交。关于如何提交，有些难以反驳的技术争议：

- 若你的 PDM 允许条件性指定依赖包，则哪个条件分支应提交依赖包代码？
- 嘿，Sam（译注：作者名字 sam boyer），很明显啦：提交所有的依赖包源码，让编译器自己选就好了
- 那为什么又首先要为 lock 文件构造参数化的依赖包图呢？
- 若不同条件的分支需要使用同一个包的不同版本呢？
- 还有。。。还有。。。

明白了没？真的很讨厌。也会有人能为完全不提交任何依赖包源码到仓库提供足够易懂的论点。问题是，谁会在乎呢？人们才不管，该提交的提交。考虑到 PDM 是减少风险的一种手段，所以正确的方法是让 PDM 提供某种功能来二选一："提交依赖包源码到仓库" 是一个安全的选择，或是一个错误的选择。

对于那些直接控制源码加载逻辑的 PDM 来说（通常是解析型或 JIT 编译语言），下载一个依赖包，且此依赖包中包含了其自带的子依赖包会很常见：你只需写个加载器来优先选择顶层项目的自带的依赖包源码。但是，像 Go 这样的语言，文件系统的结构就是规则，所以依赖包内的子依赖包源码会是一个问题：它们会覆盖掉你在顶层项目创建的依赖包源码。

对此，我们可简短地引用下分布式系统的描述：

```
分布式系统是指：你的计算机会因为一台你都不知其存在的另一台计算机而导致不可用。
```

这听起来如地狱一般。没错！既然没必要，那就不要采用类似分布式系统的机制。不应允许因 A君 滥用 PDM 而导致 B君 的构建失败。因此，若因语言限制而别无选择，则在下载到依赖包的子依赖包源码后，应在磁盘中将其依赖包的子依赖包源码删除（译注：以免影响顶层项目的构建）。

仅此，没说错吧，相当简单。



# 四种状态的变化
OK，前面已详细介绍了每个状态、每个协议。现在我们再组装成总图。

大部分 PDM 都会提供这些基本的命令给用户使用。这些命令应尽量直观易记。命令列表如下：

- **init**：创建一个清单文件，更好的甚至可以对已存在的代码进行静态分析后再填充清单文件
- **add**：添加指定依赖包到清单文件
- **rm**：从清单文件删除指定依赖包（通常没此功能，因为从清单文件直接删除对应部分即可）
- **update**：将 lock 文件中的依赖包版本升级到清单文件允许范围的最新版本
- **install**：下载依赖包源码，并将所有依赖包版本固定在 lock 文件中（若 lock 文件不存在，则创建）

我们定义的 4 种状态可很方便用图示方式表示这些命令与系统的交互。（为方便起见，这里将简称 4 个状态：P（Project Code，项目代码）、M（Manifest，清单文件）、L（Lock file，lock 文件）、D（Dependency code，依赖包代码））：

![](https://miro.medium.com/max/1000/1*ZKgIEl7VwjpLDmpIM-yVhA.png)

> 每个命令都是读取箭头起点的状态，变化到箭头终点的状态。`add/rm` 命令可由静态分析触发，但通常由用户所了解的情况而手动触发。`install` 命令会按需隐式创建 lock 文件。

不错。但有些明显的问题：我要使用某个依赖包，真的需要每次都跑起两到三个独立的命令才能到达目的吗？这挺烦人的。不过也有好的例子，[composer 的 require 命令](https://getcomposer.org/doc/03-cli.md#require)（相当于 `add` 命令）会做以下事情：在清单文件、lock 文件添加新依赖包记录、下载依赖包源码、存放到磁盘对应位置。

另一方便，npm 的 `add` 命令实质对应 [install](https://docs.npmjs.com/cli/install) 命令，此命令需要额外的参数才会修改清单文件或 lock 文件。（正如之前我指出的，npm 之前一直在图上使用重复的依赖包/树；那样会主要关心输出的依赖包，而非遵循我这里所说的状态流）因此，npm 需要额外的用户交互来更新所有的状态。Composer 会自动做这些事情，但其命令的帮助文档依然描述成分开的步骤。

我认为，应该还有更好的方式。


## 新概念：匹配、同步、持久化
注意：本方法可能比较少见。据我所知，目前没有 PDM 与本方法完全对应。非要说的话，或许可在 [Glide](https://github.com/Masterminds/glide) 中尝试一下。

可能文章开头的描述会比较清楚，但是我之所以将 PDM 问题转化为状态、协议，是为了创建一个足够简单的模型，以便将协议定义为单向转换函数：

- **f : P → M**：静态分析可在任何程度上从项目源码推导出依赖包的标识符、参数化选项，并存储到清单文件中。如果没有可行的静态分析，那这个函数只能手动进行。
- **f : M → L**：将项目清单文件中的直接的、也可能是松散的版本转换为依赖关系图中的完整可访问的一系列依赖包。每个依赖包都对应单个、不可变的版本。
- **f : L → D**：将 lock 文件中的固定的包列表转换为真正的源码，并按编译器/解释器期望的方式存放在磁盘上。

如果我们已经匹配好了输入、输出，则在给定现有输入状态的情况下，如果函数产生了现有的输出状态，则可定义每对状态为同步的。下图直观展示了一个完全同步的系统：

![](https://miro.medium.com/max/700/1*59bM6C1qc4M_Xd7ThL1YTw.png)

现在，假设用户手动编辑了清单文件，但尚未运行命令更新 lock 文件，则 M 和 L 将不会同步：

![](https://miro.medium.com/max/700/1*nyBlSTeogGR-oAVUj9g9-A.png)

> 清单文件和 lock 文件失去同步，但 lock 文件与依赖包源码依然是同步的，因为每个函数是独立的

类似的，如果用户刚 clone 了一个项目，里面包含了清单文件和 lock 文件。此时 L 和 D 是不同步的，因为 D 还不存在。

![](https://miro.medium.com/max/700/1*EBNCk2dVX4w5Q3q1Vg5JSA.png)

> 如果 lock 文件存在，但依赖包源码不存在，则是不同步

我还可列出更多情况，但我想你已经明白了。在任何指定时间，执行任何 PDM 命令，清单文件、lock 文件、依赖包源码均可处于以下 3 种状态之一：

- 不存在
- 存在，但与前面状态不同步
- 存在，且与前面状态同步

像这样将其全部列出来，可得出一个本来就很微妙的设计原理：如果可以很好地定义这些状态和协议，那 PDM 不能让这些状态同步的话，是不是显得很愚蠢？

是很愚蠢。

无论执行什么命令，PDM 都应尽量让所有状态（或至少后三个状态）完全同步。无论用户让 PDM 做什么事情，PDM 都会做，并应采取必要措施确保系统有序。采用这种 "基于同步" 的方法，我们的命令行流程图如下：

![](https://miro.medium.com/max/1000/1*2kzPU7jpXZmshq5DvA8JOg.png)

> 所有命令的发展方向没变，但都处于同步线上，以确保前置、后置环境的同步（此时可能不再需要 `install` 命令了！）

从这个角度看，PDM 的行为变成了一种前进或后退的动作，每个命令的核心是改变其中的状态，然后让系统根据改变的状态自动反应变得稳定。

这有个例子：有用户手动编辑清单文件，从中删除了 1 个依赖，然后执行 `add` 命令添加了 1 个其他依赖。对于仅限于执行用户命令的 PDM ，则不难想象此时 PDM 不会将手动删除的动作同步到 lock 文件中。对于基于同步的 PDM，该命令会在清单文件添加新的依赖项，然后将控制权交给同步系统。

对于用户来说好处很明显：无需额外的命令或参数来保证一切正常。对于 PDM 来说也有好处，只是可能没那么明显：若上一个 PDM 命令在内部结束时处于非同步状态，则下一个 PDM 命令碰到此状态时需判断为什么会导致不同步：是因为上一条命令遗留的？还是因为用户本来就想要这样的状态？确实难以判断，从而无法作出决策。而对于基于同步的 PDM 就没这种问题，因为会一直朝着合理的最终状态前进。

当然，有一个（潜在的）缺点：它会假定完全同步是唯一有用的状态。若用户真的想要一个不同步的状态的话，会是一个问题。我感觉，想要不同步的用户现在 [可能更多只是想解决当前 PDM 的问题，而非想最终解决真正的问题](http://faculty.salisbury.edu/~xswang/Research/Papers/SERelated/no-silver-bullet.pdf)。又或者，对不起，可能现在的 PDM 只是在 [误导](https://en.wikipedia.org/wiki/Gaslighting) 你。可能你还是喜欢自己的 [NoSQL 毒药](https://vimeo.com/95066828#t=13m04s)。本质上，我猜是因为基于同步的方法可让用户实际无需那么多精细的控制（译注：也能工作的很好）。

但是，基于同步的 PDM 至少还有一个主要的现实挑战：性能。若你的 lock 文件引用了 30 个 gzip tar 包，若执行每个命令时都需保证同步重新下载依赖包到磁盘的话会变得非常慢。若要让基于同步的 PDM 实用，则必须找到轻量的方案来判断每对状态是否处于同步。幸运的是，由于我们会小心地将每个同步过程定义为拥有清晰输入的函数，因此有一个比较好的解决方案：[内存化（memoizatiion）](https://en.wikipedia.org/wiki/Memoization)。在同步的过程中，PDM 应对每个函数的输入计算哈希，并将哈希作为输出信息的一部分。后续再次运行时，计算新输入的哈希，并对比旧输入的哈希，若一致，则认为状态已同步。

现在，我们实际只需关心：**M** → **L** 和 **L** → **D**。前者很简单：输入是清单文件的依赖项列表，加上一些参数，而哈希值则可塞进已生成的 lock 文件中。

后者有些不好弄。输入的计算依然很简单：对 lock 文件中已序列化的图计算哈希值，或者如果你的系统有参数化的依赖项，则计算你正使用的子图。而问题在于，没有适合的地方存储这些哈希值。所以不必费心尝试：只需写入到依赖项子目录的根目录的一个特殊命名文件中。（若你很幸运，能将 标识符+版本 的命名空间写入到中央目录，则你可跳过所有这些内容）

上述两种方法都不完美，但我猜在实践中会很好用。




# 结局
为 PDM 写一份通用的、考虑所有注意事项的指南是一项很艰巨的任务。我甚至在写本文时我已至少错过了一件重要作品。我肯定忽略了很多潜在的重要细节。欢迎对本文进行评论和建议。我希望本指南是很合理的，因为依赖包管理已是目前大多数语言的真真正正的需求，并且这是一个很广泛的问题，已大量存在于各种零碎的邮件列表讨论中。我已看到很多社区正是出于此原因而努力尝试解决。

就是说，我确实认为我已在此列举了很多方面的问题。不过，归根结底，技术细节可能没有各种模型概念那么重要：语言的依赖包管理本质是为了减少伤害，通过维持以下的平衡来实现：

- 人 vs 机器
- 混乱 vs 有序
- 开发 vs 运维
- 风险 vs 安全
- 实验 vs 可靠

由于需要长期管理，所以对于语言来说，尝试在源码中管理自己的源码时间顺序是不合理的，也即是说几乎所有语言都可使用 PDM 来避免这个问题（甚至或者可共享 1 个 PDM）。对于一个语言社区来说，我不太建议被多个互不兼容的打包工具分隔。

好了，观众们，下面将讲述 Go 语言的相关细节。

![](https://miro.medium.com/max/1953/1*QrP3uWcGAZfNWFDCwZOdmQ.jpeg)



# Go 语言 PDM
到目前为止，本文都在聊比较抽象的内容。本节将聊些具体的：

- 给 Go 创造一个 PDM 是可行的，也不会太难，甚至可在短期内很好地集成到 `go get` 命令中。
- 为 Go 创建一个包管理中心会很有用，但需从零开始。
- monorepo（译注：[大仓库模式](https://en.wikipedia.org/wiki/Monorepo)） 很适合内部使用，PDM 也应适用于此情况。但 monorepo 没有包管理中心，这对开源不利。
- 我有一个行动计划，肯定会让大家惊喜。但若想理解详情，你需读到最后。

----

Go 多年来一直在依赖包管理这一块苦苦挣扎。我们有一些部分解决方案，讨论也没结束，但没什么真正值得关注的。我认为之所以如此，是因为从未完整考虑过整个问题。确实，至少在 [最后](http://github.com/golang/go/issues/12302) [两个](https://github.com/golang/go/issues/13517) 主要的讨论中，有一个重要的挑战。也许，随着全局蓝图的浮现，视线才会不那么模糊。

首先，让我们从通用情况完全转到 Go 语言中。下面 3 个是影响我们决策的主要约束：

- GOPATH 模式是个可怕的、不可持续的策略。这点我们都认同。幸好，Go 1.6 添加了对 [vendor 目录机制](https://golang.org/cmd/go/#hdr-Vendor_Directories) 的支持，为封装构建、以项目为中心的 PDM 打开了一扇大门。
- Go 的连接器只允许一个 package 对应一个唯一的 import 路径。若允许 import 路径重写，则可解决此问题，但 Go 还有 package 层级的变量和 init 函数（即 package 层级的全局变量和非幂等变化）。这两个事实导致了在碰到钻石依赖时，类似 npm 那种 "破裂钻石" 形式的重复包模式不适用于 Go 语言。
- 缺失一个包管理中心。源码仓库依然是源码交换的原子。这与 Go 围绕目录/package 的语义十分不符。

还有，在看过多年来社区中的许多讨论之后，发现其中似乎经常遗漏了两个想法：

- 依赖包管理的方法必须是双面的：
  - 对内：当你的项目处于 "顶层"，且依赖了其他依赖包时
  - 对外：当你的项目是一个可被其他项目引用的依赖包时
- 经过大家对 "减少危害" 的必要性有所了解，但过多的关注如何通过可复现的构建来减少危害，而忽略了如何减轻开发者在日常工作中所面临的风险和不确定性。

也就是说，我认为情况并没想象中那么差。真的。有问题需要解决，但也有好消息。先来看看好的一面。



## 升级 `go get`
与 Go 的很多其他事情一样，Go 的简单、正交设计使得创建 PDM 的技术相对容易（前提是需求明确）。更好的是，有一个透明的、向后兼容的渐进路径来将功能包含进目前的核心工具链中。其实很容易描述：

![](https://miro.medium.com/max/700/1*K2iI64eyjVbPLRXqeMwhyg.png)

我们可定义一个符合本文所提到的约束的 lock 文件，并集成到 `go get` 命令中。关于 "一致性"，最重要的是 lock 文件是以下内容的完整依赖包列表：

- 当前目录所在的 package
- 所有的子 package

但不应包含：

- test 文件
- 构建的 tag 参数

也可以想象成下面仓库目录结构：

```
.git                 # 说明这是仓库目录
main.go              # main package
cmd/
  bar/
    main.go          # 另一个 main package
foo/
  foo.go             # 一个非 main package
  foo_amd64.go       # 特定架构的文件，可能包含额外的依赖包
  foo_test.go        # 测试文件，可能包含额外的依赖包
LOCKFILE             # lock 文件
vendor/              # `go get` 根据 lock 文件将依赖包源码存于此
```

如果 `LOCKFILE` 包含 3 个包（`<root>`、`foo`、`bar`）所涉及的全部依赖项的不可变具体版本，则 `LOCKFILE` 是正确的。然后 `go get` 算法会做一下动作：

1. 从指定子路径（例如 `go get <repoaddr>/cmd/bar`）返回，直到找到 `LOCKFILE` 或到达根目录
2. 若找到 `LOCKFILE`，则将依赖包源码存放到旁边的 `vendor` 目录中
3. 若找不到 `LOCKFILE`，则回退到 GOPATH 模式那种基于分支名的行为

这是可行的，因为 `LOCKFILE` 与 `vendor` 目录平行：对于那些在 `vendor` 的父目录内的目录树（本例子中，就是仓库根目录），编译器会先到 `vendor` 目录内查找依赖包，找不到再去 GOPATH 中找。

本例子中，我将 `LOCKFILE` 放到了仓库根目录，可以不放在根目录。即使放在子目录，本算法依然可行：

```
.git
README.md
src/
  LOCKFILE
  main.go
  foo/
    foo.go
```

上述目录结构不一定是最佳实践，但 `go get <repoaddr>/src` 也适用于上面算法。

此方法甚至还适用于其他情况，譬如：

```
.git
foo.go
bar/
  bar.go
cmd/
  LOCKFILE
  main.go
```

如果 main package（译注：此时 main 在 `cmd/` 目录内）同时 import 了 `foo` 和 `bar`，然后将 `vendor` 目录放到 `cmd/` 目录内 `LOCKFILE` 旁边。也就表示 `foo` 和 `bar` 不会被包含到，此时编译器会回退到从 GOPATH 中 import 包。不过，我 [发现了一个小技巧](https://github.com/sdboyer/nesttest)，可以通过将此仓库完全复制一份到其 `vendor` 目录内即可：

```
$GOPATH/github.com/sdboyer/example/
 .git
 foo.go
 bar/
   bar.go
 cmd/
   LOCKFILE
   main.go
   vendor/github.com/sdboyer/example/
       foo.go
       bar/
         bar.go
       cmd/          # PDM 将从这里往下开始忽略
         LOCKFILE
         main.go
```

看着有些奇怪，不好开发，但这样确实是以自动化的方式解决了问题。

当然，我们能适配这种场景，但不代表我们应该这样做。不过，真正的大问题是：我们应在哪放置 lock 文件及其旁边的 vendor 目录？

如果你只想满足特定的情况，就会容易陷入死胡同。但是，如果我们回到 PDM 必须以基本的 "交换原子" 为基础的设计理念，答案就会更明朗。清单文件和 lock 文件一同代表了一个项目 "原子" 的边界。尽管它们与项目源码并无必然的联系，但考虑到我们正在使用文件系统，当它们（译注：指 lock 和 vendor）位于 "原子" 所包含的源码根目录时，事情往往能运作的最好。

这不要求它们（译注：指 lock 和 vendor）一定要位于仓库的根目录，放在子目录也是可以的，只要它们所描述的 Go package 都在其子目录中。真正有问题的情况是：不同的 lock 文件（和清单文件）描述了多个不同的树或子树。前面我已提到，如果依靠仓库提供的时间线，多个包/原子共享同一个仓库是不可行的。这是毫无疑问的，但在 Go 语言里，我们不能只说 "别这样做"。

相反，我们需要在发布单个项目的最佳实践与公司内部 monorepo（大仓库）的可行方法之间划清界线。鉴于 Go 语言的诞生地 Google 内部盛行 monorepo，还有最近几年受此影响而发布的代码，划清这些界线尤为重要。



## 共享、monorepos（大仓库）、我可管理的最简单的 `别那样做`
很多适应了 FLOSS 环境的人们可能会说："为什么你要用 monorepos 呢？"。不过 monorepos 之所以被几十亿美元的科技公司采用，[肯定也有其积极的一面](http://danluu.com/monorepo/)。除此之外，monorepos 是针对 FLOSS 生态系统某些不确定性的一个缓解策略。对于个人来说，有了 monorepos，就有可能知道什么叫 "全部代码"。这样，在 [综合]() [构建]() [系统]() 的帮助下，从理论上讲，个人作者需对其更改导致下游的影响承担全面的责任。

在 monorepo 中，当然，所有的代码都存放在一个仓库中。将所有代码放到一个 monorepo 中，子组件都依赖于 monorepo 的时间线，因此它们缺少属于自己的时间信息（或称有意义的版本控制）。只要你在同一个宇宙里，使用同样的时间线，那也没问题。但是，我还是想再说一次的是，对于在 monorepo 之外的人们而言（我们这些正在将宇宙缝在一起的人），良好的版本信息至关只要。有意思的是，这还包括那些要将其他 monorepo 缝合在一起的人们。

现在，如果 Go 有一个包管理中心，则我们可借用 [Rust 的路径依赖模式](http://doc.crates.io/manifest.html#the-dependencies-section)，并提供由源码仓库的子包/子树组成的独立版本交换原子。如果你能摆脱 [git/hg/bzr 引起的斯德哥尔摩综合症的束缚（Stockholm syndrome，即 "小仓库就是好东西™，不因为什么，就因为 Unix 哲学是这么说的）"](https://bitquabit.com/post/unorthodocs-abandon-your-dvcs-and-return-to-sanity/)，你就会发现 Rust 这种方式有很多好处。如果一个人喜欢维护大量仓库，且这些仓库都只是半成品，得互相关联才能工作，那我认为这个人要么是受虐狂，要么是机器人，要么是骗子。

但 Go 目前确实还没有中央仓库，以后可能会有吧。不过现阶段，我们只能将整个仓库作为交换的原子，即接受这种方式带来的时间线约束。并且需划清公共仓库与私有仓库的界线：

- 公共仓库允许其他人 import
  - 称为 "开放"
- 私有仓库不打算让别人 import
  - 称为 "封闭"
  - 一般来说指的是 monorepos

通常有以下特点：

- 开源仓库通常可安全、放心地引用；闭源仓库则不行（只是重申一下）
- 开源仓库应包含 1 个清单文件、1 个 lock 文件、1 个 vendor 目录，最好还是在根目录
- 开源仓库应 commit 其清单文件、lock 文件，不应 commit 其 vendor 目录
- 闭源仓库可包含多个清单文件、lock 文件，但需要你自己理清其中关系
- 闭源仓库可以 commit 其 vendor 目录，但也要理清关系
- 不管是开源还是闭源仓库，你都不应修改 vendor 目录里的上游代码。若确实需要修改，请 fork 依赖包的仓库，修改后再引用你的 fork 依赖包仓库

虽然我认为上述几点很好，但 PDM 也要从实际出发：鉴于公共 Go 生态普遍存在 "封闭型" 的 monorepos（`golang.org/x/*` 说的就是你），所以需要支持在同一个仓库中引用不同的版本。这是可以的。但一个理性的、基于版本的生态需要进化，社区要么放弃这种封闭的 monorepos，要么创建一个包管理中心。我所见过的所有可能的折中方案都会引发版本控制的问题，所以都不值得考虑。



## 语言版本控制与行动计划
去年秋天，Dave Cheney 为实现语义版本控制作出了 [勇敢的努力](https://github.com/golang/go/issues/12302)（译注：提出了一个 Golang 提案）。据我估计，这次提案讨论对推进社区使用版本控制系统具有重要意义。不幸的是，这次提案没被采纳（不是因为什么问题，而是因为没有具体的成果）。

而幸运的是，我不认为没被正式采纳就没进步。采用语义版本控制只是 Go 社区必须要解决的问题之一。下面是一个大概的行动计划：

- 感兴趣的人们应共同创建并发布一个通用的建议，关于怎样的类型更改会需要什么级别的语义版本变化（译注：主版本号、次要版本号、补丁版本号分别对应什么类型的修改）
- 感兴趣的人们还可协作写一个工具，对 Go 项目仓库进行基本的静态分析，以检测变化的大致级别，从而辅助决定最初应使用怎样的版本号。（然后，可以的话基于该检测自动发布 issue）
- 如果我们感到不爽，还可以在分析、发布工具的辅助下，建立一个网站来记录 Go-dom 的语义化进展。这种方式虽然笨拙，但至少促进集体行动。
- 同时，我们还可以尝试一种最简单的可能情况：[只为仓库根目录定义一个 lock 文件](https://github.com/golang/go/issues/13517)。如果存在的话，`go get` 可读取和直接使用。
- 随着 SemVer 的使用范围越来越广，[不同的](https://github.com/Masterminds/glide) [社区](https://github.com/tools/godep) [工具](https://github.com/kardianos/govendor) 可以尝试不同的工作流、方法来解决 monorepo/版本控制的问题，但仍遵循相同的 lock 文件约定。

Go 的依赖包管理不必是很棘手的问题，只是看似难解决而已。尽管还有些方面我没涉及到，如：已编译对象的版本控制、通过静态分析减少依赖包图的大小、允许本地路径覆盖以方便开发者等，我想我已尽可能覆盖各个方面了。如果我们能朝着这些目标取得积极进展，那到今年年底或 GopherCon 会议时，可以一起研究令人耳目一新的 Go 生态。

Go 是时候添加版本管理功能了，不然太痛苦了。简单、易于静态分析的 Go PDM 可很容易成为同行中闪亮的例子。我可以毫不费力地想象在 `gofmt`/`goimport` 命令中加入直接写入清单文件的功能。或者扩展其本地代码上搜索的能力，扩展为支持从包管理中心（我们应该创建一个）自动搜索依赖包，若本地不存在则自动下载。这些，还有更多，都很类似，所缺少的只有愿意不愿意。

如果你对本文的任何内容感兴趣，欢迎 [联系我](https://twitter.com/sdboyer)。也可在 [Go PM 邮件列表](https://groups.google.com/forum/#!forum/go-package-management) 中讨论。


感谢：

- Matt Butcher、Sam Richard、Howard Tyson、Kris Brandow、Matt Farina 对本文的评论、讨论和建议
- Cayte Boyer 对我的忍耐（译注：貌似是本文作者的老婆？）
