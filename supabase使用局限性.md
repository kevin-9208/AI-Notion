如果你是个人开发者、小团队，或者正在用 Vue3 + Supabase 做 SaaS、旅游规划系统、后台管理系统，那么 **Supabase 确实是目前最好用的 BaaS（Backend as a Service）之一**。

但是很多人用了一段时间后，会发现：

> Supabase 很强，但它并不是万能后端。

很多项目在 MVP 阶段非常爽，到了用户量增长、业务复杂化之后，就会逐渐暴露一些局限性。

我从架构、数据库、性能、安全、成本、运维、开发体验等多个维度展开分析。

---

# 一、Supabase 本质是什么？

很多人理解错了。

Supabase ≠ Firebase

实际上：

```text
Supabase
├── PostgreSQL
├── Auth
├── Storage
├── Realtime
├── Edge Functions
├── API Gateway
└── Dashboard
```

本质上：

> Supabase = PostgreSQL + 一堆云服务封装

所以：

优点来自 PostgreSQL

缺点也来自 PostgreSQL

---

# 二、最大的局限：不是真正的业务后端

很多新手认为：

```text
Vue
↓
Supabase
↓
数据库
```

就能做所有系统。

实际上：

```text
Vue
↓
NestJS / SpringBoot
↓
Supabase(Postgres)
```

才是大型项目常见架构。

因为：

Supabase提供：

✅ 数据存储

✅ 用户认证

✅ 文件存储

❌ 复杂业务逻辑

❌ 工作流

❌ DDD

❌ 微服务

❌ MQ

---

举例：

旅游系统

用户下单

↓

扣库存

↓

生成订单

↓

发优惠券

↓

发邮件

↓

通知供应商

↓

写日志

↓

推送消息

---

这些业务链路：

```text
事务
事件
补偿
重试
消息队列
```

Supabase并不擅长。

你最终还是需要：

* NestJS
* Go
* Java

来实现。

---

# 三、复杂权限会越来越难维护

Supabase核心权限：

RLS

(Row Level Security)

例如：

```sql
CREATE POLICY "user own trip"

ON trips

FOR SELECT

USING (
    auth.uid() = user_id
);
```

很优雅。

---

但项目大了：

```text
超级管理员

区域管理员

运营人员

普通用户

供应商

代理商
```

开始出现：

```text
角色
组织
部门
租户
团队
项目
```

权限组合爆炸。

---

你会写出：

```sql
EXISTS(...)
AND EXISTS(...)
AND (...)
```

几十行的 Policy。

最后：

自己都看不懂。

---

企业项目经常出现：

```text
RBAC
ABAC
租户隔离
组织架构
```

此时：

Supabase RLS维护成本会快速上升。

---

# 四、Realtime并不适合高频场景

很多人看到：

```text
Realtime
```

以为：

```text
聊天
直播
协同编辑
在线游戏
```

都能做。

实际上不是。

---

Supabase Realtime底层：

```text
Postgres WAL
↓
Realtime Server
↓
WebSocket
```

监听数据库变更。

---

优点：

简单。

---

缺点：

数据库变更驱动。

---

例如：

聊天室

10000人

每秒100条消息

---

数据库：

```text
INSERT
INSERT
INSERT
INSERT
```

持续刷。

Postgres压力巨大。

---

真正大型聊天系统：

```text
Kafka
Redis
RabbitMQ
WebSocket Gateway
```

不会直接靠数据库驱动。

---

所以：

Supabase Realtime：

适合：

✅ 通知

✅ 订单状态

✅ 协同看板

✅ 后台管理

---

不适合：

❌ 微信

❌ Discord

❌ 飞书

❌ 在线游戏

---

# 五、大数据量性能问题

很多人开发阶段：

```sql
SELECT *
FROM trips
```

几十条记录。

飞快。

---

一年后：

```text
1000万条
5000万条
1亿条
```

问题来了。

---

虽然 PostgreSQL 很强。

但：

Supabase API

↓

PostgREST

↓

PostgreSQL

---

中间有额外开销。

复杂查询性能不如：

```text
NestJS + Prisma
```

自己控制。

---

例如：

复杂报表：

```sql
JOIN
JOIN
GROUP BY
WINDOW
CTE
```

---

这种时候：

API层会变得笨重。

很多团队最终：

```text
Supabase
↓
只存数据

业务API
↓
自己开发
```

---

# 六、数据库迁移能力一般

很多企业使用：

* Flyway
* Liquibase
* Alembic

管理数据库。

---

Supabase也支持 Migration。

但：

体验不如成熟方案。

---

经常出现：

```text
开发环境
测试环境
生产环境
```

Schema不一致。

---

尤其多人开发：

```bash
supabase db push
```

容易冲突。

---

大型团队：

仍然喜欢：

```text
Flyway
Liquibase
```

---

# 七、Vendor Lock-In（平台绑定）

这是很大的问题。

---

如果使用：

```text
Auth
Storage
Realtime
Functions
```

全部依赖Supabase。

未来迁移：

非常痛苦。

---

例如：

Auth

使用：

```javascript
supabase.auth
```

大量代码绑定。

---

未来迁移：

```text
Clerk
Auth0
Keycloak
```

工作量巨大。

---

Storage同理。

---

越依赖Supabase生态：

越难迁移。

---

# 八、Edge Functions能力有限

Supabase Edge Functions 基于：

Deno

---

优点：

部署快。

---

缺点：

冷启动。

运行时间限制。

资源限制。

---

例如：

AI图片生成

视频转码

PDF批量生成

爬虫任务

机器学习

---

都不适合。

---

最终还是：

```text
Docker
Kubernetes
云服务器
```

更自由。

---

# 九、搜索能力比较弱

Supabase搜索：

主要依赖：

PostgreSQL Full Text Search

---

适合：

```text
博客
文章
知识库
```

---

不适合：

```text
淘宝
京东
亚马逊
```

级别搜索。

---

因为缺少：

```text
分词
纠错
同义词
推荐
排序
向量召回
```

等能力。

---

大型项目往往增加：

Elasticsearch

或者

OpenSearch

---

# 十、成本问题（很多人忽略）

初期：

```text
免费
```

非常爽。

---

随着增长：

```text
数据库
存储
流量
Realtime
```

开始收费。

---

尤其：

图片网站

旅游网站

社交网站

---

文件存储费用增长很快。

---

有时：

```text
Supabase
+
Cloudflare R2
```

组合更便宜。

---

# 十一、国内网络环境问题

如果用户主要在中国大陆：

这是必须考虑的。

---

Supabase官方服务主要在海外。

可能出现：

```text
登录慢
上传慢
下载慢
```

---

解决方案：

```text
自建 Supabase
```

或者：

```text
腾讯云
阿里云
```

自建 PostgreSQL。

---

# 十二、什么时候不推荐 Supabase？

如果项目属于：

### 1 小程序社交平台

```text
朋友圈
聊天
动态
直播
```

不推荐。

---

### 2 大型电商

```text
库存
订单
支付
营销
```

不推荐纯Supabase。

---

### 3 企业ERP

```text
复杂权限
组织架构
审批流
```

不推荐。

---

### 4 游戏后台

```text
高并发
实时同步
```

不推荐。

---

# 十三、什么时候特别推荐？

非常适合：

### MVP验证

创业项目

### 个人作品

### 独立开发者

### 管理后台

### CMS

### 博客系统

### AI SaaS

### 工具类网站

### 预约系统

### 旅游规划系统

（你最近正在做的 Vue3 + Supabase 行程规划项目，就属于这一类。）

---

# 一个经验总结

如果按照项目规模划分：

```text
0 ~ 1万用户
```

Supabase 几乎无敌。

开发速度极快。

---

```text
1万 ~ 10万用户
```

开始加入：

NestJS
Redis
消息队列

形成混合架构。

---

```text
10万+ 用户
```

Supabase通常退化成：

PostgreSQL托管服务

Auth和Realtime逐渐被替换。

---

所以很多成熟团队最终会形成这样的架构：

```text
Vue3
    ↓
NestJS
    ↓
Redis
    ↓
Supabase(PostgreSQL)
    ↓
Storage
```

即：

**把 Supabase 当成“云数据库平台”，而不是整个后端。**

这样既能享受 PostgreSQL + Auth + Storage 的开发效率，又不会被 Supabase 的能力边界限制住业务发展。
