---
title: "Spin Introduction"
linkTitle: "Spin介绍"
type: docs
date: 2023-02-11T16:22:10+08:00
draft: false
---

### 介绍

hippo是一个WebAssembly PaaS平台，用于创建基于WebAssembly的微服务和Web应用程序。它提供了一个基于浏览器的门户、一个用于客户端CLI的API以及后端管理特性，以便与Bindle服务器、负载平衡器和Spin一起工作。Hippo平台的管理员使用nomad配置和安装Hippo的组件。服务发现、应用程序调度和联网都委托给了nomad，而Hippo则提供了干净简单的开发者体验。

Bindle 可以认为是WebAssembly的版本化包管理工具，用于存储和获取WebAssembly程序，实际上它可以存储任意类型的数据，如WebAssembly模块、Templates、Web files such as HTML, CSS, JavaScript, and images、Machine learning models、以及任何你的应用程序依赖的文件。

路由器组件是Hippo平台的一部分，负责将HTTP/s流量路由到应用程序，并代理平台API流量。由Yet-Another-Reverse-Proxy（YARP）项目提供。

Spin是一个使用WebAssembly组件构建和运行事件驱动的微服务应用程序的框架。即用于实际执行WebAssembly程序。

通过[这个案例](https://www.fermyon.com/blog/scale-to-zero-problem?&utm_medium=blog&utm_campaign=related)也能更多看到fermyon的具体架构。简单的画了一下其架构

![image-20230206181355757](https://image-1255620078.cos.ap-nanjing.myqcloud.com/image-20230206181355757.png)

### 准备

> 在Ubuntu环境下安装可以遵循以下命令教程，否则请参考[原教程](https://github.com/fermyon/installer/blob/main/local/README.md)

#### Nomad

```bash
sudo apt-get install software-properties-common

curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install nomad
```

#### Consul

[install consul](https://developer.hashicorp.com/consul/downloads)

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install consul
```

#### Spin

[spin quick starts](https://developer.fermyon.com/spin/quickstart/)

```bash
curl -fsSL https://developer.fermyon.com/downloads/install.sh | bash
sudo mv ./spin /usr/local/bin/spin
```

#### tinygo

仅在需要编译tinygo wasm程序时需要安装,[教程](https://tinygo.org/getting-started/install/linux/#arch-linux)

```
wget https://github.com/tinygo-org/tinygo/releases/download/v0.26.0/tinygo_0.26.0_amd64.deb
sudo dpkg -i tinygo_0.26.0_amd64.deb
export PATH=$PATH:/usr/local/bin
tinygo version
```

### 启动

> 以本地模式启动https://github.com/fermyon/installer/blob/main/local/README.md

通过执行`./start.sh`脚本来部署本地环境。这个脚本做的事情比较清晰。

1. 判断机器系统是否支持
2. 判断 consul、nomad、spin命令是否存在
3. 删除data目录，创建log目录用来存放启动过程中的日志
4. 启动consul
5. 启动nomad
6. 通过nomad启动traefik，启动的nomad定义在job下
7. 通过nomad启动bindle、hippo，定义都在job目录下
8. 等待hippo可访问，输出以上各项的访问地址

命令行界面会输出一些export命令，另开一个命令行终端并执行这些命令，打开http://hippo.local.fermyon.link/页面注册一个账户，并在Teminal中执行以下命令

```bash
export HIPPO_USERNAME=<username>
export HIPPO_PASSWORD=<password>
```

### 运行一个wasm程序

##### 安装模板

```bash
spin templates install --git https://github.com/fermyon/spin
```

安装完成后可以通过`spin templates list`命令来查看已安装的模板。

##### 新建一个wasm程序

通过`spin new`命令可以根据模板创建一个wasm项目，由于我个人对go比较熟悉，我们先创建一个go的项目使用http trigger。


<!-- @selectiveCpy -->

```bash
$ spin new
Pick a template to start your project with:
  http-c (HTTP request handler using C and the Zig toolchain)
  http-csharp (HTTP request handler using C# (EXPERIMENTAL))
> http-go (HTTP request handler using (Tiny)Go)
  http-grain (HTTP request handler using Grain)
  http-rust (HTTP request handler using Rust)
  http-swift (HTTP request handler using SwiftWasm)
  http-zig (HTTP request handler using Zig)
  redis-go (Redis message handler using (Tiny)Go)
  redis-rust (Redis message handler using Rust)

Enter a name for your new project: hello_rust
Project description: My first Rust Spin application
HTTP base: /
HTTP path: /...
$ tree
├── .cargo
│   └── config.toml
├── .gitignore
├── Cargo.toml
├── spin.toml
└── src
    └── lib.rs
```

项目主目录下的`spin.toml`文件定义的就是这个spin项目的配置清单文件:

```toml
spin_version = "1"
authors = ["zhangchao <zchao9100@gmail.com>"]
description = "a spin http handler in go for wasm"
name = "spinhttpgo"
trigger = { type = "http", base = "/" }
version = "0.1.0"

[[component]]
id = "spinhttpgo"
source = "main.wasm"
[component.trigger]
route = "/hello"
[component.build]
command = "tinygo build -wasm-abi=generic -target=wasi -gc=leaking -no-debug -o main.wasm main.go"
```

运行`spin build`命令会执行上面文件中的comman命令来编译wasm程序。执行完成后在项目目录下会新增一个`main.wasm`文件。

##### 源码

项目源码非常简单，胶水代码已经包含在spin sdk中，只需要在函数中实现自己的业务逻辑就行，对于这个demo来说就是在网络请求时返回纯文本`Hello Fermyon!`.

```go
package main

import (
	"fmt"
	"net/http"

	spinhttp "github.com/fermyon/spin/sdk/go/http"
)

func init() {
	spinhttp.Handle(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "text/plain")
		fmt.Fprintln(w, "Hello Fermyon!")
	})
}

func main() {}
```

##### 运行wasm程序

通过`spin up`命令可以运行wasm程序

```bash
$ spin up
Serving HTTP on address http://127.0.0.1:3000
Available Routes:
  spin-hello-world: http://127.0.0.1:3000/hello
```

访问`http://127.0.0.1:3000/hello`即可看到wasm程序运行的结果：

```bash
$ curl -i localhost:3000/hello
HTTP/1.1 200 OK
content-type: text/plain
content-length: 15
date: Mon, 12 Feb 2023 07:49:01 GMT

Hello Fermyon!
```

### 部署到云端

通过`spin deploy`命令可以将程序部署到Fermyon云端运行。

### Spin 与 wasmcloud异同点分析

> 个人对这两个项目的了解，其中难免有疏漏之处，若有不妥之处，望不吝赐教。

##### interface定义

spin 使用wasm component model来定义API，这样的好处是可以复用工具链来生成模板代码，对多语言的支持上来说所需要做的努力更少。

wasmcloud是通过代码生成，自己从0构建了一套开发者使用工具，并且生成的代码更容易使用，有些框架层的模板代码给你一起生成了（主要针对component，wasm的开发来说两者基本差不多）。

##### 可扩展性或者说用户自定义interface

spin的wasm程序与“环境”进行交互可以通过http trigger获取http的能力，通过wasi获取wasi中定义的系统能力。触发wasm程序执行除了内置的http和redis外，用户可以利用spin提供的库自己开发程序。

wasmcloud有capability模型，用户可以通过自定义interface开发Provider（提供能力）和actor（wasm程序），wasmcloud的工具会生成sdk（目前支持rust和tinygo）来简化开发避免模板代码，使用户专注于逻辑代码。

##### wasm程序与能力提供程序交互

spin来说是通过函数调用的方式来调用runtime提供的方法。

wasmcloud做了一层异步的rpc解耦封装。wasm程序调用的是将消息序列化后调用runtime一个固定的方法，runtime将信息发布到nats（消息通信组件，团队正在尝试支持其他的），Provider订阅发给自己的消息，执行对应的函数，如果是wasm调用Provider就还需要将结果发回nats中。这样就在中间引入了消息队列和更多的网络和序列化开销。

##### 学习成本

spin会低一点，用户唯一做的就是基于sdk来开发自己的应用，更适合只需要获取wasi提供的能力和网络能力的。如果要

wasmcloud因为系统中涉及的模块多，所以会高一些，至少你需要搞懂什么是Provider、actor、link。但是有UI界面和比较好的文档支持，所以个人觉得也还行。

### 参考

https://docs.hippofactory.dev/topics/architecture/

https://github.com/WebAssembly/component-model

https://www.fermyon.com/blog/webassembly-component-model

https://radu-matei.com/blog/intro-wasm-components/

https://www.fermyon.com/blog/scale-to-zero-problem?&utm_medium=blog&utm_campaign=related

 https://www.fermyon.com/blog/spin-nomad

https://www.fermyon.com/blog/scale-to-zero-problem?&utm_medium=blog&utm_campaign=related
