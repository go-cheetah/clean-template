# 项目架构约定

## 架构定位

本项目采用模块化单体和简洁功能分层。重点是职责清晰、依赖单向，不要求套用 DDD 术语或模式。

## 目录职责

每个业务模块位于 `internal/module/<module>`，按需要使用以下目录：

- `api`：HTTP 参数、响应 DTO、handler、路由和错误码转换。
- `application`：用例编排；定义本模块执行用例所需的接口。
- `core`：模块内部稳定的数据结构、通用规则、错误和仓储接口。
- `infra`：数据库、外部服务及其他模块契约的适配实现。
- `contract`：本模块明确提供给其他模块使用的稳定接口、DTO 和边界错误。
- `mock`：测试替身。
- `module.go`：模块内部装配及对外入口。

`internal/app` 是应用组合根，负责创建模块、注入跨模块依赖、注册迁移和挂载 HTTP 路由。

## 依赖规则

模块内部依赖方向：

```text
api -> application -> core
infra -----------> application/core
```

跨模块依赖方向：

```text
consumer/application <- consumer/infra -> provider/contract
```

必须遵守以下规则：

1. `api` 不直接访问数据库或其他模块。
2. `application` 不依赖 Gin、GORM、数据库模型或其他模块的内部包。
3. `core` 不依赖 `api`、`application`、`infra` 或其他模块。
4. `infra` 实现 `application` 或 `core` 定义的接口。
5. 其他模块只能导入提供方的 `contract`，不得导入其 `api`、`application`、`core`、`infra` 或 `mock`。
6. 消费方通过自己的 `infra` adapter 将外部 `contract` 转换成本模块 `application` 所需的接口和数据结构。
7. 跨模块依赖统一在 `internal/app` 装配，不使用全局变量或在 handler 中临时创建依赖。

## contract 使用条件

只有模块能力确实被其他模块使用时才创建 `contract`。`contract` 不是每个模块的必备目录，也不能代替 `core`。

`contract` 应保持最小：

- 只暴露调用方真正需要的接口和字段。
- 不暴露 `core` 类型、数据库模型或框架类型。
- 接口按能力拆分，避免提供包含大量无关方法的总接口。
- 对外错误必须是调用方可稳定判断的边界错误。

本模板中的调用链是示范实现：

```text
audit -> audit/infra/user_adapter -> user/contract
                                      -> user facade
                                      -> user/application
```

## 开发约定

- 新增用例优先放入现有模块的 `application`，不要先增加新的抽象层。
- 只有稳定、与技术设施无关且被多处使用的模型或规则才放入 `core`。
- 新增模块时由 `module.go` 收口构造过程，并在 `internal/app` 完成最终装配。
- 修改公开 `contract` 时，同时检查所有消费模块及 adapter。
- 测试重点覆盖 application 用例、adapter 转换和 contract 兼容性。
