---
title: "Debug Knative"
linkTitle: "Knative Use"
type: docs
date: 2021-11-04T15:36:50+08:00
draft: false
---

TL;DR

本文主要介绍在本机IED中如何远程调试位于k8s中的knative程序。

### 目标

由于knative组件运行在k8s中，当发生错误的时候，在对代码不是特别熟悉的情况下，单步调试程序能够观察程序运行的函数，以及运行过程中的上下文，对排查问题非常方便。实际knative程序允许你在k8s外运行服务组件，通过环境变量或者命令行参数指定kubeconfig和server参数即可。

但是本文主要介绍另外一种思路，不是所有程序都能方便的运行在本地的。所以对于运行在k8s中的程序进行调试就是本文的目标。

### 步骤

#### 镜像

> knative 镜像制作：https://www.likakuli.com/posts/knative-build/

首先要创建一个镜像，编译的二进制和启动命令都会有所不同。通过查看knative组件

由于knative使用[ko](https://github.com/google/ko)来进行镜像的制作和推送。我简单的看了一下，发现了解这个可能会给我带来一定的时间成本。所以我直接按照自己最舒服的方式自己写了一下Makefile和Dockerfile。

```makefile
REGCFLAGS = -gcflags "all=-N -l"
SRC_FOLDER := $(shell ls cmd)

prepare:
	if [ ! -d "./bin/" ]; then \
    	mkdir bin; \
    fi

default:
	@for dir in ${SRC_FOLDER}; do \
        go build $(REGCFLAGS) -mod vendor -o bin/$$dir ./cmd/$$dir ;  \
    done

remote: prepare default

docker: remote
	@for dir in ${SRC_FOLDER}; do \
        docker build --build-arg BIN=$$dir -t docker4zc/$$dir . ;  docker push docker4zc/$$dir ; \
    done

docker-local: remote
	@for dir in ${SRC_FOLDER}; do \
        docker build --build-arg BIN=$$dir -t docker4zc/$$dir . ; \
    done
```

```dockerfile
FROM golang:latest AS golang
ENV GOPROXY=https://goproxy.cn,direct
RUN CGO_ENABLED=0 go get -ldflags '-s -w -extldflags -static' github.com/go-delve/delve/cmd/dlv


#FROM gcr.oneitfarm.com/distroless/static:noroot
FROM ubuntu
ARG BIN
WORKDIR /

COPY bin/${BIN} /execbin
COPY --from=golang /go/bin/dlv /

CMD ["/dlv", "--listen=:2345", "--headless=true", "--api-version=2", "--accept-multiclient", "exec", "/execbin"]
```

首先你需要在运行命令的地方登陆自己的dockerhub账号，并且将我的dockerhub账号`docker4zc`换成你自己的账号，这样你就可以把镜像推送到自己的仓库。然后你在项目主目录下运行`make docker`即可。

然后你需要在这个网址`https://github.com/knative/serving/releases/download/v0.26.0/serving-core.yaml`(将v0.26.0替换成你需要的版本)下载安装knative serving的yaml文件，并将你想要调试的组件镜像替换，如果是`webhook`组件，那么就是将`gcr.io/knative-releases/knative.dev/serving/cmd/webhook@sha256:d512342e1a1ec454ceade96923e21c24ec0f2cb780e86ced8e66eb62033c74b5`格式的镜像替换成`docker.io/{your dockerhub account}/webhook:latest`。

另外由于我将基础镜像由`gcr.oneitfarm.com/distroless/static:noroot`替换成了`ubuntu`，所以你同时需要将yaml文件中每个deployment下`runAsNonRoot: true`去掉。

接下来你就可以`kubectl apply -f serving-core.yaml`将其部署到k8s中。

### 端口转发

#### 转发一个本地端口到 Pod 端口

以下命令将activator的2345端口转发到本地12345端口

```bash
kubectl port-forward -n knative-serving webhook-7b9b84596d-245rh 12345:2345
```

这相当于

```shell
kubectl port-forward -n knative-serving pods/webhook-7b9b84596d-245rh 12345:2345
```

或者

```shell
kubectl port-forward -n knative-serving deployment/webhook 12345:2345
```

参考[使用端口转发来访问集群中的应用](https://kubernetes.io/zh/docs/tasks/access-application-cluster/port-forward-access-application-cluster/#%E8%BD%AC%E5%8F%91%E4%B8%80%E4%B8%AA%E6%9C%AC%E5%9C%B0%E7%AB%AF%E5%8F%A3%E5%88%B0-pod-%E7%AB%AF%E5%8F%A3)

如果是在minikube中运行，那么你就是在minikube所在机器运行以上命令。并通过机器ip+12345端口进行连接。如果是在k8s中运行，那么你可以在本地将kubectl设置对应的kubeconfig后，通过127.0.0.1+12345端口进行连接。

#### Goland IDE远程调试

VS code和goland都具有此功能，这里以goland为例。在创建时选择go remote类型，设置对应的host和port即可：

![image-20211107153446536](/images/goremotedebug.png)



其他参考：

[在Kubernetes中远程调试Go服务](https://zhuanlan.zhihu.com/p/149938368)

