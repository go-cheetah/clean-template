# clean-template 设计说明

## 架构定位

这不是纯 MVC，也不是教科书式的重 DDD。  
它更接近一种：

- 模块化单体
- 模块内轻量 Clean / DDD 分层

可以把它理解成：

> 项目外层按业务模块组织，模块内按职责分层。

## 依赖方向

单个模块内部的推荐依赖方向如下：

```text
api -> application -> domain
                  -> infra(通过 domain 接口解耦)
```

再展开一点：

```text
HTTP Request
  -> api/handler
  -> application/service
  -> domain/entity + domain/repository
  -> infra/repository_gorm
  -> database
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
- `domain`
- `infra`

### 3. 把装配职责放回启动层

依赖创建和路由注册不应该散落在每个 handler 里。  
这里统一放到 `internal/app`，让它成为真正的组合根。

## 推荐目录

```text
internal/
├── app/                # 启动、装配、迁移、路由挂载
├── config/             # 配置结构
├── module/             # 业务模块
│   └── user/
│       ├── api/
│       ├── application/
│       ├── domain/
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
- 调用领域对象和仓储接口

### domain

- 定义实体
- 放核心规则
- 定义仓储接口

### infra

- 实现仓储接口
- 放 GORM 模型
- 放 mapper 和迁移

## 模板使用建议

### 模块少的时候

可以先保守一点，不必把每一层都拆很细。  
但建议至少保留：

- `api`
- `application`
- `infra`

### 模块复杂时

当单个模块越来越大，就逐步把业务规则沉到 `domain`。  
不要一开始为了“像 DDD”而生搬硬套。
