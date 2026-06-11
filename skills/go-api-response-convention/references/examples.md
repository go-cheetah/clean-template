# 示例

只有在需要具体实现形状时再读这个文件。

## 公共 response 包

```go
package response

type Response struct {
	Code    int         `json:"code"`
	Message string      `json:"message"`
	Data    interface{} `json:"data,omitempty"`
}

func Error(code int) Response {
	return Response{Code: code, Message: getMessage(code)}
}

func ErrorUnknown(code int, data string) Response {
	return Response{Code: code, Message: data}
}

func Success(data interface{}) Response {
	return Response{Code: CodeSuccess, Message: getMessage(CodeSuccess), Data: data}
}
```

## 模块错误码

```go
package res

import "{{ .Title }}/internal/pkg/response"

const (
	ErrUserNotFound      = 10001
	ErrUserAlreadyExists = 10002
)

func RegisterCode() {
	response.Register(ErrUserNotFound, "用户不存在")
	response.Register(ErrUserAlreadyExists, "用户已存在")
}
```

## 模块注册入口

```go
package user

import (
	"github.com/gin-gonic/gin"
	"gorm.io/gorm"

	"{{ .Title }}/internal/module/user/api"
	userRes "{{ .Title }}/internal/module/user/api/res"
	"{{ .Title }}/internal/module/user/application"
	"{{ .Title }}/internal/module/user/infra"
)

type Dependencies struct {
	DB *gorm.DB
}

func RegisterHTTP(rg *gin.RouterGroup, dep Dependencies) {
	userRes.RegisterCode()

	repo := infra.NewUserRepository(dep.DB)
	service := application.NewService(repo)
	handler := api.NewHandler(service)

	api.RegisterRoutes(rg, handler)
}
```

## Handler 用法

```go
result, err := h.service.Create(c.Request.Context(), req.Name, req.Email)
if err != nil {
	switch {
	case errors.Is(err, application.ErrUserAlreadyExists):
		c.JSON(http.StatusOK, response.Error(res.ErrUserAlreadyExists))
	case errors.Is(err, domain.ErrUserNameRequired):
		c.JSON(http.StatusOK, response.Error(res.ErrUserNameRequired))
	default:
		c.JSON(http.StatusOK, response.ErrorUnknown(response.ErrSQL, err.Error()))
	}
	return
}
```

## Handler 简单场景

```go
if err := c.ShouldBindJSON(&req); err != nil {
	c.JSON(http.StatusOK, response.Error(response.ErrCodeParameter))
	return
}

result, err := h.service.Create(c.Request.Context(), req.Name, req.Email)
if err != nil {
	c.JSON(http.StatusOK, response.ErrorUnknown(response.ErrSQL, err.Error()))
	return
}

c.JSON(http.StatusOK, response.Success(toResponse(result)))
```
