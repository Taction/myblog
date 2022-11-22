---
title: "一次关于sidecar CPU使用率异常偶发升高问题排查"
linkTitle: "sidecar cpu"
type: docs
date: 2021-11-20T15:58:16+08:00
draft: false
---



### 问题描述

公司采用了自研的sidecar，是用go语言实现的，在部署的时候限制了sidecar的CPU占用为0.5核。且在sidecar中实现了从`/proc/stat`中实时采集sidecar CPU和内存占用并上报的逻辑。在一次版本更新过后，从监控指标上发现sidecar存在异常的CPU毛刺。公司总共有3千多微服务，其中部分流量非常低，但是所有服务都会偶发且无规律的出现CPU占用达到上限的情况。有的持续几ms，有的持续1-2s，虽然每个服务每天可能只有几次，考虑微服务数量就会每分钟都会有几条。

而且这个现象针对单个服务来看具有偶发性和随机性，跟内存和网络io的特征都不匹配。没法确定是由于什么原因导致的，这样一是没有太好的排查思路，二是也没有办法通过某种特征来判断接下来会发生CPU占用高的情况，从而导致后面的分析都只能是从发生了CPU占用开始分析，而不能抓取到将要发生时的各项指标。

由于此次更新的代码基本上都是我提交的，而且对比之前的监控指标，发现也会有此现象但是触发频率低很多。由于代码更新量不大，所以第一反应是把自己的代码又从头到尾看了一遍。十分确定没有任何问题，看起来最有可能导致问题的也只是引入了两个常驻协程，对于异常和泄露的场景也都是有考虑的。

#### pprof抓取CPU占用

sidecar是有pprof端口的，而且在CPU占用过高的时候是有日志的，而且有时持续时间长达几秒，为我们抓取CPU占用提供了可能性。接下来实现了一个简单的脚本，每当发生此情况的时候获取 CPU profile。选取一个抓取到信息的其调用栈如下所示：

![img](https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F84484c0c-d633-4d6b-af88-6c1d7af2f486%2FUntitled.png?table=block&id=ff1eccd2-2051-41a6-9bac-69da0990f548&spaceId=f8cc2f76-a72c-46dc-97ca-570846beea3f&width=2000&userId=e6d95cb5-99c7-4b94-805e-7050fb0e6859&cache=v2)

问题的主要难点在于pprof抓到的函数占用没有业务代码，完全是go运行时。顿时心里破大防了，不要慌问题不大，反正都这样了，慌也没有用。

![IMG_2682](https://image-1255620078.cos.ap-nanjing.myqcloud.com/IMG_2682.JPG)

接下来就只能面向谷歌先看下有没有类似的问题，再看下这些函数具体作用是什么，什么情况下go runtime findrunnable（schedule）会占用CPU过高。经过浏览器不懈的努力，找到了一些信息[golang runtime.findrunnable epoll_wait lock 占用CPU 过多排查](https://gobea.cn/blog/detail/OrJAMa6d.html)、[slow down 10+ times, what happen to runtime.findrunnable?](https://groups.google.com/g/golang-nuts/c/6zKXeCoT2LM)，甚至第一个问题的CPU占用跟我们的几乎一模一样，但是都是只提出了问题，没有最终的解答。最终在go issue里找到一个看起来有点关系，也不能说是完全没用的[信息](https://github.com/golang/go/issues/18237#issuecomment-265617249)：

```
Update: most of the scheduler activity is caused by blocking network reads.

The call chain goes across two call stacks, which makes it a little tough to track down through stack traces alone, but here it is:

net.(*netFD).Read
net.(*pollDesc).wait
net.(*pollDesc).waitRead
net.runtime_pollWait
runtime.netpollblock
runtime.gopark
runtime.mcall(park_m)
runtime.park_m
runtime.schedule
runtime.findrunnable
(etc)
The raw call counts suggest that roughly 90% of the runtime.schedule calls are a consequence of this particular chain of events.

@davecheney I haven't extracted our profile format into the pprof format yet, but I hope that answers the same question you were hoping the svg web would answer.
```

这里出现了我们在pprof文件里的占用时间大头的函数`runtime.mcall(park_m)`、`runtime.park_m`、`runtime.schedule`、`runtime.findrunnable`！！！虽然这里没有告诉我们究竟为什么。但是告诉了我们几个关键信息，1. 这是个调度问题，而且举的例子是发生在不同的goruntine切换执行的时候。2. 这种情况阻塞等待读取网络内容的时候。在我的问题中这是一个异常现象，这里告诉我们网络可能会引起这个现象，但是没有告诉我们为什么会引起这个现象。

问题似乎在这个时候陷入了瓶颈，难道要先把go的调度原理啃下来才能解决这个问题吗？那得到猴年马月去啊。这个时候脑子里其实隐约有点感觉，但是说不太清，反正这是一个关于go 调度上的问题，于是就继续查阅了一些资料。加上之前看小黄书的时候，里面完整的介绍过go的GMP模型。而且这可能跟程序运行在容器里相关。于是继续查阅go在容器里调度或者运行时或者routine相关的信息。最终形成了一个可能的怀疑方向：

就是由于P（理论值应该等于CPU核数）数量设置的不正确，获取的是机器的CPU核数（8），但是实际上应该获取给sidecar的CPU核数（1）。我觉得简单打个比方就是实际上只有一个物理线程，但是go调度器以为有8个物理线程，当新的并发来的时候，假设1个物理线程已经被占用了，但是go以为还可以继续运行新来的协程，就会为它寻找可用线程，但是没有了可能找不到，就可能在这个地方导致CPU空转或者阻塞。

[GOMAXPROCS 的坑](https://pandaychen.github.io/2020/02/28/GOMAXPROCS-POT/)

[cpu使用率_Go服务在容器内CPU使用率异常问题排查手记_weixin_39643189的博客-CSDN博客](https://blog.csdn.net/weixin_39643189/article/details/109785603)

### 解决方案

在容器中打印不做任何处理时P数量，跟我们设想的是一致的，获取的机器核数量。接下来就比较简单了，获取容器中被限制的具体的CPU核数有比较成熟的库[uber-go/automaxprocs](https://github.com/uber-go/automaxprocs)，在使用之前大概对库的代码原理做了了解，并通过模型试验确认在当前条件下这个库是可以正常工作的，且对于边界条件（小于1核）情况已经做了对应的处理。这里有一个对[Uber-Automaxprocs 的代码分析](https://pandaychen.github.io/2020/02/29/AUTOMAXPROCS-ANALYSIS/)。

##### 后记

在后来看到dapr中有一个issue [daprd runtime handles logic server request to cost too much time occasionally](https://github.com/dapr/dapr/issues/3759) 。看到这个issue的时候就感觉世界线闭合了，很可能是同样的问题，于是提了一个pr并且得到反馈应该是解决了这个issue。而且还记得我们第一次看到的go issue里的回复吗？这个现象通常会由网络引起！其实根因还是在于协程调度会出问题，之所以在网络情景下明显，是因为网络必然伴随着协程的调度。

