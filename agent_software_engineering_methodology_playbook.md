# AI Coding Agent 软件工程方法论实践手册

> 面向：
>
> - AI Coding Agent
> - Autonomous Software Agent
> - Code Generation System
> - AI Pair Programming System
> - 自动化软件工程系统
>
> 目标：
>
> 让 Agent 不只是『生成代码』。
>
> 而是：
>
> - 生成可维护系统
> - 控制复杂度
> - 自动治理架构
> - 自动避免屎山
> - 长期可演进
> - 可多人协作
> - 可生产部署

---

# 一、Agent 核心行为准则

# Rule 1：禁止无边界追加代码

禁止：

```text
main.py 不断增长
```

必须：

```text
新能力 = 新模块
```

当出现以下情况时必须拆分：

| 条件 | 动作 |
|---|---|
| 文件 > 500 行 | 自动拆模块 |
| 函数 > 80 行 | 自动拆函数 |
| 类职责 > 1 | 自动拆类 |
| import > 15 | 重构依赖 |
| if嵌套 > 3 | 重构逻辑 |

---

# Rule 2：所有代码必须属于明确层级

禁止：

```python
controller 直接操作数据库
```

必须：

```text
Controller
↓
Application
↓
Domain
↓
Infrastructure
```

---

# Rule 3：禁止业务逻辑与基础设施耦合

禁止：

```python
user.py
直接 import redis/mysql/flask
```

必须：

```text
Domain 不依赖 Infra
```

---

# Rule 4：所有新增功能必须可测试

必须满足：

- 可 mock
- 无全局状态
- Side Effect 可隔离
- IO 可替换

禁止：

```python
业务逻辑直接写文件
业务逻辑直接请求网络
```

---

# Rule 5：优先组合，避免继承爆炸

优先：

```text
Composition
```

避免：

```text
巨型继承树
```

---

# 二、标准工程结构

# Python 标准结构

```text
project/
├── apps/
│   ├── api/
│   ├── worker/
│   └── scheduler/
│
├── domain/
│   ├── user/
│   ├── order/
│   └── payment/
│
├── application/
│   ├── services/
│   ├── commands/
│   └── queries/
│
├── infrastructure/
│   ├── db/
│   ├── cache/
│   ├── mq/
│   └── external/
│
├── interfaces/
│   ├── http/
│   ├── grpc/
│   └── cli/
│
├── tests/
├── scripts/
├── docs/
└── configs/
```

---

# Node.js 标准结构

```text
src/
├── modules/
│   ├── user/
│   ├── auth/
│   └── payment/
│
├── shared/
│   ├── utils/
│   ├── logger/
│   └── errors/
│
├── infrastructure/
├── interfaces/
├── configs/
└── tests/
```

---

# 三、模块设计规范

# Rule 6：模块必须具备边界

每个模块必须包含：

| 组件 | 必须 |
|---|---|
| interface | YES |
| service | YES |
| model | YES |
| test | YES |
| contract | YES |

---

# Rule 7：禁止万能 common/util

禁止：

```text
utils.py
common.py
helpers.py
misc.py
```

超过以下条件必须拆分：

| 条件 | 动作 |
|---|---|
| util > 10函数 | 按领域拆分 |
| common > 300行 | 重新建模 |

---

# Rule 8：模块依赖必须单向

允许：

```text
api → service → repository
```

禁止：

```text
repository → api
```

禁止：

```text
循环依赖
```

---

# 四、文件治理规范

# Rule 9：文件大小治理

| 类型 | 最大行数 |
|---|---|
| controller | 300 |
| service | 400 |
| domain | 300 |
| util | 200 |
| config | 150 |

超过自动触发：

```text
Refactor Pipeline
```

---

# Rule 10：函数复杂度治理

限制：

| 指标 | 最大值 |
|---|---|
| 参数数量 | 5 |
| 嵌套层级 | 3 |
| 函数长度 | 80 |
| return数量 | 3 |

---

# Rule 11：禁止 God Object

出现以下特征必须拆分：

- 超过20个方法
- 超过10个依赖
- 同时处理多个领域
- 同时负责 IO + 业务 + 状态

---

# 五、DDD 实践规则

# Rule 12：必须建立 Bounded Context

禁止：

```text
system_service.py
```

必须：

```text
user/
payment/
order/
inventory/
```

---

# Rule 13：领域对象禁止感知框架

禁止：

```python
from flask import request
```

出现在：

```text
domain/
```

---

# Rule 14：Entity 与 DTO 分离

禁止：

```text
数据库对象直接返回前端
```

必须：

```text
Entity
↓
DTO Mapper
↓
Response
```

---

# 六、依赖治理规范

# Rule 15：所有依赖必须可分析

Agent 必须生成：

```text
dependency graph
```

并检查：

- 循环依赖
- 隐式依赖
- 跨层污染
- 无用依赖

---

# Rule 16：依赖方向必须稳定

核心规则：

```text
高层不依赖低层实现
```

必须：

```text
依赖抽象
```

不是：

```text
依赖具体实现
```

---

# 七、测试方法论

# Rule 17：所有核心逻辑必须可单测

覆盖：

| 类型 | 必须 |
|---|---|
| Domain Logic | YES |
| Application Service | YES |
| Critical Flow | YES |

---

# Rule 18：测试必须隔离 IO

禁止：

```text
单元测试直接连接数据库
```

必须：

```text
mock infra
```

---

# Rule 19：E2E 只验证关键链路

禁止：

```text
所有测试都写 E2E
```

推荐比例：

| 类型 | 比例 |
|---|---|
| Unit | 70% |
| Integration | 20% |
| E2E | 10% |

---

# 八、API 设计规范

# Rule 20：API 必须幂等

尤其：

- 支付
- 创建订单
- 回调
- 重试系统

---

# Rule 21：错误码标准化

禁止：

```json
{
  "error": "something wrong"
}
```

必须：

```json
{
  "code": "USER_NOT_FOUND",
  "message": "user not found"
}
```

---

# Rule 22：禁止接口泄漏内部实现

禁止：

```text
返回数据库字段
```

必须：

```text
返回业务语义
```

---

# 九、数据库方法论

# Rule 23：数据库不是业务模型

禁止：

```text
数据库结构 = 系统结构
```

必须：

```text
领域模型 > 数据模型
```

---

# Rule 24：Repository 负责数据访问

禁止：

```python
service 直接写 SQL
```

---

# Rule 25：禁止 N+1 查询

Agent 必须检查：

- ORM lazy loading
- 循环查询
- 多次重复请求

---

# 十、并发与异步规范

# Rule 26：耗时任务必须异步化

包括：

- 邮件
- 视频处理
- AI推理
- 报表生成

---

# Rule 27：禁止阻塞主线程

Agent 必须识别：

- CPU 密集
- IO 阻塞
- 长事务

---

# Rule 28：必须具备超时机制

所有：

- HTTP
- MQ
- RPC
- DB

必须：

```text
timeout
```

---

# 十一、缓存规范

# Rule 29：缓存必须可失效

禁止：

```text
永久缓存
```

必须：

- TTL
- 主动失效
- 版本控制

---

# Rule 30：防止缓存击穿

必须考虑：

- mutex
- random ttl
- preload
- multi-level cache

---

# 十二、日志规范

# Rule 31：日志必须结构化

禁止：

```python
print("something wrong")
```

必须：

```json
{
  "trace_id": "xxx",
  "user_id": "xxx",
  "event": "payment_success"
}
```

---

# Rule 32：禁止日志泄漏敏感信息

禁止输出：

- password
- token
- api_key
- 身份证
- 银行卡

---

# 十三、安全工程规范

# Rule 33：所有输入必须校验

包括：

- 类型
- 长度
- 格式
- 白名单

---

# Rule 34：默认最小权限

禁止：

```text
admin all
```

必须：

```text
least privilege
```

---

# Rule 35：Secrets 禁止硬编码

禁止：

```python
API_KEY = "xxxxx"
```

必须：

```text
env / vault / kms
```

---

# 十四、CI/CD 规范

# Rule 36：所有提交必须自动检查

包括：

- lint
- test
- security scan
- type check

---

# Rule 37：禁止直接部署生产

必须：

```text
staging → canary → production
```

---

# 十五、Observability 规范

# Rule 38：必须具备 TraceID

所有请求链路必须可追踪。

---

# Rule 39：必须具备 Metrics

包括：

- latency
- error rate
- qps
- saturation

---

# Rule 40：必须具备报警

必须监控：

- error spike
- memory leak
- timeout
- queue backlog

---

# 十六、AI Agent 特殊治理规则

# Rule 41：Agent 不允许直接追加 main.py

新增能力必须：

```text
创建新模块
```

---

# Rule 42：Agent 必须优先重构

发现：

- 文件过大
- 模块耦合
- 重复代码

必须：

```text
先 refactor
再新增功能
```

---

# Rule 43：Agent 必须生成 ADR

所有重大变更：

- 架构调整
- 数据结构变化
- API变更

必须生成：

```text
Architecture Decision Record
```

---

# Rule 44：Agent 必须评估技术债

新增代码前必须检查：

- 是否增加耦合
- 是否破坏边界
- 是否增加复杂度
- 是否增加未来维护成本

---

# Rule 45：Agent 必须具备重构能力

包括：

- extract service
- extract adapter
- split module
- dependency inversion
- dead code cleanup

---

# 十七、Refactor Pipeline

# 自动触发条件

| 条件 | 动作 |
|---|---|
| 文件 > 500 行 | split module |
| 圈复杂度 > 15 | simplify logic |
| import > 20 | dependency cleanup |
| duplicate > 10% | extract reusable |
| coupling 过高 | interface abstraction |

---

# Refactor 顺序

```text
1. 分析 dependency graph
2. 建立 boundary
3. 提取 interface
4. 拆 infra
5. 拆 service
6. 提取 domain
7. 添加测试
8. 清理 dead code
```

---

# 十八、架构守卫系统

# Forbidden Rules

禁止：

```text
domain import infra
```

禁止：

```text
跨模块直接访问数据库
```

禁止：

```text
controller 写业务逻辑
```

禁止：

```text
service 写 SQL
```

禁止：

```text
utils.py 超过 300 行
```

---

# Complexity Budget

每个模块必须限制：

| 指标 | 上限 |
|---|---|
| 文件数 | 可治理 |
| 圈复杂度 | 15 |
| 平均函数长度 | 40 |
| 模块依赖 | 10 |

---

# 十九、生产级 Agent 最终目标

真正高级 Agent 不只是：

```text
生成代码
```

而是：

```text
自动控制复杂度
自动维持系统健康
自动治理架构
自动发现技术债
自动重构系统
自动维护边界
```

---

# 二十、最终核心理念

# 软件工程本质

```text
控制复杂度
```

# 架构本质

```text
控制熵增
```

# 工程化本质

```text
降低协作成本
```

# AI Agent 最终目标

```text
持续生成
可长期演进
可维护
可治理
可扩展
低复杂度
高稳定性
```

的软件系统。

