---
title: 3650 天自学编程
date: 2019/10/16
description: 这篇文章翻译自 Peter Norvig 的 Teach Yourself Programming in Ten Years
tag: translation, Chinese
author: Wenzhao
---

# 3650 天自学编程

> 这篇文章翻译自 Peter Norvig 的 _Teach Yourself Programming in Ten Years_，[点击这里阅读原文](http://www.norvig.com/21-days.html)。

## 为什么大家都这么着急？

走入任何一家书店，你都能看到诸如《24 小时自学 Java》这样的书，还有数不胜数的有关 C、SQL、Ruby 或者算法的类似书籍。使用 Amazon 的高级搜索功能：[title: teach, youself, hours, since: 2000](http://www.amazon.com/gp/search/ref=sr_adv_b/?search-alias=stripbooks&unfiltered=1&field-keywords=&field-author=&field-title=teach+yourself+hours&field-isbn=&field-publisher=&node=&field-p_n_condition-type=&field-feature_browse-bin=&field-subject=&field-language=&field-dateop=After&field-datemod=&field-dateyear=2000&sort=relevanceexprank&Adv-Srch-Books-Submit.x=16&Adv-Srch-Books-Submit.y=5)，你能找到 512 本书。最前面 10 本中，有 9 本和编程相关（另外一本是关于簿记的）。把 teach youself（自学）替换成 learn（学习），或者把 hours（小时）替换成 days（天）也会得到类似的结果。

所以，要么人们都急着想学编程，要么编程和其他技能相比尤其简单易学。Felleisen 等人在他们的著作 [How To Design Programs](http://www.ccs.neu.edu/home/matthias/HtDP2e/index.html) 中就用到了这个梗，他们写道：“如果不计较代码的质量有多糟糕，编程是很简单的，傻瓜都能在 21 天内学会。”Abtruse Goose 漫画[有一辑也拿这个开涮](http://abstrusegoose.com/249)。

让我们来详细分析一下“24 小时自学 C++”（[Teach Youself C++ in 24 Hours](http://www.amazon.com/Sams-Teach-Yourself-Hours-5th/dp/0672333317/ref=sr_1_6?s=books&ie=UTF8&qid=1412708443&sr=1-6&keywords=learn+c%2B%2B+days)）这样的标题到底意味着什么

- 自学：在 24 小时内，你根本没有时间去写有意义的代码，并从成功或者失败中获得经验教训。你也没有时间去和有经验的程序员合作，了解 C++ 开发生态究竟是怎样的。简单来说，你没有时间深入学习。所以，这些书仅仅能让你熟悉一下 C++，谈不上深入理解。而正如亚历山大·波普所说：一知半解最危险。
- C++：在 24 小时内，你最多学习一些 C++ 的语法（前提是你掌握了其他一门编程语言），但是你对如何使用这门语言知之甚少。打个比方，假如你是一个 Basic 程序员，你能学到的仅仅是如何用 C++ 语法写 Basic 风格的程序，但是你不可能知道 C++ 擅长（或者不擅长）做什么。[Alan Perlis](http://www-pu.informatik.uni-tuebingen.de/users/klaeren/epigrams.html) 曾经说过：“不能影响你编程思维的语言，根本不值得一学”。如果你是为了使用一个现成的工具去解决一个具体的问题而学习 C++（更有可能是去学习 JavaScript 或者 Processing 这样的语言），你并不是在学习编程，而是在学习如何解决那个问题。
- 24 小时：不行的是，这远远不够，我会在下一个章节详细阐述。

## 3650 天自学编程

研究者们（[Bloom (1985)](http://www.amazon.com/exec/obidos/ASIN/034531509X/), [Bryan & Harter (1899)](http://www.norvig.com/21-days.html#bh), [Hayes (1989)](http://www.amazon.com/exec/obidos/ASIN/0805803092), [Simmon & Chase (1973)](http://www.norvig.com/21-days.html#sc)）已经发现，在许多领域中——包括下棋、游泳、网球、神经心理学和拓扑学等——要想达到专家级的水平，大约要花 10 年时间的努力。关键在于*刻意*学习：不是简单的重复，而是选择一个具有挑战性的、稍稍超出现有能力的任务，尝试完成它，分析你的表现和成果，纠正失误，然后不断重复再重复这个过程。银弹似乎不存在：即使天才如莫扎特，4 岁开始作曲，在创作第一首世界级音乐之前也积淀了 13 年的时间。披头士在 1964 年接连空降音乐榜单首位，登上 Ed Sullivan 秀并成为现象级乐队，似乎是一夜之间发生的事情，但他们从 1957 年起就开始在利物浦的小俱乐部里表演。尽管他们早年就征服了乐迷，但直到 1967 年发表了 _Sgt. Peppers_ 后才受到音乐批评界的褒扬。

[Malcolm Gladwell](http://www.amazon.com/Outliers-Story-Success-Malcolm-Gladwell/dp/0316017922) 让这个理念广为人知，尽管他具体到了 10000 个小时而不是含糊的 10 年。Henri Cartier-Bresson（1908-2004）也发表过类似的言论：“你的头一万张照片是你最废的照片”（他并没有预料到数码相机的出现，现在有的人一周就能拍这么多）。达到大师级水平则可能需要一生的时间，Samuel Johnson（1709-1784）曾说，“只有奉献一生的时间和精力，才能在某个学科成就卓越”。乔叟抱怨过：“吾生也须臾，学海也无涯”。希波克拉底（c. 400BC）的名言更是广为人知 “ars longa, vita brevis”，这一句其实节选自 “Ars longa, vita brevis, occasio praeceps, experimentum periculosum, iudicium difficile”，翻译成中文就是“生也有涯，知也无涯，灵光稍纵即逝，实验变幻莫测，判断殊为困难”（英文原文为 “Life is short, [the] craft long, opportunity fleeting, experiment treacherous, judgment difficult.”）。当然，并不存在这样一个神奇的数字——不同的人精通不同的技艺（比如编程、象棋、跳棋、演奏乐器）需要的时间都是一样的，怎么可能！但是像 [K. Anders Ericsson](http://www.amazon.com/K.-Anders-Ericsson/e/B000APB8AQ/ref=dp_byline_cont_book_1) 教授说的那样：“在大部分领域中，即使是最天才的人为了达到最高水平，所需花费的时间也是十分巨大的。10000 小时这个数字只是给受众一个概念：我们这里谈论的是每周 10 到 20 小时，持续数年的不断投入。”

## 如果你想成为一名程序员

如果你想成为一名成功的程序员，我这里有一些建议：

- 发掘编程的**乐趣**，为了好玩而编程。确保编程是如此的有趣，值得在它身上花费 10 年 / 10000 个小时。
- **编程**。最好的学习方式就是[做中学](http://www.engines4ed.org/hyperbook/nodes/NODE-120-pg.html)。用更术语化的口吻来说：“个人在某领域的效率最大化，不是靠经验的积累自动实现的，即使是已经具备充足经验的人，也可以通过刻意的努力来提升其表现”（[366 页](http://www2.umassd.edu/swpi/DesignInCS/expertise.html)）。“最有效的学习方法要求定义良好的任务、对于学习者来说合适的难度、有效的反馈，以及重复和纠正错误的机会（20-21 页）。_[Cognition in Practice: Mind, Mathematics, and Culture in Everyday Life](http://www.amazon.com/exec/obidos/ASIN/0521357349)_ 这本书佐证了以上的观点。
- 与其他程序员**交流**。阅读其他人写的程序，这比读任何书或接受任何培训都要重要。
- 如果愿意的话，去读**大学**（能读到研究生更好）。这使得你能够去做一些需要资质证明的工作，并且让你对某个领域有更加深入的了解。但是如果你不喜欢上学的话，你可以（如果有足够的决心）通过自学或者工作来获得类似的经验。不论如何，光读书是不够的。“计算机科学教育并不能使人成为一名杰出的开发者，就像学习刷子和颜料并不能使人成为画家一样”，_The New Hacker’s Dictionary_ 的作者 Eric Raymond 曾这样说。我听说过的最厉害的开发者中的一位仅仅拥有高中学历，他写了许多[很棒](http://www.xemacs.org/)的[软件](http://www.mozilla.org/)，有他自己的[新闻集团](http://groups.google.com/groups?q=alt.fan.jwz&meta=site%3Dgroups)，并且通过股权收益买下了一间[夜店](http://en.wikipedia.org/wiki/DNA_Lounge)。
- 和其他程序员**一起做项目**。在一些项目里，要做顶尖的程序员，而在另一些项目里，可以做最糟糕的程序员。当你是顶尖程序员的时候，你可以测试自己领导一个项目的能力，用你的眼界来启发别的成员。当你是最差的程序员的时候，你可以观察顶级程序员的做法，而且你会知道他们不喜欢做什么（因为他们会把这些事情交给你来做）。
- 从其他程序员的项目里学习。理解别人写的代码，看看当原作者不在的时候，你需要耗费多少精力来学习代码和修复问题。思考如何设计代码结构以让后续的维护者更容易接手你的代码。
- 学习至少 6 种**编程语言**。包括一种强调类的语言（Java、C++），一种强调函数抽象的语言（Lisp、ML、Haskell），一种支持句法抽象的语言（Lisp）、一声明式语言（Prolog、C++ 模板），以及一种强调并行性的语言（Clojuse、Go）。
- 记得在“计算机科学”这个词里有**“计算机”**三个字。了解电脑执行指令、从内存取数（包括使用或不使用缓存的情况）、从磁盘顺序读取数据，或者是磁盘寻道各自要花费多少时间。（答案见下文）
- 参与**语言标准**的制定。可以是列席 ANSI C++ 委员会，也可以是确定团队的代码缩进是 2 个还是 4 个空格。不管怎样，你都会了解到其他人如何看待一门语言，他们的看法有多强烈，甚至是为什么他们有这样的看法。
- 知道何时**停止**纠结于语言标准。

知道了以上这些，很难不令人怀疑仅仅通过书本学习，一个人能在编程这条路上走多元。在我第一个孩子出生之前，我读了所有“如何……”这样的书，但最终还是免不了变成一个一无所知的新手爸爸。30 个月后，我的第二个孩子临盆时，我重新读了一遍那些书吗？不，相反，我依赖自己的经验，最终事实证明经验比育儿专家们写的大部头更有效。

Fred Brooks 在他的论文没有银弹（[No Silver Bullet](http://en.wikipedia.org/wiki/No_Silver_Bullet)）中提出了一个培养优秀软件架构师的三部曲：

1. 尽早开始系统地寻找顶级程序员。
2. 给候选人指定一个导师，负责他的职业发展，并仔细记录其成长轨迹。
3. 给这些成长中的程序员机会来交流并互相激励。

这个方法假定候选人已经具备了某些成为优秀架构师所必须的特质，用人方所需要做的仅仅是把他们发掘出来。[Alan Perlis](http://www-pu.informatik.uni-tuebingen.de/users/klaeren/epigrams.html) 以一种更简明的方式陈述了这一观点：“人人都可以学会雕塑，但米开朗琪罗只能学习如何不会雕塑“，他的意思是说卓越的人都有一些内在的特质来帮助他们超越他们所接受的训练。但这种特质是哪里来的？是天生的，还是通过后天的勤奋养成的呢？就像《料理鼠王》的古斯图大厨所言：“人人都能烹饪，但只有无所畏惧的人才能成为大厨”，我更愿意把这句话理解为：要成就非凡，人需要有把一生的大半时间都奉献给这件事的意愿。但是无所畏惧这个词可能仅仅只是其中的一种表达。或许像古斯图的美食批评家安东说的那样：“不是所有人都能成为伟大的艺术家，但是伟大的艺术家可能来自任何地方。”

所以尽管去买 Java、Ruby、JavaScript、PHP 教程书吧，你或许能学到一些有用的东西，但是绝不可能在 24 小时或者 21 天内改变人生，或者是让自己的开发水平有什么质的改变。你说通过 24 个月持之以恒的努力能不能做到呢？嗯，你已经开始上道了……

## 参考文献

Bloom, Benjamin (ed.) _Developing Talent in Young People_, Ballantine, 1985.

Brooks, Fred, _No Silver Bullets_, IEEE Computer, vol. 20, no. 4, 1987, p. 10-19.

Bryan, W.L. & Harter, N. "Studies on the telegraphic language: The acquisition of a hierarchy of habits. _Psychology Review_, 1899, 8, 345-375

Hayes, John R., _Complete Problem Solver_ Lawrence Erlbaum, 1989.

Chase, William G. & Simon, Herbert A. [“Perception in Chess”](http://books.google.com/books?id=dYPSHAAACAAJ&dq=%22perception+in+chess%22+simon&ei=z4PyR5iIAZnmtQPbyLyuDQ) _Cognitive Psychology_, 1973, 4, 55-81.

Lave, Jean, _Cognition in Practice: Mind, Mathematics, and Culture in Everyday Life_, Cambridge University Press, 1988.

## 答案

常见 PC 上各种操作大致所需的时间：

| 执行的常见操作              | 1/1,000,000,000 秒 = 1 纳秒 |
| --------------------------- | --------------------------- |
| 从 1 级缓存取数             | 0.5 纳秒                    |
| 分支预测                    | 5 纳秒                      |
| fetch from L2 cache memory  | 7 纳秒                      |
| Mutex lock/unlock           | 25 纳秒                     |
| fetch from main memory      | 100 纳秒                    |
| 通过 1Gbps 网络发送 2K 内容 | 20,000 纳秒                 |
| 从内存顺序读 1MB 内容       | 250,000 纳秒                |
| 磁盘读寻道时间              | 8,000,000 纳秒              |
| 从磁盘顺序读 1MB 内容       | 20,000,000 纳秒             |
| 从美国往欧洲发包并返回      | 150 毫秒 = 150,000,000 纳秒 |

## 附录：选择编程语言

有几个人问过我第一门编程语言应该选择哪个。没有标准答案，但你可以从这些方面考虑：

- _参考朋友_。当被问及“我应该用哪种操作系统，Windows Unix 还是 Mac？”的时候，我的回答通常是：“用你的朋友们在用的那种”。你从朋友那里学到的东西，大于系统或者编程语言本身的差别。或者考虑你未来的朋友——你将要加入的程序员社区的成员。你所选择的语言有一个蓬勃发展的社区，还是有一个不断萎缩的用户群？有足够的书籍、网站或者论坛能够方便地获取资源吗？你喜欢论坛里的那些人吗？
- _保持简单_。像 C++ 和 Java 这样的编程语言是给由富有经验的程序员所组成的大型团队开发专业级产品用的，他们非常在乎运行性能。结果就是，这些语言有一些非常复杂的东西来应对他们所面临的问题。而你在学习编程的时候不需要这些复杂的东西。你需要的是一门设计之初就注重易学性的语言，任何新手程序员都能掌握的那种。
- _写_。你会用哪种方式学习弹钢琴？一种是常见的、交互的方式：按下琴键就会马上听到声音；另一种是“批处理”模式，你只有弹完一整首曲子才能听到声音。显然，用交互的方式学习钢琴才会简单，这对编程来说也是一样的。坚持使用一种交互式的语言。

基于以上原则，我像编程新手们推荐的第一门编程语言是 Python 或者 Scheme。另外一个可选项是 JavaScript，并不是因为它对新手友好，而是因为有很多在线教程，比如可汗学院的这一套教程。如果学习者的背景比较特殊的话，还有其他一些好的选择。儿童可能会更喜欢 Alice、Squeak 或者 Blockly 这些语言（老年学习者或许也会喜欢）。重要的是做出选择之后马上开始学习。
