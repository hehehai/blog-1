# 为它加上 Swagger

项目地址：https://github.com/EDDYCJY/go-gin-example

## 涉及知识点

- Swagger

## 本文目标

一个好的 `API's`，必然离不开一个好的`API`文档，如果要开发纯手写 `API` 文档，不存在的（很难持续维护），因此我们要自动生成接口文档。

## 安装 swag

```
$ go get -u github.com/swaggo/swag/cmd/swag
```

若 `$GOROOT/bin` 没有加入`$PATH`中，你需要执行将其可执行文件移动到`$GOBIN`下

```
mv $GOPATH/bin/swag /usr/local/go/bin
```

### 验证是否安装成功

检查 \$GOBIN 下是否有 swag 文件，如下：

```
$ swag -v
swag version v1.1.1
```

## 安装 gin-swagger

```
$ go get -u github.com/swaggo/gin-swagger

$ go get -u github.com/swaggo/gin-swagger/swaggerFiles
```

注：三个包都有一定大小，安装需要等一会或要科学上网。

## 初始化

### 编写 API 注释

`Swagger` 中需要将相应的注释或注解编写到方法上，再利用生成器自动生成说明文件

`gin-swagger` 给出的范例：

```go
// @Summary Add a new pet to the store
// @Description get string by ID
// @Accept  json
// @Produce  json
// @Param   some_id     path    int     true        "Some ID"
// @Success 200 {string} string	"ok"
// @Failure 400 {object} web.APIError "We need ID!!"
// @Failure 404 {object} web.APIError "Can not find ID"
// @Router /testapi/get-string-by-int/{some_id} [get]
```

我们可以参照 `Swagger` 的注解规范和范例去编写

```go
// @Summary 新增文章标签
// @Produce  json
// @Param name query string true "Name"
// @Param state query int false "State"
// @Param created_by query int false "CreatedBy"
// @Success 200 {string} json "{"code":200,"data":{},"msg":"ok"}"
// @Router /api/v1/tags [post]
func AddTag(c *gin.Context) {
```

```go
// @Summary 修改文章标签
// @Produce  json
// @Param id path int true "ID"
// @Param name query string true "ID"
// @Param state query int false "State"
// @Param modified_by query string true "ModifiedBy"
// @Success 200 {string} json "{"code":200,"data":{},"msg":"ok"}"
// @Router /api/v1/tags/{id} [put]
func EditTag(c *gin.Context) {
```

参考的注解请参见 [go-gin-example](https://github.com/EDDYCJY/go-gin-example)。以确保获取最新的 swag 语法

### 路由

在完成了注解的编写后，我们需要针对 swagger 新增初始化动作和对应的路由规则，才可以使用。打开 routers/router.go 文件，新增内容如下：

```go
package routers

import (
	...

	_ "github.com/EDDYCJY/go-gin-example/docs"

	...
)

// InitRouter initialize routing information
func InitRouter() *gin.Engine {
	...
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
	...
	apiv1 := r.Group("/api/v1")
	apiv1.Use(jwt.JWT())
	{
		...
	}

	return r
}
```

### 生成

我们进入到`gin-blog`的项目根目录中，执行初始化命令

```
[$ gin-blog]# swag init
2018/03/13 23:32:10 Generate swagger docs....
2018/03/13 23:32:10 Generate general API Info
2018/03/13 23:32:10 create docs.go at  docs/docs.go

```

完毕后会在项目根目录下生成`docs`

```
docs/
├── docs.go
└── swagger
    ├── swagger.json
    └── swagger.yaml

```

我们可以检查 `docs.go` 文件中的 `doc` 变量，详细记载中我们文件中所编写的注解和说明
![image](https://image.eddycjy.com/37ae10e1714c63899a55d49c19af0860.png)

### 验证

大功告成，访问一下 `http://127.0.0.1:8000/swagger/index.html`， 查看 `API` 文档生成是否正确

![image](https://image.eddycjy.com/703b677c6756129c33b5308c1655a35c.png)

## 参考

### 本系列示例代码

- [go-gin-example](https://github.com/EDDYCJY/go-gin-example)

## 关于

### 修改记录

- 第一版：2018 年 02 月 16 日发布文章
- 第二版：2019 年 10 月 01 日修改文章

## ？

如果有任何疑问或错误，欢迎在 [issues](https://github.com/EDDYCJY/blog) 进行提问或给予修正意见，如果喜欢或对你有所帮助，欢迎 Star，对作者是一种鼓励和推进。

### 我的公众号

![image](https://image.eddycjy.com/8d0b0c3a11e74efd5fdfd7910257e70b.jpg)
