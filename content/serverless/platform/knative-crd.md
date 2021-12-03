---
title: "Knative Crd"
linkTitle: "Knative Crd"
type: docs
date: 2021-12-01T15:43:22+08:00
draft: false
---

TL；DR

本文主要介绍knative中的CRD，及以一个CRD为例介绍，只创建了ksvc其他所有的衍生CRD是如何被创建的。最后以一个实际运行的knative service为例展示了各个实际CRD的案例内容。

### 前置知识

在阅读本部分内容时，你应该已经对knative有了一个初步的了解和感觉，知道它是干什么的。另外可能需要你对k8s及自定义controller有一定的了解。如果你在阅读此部分内容有不知所云的感觉，你可以先了解下[kubernetes sample-controller](https://github.com/kubernetes/sample-controller).另外《kubernetes编程》应该也会对你在这方面的了解有所帮助。

### CRD总览

knative在创建一个service（ksvc）后，会自动创建一系列的crd。以下是创建一个ksvc所会创建的所有crd。

各个crd的基础功能在这里就不展开介绍了，只挑几个比较重点的说下：

kingress创建的目的是兼容不同网关的，网关需要监听此资源，当有kingress被创建后，需要解析此规则形成网关自己的路由规则。

revision是当前这个服务版本的一个核心CRD，一方面在于它创建了`deployment`和`podautoscalers`,并且`podautoscalers`中有ref字段指向此`deployment`。另一方面很多流程都是通过revision的Name或者Uid进行关联的。

public service就是k8s的一个service只不过没有定义label selector，这个svc也是在冷启动时流量会转到activator的核心。因为它在服务没有pod的时候会指向activator地址由activator在pod启动后转发，在服务被缩放pod启动后，会指向实际的pod。这样对这个服务的访问只要是通过这个svc，不论服务当前有没有pod，最终都能被导向到实际的pod中。

![image-20211201162120869](https://image-1255620078.cos.ap-nanjing.myqcloud.com/image-20211201162120869.png)



关于CRD的介绍在[深入探究 Knative 扩缩容的奥秘](https://mp.weixin.qq.com/s/O9LXnIQknW-vH_mwrkQltA)文章中有一个比较好的介绍。

### Controller介绍

controller是这些crd自动创建及状态维护的核心组件。[Knative Serving核心逻辑实现（上篇）](https://www.mdnice.com/writing/bb331f71193c4c28903b63209aeeb14b)这篇文章已经对代码结构做了一个非常好的概述，我在这里就不赘述了。

这里就只是简要说明一下，在`pkg/reconciler`几乎每个文件夹对应一种CRD资源。对资源的处理就在这个文件夹中。然后每种资源都会有个`ReconcileKind`方法资源被创建和更新的时候都会触发这个方法。

以ksvc为例，在`pkg/reconciler/service/service.go`文件中我们可以看到其`ReconcileKind`方法，这个方法主要就是创建其下层CRD。对于ksvc来说就是创建 `Configuration` 和 `Route` 资源。

```go
func (c *Reconciler) ReconcileKind(ctx context.Context, service *v1.Service) pkgreconciler.Event {
   ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
   defer cancel()

   logger := logging.FromContext(ctx)

   config, err := c.config(ctx, service)
   if err != nil {
      return err
   }

   if config.Generation != config.Status.ObservedGeneration {
      // The Configuration hasn't yet reconciled our latest changes to
      // its desired state, so its conditions are outdated.
      service.Status.MarkConfigurationNotReconciled()

      // If BYO-Revision name is used we must serialize reconciling the Configuration
      // and Route. Wait for observed generation to match before continuing.
      if config.Spec.GetTemplate().Name != "" {
         return nil
      }
   } else {
      logger.Debugf("Configuration Conditions = %#v", config.Status.Conditions)
      // Update our Status based on the state of our underlying Configuration.
      service.Status.PropagateConfigurationStatus(&config.Status)
   }

   // When the Configuration names a Revision, check that the named Revision is owned
   // by our Configuration and matches its generation before reprogramming the Route,
   // otherwise a bad patch could lead to folks inadvertently routing traffic to a
   // pre-existing Revision (possibly for another Configuration).
   if err := CheckNameAvailability(config, c.revisionLister); err != nil &&
      !apierrs.IsNotFound(err) {
      service.Status.MarkRevisionNameTaken(config.Spec.GetTemplate().Name)
      return nil
   }

   route, err := c.route(ctx, service)
   if err != nil {
      return err
   }

   // Update our Status based on the state of our underlying Route.
   ss := &service.Status
   if route.Generation != route.Status.ObservedGeneration {
      // The Route hasn't yet reconciled our latest changes to
      // its desired state, so its conditions are outdated.
      ss.MarkRouteNotReconciled()
   } else {
      // Update our Status based on the state of our underlying Route.
      ss.PropagateRouteStatus(&route.Status)
   }

   c.checkRoutesNotReady(config, logger, route, service)
   return nil
}
```

创建configuration资源

```go
func (c *Reconciler) config(ctx context.Context, service *v1.Service) (*v1.Configuration, error) {
   recorder := controller.GetEventRecorder(ctx)
   configName := resourcenames.Configuration(service)
   config, err := c.configurationLister.Configurations(service.Namespace).Get(configName)
   if apierrs.IsNotFound(err) {
      config, err = c.createConfiguration(ctx, service)
      if err != nil {
         recorder.Eventf(service, corev1.EventTypeWarning, "CreationFailed", "Failed to create Configuration %q: %v", configName, err)
         return nil, fmt.Errorf("failed to create Configuration: %w", err)
      }
      recorder.Eventf(service, corev1.EventTypeNormal, "Created", "Created Configuration %q", configName)
   } else if err != nil {
      return nil, fmt.Errorf("failed to get Configuration: %w", err)
   } else if !metav1.IsControlledBy(config, service) {
      // Surface an error in the service's status,and return an error.
      service.Status.MarkConfigurationNotOwned(configName)
      return nil, fmt.Errorf("service: %q does not own configuration: %q", service.Name, configName)
   } else if config, err = c.reconcileConfiguration(ctx, service, config); err != nil {
      return nil, fmt.Errorf("failed to reconcile Configuration: %w", err)
   }
   return config, nil
}
```

创建route资源

```go
func (c *Reconciler) route(ctx context.Context, service *v1.Service) (*v1.Route, error) {
   recorder := controller.GetEventRecorder(ctx)
   routeName := resourcenames.Route(service)
   route, err := c.routeLister.Routes(service.Namespace).Get(routeName)
   if apierrs.IsNotFound(err) {
      route, err = c.createRoute(ctx, service)
      if err != nil {
         recorder.Eventf(service, corev1.EventTypeWarning, "CreationFailed", "Failed to create Route %q: %v", routeName, err)
         return nil, fmt.Errorf("failed to create Route: %w", err)
      }
      recorder.Eventf(service, corev1.EventTypeNormal, "Created", "Created Route %q", routeName)
   } else if err != nil {
      return nil, fmt.Errorf("failed to get Route: %w", err)
   } else if !metav1.IsControlledBy(route, service) {
      // Surface an error in the service's status, and return an error.
      service.Status.MarkRouteNotOwned(routeName)
      return nil, fmt.Errorf("service: %q does not own route: %q", service.Name, routeName)
   } else if route, err = c.reconcileRoute(ctx, service, route); err != nil {
      return nil, fmt.Errorf("failed to reconcile Route: %w", err)
   }
   return route, nil
}
```

### webhook介绍

webhook的作用就是对CRD的个项内容进行合规性检查。这里先略过对具体执行代码的介绍，最终对各个crd合规性的校验在`pkg/apis/serving`文件夹下，有3个版本文件夹，以v1为例，每个CRD的校验逻辑都在xxx_validate.go文件中。以ksvc为例，其文件中有对Service crd和ServiceSpec的合法校验，当创建的crd校验不通过时，就会拒绝这个crd的创建。

```go
// Validate makes sure that Service is properly configured.
func (s *Service) Validate(ctx context.Context) (errs *apis.FieldError) {
   // If we are in a status sub resource update, the metadata and spec cannot change.
   // So, to avoid rejecting controller status updates due to validations that may
   // have changed (i.e. due to config-defaults changes), we elide the metadata and
   // spec validation.
   if !apis.IsInStatusUpdate(ctx) {
      errs = errs.Also(serving.ValidateObjectMetadata(ctx, s.GetObjectMeta(), false))
      errs = errs.Also(s.validateLabels().ViaField("labels"))
      errs = errs.Also(serving.ValidateRolloutDurationAnnotation(
         s.GetAnnotations()).ViaField("annotations"))
      errs = errs.ViaField("metadata")

      ctx = apis.WithinParent(ctx, s.ObjectMeta)
      errs = errs.Also(s.Spec.Validate(apis.WithinSpec(ctx)).ViaField("spec"))
   }

   if apis.IsInUpdate(ctx) {
      original := apis.GetBaseline(ctx).(*Service)
      errs = errs.Also(
         apis.ValidateCreatorAndModifier(
            original.Spec, s.Spec, original.GetAnnotations(),
            s.GetAnnotations(), serving.GroupName).ViaField("metadata.annotations"))
      errs = errs.Also(
         s.Spec.ConfigurationSpec.Template.VerifyNameChange(ctx,
            &original.Spec.ConfigurationSpec.Template).ViaField("spec.template"))
   }
   return errs
}

// Validate implements apis.Validatable
func (ss *ServiceSpec) Validate(ctx context.Context) *apis.FieldError {
   return ss.ConfigurationSpec.Validate(ctx).Also(
      // Within the context of Service, the RouteSpec has a default
      // configurationName.
      ss.RouteSpec.Validate(WithDefaultConfigurationName(ctx)))
}

// validateLabels function validates service labels
func (s *Service) validateLabels() (errs *apis.FieldError) {
   if val, ok := s.Labels[network.VisibilityLabelKey]; ok {
      errs = errs.Also(validateClusterVisibilityLabel(val))
   }
   return errs
}
```

### 附CRD示例

#### kservice

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"serving.knative.dev/v1","kind":"Service","metadata":{"annotations":{},"name":"hello","namespace":"default"},"spec":{"template":{"spec":{"containers":[{"env":[{"name":"TARGET","value":"Knative"}],"image":"gcr.oneitfarm.com/knative-samples/helloworld-go","ports":[{"containerPort":8080}]}]}}}}
  creationTimestamp: "2021-11-12T06:33:11Z"
  generation: 1
  name: hello
  namespace: default
  resourceVersion: "1400935"
  uid: d97bd364-1b92-4682-b568-cffd4b85054f
spec:
  template:
    spec:
      containerConcurrency: 0
      containers:
      - env:
        - name: TARGET
          value: Knative
        image: gcr.oneitfarm.com/knative-samples/helloworld-go
        name: user-container
        ports:
        - containerPort: 8080
          protocol: TCP
        readinessProbe:
          successThreshold: 1
          tcpSocket:
            port: 0
        resources: {}
      enableServiceLinks: false
      timeoutSeconds: 300
  traffic:
  - latestRevision: true
    percent: 100
status:
  address:
    url: http://hello.default.svc.cluster.local
  latestCreatedRevisionName: hello-00001
  latestReadyRevisionName: hello-00001
  observedGeneration: 1
  traffic:
  - latestRevision: true
    percent: 100
    revisionName: hello-00001
  url: http://hello.default.10.103.31.251.nip.io
```

#### service private

注意private的service具有selector，永远指向对应的deployment的所有pod。在没有pod的时候这个svc就不会对应到任何地址上。

```yaml
selector:
    serving.knative.dev/revisionUID: 357a23d7-24e2-4bb3-8af6-40e23122548f
```

完整内容：

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    autoscaling.knative.dev/class: kpa.autoscaling.knative.dev
    serving.knative.dev/creator: minikube-user
  creationTimestamp: "2021-11-12T06:59:31Z"
  labels:
    app: hello-00001
    networking.internal.knative.dev/serverlessservice: hello-00001
    networking.internal.knative.dev/serviceType: Private
    serving.knative.dev/configuration: hello
    serving.knative.dev/configurationGeneration: "1"
    serving.knative.dev/configurationUID: 5150d10a-8d42-464b-95be-2f38b110e863
    serving.knative.dev/revision: hello-00001
    serving.knative.dev/revisionUID: 357a23d7-24e2-4bb3-8af6-40e23122548f
    serving.knative.dev/service: hello
    serving.knative.dev/serviceUID: d97bd364-1b92-4682-b568-cffd4b85054f
  name: hello-00001-private
  namespace: default
  ownerReferences:
  - apiVersion: networking.internal.knative.dev/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: ServerlessService
    name: hello-00001
    uid: cb0495e0-f248-460f-9e5c-ecdd5af12b24
  resourceVersion: "1397163"
  uid: cbd1cb73-6e29-46a1-b532-96659cddfffd
spec:
  clusterIP: 10.101.43.6
  clusterIPs:
  - 10.101.43.6
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8012
  - name: http-autometric
    port: 9090
    protocol: TCP
    targetPort: http-autometric
  - name: http-usermetric
    port: 9091
    protocol: TCP
    targetPort: http-usermetric
  - name: http-queueadm
    port: 8022
    protocol: TCP
    targetPort: 8022
  - name: http-istio
    port: 8012
    protocol: TCP
    targetPort: 8012
  selector:
    serving.knative.dev/revisionUID: 357a23d7-24e2-4bb3-8af6-40e23122548f
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

#### configuration

```yaml
apiVersion: serving.knative.dev/v1
kind: Configuration
metadata:
  annotations:
    serving.knative.dev/routes: hello
  creationTimestamp: "2021-11-12T06:40:53Z"
  generation: 1
  labels:
    serving.knative.dev/service: hello
    serving.knative.dev/serviceUID: d97bd364-1b92-4682-b568-cffd4b85054f
  name: hello
  namespace: default
  ownerReferences:
  - apiVersion: serving.knative.dev/v1
    blockOwnerDeletion: true
    controller: true
    kind: Service
    name: hello
    uid: d97bd364-1b92-4682-b568-cffd4b85054f
  resourceVersion: "1400663"
  uid: 5150d10a-8d42-464b-95be-2f38b110e863
spec:
  template:
    metadata:
      creationTimestamp: null
    spec:
      containerConcurrency: 0
      containers:
      - env:
        - name: TARGET
          value: Knative
        image: gcr.oneitfarm.com/knative-samples/helloworld-go
        name: user-container
        ports:
        - containerPort: 8080
          protocol: TCP
        readinessProbe:
          successThreshold: 1
          tcpSocket:
            port: 0
status:
  conditions:
  - lastTransitionTime: "2021-11-12T07:09:12Z"
    status: "True"
    type: Ready
  latestCreatedRevisionName: hello-00001
  latestReadyRevisionName: hello-00001
  observedGeneration: 1
```

#### route

```yaml
apiVersion: serving.knative.dev/v1
kind: Route
metadata:
  annotations:
    serving.knative.dev/creator: minikube-user
    serving.knative.dev/lastModifier: minikube-user
  creationTimestamp: "2021-11-12T06:41:02Z"
  finalizers:
  - routes.serving.knative.dev
  generation: 1
  labels:
    serving.knative.dev/service: hello
  name: hello
  namespace: default
  ownerReferences:
  - apiVersion: serving.knative.dev/v1
    blockOwnerDeletion: true
    controller: true
    kind: Service
    name: hello
    uid: d97bd364-1b92-4682-b568-cffd4b85054f
  resourceVersion: "1400934"
  uid: 2da2c785-a4ab-498f-b9d8-32b9898d711f
spec:
  traffic:
  - configurationName: hello
    latestRevision: true
    percent: 100
status:
  address:
    url: http://hello.default.svc.cluster.local
  observedGeneration: 1
  traffic:
  - latestRevision: true
    percent: 100
    revisionName: hello-00001
  url: http://hello.default.10.103.31.251.nip.io
```

#### kingress

```yaml
apiVersion: networking.internal.knative.dev/v1alpha1
kind: Ingress
metadata:
  annotations:
    networking.internal.knative.dev/rollout: '{"configurations":[{"configurationName":"hello","percent":100,"revisions":[{"revisionName":"hello-00001","percent":100}],"stepParams":{}}]}'
    networking.knative.dev/ingress.class: kourier.ingress.networking.knative.dev
    serving.knative.dev/creator: minikube-user
    serving.knative.dev/lastModifier: minikube-user
  creationTimestamp: "2021-11-12T07:09:12Z"
  finalizers:
  - ingresses.networking.internal.knative.dev
  generation: 1
  labels:
    serving.knative.dev/route: hello
    serving.knative.dev/routeNamespace: default
    serving.knative.dev/service: hello
  name: hello
  namespace: default
  ownerReferences:
  - apiVersion: serving.knative.dev/v1
    blockOwnerDeletion: true
    controller: true
    kind: Route
    name: hello
    uid: 2da2c785-a4ab-498f-b9d8-32b9898d711f
  resourceVersion: "1400686"
  uid: b602b482-0d15-46f8-b4de-8016a75d13ce
spec:
  httpOption: Enabled
  rules:
  - hosts:
    - hello.default
    - hello.default.svc
    - hello.default.svc.cluster.local
    http:
      paths:
      - splits:
        - appendHeaders:
            Knative-Serving-Namespace: default
            Knative-Serving-Revision: hello-00001
          percent: 100
          serviceName: hello-00001
          serviceNamespace: default
          servicePort: 80
    visibility: ClusterLocal
  - hosts:
    - hello.default.10.103.31.251.nip.io
    http:
      paths:
      - splits:
        - appendHeaders:
            Knative-Serving-Namespace: default
            Knative-Serving-Revision: hello-00001
          percent: 100
          serviceName: hello-00001
          serviceNamespace: default
          servicePort: 80
    visibility: ExternalIP
status:
  conditions:
  - lastTransitionTime: "2021-11-12T07:09:12Z"
    status: "True"
    type: LoadBalancerReady
  - lastTransitionTime: "2021-11-12T07:09:12Z"
    status: "True"
    type: NetworkConfigured
  - lastTransitionTime: "2021-11-12T07:09:12Z"
    status: "True"
    type: Ready
  observedGeneration: 1
  privateLoadBalancer:
    ingress:
    - domainInternal: kourier-internal.kourier-system.svc.cluster.local
  publicLoadBalancer:
    ingress:
    - domainInternal: kourier.kourier-system.svc.cluster.local
```

#### revision

label指向了configuration和service

annotions下有各个控制缩放的字段。

```yaml
apiVersion: serving.knative.dev/v1
kind: Revision
metadata:
  annotations:
    serving.knative.dev/routes: hello
  creationTimestamp: "2021-11-12T06:41:02Z"
  generation: 1
  labels:
    serving.knative.dev/configuration: hello
    serving.knative.dev/configurationGeneration: "1"
    serving.knative.dev/configurationUID: 5150d10a-8d42-464b-95be-2f38b110e863
    serving.knative.dev/routingState: active
    serving.knative.dev/service: hello
    serving.knative.dev/serviceUID: d97bd364-1b92-4682-b568-cffd4b85054f
  name: hello-00001
  namespace: default
  ownerReferences:
  - apiVersion: serving.knative.dev/v1
    blockOwnerDeletion: true
    controller: true
    kind: Configuration
    name: hello
    uid: 5150d10a-8d42-464b-95be-2f38b110e863
  resourceVersion: "1433082"
  uid: 357a23d7-24e2-4bb3-8af6-40e23122548f
spec:
  containerConcurrency: 0
  containers:
  - env:
    - name: TARGET
      value: Knative
    image: gcr.oneitfarm.com/knative-samples/helloworld-go
    name: user-container
    ports:
    - containerPort: 8080
      protocol: TCP
    readinessProbe:
      successThreshold: 1
      tcpSocket:
        port: 0
    resources: {}
  enableServiceLinks: false
  timeoutSeconds: 300
status:
  actualReplicas: 0
  containerStatuses:
  - name: user-container
  desiredReplicas: 0
  observedGeneration: 1
```

#### deployment

label指向了configuration和revision和service，注意还注入了proxy容器。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2021-11-12T06:52:34Z"
  generation: 12
  labels:
    app: hello-00001
    serving.knative.dev/configuration: hello
    serving.knative.dev/configurationGeneration: "1"
    serving.knative.dev/configurationUID: 5150d10a-8d42-464b-95be-2f38b110e863
    serving.knative.dev/revision: hello-00001
    serving.knative.dev/revisionUID: 357a23d7-24e2-4bb3-8af6-40e23122548f
    serving.knative.dev/service: hello
    serving.knative.dev/serviceUID: d97bd364-1b92-4682-b568-cffd4b85054f
  name: hello-00001-deployment
  namespace: default
  ownerReferences:
  - apiVersion: serving.knative.dev/v1
    blockOwnerDeletion: true
    controller: true
    kind: Revision
    name: hello-00001
    uid: 357a23d7-24e2-4bb3-8af6-40e23122548f
  resourceVersion: "2999968"
  uid: 06001aab-6d40-4916-be05-0f2f8e6d392b
spec:
  progressDeadlineSeconds: 600
  replicas: 0
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      serving.knative.dev/revisionUID: 357a23d7-24e2-4bb3-8af6-40e23122548f
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: hello-00001
        serving.knative.dev/configuration: hello
        serving.knative.dev/configurationGeneration: "1"
        serving.knative.dev/configurationUID: 5150d10a-8d42-464b-95be-2f38b110e863
        serving.knative.dev/revision: hello-00001
        serving.knative.dev/revisionUID: 357a23d7-24e2-4bb3-8af6-40e23122548f
        serving.knative.dev/service: hello
        serving.knative.dev/serviceUID: d97bd364-1b92-4682-b568-cffd4b85054f
    spec:
      containers:
      - env:
        - name: TARGET
          value: Knative
        - name: PORT
          value: "8080"
        - name: K_REVISION
          value: hello-00001
        - name: K_CONFIGURATION
          value: hello
        - name: K_SERVICE
          value: hello
        image: gcr.oneitfarm.com/knative-samples/helloworld-go
        imagePullPolicy: Always
        lifecycle:
          preStop:
            httpGet:
              path: /wait-for-drain
              port: 8022
              scheme: HTTP
        name: user-container
        ports:
        - containerPort: 8080
          name: user-port
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: FallbackToLogsOnError
      - env:
        - name: SERVING_NAMESPACE
          value: default
        - name: SERVING_SERVICE
          value: hello
        - name: SERVING_CONFIGURATION
          value: hello
        - name: SERVING_REVISION
          value: hello-00001
        - name: QUEUE_SERVING_PORT
          value: "8012"
        - name: CONTAINER_CONCURRENCY
          value: "0"
        - name: REVISION_TIMEOUT_SECONDS
          value: "300"
        - name: SERVING_POD
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: SERVING_POD_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
        - name: SERVING_LOGGING_CONFIG
        - name: SERVING_LOGGING_LEVEL
        - name: SERVING_REQUEST_LOG_TEMPLATE
          value: '{"httpRequest": {"requestMethod": "{{.Request.Method}}", "requestUrl":
            "{{js .Request.RequestURI}}", "requestSize": "{{.Request.ContentLength}}",
            "status": {{.Response.Code}}, "responseSize": "{{.Response.Size}}", "userAgent":
            "{{js .Request.UserAgent}}", "remoteIp": "{{js .Request.RemoteAddr}}",
            "serverIp": "{{.Revision.PodIP}}", "referer": "{{js .Request.Referer}}",
            "latency": "{{.Response.Latency}}s", "protocol": "{{.Request.Proto}}"},
            "traceId": "{{index .Request.Header "X-B3-Traceid"}}"}'
        - name: SERVING_ENABLE_REQUEST_LOG
          value: "false"
        - name: SERVING_REQUEST_METRICS_BACKEND
          value: prometheus
        - name: TRACING_CONFIG_BACKEND
          value: none
        - name: TRACING_CONFIG_ZIPKIN_ENDPOINT
        - name: TRACING_CONFIG_DEBUG
          value: "false"
        - name: TRACING_CONFIG_SAMPLE_RATE
          value: "0.1"
        - name: USER_PORT
          value: "8080"
        - name: SYSTEM_NAMESPACE
          value: knative-serving
        - name: METRICS_DOMAIN
          value: knative.dev/internal/serving
        - name: SERVING_READINESS_PROBE
          value: '{"tcpSocket":{"port":8080,"host":"127.0.0.1"},"successThreshold":1}'
        - name: ENABLE_PROFILING
          value: "false"
        - name: SERVING_ENABLE_PROBE_REQUEST_LOG
          value: "false"
        - name: METRICS_COLLECTOR_ADDRESS
        - name: CONCURRENCY_STATE_ENDPOINT
        - name: CONCURRENCY_STATE_TOKEN_PATH
          value: /var/run/secrets/tokens/state-token
        - name: HOST_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.hostIP
        - name: ENABLE_HTTP2_AUTO_DETECTION
          value: "false"
        image: gcr.oneitfarm.com/knative-releases/knative.dev/serving/cmd/queue@sha256:5e63873df82ae864bfd5267a0ec7c3ef87c38a04c5ec894100fc4d5c48720569
        imagePullPolicy: IfNotPresent
        name: queue-proxy
        ports:
        - containerPort: 8022
          name: http-queueadm
          protocol: TCP
        - containerPort: 9090
          name: http-autometric
          protocol: TCP
        - containerPort: 9091
          name: http-usermetric
          protocol: TCP
        - containerPort: 8012
          name: queue-port
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            httpHeaders:
            - name: K-Network-Probe
              value: queue
            path: /
            port: 8012
            scheme: HTTP
          periodSeconds: 1
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          requests:
            cpu: 25m
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - all
          readOnlyRootFilesystem: true
          runAsNonRoot: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      enableServiceLinks: false
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 300
status:
  observedGeneration: 12
```

#### imagecache

```yaml
apiVersion: caching.internal.knative.dev/v1alpha1
kind: Image
metadata:
  annotations:
    serving.knative.dev/creator: minikube-user
  creationTimestamp: "2021-11-12T06:53:11Z"
  generation: 1
  labels:
    app: hello-00001
    serving.knative.dev/configuration: hello
    serving.knative.dev/configurationGeneration: "1"
    serving.knative.dev/configurationUID: 5150d10a-8d42-464b-95be-2f38b110e863
    serving.knative.dev/revision: hello-00001
    serving.knative.dev/revisionUID: 357a23d7-24e2-4bb3-8af6-40e23122548f
    serving.knative.dev/service: hello
    serving.knative.dev/serviceUID: d97bd364-1b92-4682-b568-cffd4b85054f
  name: hello-00001-cache-user-container
  namespace: default
  ownerReferences:
  - apiVersion: serving.knative.dev/v1
    blockOwnerDeletion: true
    controller: true
    kind: Revision
    name: hello-00001
    uid: 357a23d7-24e2-4bb3-8af6-40e23122548f
  resourceVersion: "1394796"
  uid: 0eb86961-74d2-4a23-bc21-24da31526b26
spec:
  image: ""
```

#### KPA

label指向了configuration、revision、service。scaleTargetRef指向了需要被缩放的目标deployment。

```yaml
apiVersion: autoscaling.internal.knative.dev/v1alpha1
kind: PodAutoscaler
metadata:
  annotations:
    autoscaling.knative.dev/class: kpa.autoscaling.knative.dev
    autoscaling.knative.dev/metric: concurrency
  creationTimestamp: "2021-11-12T06:59:30Z"
  generation: 6
  labels:
    app: hello-00001
    serving.knative.dev/configuration: hello
    serving.knative.dev/configurationGeneration: "1"
    serving.knative.dev/configurationUID: 5150d10a-8d42-464b-95be-2f38b110e863
    serving.knative.dev/revision: hello-00001
    serving.knative.dev/revisionUID: 357a23d7-24e2-4bb3-8af6-40e23122548f
    serving.knative.dev/service: hello
    serving.knative.dev/serviceUID: d97bd364-1b92-4682-b568-cffd4b85054f
  name: hello-00001
  namespace: default
  ownerReferences:
  - apiVersion: serving.knative.dev/v1
    blockOwnerDeletion: true
    controller: true
    kind: Revision
    name: hello-00001
    uid: 357a23d7-24e2-4bb3-8af6-40e23122548f
  resourceVersion: "2999966"
  uid: a630c17f-600f-44a3-ab44-f6e4213ac229
spec:
  protocolType: http1
  reachability: Reachable
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hello-00001-deployment
status:
  actualScale: 0
  desiredScale: 0
  metricsServiceName: hello-00001-private
  observedGeneration: 6
  serviceName: hello-00001
```

#### SKS

```yaml
apiVersion: v1
items:
- apiVersion: networking.internal.knative.dev/v1alpha1
  kind: ServerlessService
  metadata:
    annotations:
      autoscaling.knative.dev/class: kpa.autoscaling.knative.dev
      serving.knative.dev/creator: minikube-user
    creationTimestamp: "2021-11-12T06:59:30Z"
    generation: 13
    labels:
      app: hello-00001
      serving.knative.dev/configuration: hello
      serving.knative.dev/configurationGeneration: "1"
      serving.knative.dev/configurationUID: 5150d10a-8d42-464b-95be-2f38b110e863
      serving.knative.dev/revision: hello-00001
      serving.knative.dev/revisionUID: 357a23d7-24e2-4bb3-8af6-40e23122548f
      serving.knative.dev/service: hello
      serving.knative.dev/serviceUID: d97bd364-1b92-4682-b568-cffd4b85054f
    name: hello-00001
    namespace: default
    ownerReferences:
    - apiVersion: autoscaling.internal.knative.dev/v1alpha1
      blockOwnerDeletion: true
      controller: true
      kind: PodAutoscaler
      name: hello-00001
      uid: a630c17f-600f-44a3-ab44-f6e4213ac229
    resourceVersion: "2999964"
    uid: cb0495e0-f248-460f-9e5c-ecdd5af12b24
  spec:
    mode: Proxy
    numActivators: 2
    objectRef:
      apiVersion: apps/v1
      kind: Deployment
      name: hello-00001-deployment
    protocolType: http1
  status:
    observedGeneration: 7
    privateServiceName: hello-00001-private
    serviceName: hello-00001
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

#### Service public

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    autoscaling.knative.dev/class: kpa.autoscaling.knative.dev
  creationTimestamp: "2021-11-12T06:59:31Z"
  labels:
    app: hello-00001
    networking.internal.knative.dev/serverlessservice: hello-00001
    networking.internal.knative.dev/serviceType: Public
    serving.knative.dev/configuration: hello
    serving.knative.dev/configurationGeneration: "1"
    serving.knative.dev/configurationUID: 5150d10a-8d42-464b-95be-2f38b110e863
    serving.knative.dev/revision: hello-00001
    serving.knative.dev/revisionUID: 357a23d7-24e2-4bb3-8af6-40e23122548f
    serving.knative.dev/service: hello
    serving.knative.dev/serviceUID: d97bd364-1b92-4682-b568-cffd4b85054f
  name: hello-00001
  namespace: default
  ownerReferences:
  - apiVersion: networking.internal.knative.dev/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: ServerlessService
    name: hello-00001
    uid: cb0495e0-f248-460f-9e5c-ecdd5af12b24
  resourceVersion: "1397165"
  uid: a1117e40-8a81-4700-b13e-53cd52bf55d5
spec:
  clusterIP: 10.109.208.245
  clusterIPs:
  - 10.109.208.245
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8012
  sessionAffinity: None
  type: ClusterIP
```

#### metrics

metrics是定义这个服务流量指标获取的一些参数。

```yaml
apiVersion: autoscaling.internal.knative.dev/v1alpha1
kind: Metric
metadata:
  annotations:
    autoscaling.knative.dev/class: kpa.autoscaling.knative.dev
    autoscaling.knative.dev/metric: concurrency
  creationTimestamp: "2021-11-12T07:08:59Z"
  generation: 1
  labels:
    app: hello-00001
    serving.knative.dev/configuration: hello
    serving.knative.dev/configurationGeneration: "1"
    serving.knative.dev/configurationUID: 5150d10a-8d42-464b-95be-2f38b110e863
    serving.knative.dev/revision: hello-00001
    serving.knative.dev/revisionUID: 357a23d7-24e2-4bb3-8af6-40e23122548f
    serving.knative.dev/service: hello
    serving.knative.dev/serviceUID: d97bd364-1b92-4682-b568-cffd4b85054f
  name: hello-00001
  namespace: default
  ownerReferences:
  - apiVersion: autoscaling.internal.knative.dev/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: PodAutoscaler
    name: hello-00001
    uid: a630c17f-600f-44a3-ab44-f6e4213ac229
  resourceVersion: "3138042"
  uid: 7d67484b-0d49-4c9e-83e3-8c967889a1e6
spec:
  panicWindow: 6000000000
  scrapeTarget: hello-00001-private
  stableWindow: 60000000000
status:
  conditions:
  - lastTransitionTime: "2021-11-15T13:11:17Z"
    status: "True"
    type: Ready
  observedGeneration: 1
```



参考：

流量机制探索https://juejin.cn/post/6844904084005191693

如何基于 Knative 开发 自定义controllerhttps://mp.weixin.qq.com/s/L2jt2m1SUjXTD2kuBqB4Ow

【超详细】深入探究 Knative 扩缩容的奥秘https://mp.weixin.qq.com/s/O9LXnIQknW-vH_mwrkQltA

【源码解析】Knative Serving核心逻辑实现（上篇）https://www.mdnice.com/writing/bb331f71193c4c28903b63209aeeb14b

Knative 全链路流量机制探索与揭秘https://www.infoq.cn/article/niwwtl3apwoz7uigdmsb

