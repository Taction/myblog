---
title: "Wasmcloud Actor与Dapr微服务结合构建更灵活的微服务架构"
linkTitle: "Dapr Provider"
type: docs
date: 2022-11-22T19:45:05+08:00
draft: false
---

### Wasmcloud和Dapr基础介绍

[Dapr](https://docs.dapr.io/concepts/overview/)是一个可移植的、事件驱动的运行时，它使任何开发人员都能轻松地构建在云和边缘运行的有弹性、无状态和有状态的应用程序，它同时也支持多种编程语言和开发框架。Dapr利用边车架构的优势，帮助解决构建微服务所带来的挑战，并使代码与平台无关。

[Wasmcloud](https://wasmcloud.dev/overview/)是一个旨在帮助开发人员快速轻松地编写默认安全业务逻辑的平台，具有快速反馈循环，不受样板文件、integrated（e.g. tangled）dependencies以及与非功能需求的紧密耦合的负担。它是一套工具和库，可用于构建由称为*actor*的可移植业务逻辑单元组成的分布式应用程序。其actor就是WebAssembly应用程序。

以个人浅显的理解简单来说Wasmcloud是运行WebAssembly程序的一个编排调度平台，有一点类似于WebAssembly的k8s。

### wasmcloud-dapr-http-provider介绍

当我们想把部分应用程序迁移到WebAssembly放进Wasmcloud进行托管，以享受带来的安全、灵活、冷启动快速等优势的时候。我们并不想花费比较大的代价把已有的一些微服务进行迁移，特别是有些服务不能直接被编译为WebAssembly或者更适合运行在k8s或者裸机上。

那么有没有一种方式，将一些新的业务逻辑以WebAssembly的形式在Wasmcloud中运行，并且跟现有的dapr托管的微服务之间能够无感的相互调用呢？

答案是可以的！wasmcloud-dapr-http-provider就是为了这个目的诞生的！从名字上你可以看出这是一个Wasmcloud provider，在Wasmcloud系统中为actor提供各种各样的能力（capability），它可以发现dapr的对应服务并且以dapr兼容的协议向对应的dapr发送请求。同时这也是与dapr进行相互沟通的桥梁，将与之link的actor在dapr nameresolution(consul)中进行注册，因而dapr可以发现这个actor并进行调用，将请求发送到这个provider，provider会将请求发送给actor。

### 如何使用

#### Dapr服务调用介绍

[Dapr服务调用示例](https://docs.dapr.io/getting-started/quickstarts/serviceinvocation-quickstart/)讲述了使用dapr进行微服务托管的情况下，如何进行服务调用。在这个案例中checkout微服务调用order-processor微服务。[以checkout app go代码](https://github.com/dapr/quickstarts/blob/master/service_invocation/go/http/checkout/app.go#L31)为例，只需要在header中指定调用的app就可以:`req.Header.Add("dapr-app-id", "order-processor")`。其调用过程如下所示：

![image-20221122152232360](https://image-1255620078.cos.ap-nanjing.myqcloud.com/image-20221122152232360.png)

#### 运行测试案例

将Dapr的服务调用示例稍微改动一下。实际上，我们只需要改动[一行代码](https://github.com/dapr/quickstarts/blob/master/service_invocation/go/http/checkout/app.go#L31)将`order-processor`改为`wasm-processor`，使得checkout app调用Wasmcloud中运行的actor，然后我们在actor中调用`order-processor`这个dapr托管的微服务，最终的调用链路如下所示：

![image-20221122152428208](https://image-1255620078.cos.ap-nanjing.myqcloud.com/image-20221122152428208.png)

##### 运行order-processor app

修改代码后，根据[Dapr服务调用示例](https://docs.dapr.io/getting-started/quickstarts/serviceinvocation-quickstart/)go语言案例启动`order-processor`.

##### 启动Wasmcloud actor

actor的代码很简单，就是在接收到请求后，拼接上Proxy by wasmcloud，然后请求dapr托管的order-processor应用。

```go
func (e *Echo) HandleRequest(ctx *actor.Context, req httpserver.HttpRequest) (*httpserver.HttpResponse, error) {
	provider := httpserver.NewProviderHttpServer()
	req.Header["dapr-app-id"] = []string{"order-processor"} // 请求dapr order-processor app 
	req.Body = append([]byte(`{"Proxy": "by wasmcloud", "origin":`), req.Body...)
	req.Body = append(req.Body, []byte(`}`)...)
	res, err := provider.HandleRequest(ctx, req)
	if err != nil {
		return InternalServerError(err), nil
	}
	return res, nil

}
```

你可以选择从源码编译wasm actor程序，或者运行actor `docker.io/docker4zc/dapr-wasm-actor:0.1.0`

![image-20221122164108329](https://image-1255620078.cos.ap-nanjing.myqcloud.com/image-20221122164108329.png)

运行provider，你可以从源码编译运行，也可以运行`docker.io/docker4zc/dapr-provider-go:0.0.4`（注意这个仅支持linux平台）。在启动provider时，需要添加配置`{"resolver_address":"http://127.0.0.1:8500","external_address":"127.0.0.1"}`。这里的resolver_address是consul的访问地址，如果你的consul不是在本地启动的你需要修改这个地址；external_address是provider的访问地址，将会被dapr使用来访问Wasmcloud中的actor，这里同样假定你是在本地启动的dapr和Wasmcloud。

![image-20221122165710560](https://image-1255620078.cos.ap-nanjing.myqcloud.com/image-20221122165710560.png)

定义Link，需要添加values`address=0.0.0.0:8888,unique_id=order-processor`.这里注意values中的unique_id是此actor在dapr中的app-id，通过此标识进行调用。address是provider监听的端口，每个actor link都需要有自己的一个唯一端口，provider在向consul注册时会取用上面的external_address加这里的端口号作为访问地址注册。

![image-20221122164343723](https://image-1255620078.cos.ap-nanjing.myqcloud.com/image-20221122164343723.png)

##### 运行checkout app

根据[Dapr服务调用示例](https://docs.dapr.io/getting-started/quickstarts/serviceinvocation-quickstart/)go语言案例启动`checkout`.

你应该能从命令行终端看到以下输出：

````
```
Order passed:  {"Proxy": "by wasmcloud", "origin":{"orderId":1}}
Order passed:  {"Proxy": "by wasmcloud", "origin":{"orderId":2}}
Order passed:  {"Proxy": "by wasmcloud", "origin":{"orderId":3}}
Order passed:  {"Proxy": "by wasmcloud", "origin":{"orderId":4}}
Order passed:  {"Proxy": "by wasmcloud", "origin":{"orderId":5}}
Order passed:  {"Proxy": "by wasmcloud", "origin":{"orderId":6}}
Order passed:  {"Proxy": "by wasmcloud", "origin":{"orderId":7}}
Order passed:  {"Proxy": "by wasmcloud", "origin":{"orderId":8}}
Order passed:  {"Proxy": "by wasmcloud", "origin":{"orderId":9}}
Order passed:  {"Proxy": "by wasmcloud", "origin":{"orderId":10}}
```
````

在order-processor的命令行tab，你应该能看到以下输出：

````
```
Order received :  {"Proxy": "by wasmcloud", "origin":{"orderId":1}}
Order received :  {"Proxy": "by wasmcloud", "origin":{"orderId":2}}
Order received :  {"Proxy": "by wasmcloud", "origin":{"orderId":3}}
Order received :  {"Proxy": "by wasmcloud", "origin":{"orderId":4}}
Order received :  {"Proxy": "by wasmcloud", "origin":{"orderId":5}}
Order received :  {"Proxy": "by wasmcloud", "origin":{"orderId":6}}
Order received :  {"Proxy": "by wasmcloud", "origin":{"orderId":7}}
Order received :  {"Proxy": "by wasmcloud", "origin":{"orderId":8}}
Order received :  {"Proxy": "by wasmcloud", "origin":{"orderId":9}}
Order received :  {"Proxy": "by wasmcloud", "origin":{"orderId":10}}
```
````



[wasmcloud-dapr-http-provider github地址](https://github.com/Taction/wasmcloud-dapr-http-provider)

