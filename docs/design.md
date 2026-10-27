# clean-template 设计说明

## 架构定位

这不是纯 MVC，也不强调业务领域建模。
它采用的是：

- 模块化单体
- 模块内简洁的功能分层

可以把它理解成：

> 项目外层按业务模块组织，模块内按职责分层。

## 依赖方向

单个模块内部的推荐依赖方向如下：

```text
api -> application -> core
infra -----------> application/core
```

再展开一点：

```text
HTTP Request
  -> api/handler
  -> application/service
  -> core/repository interface
  -> infra/repository_gorm implementation
  -> database
```

跨模块调用使用提供方公开的 `contract`，并由消费方的 `infra` 负责适配：

```text
audit/application <- audit/infra/user_adapter -> user/contract
```

## 为什么这样设计

### 1. 先解决“找代码困难”

很多项目随着规模增长，会出现：

- handler 到处都是
- service 到处都是
- repo 到处都是

最后一个功能要跨很多目录来回跳。  
所以这里先按模块收口，再在模块内部做细分。

### 2. 再解决“依赖失控”

如果只做模块目录，不做内部边界，模块很快又会变成一个大包。  
所以模块内部继续分成：

- `api`
- `application`
- `core`
- `infra`
- `contract`（仅在存在跨模块消费者时）

### 3. 把装配职责放回启动层

依赖创建和路由注册不应该散落在每个 handler 里。  
这里统一放到 `internal/app`，让它成为真正的组合根。

## 推荐目录

```text
internal/
├── app/                # 启动、装配、迁移、路由挂载
├── config/             # 配置结构
├── module/             # 业务模块
│   ├── audit/
│   │   ├── api/
│   │   ├── application/
│   │   ├── infra/
│   │   └── module.go
│   └── user/
│       ├── api/
│       ├── application/
│       ├── core/
│       ├── contract/
│       ├── infra/
│       └── module.go
└── pkg/                # 内部共享小能力，例如 response、migration
```

## 各层职责

### api

- 接收 HTTP 请求
- 做参数绑定和响应转换
- 不直接写数据库逻辑

### application

- 编排一个完整用例
- 处理流程顺序和默认值
- 调用核心模型和仓储接口

### core

- 定义核心数据结构
- 放与技术设施无关的通用规则
- 定义仓储接口

### infra

- 实现仓储接口
- 放 GORM 模型
- 放 mapper 和迁移
- 适配其他模块公开的 contract

### contract

- 定义本模块对其他模块公开的最小接口和 DTO
- 不暴露 core 类型、数据库模型和框架类型
- 没有跨模块消费者时不创建

## 跨模块示例

`user` 模块通过 `UserReader` contract 提供最小用户信息。`audit` 模块不直接引用 user 的 application、core 或 infra，而是在自己的 infra 中将该 contract 适配为 application 所需的 `UserReader`。

两个模块由 `internal/app` 创建和连接，因此模块内部不需要全局变量，也不负责寻找自己的外部依赖。

## 模板使用建议

### 模块少的时候

可以先保守一点，不必把每一层都拆很细。  
但建议至少保留：

- `api`
- `application`
- `infra`

### 模块复杂时

当单个模块越来越大，就逐步把稳定的模型和规则沉到 `core`。
不要为了形式增加不必要的抽象。
