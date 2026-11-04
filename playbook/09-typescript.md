# 09 TypeScript 规范与最佳实践

版本 2.0
适用范围：所有 TS / TSX 项目——Node 服务、前端、CLI、Agent 系统
依据：v1 内容 + TypeScript 5.x / Node 26 现状勘误（勘误表见附录 A）

**与 01-07 的关系**：语言无关的规则以 01-07 为准，本文件只写 TS 特有或 TS 中的具体形态。两者冲突时以本文件为准，并在交付说明里指出。与 08-rust 平行：同一套工程观，不同语言的具体形态。

---

## 0. 工具链

### 0.1 tsconfig 基线

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "skipLibCheck": true
  }
}
```

| 选项 | 作用 | 为什么必须 |
| --- | --- | --- |
| `strict` | 打开全部严格检查族 | 关闭它等于放弃 TS 的主要价值 |
| `noUncheckedIndexedAccess` | `arr[i]` 类型变为 `T \| undefined` | 索引越界是 JS 最高频的运行时错误来源 |
| `exactOptionalPropertyTypes` | 区分「属性缺失」与「值为 undefined」 | 序列化、合并、可选参数语义相关的 bug 大多来自这里 |
| `verbatimModuleSyntax` | 强制 `import type`，导入不被擅自擦除 | 避免运行时导入被类型擦除后破坏循环依赖与副作用 |
| `isolatedModules` | 保证单文件可独立转译 | esbuild / swc / Node 类型剥离都按单文件处理 |

- **必须** 新项目从 `strict: true` 起步。存量项目逐个开启，先开 `strict`，再开 `noUncheckedIndexedAccess`。
- **必须** 使用 ESM 时在 `package.json` 写 `"type": "module"`。在 `module: nodenext` + `verbatimModuleSyntax` 下，缺这个字段会把 `.ts` 当成 CommonJS，`import` / `export` 语句直接报 TS1295（已实测）。这一条写错会让整条工具链报一堆看似无关的错。
- **必须** 前端/打包器项目用 `"moduleResolution": "bundler"` + `"noEmit": true`，产物交给 Vite / esbuild / swc。
- `skipLibCheck: true` 是务实选择（跳过 `.d.ts` 内部检查，显著提速）；但它会掩盖依赖间的类型冲突，升级依赖后应临时关掉检查一次。

### 0.2 运行与构建

- Node 22.6+ 起可原生执行 `.ts`（类型剥离），Node 26 已默认可用。实测限制：**strip-only 模式不支持 `enum`、`namespace`、参数属性、`import =`**，遇到直接抛 `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`。
- 因此要么全程使用可擦除语法（推荐，配置 `"erasableSyntaxOnly": true` 强制约束），要么走完整构建链。不要两套混用。
- **必须** 类型检查与构建分离：CI 跑 `tsc --noEmit`，产物构建交给 bundler。用 bundler 的 `transpileOnly` 跳过类型检查后又不跑 `tsc`，等于没有类型系统。
- **必须** 锁文件入库，CI 用 `npm ci` / `pnpm install --frozen-lockfile`。包管理器按项目既有选择，不擅自更换。

### 0.3 lint 与格式化

- ESLint + `typescript-eslint`（flat config），或改用 Biome（更快的单一工具，规则集较少）。
- **必须开启**：`@typescript-eslint/no-floating-promises`、`no-misused-promises`、`no-explicit-any`、`consistent-type-imports`、`no-unnecessary-condition`（发现多余的空值判断）。
- 格式化交给 Prettier 或 Biome，禁止手工对齐；格式问题不进 code review。

---

## 1. 类型系统

### 1.1 any 与 unknown

- **禁止** `any` 出现在公开 API、函数参数、返回值、catch 变量中。唯一例外是第三方库类型缺失，且必须加注释说明并就近收窄。
- **必须** 用 `unknown` 接收不可信值，再用类型守卫或 schema 收窄。`unknown` 不是终点，收窄成具体类型才是。
- `Record<string, unknown>` 比 `any` 安全，但它只是「未知对象」的占位，不能替代业务建模。

```ts
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    typeof (value as { id: unknown }).id === 'string'
  );
}
```

### 1.2 用类型表达约束

- **必须** 有限取值用字面量联合，不用 `string`：

```ts
type TaskStatus = 'pending' | 'running' | 'completed' | 'failed';
```

- **必须** 带载荷的状态用判别联合，`switch` 后做穷尽检查：

```ts
type TaskEvent =
  | { kind: 'started'; at: number }
  | { kind: 'failed'; at: number; error: Error }
  | { kind: 'completed'; at: number; durationMs: number };

function assertNever(x: never): never {
  throw new Error(`unhandled event: ${JSON.stringify(x)}`);
}
```

- **必须** 语义不同的同构值用 branded type，防止 `userId` 与 `orderId` 互相赋值：

```ts
type UserId = string & { readonly __brand: 'UserId' };
```

- **必须** 对外暴露的状态与配置标 `readonly`，数组用 `readonly T[]`。

### 1.3 satisfies 与 as const

- **必须** 用 `satisfies` 校验对象形状，同时保留字面量类型。`as` 会放弃检查，`satisfies` 不会：

```ts
const routes = {
  home: '/',
  user: '/users/:id',
} satisfies Record<string, `/${string}`>;
```

- **必须** 需要字面量类型时用 `as const`，不要手写重复的字面量联合。
- **禁止** `as any as T` 双重断言；**禁止** 用 `!` 掩盖可能为空的值。断言只能用于「已能证明」的情况，并写明依据。

### 1.4 泛型

- 泛型参数加约束：`<T extends object>` 优于裸 `<T>`。
- 泛型推断不出来时，优先调整签名，而不是让调用方反复写显式参数。
- 避免为「只在一处使用」的场景引入泛型；具体类型更好读。

### 1.5 enum 不用

`enum` 在 Node 类型剥离下直接失败，在 `isolatedModules` 下行为受限，且会生成运行时代码。**必须** 用联合类型或 `as const` 对象替代：

```ts
const Color = { Red: 'red', Green: 'green' } as const;
type Color = (typeof Color)[keyof typeof Color];
```

---

## 2. 空值与可选性

- **必须** 处理 `find` / `get` / 索引访问返回的 `undefined`，不能直接解引用。
- **必须** 用 `??` 表达默认值，不用 `||`——后者会把 `0`、`''`、`false` 当失败。
- **必须** 用 `?.` 短路，但不要在同一个表达式里串三次以上，可读性会崩。
- 可选属性（`nickname?: string`）表示「可能不存在」；显式 `| undefined` 表示「存在但可能为空」。二者在 `exactOptionalPropertyTypes` 下不等价，按语义选。

---

## 3. 函数与模块

- **必须** 参数 ≤ 4，超过用参数对象；**禁止** 布尔标志参数（`save(user, true)` 应拆成两个函数）。
- **必须** 公开 API 显式标注返回类型；内部函数可依赖推导。
- **必须** 一个函数一件事，嵌套 ≤ 3，长度 ≤ 80 行（新代码目标 40）。
- **必须** 用 `import type` / `export type` 导入导出纯类型；**禁止** `import *`。
- **禁止** 循环依赖。出现时检查模块边界，用接口或事件拆开。

---

## 4. 错误处理

- **禁止** 空 catch，**禁止** catch 后只 `console.log`。
- **禁止** 用 `catch (e: any)` 绕过类型；`useUnknownInCatchVariables`（含在 `strict`）会让 catch 变量是 `unknown`，先收窄再处理。
- **必须** 业务错误用具名类，保留 `cause`：

```ts
class TaskExecutionError extends Error {
  constructor(message: string, options?: { cause?: unknown }) {
    super(message, options);
    this.name = 'TaskExecutionError';
  }
}
```

- **必须** 在边界转换错误：HTTP 状态码、数据库错误码在适配器层转成领域错误，不向业务层泄漏。
- **必须** 需要显式建模失败时用判别联合，而不是抛异常或返回 `null`：

```ts
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };
```

---

## 5. 异步与并发

- **禁止** `array.forEach(async ...)`——外层不会等待，异常会逃逸。并行用 `Promise.all(arr.map(...))`，串行用 `for...of` + `await`，容错并行用 `Promise.allSettled`。
- **禁止** 无边界并发。数据规模不可控时加信号量或 `p-limit`，并发度是显式配置项。
- **禁止** 裸调用返回 Promise 的函数而不处理。必须 `await`，或显式 `void p.catch(onError)`（`void` 本身不消除未处理拒绝，必须配 `.catch`）；lint 的 `no-floating-promises` 兜底。
- **必须** 每个外部调用有超时：`AbortSignal.timeout(ms)`；组合父子信号用 `AbortSignal.any([signal, AbortSignal.timeout(ms)])`。
- **必须** 把 `AbortSignal` 贯穿到下游，而不是只在上层放弃等待。
- `Promise.withResolvers()` 替代手写 `new Promise` 外部捕获 `resolve`。
- **禁止** 在 `finally` 里 `return`，会吞掉异常。

---

## 6. 资源生命周期

- **必须** 有明确释放的资源（文件句柄、连接、worker、子进程、定时器、listener、WebSocket）配对释放。
- **优先** `using` / `await using`（Node 26 实测可用，配合 `Symbol.dispose` / `Symbol.asyncDispose`），让释放跟着作用域走；不能用时用 `try/finally`。
- **必须** 注册 listener 的地方返回 `cleanup` 函数，并在卸载路径调用。
- **必须** `setTimeout` / `setInterval` 有清零点；长期运行的进程禁止只创建不清理。

---

## 7. 输入验证与安全

- **必须** 所有外部输入在边界验证一次：HTTP body、query、CLI 参数、文件内容、环境变量、第三方 API、LLM 输出。验证后得到具体类型，向内不再重复校验。
- 优先项目已有的 schema 方案（Zod / Valibot / ArkType），用 `z.infer` 从 schema 反推类型，保证「校验」与「类型」是同一来源。

### 7.1 路径安全

`path.join(base, userInput)` **不能**防止路径穿越——实测 `join('/base', '../../etc/passwd')` 直接得到 `/etc/passwd`。必须归一化后做前缀校验：

```ts
const base = path.resolve(baseDir);
const target = path.resolve(base, userInput);
if (target !== base && !target.startsWith(base + path.sep)) {
  throw new PathTraversalError(userInput);
}
```

还要考虑符号链接（必要时 `realpath` 后再校验）。

### 7.2 命令执行与其它

- **禁止** `child_process.exec` 拼接用户输入；用 `execFile` / `spawn` 传参数数组。
- **禁止** `eval`、`new Function`、动态 `require` 未验证的模块。
- SQL 一律参数化；HTML 输出一律转义；URL 拼接用 `URL` / `URLSearchParams`。
- 密钥走环境变量或密钥服务，启动时校验存在性；日志与异常上报脱敏。

---

## 8. Agent / LLM 项目专项

- **LLM 输出永远不可信**。流程固定为：解析 -> schema 校验 -> 语义校验 -> 安全检查 -> 执行。禁止把模型返回直接喂给工具或 SQL。
- **确定性逻辑归 TypeScript，概率性判断归模型**：权限、参数校验、状态机、重试、事务、工具执行都是代码；意图理解、语义分类、自然语言总结才是模型。禁止用 prompt 补代码缺陷。
- **工具调用**：每个工具一份参数 schema，校验通过才执行；统一走 ToolExecutor，负责权限、超时、重试、结果归一化。
- **状态机替代布尔汤**：`isRunning` / `isFailed` / `isCancelled` 会产生非法组合，用单一 `status` 字段加显式转换表。
- **重试**：有上限、有超时、有退避与抖动、能判定可重试性、记录每次失败原因。禁止 `while (true) { try ... catch {} }`。
- **日志**：结构化字段（traceId、taskId、toolName、event），不含 token、密钥、完整 Authorization 头。

---

## 9. 测试

- 框架跟随项目：`vitest`（新项目默认）或 `node:test`。
- **必须** 覆盖失败路径：非法输入、空值、依赖失败、超时、并发。
- **必须** 修 bug 附回归测试。
- **必须** 只 mock 外部依赖，不 mock 被测对象本身。
- 解析器、序列化、状态迁移适合用 `fast-check` 做属性测试。

---

## 10. 规模与边界

阈值与 01-code-style 保持一致，不另立一套：

| 指标 | 阈值 |
| --- | --- |
| 单文件 | 400 行 |
| 单函数 | 80 行（目标 40） |
| 嵌套 | 3 层 |
| 参数 | 4 个 |
| 圈复杂度 | 10 |
| 单目录平级文件 | 30 个 |

- **必须** 按职责拆分，禁止 `foo.ts` -> `foo-1.ts` / `foo-2.ts` 的机械切割。
- **必须** 大文件先判断：是否承担多个职责、是否有独立变化原因、是否有可独立测试的逻辑。三项都否，才允许继续追加。
- `types.ts` / `schema.ts` / `index.ts` 超过 400 行或导出超过 20 项，按领域拆目录。
- Barrel 文件（`index.ts` 重导出）只服务清晰的模块边界；**禁止** 建全项目统一的 `export *` 汇总文件（循环依赖、tree shaking 失效、边界模糊）。

---

## 11. 硬规则

时间不足时，至少执行这 12 条：

1. 先读代码和调用方，再写代码。
2. 禁止 `any` 进入公开 API；用 `unknown` + 收窄。
3. 禁止用 `as` / `!` 掩盖类型错误。
4. 禁止空 catch 和只打印日志的 catch。
5. 禁止 `forEach(async)`、裸 Promise、无边界并发。
6. 禁止无超时、无上限的重试与循环。
7. 禁止未验证的外部输入和 LLM 输出进入核心逻辑。
8. 禁止路径拼接信任用户输入（`join` 不防穿越）。
9. 禁止全局可变状态和循环依赖。
10. 禁止为小需求做大规模重构或顺手升级依赖。
11. 修 bug 附回归测试。
12. 交付前跑 `tsc --noEmit`、lint、测试、build。

---

## 附录 A：v1 勘误

v1 共 3096 行、63 节。以下按四类列出问题与处理。

### A1 结构问题

| # | 问题 | 处理 |
| --- | --- | --- |
| 1 | 章节编号重复：两个 `# 6.`（函数设计规范 / 文件规模控制），`## 6.1` 与 `## 6.2` 各出现两次 | 重新编号，函数并入第 3 节 |
| 2 | 规模控制占 6.1-6.23 共 23 节，反复陈述同一件事（300/400/600、按职责拆分、不要机械拆分） | 压缩为第 10 节一张表 + 三条判断 |
| 3 | 全文用 ```text 包裹单词和短句（如把 "300 行" 单独做成代码块），信息密度极低 | 改为正文、表格与列表 |
| 4 | 使用 Unicode 制表符绘制目录树，违反 CLAUDE.md 的 ASCII 约束 | 改为 ASCII 缩进 |
| 5 | 通用规则（最小修改、先读后写、不原子化重构、Git 安全）与工具链细节混排 | 通用部分交给 01-07，本文只保留 TS 特有内容 |

### A2 与通用规范冲突

| # | 问题 | 处理 |
| --- | --- | --- |
| 6 | 文件行数给出 300/400/600 三层阈值，与 01-code-style 的 400 单值冲突 | 统一采用 01 的阈值 |
| 7 | "Return 分支 ≤ 5、> 10 禁止"——守卫子句会增加 return 数但不增加复杂度 | 删除该指标，只约束嵌套深度 |
| 8 | 单函数 30 行建议过紧，导致为凑数机械拆分 | 改为 80 行硬上限、40 行目标 |

### A3 事实错误

| # | 问题 | 处理 |
| --- | --- | --- |
| 9 | §13 认为 `void p` 足以表达「故意不等待」。`void` 只抑制 lint，不消除未处理拒绝 | 改为 `void p.catch(onError)`，或直接 await |
| 10 | §43 只说「必须防止 ../」，未给方法；`path.join` 本身不防穿越（实测） | 补 resolve + 前缀校验的具体写法 |
| 11 | 全文未提 enum 在现代运行时的限制；Node 类型剥离下 enum/namespace 直接报错（实测） | 新增 1.5，推荐联合类型与 `as const` |
| 12 | §44 只禁 `child_process.exec`，未给替代 | 指定 `execFile` / `spawn` + 参数数组 |

### A4 缺失的现代实践

| # | 缺失项 | 已补入 |
| --- | --- | --- |
| 13 | `strict` 全家桶、`noUncheckedIndexedAccess`、`exactOptionalPropertyTypes`——TS 最大的质量杠杆完全没写 | 0.1 |
| 14 | `satisfies`、`as const`、判别联合、branded type | 1.2 / 1.3 |
| 15 | ESM / `verbatimModuleSyntax` / `moduleResolution` | 0.1 |
| 16 | Node 原生运行 TS 及其 strip-only 限制 | 0.2 |
| 17 | `AbortSignal.timeout` / `AbortSignal.any`、`Promise.withResolvers`、有界并发 | 5 |
| 18 | `using` / `await using` 显式资源管理（Node 26 实测可用） | 6 |
| 19 | 现代 lint 与测试栈（typescript-eslint flat config、Biome、vitest、node:test） | 0.3 / 9 |
| 20 | 锁文件与 CI 安装命令（`npm ci` / `--frozen-lockfile`） | 0.2 |
