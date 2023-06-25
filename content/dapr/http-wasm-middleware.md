---
title: "Http Wasm Middleware"
date: 2023-06-25T22:39:52+08:00
linkTitle: "Http Wasm Middleware"
type: docs
draft: false
---

本文从源码角度分析Dapr Http Wasm的实现原理，以及给出一些常见的使用场景案例。

### Dapr runtime middleware

dapr将会在初始化的时候调用一次`GetHandler`这个函数来获取http middleware的构造函数。这里是http middleware构造链式中间件调用的逻辑，与wasm无关，但是是这部分代码应用到middleware一个粘合和基础。

```go
httpMiddlewareLoader.DefaultRegistry.RegisterComponent(func(log logger.Logger) httpMiddlewareLoader.FactoryMethod {
		return func(metadata middleware.Metadata) (httpMiddleware.Middleware, error) {
			return wasm.NewMiddleware(log).GetHandler(context.TODO(), metadata)
		}
	}, "wasm")
```

GetHandler会返回一个http中间件函数的构造函数。这个函数的主要作用是解析wasm url，新建一个NewMiddleware。

```go
// GetHandler返回的是http.Handler的一个构造函数。
func (m *middleware) GetHandler(ctx context.Context, metadata dapr.Metadata) (func(next http.Handler) http.Handler, error) {
	rh, err := m.getHandler(ctx, metadata)
	if err != nil {
		return nil, err
	}
	return rh.requestHandler, nil
}

// getHandler 解析metadata配置
func (m *middleware) getHandler(ctx context.Context, metadata dapr.Metadata) (*requestHandler, error) {
  // Metadata解析目前主要是为了获取wasm bytes。
	meta, err := wasm.GetInitMetadata(ctx, metadata.Base)
	if err != nil {
		return nil, fmt.Errorf("wasm: failed to parse metadata: %w", err)
	}

  // 主要是为了获取wasm http Middleware实例，以及配置wasm的运行时
	var stdout, stderr bytes.Buffer
	mw, err := wasmnethttp.NewMiddleware(ctx, meta.Guest,
		handler.Logger(m),
		handler.ModuleConfig(wazero.NewModuleConfig().
			WithName(meta.GuestName).
			WithStdout(&stdout). // reset per request
			WithStderr(&stderr). // reset per request
			// The below violate sand-boxing, but allow code to behave as expected.
			WithRandSource(rand.Reader).
			WithSysNanosleep().
			WithSysWalltime().
			WithSysNanosleep()))
	if err != nil {
		return nil, err
	}

	return &requestHandler{mw: mw, logger: m.logger, stdout: &stdout, stderr: &stderr}, nil
}
```

GetHandler的返回值是requestHandler函数。可以看到通过传递给`requestHandler` `http.Handler`来构造链式的http路由中间件。也是在这个函数中实际包含wasm中间件的逻辑。通过调用`rh.mw.NewHandler`方法新建一个实例，调用其`ServeHTTP(w, r)`对请求进行处理。

```go
func (rh *requestHandler) requestHandler(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		h := rh.mw.NewHandler(r.Context(), next)
		defer func() {
			rh.stdout.Reset()
			rh.stderr.Reset()
		}()

		h.ServeHTTP(w, r)

		if stdout := rh.stdout.String(); len(stdout) > 0 {
			rh.logger.Debugf("wasm stdout: %s", stdout)
		}
		if stderr := rh.stderr.String(); len(stderr) > 0 {
			rh.logger.Debugf("wasm stderr: %s", stderr)
		}
	})
}
```

### http-wasm-host-go

实际的wasm host逻辑都在这个库中。在习惯上，会将运行wasm程序的运行时部分称为host，将wasm程序的sdk称为guest。总体思路上来说这个项目为了留出适配其他http框架（例如fasthttp），将逻辑进行了分层处理，与nethttp相关的逻辑比如说生成中间件，设置http上下文都在这个包下。而通用的逻辑处理，比如说实例化wasm程序，调用wasm程序函数，注册host function等都放在了`handler`包下。了解了这个在查看代码的时候就会对其代码的组织会有更好的了解。在关注具体逻辑之前，先来看一下其目录结构，以及包含的代码逻辑内容。

#### 目录结构

```
├── api # interface和常量定义
│   └── handler # 主要就是middleware 和host interface
├── examples #例子
├── handler # middleware的通用逻辑及实际执行
│   ├── cstring.go
│   ├── middleware.go # 通用的middleware逻辑定义，host function定义
│   ├── nethttp # 针对不同的实际场景（net.http）适配，针对不同http框架的wrapper
│   │   ├── benchmark_test.go
│   │   ├── buffer.go
│   │   ├── buffer_test.go
│   │   ├── example_test.go
│   │   ├── host.go # host function修改http信息的实际逻辑
│   │   ├── host_test.go
│   │   ├── middleware.go # 针对net http特定的逻辑，是包装在handler/middleware.go外面的适配层。
│   │   ├── middleware_test.go
│   │   └── tck_test.go
│   ├── options.go
│   └── state.go
├── internal # 目前只包含测试
│   └── test
│       └── testdata
│           ├── bench
│           └── e2e
├── tck # technology compatibility kit
│   └── guest
└── testing # 测试
    └── handlertest
```

nethttp wrapper主文件中逻辑其实很简单，外部会使用的逻辑主要集中在两个函数中,在新建handler的时候会将底层middleware的HandleRequest和HandleResponse赋值给自己。

```go
func NewMiddleware(ctx context.Context, guest []byte, options ...handler.Option) (Middleware, error) {
  // 初始化host function，验证wasmapp正确，然后返回一个base的middleware。
	m, err := handler.NewMiddleware(ctx, guest, host{}, options...)
	if err != nil {
		return nil, err
	}
	return &middleware{m: m}, nil
}
// NewHandler implements the same method as documented on handler.Middleware.
// 这里的NewHandler函数就是上文中调用的库函数。这里在新建的时候传入了后续中间件next，形成责任链的链式调用。
// 另外可以看到在这个地方将request和response的实际处理函数赋值成了底层的通用函数。
func (w *middleware) NewHandler(_ context.Context, next http.Handler) http.Handler {
	return &guest{
		handleRequest:  w.m.HandleRequest,
		handleResponse: w.m.HandleResponse,
		next:           next,
		features:       w.m.Features(),
	}
}
```

在处理网络请求的时候首先将http.ResponseWriter、*http.Request、next     http.Handler、enabled Features组装成requestState放到context中。然后分别调用HandleRequest、next（handler）、HandleResponse。（requestState是非常重要的一个结构体）。这里的

```go
// ServeHTTP implements http.Handler
func (g *guest) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// The guest Wasm actually handles the request. As it may call host
	// functions, we add context parameters of the current request.
	s := newRequestState(w, r, g)
	ctx := context.WithValue(r.Context(), requestStateKey{}, s)
	outCtx, ctxNext, requestErr := g.handleRequest(ctx)
	if requestErr != nil {
		handleErr(w, requestErr)
	}

	// If buffering was enabled, ensure it flushes.
	if bw, ok := s.w.(*bufferingResponseWriter); ok {
		defer bw.release()
	}

	// Returning zero means the guest wants to break the handler chain, and
	// handle the response directly.
  // 这里注意是否继续执行只与next有关，与在handleRequest是否设置了返回无关。也就是说即使设置了返回，那么仍然可以设置继续（虽然这种写法是错误的），响应body就是两者之和，至于header等信息就看具体实现是先设置生效还是后设置生效。
	if uint32(ctxNext) == 0 {
		return
	}

	// Otherwise, the host calls the next handler.
	err := s.handleNext()

	// Finally, call the guest with the response or error
	if err = g.handleResponse(outCtx, uint32(ctxNext>>32), err); err != nil {
		panic(err)
	}
}
```

##### middleware通用逻辑

在`handler/middleware.go`文件中定义了wasm的通用逻辑，获取实例，以及调用wasm中定义的request和response的hander。NewMiddleware包含了host function的注册和wasm的验证。

```go
func NewMiddleware(ctx context.Context, guest []byte, host handler.Host, opts ...Option) (Middleware, error) {
  // 默认wasm运行时配置，及应用外部传入的配置。
	o := &options{
  newRuntime:   DefaultRuntime,
		moduleConfig: wazero.NewModuleConfig(),
		logger:       api.NoopLogger{},
	}
	for _, opt := range opts {
		opt(o)
	}
	// runtime实例，默认的是wazero运行时
	wr, err := o.newRuntime(ctx)
	if err != nil {
		return nil, fmt.Errorf("wasm: error creating middleware: %w", err)
	}

	m := &middleware{
		host:         host,
		runtime:      wr,
		moduleConfig: o.moduleConfig,
		guestConfig:  o.guestConfig,
		logger:       o.logger,
	}
	// decode wasm二进制，解析为内存内的数据模式，验证代码需要导出的函数存在且签名正确
  if m.guestModule, err = m.compileGuest(ctx, guest); err != nil {
		_ = wr.Close(ctx)
		return nil, err
	}
  // Detect and handle any host imports or lack thereof.
  // 检查用户wasm中的所有的导入定义，看看是否用到了WasiP1或者HttpHandler（也就是本项目模块abi）module。用到了就需要实例化对应的模块并链接。
	imports := detectImports(m.guestModule.ImportedFunctions())
	switch {
	case imports&importWasiP1 != 0:
    // 导入wasi中定义的host function
		if _, err = wasi_snapshot_preview1.Instantiate(ctx, m.runtime); err != nil {
			_ = wr.Close(ctx)
			return nil, fmt.Errorf("wasm: error instantiating wasi: %w", err)
		}
		fallthrough // proceed to configure any http_handler imports
	case imports&importHttpHandler != 0:
    // 注册所有的host function
		if _, err = m.instantiateHost(ctx); err != nil {
			_ = wr.Close(ctx)
			return nil, fmt.Errorf("wasm: error instantiating host: %w", err)
		}
	}
	// 这里面实例化了一个wasm app，主要执行下初始函数验证下没有错误以尽早发现错误fail-fast。
	if g, err := m.newGuest(ctx); err != nil {
		_ = wr.Close(ctx)
		return nil, err
	} else {
		m.pool.Put(g)
	}

	return m, nil
}
```

compileHost就是注册所有的host function。

```go
func (m *middleware) compileHost(ctx context.Context) (wazero.CompiledModule, error) {
	if compiled, err := m.runtime.NewHostModuleBuilder(handler.HostModule).
		NewFunctionBuilder().
		WithGoFunction(wazeroapi.GoFunc(m.enableFeatures), []wazeroapi.ValueType{i32}, []wazeroapi.ValueType{i32}).
		WithParameterNames("features").Export(handler.FuncEnableFeatures).
		NewFunctionBuilder().
		WithGoModuleFunction(wazeroapi.GoModuleFunc(m.getConfig), []wazeroapi.ValueType{i32, i32}, []wazeroapi.ValueType{i32}).
		WithParameterNames("buf", "buf_limit").Export(handler.FuncGetConfig).
		NewFunctionBuilder().
		WithGoFunction(wazeroapi.GoFunc(m.logEnabled), []wazeroapi.ValueType{i32}, []wazeroapi.ValueType{i32}).
		WithParameterNames("level").Export(handler.FuncLogEnabled).
  // 这里就是注册一个set_method这个host函数，这样wasm程序就可以通过调用这个函数来修改http method。
		NewFunctionBuilder().
		WithGoModuleFunction(wazeroapi.GoModuleFunc(m.setMethod), []wazeroapi.ValueType{i32, i32}, []wazeroapi.ValueType{}).
		WithParameterNames("method", "method_len").Export(handler.FuncSetMethod/*"set_method"*/).
		//......more function
		Compile(ctx); err != nil {
		return nil, fmt.Errorf("wasm: error compiling host: %w", err)
	} else {
		return compiled, nil
	}
}
```

以setMethod为例看下一个host function的实现，先吧数据读出来，然后调用host的SetMethod，这里之所以转到host是因为不同的网络框架请求和返回结构体不一样，所以这里实际的设置转到了nethttp下的host。实际的设置是从ctx中把request拿出来然后设置。这里之所以ctx中能拿到也是跟wazero这个运行时有关系，它在所有函数调用不管是host还是wasm都维持了context传递，所以它的状态不用通过全局变量保存，直接通过context传递就可以了。

```go
// getHeader implements the WebAssembly host function handler.FuncSetMethod.
func (m *middleware) setMethod(ctx context.Context, mod wazeroapi.Module, params []uint64) {
	// 数据指针和数据长度
  method := uint32(params[0])
	methodLen := uint32(params[1])

  // 对request的设置都必须在next handler执行之前（不然也没有意义）
	_ = mustBeforeNext(ctx, "set", "method")

	var p string
	if methodLen == 0 {
		panic("HTTP method cannot be empty")
	}
  // 从wasm的memory中读取数据，这里传入的"method"是打日志用的，真正有意义的是其他三个值。
	p = mustReadString(mod.Memory(), "method", method, methodLen)
	m.host.SetMethod(ctx, p)
}
// SetMethod implements the same method as documented on handler.Host.
// 实际设置Method函数，从ctx中取出request结构体，进行赋值。
func (host) SetMethod(ctx context.Context, method string) {
	r := requestStateFromContext(ctx).r
	r.Method = method
}
// requestStateFromContext就是从ctx中获取到*requestState
func requestStateFromContext(ctx context.Context) *requestState {
	return ctx.Value(requestStateKey{}).(*requestState)
}
```

### 边界检查

对于很多情况都有检查，比如说不能在HandleResponseFn中修改request信息，没有开启*FeatureBufferResponse*就不能修改来自最终网络调用的返回等。

但是有一些奇怪的行为是可以运行的（但是当你在写这样的代码的时候，你会意识到这是逻辑错误），比如在HandleRequestFn设置response信息，同时next为true去执行后续中间件，最终client得到的body就是两者叠加（是否开启*FeatureBufferResponse*无影响）。因为这个writer先被HandleRequestFn写了信息，又被来自网络调用写了信息，读的时候自然就都读出来了。例如使用如下的代码

```go
func main() {
	handler.HandleRequestFn = handleRequest
}

// handleRequest serves a static response from the Dapr sidecar.
func handleRequest(req api.Request, resp api.Response) (next bool, reqCtx uint32) {
	resp.Headers().Set("Content-Type", "text/plain")
	resp.Body().WriteString("hello " + req.GetURI())
	return true, 0
}
```

### 使用案例分析

在[middleware examples](https://github.com/Taction/dapr-wasm-example/tree/main/middleware/go)中给出了一些使用案例。这里简单罗列下比较常见的几个使用场景：

#### 拦截请求

```go
package main

import (
	"github.com/http-wasm/http-wasm-guest-tinygo/handler"
	"github.com/http-wasm/http-wasm-guest-tinygo/handler/api"
)

func main() {
	handler.HandleRequestFn = handleRequest
}

// handleRequest serves a static response from the Dapr sidecar.
func handleRequest(req api.Request, resp api.Response) (next bool, reqCtx uint32) {
	resp.Headers().Set("Content-Type", "text/plain")
	resp.Body().WriteString("Hello, this is wasm middleware, you are requesting: " + req.GetURI())
	return false, 0 // do not execute next middleware
}
```

在`handleRequest`函数中可以直接设置网络请求返回，这对通过请求中的某些数据判断拦截请求的场景非常有用。可以注意到`handleRequest`有两个返回值，分别是代表是否执行后续中间件/请求逻辑布尔类型的next，以及用于上下文保持的uint32类型reqCtx。当拦截请求时next应该返回false。

#### 修改请求返回数据

用户也可以通过`handleResponse`函数对请求返回数据进行修改。包括header、body、trailer信息。下面是一个修改返回的例子：

```go
package main

import (
	"bytes"
	"github.com/http-wasm/http-wasm-guest-tinygo/handler"
	"github.com/http-wasm/http-wasm-guest-tinygo/handler/api"
)

func main() {
	handler.Host.EnableFeatures(api.FeatureBufferResponse) // change response must enable buffer response feature
	handler.HandleResponseFn = handleResponse
}

// handleResponse can modify the response
func handleResponse(reqCtx uint32, req api.Request, resp api.Response, isError bool) {
	// get the original response body
	buf := &bytes.Buffer{}
	resp.Body().WriteTo(buf)
	resp.Headers().Set("Content-Type", "text/plain")
	// modify the response body, adding a prefix "hello"
	resp.Body().WriteString("hello ")
	resp.Body().Write(buf.Bytes())
}
```

这里值得注意的是，如果想要修改返回信息，必须开启`BufferRespons`功能，如上述代码所示就是在main函数中设置功能开启。在`handleResponse`函数中可以获取请求和返回的信息，以及设置返回信息。

#### 上下文保持

在某些场景下，用户可能需要对网络请求数据进行处理，并且在`handleResponse`阶段仍然能够获取到这些数据。这就可以通过上下文保持的功能来实现：

```go
package main

import (
	"github.com/http-wasm/http-wasm-guest-tinygo/handler"
	"github.com/http-wasm/http-wasm-guest-tinygo/handler/api"
)

func main() {
	handler.Host.EnableFeatures(api.FeatureBufferResponse)
	handler.HandleRequestFn = handleRequest
	handler.HandleResponseFn = handleResponse
}

var globalContext = make(map[uint32]string)
var contextCounter uint32

func handleResponse(reqCtx uint32, req api.Request, resp api.Response, isError bool) {
	handler.Host.Log(api.LogLevelInfo, "handleResponse get uri from context "+globalContext[reqCtx])

	// Serve response
	resp.Headers().Set("Content-Type", "text/plain")
	resp.SetStatusCode(200)
	resp.Body().WriteString("Hello ")
	resp.Body().WriteString(globalContext[reqCtx])
	return
}

// handleRequest serves a static response from the Dapr sidecar.
func handleRequest(req api.Request, resp api.Response) (next bool, reqCtx uint32) {
	contextCounter++

	globalContext[contextCounter] = req.GetURI()
	handler.Host.Log(api.LogLevelInfo, "handleRequest uri: "+req.GetURI())
	return true, contextCounter // continue to execute the next middleware
}
```

在这个例子中，我们简单的做了一个示例，在`handleRequest`时将信息设置为与一个uint32类型的值相关联（reqCtx），当`handleResponse`被执行时这个值会被wasm运行时传入。通过`reqCtx`可以通过全局map获取到相关的信息。
