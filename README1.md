# Mongodb 优化策略
*翻译自Mongodb官网：[Optimization Strategies for MongoDB](http://docs.mongodb.org/manual/administration/optimization/)*

很多因素都会影响到数据库的性能，包括索引的使用，查询结构，数据模型，应用的设计以及具体操作方面的原因，例如架构和系统设置。
这一节我们会描述优化mongodb应用性能的一些手段。

###评估当前操作（Current Operations）的性能
MongoDB 提供了可以描述执行中查询的反查工具，让用户可以测试查询和构建更有效率的查询。

###为快速读写使用固定集合(Capped Collection)
简述一个使用固定集合的用例，它是如何优化一个指定的数据获取工作流程的。

###优化查询性能
介绍使用Projections来减少Mongdb发送给客户端的数据大小。

###设计要点
关于架构，设计和管理基于Mongodb的的应用


