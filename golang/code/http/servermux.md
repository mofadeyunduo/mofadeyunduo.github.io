## Golang 版本

1.13

## 示范

一些匹配原则：
- 路由可以是 host + path，但大多数情况都不会使用 host
- 路由 path 最后一位如果是 /，代表通配，否则是精确匹配
- 最先尝试精确匹配，利用 path 和 host + path 精确匹配，若还未匹配上，优先最长路径尝试通配

基本是如下工作的：

| 路由 | 访问 | 结果|
| :--- | :--- | :--- |
| /login/ | /login | 301 /login/ => 200 通配匹配成功 |
| /login/ | /login/ | 200 通配匹配成功 |
| /who_am_i | /who_am_i/ | 404 精确、通配匹配失败 |
| /who_am_i | /who_am_i | 200 精确匹配成功 |


可以尝试运行如下代码：
```
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	// svr 初始化
	http.HandleFunc("/login/", func(writer http.ResponseWriter, request *http.Request) {
		writer.WriteHeader(http.StatusOK)
	})
	http.HandleFunc("/who_am_i", func(writer http.ResponseWriter, request *http.Request) {
		writer.WriteHeader(http.StatusNoContent)
	})
	svr := http.Server{
		Addr:    ":8888",
		Handler: http.DefaultServeMux,
	}
	go func() {
		if err := svr.ListenAndServe(); err != nil {
			log.Println(err)
		}
	}()
	// 平滑退出
	quit := make(chan os.Signal)
	signal.Notify(quit, syscall.SIGTERM)
	select {
	case <-quit:
		break
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*5)
	defer cancel()
	if err := svr.Shutdown(ctx); err != nil {
		log.Println(err)
	}
	return
}
```

尝试用 curl 请求验证：
```
% curl -I localhost:8888/login
HTTP/1.1 301 Moved Permanently
Content-Type: text/html; charset=utf-8
Location: /login/
Date: Thu, 12 Nov 2020 09:28:42 GMT

% curl -I localhost:8888/login/
HTTP/1.1 200 OK
Date: Thu, 12 Nov 2020 09:29:14 GMT

% curl -I localhost:8888/who_am_i
HTTP/1.1 204 No Content
Date: Thu, 12 Nov 2020 09:29:32 GMT

% curl -I localhost:8888/who_am_i/
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Date: Thu, 12 Nov 2020 09:29:35 GMT
Content-Length: 19
```

## 分析

### 注册路由

路由注册分为精确匹配注册和通配注册。

精确匹配注册直接注册到 map 中，查找时 O(1) 遍历，速度很快。

通配注册注册到 slice 中，按照长度排序，查找时从头遍历，只要完全路径包含通配路径，就匹配到最长通配路径。

```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {
    // 加锁，防止并发注册
	mux.mu.Lock()
	defer mux.mu.Unlock()

	if pattern == "" {
		panic("http: invalid pattern")
	}
	if handler == nil {
		panic("http: nil handler")
	}
    // 不能重复注册
	if _, exist := mux.m[pattern]; exist {
		panic("http: multiple registrations for " + pattern)
	}

	if mux.m == nil {
		mux.m = make(map[string]muxEntry)
	}
	e := muxEntry{h: handler, pattern: pattern}
    // 大部分情况，都可以直接匹配到，注册到一个 map 中
    // 查找时，是 O(1) 的时间复杂度
	mux.m[pattern] = e
    // / 结尾的代表通配，排序并加入 es 数组中
	if pattern[len(pattern)-1] == '/' {
		mux.es = appendSorted(mux.es, e)
	}

    // 如果开头第 0 个字符不是 /，说明可能是 localhost:80/login 这样的请求，
    // 把 hosts 变为 true，表明可能是携带有 host 的路由
    // 一般不会有 host 的路由，所以通过这种标记，能提高访问效率
	if pattern[0] != '/' {
		mux.hosts = true
	}
}

func appendSorted(es []muxEntry, e muxEntry) []muxEntry {
	n := len(es)
    // 按照长度排序
	i := sort.Search(n, func(i int) bool {
		return len(es[i].pattern) < len(e.pattern)
	})
    // 经典插入技巧，效率最高
	if i == n {
		return append(es, e)
	}
	// we now know that i points at where we want to insert
	es = append(es, muxEntry{}) // try to grow the slice in place, any entry works.
	copy(es[i+1:], es[i:])      // Move shorter entries down
	es[i] = e
	return es
}
```

### 路由查询

上面已经说了路由的规则，这里就是具体代码实现。

所有的 HTTP 处理最后都走到了这个接口上。
```go
// A Handler responds to an HTTP request.
//
// ServeHTTP should write reply headers and data to the ResponseWriter
// and then return. Returning signals that the request is finished; it
// is not valid to use the ResponseWriter or read from the
// Request.Body after or concurrently with the completion of the
// ServeHTTP call.
//
// Depending on the HTTP client software, HTTP protocol version, and
// any intermediaries between the client and the Go server, it may not
// be possible to read from the Request.Body after writing to the
// ResponseWriter. Cautious handlers should read the Request.Body
// first, and then reply.
//
// Except for reading the body, handlers should not modify the
// provided Request.
//
// If ServeHTTP panics, the server (the caller of ServeHTTP) assumes
// that the effect of the panic was isolated to the active request.
// It recovers the panic, logs a stack trace to the server error log,
// and either closes the network connection or sends an HTTP/2
// RST_STREAM, depending on the HTTP protocol. To abort a handler so
// the client sees an interrupted response but the server doesn't log
// an error, panic with the value ErrAbortHandler.
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

这个接口实现的一般是 ServerMux，Mux 的含义是多路复用器。
```go
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
	// 获取处理请求的实例
	h, _ := mux.Handler(r)
	// 处理请求
	h.ServeHTTP(w, r)
}

// Handler returns the handler to use for the given request,
// consulting r.Method, r.Host, and r.URL.Path. It always returns
// a non-nil handler. If the path is not in its canonical form, the
// handler will be an internally-generated handler that redirects
// to the canonical path. If the host contains a port, it is ignored
// when matching handlers.
//
// The path and host are used unchanged for CONNECT requests.
//
// Handler also returns the registered pattern that matches the
// request or, in the case of internally-generated redirects,
// the pattern that will match after following the redirect.
//
// If there is no registered handler that applies to the request,
// Handler returns a ``page not found'' handler and an empty pattern.
// 有的时候，自带的注释是最好的帮手，请仔细阅读
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {

	// CONNECT requests are not canonicalized.
	if r.Method == "CONNECT" {
		// If r.URL.Path is /tree and its handler is not registered,
		// the /tree -> /tree/ redirect applies to CONNECT requests
		// but the path canonicalization does not.
		if u, ok := mux.redirectToPathSlash(r.URL.Host, r.URL.Path, r.URL); ok {
			return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
		}

		return mux.handler(r.Host, r.URL.Path)
	}

	// All other requests have any port stripped and path cleaned
	// before passing to mux.handler.
    // 清理一下 host 和 path，例如吧 path 中的 query 去除
	host := stripHostPort(r.Host)
	path := cleanPath(r.URL.Path)

	// If the given path is /tree and its handler is not registered,
	// redirect for /tree/.
    // /login 这样的接口如果没注册，但是 /login/ 注册了，那么会 301 /login/
	if u, ok := mux.redirectToPathSlash(host, path, r.URL); ok {
		return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
	}

    // TODO：不知道为什么这里还要判断，没有情况可以修改 path 和 r.URL.PATH，或许是为了保险起见
	if path != r.URL.Path {
		_, pattern = mux.handler(host, path)
		url := *r.URL
		url.Path = path
		return RedirectHandler(url.String(), StatusMovedPermanently), pattern
	}

    // 其他情况，进一步处理
	return mux.handler(host, r.URL.Path)
}

// handler is the main implementation of Handler.
// The path is known to be in canonical form, except for CONNECT methods.
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
	mux.mu.RLock()
	defer mux.mu.RUnlock()

	// Host-specific pattern takes precedence over generic ones
    // 如果有 localhost:80/login 这样的匹配规则，会加上 host 尝试匹配
	if mux.hosts {
		h, pattern = mux.match(host + path)
	}
    // 如果之前没匹配上，进一步匹配
	if h == nil {
		h, pattern = mux.match(path)
	}
    // 还是没匹配上，返回 404
	if h == nil {
		h, pattern = NotFoundHandler(), ""
	}
	return
}

// Find a handler on a handler map given a path string.
// Most-specific (longest) pattern wins.
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
	// Check for exact match first.
    // 进行精确匹配，不带有 / 的路径
	v, ok := mux.m[path]
	if ok {
		return v.h, v.pattern
	}

	// Check for longest valid match.  mux.es contains all patterns
	// that end in / sorted from longest to shortest.
    // 从最长路由 path 开始判断，如果最长路由 path 包含了当前 path，匹配成功
    // 这里的 es 是从长到端排过序的，如果匹配到，第一个就是最长的
	for _, e := range mux.es {
		if strings.HasPrefix(path, e.pattern) {
			return e.h, e.pattern
		}
	}
	return nil, ""
}
```