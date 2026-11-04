# 02 架构与模块

> 归属：AI Coding Agent 软件工程手册

## 分层

服务型项目的默认四层，依赖只能从上往下。项目已有自己的分层约定时（例如 Conductor 的 Harness -> Loop -> Graph -> Pipeline），以项目约定为准；本节只强制保留"依赖单向、不跨层调用"这一条：

```
interfaces    (HTTP / gRPC / CLI / 消息消费)
     |
application   (用例编排、事务边界、权限校验)
     |
domain        (实体、值对象、领域服务、领域规则)
     |
infrastructure (数据库、缓存、MQ、外部 API)
```

- **禁止** `domain` 依赖 `infrastructure` 或任何框架。
- **禁止** controller 写业务逻辑、service 拼 SQL、repository 做权限判断。
- **依赖倒置**：domain 需要外部能力时，在 domain 定义接口（port），由 infrastructure 实现（adapter）。
- **跨层调用必须显式**：application 直接调用 infrastructure 的具体类，说明接口定义缺失。

## 模块边界

- **一个模块一个能力**，命名反映业务语义（`payment`、`inventory`），不反映技术分层。
- **禁止万能工具模块**。`utils`、`common`、`helpers`、`misc` 是垃圾场。一个工具函数只有同时满足两点才能单独存在：与业务无关，且属于可命名的类别（时间解析、字节编码）。否则放回它服务的领域。
- **公开面最小**。模块默认私有，只导出承诺的能力。
- **单向依赖**，禁止循环。出现循环说明边界划错，用事件或端口打破。

## 边界与第三方

- **外部 SDK 走适配器**。核心逻辑不 import 具体 SDK；适配器负责错误转换、重试、超时、鉴权。
- **对依赖写 learning test**。接入第三方库时先写一组验证其行为的测试，锁定版本升级前后的差异。
- **框架类型不进领域层**。`flask.request`、Django 的 `settings`、`actix_web::HttpRequest` 只允许出现在 interfaces 层。

## 目录结构

Python 项目：

```
project/
  apps/           # 可部署入口：api、worker、scheduler
  domain/         # 按业务划分：user、order、payment
  application/    # services、commands、queries
  infrastructure/ # db、cache、mq、external
  interfaces/     # http、grpc、cli
  tests/
  docs/
  scripts/
```

Node 项目：

```
src/
  modules/        # 按业务划分：user、auth、payment
  shared/         # 跨模块基础设施：logger、errors、config
  infrastructure/
  interfaces/
tests/
```

结构可以不同，约束不变：业务代码按领域聚拢，技术细节集中在 infrastructure，入口只做编排。

## 构造与使用分离

- 依赖在入口（`main`、容器、DI 框架）一次性装配，业务代码只接收已构造好的依赖。
- 禁止在业务函数内部 `new` 数据库连接、读全局配置、创建 HTTP 客户端。
- 禁止文件级可变全局状态。确需全局的，用显式单例容器并限定读写位置。
