# Ch1: Scale From Zero To Millions Of Users

## 演进路径
单服务器 → Web/DB分离 → LB+主从复制 → Cache+CDN → 无状态Web → 多数据中心 → 消息队列 → DB分片

---

## 单服务器
DNS解析域名→返回IP→HTTP请求→Web Server返回HTML/JSON

---

## DB选型：何时用 NoSQL
- 超低延迟
- 非结构化数据
- 只需序列化/反序列化
- 海量数据

---

## 垂直 vs 水平扩展
- 垂直：加CPU/RAM，有硬件上限，有单点故障
  - 例：Amazon RDS 最高 24TB RAM；StackOverflow 2013年 1000万月活仅用 1台 Master DB
- 水平：加机器，无上限，有冗余

---

## 负载均衡
- 用户访问 LB 公网IP → LB 用**私网IP**与 Web Server 通信（Web Server不暴露公网）
- 解决：单点故障 + 流量峰值

---

## 数据库复制（主从）
- Master：只写；Slave：只读（读远多于写，Slave通常多个）
- Slave宕：读临时打Master，新Slave替代
- Master宕：Slave晋升为Master（需数据恢复脚本补齐落后数据）

---

## 缓存
**Read-through**：查缓存→未命中→查DB→写缓存→返回

关键注意点：
- 读多写少才用缓存；重要数据不能只存缓存（重启丢失）
- 过期时间：太短频繁回源；太长数据陈旧
- 一致性：DB和缓存更新非原子，跨region尤难（见Facebook Scaling Memcache）
- 避免SPOF：多节点Cache
- 淘汰策略：**LRU**（最常用）/ LFU / FIFO

---

## CDN
缓存**静态资源**（JS/CSS/图片/视频），就近返回。

- 回源：CDN无缓存→拉Origin→按TTL缓存
- 更新文件：API使对象失效 或 URL加版本号（`image.png?v=2`）
- 低频资源不要放CDN（按流量计费）

---

## 无状态 Web 层
- 有状态问题：Session存在某台Server上，必须Sticky Session，难扩展
- 无状态方案：Session存**共享存储**（Redis/NoSQL，优先 NoSQL 因为易于扩展），任意Server可处理任意请求，支持Auto-scaling

---

## 多数据中心
- **GeoDNS**：按用户地理位置路由到最近DC
- 某DC故障→100%流量切健康DC
- 三大挑战：流量调度（GeoDNS）/ 数据同步（跨DC异步复制）/ 自动化部署（保证各DC一致）

---

## 消息队列
Producer → [Queue] → Consumer，异步解耦

- Producer/Consumer可独立扩展
- 队列积压→加Consumer；队列空→减Consumer

---

## 日志/监控/自动化
- Logging：集中汇聚错误日志
- Metrics：Host级（CPU/内存/IO）+ 服务层级 + 业务指标（DAU/收入）
- Automation：CI/CD

---

## 数据库分片（Sharding）
数据库水平扩展，按**Sharding Key** hash路由到对应Shard（`user_id % N`）

三大挑战：

| 问题 | 解法 |
|------|------|
| Resharding（数据增长/分布不均） | 一致性哈希（Ch5） |
| Celebrity热点（单Shard过载） | 为热点单独分Shard |
| 跨Shard无法JOIN | 反范式化（数据冗余） |

选Sharding Key核心标准：**数据分布均匀**

分片后部分非关系型功能可迁移到 NoSQL，进一步减轻关系型 DB 负载

---

## 总结原则
1. Web层无状态
2. 每层冗余
3. 尽量多缓存
4. 多数据中心
5. 静态资源CDN
6. 数据库分片
7. 拆分服务独立扩展
8. 监控+自动化
