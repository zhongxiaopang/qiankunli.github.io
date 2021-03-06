---

layout: post
title: 《软件设计之美》笔记
category: 架构
tags: Architecture
keywords: system design principle 

---

## 简介（未完成）

* TOC
{:toc}

软件设计学习的难度，不在于一招一式，而在于融会贯通。

算法对抗的是数据的规模，而软件设计对抗的是需求的规模。

## 分离关注点

大多数系统设计的不够好，问题常常出在分解这步没做好。常见的分解问题就是分解粒度太大，把各种维度混淆在一起（比如技术维度和业务维度）。 举个例子，比如因为存储性能不够，又是批量又是缓冲，代码写的很麻烦。但其实最根本的解决之道 是找一个性能较高的存储。

## 如何了解一个软件的设计？

了解设计三步走：模型 ==> 接口 ==> 实现。

1. 模型，也可以称为抽象，是一个软件的核心部分，是这个系统与其它系统有所区别的关键，是我们理解整个软件设计最核心的部分。
2. 接口，是通过怎样的方式将模型提供的能力暴露出去，是我们与这个软件交互的入口。
3. 实现，就是软件提供的模型和接口在内部是如何实现的

我们肯定要先知道项目提供了哪些模型，模型又提供了怎样的能力。如果模型都还没有弄清楚，就贸然进入细节的讨论，你很难分清哪些东西是核心，是必须保留的，哪些东西是可以替换的。如果你清楚了解了模型，也就知道哪些内容在系统中是广泛适用的，哪些内容必须要隔离。

**但如果只知道这些，你只是在了解别人设计的结果，这种程度并不足以支撑你后期对模型的维护**。在一个项目中，常常会出现新人随意向模型中添加内容，修改实现，让模型变得难以维护的情况。造成这一现象的原因就在于他们对于模型的理解不到位。

我们都知道，任何模型都是为了解决问题而生的，所以，理解一个模型，需要了解在没有这个模型之前，问题是如何被解决的，这样，你才能知道新的模型究竟提供了怎样的提升。也就是说，**理解一个模型的关键在于**，要了解这个模型设计的来龙去脉，知道它是如何解决相应的问题。

## 程序设计语言

程序设计语言的发展就是一个“逐步远离计算机硬件，向着待解决的问题靠近”的过程。

程序库就是为了消除重复而出现的。而**消除重复，也是软件设计的初衷**。

程序库最初只是为了消除重复。后来，逐渐有了标准库，然后有了大量的第三方库，进而发展出包管理器。程序设计语言的接口不只包含语法，还有程序库。而且，学习一种程序设计语言提供的模型时，不仅仅要看语法本身有什么，还要了解有语言特性的一些程序库。语法和程序库是在解决同一个问题，二者之间是相互促进的关系。一些经过大量实践验证过的程序库会变成语言的语法；如果语法不够好，新的程序库就会出现，新一轮的编程模型就开始孵化。比如synchronized ==> aqs ==> synchronized。

## 扩展系统

《系统性能调优必知必会》

AKF 立方体在《The Art of Scalability》一书中被首次提出，旨在提供一个系统化的扩展思路。AKF 把系统扩展分为以下三个维度：
1. X 轴：直接水平复制应用进程来扩展系统。X 轴扩展系统时实施成本最低，只需要将程序复制到不同的服务器上运行，再用下游的负载均衡分配流量即可。X 轴只能应用在无状态进程上，故无法解决数据增长引入的性能瓶颈。
2. Y 轴：将功能拆分出来扩展系统。Y 轴扩展系统时实施成本最高，通常涉及到部分代码的重构，但它通过拆分功能，使系统中的组件分工更细，因此可以解决数据增长带来的性能压力，也可以提升系统的总体效率。比如关系数据库的读写分离、表字段的垂直拆分，或者引入缓存
3. Z 轴：基于用户信息扩展系统。Z 轴扩展系统时实施成本也比较高，但它基于用户信息拆分数据后，可以在解决数据增长问题的同时，基于地理位置就近提供服务，进而大幅度降低请求的时延，比如常见的 CDN 就是这么提升用户体验的。但 Z 轴扩展系统后，一旦发生路由规则的变动导致数据迁移时，运维成本就会比较高。

## 体会

开发层面讨论微服务的更多是框架、治理、性能等，但是**从完整的软件工程来看我们严重缺失分析、设计能力**，这也是我们现在的工程师普遍缺乏的技术。我们经常会发现一旦你想重构点东西是多么的艰难，**就是因为在初期构造这栋建筑的时候严重缺失了通盘的分析、设计**，最终导致这个建筑慢慢僵化最后人见人怕，因为他逐渐变成一个怪物。

依赖方先ready，然后我们紧接着进行测试、发布吗。如果是业务、架构合理的情况下，这种场景最大的问题就是我们的项目容易被依赖方牵制，这会带来很多问题，比如，研发人员需要切换出来做其他事情，branch 一直挂着，不知道哪天突然来找你说可以对接了，也许这已经过去一个月或者更久，这种方式一旦养成习惯性研发流程就很容易产生线上 BUG 。

“最简单的需求分析，是将需求抽象成函数，比如findMax(),findMin()。好的需求分析，是将需求抽象成参数,比如findData(int sortIndex)”