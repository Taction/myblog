---
type: docs
title: "运行docker wasm preview 失败教程"
linkTitle: "docker wasm preview 踩坑指北"
date: 2022-11-28T19:02:27+08:00
weight: 200
draft: false
---

### 背景

docker宣布支持wasm，并且在wasm day上做了演讲。看到这个消息的时候初步跑了一下官方教程，成功的失败了，止步在`containerd-shim-wasmedge-v1`没有找到的错误。最近相关的消息多了起来，于是就心血来潮打算抽一些时间跑一下，提供一个趟坑教程方便想运行一下的小伙伴。

先说下最终的结果当然是失败的，本文记述了在这个过程中遇到的问题以及尝试的方案。最终本次尝试停止在找到一个能运行的docker版本。

### 按照官方文档安装基础组件

#### 安装 docker preview和WasmEdge

首先需要安装docker preview版本，在官方文档里有下载链接。这里以linux下的安装为例，其他类型的可以在官方文档处找到下载链接。

如果你已经安装了docker，你应该先卸载旧的版本。可以通过官方教程或者以下命令卸载：

```sh
sudo apt-get remove docker docker-engine docker.io containerd runc
# if desktop
sudo apt remove docker-desktop
rm -r $HOME/.docker/desktop
sudo rm /usr/local/bin/com.docker.cli
sudo apt purge docker-desktop
```

接下来开始安装流程。

PS:对于非Gnome Desktop环境，必须安装`gnome-terminal`。

```
sudo apt install gnome-terminal
```

对于linux来说你可以通过以下命令安装，否则你需要跳转到[官方文档](https://docs.docker.com/desktop/wasm/)的下载链接，下载对应自己机型的并进行安装。

```
wget https://www.docker.com/download/wasm-preview/linuxamd64deb
mv linuxamd64deb linuxamd64.deb
sudo apt-get install ./linuxamd64.deb
```

然后你需要安装WasmEdge，你可以通过以下命令来进行安装

```
curl -sSf https://raw.githubusercontent.com/WasmEdge/WasmEdge/master/utils/install.sh | sudo bash -s -- -e all -p /usr/local
```

对于官方文档来说基本教程就是这些，接下来你可以运行以下命令来运行wasm程序。但是很显然，如果能运行成功的话，我就不会写这篇文档了。

```
docker run -dp 8080:8080 \
  --name=wasm-example \
  --runtime=io.containerd.wasmedge.v1 \
  --platform=wasi/wasm32 \
  michaelirwin244/wasm-example
```

接下来你会遇到第一个报错,这是由于containerd-shim-wasmedge-v1没有安装。

```
Error response from daemon: failed to start shim: failed to resolve runtime path: runtime "io.containerd.wasmedge.v1" binary not installed "containerd-shim-wasmedge-v1": file does not exist: unknown
```

### 开始踩坑

#### 安装containerd-shim-wasmedge-v1

1. 从 `containerd-shim-wasmedge-v1` 的 [release page](https://github.com/second-state/runwasi/releases)下载二进制
2. 把它放在你 `$PATH`包含的某个路径下 (e.g. `/usr/local/bin`)

当然你可以通过运行以下命令来安装。

```
wget https://github.com/second-state/runwasi/releases/download/v0.3.3/containerd-shim-wasmedge-v1-v0.3.3-linux-amd64.tar.gz
tar -zxvf containerd-shim-wasmedge-v1-v0.3.3-linux-amd64.tar.gz
sudo mv containerd-shim-wasmedge-v1 /usr/local/bin/
```

安装之后你可以再次运行这个命令来启动一个wasm程序

```sh
docker run -dp 8080:8080 \
  --name=wasm-example \
  --runtime=io.containerd.wasmedge.v1 \
  --platform=wasi/wasm32 \
  michaelirwin244/wasm-example
```

接下来你会遇到你的第二个报错

```
docker: Error response from daemon: Unknown runtime specified io.containerd.wasmedge.v1.
See 'docker run --help'.
```

这是由于你的docker启动配置还有问题，没有定义`io.containerd.wasmedge.v1`这个运行时。

因此将下列内容加入到你的配置文件中`sudo vim /etc/docker/daemon.json`,注意io.containerd.wasmedge.v1的路径要跟你的安装路径一致。

```json
{
	"features": {
		"containerd-snapshotter": true
	},
	"runtimes": {
		"io.containerd.wasmedge.v1": {
			"path": "/usr/local/bin/containerd-shim-wasmedge-v1",
			"runtimeArgs": []
		}
	}
}
```

然后你需要重启你的docker

```sh
sudo systemctl daemon-reload
sudo systemctl restart docker
```

你可以通过`docker  infor`命令来查看配置是否添加成功,成功后你的Runtimes中应该包含`io.containerd.wasmedge.v1`。Containerd 在调用时会将 shim 的名称解析为二进制文件，并在 $PATH 中查找这个二进制文件。例如 io.containerd.runc.v2 会被解析成二进制文件 containerd-shim-runc-v2，io.containerd.runhcs.v1 会被解析成二进制文件 containerd-shim-runhcs-v1.exe。客户端在创建容器时可以指定使用哪个 shim，如果不指定就使用默认的 shim。以下是省略部分信息的输出

```
Client:
Server:
 Runtimes: io.containerd.runtime.v1.linux io.containerd.wasmedge.v1 runc io.containerd.runc.v2
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 1c90a442489720eec95342e1789ee8a5e1b9536f
 runc version: v1.1.4-0-g5fd4c4d
 init version: de40ad0
```

然后足够幸运的话你会遇到接下来的这个报错：

```
Using default tag: latest
latest: Pulling from michaelirwin244/wasm-example
operating system is not supported
```

这个错误就比较没有头绪了，因为功能比较新，所以这个错误以及加上docker wasm的关键词并没有查找到比较有用的内容。在项目issue里这些关键字加上WasmEdge也没有搜索到比较有用的信息。

首先怀疑的是这是一个高阶配置，而目前没有文档说明。这种一般通过源码梳理一下就能比较方便找到怎么配置的。但是看到源码的时候感觉可能不是这么回事。那么也可能是分支错了，由于我下载的是官网preview版本，但是我将代码切到对应的分支和commit-hash，也不行。贴一下其中一个关键函数就明白了：

```go
// IsOSSupported determines if an operating system is supported by the host
func IsOSSupported(os string) bool {
	if strings.EqualFold("windows", os) {
		return true
	}
	if LCOWSupported() && strings.EqualFold(os, "linux") {
		return true
	}
	return false
}
```

只能反向整理搜索相关资料，交叉验证是否在安装的时候有错漏的步骤。终于结合这篇文章[Installing Docker Engine with WASM support](https://github.com/chris-crone/wasm-day-na-22/tree/main/server)可以看到我们可以自己手动编译二进制，基于的是fork出来的项目的`wasmedge`分支。但是很遗憾，点进[这个项目](https://github.com/rumpl/moby/)你会发现没有`wasmedge`这个分支，来晚了几天已经被删除了:(。

##### containerd-shim-wasmedge-v1兼容性

关于/usr/local/bin/containerd-shim-wasmedge-v1还有一个小问题,它对，特别是你的系统版本比较低的情况下会有问题。运行一下`/usr/local/bin/containerd-shim-wasmedge-v1`如果 报错信息中含有`/lib/x86_64-linux-gnu/libc.so.6: version 'GLIBC_2.32' not found (required by ./containerd-shim-wasmedge-v1)`那么就表明你系统的libc版本太低了。GLIBC版本只要大于等于2.32就可以。由于这个库比较重要，不建议单独升级，如果想升级的话建议直接升级系统版本更简单。（别问我为什么提这个建议，都是泪）

#### 自己构建docker二进制

这个时候我找到一次[提交](https://github.com/rumpl/moby/commit/1a3d8019d1ddb82e8a6b437a8eccf2d22cbc8b5d)，就很简单把调用`IsOSSupported`函数校验的代码删掉就行了。。。这个时候我看了下我从官方下载的dockerd版本，是`20.10`的，由于对docker源码不熟，加上我们只是运行下这个功能验证感受一下，所以我直接下载官方源码切到版本分支，把`IsOSSupported`这个函数直接返回true不就行了嘛。然后构建并运行。

```sh
git clone https://github.com/moby/moby.git
cd moby
git checkout -b 20.10 origin/20.10
# 打开文件修改代码直接return true
vim pkg/system/lcow.go
vim pkg/system/lcow_unsupported.go
```

然后执行`make binary`（注意国内用户此命令需要在网络通畅的环境下运行）构建后的dockerd文件在`./bundles/binary-daemon/dockerd`路径下。由于我们跟官方preview用的相同版本，所以其他的依赖的组件版本肯定是兼容的，你可以用构建的这个dockerd替换本机dockerd，或者通过以下方式来运行一个新的dockerd并通过它来启动容器。

```sh
nohup sudo -b sh -c "./bundles/binary-daemon/dockerd -D -H unix:///tmp/docker.sock --data-root /tmp/root --pidfile /tmp/docker.pid"
docker context create wasm --docker "host=unix:///tmp/docker.sock"
# 注意下面的指令可以不运行只要在每次的docker命令后面加上--context wasm就可以
docker context use wasm
```

使用官方版本改代码的方式，在运行上面启动一个wasm docker的时候遇到了以下错误，查看dockerd日志是代码panic了，所以肯定不是这个分支的代码。

```
Unable to find image 'michaelirwin244/wasm-example:latest' locally
latest: Pulling from michaelirwin244/wasm-example
e049f00c5289: Pull complete
docker: unexpected EOF.
See 'docker run --help'.
```

由于wasm day的文档是指向[这个项目](https://github.com/rumpl/moby)自己构建二进制的，而且目标分支已经删除了，让我们看下这个项目的哪个分支可能是包含了这部分功能。首先可以看到`wasmedge`分支在被删除之前是在尝试合并入`c8d`分支，那么这个分支很有可能就是目前此功能的开发分支。那么就remote添加这个源，并且把分支切到这个分支上，构建并运行。

这个时候我就遇到本次试验之旅的大boss，最后一个问题的报错如下：

```
docker: Error response from daemon: failed to create shim task: OCI runtime create failed: unable to retrieve OCI runtime error (open /run/containerd/io.containerd.runtime.v2.task/moby/bec2a500d29e50539dddc3050ac823575053f70e75f8e0be814eb7cb84a345d1/log.json: no such file or directory): /usr/local/bin/containerd-shim-wasmedge-v1 did not terminate successfully: exit status 1: unknown.
```

对应的dockerd报错为

```
DEBU[2022-11-29T16:41:45.018483384+08:00] DeleteConntrackEntries purged ipv4:0, ipv6:0
DEBU[2022-11-29T16:41:45.022664827+08:00] EnableService c0feb9e8fe0df448f23a5a74014e27def55d6ad5c1d4af295f1e0417af821caf START
DEBU[2022-11-29T16:41:45.022690439+08:00] EnableService c0feb9e8fe0df448f23a5a74014e27def55d6ad5c1d4af295f1e0417af821caf DONE
DEBU[2022-11-29T16:41:45.033219799+08:00] createSpec: cgroupsPath: system.slice:docker:c0feb9e8fe0df448f23a5a74014e27def55d6ad5c1d4af295f1e0417af821caf
DEBU[2022-11-29T16:41:45.036729599+08:00] bundle dir created                            bundle=/var/run/docker/containerd/c0feb9e8fe0df448f23a5a74014e27def55d6ad5c1d4af295f1e0417af821caf module=libcontainerd namespace=moby root=rootfs
ERRO[2022-11-29T16:41:45.134961283+08:00] stream copy error: reading from a closed fifo
ERRO[2022-11-29T16:41:45.137885178+08:00] stream copy error: reading from a closed fifo
DEBU[2022-11-29T16:41:45.168345866+08:00] Revoking external connectivity on endpoint ecstatic_varahamihira (6b6101c5cf94aa2040b69f4e02dbfec84c98689d63a7473515f3d5597e9621a6)
DEBU[2022-11-29T16:41:45.169235448+08:00] /usr/sbin/iptables, [--wait -t nat -C DOCKER -p tcp -d 0/0 --dport 8080 -j DNAT --to-destination 172.17.0.2:8080 ! -i docker0]
DEBU[2022-11-29T16:41:45.171210165+08:00] /usr/sbin/iptables, [--wait -t nat -D DOCKER -p tcp -d 0/0 --dport 8080 -j DNAT --to-destination 172.17.0.2:8080 ! -i docker0]
```

这里有几种可能性，一个是我用的docker版本呢还是不对，因为毕竟原来的分支被删除了，这个分支的代码可能仍然是有问题的。另一个是dockerd、containerd、containerd-shim-wasmedge-v1等组件之间的版本存在不兼容情况（因为我从官方的release分支切换到了一个开发者分支）。

首先针对第一中情况，继续看分支，开会做案例的时间跟这两个分支比较接近 backup-c8d-2022-10-21、backup-c8d-2022-11-07，所以依次尝试了下，不行。

然后看到官方修改oscheck的[代码pr](https://github.com/moby/moby/pull/44181/files)刚好就是他提交的，在remove-os-check分支上，尝试了下这个分支代码，依然不行。

好了本次的踩坑之旅到此就结束了，如果想要继续深入尝试的话应该需要对这些组件有更进一步的了解。而且在官方issue中也有说明，这个功能目前还是试验性质的，没有production ready，还有很多功能要做。所以还是等官方这个功能稳定之后吧。

附一张架构截图，docker engine会将启动信息下发到containerd，containerd下发到containerd-shim，containerd-shim发送到对应的运行时，在本例中就是wasmedge，然后wasmedge运行对应的wasm程序。

![docker-containerd-wasm-diagram.png](https://www.docker.com/wp-content/uploads/2022/10/docker-containerd-wasm-diagram.png.webp)

于2022年11月19日

### 参考

https://docs.docker.com/desktop/wasm/

https://www.docker.com/blog/docker-wasm-technical-preview/

https://github.com/chris-crone/wasm-day-na-22/tree/main/server

https://github.com/NVIDIA/nvidia-docker/issues/838

https://docs.docker.com/desktop/install/ubuntu/

https://docs.docker.com/engine/install/ubuntu/

https://github.com/second-state/runwasi

https://blog.csdn.net/wq_0708/article/details/121105055

https://blog.csdn.net/dayidson/article/details/5067167

https://askubuntu.com/questions/1345342/how-to-install-glibc-2-32-when-i-already-have-glibc2-31

https://computingforgeeks.com/how-to-install-docker-on-ubuntu/

https://docs.docker.com/desktop/wasm/

https://openyurt.io/docs/v0.6.0/user-manuals/runtime/WasmEdge/

 https://github.com/second-state/crunw

https://github.com/second-state/wasmedge-containers-examples/blob/main/containerd/README.md

https://www.freecodecamp.org/news/edge-cloud-microservices-with-wasmedge-and-rust/

https://github.com/WasmEdge/wasmedge_hyper_demo

