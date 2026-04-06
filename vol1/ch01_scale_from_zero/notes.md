# Ch1: Scale From Zero To Millions Of Users

## 演进路径 / Evolution Path
单服务器 → Web/DB分离 → LB+主从复制 → Cache+CDN → 无状态Web → 多数据中心 → 消息队列 → DB分片

Single Server → Separate Web/DB → LB + Master-Slave Replication → Cache + CDN → Stateless Web → Multi Data Center → Message Queue → DB Sharding

---

## 单服务器 / Single Server
DNS解析域名→返回IP→HTTP请求→Web Server返回HTML/JSON

DNS resolves domain → returns IP → HTTP request → Web Server returns HTML/JSON

---

## DB选型：何时用 NoSQL / When to Use NoSQL
- 超低延迟 / Super-low latency
- 非结构化数据 / Unstructured data
- 只需序列化/反序列化（存进去取出来，无需JOIN）/ Only need to serialize/deserialize data (store and retrieve, no JOIN needed)
- 海量数据 / Massive amount of data

---

## 垂直 vs 水平扩展 / Vertical vs Horizontal Scaling
- 垂直（Scale Up）：加CPU/RAM，有硬件上限，有单点故障
  - Vertical: add CPU/RAM, has hardware limit, has single point of failure
  - 例 / e.g.: Amazon RDS 最高 24TB RAM；StackOverflow 2013年 1000万月活仅用 1台 Master DB
- 水平（Scale Out）：加机器，无上限，有冗余
  - Horizontal: add servers, no hard limit, built-in redundancy

---

## 负载均衡 / Load Balancer
- 用户访问 LB 公网IP → LB 用**私网IP**与 Web Server 通信（Web Server不暴露公网）
- Users connect to LB's public IP → LB communicates with Web Servers via **private IP** (Web Servers not exposed to internet)
- 解决 / Solves: 单点故障 + 流量峰值 / single point of failure + traffic spikes

---

## 数据库复制 / Database Replication（主从 / Master-Slave）
- Master：只写 / writes only；Slave：只读 / reads only（读远多于写，Slave通常多个 / reads >> writes, usually multiple Slaves）
- Slave宕 / Slave down：读临时打Master，新Slave替代 / reads go to Master temporarily, new Slave replaces it
- Master宕 / Master down：Slave晋升为Master（需数据恢复脚本补齐落后数据）/ Slave promoted to Master (may need data recovery scripts to catch up)

---

## 缓存 / Cache
**Read-through**：查缓存→未命中→查DB→写缓存→返回 / Check cache → miss → query DB → write to cache → return

关键注意点 / Key considerations:
- 读多写少才用缓存；重要数据不能只存缓存（重启丢失）/ Use cache for read-heavy data; don't store critical data only in cache (lost on restart)
- 过期时间 / Expiration: 太短频繁回源 / too short = frequent DB calls；太长数据陈旧 / too long = stale data
- 一致性 / Consistency: DB和缓存更新非原子，跨region尤难（见Facebook Scaling Memcache）/ DB and cache updates are not atomic, especially hard across regions
- 避免SPOF / Avoid SPOF: 多节点Cache / multiple cache nodes
- 淘汰策略 / Eviction policy: **LRU**（最常用 / most common）/ LFU / FIFO

---

## CDN
缓存**静态资源** / Caches **static content**（JS/CSS/图片/视频 / images/videos），就近返回 / served from nearest edge node

- 回源 / Cache miss: CDN无缓存→拉Origin→按TTL缓存 / fetch from Origin → cache by TTL
- 更新文件 / Invalidation: API使对象失效 / API invalidation 或 URL加版本号 / URL versioning（`image.png?v=2`）
- 低频资源不要放CDN（按流量计费）/ Don't cache infrequently used assets (charged by data transfer)

---

## 无状态 Web 层 / Stateless Web Tier
- 有状态问题 / Stateful problem: Session存在某台Server上，必须Sticky Session，难扩展 / Session tied to one server, requires sticky sessions, hard to scale
- 无状态方案 / Stateless solution: Session存**共享存储**（Redis/NoSQL，优先NoSQL因为易于扩展）/ Store session in **shared storage** (Redis/NoSQL, prefer NoSQL for easy scaling)，任意Server可处理任意请求，支持Auto-scaling / any server can handle any request, enables auto-scaling

---

## 多数据中心 / Multi Data Center
- **GeoDNS**：按用户地理位置路由到最近DC / routes users to nearest data center based on location
- 某DC故障→100%流量切健康DC / DC failure → 100% traffic to healthy DC
- 三大挑战 / 3 challenges:
  - 流量调度 / Traffic redirection：GeoDNS
  - 数据同步 / Data sync：跨DC异步复制 / async replication across DCs（参考Netflix方案 / ref: Netflix）
  - 自动化部署 / Automated deployment：保证各DC一致 / keep services consistent across DCs

---

## 消息队列 / Message Queue
Producer → [Queue] → Consumer，异步解耦 / async decoupling

- Producer/Consumer可独立扩展 / scale independently
- 队列积压→加Consumer / queue backlog → add Consumers；队列空→减Consumer / queue empty → remove Consumers

---

## 日志/监控/自动化 / Logging / Metrics / Automation
- Logging：集中汇聚错误日志 / centralized error log aggregation
- Metrics：Host级（CPU/内存/IO）+ 服务层级 / service-level + 业务指标（DAU/收入）/ business metrics
- Automation：CI/CD，自动构建/测试/部署 / automated build, test, deploy

---

## 数据库分片 / Database Sharding
数据库水平扩展，按**Sharding Key** hash路由到对应Shard（`user_id % N`）

Horizontal DB scaling: route data to shards via **Sharding Key** hash (`user_id % N`)

三大挑战 / 3 challenges:

| 问题 / Problem | 解法 / Solution |
|------|------|
| Resharding（数据增长/分布不均 / data growth or uneven distribution） | 一致性哈希 / Consistent Hashing（Ch5） |
| Celebrity热点（单Shard过载 / single shard overloaded） | 为热点单独分Shard / dedicate a shard per celebrity |
| 跨Shard无法JOIN / Cross-shard JOIN not supported | 反范式化 / Denormalization（数据冗余 / data redundancy） |

选Sharding Key核心标准：**数据分布均匀** / Key criterion: **evenly distributed data**

分片后部分非关系型功能可迁移到 NoSQL，进一步减轻关系型 DB 负载
After sharding, some non-relational functionality can move to NoSQL to further reduce load on the relational DB

---

## 总结原则 / Summary
1. Web层无状态 / Keep web tier stateless
2. 每层冗余 / Build redundancy at every tier
3. 尽量多缓存 / Cache data as much as you can
4. 多数据中心 / Support multiple data centers
5. 静态资源CDN / Host static assets in CDN
6. 数据库分片 / Scale data tier by sharding
7. 拆分服务独立扩展 / Split tiers into individual services
8. 监控+自动化 / Monitor your system and use automation tools
