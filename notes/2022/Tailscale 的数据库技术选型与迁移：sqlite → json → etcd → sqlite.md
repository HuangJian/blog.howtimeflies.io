# Tailscale 的数据库技术选型与迁移：sqlite → json → etcd → sqlite

## 原文链接
- [An unlikely database migration](https://tailscale.com/blog/an-unlikely-database-migration/) 2021-01-13
- [A database for 2022](https://tailscale.com/blog/database-for-2022/) 2022-04-01

## 要点
> - 数据库需求：
>   - Tailscale 是一个 VPN 工具
>   - 保证 VPN 网络的配置数据在客户端节点上保持同步 (所有客户端本地都有一个 **微型数据库** )
>   - 「文中未提及，我猜想的」：服务端 (coordination server) 的数据库类型与客户端保持一致，减少技术栈复杂度
>
> - 技术目标：
>    - 保证开发迭代速度：本地跑单元测试飞快，开发环境与生产环境一致
>       > recompile and run locally in a fraction of a second and deploy ten times a day
>
>   - 不选择 mysql 或者 postgres
>       > It requires Docker on your machine and it’s not particularly fast.
>       >
>       > several of us on the team have operational experience running these databases and didn’t enjoy the prospect of wrestling with the ops overhead of making live replication work and behave well.
>
> - 早期原型：sqlite
>   - 优势：符合最佳实践 - sqlite 是客户端数据库的首选
>   - 劣势：频繁修改数据模型需要敲大量的 ORM 代码，迭代慢
>
> - 迁移⓪(上线伊始)： sqlite → json
>   - 优势：实现简单 (~20行 go 代码)；便于测试；可满足项目原型需求(数据量少时)
>   - 劣势：文件体积增长后(150M+)，全文件回写操作耗时严重(> 1s)
>   - 过程：20行 go 代码搞定，逻辑清晰，迭代快速。
>
> - 迁移①：json → etcd
>   - 场景：核心数据模型非常契合键值数据库的结构；数据量少，能全部载入内存；读操作远多于写操作。
>   - 优势：
>       - 没看懂……
>           > breaking the BigLock into something more akin to a `sync.RWMutex`
>       - 只需要持久化被修改的数据(写操作缩短到毫秒级)
>       - 客户端间的数据同步，正好契合 etcd 的核心能力
>   - 劣势：
>       - etcd 没有索引功能，有数据量限制，有写操作事务频率限制，……
>       - 作者写了一些库来绕过这些限制；于是陷入另一个问题 - 这些代码别人不会看不敢改。
>   - 过程：
>       > We ran both systems in parallel for a while and at some point stopped using the old one
>
> - 迁移②：etcd → sqlite
>   - 优势：
>       - 准实时数据备份和重放 (litestream)
>       - 避免魔改(补丁)第三方产品，减少产品维护的技术复杂度
>   - 劣势：未找到 sqlite(litestream) 的只读副本方案，可能限制将来的能力扩展。
>   - 过程：
>       1. 先移植频繁写数据到 sqlite，核心数据和「冗余」分别存储到不同的 sqlite 库；
>       1. 创建一个表模拟键值型数据库：双写 → 单读 → 关闭 etcd；
>       1. 从大文档表里拆分出细化的数据模型，分别建表。

## 思辨：
### 关于docker：
- 文章里似乎仅分析了服务端的数据库迁移。如果不涉及客户端，难以理解为何要坚持不引入 docker。
- docker 是当前开发环境的标准配置之一，类似于 jdk/node/python、数据库；它能保证开发／测试／演示／生产环境的一致性。
- docker 启动 mysql 和 postgres 容器，确实比起进程内直接调用 sqlite 库操作要慢，但是每次 10s 以内的容器启动延迟还是可以接受的吧。
    - 所以我的项目里，单元测试里都引入了 testcontainers，用于测试数据库操作。
    - 2020年的数据：mysql 8 启动约耗时 8s， postgres 11 启动约耗时 3s。均未做深度调研未做极限优化。
    - docker 容器使用文件内存映射，读写速度飞快。
- 客户端环境下，docker 过于重量级不可用；也许 web assembly 可以成为一个候选项(脱离浏览器环境的 wasm 运行时体积有多大？）。

### 关于技术选型：
- 技术选型若只着眼于短期需求与快速出货，当项目增长时就难免要做技术迁移。这是一个耗时多风险高的过程。
- 大众的主流的技术，维护成本更低：更活跃的技术社区、更庞大的人才市场。
- 有时做技术选型时就很清楚这不是长久之计，预计会在「不远的将来」修订；实际在「很久的将来」里，永远有优先级更高的任务要完成；如果产品不出现严重故障，这一修订大概率不会发生。
    > Never underestimate how long your “temporary” hack will stay in production!
- 之所以推崇最佳实践，它们是业界反复踩坑验证出来的可行方案；很多时候自作聪明走捷径再打补丁，只不过是再踩一次坑。
- 所以，客户端数据库选型还是上 sqlite 吧，服务端传统 CRUD 应用的数据库还是选 mysql / postgres 吧。

### 关于数据研发：
- Tailscale 的反复技术迁移看起来似乎走了不少「弯路」，实际上也符合软件项目的数据研发规律：
    - 启动阶段的数据模型变更频繁，选择 nosql 数据库能更快更灵活地响应变化。
    - 稳定阶段，数据量剧增，对数据的应用和探索更多。标准 sql 是团队公共能力，关系性数据库更能支持数据应用。
    - 成熟阶段，数据量爆炸，对数据价值的挖掘提上日程。需要把数据分发到不同的大数据工具(hadoop、es、clickhouse、tidb)处理。

### 关于 ORM：
- CRUD 应用的 ORM 代码确实是很繁琐的体力活，不管是手写 ORM，还是使用模板自动从数据库模型生成。
- 这就是为什么我很推崇 [postgraphi](https://www.graphile.org/postgraphile/)：只需要定义好数据模型，从 ORM 到 API 都搞定了，也预留了代码优化的空间。
