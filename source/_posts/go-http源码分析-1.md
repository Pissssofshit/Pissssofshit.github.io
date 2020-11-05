---
title: go-http源码分析(1)
date: 2020-11-03 15:53:07
tags:
---

众所周知go用一行代码就可以实现一个web server,如下

    http.ListenAndServe(":8080",nil)

这个ListendAndServe的实现也很简单

    func ListenAndServe(addr string, handler Handler) error {
    	server := &Server{Addr: addr, Handler: handler}
    	return server.ListenAndServe()
    }

addr好理解，定义了web服务器的ip、端口，那么这个Handler是什么呢？实际上这是一个接口，定义如下

    type Handler interface {
    	ServeHTTP(ResponseWriter, *Request)
    }

只有一个方法，就可以实现~

要搞懂它是干嘛的，我们还要再顺着它的调用往下看几个函数，放心，不长

    func (srv *Server) Serve(l net.Listener) error {
    
    	//有删除
    
    	origListener := l
    	l = &onceCloseListener{Listener: l}
    	defer l.Close()
    
    	//有删除
    
    	var tempDelay time.Duration // how long to sleep on accept failure
    
    	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    	for {
    		rw, err := l.Accept()
    		if err != nil {
    		//有删除
    		//错误处理
    		}
    		connCtx := ctx
    		if cc := srv.ConnContext; cc != nil {
    			connCtx = cc(connCtx, rw)
    			if connCtx == nil {
    				panic("ConnContext returned nil")
    			}
    		}
    		tempDelay = 0
    		c := srv.newConn(rw)
    		c.setState(c.rwc, StateNew) // before Serve can return
    		// 一个连接一个协程，体现了go的高并发
    		go c.serve(connCtx)
    	}
    }

再往下看c.serve方法:

    func (c *conn) serve(ctx context.Context) {
    	c.remoteAddr = c.rwc.RemoteAddr().String()
    	ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
    
    	//有删除,连接结束清除工作
    
    	//有删除 tls 握手验证
    
    	// HTTP/1.x from here on.
    
    	ctx, cancelCtx := context.WithCancel(ctx)
    	c.cancelCtx = cancelCtx
    	defer cancelCtx()
    
    	c.r = &connReader{conn: c}
    	c.bufr = newBufioReader(c.r)
    	c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)
    
    	for {
    		w, err := c.readRequest(ctx)
    		if c.r.remain != c.server.initialReadLimitSize() {
    			// If we read any bytes off the wire, we're active.
    			c.setState(c.rwc, StateActive)
    		}
    		if err != nil {
    		// 有删除，错误处理
    		}
    
    		//有删除
    
    		//前面都是建立连接，确认协议，设置连接状态等等等
    		//这里是真正处理请求的地方
    		serverHandler{c.server}.ServeHTTP(w, w.req)
    
    		w.cancelCtx()
    		if c.hijacked() {
    			return
    		}
    		w.finishRequest()
    		//有删除，关闭连接等操作
    	}
    }

再把serverHandler拎出来

    type serverHandler struct {
    	srv *Server
    }
    
    func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    	handler := sh.srv.Handler
    	//这里查询server是否设置过handle方法,如果没有,使用默认的DefaultServeMux,所以回过头去看第一行代码，是不是明白了为啥只传一个参数也可以呢？
    	if handler == nil {
    		handler = DefaultServeMux
    	}
    	if req.RequestURI == "*" && req.Method == "OPTIONS" {
    		handler = globalOptionsHandler{}
    	}
    	//看这句
    	handler.ServeHTTP(rw, req)
    }

看到了吗，其实这么一长串调用下来，请求最终交给了Handle的ServeHTTP方法
那么我们可以说，这个Handle其实扮演的就是路由的角色（是他将每一个请求分配到了对应的方法）

这里啰嗦一下，这里的Handler是一个接口，只要我们实现了这个接口的方法就可以实现自己的路由

那么首先，让我们来看看这个默认的handler的结构是怎么样的

    type ServeMux struct {
    	mu    sync.RWMutex
    	m     map[string]muxEntry
    	es    []muxEntry // slice of entries sorted from longest to shortest. // 按照pattern的长度从长到短排列的
    	hosts bool       // whether any patterns contain hostnames
    }
    
    type muxEntry struct {
    	h       Handler
    	pattern string
    }
    
    // DefaultServeMux is the default ServeMux used by Serve.
    var DefaultServeMux = &defaultServeMux
    
    var defaultServeMux ServeMux

看它是怎么实现注册、分配路由的

    //注册路由
    func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    	if handler == nil {
    		panic("http: nil handler")
    	}
    	mux.Handle(pattern, HandlerFunc(handler))
    }
    // 这个方法将普通的方法转换为了满足Handle接口的方法
    type HandlerFunc func(ResponseWriter, *Request)
    func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    	//执行了自身
    	f(w, r)
    }
    
    func (mux *ServeMux) Handle(pattern string, handler Handler) {
    	mux.mu.Lock()
    	defer mux.mu.Unlock()
    
    	if pattern == "" {
    		panic("http: invalid pattern")
    	}
    	if handler == nil {
    		panic("http: nil handler")
    	}
    	if _, exist := mux.m[pattern]; exist {
    		panic("http: multiple registrations for " + pattern)
    	}
    
    	if mux.m == nil {
    		mux.m = make(map[string]muxEntry)
    	}
    	//存储到路由表里
    	e := muxEntry{h: handler, pattern: pattern}
    	mux.m[pattern] = e
    	if pattern[len(pattern)-1] == '/' {
    		mux.es = appendSorted(mux.es, e)
    	}
    
    	if pattern[0] != '/' {
    		mux.hosts = true
    	}
    }
    //分配路由
    func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    	if r.RequestURI == "*" {
    		if r.ProtoAtLeast(1, 1) {
    			w.Header().Set("Connection", "close")
    		}
    		w.WriteHeader(StatusBadRequest)
    		return
    	}
    	h, _ := mux.Handler(r)
    	//处理
    	h.ServeHTTP(w, r)
    }
    func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
    
    	//大段删除，都是处理字符串格式、判断的方法
    
    	return mux.handler(host, r.URL.Path)
    }
    
    func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
    	mux.mu.RLock()
    	defer mux.mu.RUnlock()
    
    	// Host-specific pattern takes precedence over generic ones
    	//寻找匹配的路由
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
    func (mux *ServeMux) match(path string) (h Handler, pattern string) {
    	// Check for exact match first.
    	v, ok := mux.m[path]
    	if ok {
    		return v.h, v.pattern
    	}
    
    	// Check for longest valid match.  mux.es contains all patterns
    	// that end in / sorted from longest to shortest.
    	for _, e := range mux.es {
    		if strings.HasPrefix(path, e.pattern) {
    			return e.h, e.pattern
    		}
    	}
    	return nil, ""
    }

使用默认的注册路由方法:

    http.HandleFunc("/", sayhelloName)

方法的实现是:

    func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    	DefaultServeMux.HandleFunc(pattern, handler)
    }

参考资料:
[build go web application](https://astaxie.gitbooks.io/build-web-application-with-golang/content/zh/03.3.html)
[煎鱼博客](https://eddycjy.gitbook.io/golang/di-3-ke-gin)
[http包源码分析](https://www.ulovecode.com/2019/09/03/Go/Golang%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/Go%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(1)-net.http/)

