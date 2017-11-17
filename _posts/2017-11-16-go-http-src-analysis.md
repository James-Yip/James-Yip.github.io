---
layout:     post
title:      "Golang http包源码分析"
subtitle:   "对于web服务器搭建, http包的内部实现"
date:       2017-11-16 23:00
author:     "Yezh"
header-img: "img/404-bg.jpg"
header-mask: true
catalog:     true
tags:
    - go
    - http
---

## 搭建简单web服务器
利用 Go 内置的 net/http 包, 可以快速搭建一个简单的web服务器:

main.go

```
package main

import (
    "fmt"
    "net/http"
    "strings"
    "log"
)

func handleRequest(w http.ResponseWriter, r *http.Request) {
    // your handle function implementation
    // blabla...
}

func main() {
    // set route
    http.HandleFunc("/", handleRequest)
    fmt.Println("Listening at port 9090...")
    // listen at port 9090 and serve requests
    err := http.ListenAndServe(":9090", nil)
    if err != nil {

        log.Fatal("ListenAndServe: ", err)
    }
}
```
上面代码的运行结果就不贴出来了(因为也没什么输出hhh)

接下来将以上面的程序为例, 具体分析http包在内部是如何实现的。

## http包内部实现
在上面给出的代码中, 实际上关键的函数就两个:
```
http.HandleFunc("/", handleRequest)

http.ListenAndServe(":9090", nil)
```
其功能正如注释中所说, 注册处理函数, 监听端口, 提供服务(当客户端 request url 的路径为 `/` 时, 调用handleRequest函数)

---

为了了解http包的内部实现, 我们需要知道这两个函数内部究竟做了什么。

具体源码可在当前 `$GOROOT` 下的 src 目录下找到,
以我的电脑为例(Ubuntu 16.04), 源码的具体路径为:

 `/usr/lib/go-1.6/src/net/http/server.go`

注意事项:
1. 提供的源码路径仅供参考, 可能以后有了新版本的go, 文件目录结构又变了, 但一般内置的库都是放在 `$GOROOT` 路径下的;
2. http目录下有两个同名文件server.go, 我们要研究的源码在 package 为 http 的 server.go 文件中 (另一个的 package 为 httptest)。

---

### 内置结构体 & 类型 定义
在对函数进行分析前, 先列出其中出现的一些结构体以及接口等的类型定义:

#### Handler 接口

```
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
该接口只定义了一个方法, 也就是说, 只要一个类型实现了这个 `ServeHTTP` 方法, 我们就可以把该类型的变量直接赋值给 Handler接口, 这在后面的具体内部实现中多次出现, 很有用。

#### ServeMux 结构体
```
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	hosts bool // whether any patterns contain hostnames
}

type muxEntry struct {
	explicit bool
	h        Handler
	pattern  string
}

// NewServeMux allocates and returns a new ServeMux.
func NewServeMux() *ServeMux { return &ServeMux{m: make(map[string]muxEntry)} }

var DefaultServeMux = NewServeMux()
```


1. ServeMux结构说明
    ```
    ServeMux
        mu    : 用于并发控制的锁
        m     : 存有muxEntry的map (key 为 pattern)
        hosts : 用于判断模式中是否包含主机名
    ```
2. muxEntry结构说明
    ```
    // 每一个 muxEntry 对应一个处理函数
    muxEntry
        explicit : 当为 true 时表明该 pattern 已注册, 防止再次有相同 pattern 进行注册
        h        : 处理函数 (如本例中的 handleRequest 函数)
        pattern  : 模式 (如本例中的 "/")
    ```
3. DefaultServeMux 为默认的 `HTTP request multiplexer`, 当服务端程序未指明特定的 multiplexer 时, 就会使用这个http包中声明好的 ServeMux 实例。


对于 ServeMux 结构体, http包中的注释如下所示:
> ServeMux is an HTTP request multiplexer.
 It matches the URL of each incoming request against a list of registered
 patterns and calls the handler for the pattern that
 most closely matches the URL.

简单来说, ServeMux这个结构体主要存储了不同 pattern 对应的逻辑处理函数 (存储于一个map中)。

对于每一个请求, 将其 url 与已注册 pattern 匹配, 若匹配成功, 则调用相应的处理函数。



#### HandlerFunc类型
```
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```
HandlerFunc类型实现了ServeHTTP方法, 因此可以赋值给前面提到的Handler接口

**注意:** 区分 HandlerFunc(类型) 和 HandleFunc(函数)

#### serverHandler结构体
```
type serverHandler struct {
    srv *Server
}
```
该结构体类型也实现了ServeHTTP方法, 也可以赋值给Handler接口

---

<br/>

下面进行具体的源码分析:

### http.HandleFunc
```
// HandleFunc registers the handler function for the given pattern
// in the DefaultServeMux.
// The documentation for ServeMux explains how patterns are matched.
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
```

http.HandleFunc 中,

调用了ServeMux.HandleFunc方法 (receiver 为 DefaultServeMux), 传入参数 "/" 和 handleRequest 函数

### ServeMux.HandleFunc
```
// HandleFunc registers the handler function for the given pattern.
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	mux.Handle(pattern, HandlerFunc(handler))
}
```
ServeMux.HandleFunc中,

将传入的handler (即handleRequest函数) 强制类型转换为 HandlerFunc 类型 (如前面提到的, 该类型实现了Handler接口, 可以赋值给Handler接口)

然后, 调用了 ServeMux.Handle 方法 (receiver 为 DefaultServeMux), 传入参数"/"和已变为 HandlerFunc 类型的 handleRequest 函数

### ServeMux.Handle
```
// Handle registers the handler for the given pattern.
// If a handler already exists for pattern, Handle panics.
func (mux *ServeMux) Handle(pattern string, handler Handler) {
	mux.mu.Lock()
	defer mux.mu.Unlock()

	if pattern == "" {
		panic("http: invalid pattern " + pattern)
	}
	if handler == nil {
		panic("http: nil handler")
	}
	if mux.m[pattern].explicit {
		panic("http: multiple registrations for " + pattern)
	}

	mux.m[pattern] = muxEntry{explicit: true, h: handler, pattern: pattern}

	if pattern[0] != '/' {
		mux.hosts = true
	}

	// Helpful behavior:
	// If pattern is /tree/, insert an implicit permanent redirect for /tree.
	// It can be overridden by an explicit registration.
	n := len(pattern)
	if n > 0 && pattern[n-1] == '/' && !mux.m[pattern[0:n-1]].explicit {
		// If pattern contains a host name, strip it and use remaining
		// path for redirect.
		path := pattern
		if pattern[0] != '/' {
			// In pattern, at least the last character is a '/', so
			// strings.Index can't be -1.
			path = pattern[strings.Index(pattern, "/"):]
		}
		url := &url.URL{Path: path}
		mux.m[pattern[0:n-1]] = muxEntry{h: RedirectHandler(url.String(), StatusMovedPermanently), pattern: pattern}
	}
}
```

ServeMux.Handle中,

前面主要进行了输入参数(pattern 和 handler)的合法性检测

我们重点关注这一行代码:

```
mux.m[pattern] = muxEntry{explicit: true, h: handler, pattern: pattern}
```
这行代码声明并初始化了一个muxEntry结构体, 并根据其 pattern 将其放到 DefaultServeMux 的 map `m` 中, 实现了相应路由处理函数的注册

(注: 将 explicit 设为 true 表明该 pattern 已注册, 当有相同 pattern 进行注册时, 会调用panic函数)

---

至此, main中第一个关键函数调用 `http.HandleFunc("/", HandleRequest)` 就分析完毕了, 实际上就是把 pattern("/") 和 handler(HandleRequest) 存入 DefaultServeMux 的 map 中

下面对第二个关键调用 `http.ListenAndServe(":9090", nil)` 进行分析

---

### http.ListenAndServe
```
// ListenAndServe always returns a non-nil error.
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```
http.ListenAndServe中,

首先用传入的参数 (":9090" & nil) 初始化了一个Server结构体

然后调用了Server.ListenAndServe方法 (receiver 为刚创建的server)

### Server.ListenAndServe
```
// ListenAndServe listens on the TCP network address srv.Addr and then
// calls Serve to handle requests on incoming connections.
// Accepted connections are configured to enable TCP keep-alives.
// If srv.Addr is blank, ":http" is used.
// ListenAndServe always returns a non-nil error.
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
Server.ListenAndServe中,

对给定的端口进行监听 (基于TCP协议)

并调用Server.Serve函数(receiver 为之前创建的实例 server), 传入参数 ln (调用listen函数后返回的Listener接口)


### Server.Serve
```
// Serve accepts incoming connections on the Listener l, creating a
// new service goroutine for each. The service goroutines read requests and
// then call srv.Handler to reply to them.
// Serve always returns a non-nil error.
func (srv *Server) Serve(l net.Listener) error {
	defer l.Close()
	if fn := testHookServerServe; fn != nil {
		fn(srv, l)
	}
	var tempDelay time.Duration // how long to sleep on accept failure
	if err := srv.setupHTTP2(); err != nil {
		return err
	}
	for {
		rw, e := l.Accept()
		if e != nil {
			if ne, ok := e.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return e
		}
		tempDelay = 0
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew) // before Serve can return
		go c.serve()
	}
}
```
Server.Serve中,
可将上述代码简化为:
```
for {
    rw, e := l.Accept()
    ...
    c, err := srv.newConn(rw)
    ...
    go c.serve()
}
```
Listener 接收请求

当接收到请求时, 建立新连接

并用 goroutine 运行 conn.serve (receiver 为 建立新连接后返回的 Conn指针 `c`)

### conn.serve
```
// Serve a new connection.
func (c *conn) serve() {
	c.remoteAddr = c.rwc.RemoteAddr().String()
	defer func() {
		if err := recover(); err != nil {
			const size = 64 << 10
			buf := make([]byte, size)
			buf = buf[:runtime.Stack(buf, false)]
			c.server.logf("http: panic serving %v: %v\n%s", c.remoteAddr, err, buf)
		}
		if !c.hijacked() {
			c.close()
			c.setState(c.rwc, StateClosed)
		}
	}()

	if tlsConn, ok := c.rwc.(*tls.Conn); ok {
		if d := c.server.ReadTimeout; d != 0 {
			c.rwc.SetReadDeadline(time.Now().Add(d))
		}
		if d := c.server.WriteTimeout; d != 0 {
			c.rwc.SetWriteDeadline(time.Now().Add(d))
		}
		if err := tlsConn.Handshake(); err != nil {
			c.server.logf("http: TLS handshake error from %s: %v", c.rwc.RemoteAddr(), err)
			return
		}
		c.tlsState = new(tls.ConnectionState)
		*c.tlsState = tlsConn.ConnectionState()
		if proto := c.tlsState.NegotiatedProtocol; validNPN(proto) {
			if fn := c.server.TLSNextProto[proto]; fn != nil {
				h := initNPNRequest{tlsConn, serverHandler{c.server}}
				fn(c.server, tlsConn, h)
			}
			return
		}
	}

	c.r = &connReader{r: c.rwc}
	c.bufr = newBufioReader(c.r)
	c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

	for {
		w, err := c.readRequest()
		if c.r.remain != c.server.initialReadLimitSize() {
			// If we read any bytes off the wire, we're active.
			c.setState(c.rwc, StateActive)
		}
		if err != nil {
			if err == errTooLarge {
				// Their HTTP client may or may not be
				// able to read this if we're
				// responding to them and hanging up
				// while they're still writing their
				// request.  Undefined behavior.
				io.WriteString(c.rwc, "HTTP/1.1 431 Request Header Fields Too Large\r\nContent-Type: text/plain\r\nConnection: close\r\n\r\n431 Request Header Fields Too Large")
				c.closeWriteAndWait()
				return
			}
			if err == io.EOF {
				return // don't reply
			}
			if neterr, ok := err.(net.Error); ok && neterr.Timeout() {
				return // don't reply
			}
			var publicErr string
			if v, ok := err.(badRequestError); ok {
				publicErr = ": " + string(v)
			}
			io.WriteString(c.rwc, "HTTP/1.1 400 Bad Request\r\nContent-Type: text/plain\r\nConnection: close\r\n\r\n400 Bad Request"+publicErr)
			return
		}

		// Expect 100 Continue support
		req := w.req
		if req.expectsContinue() {
			if req.ProtoAtLeast(1, 1) && req.ContentLength != 0 {
				// Wrap the Body reader with one that replies on the connection
				req.Body = &expectContinueReader{readCloser: req.Body, resp: w}
			}
		} else if req.Header.get("Expect") != "" {
			w.sendExpectationFailed()
			return
		}

		// HTTP cannot have multiple simultaneous active requests.[*]
		// Until the server replies to this request, it can't read another,
		// so we might as well run the handler in this goroutine.
		// [*] Not strictly true: HTTP pipelining.  We could let them all process
		// in parallel even if their responses need to be serialized.
		serverHandler{c.server}.ServeHTTP(w, w.req)
		if c.hijacked() {
			return
		}
		w.finishRequest()
		if !w.shouldReuseConnection() {
			if w.requestBodyLimitHit || w.closedRequestBodyEarly() {
				c.closeWriteAndWait()
			}
			return
		}
		c.setState(c.rwc, StateIdle)
	}
}
```

类似的, 我们重点关注for循环部分并将其简化为:
```
for{
  w, err := c.readRequest()
  ...
  serverHandler{c.server}.ServeHTTP(w, w.req)
  ...
  w.finishRequest()
  ...
}
```

对于同一个连接, 循环地执行读取请求, 处理请求, 完成请求三个操作

对于其中三个操作, 我们重点看处理请求部分

即: `serverHandler{c.server}.ServeHTTP(w, w.req)`

该语句调用了 serverHandler.ServeHTTP 方法

### serverHandler.ServeHTTP

```
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
        handler = globalOptionsHandler{}
    }
    handler.ServeHTTP(rw, req)
}
```
serverHandler.ServeHTTP中,

根据我们在一开始调用http.ListenAndServe函数时传入的handler参数来确定 handler (ServeMux结构体类型), 若传入为 `nil`, 则 handler 为默认的 DefaultServeMux,
然后调用 ServeMux.ServeHTTP 方法 (receiver 为handler, 本例中即为 DefaultServeMux)


### ServeMux.ServeHTTP
```
// ServeHTTP dispatches the request to the handler whose
// pattern most closely matches the request URL.
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	if r.RequestURI == "*" {
		if r.ProtoAtLeast(1, 1) {
			w.Header().Set("Connection", "close")
		}
		w.WriteHeader(StatusBadRequest)
		return
	}
	h, _ := mux.Handler(r)
	h.ServeHTTP(w, r)
}
```

ServeMux.ServeHTTP中,

我们主要关注下面两行代码:
```
h, _ := mux.Handler(r)
h.ServeHTTP(w, r)
```
第一行代码 调用了 ServeMux.Handler 方法 (此处 receiver 为 DefaultServeMux), 传入请求 `r`, 返回相应的逻辑处理函数 (HandlerFunc 类型)

第二行代码 则是执行返回的 handler 的 ServeHTTP 方法(即 `HandlerFunc.ServeHTTP`, 该方法在前面的 `内置结构体 & 类型 定义` 那节给出, 实际上就是执行了 receiver 本身)

至于 ServeMux.Handler, 其具体实现如下所示

### ServeMux.Handler
```
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
	if r.Method != "CONNECT" {
		if p := cleanPath(r.URL.Path); p != r.URL.Path {
			_, pattern = mux.handler(r.Host, p)
			url := *r.URL
			url.Path = p
			return RedirectHandler(url.String(), StatusMovedPermanently), pattern
		}
	}

	return mux.handler(r.Host, r.URL.Path)
}
```
ServeMux.Handler中,

调用了 ServeMux.handler 方法(receiver 为 DefaultServeMux), 传入请求 `r` 的 主机名 & URL的路径部分


### ServeMux.handler
```
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
    mux.mu.RLock()
    defer mux.mu.RUnlock()

    // Host-specific pattern takes precedence over generic ones
    if mux.hosts {
        h, pattern = mux.match(host + path)
    }
    if h == nil {
        h, pattern = mux.match(path)
    }
    if h == nil {
        h, pattern = NotFoundHandler(), ""
    }
    return
}
```
ServeMux.Handler中,

调用 ServeMux.match 方法, 传入相应的请求路径, 返回 匹配的 handler 和 pattern


### ServeMux.match
```
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
	var n = 0
	for k, v := range mux.m {
		if !pathMatch(k, path) {
			continue
		}
		if h == nil || len(k) > n {
			n = len(k)
			h = v.h
			pattern = v.pattern
		}
	}
	return
}
```

ServeMux.match中,

遍历 receiver (此处为 DefaultServeMux) 的 map, 对于输入的路径, 找到能与之相匹配的已注册pattern, 返回 相应的 handler 和 pattern

---

最后, 重新整理一下主要的函数调用过程:
```
main
├── http.HandleFunc
|   └── ServeMux.HandleFunc
|       └── ServeMux.Handle
└── http.ListenAndServe
    └── Server.ListenAndServe
        ├── net.Listen
        └── Server.Serve
            ├── Listener.Accept
            ├── Server.newConn
            └── conn.serve
                ├── conn.readRequest
                ├── serverHandler.ServeHTTP
                |   └── ServeMux.ServeHTTP
                |       ├── ServeMux.Handler
                |       |   └── ServeMux.handler
                |       |       └── ServeMux.match
                |       └── HandlerFunc.ServeHTTP
                └── response.finishRequest
```

至此, 对于实现web服务器搭建部分的http源码就基本分析完啦。

注: 代码高亮暂时存在问题, 等以后解决了再更新。(貌似暂不支持golang语法高亮)
