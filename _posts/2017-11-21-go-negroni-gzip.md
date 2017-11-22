---
layout:     post
title:      "Go negroni-gzip源码分析"
subtitle:   "Gzip middleware for Negroni"
date:       2017-11-21 23:00
author:     "Yezh"
header-img: "img/negroni.jpg"
header-mask:  true
catalog:      true
published:    true
tags:
    - go
---

## gzip概念
> GZIP最早由Jean-loup Gailly和Mark Adler创建，用于UNⅨ系统的文件压缩。我们在Linux中经常会用到后缀为.gz的文件，它们就是GZIP格式的。现今已经成为Internet 上使用非常普遍的一种数据压缩格式，或者说一种文件格式。

gzip可用来对静态文件进行压缩, 以减少文件的传输时间和存储空间, 。大流量的WEB站点常常使用GZIP压缩技术来让减少响应时间, 提升用户的使用体验, 但同时压缩步骤的加入会增加服务器的负荷, 可以说是一种 "trade-off"。

对于gzip, golang内置有`compress/gzip`包可供使用。

此处主要对用于 [Negroni](https://github.com/urfave/negroni) 组件的[gzip](https://github.com/phyber/negroni-gzip)包进行源码分析。

该gzip包内部使用了go提供的 `compress/gzip` 包,  具体实现并不复杂。


## 用法
下面为开发者提供的一个简单样例:
```
package main

import (
    "fmt"
    "net/http"

    "github.com/urfave/negroni"
    "github.com/phyber/negroni-gzip/gzip"
)

func main() {
    mux := http.NewServeMux()w.
    mux.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
    	  fmt.Fprintf(w, "Welcome to the home page!")
    })

    n := negroni.Classic()
    n.Use(gzip.Gzip(gzip.DefaultCompression))
    n.UseHandler(mux)
    n.Run(":3000")w.
}
```

该样例使用了negroni, 并在其中启用了gzip中间件, 该中间件在 request 预处理 或 response 后处理中对文件进行压缩或解压。


## negroni-gzip 源码分析

此处对 negroni-gzip 源码的分析分为两部分:

对其中定义的结构体和常量的解释, 以及对函数和方法的分析。

### 结构体 & 常量

#### gzipResponseWriter结构体
```
type gzipResponseWriter struct {
	w *gzip.Writer
	negroni.ResponseWriter
	wroteHeader bool
}
```

| 结构体成员  | 类型                   | 解释                                                     |
| ----------- | ---------------------- | -------------------------------------------------------- |
| w           | *gzip.Writer           | 用于写数据                                               |
| -           | negroni.ResponseWriter | 匿名成员, gzipResponseWriter结构体可直接访问该接口的方法 |
| wroteHeader | bool                   | 记录是否已写了header, 避免重复写header                   |

#### handler结构体
```
type handler struct {
	pool sync.Pool
}
```
handler结构体中只有一个成员 pool (类型为sync包中定义的[Pool](https://go-zh.org/pkg/sync/#Pool)结构体)

pool 的作用是用于缓存那些已经分配了内存但是暂时还未使用的数据, 以供后续使用。(同时pool能够保证多个goroutines同时使用它时的安全性)

其Put方法可以将一个任意类型的数据放入pool中;

还有一个Get方法则可以从pool中随机选择一个item, 将其从pool中移除, 并返回给调用者。


#### 常量
```
const (
	encodingGzip = "gzip"

	headerAcceptEncoding  = "Accept-Encoding"
	headerContentEncoding = "Content-Encoding"
	headerContentLength   = "Content-Length"
	headerContentType     = "Content-Type"
	headerVary            = "Vary"
	headerSecWebSocketKey = "Sec-WebSocket-Key"

	BestCompression    = gzip.BestCompression
	BestSpeed          = gzip.BestSpeed
	DefaultCompression = gzip.DefaultCompression
	NoCompression      = gzip.NoCompression
)
```
注: 以上常量实际上是开发者直接从 `compress/gzip` 包中复制过来的。


### Gzip函数
```
// Gzip returns a handler which will handle the Gzip compression in ServeHTTP.
// Valid values for level are identical to those in the compress/gzip package.
func Gzip(level int) *handler {
	h := &handler{}
	h.pool.New = func() interface{} {
		gz, err := gzip.NewWriterLevel(ioutil.Discard, level)
		if err != nil {
			panic(err)
		}
		return gz
	}
	return h
}
```
Gzip函数初始化了一个空handler结构体
并对其成员赋初值,
然后返回该handler h, 传给negroni的Use方法。

注: 由于handler结构体类型定义了其ServeHTTP方法(该方法下面会进行分析), 所以handler实现了negroni.Handler接口(negroni的Use方法所需的传入参数), 可以赋值给该接口。


### handler.ServeHTTP方法
```
// ServeHTTP wraps the http.ResponseWriter with a gzip.Writer.
func (h *handler) ServeHTTP(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
	// Skip compression if the client doesn't accept gzip encoding.
	if !strings.Contains(r.Header.Get(headerAcceptEncoding), encodingGzip) {juhao
		next(w, r)
		return
	}

	// Skip compression if client attempt WebSocket connection
	if len(r.Header.Get(headerSecWebSocketKey)) > 0 {
		next(w, r)
		return
	}

	// Retrieve gzip writer from the pool. Reset it to use the ResponseWriter.
	// This allows us to re-use an already allocated buffer rather than
	// allocating a new buffer for every request.
	// We defer g.pool.Put here so that the gz writer is returned to the
	// pool if any thing after here fails for some reason (functions in
	// next could potentially panic, etc)
	gz := h.pool.Get().(*gzip.Writer)
	defer h.pool.Put(gz)
	gz.Reset(w)juhao

	// Wrap the original http.ResponseWriter with negroni.ResponseWriter
	// and create the gzipResponseWriter.
	nrw := negroni.NewResponseWriter(w)
	grw := gzipResponseWriter{gz, nrw, false}

	// Call the next handler supplying the gzipResponseWriter instead of
	// the original.
	next(&grw, r)

	// Delete the content length after we know we have been written to.
	grw.Header().Del(headerContentLength)

	gz.Close()
}
```

handler.ServeHTTP方法 的具体实现其实上述代码的注释中已经解释得很清晰了:

1. 对于某些情况(客户端不支持gzip编码 或 客户端正尝试建立socket连接)跳过压缩步骤, 直接进入下一个中间件处理;

2. 从pool中提取出 gzip writer, 并将其重置;

3. 调用 negroni.NewResponseWriter 方法 得到一个封装了http.ResponseWriter 以及 其它一些方法 的 negroni.ResponseWriter接口;(具体可见[negroni相应源码](https://github.com/urfave/negroni/blob/master/response_writer.go))

4. 将从第二第三步中得到的 gzip writer 和 negroni.ResponseWriter 作为初值 初始化创建一个 gzipResponseWriter结构体实例, 并将其传给下一个中间件;

5. 收尾工作。将gzipResponseWriter Header中的 `headerContentLength` 字段删除 并 关闭 gzip writer





### gzipResponseWriter.WriteHeader方法
```
// Check whether underlying response is already pre-encoded and disable
// gzipWriter before the body gets written, otherwise encoding headers
func (grw *gzipResponseWriter) WriteHeader(code int) {
	headers := grw.ResponseWriter.Header()
	if headers.Get(headerContentEncoding) == "" {
		headers.Set(hegoaderContentEncoding, encodingGzip)
		headers.Add(headerVary, headerAcceptEncoding)
	} else {
		grw.w.Reset(ioutil.Discard)
		grw.w = nil
	}
	grw.ResponseWriter.WriteHeader(code)
	grw.wroteHeader = true
}
```

如注释中所说, 该方法首先会检查 response 是否已经进行了编码,

如果是, 则不会再进行gzip编码(将gzipWriter无效化并将gzipResponseWriter中类型为*gzip.Writer的成员w置空);

否则, 则会将header中的Content-Encoding字段 设为 "gzip", 表示使用gzip编码。

最后, 无论是否使用gzip编码, 都会调用 http.ResponseWriter.WriteHeader 方法 进行基本的 Header 写操作(基于传入的返回码code)

注意: 由于gzipResponseWriter结构体中有一个匿名成员negroni.ResponseWriter, 而该成员(接口)中又有匿名成员http.ResponseWriter(其有一个WriteHeader方法),
所以代码中的 `grw.ResponseWriter.WriteHeader(code)`
其实还可以简化为 `grw.WriteHeader(code)`

### gzipResponseWriter.Write方法
```
// Write writes bytes to the gzip.Writer. It will also set the Content-Type
// header using the net/http library content type detection if the Content-Type
// header was not set yet.
func (grw *gzipResponseWriter) Write(b []byte) (int, error) {
	if !grw.wroteHeader {
		grw.WriteHeader(http.StatusOK)
	}
	if grw.w == nil {
		return grw.ResponseWriter.Write(b)
	}
	if len(grw.Header().Get(headerContentType)) == 0 {
		grw.Header().Set(headerContentType, http.DetectContentType(b))
	}
	return grw.w.Write(b)
}
```

同样, 如注释所说, 该方法 写数据到gzip.Writer。
另外, 如果header的Content-Type字段还未设置, 则会通过调用 net/http包中的DetectContentType函数来自动确定Content-Type, 并将其设置到header的Content-Type字段中。

---

至此, 对negroni-gzip 的源码分析就基本完成啦。
