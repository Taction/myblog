---
title: "Deploying Pluggable K8s"
linkTitle: "Deploying Pluggable K8s"
type: docs
date: 2023-03-16T18:12:04+08:00
draft: false
---

本文以dapr [Hello Kubernetes](https://github.com/dapr/quickstarts/tree/master/tutorials/hello-kubernetes)教程为基础，将其中state组件替换为pluggable state组件，通过这个过程来介绍pluggable component的使用。

### 先决条件

集群里安装一个Redis

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install redis bitnami/redis --set image.tag=6.2
```

安装dapr，[详细安装教程](https://docs.dapr.io/operations/hosting/kubernetes/kubernetes-deploy/)，这里展示通过官方helm chart安装：

```shell
helm repo add dapr https://dapr.github.io/helm-charts/
helm repo update
helm upgrade --install dapr dapr/dapr \
--version=v1.11.0-rc.5 \
--namespace dapr-system \
--create-namespace \
--wait
```

### 组件定义

可以采用手动定义和自动注入（推荐）的方式来定义pluggable component。采用以下两种方式任意一种即可。

[hello-kubernetes](https://github.com/dapr/quickstarts/tree/master/tutorials/hello-kubernetes)中包含了一个在Kubernetes集群中使用dapr的案例，案例中使用了build-in的redis state组件。我们将以这个案例为基础来运行使用pluggable组件，这样可以看到在dapr中使用pluggable需要做哪些修改。

首先我们克隆项目并切换到对应的文件夹

```sh
git clone https://github.com/dapr/quickstarts.git
cd quickstarts/tutorials/hello-kubernetes
```

#### 手动定义

为deployment添加Pluggable的pod以及设置挂载的volume。`docker.io/docker4zc/dapr-pluggable-pubsub:0.0.1`是我之前构建的一个pluggable component，包含`my-component-pubsub`和`my-component-state`两个pluggable组件。

```diff
diff --git a/tutorials/hello-kubernetes/deploy/node.yaml b/tutorials/hello-kubernetes/deploy/node.yaml
index 4db9990..5ae70bf 100644
--- a/tutorials/hello-kubernetes/deploy/node.yaml
+++ b/tutorials/hello-kubernetes/deploy/node.yaml
@@ -34,13 +34,23 @@ spec:
         dapr.io/app-id: "nodeapp"
         dapr.io/app-port: "3000"
         dapr.io/enable-api-logging: "true"
+        dapr.io/unix-domain-socket-path: "/tmp/dapr-components-sockets"
     spec:
+      volumes: ## required, the sockets volume
+        - name: dapr-unix-domain-socket
+          emptyDir: {}
       containers:
       - name: node
         image: ghcr.io/dapr/samples/hello-k8s-node:latest
         env:
         - name: APP_PORT
           value: "3000"
         ports:
         - containerPort: 3000
         imagePullPolicy: Always
+      - name: pluggable
+        image: docker.io/docker4zc/dapr-pluggable-pubsub:0.0.1
+        imagePullPolicy: Always
+        volumeMounts: # required, the sockets volume mount
+          - name: dapr-unix-domain-socket
+            mountPath: /tmp/dapr-components-sockets
```

修改component定义

```diff
diff --git a/tutorials/hello-kubernetes/deploy/redis.yaml b/tutorials/hello-kubernetes/deploy/redis.yaml
index de9be2a..0f6a359 100644
--- a/tutorials/hello-kubernetes/deploy/redis.yaml
+++ b/tutorials/hello-kubernetes/deploy/redis.yaml
@@ -3,7 +3,7 @@ kind: Component
 metadata:
   name: statestore
 spec:
-  type: state.redis
+  type: state.my-component-state
   version: v1
   metadata:
```

#### 自动注入

从dapr 1.11.0开始支持自动注入，下面所示的方式与上述手动定义的方式可以达到完全相同的效果。

首先在App 定义中添加`dapr.io/inject-pluggable-components`注解，允许为此app注入pluggable组件。凡是没有定义`Scopes`或者`Scopes`包含本app-id的pluggable组件都会被自动注入。

```diff
diff --git a/tutorials/hello-kubernetes/deploy/node.yaml b/tutorials/hello-kubernetes/deploy/node.yaml
index 4db99904..a093fdaf 100644
--- a/tutorials/hello-kubernetes/deploy/node.yaml
+++ b/tutorials/hello-kubernetes/deploy/node.yaml
@@ -34,10 +34,11 @@ spec:
         dapr.io/app-id: "nodeapp"
         dapr.io/app-port: "3000"
         dapr.io/enable-api-logging: "true"
+        dapr.io/inject-pluggable-components: "true"
     spec:
       containers:
       - name: node

```

pluggable component定义中添加`dapr.io/component-container`注解，其内容为json格式的container定义。

```diff
diff --git a/tutorials/hello-kubernetes/deploy/redis.yaml b/tutorials/hello-kubernetes/deploy/redis.yaml
index de9be2a1..2e1f34fd 100644
--- a/tutorials/hello-kubernetes/deploy/redis.yaml
+++ b/tutorials/hello-kubernetes/deploy/redis.yaml
@@ -2,20 +2,27 @@ apiVersion: dapr.io/v1alpha1
 kind: Component
 metadata:
   name: statestore
+  annotations:
+    dapr.io/component-container: >
+      {
+        "name": "pluggable",
+        "image": "docker.io/docker4zc/dapr-pluggable-component:0.0.1",
+        "env": [{"name": "NAME", "value": "VALUE"}]
+      }
 spec:
-  type: state.redis
+  type: state.my-component-state
   version: v1
   metadata:
```

### 部署及验证

向集群内添加node app和redis state 组件。

```sh
kubectl apply -f deploy/node.yaml
kubectl apply -f deploy/redis.yaml
```

暴露服务可访问端口

```
kubectl port-forward service/nodeapp --address='0.0.0.0' 8080:80
```

验证service正常运行

```sh
curl http://localhost:8080/ports
```

期望输出

```json
{"DAPR_HTTP_PORT":"3500","DAPR_GRPC_PORT":"50001"}
```

调用`neworder` api 设置order的值：

```shell
curl --request POST --data '{"data":{"orderId":"42"}}' --header Content-Type:application/json http://localhost:8080/neworder
```

期望得到空白输出

通过调用应用程序`order`API来确认订单已经持久化存储到pluggable state中，并且pluggable state组件存取功能正常。

```shell
curl http://localhost:8080/order
```

期望输出

```json
{"orderId":"42"}
```

