---
title: "浅谈gin中间件"
publishDate: "21 Aug 2024"
description: "谈一谈关于gin中间件"
tags: ["gin", "blog", "golang", "middleware"]
draft: false
---


## 中间件是什么

[Wikipedia中间件](https://en.wikipedia.org/wiki/Middleware)

`"Middleware is a type of computer software program that provides services to software applications beyond those available from the operating system. It can be described as "software glue"."`

中间件本质是一段用于处理、修改，拦截Http请求和相应的代码，在许多Web框架中起到关键作用

在Gin中，中间件主要作用包括：
- 请求处理
- 响应处理
- 错误处理
- 数据处理

## Gin中的中间件是怎么工作的

Gin中，中间件通过函数链的形式工作，每个中间件接受`*gin.Context`对象，这个对象会在中间件链中进行传递。中间件会在请求到达最终处理函数之前执行一些逻辑，也可以在处理函数之后执行逻辑

以以下这段demo进行理解
``` go
// 日志中间件
func LoggerMiddleware(c *gin.Context) {
    log.Println("Request received")
    c.Next() // 继续处理下一个中间件或处理函数
    log.Println("Response sent")
}

// 认证中间件
func AuthMiddleware(c *gin.Context) {
    // 模拟认证逻辑
    log.Println("Performing authentication")
    // 这里可以根据认证结果决定是否调用 c.Next()
    c.Next() // 继续处理下一个中间件或处理函数
}

func main() {
    r := gin.Default()

    // 定义中间件函数
    r.Use(LoggerMiddleware, AuthMiddleware)

    // 定义处理函数
    r.GET("/hello", func(c *gin.Context) {
        log.Println("Handling request")
        c.JSON(200, gin.H{"message": "Hello, world!"})
    })

    r.Run()
}
```

### Demo执行流程

1. `LoggerMiddleware`中间件开始，先输出"Request received"
2. 执行日志中间件的`C.Next()`
3. 进入认证中间件的模拟认证逻辑
4. 执行认整中间件的`C.Next()`
5. 调用路由的处理函数，生成并发送 "Hello, world!" 响应
6. 响应发送后，先执行认证中间件`C.Next()`后的代码，因为在中间件中没有后续的操作所以直接进入下一步
7. 执行日志中间件`c.Next()`之后的"Respose sent"

在`LoggerMiddleware`中间件开始时，会先输出"Request received"，在执行`c.Next()`后，Gin会继续调用下一个中间件或处理函数，此时`LoggerMiddleware`中间件的`log.Println("Response sent")`还未被执行，

### 总结

1. 请求到达
2. 依次执行中间件
3. 调用处理函数
4. 响应返回


## Gin中间件源码分析

Gin中，中间件的实现依靠责任链模式，这个设计模式使得中间件可以串联起来，依次处理请求。

在Gin的默认路由中，会给你默认注册两个中间件，用于日志记录，以及捕获和处理Panic

``` go
func Default() *Engine {
	debugPrintWARNINGDefault()
    // New用于注册一个新的路由，不带任何默认的中间件
	engine := New()
	engine.Use(Logger(), Recovery())
	return engine
}
```
> 以下内容参考 [一文讲透gin中间件使用及源码解析](https://juejin.cn/post/7034338727883177997)

从上述Demo中可以看出，Gin中注册中间件使用的是`Use`方法。

```go
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
	engine.RouterGroup.Use(middleware...)
	engine.rebuild404Handlers()
	engine.rebuild405Handlers()
	return engine
}
```
它可用于接受一个边长参数`middleware`，而这个参数的类型，实际上就是`HandlerFunc`切片。

```go
type HandlerFunc func(*Context)
```

继续查看`engine.RouterGroup.Use(middleware...)` 这个路由组中的`Use`

```go
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
	group.Handlers = append(group.Handlers, middleware...)
	return group.returnObj()
}
```
发现它其实就是将中间件`HandlerFunc`追加到`group.Handlers`中，而在注册路由的时候，会将对应的路由函数和中间件函数结合到一起，最终按链的顺序执行
```go
type HandlersChain []HandlerFunc
```


```go
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	absolutePath := group.calculateAbsolutePath(relativePath)
	handlers = group.combineHandlers(handlers)  // 将处理请求的函数与中间件函数结合
	group.engine.addRoute(httpMethod, absolutePath, handlers)
	return group.returnObj()
}
```
其中结合操作的函数内容如下，注意观察这里是如何实现拼接两个切片得到一个新切片的
```go
const abortIndex int8 = math.MaxInt8 / 2

func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
	finalSize := len(group.Handlers) + len(handlers)
	if finalSize >= int(abortIndex) {  // 这里有一个最大限制
		panic("too many handlers")
	}
	mergedHandlers := make(HandlersChain, finalSize)
	copy(mergedHandlers, group.Handlers)
	copy(mergedHandlers[len(group.Handlers):], handlers)
	return mergedHandlers
}
```

而关于中间件的执行，Gin提供了四个接口

- `c.Next()`
    继续执行下一个中间件或处理函数，如不调用，则处理链会被中断，后续的中间件或处理函数不会被执行
- `c.Abort()`
    用于中止请求的处理流程，后续的中间件和处理函数不会被执行
    可以用于中间件中组织请求的进一步处理，比如返回身份验证失败的错误响应
- `c.Set(key stirng, value infetface)`
    可以存储一个键值对到`*gin.Context`用于在链处理的不同阶段共享数据给中间件或处理函数
- `c.Get(key string)`
    从`*gin.Context`中获取指定的键值对

![一文讲透gin中间件使用及源码解析](https://img.jontyding.com/jonty-imgs/2024/08/4e3596aafd9b031d7097a16201a6e9d2.png)


## 参考内容
[一文讲透gin中间件使用及源码解析 - 大师兄技术私享](https://juejin.cn/post/7034338727883177997)