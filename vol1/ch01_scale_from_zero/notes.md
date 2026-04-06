# Ch1: Scale From Zero To Millions Of Users

## Evolution Path / 演进路径
Single Server → Separate Web/DB → LB + Master-Slave Replication → Cache + CDN → Stateless Web → Multi Data Center → Message Queue → DB Sharding

单服务器 → Web/DB分离 → LB+主从复制 → Cache+CDN → 无状态Web → 多数据中心 → 消息队列 → DB分片

---

## Single Server / 单服务器
DNS resolves domain → returns IP → HTTP request → Web Server returns HTML/JSON

DNS解析域名→返回IP→HTTP请求→Web Server返回HTML/JSON

---

## When to Use NoSQL / 何时用 NoSQL
- Super-low latency 超低延迟
- Unstructured data 非结构化数据
- Only need to serialize/deserialize (store and retrieve, no JOIN needed) 只需序列化/反序列化，无需JOIN
- Massive amount of data 海量数据

Otherwise, relational DB is the default (40+ years of battle-tested reliability, supports JOIN).
其余情况优先关系型数据库。

---

## Vertical vs Horizontal Scaling / 垂直 vs 水平扩展

| | Vertical (Scale Up) | Horizontal (Scale Out) |
|--|--|--|
| How 方式 | Add CPU/RAM | Add servers 加机器 |
| Limit 上限 | Hardware ceiling 有硬件上限 | No hard limit 无上限 |
| Failure 故障 | Single point of failure 单点故障 | Redundant 有冗余 |

- e.g. Amazon RDS up to 24TB RAM; StackOverflow (2013) served 10M monthly users with just 1 Master DB
- 例：Amazon RDS 最高 24TB RAM；StackOverflow 2013年 1000万月活仅用 1台 Master DB

---

## Load Balancer / 负载均衡
- Users connect to LB's **public IP** → LB communicates with Web Servers via **private IP** (Web Servers not exposed to internet)
- 用户访问 LB 公网IP → LB 用**私网IP**与 Web Server 通信（Web Server不暴露公网）
- Solves: single point of failure + traffic spikes 解决单点故障和流量峰值

---

## Database Replication / 数据库复制（Master-Slave / 主从）
- **Master**: writes only 只写；**Slave**: reads only 只读（reads >> writes, usually multiple Slaves 读远多于写，Slave通常多个）
- Slave down 宕机: reads go to Master temporarily, new Slave replaces it 读临时打Master，新Slave替代
- Master down 宕机: Slave promoted to Master — may need data recovery scripts to catch up on missing data
  Slave晋升为Master，需数据恢复脚本补齐落后数据

**Advantages 优点:** better performance (parallel reads/writes), reliability (multi-location backup), high availability

---

## Cache / 缓存

**Read-through strategy 策略:**
Check cache → miss → query DB → write to cache → return
查缓存 → 未命中 → 查DB → 写缓存 → 返回

**Key considerations / 关键注意点:**

| Consideration | Detail |
|------|------|
| When to use 何时用 | Read-heavy data; don't store critical data only in cache (lost on restart) 重要数据不能只存缓存，重启丢失 |
| Expiration 过期时间 | Too short = frequent DB calls; too long = stale data 太短频繁回源；太长数据陈旧 |
| Consistency 一致性 | DB and cache updates are not atomic, especially hard across regions 跨region尤难（ref: Facebook Scaling Memcache） |
| Avoid SPOF | Multiple cache nodes across data centers 多节点Cache |
| Eviction policy 淘汰策略 | **LRU** (most common 最常用) / LFU / FIFO |

---

## CDN

A network of geographically dispersed **edge servers** that cache static content and serve it from the location closest to the user.
全球分布的**边缘节点**网络，缓存静态资源并就近返回。

**What CDN caches / 缓存什么:**

| Type | Examples | CDN? |
|------|----------|------|
| Static assets 静态资源 | JS, CSS, images, videos, fonts | ✓ |
| Dynamic content 动态内容 | User pages, search results | ✗ (usually) |
| API responses | `/api/user/123` | ✗ (usually) |

**Request flow / 请求流程:**
```
First request (cache miss 未命中):
User → CDN edge (no cache) → Origin (S3/Web Server)
     ← Origin returns file + TTL header
     ← CDN caches file, returns to user

Subsequent requests (cache hit 命中, TTL not expired):
User → CDN edge → returns directly, Origin never involved
```

**TTL:**
- Too short → frequent origin fetches, CDN ineffective 频繁回源，CDN形同虚设
- Too long → stale content after updates 内容更新后用户拿到旧版本
- Time-sensitive content (news): short TTL; Rarely changed (logo, fonts): long TTL

**File invalidation / 文件更新两种方式:**
- **URL versioning** (preferred 推荐): `app.js?v=1` → `app.js?v=2` — CDN treats them as different resources, old cache naturally expires
- **API invalidation**: call CDN vendor API to force-delete a cached object — has cost and rate limits

**CDN fallback / 故障兜底:**
Client detects load failure → retries from Origin URL. CDN outage should not take down the site.
客户端检测资源加载失败后降级回源，CDN故障不应导致网站不可用。

**Cost / 成本:**
Charged by data transfer out. Don't put infrequently accessed assets in CDN — no benefit, extra cost.
按出流量计费，低频资源放CDN没意义反而增加成本。

---

## Stateless Web Tier / 无状态 Web 层
- **Stateful problem 有状态问题:** Session tied to one server → requires Sticky Sessions → hard to scale, hard to handle failures
  Session存在某台Server上，必须Sticky Session，难扩展
- **Stateless solution 无状态方案:** Store session in **shared storage** (Redis/NoSQL — prefer NoSQL for easy scaling) → any server can handle any request → enables Auto-scaling
  Session存共享存储（优先NoSQL因为易于扩展），任意Server可处理任意请求，支持Auto-scaling

---

## Multi Data Center / 多数据中心
- **GeoDNS:** resolves domain to IP based on user's geographic location, routing to nearest DC
  按用户地理位置将域名解析到最近DC
- DC failure → 100% traffic rerouted to healthy DC 某DC故障→流量全切健康DC

**3 technical challenges / 三大挑战:**

| Challenge | Solution |
|------|------|
| Traffic redirection 流量调度 | GeoDNS |
| Data sync 数据同步 (different regions may have different data) | Async replication across DCs 跨DC异步复制 (ref: Netflix) |
| Test & deployment 测试与部署 | Automated deployment tools CI/CD 自动化部署 |

---

## Message Queue / 消息队列
`Producer → [Queue] → Consumer` — async decoupling 异步解耦

- Producer and Consumer **scale independently 独立扩展**
- Queue backlog → add Consumers 队列积压加Consumer; Queue empty → remove Consumers 队列空减Consumer
- Example: Web Server publishes photo processing jobs → Photo Workers consume asynchronously
  Web Server发图片处理任务→Photo Worker异步消费

---

## Logging / Metrics / Automation / 日志/监控/自动化
- **Logging:** centralized error log aggregation 集中汇聚错误日志
- **Metrics:**
  - Host level: CPU, Memory, disk I/O
  - Service level: DB tier, Cache tier performance
  - Business metrics: DAU, retention, revenue 日活/留存/收入
- **Automation:** CI/CD — automated build, test, deploy 自动构建/测试/部署

---

## Database Sharding / 数据库分片
Horizontal DB scaling: split data across shards using a **Sharding Key** + hash function (`user_id % N`)
数据库水平扩展，按Sharding Key hash路由到对应Shard

**Key criterion for Sharding Key: evenly distributed data 数据分布均匀**

**3 challenges / 三大挑战:**

| Problem | Description | Solution |
|------|------|------|
| Resharding | Data growth or uneven distribution 数据增长/分布不均 | Consistent Hashing 一致性哈希 (Ch5) |
| Celebrity / Hotspot | Single shard overloaded 单Shard过载 | Dedicate a shard per celebrity 为热点单独分Shard |
| Cross-shard JOIN | Data spread across machines, JOIN not supported 跨Shard无法JOIN | Denormalization 反范式化 (data redundancy 数据冗余) |

After sharding, non-relational functionality can move to NoSQL to further reduce relational DB load.
分片后部分非关系型功能可迁移到NoSQL，进一步减轻关系型DB负载。

---

## Summary / 总结
1. Keep web tier **stateless** 无状态
2. Build **redundancy** at every tier 每层冗余
3. **Cache** data as much as you can 尽量多缓存
4. Support **multiple data centers** 多数据中心
5. Host static assets in **CDN** 静态资源CDN
6. Scale data tier by **sharding** 数据库分片
7. Split tiers into **individual services** 拆分服务独立扩展
8. **Monitor** your system and use **automation** tools 监控+自动化
