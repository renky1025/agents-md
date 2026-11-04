# TypeScript Agent 开发规范与工程准则

> **适用对象**：所有负责 TypeScript 项目开发、修改、重构、修复、测试的 AI Agent
> **核心目标**：让 Agent 写出可维护、可测试、可扩展、可审查的生产级代码，禁止为了'完成任务'堆砌临时逻辑、重复代码、隐式状态和未经验证的假设。

---

# 1. 总则

## 1.1 第一原则：先理解，再修改

Agent 在修改代码之前，必须理解：

* 项目目录结构
* `package.json`
* `tsconfig.json`
* 构建工具
* 测试框架
* lint / formatter 配置
* 现有模块职责
* 目标代码的调用关系
* 相关类型定义
* 相关测试
* 现有错误处理方式

禁止：

```text
看到需求 → 直接创建文件 → 开始写代码
```

必须遵循：

```text
理解需求
  ↓
定位代码
  ↓
理解上下文
  ↓
确认架构边界
  ↓
设计最小修改方案
  ↓
实现
  ↓
类型检查
  ↓
测试
  ↓
Lint / Format
  ↓
Review
```

---

# 2. Agent 工作纪律

## 2.1 修改前必须回答 5 个问题

在开始编码之前，Agent 必须明确：

1. 我要解决什么问题？
2. 当前代码为什么无法满足需求？
3. 哪些文件真正需要修改？
4. 修改后会影响哪些调用方？
5. 如何证明修改正确？

如果无法回答这些问题，应继续阅读代码，禁止直接实现。

---

## 2.2 最小修改原则

默认采用：

> **Minimum Change, Maximum Correctness**

优先修改现有设计，而不是重新创造一套设计。

禁止：

* 为一个小需求重构整个项目
* 无理由修改公共 API
* 无理由改变目录结构
* 无理由替换依赖
* 无理由引入新的设计模式
* 为了'看起来更优雅'而大规模重写代码

例如：

```text
修复一个 bug
↓
修改 1 个函数
```

Agent 不应该变成：

```text
修复一个 bug
↓
重写 Service
↓
新增 Repository
↓
新增 Factory
↓
新增 EventBus
↓
替换整个数据层
```

除非现有架构确实无法实现需求，并且已经说明原因。

---

# 3. 代码质量红线

以下行为默认禁止。

## 3.1 禁止 `any`

禁止：

```ts
const data: any = ...
```

禁止：

```ts
function process(data: any) {}
```

禁止：

```ts
catch (error: any) {}
```

优先：

```ts
const data: unknown = ...
```

然后通过类型守卫进行收窄：

```ts
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value
  );
}
```

只有第三方库类型缺失、动态边界无法建模等特殊情况允许局部使用 `any`。

使用时必须有明确理由。

---

# 4. TypeScript 类型系统规范

## 4.1 类型优先

任何跨模块的数据结构都必须拥有明确类型。

例如：

```ts
interface User {
  id: string;
  name: string;
  email: string;
}
```

禁止：

```ts
function createUser(data: {
  id: string;
  name: string;
  email: string;
}) {}
```

当该类型需要跨多个模块使用时，应抽取为独立类型。

---

## 4.2 优先使用精确类型

避免：

```ts
status: string;
```

优先：

```ts
type TaskStatus =
  | 'pending'
  | 'running'
  | 'completed'
  | 'failed';
```

避免：

```ts
type Config = Record<string, unknown>;
```

如果结构已知，应定义明确接口。

---

## 4.3 禁止类型欺骗

禁止滥用：

```ts
as SomeType
```

尤其禁止：

```ts
value as any as User
```

禁止：

```ts
foo!
```

用于掩盖实际可能为空的问题。

类型断言只能用于 Agent 已经能够证明数据满足目标类型的情况。

---

# 5. Null / Undefined 规范

必须显式处理可能为空的值。

禁止：

```ts
const user = users.find(x => x.id === id);

console.log(user.name);
```

正确：

```ts
const user = users.find(x => x.id === id);

if (!user) {
  throw new UserNotFoundError(id);
}

console.log(user.name);
```

对于真正可选的数据，应明确表达：

```ts
interface User {
  id: string;
  nickname?: string;
}
```

---

# 6. 函数设计规范
# 6. 文件与代码规模控制规范

代码规模必须受到约束。Agent 不得因为'功能都相关'就持续向同一个文件、类或函数中追加代码。

规模控制的目的：

```text
控制认知负担
↓
保持模块职责清晰
↓
降低修改风险
↓
避免 God Object / God File
↓
提高测试可维护性
↓
降低 Agent 后续修改成本
```

---

## 6.1 文件规模硬性限制

默认情况下：

| 指标        |   建议值 |   警戒值 | 原则上禁止 |
| --------- | ----: | ----: | ----: |
| 单文件代码行数   | ≤ 300 | > 400 | > 600 |
| 单文件有效代码行数 | ≤ 250 | > 350 | > 500 |
| 单文件导出数量   |  ≤ 15 |  > 20 |  > 30 |
| 单文件主要职责   |   1 个 |   2 个 | 3 个以上 |

这里的代码行数主要指有效代码，不应通过大量空行、注释来规避限制。

### Agent 规则

当文件超过：

```text
300 行
```

新增功能之前必须先判断是否应该拆分。

当文件超过：

```text
400 行
```

原则上禁止继续无条件追加功能。

当文件超过：

```text
600 行
```

必须考虑拆分，除非该文件属于确实需要集中定义的特殊文件，例如：

```text
Schema 定义
大型静态配置
自动生成代码
类型声明
```

即使属于这些特殊情况，也必须经过合理性判断。

---

# 6.2 文件职责限制

一个文件应该有一个明确的核心职责。

例如：

```text
task-service.ts
```

主要负责：

```text
Task 业务操作
```

不应该同时负责：

```text
Task
+
用户认证
+
数据库连接
+
HTTP 请求
+
日志格式化
+
文件导出
```

如果一个文件需要用：

```text
'它负责 X，同时也负责 Y，还顺便处理 Z'
```

解释其职责，通常说明应该拆分。

---

# 6.3 文件过大的拆分规则

假设：

```text
task-service.ts
```

已经达到 700 行。

禁止简单地：

```text
task-service.ts
↓
task-service-2.ts
```

这种拆分没有解决职责问题。

应该根据领域职责拆分：

```text
task/
├── task-service.ts
├── task-validator.ts
├── task-executor.ts
├── task-repository.ts
├── task-state.ts
├── task-errors.ts
└── task-types.ts
```

拆分依据应该是：

```text
职责
生命周期
数据边界
变化原因
依赖关系
```

而不是单纯按照行数切割。

---

# 6.4 类规模限制

默认情况下：

| 指标         |   建议值 |   警戒值 | 原则上禁止 |
| ---------- | ----: | ----: | ----: |
| 类代码行数      | ≤ 300 | > 400 | > 600 |
| Public 方法  |  ≤ 10 |  > 15 |  > 20 |
| Private 方法 |  ≤ 15 |  > 20 |  > 30 |
| 构造函数依赖     |   ≤ 5 |   > 7 |  > 10 |
| 类承担核心职责    |     1 |     2 |    3+ |

例如：

```ts
class AgentService {
  // 1000 lines
}
```

属于明显的架构风险。

尤其当一个类同时包含：

```text
LLM 调用
Prompt 构建
Memory
Tool 调用
任务调度
状态管理
文件操作
日志
错误恢复
```

应立即拆分。

推荐：

```text
Agent
├── AgentRunner
├── PromptBuilder
├── ToolExecutor
├── MemoryManager
├── TaskPlanner
├── StateManager
└── AgentErrorHandler
```

---

# 6.5 函数规模限制

默认：

| 指标                    |  建议值 |  警戒值 | 原则上禁止 |
| --------------------- | ---: | ---: | ----: |
| 函数代码行数                | ≤ 30 | > 50 | > 100 |
| 参数数量                  |  ≤ 4 |  > 5 |   > 7 |
| 嵌套层级                  |  ≤ 3 |  > 4 |   > 5 |
| Cyclomatic Complexity | ≤ 10 | > 15 |  > 20 |
| Return 分支             |  ≤ 5 |  > 8 |  > 10 |

超过限制时，Agent 必须考虑拆分。

例如：

```ts
async function executeTask() {
  // 150 lines
}
```

应拆成：

```ts
validateTask();
prepareContext();
resolveTools();
executeSteps();
handleFailure();
buildResult();
```

---

# 6.6 函数参数限制

禁止出现：

```ts
createTask(
  name,
  description,
  type,
  priority,
  timeout,
  retryCount,
  userId,
  projectId,
  metadata,
  callback,
);
```

应考虑：

```ts
interface CreateTaskInput {
  name: string;
  description: string;
  type: TaskType;
  priority: TaskPriority;
  timeout: number;
  retryCount: number;
  userId: string;
  projectId: string;
  metadata: TaskMetadata;
}
```

然后：

```ts
createTask(input: CreateTaskInput);
```

但禁止为了减少参数数量而创建毫无意义的巨大参数对象。

---

# 6.7 React / Vue 组件规模

如果项目使用 React / Vue，同样必须限制组件规模。

默认建议：

| 指标                |     建议值 |   警戒值 |
| ----------------- | ------: | ----: |
| 单组件代码             | ≤ 300 行 | > 400 |
| Template / JSX    | ≤ 150 行 | > 250 |
| Props             |    ≤ 10 |  > 15 |
| Emits / Callbacks |    ≤ 10 |  > 15 |
| 状态变量              |    ≤ 10 |  > 15 |
| Effect / Watch    |     ≤ 5 |   > 8 |

一个页面组件同时承担：

```text
数据请求
状态管理
业务计算
表单验证
弹窗
列表
分页
拖拽
WebSocket
复杂渲染
```

属于明显的职责过载。

推荐：

```text
Page
├── Container
├── Toolbar
├── List
├── ListItem
├── Filter
├── Dialog
└── hooks / composables
```

---

# 6.8 Hook / Composable 规模

React Hook 或 Vue Composable 不应该成为新的 God Function。

例如：

```ts
useAgent()
```

如果包含：

```text
WebSocket
History
Streaming
Tool Call
State
Retry
Permission
Persistence
UI State
```

并达到数百行，应拆分：

```text
useAgentSession()
useAgentStream()
useAgentHistory()
useAgentTools()
useAgentState()
```

---

# 6.9 类型文件规模

类型定义也必须控制。

禁止：

```text
types.ts
```

达到：

```text
2000+ 行
```

然后所有模块都从这里 import。

应该按照领域拆分：

```text
types/
├── agent.ts
├── task.ts
├── tool.ts
├── session.ts
├── message.ts
├── project.ts
└── common.ts
```

---

# 6.10 Schema 文件规模

Zod / Valibot 等 Schema 不应该全部塞进：

```text
schema.ts
```

推荐：

```text
schemas/
├── agent.schema.ts
├── task.schema.ts
├── tool.schema.ts
├── message.schema.ts
└── project.schema.ts
```

---

# 6.11 Test 文件规模

测试代码同样需要拆分。

禁止：

```text
agent.test.ts
```

达到：

```text
2000 行
```

推荐：

```text
agent/
├── agent.unit.test.ts
├── agent.tool.test.ts
├── agent.error.test.ts
├── agent.streaming.test.ts
└── agent.integration.test.ts
```

测试文件拆分应该围绕测试行为，而不是机械按照行数拆。

---

# 6.12 Prompt 文件规模

对于 Agent 项目，Prompt 同样属于代码资产。

禁止：

```text
system-prompt.ts
```

包含：

```text
3000 行字符串
```

然后所有 Agent 共用一个巨大 Prompt。

推荐：

```text
prompts/
├── system/
│   ├── core.ts
│   ├── safety.ts
│   └── behavior.ts
├── planning/
│   ├── planner.ts
│   └── decomposition.ts
├── coding/
│   ├── implementation.ts
│   └── review.ts
└── tools/
    ├── tool-selection.ts
    └── tool-execution.ts
```

Prompt 也必须具备：

```text
职责边界
输入
输出
约束
失败处理
```

---

# 6.13 单模块规模限制

除了文件，还必须控制模块规模。

建议：

```text
一个业务模块
≤ 10~15 个核心文件
```

如果一个模块出现：

```text
30+
```

个相互依赖的文件，应检查模块边界。

大型模块应该进一步划分子领域。

---

# 6.14 目录规模限制

单个目录下如果存在：

```text
30+
```

个平级 TypeScript 文件，应考虑重新组织。

例如：

```text
services/
├── user.ts
├── task.ts
├── agent.ts
├── tool.ts
├── session.ts
├── ...
```

逐渐发展到几十个文件后，应考虑：

```text
services/
├── user/
├── task/
├── agent/
├── tool/
└── session/
```

---

# 6.15 导出数量限制

一个文件不应该成为'万能出口'。

例如：

```ts
export {
  User,
  Task,
  Agent,
  Tool,
  Session,
  Message,
  Project,
  Config,
  ...
};
```

如果一个文件导出几十个完全不同领域的对象，说明模块边界可能存在问题。

---

# 6.16 Barrel 文件规范

谨慎使用：

```text
index.ts
```

禁止创建一个超级 Barrel：

```ts
export * from './user';
export * from './task';
export * from './agent';
export * from './tool';
export * from './project';
export * from './database';
export * from './server';
```

最终导致：

```text
任何模块
 ↓
index.ts
 ↓
整个系统
```

容易造成：

```text
循环依赖
依赖污染
模块边界模糊
构建问题
Tree Shaking 问题
```

Barrel 应该服务于清晰的模块边界。

---

# 6.17 单函数复杂度与文件规模必须同时检查

不能认为：

```text
函数都很短
```

就代表代码质量很好。

例如：

```text
agent-service.ts
800 行
30 个函数
```

每个函数只有 20 行。

依然可能是 God File。

同样：

```text
agent-service.ts
100 行
execute() 100 行
```

也属于明显问题。

因此必须同时检查：

```text
File Size
Class Size
Function Size
Module Size
Dependency Count
Complexity
```

---

# 6.18 Agent 自动触发拆分规则

Agent 在发现以下情况时，必须主动进行架构检查：

```text
文件 > 300 行
类 > 300 行
函数 > 50 行
函数参数 > 5 个
嵌套 > 3 层
类依赖 > 5 个
目录 > 30 个文件
类型文件 > 500 行
测试文件 > 500 行
Prompt > 500 行
```

触发后必须回答：

```text
1. 是否存在多个职责？
2. 是否存在独立变化原因？
3. 是否存在明显的数据边界？
4. 是否存在可以独立测试的逻辑？
5. 是否存在高频修改区域？
6. 是否存在可以降低依赖的拆分方式？
```

如果答案多数为'是'，应拆分。

---

# 6.19 规模限制不是机械规则

Agent 不得为了满足行数限制而进行垃圾拆分。

错误：

```text
foo.ts
1000 行
```

为了满足 300 行限制：

```text
foo-1.ts
foo-2.ts
foo-3.ts
foo-4.ts
```

但是四个文件仍然共享：

```text
状态
变量
职责
生命周期
```

这种拆分没有意义。

真正的拆分应该形成：

```text
高内聚
低耦合
明确职责
清晰依赖
独立测试
```

---

# 6.20 最终规模红线

Agent 应将以下规则视为默认工程红线：

```text
单文件
> 600 行
必须重新评估

单类
> 600 行
必须重新评估

单函数
> 100 行
必须重新评估

函数参数
> 7 个
必须重新设计

嵌套
> 5 层
必须重构

Cyclomatic Complexity
> 20
必须重构

单目录
> 30 个 TS 文件
必须重新评估模块边界

单模块
> 15 个核心文件
必须检查是否存在子领域

单类型文件
> 1000 行
必须拆分

单测试文件
> 1000 行
必须拆分

单 Prompt
> 1000 行
必须拆分
```

---

# 6.21 Agent 的规模控制决策树

Agent 每次新增代码时都应该执行：

```text
我要修改这个文件
        ↓
文件当前多少行？
        ↓
< 300
        ↓
继续检查职责
        ↓
是否引入第二个职责？
        ↓
否 → 修改
是 → 考虑拆分
```

如果：

```text
文件 > 300
```

则：

```text
继续修改？
    ↓
是否属于同一职责？
    ↓
是
    ↓
是否会进一步超过 400？
    ↓
是 → 优先拆分
否 → 可以修改
```

如果：

```text
文件 > 600
```

默认：

```text
禁止继续堆代码
        ↓
分析职责
        ↓
拆分模块
        ↓
测试
        ↓
继续开发
```

---

# 6.22 Agent 必须遵守的核心原则

> **代码规模是架构信号，不是单纯的数字指标。**

当代码持续增长时，Agent 必须主动思考：

```text
为什么代码越来越多？
        ↓
是不是职责越来越多？
        ↓
是不是抽象边界不清？
        ↓
是不是模块设计有问题？
        ↓
是不是状态集中在一个地方？
        ↓
是不是某个类正在成为 God Object？
```

因此：

```text
300 行
```

是提醒。

```text
400 行
```

是警戒。

```text
600 行
```

是强烈的架构重构信号。

---

# 6.23 最终代码规模检查表

每次任务完成前，Agent 必须检查：

```text
[ ] 单文件是否 ≤ 300 行
[ ] 是否存在 > 400 行的文件
[ ] 是否存在 > 600 行的文件
[ ] 单类是否 ≤ 300 行
[ ] 是否存在 > 600 行的类
[ ] 单函数是否 ≤ 30 行
[ ] 是否存在 > 50 行的函数
[ ] 是否存在 > 100 行的函数
[ ] 函数参数是否 ≤ 4 个
[ ] 是否存在 > 7 个参数的函数
[ ] 嵌套是否 ≤ 3 层
[ ] 是否存在 > 5 层嵌套
[ ] Cyclomatic Complexity 是否合理
[ ] 单目录是否存在大量 TS 文件
[ ] 是否存在 God Object
[ ] 是否存在 God File
[ ] 是否存在超级 index.ts
[ ] 是否存在超级 types.ts
[ ] 是否存在超级 utils.ts
[ ] 是否存在超级 schema.ts
[ ] 是否存在超级 prompt.ts
[ ] 是否因为行数限制进行了机械拆分
[ ] 拆分后模块职责是否真正清晰
```

**Agent 不允许使用'代码虽然很长，但功能相关'作为无限扩大文件的理由。**

当一个文件持续增长时，首先应该重新审视职责边界，而不是继续往里面添加代码。

## 6.1 单一职责

一个函数只负责一个明确的职责。

禁止：

```ts
async function processUser() {
  // 查询数据库
  // 校验参数
  // 调用 API
  // 修改状态
  // 发送消息
  // 写日志
  // 生成报告
}
```

应该拆分为：

```ts
validateUser();
loadUser();
updateUser();
publishUserEvent();
```

---

## 6.2 控制函数复杂度

如果一个函数出现以下情况，应考虑拆分：

* 超过约 50 行
* 嵌套超过 3 层
* 大量 `if / else`
* 大量 `switch`
* 同时处理多个业务概念
* 出现大量临时变量
* 很难给函数起一个准确名字

函数应该能够用一句话解释清楚。

---

# 7. 命名规范

名称必须表达业务含义。

推荐：

```ts
calculateTotalPrice()
validateTaskInput()
findActiveSession()
createExecutionPlan()
```

禁止：

```ts
handle()
process()
doSomething()
run()
data()
result()
temp()
obj()
info()
```

除非上下文已经明确限定其含义。

---

# 8. Boolean 命名

Boolean 必须具有明确语义。

推荐：

```ts
isActive
isValid
hasPermission
canRetry
shouldRefresh
```

避免：

```ts
active
valid
permission
retry
refresh
```

---

# 9. 常量规范

禁止魔法数字：

```ts
if (retryCount > 3) {}
```

推荐：

```ts
const MAX_RETRY_COUNT = 3;

if (retryCount > MAX_RETRY_COUNT) {}
```

禁止：

```ts
setTimeout(resolve, 3000);
```

推荐：

```ts
const REQUEST_TIMEOUT_MS = 3000;
```

---

# 10. 错误处理规范

## 10.1 禁止吞掉异常

绝对禁止：

```ts
try {
  await execute();
} catch {
}
```

也禁止：

```ts
try {
  await execute();
} catch (error) {
  console.log(error);
}
```

如果错误可以恢复：

```ts
try {
  await execute();
} catch (error) {
  logger.error('Task execution failed', { error });
  return fallbackResult();
}
```

如果无法恢复：

```ts
try {
  await execute();
} catch (error) {
  throw new TaskExecutionError('Task execution failed', {
    cause: error,
  });
}
```

---

# 11. Error 类型规范

业务错误应该具有明确类型。

例如：

```ts
class UserNotFoundError extends Error {
  constructor(userId: string) {
    super(`User not found: ${userId}`);
    this.name = 'UserNotFoundError';
  }
}
```

调用方可以：

```ts
if (error instanceof UserNotFoundError) {
  // ...
}
```

禁止所有错误最终都变成：

```ts
throw new Error('something went wrong');
```

---

# 12. Async / Promise 规范

禁止：

```ts
array.forEach(async item => {
  await process(item);
});
```

应该：

```ts
await Promise.all(
  array.map(item => process(item))
);
```

如果必须串行：

```ts
for (const item of array) {
  await process(item);
}
```

必须明确选择：

```text
并行
Promise.all()

串行
for...of + await

容错并行
Promise.allSettled()
```

---

# 13. Promise 错误处理

禁止创建未处理 Promise：

```ts
doSomethingAsync();
```

如果确实无需等待，也必须明确表达意图，例如项目已有 `void` 约定时：

```ts
void doSomethingAsync();
```

但必须确认函数内部能够自行处理异常。

---

# 14. 并发规范

并发必须有边界。

禁止无上限：

```ts
await Promise.all(
  hugeList.map(item => process(item))
);
```

当数据规模不可控时，应采用并发限制。

例如：

```text
10000 个任务
↓
并发 10
↓
处理完成一个
↓
继续下一个
```

不得因为'Promise.all 很方便'就制造请求风暴。

---

# 15. 不可变数据原则

优先使用不可变操作。

避免：

```ts
user.name = name;
user.roles.push(role);
```

在共享状态场景尤其要谨慎。

优先：

```ts
const updatedUser = {
  ...user,
  name,
};
```

对于复杂状态系统，必须明确谁拥有修改权限。

---

# 16. 禁止隐式全局状态

禁止：

```ts
let currentUser: User | undefined;
let currentTask: Task | undefined;
```

然后让大量模块直接修改。

优先：

```ts
class SessionStore {
  private currentUser?: User;

  getUser(): User | undefined {
    return this.currentUser;
  }

  setUser(user: User): void {
    this.currentUser = user;
  }
}
```

更复杂的状态必须明确：

```text
Owner
Mutation
Read
Lifecycle
Persistence
```

---

# 17. 模块设计

模块必须具有明确职责。

推荐：

```text
src/
├── domain/
├── application/
├── infrastructure/
├── interfaces/
├── shared/
└── config/
```

具体目录结构根据项目实际情况决定。

禁止为了遵守某种架构模板而强行拆目录。

---

# 18. 依赖方向

高层业务逻辑不应该随意依赖底层实现。

例如：

```text
Controller
   ↓
Service
   ↓
Repository
   ↓
Database
```

业务层不应该直接到处：

```ts
mysql.query(...)
redis.get(...)
axios.post(...)
```

应该通过清晰的边界管理基础设施依赖。

---

# 19. Import 规范

禁止循环依赖。

例如：

```text
A → B
B → C
C → A
```

如果发现循环依赖，应重新检查模块边界。

禁止：

```ts
import * as everything from './everything';
```

除非确实符合项目现有模块设计。

优先精确导入。

---

# 20. API 设计

公共 API 必须稳定、明确。

例如：

```ts
interface TaskService {
  create(input: CreateTaskInput): Promise<Task>;
  getById(id: string): Promise<Task | undefined>;
  cancel(id: string): Promise<void>;
}
```

避免：

```ts
process(data: any): Promise<any>;
```

API 输入输出必须可推导。

---

# 21. 数据边界必须验证

外部输入永远视为不可信。

包括：

* HTTP 请求
* CLI 参数
* 用户输入
* 文件内容
* JSON
* 环境变量
* 第三方 API
* MCP 返回值
* LLM 返回值
* 数据库动态字段

必须经过：

```text
输入
 ↓
解析
 ↓
验证
 ↓
类型收窄
 ↓
业务处理
```

推荐使用项目已有的 Schema Validation 工具，例如：

```text
Zod
Valibot
ArkType
```

不要为了一个简单项目强行引入新依赖，优先使用项目已有方案。

---

# 22. LLM / Agent 项目特殊规范

对于 Agent 项目，以下内容尤其重要。

## 22.1 LLM 输出永远不能直接信任

禁止：

```ts
const result = await llm.generate();

execute(result);
```

必须：

```text
LLM
 ↓
Parse
 ↓
Schema Validation
 ↓
Semantic Validation
 ↓
Safety Check
 ↓
Execution
```

例如：

```ts
const raw = await llm.generate();

const parsed = TaskPlanSchema.parse(
  JSON.parse(raw)
);

validateTaskPlan(parsed);

await executeTaskPlan(parsed);
```

---

# 23. Agent 不允许通过 Prompt 弥补代码缺陷

禁止把程序逻辑全部写进 Prompt。

例如：

```text
如果 A 就调用 X
如果 B 就调用 Y
如果 C 就调用 Z
如果发生错误重新执行
如果结果不对再修复
```

这些涉及确定性行为的部分应该由程序控制。

原则：

```text
确定性逻辑 → TypeScript
概率性判断 → LLM
```

例如：

```text
权限检查 → TypeScript
参数验证 → TypeScript
状态机 → TypeScript
重试机制 → TypeScript
事务 → TypeScript
工具执行 → TypeScript

意图理解 → LLM
语义分类 → LLM
自然语言总结 → LLM
复杂推理 → LLM
```

---

# 24. Agent 工具调用规范

工具调用必须拥有明确的类型：

```ts
interface ToolCall {
  toolName: string;
  arguments: Record<string, unknown>;
}
```

工具执行器必须负责：

```text
Tool Discovery
↓
Argument Validation
↓
Permission Check
↓
Execution
↓
Timeout
↓
Retry
↓
Result Normalization
↓
Error Handling
```

禁止：

```ts
eval()
```

禁止动态执行未经验证的代码。

---

# 25. 状态机优先于大量 Boolean

禁止：

```ts
isRunning
isFinished
isFailed
isCancelled
```

这些状态很容易产生非法组合。

推荐：

```ts
type TaskStatus =
  | 'pending'
  | 'running'
  | 'completed'
  | 'failed'
  | 'cancelled';
```

状态转换应该明确：

```text
pending
  ↓
running
  ↓
completed

running
  ↓
failed

running
  ↓
cancelled
```

禁止任意状态跳转。

---

# 26. 重试机制规范

禁止：

```ts
while (true) {
  try {
    await execute();
    break;
  } catch {}
}
```

重试必须：

* 有最大次数
* 有超时
* 有退避
* 判断错误是否可重试
* 记录失败原因

例如：

```text
retry = 0
retry < MAX_RETRIES
        ↓
execute
        ↓
success → return
        ↓
retryable?
   ↓          ↓
 yes          no
   ↓           ↓
backoff      throw
```

---

# 27. 日志规范

日志必须能够回答：

```text
发生了什么？
什么时候发生？
哪个任务？
哪个请求？
哪个 Agent？
哪个工具？
为什么失败？
```

推荐：

```ts
logger.error('Tool execution failed', {
  toolName,
  taskId,
  error,
});
```

禁止大量：

```ts
console.log('here');
console.log('test');
console.log(data);
```

生产代码禁止残留调试日志。

---

# 28. 日志禁止泄露敏感信息

禁止记录：

```text
API Key
Token
Password
Cookie
完整 Authorization Header
用户隐私数据
```

例如：

```ts
logger.info('Request', {
  authorization: request.headers.authorization,
});
```

属于严重问题。

---

# 29. 配置管理

禁止把配置硬编码：

```ts
const apiUrl = 'http://192.168.1.100:8080';
```

应该：

```ts
const apiUrl = config.apiUrl;
```

环境配置应该集中管理。

例如：

```text
config/
├── env.ts
├── app-config.ts
└── model-config.ts
```

环境变量必须在启动阶段完成校验。

---

# 30. 注释规范

注释解释：

```text
Why
```

而不是：

```text
What
```

差：

```ts
// Add 1
count += 1;
```

好：

```ts
// Keep the counter monotonic because downstream consumers
// use it as the event ordering key.
count += 1;
```

如果代码本身已经非常清晰，不需要添加无意义注释。

---

# 31. 禁止过度注释

禁止：

```ts
// Create user
const user = createUser();

// Return user
return user;
```

这些注释增加噪声。

---

# 32. 测试规范

任何涉及业务逻辑的修改，都必须考虑测试。

至少覆盖：

```text
正常情况
边界情况
异常情况
```

例如：

```text
输入为空
输入非法
资源不存在
重复请求
并发调用
网络失败
超时
权限不足
```

---

# 33. Bug 修复必须增加回归测试

修复：

```text
Bug #123
```

必须尽可能形成：

```text
Regression Test
```

防止以后再次出现。

---

# 34. 测试禁止只测 Happy Path

禁止只写：

```ts
it('should work', ...)
```

应该覆盖：

```text
valid input
invalid input
empty input
boundary input
dependency failure
```

---

# 35. Mock 规范

Mock 应该模拟外部依赖。

不要 Mock 被测试的核心逻辑。

例如：

```text
Service
 ↓
Repository
 ↓
Database
```

测试 Service 时可以 Mock Repository。

测试 Service 时不要把 Service 自己 Mock 掉。

---

# 36. 性能规范

Agent 不允许在没有分析的情况下进行性能优化。

优化之前应明确：

```text
问题是什么？
瓶颈在哪里？
当前复杂度？
数据规模？
优化收益？
副作用？
```

禁止为了'可能更快'就引入：

* 缓存
* 多线程
* Worker
* Redis
* MQ
* 并行计算
* 数据库索引
* 复杂算法

---

# 37. 数据库规范

禁止：

```ts
for (const user of users) {
  await db.query(...);
}
```

如果形成 N+1 查询，应重新设计查询策略。

必须关注：

```text
N+1
Index
Transaction
Connection Pool
Pagination
Lock
Consistency
Timeout
```

大数据量场景禁止一次性加载全部数据：

```ts
const allRecords = await repository.findAll();
```

应该使用：

```text
Pagination
Cursor
Batch
Stream
```

---

# 38. API 请求规范

外部 API 调用必须考虑：

```text
Timeout
Retry
Rate Limit
Cancellation
Error Mapping
Response Validation
```

禁止无限等待。

---

# 39. AbortController

支持取消的异步任务，应尽可能支持：

```ts
AbortSignal
```

例如：

```ts
async function fetchData(
  signal?: AbortSignal,
): Promise<Data> {
  // ...
}
```

Agent、CLI、Web 请求、长任务尤其应该考虑取消机制。

---

# 40. 资源生命周期

任何资源都必须明确：

```text
Create
Use
Dispose
```

包括：

* File Handle
* Stream
* Database Connection
* Worker
* Child Process
* Timer
* Event Listener
* WebSocket
* MCP Client
* Browser
* Temporary File

禁止创建资源后不考虑释放。

---

# 41. Event Listener 规范

禁止：

```ts
emitter.on('event', handler);
```

然后永远不解除。

长期运行的 Agent 系统必须防止 Listener 泄漏。

应该：

```ts
const cleanup = () => {
  emitter.off('event', handler);
};
```

---

# 42. 文件操作规范

必须考虑：

```text
文件不存在
权限不足
路径非法
编码错误
磁盘错误
并发写入
部分写入
```

禁止假设：

```ts
await fs.writeFile(path, content);
```

一定成功。

---

# 43. 路径安全

禁止直接信任用户输入构造文件路径。

例如：

```ts
const path = join(baseDir, userInput);
```

必须防止：

```text
../
../../
绝对路径
符号链接
路径逃逸
```

---

# 44. 安全规范

禁止：

```ts
eval()
new Function()
child_process.exec(userInput)
```

禁止把用户输入直接拼接进：

```text
SQL
Shell
File Path
HTML
URL
Code
```

任何动态执行都必须经过明确的安全边界。

---

# 45. 依赖管理

引入第三方依赖之前必须确认：

1. 现有依赖能否解决？
2. 是否真的需要？
3. 包是否成熟？
4. 是否长期维护？
5. 是否增加明显 bundle / runtime 成本？
6. 是否引入安全风险？
7. 是否与当前项目许可证兼容？

禁止：

```text
为了一个 10 行功能
引入一个 500KB npm 包
```

---

# 46. 不允许重复造轮子

添加工具函数之前，必须搜索：

```text
项目内部
已有依赖
标准库
现有工具模块
```

例如项目已经存在：

```ts
parseJson()
retry()
logger()
httpClient()
```

禁止 Agent 再创建：

```ts
myParseJson()
safeParseJson()
betterParseJson()
customRetry()
```

---

# 47. DRY 原则

发现重复代码时，应判断是否真正属于同一抽象。

禁止为了消除两段类似代码就创建过度抽象：

```ts
AbstractBaseProcessorFactoryManager
```

抽象必须有稳定的共同语义。

---

# 48. YAGNI 原则

没有需求的功能不要提前实现。

禁止：

```text
未来可能需要
所以先支持
```

例如：

```ts
interface Config {
  mode: string;
  retry: number;
  cache: boolean;
  distributed: boolean;
  cluster: boolean;
  multiRegion: boolean;
  pluginSystem: boolean;
}
```

如果当前只有：

```ts
mode
```

就只实现：

```ts
interface Config {
  mode: string;
}
```

---

# 49. 反模式禁止清单

Agent 禁止产生以下代码：

```text
any 滥用
as any
双重类型断言
!
空 catch
无限循环
无限重试
魔法数字
魔法字符串
超长函数
超大类
God Object
循环依赖
全局可变状态
N+1 查询
未处理 Promise
无边界 Promise.all
重复代码
重复工具函数
无意义抽象
无意义注释
无意义日志
硬编码配置
硬编码密钥
直接信任外部输入
eval
new Function
未经验证的 Shell 执行
未经验证的 SQL 拼接
未经验证的 LLM 输出
```

---

# 50. 修改现有代码的规范

Agent 修改代码时：

### Step 1

先阅读整个目标文件。

### Step 2

搜索：

```text
所有调用方
所有 import
相关类型
相关测试
相关接口
```

### Step 3

判断修改影响范围。

### Step 4

进行最小修改。

### Step 5

检查：

```text
类型
异常
边界
并发
资源生命周期
兼容性
```

### Step 6

运行测试。

### Step 7

运行：

```bash
tsc --noEmit
```

以及项目已有的：

```bash
lint
format
test
build
```

具体命令必须以项目配置为准。

---

# 51. 新增功能的标准流程

任何新功能必须经过：

```text
Requirement
    ↓
Codebase Investigation
    ↓
Design
    ↓
Type Definition
    ↓
Implementation
    ↓
Unit Test
    ↓
Integration Test
    ↓
Type Check
    ↓
Lint
    ↓
Build
    ↓
Review
```

---

# 52. 重构规范

重构必须保持行为一致，除非需求明确要求改变行为。

重构之前必须明确：

```text
Current Behavior
Target Structure
Behavioral Invariants
```

推荐：

```text
小步重构
↓
测试
↓
继续重构
↓
测试
```

禁止一次性：

```text
删除 30 个文件
新增 50 个文件
最后运行一次测试
```

---

# 53. API / 类型变更规范

修改：

```ts
interface
type
function signature
class public method
export
API response
database schema
```

必须搜索所有使用方。

尤其关注：

```text
直接调用
间接调用
测试
Mock
类型推导
序列化
反序列化
```

---

# 54. 向后兼容

修改公共接口时，必须考虑：

```text
Existing Consumers
Existing Data
Existing Configuration
Existing API Clients
```

如果确实需要破坏兼容性，必须明确说明：

```text
Breaking Change
Migration Strategy
```

---

# 55. Git 规范

Agent 不应随意：

```text
git reset --hard
git clean -fd
git push --force
```

除非用户明确要求并确认影响。

提交应该具有明确目的：

```text
feat: add task retry policy
fix: prevent duplicate task execution
refactor: extract task validator
test: add task failure regression tests
```

禁止：

```text
update
fix
change
test
modify
```

这种无法表达具体内容的提交信息。

---

# 56. Agent 禁止擅自修改

以下内容原则上需要谨慎：

```text
package.json
lockfile
tsconfig.json
数据库 Schema
CI/CD
Dockerfile
部署配置
环境变量
权限配置
公共 API
```

如果任务不需要，不要顺手修改。

---

# 57. Agent 禁止'顺手优化'

例如用户要求：

```text
修复登录 Bug
```

Agent 不应该顺手：

```text
升级 React
升级 TypeScript
替换 Axios
重构目录
修改 ESLint
重新设计数据库
```

除非这些变化直接解决当前问题。

---

# 58. 完成任务的验收标准

Agent 不得以：

```text
代码写完了
```

作为任务完成标准。

必须至少确认：

```text
[ ] 需求已经理解
[ ] 修改范围明确
[ ] 类型正确
[ ] 异常处理完整
[ ] 边界情况处理
[ ] 没有明显重复代码
[ ] 没有无意义抽象
[ ] 没有调试代码
[ ] 没有敏感信息泄露
[ ] 测试通过
[ ] TypeScript 类型检查通过
[ ] Lint 通过
[ ] Build 通过
```

---

# 59. Agent 自我 Review

提交代码之前必须进行一次独立 Review。

至少检查：

### Correctness

```text
逻辑正确吗？
边界正确吗？
异常正确吗？
```

### Type Safety

```text
有没有 any？
有没有错误类型断言？
有没有可能 undefined？
```

### Maintainability

```text
以后的人能看懂吗？
职责清晰吗？
命名准确吗？
```

### Reliability

```text
超时怎么办？
失败怎么办？
重试怎么办？
取消怎么办？
资源怎么释放？
```

### Performance

```text
有没有 N+1？
有没有无界并发？
有没有重复计算？
有没有不必要 IO？
```

### Security

```text
用户输入安全吗？
文件路径安全吗？
Shell 安全吗？
SQL 安全吗？
Secret 泄露了吗？
LLM 输出验证了吗？
```

---

# 60. Agent 的最终行为准则

Agent 必须遵守以下优先级：

```text
正确性
  >
安全性
  >
可维护性
  >
可测试性
  >
性能
  >
代码简洁
```

当多个目标冲突时，优先保证正确性和安全性。

---

# 61. 最重要的 20 条硬规则

如果 Agent 无法完整执行所有规范，至少必须严格执行下面 20 条：

```text
1. 先读代码，再写代码。

2. 先理解调用关系，再修改接口。

3. 禁止滥用 any。

4. 禁止用类型断言掩盖错误。

5. 禁止吞异常。

6. 禁止无限重试。

7. 禁止无限循环。

8. 禁止无边界并发。

9. 禁止未处理 Promise。

10. 禁止魔法数字和魔法字符串。

11. 禁止全局可变状态。

12. 禁止循环依赖。

13. 禁止 N+1 查询。

14. 禁止未经验证的外部输入进入核心逻辑。

15. 禁止未经验证的 LLM 输出直接执行。

16. 禁止为了小需求进行大规模重构。

17. 禁止没有需求就增加复杂架构。

18. Bug 修复必须尽可能增加回归测试。

19. 修改完成后必须执行类型检查、测试和项目已有的质量检查。

20. Agent 必须能够解释自己修改的每一处代码为什么存在。
```

---

# 62. Agent 执行前检查协议

在真正修改代码之前，Agent 应执行：

```text
[UNDERSTAND]

阅读项目结构
↓
读取 package.json
↓
读取 tsconfig.json
↓
读取相关配置
↓
定位目标代码
↓
搜索调用方
↓
搜索相关测试
↓
理解现有实现

[PLAN]

明确问题
↓
明确根因
↓
明确修改范围
↓
明确风险
↓
明确验证方法

[IMPLEMENT]

最小修改
↓
保持现有架构
↓
保持类型安全
↓
处理异常
↓
处理边界

[VERIFY]

TypeScript
↓
Lint
↓
Unit Test
↓
Integration Test
↓
Build

[REVIEW]

检查副作用
↓
检查重复代码
↓
检查错误处理
↓
检查安全问题
↓
检查性能问题
↓
确认没有无关修改
```

---

# 63. Agent 最终原则

> **不要为了让代码看起来'完成了'而写代码。**

真正完成一个任务意味着：

```text
需求被正确理解
        ↓
设计与现有系统一致
        ↓
代码具有明确职责
        ↓
类型系统能够约束错误
        ↓
异常能够正确传播
        ↓
边界条件得到处理
        ↓
测试能够证明行为
        ↓
构建能够通过
        ↓
后续开发者能够维护
```

Agent 的目标不是：

```text
尽快生成最多代码
```

而是：

```text
用最少的正确代码
解决明确的问题
并且让下一次修改仍然容易
```

**任何无法解释、无法验证、无法测试、无法维护的代码，都应该被视为未完成。**

