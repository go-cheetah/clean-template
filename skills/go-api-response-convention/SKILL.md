---
name: go-api-response-convention
description: 当 Go 服务需要统一 API 返回结构和模块级错误码约定时使用，尤其适用于 clean-template 风格项目，包括 internal/pkg/response、模块 api/res 错误码注册，以及按需选择是否抽错误映射层。
---

# Go API 返回约定

当用户希望做这些事情时使用这个 skill：
- 统一 Go HTTP 接口返回值
- 引入或扩展业务错误码
- 让 handler 遵循 `clean-template` 风格的返回约定
- 为新模块补齐 `response`、`api/res` 这一套结构，并按需决定是否抽错误映射

控制上下文体积。先读 [references/layout.md](references/layout.md)，只有在需要具体代码形状时再读 [references/examples.md](references/examples.md)。

## 目标约定

- HTTP handler 统一返回固定 JSON 包装结构。
- HTTP 状态码通常固定返回 `200`；业务成功失败通过 `code`、`message`、`data` 表达。
- 公共响应能力放在 `internal/pkg/response`。
- 模块级错误码放在 `internal/module/<name>/api/res/response_code.go`。
- handler 负责收参数、调用 service、写响应；如果错误翻译逻辑明显重复，再额外抽 mapper。

## 工作流程

1. 先找到现有的 response 包和当前 handler 返回风格。
2. 如果项目还没有共享 response 包，先补 `internal/pkg/response`。
3. 在 `api/res` 下新增或更新模块级错误码。
4. 先让 handler 直接返回 `response.Success`、`response.Error` 或 `response.ErrorUnknown`。
5. 在模块 HTTP 注册入口里注册模块错误码。
6. 如果 domain/application 层还是字符串错误，先引入导出的哨兵错误，再接响应码。
7. 只有在多个 handler 重复同一套错误翻译时，才考虑新增独立 mapper。
8. 格式化变更文件，并在可能的情况下执行最小必要验证。

## 约束

- 除非仓库本身已经使用混合风格，否则不要把 HTTP 传输状态和业务状态混用。
- 不要把错误码注册逻辑放进 handler。
- 不要让 mock 或测试依赖 infra 专属的响应辅助代码。
- 对用户可感知的业务错误，优先使用模块级业务码，不要一律复用 `ErrSQL`。
- 不要把 `error_mapper.go` 当成强制结构。
- `SKILL.md` 只保留高层规则，具体代码形状放到 reference 文件。

## 验证点

- 确认 `response.Init()` 是幂等的。
- 确认每个模块的响应码不冲突。
- 确认 handler 里没有临时拼装的响应结构。
- 确认新增业务错误都能落到稳定的响应消息上。
