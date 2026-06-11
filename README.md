# clean-template

一个适合中小型 Go 单体服务的模板，核心思路是：

- 项目顶层按业务模块组织，方便导航和迁移
- 模块内部按 `api / application / domain / infra` 分层
- 在 `internal/app` 完成依赖装配，而不是在 handler 里偷拿全局变量
- 保留足够轻量的 DDD / Clean 思想，但不过度堆术语

## 适用场景

- 想保留单体应用的简单部署方式
- 业务模块较多，希望代码能按模块收口
- 希望比传统 MVC 更好测、更好维护
- 不想一上来就引入很重的 DDD 仪式感

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
│   │   └── user
│   │       ├── api
│   │       │   ├── dto.go.tmpl
│   │       │   ├── error_mapper.go.tmpl
│   │       │   ├── handler.go.tmpl
│   │       │   ├── res
│   │       │   │   └── response_code.go.tmpl
│   │       │   └── routes.go.tmpl
│   │       ├── application
│   │       │   ├── errors.go.tmpl
│   │       │   └── service.go.tmpl
│   │       ├── domain
│   │       │   ├── entity.go.tmpl
│   │       │   ├── errors.go.tmpl
│   │       │   └── repository.go.tmpl
│   │       ├── infra
│   │       │   ├── mapper.go.tmpl
│   │       │   ├── migrate.go.tmpl
│   │       │   ├── repository_gorm.go.tmpl
│   │       │   └── model
│   │       │       └── user.go.tmpl
│   │       ├── mock
│   │       │   └── repository.go.tmpl
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
- `domain`：业务实体、规则、仓储接口
- `infra`：数据库模型、仓储实现、迁移
- `mock`：面向单元测试的内存版仓储替身

### 3. 组合根在 `internal/app`

应用启动、依赖注入、路由挂载、迁移注册，都由 `internal/app` 负责。  
模块只暴露注册函数，不自己偷偷创建全局对象。

### 4. 模板是轻量的

这个模板不会强制你给每个动作都建一层，也不会要求你把值对象、领域事件一口气全上。  
先把边界立住，比术语更重要。

### 5. 返回值和错误码统一

- HTTP 层统一返回 `200`
- 业务成功失败通过 `internal/pkg/response` 的 `code / message / data` 表达
- 每个模块可以在自己的 `api/res/response_code.go` 注册业务错误码

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
├── domain
├── infra
├── mock
└── module.go
```

这样项目级按模块导航，模块内按职责治理，迁移成本会比较低。
