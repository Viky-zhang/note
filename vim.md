# Vim 使用感受

本文不是为了教你如何学习 Vim，而是聊一下为什么喜欢用 Vim。

这里说的 Vim 可以指 Vim 本身，或某些编辑器中的 Vim 插件（如 VSCode、PyCharm）。

# 目录

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [以前、现在](#%E4%BB%A5%E5%89%8D%E7%8E%B0%E5%9C%A8)
- [目的](#%E7%9B%AE%E7%9A%84)
- [简要说明 Vim 相比其他编辑器的好处](#%E7%AE%80%E8%A6%81%E8%AF%B4%E6%98%8E-vim-%E7%9B%B8%E6%AF%94%E5%85%B6%E4%BB%96%E7%BC%96%E8%BE%91%E5%99%A8%E7%9A%84%E5%A5%BD%E5%A4%84)
- [什么是 `肌肉记忆`？](#%E4%BB%80%E4%B9%88%E6%98%AF-%E8%82%8C%E8%82%89%E8%AE%B0%E5%BF%86)
- [对比下上述的优缺点](#%E5%AF%B9%E6%AF%94%E4%B8%8B%E4%B8%8A%E8%BF%B0%E7%9A%84%E4%BC%98%E7%BC%BA%E7%82%B9)
- [本质分析](#%E6%9C%AC%E8%B4%A8%E5%88%86%E6%9E%90)
- [肌肉记忆真的值得么？](#%E8%82%8C%E8%82%89%E8%AE%B0%E5%BF%86%E7%9C%9F%E7%9A%84%E5%80%BC%E5%BE%97%E4%B9%88)
- [肌肉记忆太难形成怎么办？](#%E8%82%8C%E8%82%89%E8%AE%B0%E5%BF%86%E5%A4%AA%E9%9A%BE%E5%BD%A2%E6%88%90%E6%80%8E%E4%B9%88%E5%8A%9E)
- [感觉 Vim 的快捷键毫无规律，怎么学？](#%E6%84%9F%E8%A7%89-vim-%E7%9A%84%E5%BF%AB%E6%8D%B7%E9%94%AE%E6%AF%AB%E6%97%A0%E8%A7%84%E5%BE%8B%E6%80%8E%E4%B9%88%E5%AD%A6)
- [Vim 的快捷键设置真的合理么？](#vim-%E7%9A%84%E5%BF%AB%E6%8D%B7%E9%94%AE%E8%AE%BE%E7%BD%AE%E7%9C%9F%E7%9A%84%E5%90%88%E7%90%86%E4%B9%88)
- [Vim 能有 IDE 的功能么？](#vim-%E8%83%BD%E6%9C%89-ide-%E7%9A%84%E5%8A%9F%E8%83%BD%E4%B9%88)
- [Vim 的功能是不是很弱？](#vim-%E7%9A%84%E5%8A%9F%E8%83%BD%E6%98%AF%E4%B8%8D%E6%98%AF%E5%BE%88%E5%BC%B1)
- [Vim 与 IDE 的结合](#vim-%E4%B8%8E-ide-%E7%9A%84%E7%BB%93%E5%90%88)
- [Vim 的插件安装是不是很麻烦？](#vim-%E7%9A%84%E6%8F%92%E4%BB%B6%E5%AE%89%E8%A3%85%E6%98%AF%E4%B8%8D%E6%98%AF%E5%BE%88%E9%BA%BB%E7%83%A6)
- [Vim 的配置文件是不是太复杂？](#vim-%E7%9A%84%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E6%98%AF%E4%B8%8D%E6%98%AF%E5%A4%AA%E5%A4%8D%E6%9D%82)
- [总的建议](#%E6%80%BB%E7%9A%84%E5%BB%BA%E8%AE%AE)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# 以前、现在

好多年前就知道有 Vim，但很长时间内都很抗拒使用 Vim，觉得 Vim 不好用，反人类。

没想到现在居然喜欢用 Vim，甚至觉得没有 Vim 模式的编辑器很难用。

# 目的

个人认为这种转变的理由值得聊一聊，有点点哲学的味道。

想说服一些认识 Vim 很久，但总觉得 Vim 无从喜欢的人去喜欢使用 Vim。

至少喜欢使用 Vim 的模式（如 VSCode 中的 Vim 插件），而不一定纯 Vim 本身。

# 简要说明 Vim 相比其他编辑器的好处

一句话：本质都是文本编辑的需求，Vim 是肌肉记忆，普通文本编辑器不是。

# 什么是 `肌肉记忆`？

肌肉记忆就是中学生物书上说的 `条件反射`，类似所谓的 `望梅止渴`。

譬如，有这样的文本编辑需求：`鼠标目前在双引号内，需删掉双引号内的所有文本，并重新输入其他文字`。

- 普通文本编辑器：手移到鼠标，眼睛看着屏幕，移动鼠标选中词双引号内的文本，手移到键盘，按下删除键，开始输入新文字。
- Vim：眼睛看着屏幕，键盘输入 `ci"`（即删除当前光标下，且在双引号内的文本），开始输入新文字

# 对比下上述的优缺点

- 普通文本编辑器：
  - 优点：所见即所得，想选择啥文本就选择啥，几乎不用学习，人类直观感觉几乎就知道怎么做
  - 缺点：
    - 手与眼睛需要做的步骤更多
    - 移动鼠标去选择文本时，不容易选中，容易误选，从而消耗较多时间
    - 若是使用笔记本的触摸板，比鼠标更不容易精确选中文本
- Vim 编辑器：
  - 优点：
    - 步骤更简单，完全不需要操作鼠标
    - 形成肌肉记忆后，选中文本更精确快速，节省时间
  - 缺点：刚接触 Vim，完全看不懂 `ci"` 是什么意思，需要去记忆，需要多练习，经过一段时间（可能一天、一周、一个月）才能记住

# 本质分析

不管是普通文本编辑器，还是 Vim，所做的本质都是实现了：`鼠标目前在双引号内，需删掉双引号内的所有文本，并重新输入其他文字`。

Vim 其实只是将一系列文本的 `选择`、`动作` 做成了快捷键（如 `ci"`），Vim 还有很多其他类似的快捷键，如：

- 回到文件第一行：`gg`
- 删除一行并重新输入新文字：`cc`
- 移动上下左右：`kjhl`
- 选择当前行以及后面的几行：`Vjjjj`
- 从当前光标开始删除至行末，并输入新文字：`C`
- 跳到行末，并开始输入文字：`A`
- 跳到行首，并开始输入文字：`I`
- 还有很多很多可组合的快捷键

# 肌肉记忆真的值得么？

Vim 里有太多的快捷键，根本记不过来。

答：肌肉记忆靠的不是记忆，而是练习后、或长期使用后的条件反射。

肌肉记忆就像小学时学的 `九九乘法表`，为什么要背诵 7 乘以 9 等于 63？明明当年数学老师花费了好几节课来推导、讲解乘法表的来源。

你现在说出乘法表中内容时，应该也是几乎不用思考就脱口而出，这可算是肌肉记忆的一种。

# 肌肉记忆太难形成怎么办？

不得不承认，Vim 的快捷键太多了：

- 刚开始能记住的没几个
- 用了一年后发现记住了不少，但还有更多没记住的
- 用了 2 年后发现又记住了很多，但依然还有很多没记住的

可能会有人问，反正都记不全，为啥还有用 Vim？

答：记住一部分快捷键，也能满足日常文本编辑使用。

总之，狠下心来，坚持使用一个月以上，你才会对 Vim 不太抗拒。

# 感觉 Vim 的快捷键毫无规律，怎么学？

Vim 的快捷键也不是完全随意的，也可以有些推导规则，如 `文本对象`、`操作动作`、`如何重复`。

# Vim 的快捷键设置真的合理么？

答：实践证明总体比较合理，也存在不合理，但也没看到专门理论分析最合理该是如何。

以 `上下左右` 移动光标为例：

- `j`：向下
- `k`：向上
- `h`：向左
- `l`：向右

为什么不是 `yuio`键 或 `abcd`键 来实现上下左右移动呢？

答：

- jkhl 刚好在平时手放键盘手指的正下方，手指移动的距离较少（不一定是最少）
- 确实有点随意，但实践使用下来后发现，还行

# Vim 能有 IDE 的功能么？

答：可以有，但难。

那为啥不全去用 IDE 好了？

答：可以，可以在 IDE 中安装 Vim 插件，既有 IDE 的功能，也有 Vim 的方便快捷键。

对于服务器，IDE 界面是很难有的，原因包括：

- 系统资源无必要消耗：服务器运行时的大部分时间是提供服务，而非编程，所以提供界面会导致资源额外消耗
- 网络传输不适合实时界面操作：服务器通常是远程控制的，若使用界面，则网络传输速度较慢，很容易导致界面传输卡顿

# Vim 的功能是不是很弱？

答：是，也不是。

是的原因：

- 代码自动补全，此功能需安装插件，这个需要一定门槛，而且有时容易装错，而 IDE 开箱即用的自动补全功能
- 对于 Java 代码的重构，Vim 确实难以比 IntelliJ IDEA 更好用
- 同理于代码 Debug
- 同理于代码格式化

不是的原因：

- 对于普通代码、文本编辑，Vim 的功能几乎都有，譬如：
  - 增删改查
  - 语法高亮
  - 多窗口
  - 多文件编辑
  - 文件树浏览
  - 快速跳转
  - 快速重复编辑动作
  - 等等

# Vim 与 IDE 的结合

Mac 下，我比较喜欢用 VSCode + Vim 插件，既可简化代码补全、参数提示、Debug，也可使用 Vim 的各种快捷操作。

服务器中，可以 VSCode 远程开发，同样 VSCode + Vim 插件。

非远程开发时，服务器上经过简单配置，Vim 也可以很好用。

# Vim 的插件安装是不是很麻烦？

答：

- 不麻烦。本质是通过配置文件加载指定的插件而已。
- 个别插件依赖外部环境太多可能会难安装些

# Vim 的配置文件是不是太复杂？

答：可以不用配置太复杂就能很好使用。

# 总的建议

入门时：坚持使用 1 个月以上 Vim，形成初步肌肉记忆，才能体会其中好处。

更深入使用：多思考下平时的文本编辑习惯中，还有哪些耗时的编辑动作没用到 Vim，再去网上搜对应的快捷键或方法。如此形成良性循环。不用怕 Vim 的超多快捷键，够用即可。
