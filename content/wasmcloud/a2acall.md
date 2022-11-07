---
title: "Actor to Actor call"
linkTitle: "Actor to Actor call"
type: docs
date: 2022-11-06T15:55:34+08:00
draft: false
---

### 概述

在[wasmcloud actor 调用 actor](https://wasmcloud.dev/app-dev/a2a/)的文档中，有介绍如何通过actor调用actor，但是在查看[wasmcloudexamples](https://github.com/wasmCloud/examples)和[interfaces](https://github.com/wasmCloud/interfaces)示例项目的时候均没有看到示例。因此本文档从零开始创建一个actor调用actor的示例。

本示例涉及到一个interface项目，定义actor与actor之间的调用协议，只包含一个函数Upper将输入转化为大写；两个actor，发起请求方为actor client，接受请求并将请求内容转化为大写的为actor server。

这两个actor分别拥有go语言和rust语言实现，并且go和rust语言实现的actor具有相同的call alias。这意味着在测试本示例的时候可以忽略实现语言，可以部署一个任意语言的actor server和一个任意语言的actor client。另外根据wasmcloud的说明，当使用call alias的时候不允许重名，所以你应仅部署一种语言的actor server。

actor client通过使用wasmcloud的httpprovider来接收外部请求，并将http body发送到actor server。

![image-20221107155801950](https://image-1255620078.cos.ap-nanjing.myqcloud.com/image-20221107155801950.png)

### 创建项目

#### 根据模板创建项目

首先我们创建3个项目，分别是interface、actor client、actor server。其中：

interface我们命名为uppercase，定义interface的功能，将传入的字符串变成大写，参考[创建interface流程](https://wasmcloud.dev/app-dev/create-provider/new-interface/)。

actor client是调用的发起方，actor server是被调用的actor。

```shell
wash new interface uppercase

wash new actor actor-client --template-name hello

wash new actor actor-server --template-name hello
```

同时我们创建go语言对应的两个actor

```shell
wash new actor actor-client-go --template-name echo-tinygo

wash new actor actor-server-go --template-name echo-tinygo
```

这个时候你当前的目录下应该有以下几个文件夹

```
$ ls
actor-client    actor-client-go actor-server    actor-server-go uppercase
```

#### smithy定义修改

首先让我们打开uppercase目录下的`uppercase.smithy`文件，修改为以下内容，定义交互ABI。这里需要注意`@wasmbus( actorReceive: true )` 通过定义`actorReceive`表明这是一个actor to actor的interface。

```smithy
// uppercase.smithy
//

// Tell the code generator how to reference symbols defined in this namespace
metadata package = [ { namespace: "com.oneitfarm.wasm.uppercase", crate: "uppercase" } ]

namespace com.oneitfarm.wasm.uppercase

use org.wasmcloud.model#wasmbus

/// Description of Uppercase service
@wasmbus( actorReceive: true )

service Uppercase {
  version: "0.1",
  operations: [ Upper ]
}

/// Uppercase - Execute transaction
operation Upper {
    input: UppercaseRequest,
    output: UppercaseResponse,
}

/// Parameters sent for Uppercase
structure UppercaseRequest {
    /// data to be Uppercased
    @required
    data: String,
}

/// Response to Uppercase
structure UppercaseResponse {
    /// Indicates a successful request
    @required
    success: Boolean,

    /// response data
    data: String,
}

```

通过模板生成的项目，在默认情况下，仅会生成rust的sdk，如果希望同时创建tinygo的sdk，需要修改codegen.toml文件，添加以下内容

```toml
[tinygo]
formatter = [ "goimports", "-w" ]
output_dir = "tinygo"
files = [
    { path = "uppercase.go", package="uppercase", namespace = "com.oneitfarm.wasm.uppercase" },
]
```

最后让我们来生成代码！

```shell
cd uppercase
wash gen
```

#### Rust actor 代码

修改actor-server和actor-client的Cargo.toml,在dependencies下增加uppercase = { path="../uppercase/rust" }

##### actor-client

actor client作为actor call actor的client，向actor server发起uppercase的请求。

```rust
        let provider = UppercaseSender::to_actor("MBVITL2QJQ42KDTVVQJ2WZM6K3BMZ2CVJXLFYYQKYCLEPS5R5WOUKXIU");
        let res = provider.upper(_ctx, &UppercaseRequest{ data: text }).await?;
```



##### actor-server

首先#[services]中需要加入trait名字Uppercase，然后Actor需要实现Uppercase trait，也就是upper方法。

```rust
use uppercase::*;

#[derive(Debug, Default, Actor, HealthResponder)]
#[services(Actor, Uppercase)]
struct ActorServerActor {}

/// Implementation of HttpServer trait methods
#[async_trait]
impl Uppercase for ActorServerActor {
    /// Returns a uppercase string in the response body.
    async fn upper(&self, ctx: &Context, arg: &UppercaseRequest) -> RpcResult<UppercaseResponse> {
        return RpcResult::Ok(UppercaseResponse{ data: Some(arg.data.to_uppercase()), success: true })

    }
}
```

#### Go actor 代码

在运行`wash gen`命令生成Go语言SDK之后，我们需要对它进行初始化。go mod init `package`中的package可以换成任意你想用的，只要在引入的时候保持就可以。

```
cd uppercase/tinygo
go mod init github.com/oneitfarm/uppercase
go mod tidy
```

#### Go代码修改

对server和client项目增加SDK依赖，由于我没有真正的将uppercase上传为一个github项目，所以在引入时我采用了替换为本地相对路径的方式，在go mod中增加以下两行。你可以根据你的需要在符合go mod的约束条件下任意修改。

```
require github.com/oneitfarm/uppercase v0.0.0

replace github.com/oneitfarm/uppercase => ../uppercase/tinygo
```

##### go-actor-client 代码修改

注意每个人编译的actor server的id可能是不一样的，你在构建actor sender实例的时候需要填写自己的id，当然你也可以使用call alias：`oneitfarm/actor_server`，在实际代码示例中我们将使用call alias。

```go
func (e *ActorClientGo) HandleRequest(ctx *actor.Context, req httpserver.HttpRequest) (*httpserver.HttpResponse, error) {
	provider := uppercase.NewActorUppercaseSender("MA232W2RTI4GC7A645VJKK3GI7X54XOJR2KNDKEGAMISVN7OQOG55VOC")
	res, err := provider.Upper(ctx, uppercase.UppercaseRequest{
		Data: string(req.Body),
	})
	if err != nil {
		return &httpserver.HttpResponse{
			StatusCode: 500,
			Header:     make(httpserver.HeaderMap, 0),
			Body:       []byte(err.Error()),
		}, nil
	}
	r := httpserver.HttpResponse{
		StatusCode: 200,
		Header:     make(httpserver.HeaderMap, 0),
		Body:       []byte(res.Data),
	}
	return &r, nil
}
```

##### go-actor-server 代码

```go
func main() {
	me := ActorServerGo{}
	actor.RegisterHandlers(uppercase.UppercaseHandler(&me))
}

type ActorServerGo struct{}

func (e *ActorServerGo) Upper(ctx *actor.Context, req uppercase.UppercaseRequest) (*uppercase.UppercaseResponse, error) {
	r := uppercase.UppercaseResponse{
		Data: strings.ToUpper(req.Data),
	}
	return &r, nil
}
```

#### Call alias（调用别名）介绍

通过上面的例子可以看出来通过id来调用actor非常不方便，所以可以通过call 别名的方式。通过wasmcloud文档我们可以得知，设置别名需要在claim sign的时候增加`call-alias`flag。由于call alias是其他actor调用自己的时候的别名，因此只需要针对actor server设置就可以了。

对于go代码来说，在actor-server-go项目的Makefile中添加一行`ACTOR_ALIAS = oneitfarm/actor_server`.然后actor-client-go项目中的调用就可以改为 `provider := uppercase.NewActorUppercaseSender("oneitfarm/actor_server_go")`了。

对于rust代码来说是同样的，在actor-server项目的Makefile中增加一行：`ACTOR_ALIAS = oneitfarm/actor_server`即可。

##### 别名规则

可以包含小写字母、数字、`/`和`_`。

### 注意！

operation和service不能重名！！！否则生成的代码会有异常。比如如果将interface中的smithy文件中的operation定义为Uppercase就会出问题。初步追踪了一下，应该是在解析smithy格式文件的时候，有一个步骤使用了以名字为key的扁平结构，造成内容混乱的问题。

```
/// ...

service Uppercase {
  version: "0.1",
  operations: [ Uppercase ]
}

/// Uppercase - Execute transaction
operation Uppercase {
    input: UppercaseRequest,
    output: UppercaseResponse,
}

/// ...
```

