# 基础聊天

添加子模块 git submodule add https://github.com/apache/ignite vendors/ignite

检测标签 2.17.0，代码分支切换到该标签

## 探索

ignite 支持哪些 DDL 操作？怎么使用？API入口在哪？

以 IgniteCache.query(new SqlFieldsQuery(sql)) 为起点，研究 ignite 如何实现 CREATE TABLE 操作，梳理出完整的执行流程，并将流程进行大致的划分：客户端、服务端、数据存储层、网络通信层、分布式机制层，流程分析要有理有据，不能脱离源码。最终将研究结果写到 @docs-research 路径下。

深入分析 ignite 的数据存储层设计，拆解出有哪些核心概念、关键抽象、核心模块，关键设计，各自在解决什么问题，各个模块之间是什么关系，相互之间是怎么协作的。我是资深java工程师，但是之前没写过存储，所以要求详实、易懂，要有各种架构图、流程图等各种图表帮助我理解。结果写到 @docs-research 路径下。

将该项目的代码上传到 github 仓库，仓库地址为 https://github.com/XianReallyHot-ZZH/Ignite-Research.git


## 存储层探索

当前我在探索 ignite 存储层，docs-research/03-ignite-storage-layer.md 是前期对 ignite 存储层设计的探索结果，但是对我来说该文档过于深奥，理解曲线困难。
所以以下是我的提示词，用来指导 AI 进行进一步的拆解，来帮助我探索 ignite 存储层。提示词：

docs-research/03-ignite-storage-layer.md 是前期对 ignite 存储层设计的探索结果，但是该文档对于一个初学者来说，过于深奥，理解曲线困难。
现在要求你对 ignite 存储层进行深度的拆解，要求：
- 一开始不要融入过多的代码，要从原理出发，讲透原理了才能更好的理解代码实现
- 拆解的足够细致，每一个点的理解对读者来说要清晰、没有负担
- 每一个点要讲清楚，在解决什么问题，解决方案是什么，为什么这么设计
- 拆解的要足够完整，最终要对应到 ignite 存储层的完整实现
- 循序渐进，各个讲解的要点要像长出来的一样

请你对以上提示词进行评估，有没有更好的建议？有没有优化空间？



基于源码，你先检视一遍 docs-research/03-ignite-storage-layer.md 有没有明显的错误，内容是不是合理、完整、准确，有没有优化的空间，先把这个纲领性（天花板）的文档做到满分。

