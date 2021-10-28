---
type: docs
title: "基于WebAsssembly的Serverless探索2：相关项目分析"
date: 2021-10-16T12:59:01+08:00
draft: false
bookCollapseSection: false
weight: 80
linkTitle: "相关项目分析"
---

### 背景

#### 云原生正在成为新的基础设施

云原生技术有利于各组织在公有云、私有云和混合云等新型动态环境中，构建和运行可弹性扩展的应用。云原生向下封装资源，将复杂性下沉到基础设施层，加速了应用与基础设施资源之间的解耦，向上支撑应用，让开发者更关注业务价值，解放了开发者的生产力。业务与企业运维能够依托于云原生所提供的基础设施，实现智能升级价值，充分释放云计算红利，云原生技术正在成为全新的生产力工具。

#### 什么是serverless

无服务器架构应用仅在需要时启动。有事件触发应用代码运行时，公共云提供商才会为这一代码分配资源。该代码执行结束后，用户便不再付费。除了成本与效率上的优势外，进一步释放了云计算的能力，将运维、安全、可用性、可伸缩性等需求下沉到基础设施实现。无服务器也能将开发人员从运维相关的琐碎日常任务中解放出来。

程序员的工作一定是越来越简化的，对于业务开发人员来说，可以专注于业务代码开发，摆脱诸如服务器置备和管理应用程序等复杂的工作。

#### 为什么选择WebAsssembly

[WebAsssembly](https://webassembly.github.io/spec/core/intro/introduction.html)是一种安全、可移植的低层次二进制代码。它有一种紧凑的二进制格式，能够以接近原生性能的速度运行。作为应用层级的后端语言，它具有以下优点：

![image-20211016112123586](/images/image-20211016112123586.png)

WebAssembly对比docker冷启动延迟可以降低90%以上，对比于js通常具有两倍的运行速度优势。其冷启动方面的延迟是它非常显著的一个优势。与faas和serverless的理念非常贴合。从下图WebAssembly与docker的一个对比上，可以看到它在冷启动延迟和速度方面均具有独特的优势。

![image-wasmvsdocker](/images/wasmvsdocker.png)

#### istio（envoy）与WebAssembly结合



#### dapr与WebAsssembly结合

最初了解WebAsssembly还是通过易立的这篇[WebAssembly + Dapr = 下一代云原生运行时?](https://developer.aliyun.com/article/783864)

作为dapr的社区成员，本身对dapr也非常了解，加上公司内部也在使用dapr的某些功能。我对dapr+WebAsssembly模式的前景还是非常看好的。

### proxy-wasm

#### 简介

前面说到，istio使用WebAsssembly动态扩展其数据面功能。只要遵循其标准的WebAsssembly程序都可以在运行时被动态加载运行。那么这个标准是什么标准呢？这样就有了proxy-wasm，

#### 适配





### dapr-wasm

#### 简介

second state 发布了一个[dapr-wasm](https://github.com/second-state/dapr-wasm) 的项目，这个项目是结合dapr和wasmedge提供的一个图片识别的例子。跟项目的简介说的一样，这是一个模板项目，用于演示如何在使用 dapr作为sidecar 运行 WebAssembly 的微服务。

我大致看了一下go的源码，虽然代码量很少，但是为了这个demo的运行，背后还是有一些工作的。首先是wasm正常运行所需要的ABI，包括图片相关的和Tensorflow相关的导入函数。这些方法是在go的[WasmEdge-go](https://github.com/second-state/WasmEdge-go)(实际逻辑代码在[WasmEdge-image](https://github.com/second-state/WasmEdge-image)、[WasmEdge-tensorflow](https://github.com/second-state/WasmEdge-tensorflow)中)项目中封装的，通过差异化编译方式在正常编译是不附带这些方法。

在rust代码侧使用了两个额外的ABI sdk[wasm-bindgen](https://github.com/rustwasm/wasm-bindgen) 和适配TensorFlow的[wasmedge_tensorflow_interface](https://github.com/second-state/wasmedge_tensorflow_interface)

在这个项目中可以看到一个wasm项目的几个基础点。制定交互ABI、host实现、目标语言sdk实现、项目引用sdk编写业务逻辑编译为wasm。在本项目中，应用与dapr进行结合，首先dapr作为应用的运行时负责应用与外部的交互（在此项目中体现较少），host加载wasm整体作为一个应用对外提供服务。

#### 搭建及运行

跟随[项目说明](https://github.com/second-state/dapr-wasm#3-prerequisites)进行预装软件的安装以及构建及运行项目。

这里有几点额外说明一下。首先在安装环境和预装软件的时候，使用国内的服务器会非常慢所以可能需要一定的科学上网的手段，或者直接看下安装脚本将必要的文件/软件本地下载并上传到服务器(由于下载的东西比较多，所以不太推荐根据脚本手动下载并上传)。或者使用国外的服务器下载过程就非常丝滑。

在运行的时候推荐的命令是阻塞式的，如果你想只开一个命令行窗口把所有必要的软件运行起来，你可以将其后台运行（并将控制台输出重定向到文件）。

```
make run-api-go & ## Run the image-api-go
make run-api-rs & ## Run the image-api-rs
make run-web & ## Run the Web port service
```

如果你对某个库有代码修改，想要重新运行，你可以通过以下方式：

```
dapr list
```

```
  APP ID        HTTP PORT  GRPC PORT  APP PORT  COMMAND               AGE  CREATED              PID
  image-api-rs  3502       46317      9004      ./target/release/...  20h  2021-10-12 13:31.33  2327077
  image-api-go  3501       42157      9003      ./image-api-go        20h  2021-10-12 13:32.25  2327404
  go-web-port   3500       46121      8080      ./web-port            20h  2021-10-12 13:32.44  2327618
```

以重新运行前端为例，其app id为go-web-port,将其先stop然后重新编译运行即可

```
dapr stop go-web-port
make build-web
make run-web
```

#### 效果

运行时会启动3个daprsidecar和3个项目的进程分别为web、go和rust项目。前端项目运行在8080端口通过dapr访问go项目后端api，对图片进行识别。

![image-20211012152207326](/images/image-20211012152207326.png)

#### 压测

主要目的是想通过压测查看是否存在内存泄露现象，所以对于压测的次数和时间没有精确要求控制。而且请求会附带图片参数，通过ab和wrk构造压测请求较为麻烦。所以偷懒一下，直接在web-port子项中改造下发送请求的地方，每次点击额外发送100条请求。通过top来查看内存占用情况。由于可能会重启后端服务其pid会变，linux下可以通过运行`top -p $(ps -ef | grep ./image-api-go | grep -v grep | grep -v dapr | awk '{print $2}')`

发送前记录

```
top - 10:25:41 up 24 days, 18:14,  2 users,  load average: 2.46, 1.23, 0.68
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.8 us,  0.8 sy,  0.0 ni, 98.3 id,  0.0 wa,  0.0 hi,  0.1 si,  0.0 st
MiB Mem :   7692.5 total,    178.5 free,   1577.7 used,   5936.3 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   5839.4 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
2327420 root      20   0 2993084 694812  72148 S   0.0   8.8   5:27.18 image-api-go
```

101次请求运行完成后记录：

```
top - 10:26:02 up 24 days, 18:15,  2 users,  load average: 4.31, 1.73, 0.85
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.0 us,  0.8 sy,  0.0 ni, 98.1 id,  0.0 wa,  0.0 hi,  0.1 si,  0.0 st
MiB Mem :   7692.5 total,    174.3 free,   1581.7 used,   5936.4 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   5835.4 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
2327420 root      20   0 2993084 697244  72148 S   0.0   8.9   6:01.23 image-api-go
```

增长2432K。每次内存增长并不稳定，最小大概200k，大部分在1-2M左右。（等待所有请求完成后记录的内存大小，运行时的内存峰值在七百多M）

此问题已经给相关项目提了issue，正在排查解决中。

### 写在最后

wasm已经展现出来了其强大的生命力。即使是处于一个早期阶段，对后端和多语言的各项支持还不是很完善，但是因为其美好的前景，在云原生领域被广泛的接纳。已经有越来越多的webassembly相关项目进入到了cncf中。

从上面的案例可以看出，webassembly作为后端程序被使用时，整体程序架构类似于下图：

![image-20211015145237598](/images/wasm_code_structure.png)

其核心是ABI interface的制定，在不同的细分领域可能会有不同的ABI标准。比如在语言层面交互时可能会是WASI，在代理中间件场景就是proxy-wasm，与dapr运行时交互的业务wasm代码可能会在未来发展出dapr-wasm abi。

在制定完成ABI之后用于编译成wasm的语言，比如c、rust、go等就需要按照abi标准去实现和调用外部函数。这部分胶水代码对于同一个语言来说都是一模一样的，而且在大部分场景下完整的实现完成都会有一定的复杂性，所以自然而然的就会产生不同语言自己的clientSDK。

对于wasm的host程序来说也是一样的，需要实现ABI中定义的导入函数，并将对wasm内的函数调用封装成自身语言的函数，对外提供。这样程序只需要集成ABI host Impl代码，即可像调用本地函数一样调用wasm内实现的函数。

因此我觉得在未来，wasm在各个领域的争夺必有一项是关于自身领域ABI标准规范的话语权的争夺。当然也有可能WASI博取百家之长成为一套完备的ABI规范。

