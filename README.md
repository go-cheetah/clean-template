# clean-template

一个适合中小型 Go 单体服务的模板，核心思路是：

- 项目顶层按业务模块组织，方便导航和迁移
- 模块内部按 `api / application / core / infra` 分层，按需提供跨模块 `contract`
- 在 `internal/app` 完成依赖装配，而不是在 handler 里偷拿全局变量
- 保持清晰的依赖边界，但不过度堆叠架构术语

## 适用场景

- 想保留单体应用的简单部署方式
- 业务模块较多，希望代码能按模块收口
- 希望比传统 MVC 更好测、更好维护
- 希望采用简洁、清晰的功能分层

## 目录结构

```text
.
├── build
│   └── Dockerfile.tmpl
├── cmd
│   └── app
│       ├── app
│       │   ├── options
│       │   │   └── options.go.tmpl
│       │   └── server.go.tmpl
│       └── main.go.tmpl
├── config
│   └── config.yaml.tmpl
├── docs
│   └── design.md
├── internal
│   ├── app
│   │   ├── app.go.tmpl
│   │   ├── http.go.tmpl
│   │   └── providers.go.tmpl
│   ├── config
│   │   ├── config.go.tmpl
│   │   ├── db.go.tmpl
│   │   ├── log.go.tmpl
│   │   └── server.go.tmpl
│   ├── module
│   │   ├── audit
│   │   │   ├── api
│   │   │   ├── application
│   │   │   ├── infra
│   │   │   │   └── user_adapter.go.tmpl
│   │   │   └── module.go.tmpl
│   │   └── user
│   │       ├── api
│   │       │   ├── dto.go.tmpl
│   │       │   ├── handler.go.tmpl
│   │       │   ├── res
│   │       │   │   └── response_code.go.tmpl
│   │       │   └── routes.go.tmpl
│   │       ├── application
│   │       │   ├── errors.go.tmpl
│   │       │   └── service.go.tmpl
│   │       ├── core
│   │       │   ├── user.go.tmpl
│   │       │   ├── errors.go.tmpl
│   │       │   └── repository.go.tmpl
│   │       ├── contract
│   │       │   └── user.go.tmpl
│   │       ├── infra
│   │       │   ├── mapper.go.tmpl
│   │       │   ├── migrate.go.tmpl
│   │       │   ├── repository_gorm.go.tmpl
│   │       │   └── model
│   │       │       └── user.go.tmpl
│   │       ├── mock
│   │       │   └── repository.go.tmpl
│   │       ├── contract.go.tmpl
│   │       └── module.go.tmpl
│   └── pkg
│       ├── migration
│       └── response
├── pkg
│   ├── cors
│   ├── database
│   ├── generator
│   └── log
├── Makefile.tmpl
└── go.mod.tmpl
```

## 设计重点

### 1. 顶层按模块看

像 `user`、`server`、`installation` 这种业务，应该优先按模块归档，而不是先把所有 handler、service、repo 打散到全局目录。

### 2. 模块内部再分层

以 `internal/module/user` 为例：

- `api`：HTTP 入参、出参、handler、路由
- `application`：用例编排
- `core`：核心数据结构、通用规则、仓储接口
- `infra`：数据库模型、仓储实现、迁移
- `contract`：仅在跨模块调用时提供稳定接口和 DTO
- `mock`：面向单元测试的内存版仓储替身

`audit` 模块通过 `user/contract` 获取用户信息，并在自己的 `infra/user_adapter.go` 中完成模型和错误转换。它用于演示模块间只依赖公开契约，不直接引用对方内部实现。

### 3. 组合根在 `internal/app`

应用启动、依赖注入、路由挂载、迁移注册，都由 `internal/app` 负责。  
模块通过 `Module` 暴露注册函数和必要契约，不自己创建全局对象。

### 4. 模板是轻量的

这个模板不会强制为每个动作增加抽象层。
先按功能划清边界，再根据复杂度逐步拆分。

### 5. 返回值和错误码统一

- HTTP 层统一返回 `200`
- 业务成功失败通过 `internal/pkg/response` 的 `code / message / data` 表达
- 每个模块可以在自己的 `api/res/response_code.go` 注册业务错误码
- 简单模块可以直接在 handler 里返回业务码，只有在复用价值明显时再抽错误映射

## 启动流程

```text
cmd/app/main.go
  -> cmd/app/app/server.go
  -> cmd/app/app/options/options.go
  -> internal/app/app.go
```

启动时主要完成：

1. 加载配置
2. 初始化 Gin、日志、数据库
3. 注册迁移
4. 注册各业务模块的 HTTP 路由
5. 启动服务

## 模块扩展示例

如果你要新增 `server` 模块，可以复制 `internal/module/user` 的结构，改成：

```text
internal/module/server/
├── api
├── application
├── core
├── contract          # 仅在被其他模块调用时创建
├── infra
├── mock
└── module.go
```

这样项目级按模块导航，模块内按职责治理，迁移成本会比较低。

更完整的架构使用规则见 [`AGENTS.md`](AGENTS.md)。
