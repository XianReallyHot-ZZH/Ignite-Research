# 基础聊天

添加子模块 git submodule add https://github.com/apache/ignite vendors/ignite

检测标签 2.17.0，代码分支切换到该标签

## 探索

ignite 支持哪些 DDL 操作？怎么使用？API入口在哪？

以 IgniteCache.query(new SqlFieldsQuery(sql)) 为起点，研究 ignite 如何实现 CREATE TABLE 操作，梳理出完整的执行流程，并将流程进行大致的划分：客户端、服务端、数据存储层、网络通信层、分布式机制层，流程分析要有理有据，不能脱离源码。最终将研究结果写到 @docs-research 路径下。

深入分析 ignite 的数据存储层设计，拆解出有哪些核心概念、关键抽象、核心模块，关键设计，各自在解决什么问题，各个模块之间是什么关系，相互之间是怎么协作的。我是资深java工程师，但是之前没写过存储，所以要求详实、易懂，要有各种架构图、流程图等各种图表帮助我理解。结果写到 @docs-research 路径下。

将该项目的代码上传到 github 仓库，仓库地址为 https://github.com/XianReallyHot-ZZH/Ignite-Research.git