---
title: "Knative Queue"
linkTitle: "Knative Queue"
type: docs
date: 2021-12-01T15:46:27+08:00
draft: false
---

### 简介

queue是knative在每个deployment都会为业务容器注入的"sidecar"，负责其入口流量代理行为。并且会对请求进行计数，对外暴露metrice接口，autoscaler会定期拉取这些指标数据。

queue的几个端口表示如下：

• 8012： queue-proxy 代理的http端口，流量的入口都会到 8012
• 8013： http2 端口，用于grpc流量的转发
• 8022： queue-proxy 管理端口，如健康检查
• 9090： queue-proxy的监控端口，暴露指标供 autoscaler 采集，用于扩缩容
• 9091： prometheus 应用监控指标（请求数，响应时长等）

此外还有一个通过环境变量USER_PORT配置的用户容器端口，即业务实际暴露的服务端口，最初是在ksvc container port中配置的，然后一步一步带下来的。

### 指标上报

#### 指标上报server

> queue的指标上报是比较简单的一个逻辑，首先是在有请求的时候进行计数，然后暴露拉取接口。

queue会启动一个metrics server。同时支持protobuf和json格式的数据上报，原理都是一致的， 接下来以http reporter为例介绍。

```go
func buildMetricsServer(promStatReporter *queue.PrometheusStatsReporter, protobufStatReporter *queue.ProtobufStatsReporter) *http.Server {
   metricsMux := http.NewServeMux()
   metricsMux.Handle("/metrics", queue.NewStatsHandler(promStatReporter, protobufStatReporter))
   return &http.Server{
      Addr:    ":" + strconv.Itoa(networking.AutoscalingQueueMetricsPort),
      Handler: metricsMux,
   }
}
```

reporter创建，首先检查指标上报labelnamespace、configuration、revision、pod是否配置，然后都是prometheus一些模式化的代码，将各个指标数据设置为自己的成员变量。

```go
// NewPrometheusStatsReporter creates a reporter that collects and reports queue metrics.
func NewPrometheusStatsReporter(namespace, config, revision, pod string, reportingPeriod time.Duration) (*PrometheusStatsReporter, error) {
   if namespace == "" {
      return nil, errors.New("namespace must not be empty")
   }
   if config == "" {
      return nil, errors.New("config must not be empty")
   }
   if revision == "" {
      return nil, errors.New("revision must not be empty")
   }
   if pod == "" {
      return nil, errors.New("pod must not be empty")
   }

   registry := prometheus.NewRegistry()
   for _, gv := range []*prometheus.GaugeVec{
      requestsPerSecondGV, proxiedRequestsPerSecondGV,
      averageConcurrentRequestsGV, averageProxiedConcurrentRequestsGV,
      processUptimeGV} {
      if err := registry.Register(gv); err != nil {
         return nil, fmt.Errorf("register metric failed: %w", err)
      }
   }

   labels := prometheus.Labels{
      destinationNsLabel:     namespace,
      destinationConfigLabel: config,
      destinationRevLabel:    revision,
      destinationPodLabel:    pod,
   }

   return &PrometheusStatsReporter{
      handler:   promhttp.HandlerFor(registry, promhttp.HandlerOpts{}),
      startTime: time.Now(),

      reportingPeriodSeconds: reportingPeriod.Seconds(),

      requestsPerSecond:                requestsPerSecondGV.With(labels),
      proxiedRequestsPerSecond:         proxiedRequestsPerSecondGV.With(labels),
      averageConcurrentRequests:        averageConcurrentRequestsGV.With(labels),
      averageProxiedConcurrentRequests: averageProxiedConcurrentRequestsGV.With(labels),
      processUptime:                    processUptimeGV.With(labels),
   }, nil
}
```

请求数量的记录借助了一个中间的结构体`RequestStats`，在构建代理pod请求的server的时候会传入此结构体的一个实例，在请求到达时进行计数。然后会周期取出各项指标，设置到`PrometheusStatsReporter`中供拉取指标时使用。

#### 指标记录

将proxyHandler路由中间件逻辑简化后如下所示，就是在流量到达和转发完成后分别触发stats HandleEvent来进行记录。

```go
// ProxyHandler sends requests to the `next` handler at a rate controlled by
// the passed `breaker`, while recording stats to `stats`.
func ProxyHandler(breaker *Breaker, stats *network.RequestStats, tracingEnabled bool, next http.Handler) http.HandlerFunc {
   return func(w http.ResponseWriter, r *http.Request) {
      
      stats.HandleEvent(network.ReqEvent{Time: time.Now(), Type: in})
      defer func() {
         stats.HandleEvent(network.ReqEvent{Time: time.Now(), Type: out})
      }()
      // ......
      next.ServeHTTP(w, r)
   }
}
```

HandleEvent就是针对不同情况下的出入流量行为进行对应的计数。如果入请求就将并发数和请求数增加。如果是请求结束就将并发数减一。如果是activator代理过来的入请求就在以上基础上对将代理并发数和请求数加一，如果是代理请求结束就在以上基础上将代理并发减一。

```go
// HandleEvent handles an incoming or outgoing request event and updates
// the state accordingly.
func (s *RequestStats) HandleEvent(event ReqEvent) {
   s.mux.Lock()
   defer s.mux.Unlock()

   s.compute(event.Time)

   switch event.Type {
   case ProxiedIn:
      s.proxiedConcurrency++
      s.proxiedCount++
      fallthrough
   case ReqIn:
      s.requestCount++
      s.concurrency++
   case ProxiedOut:
      s.proxiedConcurrency--
      fallthrough
   case ReqOut:
      s.concurrency--
   }
}
```

#### 指标更新

指标更新是在main函数中定义的，定时从stats中取出指标，将其更新到http和protobuf的metrics server中。

```go
func main() {
   protoStatReporter := queue.NewProtobufStatsReporter(env.ServingPod, reportingPeriod)

   reportTicker := time.NewTicker(reportingPeriod)
   defer reportTicker.Stop()

   stats := network.NewRequestStats(time.Now())
   go func() {
      for now := range reportTicker.C {
         stat := stats.Report(now)
         promStatReporter.Report(stat)
         protoStatReporter.Report(stat)
      }
   }()
}
```

对于http metrics server指标更新非常简单，由于这些指标项已经按prometheus要求初始化，只要对这些值进行设置就可以。

```go
// Report captures request metrics.
func (r *PrometheusStatsReporter) Report(stats network.RequestStatsReport) {
   // Requests per second is a rate over time while concurrency is not.
   r.requestsPerSecond.Set(stats.RequestCount / r.reportingPeriodSeconds)
   r.proxiedRequestsPerSecond.Set(stats.ProxiedRequestCount / r.reportingPeriodSeconds)
   r.averageConcurrentRequests.Set(stats.AverageConcurrency)
   r.averageProxiedConcurrentRequests.Set(stats.AverageProxiedConcurrency)
   r.processUptime.Set(time.Since(r.startTime).Seconds())
}
```

如果你对这部分代码想要详细了解，但是对Prometheus收集指标不太了解的话，你可以在它的[client项目](github.com/prometheus/client_golang)获得更多了解。
