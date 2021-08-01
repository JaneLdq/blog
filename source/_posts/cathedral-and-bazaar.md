---
title: 《大教堂与集市》笔记与读后感
date: 2017-07-09 23:19:42
categories: 读书笔记
---

在前辈的介绍下阅读了The Cathedral and The Bazaar（《大教堂与集市》）这本书。本书的作者Eric S. Raymond（埃里克·斯蒂芬·雷蒙），被称作是"an open-source software advocate"，也是开放源代码促进会（Open Source Initiative）的主要创办人之一。
《大教堂与集市》这本书1999年出版，在2017年来看已经是相当古老了，然而如今Linux开源社区的蓬勃发展恰好证实了书中提出的诸多观点。
ESR在书中总结了一些与软件开发相关的经验教训，包括他对开源与闭源软件开发模式的比较，他在开发Fetchmail的过程中对市集模式开发的试验结果等等。

<!--more-->
## Lessons摘录
1. Every good work for software starts by scratching a developer's personal itch.（每一个好的软件都源于戳到了开发者本人的痛点。）
2. Good programmers know what to write. Great ones know what to rewrite(and reuse).（好的程序员知道要写什么，伟大的程序员知道要如何重写（与重用）。）
    ESR在这里用到了一个词constructive laziness（建设性的懒惰），我觉得很形象。我们常说，聪明人都是会偷懒的人，怎么才能用最少的代价、在现有的资源上获得最大的利益或者是跟进一步的提升。既然已经有巨人的肩膀可以借用，或者即使只有几个台阶，那肯定是要比自己从零开始要简单的。
    至于如何获取台阶，怎么跟巨人搞好关系，一方面就跟个人的知识储备十分相关，另一方面也是考验快速获取与过滤信息的能力。
3. "Plan to throw one away; you will, anyhow." - Fred Brooks, The Mythical Man-Month （计划扔掉一个；无论如何，你总会扔掉一个的。——《人月神话》）
    > You often don't really understand the problem until after the first time you implement a solution. The second time, maybe you know enough to do it right. So if you want to get it right, be ready to start over at least one.

    在读到上面这段时，深以为然。虽然现在比较正式的项目经验基本为零，但在之前的学习过程中，一般写出来的项目代码最后自己都不忍直视，实在是太糟糕了。这就是因为在实际开发过程中，很多问题在真正着手开始设计或实现时需要思考到的细节和深度是在初步设想中无法预料到的，在这一阶段会暴露出很多模糊的点，很可能在逐一弄明白之后发现原来的设计存在很多不合理。由此可得，在问题清晰以及有了初版的实现之后，对项目进行重构几乎是百分百要做的事情。

4. If you have the right attitude, interesting problems will find you.（如果你有正确的态度，有趣的问题自然就来了。）
    不知道怎样的态度才算正确能让我遇到有趣的灵魂呀。

5. When you lose interest in a program, your last duty to it is to hand it off to a competent successsor.（如果你对一个项目失去了兴趣，你最后的职责就是把它交给一个称职的继承者。）
    虽然你们没有能一起走到白头，但没买不成仁义在，请和平分手。

6. Treating your users as co-developers is your least-hassle route to rapid code improvement and effective debugging.（把用户当做合作者对待是通往快速改进代码和有效调试的最佳路径。）
    这一条在开源项目上能体现到代码级别，因为开源项目的用户很多也是对该项目感兴趣的开发者，在有编程背景（而且通常是能力很强的hacker）的用户参与下获取的改进信息通常也是高质量的。
    同时我觉得即使是从一般软件的开发过程来看，用户作为软件的最终使用者，与用户的持续合作也是保证产品从宏观的功能上不跑偏的有效措施。
    ESR在这里提到了“Two-level architecture”和“Two-tier user community”，并以MATLAB为例指出“对于类似架构的产品，产品的开放部分——有一个巨大多样的用户群可以推敲的地方——才是动力、热情和创新的所在”。原文如下：
    > Software products with a two-level architecture and a two-tier user community that combined a cathedral-mode core and a bazaar-mode toolbox. ... Users of MATLAB and other products with a similar structure invariably report that the action, the ferment, the innovation mostly takes place in the open part of the tool where a large and varied community can tinker with it.

7. Release early. Release often. And listen to your customers.（早发布。常发布。听取用户意见。）
    1999年的观点，发展到2017年，其实已经不需要多说什么。也许完美主义者总是想把所有问题都解决好之后再发布一个产品，这样的变更代价是巨大的。尽早发布一个项目虽然会有很多bug，但是只要产品的核心价值能戳中用户的痛点，使其能忽视其他无伤大雅的bug就不会有问题了；再加上快速迭代，快速更新，认真汲取反馈的好态度，用户就像是到了一个味道超棒的小店，虽然这个餐厅有点小而乱，但美食的诱惑实在太大还是会成为忠实的回头客。

8. Given a large enough beta-tester and co-developer base, almost every problem will be characterized quickly and the fix obvious to someone.（如果beta测试者和合作开发者的群体足够大，几乎每个问题都会快速显形，会有人轻而易举地把它解决。）
    这里提炼出了ESR的著名理论Linus's Law —— "Given enough eyeballs, all bugs are shallow."。这其中包含了大教堂模式与市集模式的差别，即在市集模式中，有很多合作开发者琢磨分析一个新版本的产品，不同的人有不同的知识技能，从不同的角度看待问题，那么这些bug就很有可能对于一些人而言是非常简单的,从而快速解决问题。相反在大教堂模式下，只有少部分固定的人，可能需要花费很多时间来钻研。
    Debugging is parallelizable.开源项目在开发上的优势：beta测试者对源码有一定了解，在提交bug发生时运行环境能给出更详细的描述，而不只是停留于表面症状；核心成员少，沟通成本较低（更多的是beta测试者和协助人员，这些外沿开发者通常是平行单独的子项目，彼此交接少）

9. Smart data structures and dumb code works a lot better than the other way around.（优秀的数据结构与简陋的代码组合远比反过来的组合好。）
    任何的业务逻辑都是对数据的处理，业务上的信息在收集、处理后在计算机的世界里都将被表现为数据。对数据的优雅存取、转换、处理、分析是十分重要的了，尤其在大数据爆炸的时代。所以要好好学数据结构呀。

10. If you treat your beta-testers as if they're your most valuable resource, they will respond by becoming your most valuable resource.（如果你以“最有价值的资源”来对待你的beta测试者，他们会以成为“最有价值的资源”来回应）

11. The next best thing to having good ideas is recognizing good ideas from your users. Sometimes the latter is better.（仅次于拥有好主意的就是识别出来自于用户的好主意。有时候后者会更好。）

12. Often, the most striking and innovative solutions come from realizing that your concept of the problem was wrong.（最有突破和创新的方案常常来自于意识到你把问题的概念弄错了。）
    这一点和第三条有点相似，在不断的深入分析问题的过程中，可能会出现顿悟的情况，茅塞顿开，突然意识到方向跑偏了。那么，也存在开发过程中遇到死胡同，死活找不到解决方案时，就需要想一想“问题”本身是对的吗？或许问题要需要重新定义哦。

13. "Perfection(in design) is achieved not when there is nothing more to add, but rather when there is nothing more to take away." by Antoine de Saint-Exupéry（设计达到完美的时候，不是增加得不能再增加了，而是减少得不能再减少了。——圣埃克苏佩里，《小王子》作者）
    在产品功能定位，软件设计包括后期的开发、迭代过程中，一直持有这个意识非常重要。很多时候，我们都觉得能够尽量多的为用户提供各种各样功能会增加用户友好度，仿佛提供了便捷。但更多的选择意味着更高的决策成本，更复杂的上手过程。
    想起这句话，Less is more， by Ludwig Mies Van der Rohe，德国建筑师。

14. Any tool should be useful in the expected way, but a truly great tool lends itself to uses you never expected.（任何一个工具都应该达到预期的用处，但是一个真正棒的工具会带来你从未预想过的用处。）

15. When writing gateway software of any kind, take pains to disturb the data stream as little as possible - and never throw away information unless the recipient forces you to!（在写任何关口软件的时候，花点功夫尽可能不要干扰数据流——除非用户强迫你，永远不要扔掉任何信息。）

16. When your language is nowhere near Turing-complete, syntactic sugar can be your friend.（当你的语言离图灵穷尽还差得远的时候，给语法加点风味可以有所帮助。）
    在计算资源昂贵的时期，程序员习惯选择更简洁无冗余的控制语法来编写程序，但是发展到今天，资源成本大大降低，软件的规模有可能会很大，那么代码的可读性、可维护性都要被纳入考虑。在成熟的开发过程中，不同的团队或者公司都会制定各自的代码风格或编码规范。

17. A security system is only as secure as its secret. Beware of pseudo-secrets.（一个安全系统的安全性取决于它保守的秘密的安全性。小心伪秘密。）
    其实我没有很理解这一条，大概就是要注意区别一些看似提高了安全性的方法在实质上是否真的能做到这一点，是不是多此一举，暂时没有想出很好的例子。

18. To solve an interesting problem, start by finding a problem that is interesting to you.（要解决一个有意思的问题，首先找到一个你觉得有意思的问题。）
    兴趣是最好的老师啦。

19. Provided the development coordinator has a communication medium at least as good as the Internet, and knows how to lead without coercion, many heads are inevitably better than one.（如果开发的协调者有一个至少和互联网一样好的通讯媒介，并且懂得如何不通过强迫来领导，多个头脑一定是由于单个头脑的）

## 新的开始
在阅读过程中，会觉得有些观点是显而易见的，有一些又是当看到作者提出来的后会觉得是能够在开发过程中以此作为新的切入点来思考问题的，在摘录时也加上了一些自己的理解，不管是浅陋还是深刻，希望从此培养起写读书笔记的习惯。:)
2017-07-09
