# 从一个 Demo 开始

因为 Echo 框架本身是没有 Main 入口的，我们只能通过官方给的一个小的基础 Demo，作为源码阅读的起点

## Demo - Hello World

```go
package main

import (
	"net/http"
	
	"github.com/labstack/echo/v4"
)

func main() {
	e := echo.New()
	e.GET("/", func(c echo.Context) error {
		return c.String(http.StatusOK, "Hello, World!")
	})
	e.Logger.Fatal(e.Start(":1323"))
}
```

# 解析

## 入口 - New

```go
func New() (e *Echo) {
	e = &Echo{
		Server:    new(http.Server),
		TLSServer: new(http.Server),
		AutoTLSManager: autocert.Manager{
			Prompt: autocert.AcceptTOS,
		},
		Logger:          log.New("echo"),
		colorer:         color.New(),
		maxParam:        new(int),
		ListenerNetwork: "tcp",
	}
	e.Server.Handler = e
	e.TLSServer.Handler = e
	e.HTTPErrorHandler = e.DefaultHTTPErrorHandler
	e.Binder = &DefaultBinder{}
	e.Logger.SetLevel(log.ERROR)
	e.StdLogger = stdLog.New(e.Logger.Output(), e.Logger.Prefix()+": ", 0)
	e.pool.New = func() interface{} {
		return e.NewContext(nil, nil)
	}
	e.router = NewRouter(e)
	e.routers = map[string]*Router{}
	return
}
```

该函数是整个框架的入口，也是用户调用的第一步，它返回了一个 Echo 接口，并同时初始化了几个参数

- Server
- TLSServer
- AutoTLSManager
- Logger
- colorer
- maxParam
- ListenerNetwork

### Server

Echo 其实是基于 Go Http 标准库上层开发的一个框架，因此 Server 直接 New 了一个  http.Server，我们直接查看 http.Server 的底层代码可能理解更方便些

```go
// A Server defines parameters for running an HTTP server.
// The zero value for Server is a valid configuration.
type Server struct {
    Addr           string        // TCP address to listen on, ":http" if empty
    Handler        Handler       // handler to invoke, http.DefaultServeMux if nil
    ......
}

// ListenAndServe listens on the TCP network address srv.Addr and then
// calls Serve to handle requests on incoming connections.  If
// srv.Addr is blank, ":http" is used.
func (srv *Server) ListenAndServe() error {
    addr := srv.Addr
    if addr == "" {
        addr = ":http"
    }
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
} 
```

Server 保存了运行 HTTP 服务需要的参数，也正因为如此，像启动 Echo Server 的时候完全能和标准库语法一致

```go
e.Server.Addr = ":80"
e.Server.ListenAndServe()
```

## 路由 - Method

```go
// CONNECT registers a new CONNECT route for a path with matching handler in the
// router with optional route-level middleware.
func (e *Echo) CONNECT(path string, h HandlerFunc, m ...MiddlewareFunc) *Route {
	return e.Add(http.MethodConnect, path, h, m...)
}

// DELETE registers a new DELETE route for a path with matching handler in the router
// with optional route-level middleware.
func (e *Echo) DELETE(path string, h HandlerFunc, m ...MiddlewareFunc) *Route {
	return e.Add(http.MethodDelete, path, h, m...)
}

// GET registers a new GET route for a path with matching handler in the router
// with optional route-level middleware.
func (e *Echo) GET(path string, h HandlerFunc, m ...MiddlewareFunc) *Route {
	return e.Add(http.MethodGet, path, h, m...)
}
```

关于 Echo 的 Method URL 声明，从代码中可以明显看出是对 Add 函数的一个封装

```go
// Add registers a new route for an HTTP method and path with matching handler
// in the router with optional route-level middleware.
func (e *Echo) Add(method, path string, handler HandlerFunc, middleware ...MiddlewareFunc) *Route {
	return e.add("", method, path, handler, middleware...)
}

func (e *Echo) add(host, method, path string, handler HandlerFunc, middleware ...MiddlewareFunc) *Route {
	name := handlerName(handler)
	router := e.findRouter(host)
	router.Add(method, path, func(c Context) error {
		h := applyMiddleware(handler, middleware...)
		return h(c)
	})
	r := &Route{
		Method: method,
		Path:   path,
		Name:   name,
	}
	e.router.routes[method+path] = r
	return r
}

func handlerName(h HandlerFunc) string {
	t := reflect.ValueOf(h).Type()
	if t.Kind() == reflect.Func {
		return runtime.FuncForPC(reflect.ValueOf(h).Pointer()).Name()
	}
	return t.String()
}

func (e *Echo) findRouter(host string) *Router {
	if len(e.routers) > 0 {
		if r, ok := e.routers[host]; ok {
			return r
		}
	}
	return e.router
}
```

首先使用 `runtime.FuncForPC` 反射拿到 Handler 的函数名，后面的 `e.routers` 的类型实际为 `map[string]*Router` ， `findRouter` 函数保证了我们即使定义了重复的 Router 也只会生效第一个，并在之后将传入的路由路径放到 Router 结构体当中，关于更深入的机制，就需要详细查看 Router 结构体的声明和实现，这里为了保证 Demo 解读的通顺，暂时先不深追具体的细节

在我们调用了 `e.AnyMethod` 函数之后，例如 GET、POST 等，echo 会在 `Router.router.routes` 这个 map 中追加 `method+path:r` 这对 `key:value` ， `r` 为 `*Route` ，里面包含了方法、路径和调用的函数名

而作为执行函数的 HandlerFunc，则会被抛到和中间件一起归拢的函数 `applyMiddleware` ，而这个函数的作用，就是将所在 URL 的 middleware 列表压入一个执行函数

```go
func applyMiddleware(h HandlerFunc, middleware ...MiddlewareFunc) HandlerFunc {
	for i := len(middleware) - 1; i >= 0; i-- {
		h = middleware[i](h)
	}
	return h
}
```

## Start

```go
func (e *Echo) Start(address string) error {
	e.startupMutex.Lock()
	e.Server.Addr = address
	if err := e.configureServer(e.Server); err != nil {
		e.startupMutex.Unlock()
		return err
	}
	e.startupMutex.Unlock()
	return e.serve()
}
```

Demo 的最后一行就是整个框架的启动语句了，在我们配置好了路由之后，它的启动依赖了 `http.Server{}.Serve()` 这个函数，剩下在互斥锁中的语句其实就是启动 Web 框架之前的行为了，例如显示 Logo、隐藏监听端口这些
