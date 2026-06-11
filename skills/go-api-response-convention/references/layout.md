# 布局约定

当需要判断响应相关代码应该放在哪里时，读取这个文件。

## 推荐目录结构

```text
internal/
├── pkg/
│   └── response/
│       ├── common.go
│       └── response.go
└── module/
    └── <module>/
        ├── api/
        │   ├── handler.go
        │   ├── res/
        │   │   └── response_code.go
        │   └── routes.go
        ├── application/
        │   └── errors.go
        └── domain/
            └── errors.go
```

## 职责划分

- `internal/pkg/response`
  公共响应结构、公共错误码、注册表和通用辅助函数。
- `domain/errors.go`
  业务事实类错误，例如不存在、状态非法、必填字段缺失。
- `application/errors.go`
  用例层错误，例如资源重复、业务冲突。
- `api/res/response_code.go`
  模块对外使用的响应码和响应文案。
- `api/handler.go`
  负责收参、调 service、回响应。简单场景直接返回业务码；只有在错误翻译重复明显时再抽 mapper。

## 默认错误码范围

- `0`: success
- `4xxxx`: request or parameter problems
- `5xxxx`: infrastructure or unknown server problems
- 模块业务码：在整个服务内保持唯一，并按模块成组分配，例如 `10001`、`10002`、`10003`
