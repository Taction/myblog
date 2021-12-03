---
type: docs
no_list: true
title: "platform"
weight: 10
---

介绍在serverless的一些不同的平台及其分析。目前主要是对于knative的介绍。



[运行knative hello world]({{< ref knative-use.md >}})，对knative有一个大概的了解。

[如何 debug knative]({{< ref debug-knative.md >}})，debug运行在k8s的knative程序，可以更好的了解程序运行细节。

[knative整体介绍及CRD介绍]({{< ref knative-crd.md >}})，对knative有个简单的介绍，同时对controller源码结构有个简单的介绍。

[knative从0扩缩容介绍及源码解析]({{< ref knative-scalefrom0.md >}})，本文主要介绍从0扩缩容原理，及从源码角度讲述这个过程中涉及的各个组件及其代码同时也包含扩容的决策逻辑。

[knative代理组件的指标上报]({{< ref knative-queue.md >}})，本文主要从源码角度介绍knative代理组件，总体来看这部分代码相对还是比较简单的。

最后以一个实际案例的方式，了解如果我们如果想[扩展knative revision支持引用deployment]({{< ref knative-demo.md >}})应该对代码进行哪些修改。

// todo:对knative核心组件的性能压测



