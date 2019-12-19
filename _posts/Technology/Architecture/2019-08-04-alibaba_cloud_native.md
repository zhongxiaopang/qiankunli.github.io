---

layout: post
title: 《阿里巴巴云原生实践15讲》笔记
category: 技术
tags: Architecture
keywords: kubernetes cni

---

## 简介（未完成）

* TOC
{:toc}


云原生的“上下文”是什么？阿里2015年前全部使用内部基础设施 ==> 2015-2018 活动期间 向阿里云申请“机房”（一批服务器）双十一后归还； 部分业务上云 ==> 全面上云 ==> 云原生

## 全面上云

[把阿里巴巴的核心系统搬到云上，架构上的挑战与演进是什么？](https://www.infoq.cn/article/NWRrJjBWzgCQoq4QxoJw)全面上云也不只是将机器搬到云上，更重要的是 replatforming，集团的技术栈和云产品融合，应用通过神龙服务器 + 容器和 K8s 上云，数据库接入 PolarDB，中间件采用云中间件和消息产品，负载均衡采用云 SLB 产品等。

1. 自研神龙服务器
2. 存储方面，走向了全面云化存储，也即全面存储计算分离。盘古 

    ![](/public/upload/architecture/pangu.jpeg)

## 上云架构未来演进：云原生

**基础设施和业务的边界应该在哪里？真正的判断标准是业务关注的边界而非架构边界，基础设施应无须业务持续关注和维护**。例如，使用了容器服务，是否能让业务无须关注底层资源？也未必，取决于资源的弹性能力是否有充分的支持（否则业务仍需时刻关注流量和资源压力），也取决于可观测性（否则问题的排查仍需关注底层环境）的能力。

1. 节点托管的 K8s 服务
2. Service Mesh 化
3. 应用和基础设施全面解耦
4. 应用交付的标准化：OAM

解耦前

![](/public/upload/architecture/application_infrastructure_coupling.jpeg)

解耦后

![](/public/upload/architecture/application_infrastructure_decoupling.jpeg)

通过上云，最大化的使阿里巴巴的业务使用云的技术，通过技术架构的演进使业务更好的聚焦于自身的发展而无须关于通用底层技术，使业务研发效率提升，迭代速度更快是技术人的真正目标。PS：笔者的感受就是，有一阵鼓吹DevOps，代码上线之前评估流量、压测、上下游服务是否受得了等。从另一个视角看，我负责开发一个服务，那么除了我这个服务，**其它所有的一切计算、存储、上下游服务对我来说都是基础设施**，都可以假定是可用的。

## 云原生如何落地

当我们回顾云计算的发展历程，会看到**基础架构**经历了从物理机到虚拟机，从虚 拟机再到容器的演进过程。在这大势之下，**应用架构**也在同步演进，从单体过渡到多层，再到当下的微服务。在变化的背后，有一股持续的动力，它来自于三个不变的追求:

1. 提高资源利用率
2. 优化开发运维体验
3. 以及更好地支持业务发展。

[对话阿里云叔同：释放云价值，让容器成为“普适”技术](https://mp.weixin.qq.com/s/cIdQY4mWpAtDagEuvmH_1A)云原生正在通过方法论、工具集和理念重塑整个软件技术栈和生命周期

1. 过去我们常以虚拟化作为云平台和与客户交互的界面，为企业带来灵活性的同时也带来一定的管理复杂度；
2. 容器的出现，在虚拟化的基础上向上封装了一层，逐步成为云平台和与客户交互的新界面之一，应用的构建、分发和交付得以在这个层面上实现标准化，大幅降低了企业 IT 实施和运维成本，提升了业务创新的效率。从技术发展的维度看，开源让云计算变得越来越标准化，容器已经成为应用分发和交付的标准，可以将应用与底层运行环境解耦；3. Kubernetes成为资源调度和编排的标准，屏蔽了底层架构的差异性，帮助应用平滑运行在不同的基础设施上；
4. 在此基础上建立的上层应用抽象如微服务和服务网格，逐步形成应用架构现代化演进的标准，开发者只需要关注自身的业务逻辑，无需关注底层实现

### Kubernetes的价值

![](/public/upload/kubernetes/application_delivery.jpg)

docker 让镜像和容器融合在一起，`docker run` 扣动扳机，实现镜像到 容器的转变。但`docker run` 仍然有太多要表述的，比如`docker run`的各种参数： 资源、网络、存储等，“一个容器一个服务”本身也不够反应应用的复杂性（或者 说还需要额外的信息 描述容器之间的关系，比如`--link`）。

我们学习linux的时候，知道linux 提供了一种抽象：一切皆文件。没有人会质疑基本单元为何是文件不是磁盘块？linux 还提供了一种抽象叫“进程”，没有人会好奇为何 linux 不让你直接操纵cpu 和 内存。**一个复杂的系统，最终会在某个层面收敛起来**，就好像一个web系统，搞到最后就是一系列object 的crud。类似的，如果我们要实现一个“集群操作系统”，容器的粒度未免太小。也就是说，镜像和容器之间仍然有鸿沟要去填平？kubernetes 叫pod，marathon叫Application，中文名统一叫“应用”。在这里，应用是一组容器的有机组 合，同时也包括了应用运行所需的网络、存储的需求的描述。而像这样一个“描述” 应用的 YAML 文件，放在 etcd 里存起来，然后通过控制器模型驱动整个基础设施的 状态不断地向用户声明的状态逼近，就是 Kubernetes 的核心工作原理了。“有了 Pod 和容 器设计模式，我们的**应用基础设施才能够与应用(而不是容器)进行交互和响应的能力**，实现了“云”与“应用”的直接对接。而有了声明式 API，我们的应用基础而设施才能真正同下层资源、调度、编排、网络、存储等云的细节与逻辑解耦”。PS：笔者最近在团队管理上也有一些困惑， 其实，你直接告诉小伙伴怎么做并不是一个很好的方式，他们不自由，你也很心累。比较好的方式是做好目标管理，由他们自己不断去填平目标与现实的鸿沟。


阿里巴巴落地 Kubernetes 可以分为三个阶段:

1. 首先**通过 Kubernetes 提供资源 供给**，但是不过多干扰运维流程，这系统容器是富容器，将镜像标准化与轻量级虚拟 化能力带给了上面的 PaaS 平台。
2. 通过 Kubernetes controller 的形式改造PaaS 平台的运维流程，给 PaaS 带来更强的**面向终态**的自动化能力。
3. 把运行 环境等传统重模式改成原生容器与 pod 的轻量模式，同时将 PaaS 能力完全移交给 Kubernetes controller，从而形成一个完全云原生的架构体系。

演进中的挑战

1. Kubernetes 本身的管理，节点发布回滚策略，按规则要求灰度发布;将环境进行镜像切分，分为模拟环境和生产环境;并且在监控侧下足功夫，将 Kubernetes 变得更白盒化和透明化，及早发现问题、预防问题、解决问题。
2.  Kubernetes 的多租户管理

    ![](/public/upload/kubernetes/kubernetes_multi_tenancy.png)

3. 在web级集群中动态调整Pod资源限制 https://github.com/openkruise
4. 大规模k8s集群下的巡检 k8s集群运维通过巡检走向AIOPS

### Serverless

相 比容器技术，Serverless 可以将资源管理的粒度更加细化，使开发者更快上手云原 生，并且倡导**事件驱动模型支持业务发展**。从而帮助用户解决了资源管理复杂、低频业务资源占用等问题;实现**面向资源使用**，以取代**面向资源分配**的模式。

[Serverless 的喧哗与骚动（一）：Serverless 行业发展简史](https://www.infoq.cn/article/SXv6xredWW03P7NXaJ4m)

## 向阿里学习什么？

1. 一切问题 都可以变着法儿用技术问题去解决。
2. 就一个故障恢复，想不透的只能空口抱怨，若是想透了， 拉一个部门这个活儿都干不完。

![](/public/upload/kubernetes/fault_analysis_handle.png)