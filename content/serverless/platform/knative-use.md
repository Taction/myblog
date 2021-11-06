---
title: "Knative Use"
type: docs
date: 2021-11-01T13:41:20+08:00
draft: false
linkTitle: "Knative Use"
---

本文档主要介绍跟随[官网入门教程](https://knative.dev/docs/getting-started/)和[minikube](https://github.com/csantanapr/knative-minikube)案例运行knative的hello world。中间部分命令根据国内众所周知的网络特点做了一下适配。本篇基本未涉及原理性介绍。

首先确认安装[kind](https://kind.sigs.k8s.io/docs/user/quick-start)或者[minikube](https://minikube.sigs.k8s.io/docs/start/)、[kubectl](https://kubernetes.io/docs/tasks/tools/)、[kn](https://knative.dev/docs/getting-started/#install-the-knative-cli)这些必要的软件。如果你跟随本教程，那么你只需要确认安装minikube和kubectl即可。

### 启动minikube

当minikube镜像拉取过慢的时候可以参考[配置代理](https://www.cxyzjd.com/article/TinyJian/109699420)。或者通过以下命令来运行minikube`minikube start --image-mirror-country='cn' --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'`但是这样是不够的。还是无法解决minikube拉取knative镜像问题。你会发现pod的状态被卡在`ImagePullBackOff`状态中。

作为一个成熟的程序员，命令行代理你肯定已经非常熟悉了。这里主要说明一下启动minikube设置的docker-env是配置在拉取镜像中通过此代理拉取。这里要注意第一要把本地代理client监听0.0.0.0，如果为了安全你可以通过网络安全组来限制只能自己的ip访问这个端口。第二将以下命令行中{your ip}替换成你主机的实际公网ip。

```bash
# 命令行代理
export http_proxy="http://127.0.0.1:1080"
export https_proxy="http://127.0.0.1:1080"
# 启动minikube
minikube start \
--image-mirror-country cn     \
--image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers     \
--docker-env http_proxy=http://{your ip}:1080 \
--docker-env https_proxy=http://{your ip}:1080 \
--docker-env no_proxy=localhost,127.0.0.1,10.96.0.0/12,192.168.99.0/24,192.168.39.0/24
```

### 安装knative

选择要安装的版本

```bash
export KNATIVE_VERSION="0.26.0"
```

定义knative自己的各项CustomResourceDefinition（CRD）来扩展k8s API。

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/v$KNATIVE_VERSION/serving-crds.yaml
kubectl wait --for=condition=Established --all crd
```

创建knative-serving namespace并且安装各项knative组件。在这个yaml里定义了namespace、k8s权限及绑定、一些crd资源以及knative组件deployment。

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/v$KNATIVE_VERSION/serving-core.yaml

kubectl wait pod --timeout=-1s --for=condition=Ready -l '!job-name' -n knative-serving > /dev/null
```

选择你要安装的Net Kourier版本

```bash
export KNATIVE_NET_KOURIER_VERSION="0.26.0"
```

在`kourier-system` namespace下安装kourier

```bash
kubectl apply -f https://github.com/knative/net-kourier/releases/download/v$KNATIVE_NET_KOURIER_VERSION/kourier.yaml
kubectl wait pod --timeout=-1s --for=condition=Ready -l '!job-name' -n kourier-system
kubectl wait pod --timeout=-1s --for=condition=Ready -l '!job-name' -n knative-serving
```

你也可以通过以下命令来查看创建了哪些pod以及它们的状态`watch kubectl get pods -n knative-serving`。

```
$ kubectl get pods -n knative-serving
NAME                                     READY   STATUS    RESTARTS   AGE
activator-7b9b84596d-245rh               1/1     Running   0          23m
autoscaler-65cbff8f7d-bg4w7              1/1     Running   0          23m
controller-7d8f4849d8-dnmsq              1/1     Running   0          23m
domain-mapping-676785d476-jx6dd          1/1     Running   0          23m
domainmapping-webhook-7949444d7d-z8plp   1/1     Running   0          23m
webhook-58975ff8d-kqtrx                  1/1     Running   0          23m
```

当所有的都处于running状态的时候，即启动完成，否则你可以根据对应的status判断是否出现了问题。

新开一个命令行终端运行以下命令。您需要这样做才能使用`EXTERNAL-IP`for kourier Load Balancer 服务。

```
minikube tunnel
```

将环境变量设置为`EXTERNAL_IP`工作节点的外部 IP 地址，您可能需要多次运行此命令，直到服务就绪。

```bash
EXTERNAL_IP=$(kubectl -n kourier-system get service kourier -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo EXTERNAL_IP=$EXTERNAL_IP
```

使用以下命令将环境变量设置`KNATIVE_DOMAIN`为 DNS 域`nip.io`，并检查dns可以被正常解析。

```bash
KNATIVE_DOMAIN="$EXTERNAL_IP.nip.io"
echo KNATIVE_DOMAIN=$KNATIVE_DOMAIN
dig $KNATIVE_DOMAIN
```

为 Knative 服务配置 DNS

```bash
kubectl patch configmap -n knative-serving config-domain -p "{\"data\": {\"$KNATIVE_DOMAIN\": \"\"}}"
```

配置knative使用kourier

```bash
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress.class":"kourier.ingress.networking.knative.dev"}}'
```

接下来你就可以进行服务部署了。

### 部署服务

```
cat <<EOF | kubectl apply -f -
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    spec:
      containers:
        - image: gcr.oneitfarm.com/knative-samples/helloworld-go
          ports:
            - containerPort: 8080
          env:
            - name: TARGET
              value: "Knative"
EOF
```

等待服务部署完成

```bash
kubectl wait ksvc hello --all --timeout=-1s --for=condition=Ready
```

获取服务的访问url

```bash
SERVICE_URL=$(kubectl get ksvc hello -o jsonpath='{.status.url}')
echo $SERVICE_URL
```

访问服务,这个时候控制台应该会输出`Hello Knative`

```
$ curl $SERVICE_URL
```

我定义了一下输出内容，这样可以看到运行的时候的请求延迟。分别对服务进行冷启动请求和热请求测试。

```
curl -w "@curl-format.txt" $SERVICE_URL
```

运行结果

```bash
# 冷启动请求
curl -w "@curl-format.txt" $SERVICE_URL
Hello Knative!
time_namelookup:  0.001831
       time_connect:  0.002052
    time_appconnect:  0.000000
      time_redirect:  0.000000
   time_pretransfer:  0.002110
 time_starttransfer:  1.966370
                    ----------
         time_total:  1.966427
         
# 热启动请求
curl -w "@curl-format.txt" $SERVICE_URL
Hello Knative!
time_namelookup:  0.194799
       time_connect:  0.195032
    time_appconnect:  0.000000
      time_redirect:  0.000000
   time_pretransfer:  0.195137
 time_starttransfer:  0.196945
                    ----------
         time_total:  0.196989
```

在这个过程中我们可以另外开一个命令行，通过watch pods来查看当有请求时容器被启动，当一段时间内没有请求后，容器数量被缩放到0这一现象。

![image-20211101204945605](/images/knative-scale.png)

#### 部署新版本及分流

默认情况下knative会将所有流量都导入新版本，你可以增加`traffic`字段来指定不同版本的流量比例。

```
cat <<EOF | kubectl apply -f -
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    metadata:
      name: hello-knative-v2
    spec:
      containers:
        - image: gcr.oneitfarm.com/knative-samples/helloworld-go
          ports:
            - containerPort: 8080
          env:
            - name: TARGET
              value: "Knative V2"
  traffic:
  - latestRevision: true
    percent: 50
  - revisionName: hello-00001
    percent: 50
EOF
```

配置成功控制台会输出

```
service.serving.knative.dev/hello configured
```

这个时候我们再请求就会发现流量被路由到了不同的版本，你可以通过以下命令来查看不同的版本。

```bash
kn revisions list
# or
kubectl get revisions
```

#### 服务缩放定义

可以通过annotations中的定义来指定缩放策略。比如以下策略可以定义从1-5的pod缩放。

```bash
cat <<EOF | kubectl apply -f -
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    metadata:
      name: hello-knative-v3
      annotations:
        # the minimum number of pods to scale down to
        autoscaling.knative.dev/minScale: "1"
        # the maximum number of pods to scale up to
        autoscaling.knative.dev/maxScale: "5"
        # Target in-flight-requests per pod.
        autoscaling.knative.dev/target: "1"
    spec:
      containers:
        - image: gcr.oneitfarm.com/knative-samples/helloworld-go
          ports:
            - containerPort: 8080
          env:
            - name: TARGET
              value: "Knative V3"
EOF
```

这时用hey压测就会得到以下结果

```bash
$ hey -z 30s -c 50 http://hello.default.10.104.46.103.nip.io

Summary:
  Total:	30.0150 secs
  Slowest:	2.1715 secs
  Fastest:	0.0024 secs
  Average:	0.0353 secs
  Requests/sec:	1412.4594

  Total data:	763110 bytes
  Size/request:	18 bytes

Response time histogram:
  0.002 [1]	|
  0.219 [42314]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.436 [30]	|
  0.653 [0]	|
  0.870 [0]	|
  1.087 [0]	|
  1.304 [0]	|
  1.521 [0]	|
  1.738 [0]	|
  1.955 [0]	|
  2.171 [50]	|


Latency distribution:
  10% in 0.0151 secs
  25% in 0.0204 secs
  50% in 0.0285 secs
  75% in 0.0399 secs
  90% in 0.0546 secs
  95% in 0.0671 secs
  99% in 0.1100 secs

Details (average, fastest, slowest):
  DNS+dialup:	0.0024 secs, 0.0024 secs, 2.1715 secs
  DNS-lookup:	0.0024 secs, 0.0000 secs, 2.0054 secs
  req write:	0.0000 secs, 0.0000 secs, 0.0108 secs
  resp wait:	0.0329 secs, 0.0023 secs, 0.2963 secs
  resp read:	0.0001 secs, 0.0000 secs, 0.0073 secs
```

这个时候我们查看pod状态，就可以看到其中有一个预启动的比其他存活时间更久

```bash
$ k get pods
NAME                                          READY   STATUS    RESTARTS   AGE
hello-knative-v3-deployment-fdf969d79-b8zts   2/2     Running   0          81s
hello-knative-v3-deployment-fdf969d79-gr9t2   2/2     Running   0          3m17s
hello-knative-v3-deployment-fdf969d79-qhd2s   2/2     Running   0          81s
hello-knative-v3-deployment-fdf969d79-sfb8g   2/2     Running   0          81s
hello-knative-v3-deployment-fdf969d79-sz4hl   2/2     Running   0          81s
```

注意：如果你设置`minScale > 0`将导致每个`Revision`Pod 始终至少运行指定数量的 Pod，尽管它们没有获得任何流量。在这种情况下，不要忘记清理旧版本。

### 后记

通过本篇文章，你可以在自己的minikube中运行knative并且观察其对于服务的实际缩放的现象。并且能够看到服务的冷热启动的请求延迟。

接下来计划陆续会进行远程调试k8s中的knative组件、对knative组件分析、knative与kong结合案例的分析、knative关键源码分析。



