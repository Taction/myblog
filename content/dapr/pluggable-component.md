---
title: "Pluggable Component"
linkTitle: "Pluggable Component"
type: docs
date: 2023-03-07T21:09:38+08:00
draft: false
---

本文主要从dapr运行时、SDK和部署3部分的源码来介绍dapr的Pluggable component

### dapr运行时

#### Pluggable 组件自动发现机制

Dapr在实例化一个组件的时候通过type+version来唯一确定一个组件的类型，那么如何向Dapr配置pluggable组件呢。Dapr在启动时会进行组件的自动发现，由于dapr与Pluggable 组件仅通过UDS进行通信，所以Dapr对配置的UDS文件夹路径进行扫描，查找所有的UDS，将其文件名作为组件类型。以下是具体代码。

`initPluggableComponents`是Pluggable 组件自动发现的入口，在initRuntime中被调用。

```go
// initPluggableComponents discover pluggable components and initialize with their respective registries.
func (a *DaprRuntime) initPluggableComponents() {
   if runtime.GOOS == "windows" {
      log.Debugf("the current OS does not support pluggable components feature, skipping initialization")
      return
   }
   if err := pluggable.Discover(a.ctx); err != nil {
      log.Errorf("could not initialize pluggable components %v", err)
   }
}
```

`Discover`函数从`componentsSocketPath`下面发现文件，根据文件mode判断是否是Unix domain socket。这也是dapr文档里提到为什么一定需要pluggable component先启动的原因。然后通过注册的`callbackFunc func(name string, dialer GRPCConnectionDialer)`向`DefaultRegistry`来注册对应component的初始化函数。其他内置的组件也会在初始化的时候注册初始化函数，因此在注册完成后运行时对组件的处理上就“一视同仁”了。

```go
// Discover discover the pluggable components and callback the service discovery with the given component name and grpc dialer.
func Discover(ctx context.Context) error {
   services, err := serviceDiscovery(func(socket string) (reflectServiceClient, func(), error) {
      conn, err := SocketDial(
         ctx,
         socket,
         grpc.WithBlock(),
      )
      if err != nil {
         return nil, nil, err
      }
      client := grpcreflect.NewClientV1Alpha(ctx, reflectpb.NewServerReflectionClient(conn))
      return client, reflectServiceConnectionCloser(conn, client), nil
   })
   if err != nil {
      return err
   }
	 // 这里就是把自己注册到DefaultRegistry，从而在解析对应类型的组件的时候，可以获取对应的工厂
   callback(services)
   return nil
}
```

`serviceDiscovery`执行具体的发现逻辑，遍历文件夹下所有文件，如果是uds就反射其实现的service，记录下来。后面会通过callback根据全局的service->注册函数来进行向全局工厂注册。

```go

// serviceDiscovery returns all available discovered pluggable components services.
// uses gRPC reflection package to list implemented services.
func serviceDiscovery(reflectClientFactory func(string) (reflectServiceClient, func(), error)) ([]service, error) {
	services := []service{}
	componentsSocketPath := GetSocketFolderPath()
	_, err := os.Stat(componentsSocketPath)

	if os.IsNotExist(err) { // not exists is the same as empty.
		return services, nil
	}

	log.Debugf("loading pluggable components under path %s", componentsSocketPath)
	if err != nil {
		return nil, err
	}

	files, err := os.ReadDir(componentsSocketPath)
	if err != nil {
		return nil, fmt.Errorf("could not list pluggable components unix sockets: %w", err)
	}

	for _, dirEntry := range files {
		if dirEntry.IsDir() { // skip dirs
			continue
		}

		f, err := dirEntry.Info()
		if err != nil {
			return nil, err
		}

		socket := filepath.Join(componentsSocketPath, f.Name())
		if !utils.IsSocket(f) {
			discoveryLog.Warnf("could not use socket for file %s", socket)
			continue
		}

    // dail server构造反射client
		refctClient, cleanup, err := reflectClientFactory(socket)
		if err != nil {
			return nil, err
		}
    // 关闭与server链接，排空GRPC反射数据流
		defer cleanup()

    // 列出服务端实现的所有service，包括反射service本身。例如对于pubsub来说，会包含以下两项`dapr.proto.components.v1.PubSub`,`grpc.reflection.v1alpha.ServerReflection`。
		serviceList, err := refctClient.ListServices()
		if err != nil {
			return nil, fmt.Errorf("unable to list services: %w", err)
		}
    // 将socket地址和额外的grpc选项闭包方式传入dialer方法，dialer将会接收`componentName`并且通过`WithStreamInterceptor`option将其自动加入到metadata中。
		dialer := socketDialer(socket, grpc.WithBlock(), grpc.FailOnNonTempDialError(true))

    // componentName就是componet在注册的时候提供的名称，在实例化这个component的时候需要以`type.componentName`例如`pubsub.my-component`
		componentName := removeExt(f.Name())
    // 这里根据这个socket实现的service注册到列表里。后面会通过callback根据实现的service进行注册。
		for _, svc := range serviceList {
			services = append(services, service{
				componentName: componentName,
				protoRef:      svc,
				dialer:        dialer,
			})
		}
	}
	log.Debugf("found %d pluggable component services", len(services)-1) // reflection api doesn't count.
	return services, nil
}
```

callback函数的作用是将组件注册到对应的组件种类下（pubsub、state、binding...）。Dapr中的组件类型格式是`组件种类.组件具体类型`例如`pubsub.redis`，不同的组件种类需要注册到不同的组件工厂中。这里就是通过一个全局的类型=>注册函数映射来调用实际的注册函数。

```go
// callback invoke callback function for each given service
func callback(services []service) {
	for _, service := range services {
		callback, ok := onServiceDiscovered[service.protoRef]
		if !ok { // ignoring unknown service
			continue
		}
		callback(service.componentName, service.dialer)
		log.Infof("pluggable component '%s' was successfully registered for '%s'", service.componentName, service.protoRef)
	}
}
```

所有支持Pluggable 组件的种类，都会在初始化的时候注册自己的种类的工厂注册函数。例如下面就是对于pubsub的注册。

```go
func init() {
   //nolint:nosnakecase
   pluggable.AddServiceDiscoveryCallback(proto.PubSub_ServiceDesc.ServiceName, func(name string, dialer pluggable.GRPCConnectionDialer) {
      DefaultRegistry.RegisterComponent(newGRPCPubSub(dialer), name)
   })
}
```



#### 加载

根据k8s还是standalone模式，初始化不同的loader，然后加载yaml文件，首先解析`typeInfo`判断Kind是否跟要解析的目标一致，针对此情况，就是`Component`。加载完时候过滤一下scop，去除掉不是自己app-id的。然后把comp丢进Runtime `pendingComponents`channel中，有一个协程在启动时异步进行处理。先加载机密存储组件，然后加载其余的组件。这里加载组件是不区分build-in还是pluggable的。

```go
func (a *DaprRuntime) loadComponents(opts *runtimeOpts) error {
   var loader components.ComponentLoader
	// 根据运行环境时k8s还是Standalone来使用不同的加载配置模式
   switch a.runtimeConfig.Mode {
   case modes.KubernetesMode:
      loader = components.NewKubernetesComponents(a.runtimeConfig.Kubernetes, a.namespace, a.operatorClient, a.podName)
   case modes.StandaloneMode:
      loader = components.NewStandaloneComponents(a.runtimeConfig.Standalone)
   default:
      return fmt.Errorf("components loader for mode %s not found", a.runtimeConfig.Mode)
   }

   log.Info("loading components")
   comps, err := loader.LoadComponents()
   if err != nil {
      return err
   }

   for _, comp := range comps {
      log.Debugf("found component. name: %s, type: %s/%s", comp.ObjectMeta.Name, comp.Spec.Type, comp.Spec.Version)
   }

   authorizedComps := a.getAuthorizedComponents(comps)

   a.componentsLock.Lock()
   a.components = authorizedComps
   a.componentsLock.Unlock()

   // Iterate through the list twice
   // First, we look for secret stores and load those, then all other components
   // Sure, we could sort the list of authorizedComps... but this is simpler and most certainly faster
  // 先处理机密存储，因为其他组件可能会依赖机密存储中的信息
   for _, comp := range authorizedComps {
      if strings.HasPrefix(comp.Spec.Type, string(components.CategorySecretStore)+".") {
         a.pendingComponents <- comp
      }
   }
   for _, comp := range authorizedComps {
      if !strings.HasPrefix(comp.Spec.Type, string(components.CategorySecretStore)+".") {
         a.pendingComponents <- comp
      }
   }

   return nil
}
```

##### 处理单个组件

这个函数的具体逻辑就是针对单个组件定义，通过工厂方法创建组件实例，然后根据类型存储在runtime中。这里面背后的详细逻辑比较多就不展开介绍了，会在后面的文章中进行介绍。

```go
func (a *DaprRuntime) processComponentAndDependents(comp componentsV1alpha1.Component) error {
   log.Debugf("loading component. name: %s, type: %s/%s", comp.ObjectMeta.Name, comp.Spec.Type, comp.Spec.Version)
   res := a.preprocessOneComponent(&comp)
   if res.unreadyDependency != "" {
      a.pendingComponentDependents[res.unreadyDependency] = append(a.pendingComponentDependents[res.unreadyDependency], comp)
      return nil
   }
	 // 解析组件类型，pubsub、state等，就是在注册到registry时.前面的部分。
   compCategory := a.extractComponentCategory(comp)
   if compCategory == "" {
      // the category entered is incorrect, return error
      return fmt.Errorf("incorrect type %s", comp.Spec.Type)
   }

   ch := make(chan error, 1)

   timeout, err := time.ParseDuration(comp.Spec.InitTimeout)
   if err != nil {
      timeout = defaultComponentInitTimeout
   }

   // 异步实际解析，好在超时的时候进行退出。这个函数就是分类型进行实际处理的。
   go func() {
      ch <- a.doProcessOneComponent(compCategory, comp)
   }()

   select {
   case err := <-ch:
      if err != nil {
         return err
      }
   case <-time.After(timeout):
      diag.DefaultMonitoring.ComponentInitFailed(comp.Spec.Type, "init", comp.ObjectMeta.Name)
      err := fmt.Errorf("init timeout for component %s exceeded after %s", comp.Name, timeout.String())
      return NewInitError(InitComponentFailure, comp.LogName(), err)
   }

   log.Infof("component loaded. name: %s, type: %s/%s", comp.ObjectMeta.Name, comp.Spec.Type, comp.Spec.Version)
   a.appendOrReplaceComponents(comp)
   diag.DefaultMonitoring.ComponentLoaded()

   dependency := componentDependency(compCategory, comp.Name)
   if deps, ok := a.pendingComponentDependents[dependency]; ok {
      delete(a.pendingComponentDependents, dependency)
      for _, dependent := range deps {
         if err := a.processComponentAndDependents(dependent); err != nil {
            return err
         }
      }
   }

   return nil
}
```

pubsub初始化

```go
func (a *DaprRuntime) initPubSub(c componentsV1alpha1.Component) error {
   fName := c.LogName()
   pubSub, err := a.pubSubRegistry.Create(c.Spec.Type, c.Spec.Version, fName)
   if err != nil {
      diag.DefaultMonitoring.ComponentInitFailed(c.Spec.Type, "creation", c.ObjectMeta.Name)
      return NewInitError(CreateComponentFailure, fName, err)
   }

   baseMetadata := a.toBaseMetadata(c)
   properties := baseMetadata.Properties
   consumerID := strings.TrimSpace(properties["consumerID"])
   if consumerID == "" {
      consumerID = a.runtimeConfig.ID
   }
   properties["consumerID"] = consumerID

   err = pubSub.Init(context.TODO(), pubsub.Metadata{Base: baseMetadata})
   if err != nil {
      diag.DefaultMonitoring.ComponentInitFailed(c.Spec.Type, "init", c.ObjectMeta.Name)
      return NewInitError(InitComponentFailure, fName, err)
   }

   pubsubName := c.ObjectMeta.Name

   a.pubSubs[pubsubName] = pubsubItem{
      component:           pubSub,
      scopedSubscriptions: scopes.GetScopedTopics(scopes.SubscriptionScopes, a.runtimeConfig.ID, properties),
      scopedPublishings:   scopes.GetScopedTopics(scopes.PublishingScopes, a.runtimeConfig.ID, properties),
      allowedTopics:       scopes.GetAllowedTopics(properties),
      namespaceScoped:     metadataContainsNamespace(c.Spec.Metadata),
   }
   diag.DefaultMonitoring.ComponentInitialized(c.Spec.Type)

   return nil
}
```

主要就是给logger加上component名称，然后调用工厂函数。

```go
func (p *Registry) getPubSub(name, version, logName string) (func() pubsub.PubSub, bool) {
   nameLower := strings.ToLower(name)
   versionLower := strings.ToLower(version)
   pubSubFn, ok := p.messageBuses[nameLower+"/"+versionLower]
   if ok {
      return p.wrapFn(pubSubFn, logName), true
   }
   if components.IsInitialVersion(versionLower) {
      pubSubFn, ok = p.messageBuses[nameLower]
      if ok {
         return p.wrapFn(pubSubFn, logName), true
      }
   }

   return nil, false
}
```

`newGRPCPubSub`最终返回的是`*grpcPubSub`这个结构体实现了`pubsub.PubSub`是对实际的Pluggable的一层包装。在init的时候实例化一个Pluggable component，包含三部分：

- features缓存，因为这个不会变，所以获取一次存起来就行了，减少网络交互
- GRPCConnector创建与pluggable的GRPC连接函数。这是在组件自动发现的时候注册进来的。
- logger日志实体类

```go
// newGRPCPubSub creates a new grpc pubsub for the given pluggable component.
func newGRPCPubSub(dialer pluggable.GRPCConnectionDialer) func(l logger.Logger) pubsub.PubSub {
   return func(l logger.Logger) pubsub.PubSub {
      return fromConnector(l, pluggable.NewGRPCConnectorWithDialer(dialer, proto.NewPubSubClient))
   }
}
// fromConnector creates a new GRPC pubsub using the given underlying connector.
func fromConnector(l logger.Logger, connector *pluggable.GRPCConnector[proto.PubSubClient]) *grpcPubSub {
   return &grpcPubSub{
      features:      make([]pubsub.Feature, 0),
      GRPCConnector: connector,
      logger:        l,
   }
}
```

在init的时候创建grpc client，然后调用component的init接口进行初始化。缓存了一下feature。

```go
// grpcPubSub is a implementation of a pubsub over a gRPC Protocol.
type grpcPubSub struct {
   *pluggable.GRPCConnector[proto.PubSubClient]
   // features the list of state store implemented features features.
   features []pubsub.Feature
   logger   logger.Logger
}

// Init initializes the grpc pubsub passing out the metadata to the grpc component.
// It also fetches and set the component features.
func (p *grpcPubSub) Init(ctx context.Context, metadata pubsub.Metadata) error {
   if err := p.Dial(metadata.Name); err != nil {
      return err
   }

   protoMetadata := &proto.MetadataRequest{
      Properties: metadata.Properties,
   }

   _, err := p.Client.Init(p.Context, &proto.PubSubInitRequest{
      Metadata: protoMetadata,
   })
   if err != nil {
      return err
   }

   // TODO Static data could be retrieved in another way, a necessary discussion should start soon.
   // we need to call the method here because features could return an error and the features interface doesn't support errors
   featureResponse, err := p.Client.Features(p.Context, &proto.FeaturesRequest{})
   if err != nil {
      return err
   }

   p.features = make([]pubsub.Feature, len(featureResponse.Features))
   for idx, f := range featureResponse.Features {
      p.features[idx] = pubsub.Feature(f)
   }

   return nil
}
```



#### dapr适配

pluggable组件与dapr通过grpc进行通信，dapr与组件之间通过interface来进行交互。所以需要一层适配层将grpc的client转化为dapr的组件interface，这针对不同的组件种类需要不同的实现。例如对于pubsub来说：

##### publish

```go
// Publish publishes data to a topic.
func (p *grpcPubSub) Publish(ctx context.Context, req *pubsub.PublishRequest) error {
	_, err := p.Client.Publish(ctx, &proto.PublishRequest{
		Topic:      req.Topic,
		PubsubName: req.PubsubName,
		Data:       req.Data,
		Metadata:   req.Metadata,
	})
	return err
}
```

##### subscribe

订阅消息针对每次调用，新增一个`PullMessages`的双向grpc stream，循环读取消息，处理消息（发送给app），处理完成后发送ack。

```go
// pullMessages pull messages of the given subscription and execute the handler for that messages.
func (p *grpcPubSub) pullMessages(ctx context.Context, topic *proto.Topic, handler pubsub.Handler) error {
	// first pull should be sync and subsequent connections can be made in background if necessary
	pull, err := p.Client.PullMessages(ctx)
	if err != nil {
		return fmt.Errorf("unable to subscribe: %w", err)
	}

	streamCtx, cancel := context.WithCancel(pull.Context())

	err = pull.Send(&proto.PullMessagesRequest{
		Topic: topic,
	})

	cleanup := func() {
		if closeErr := pull.CloseSend(); closeErr != nil {
			p.logger.Warnf("could not close pull stream of topic %s: %v", topic.Name, closeErr)
		}
		cancel()
	}

	if err != nil {
		cleanup()
		return fmt.Errorf("unable to subscribe: %w", err)
	}

	handle := p.adaptHandler(streamCtx, pull, handler)
	go func() {
		defer cleanup()
		for {
			msg, err := pull.Recv()
			if err == io.EOF { // no more messages
				return
			}

			// TODO reconnect on error
			if err != nil {
				p.logger.Errorf("failed to receive message: %v", err)
				return
			}

			p.logger.Debugf("received message from stream on topic %s", msg.TopicName)

			go handle(msg)
		}
	}()

	return nil
}
```

`adaptHandler`是对hander的一个封装，主要作用是为了控制ack的时候的并发问题。可以看到这里通过闭包的方式创建了处理函数，持有同一个锁来控制并发。PS：这里通过锁控制在高并发情况下**理论上**容易发生惊群效应，引起CPU的升高，如果在使用中观察到这个现象，改成通过channel会好一点。

```go
// adaptHandler returns a non-error function that handle the message with the given handler and ack when returns.
//
//nolint:nosnakecase
func (p *grpcPubSub) adaptHandler(ctx context.Context, streamingPull proto.PubSub_PullMessagesClient, handler pubsub.Handler) messageHandler {
   safeSend := &sync.Mutex{}
   return func(msg *proto.PullMessagesResponse) {
      m := pubsub.NewMessage{
         Data:        msg.Data,
         ContentType: &msg.ContentType,
         Topic:       msg.TopicName,
         Metadata:    msg.Metadata,
      }
      var ackError *proto.AckMessageError

      if err := handler(ctx, &m); err != nil {
         p.logger.Errorf("error when handling message on topic %s", msg.TopicName)
         ackError = &proto.AckMessageError{
            Message: err.Error(),
         }
      }

      // As per documentation:
      // When using streams,
      // one must take care to avoid calling either SendMsg or RecvMsg multiple times against the same Stream from different goroutines.
      // In other words, it's safe to have a goroutine calling SendMsg and another goroutine calling RecvMsg on the same stream at the same time.
      // But it is not safe to call SendMsg on the same stream in different goroutines, or to call RecvMsg on the same stream in different goroutines.
      // https://github.com/grpc/grpc-go/blob/master/Documentation/concurrency.md#streams
      safeSend.Lock()
      defer safeSend.Unlock()

      if err := streamingPull.Send(&proto.PullMessagesRequest{
         AckMessageId: msg.Id,
         AckError:     ackError,
      }); err != nil {
         p.logger.Errorf("error when ack'ing message %s from topic %s", msg.Id, msg.TopicName)
      }
   }
}
```



### SDK

##### example及初始化

对于SDK来说，可以注册运行多个pluggable componet实例，以名字作为唯一区分。注册也很简单，返回一个实现了对应component接口的实例就可以。例如以pubsub为例：

```go
func main() {
	dapr.Register("my-component-redis", dapr.WithPubSub(func() pubsub.PubSub {
		return &RedisComponent{}
	}))
	dapr.MustRun()
}
```

WithPubSub factory方法返回的是一个实现了contrib中定义的pubsub.PubSub接口的实例，这也是build-in的组件实现的接口。`pubsub.Register`会在后面介绍，主要就是将pubsub的实现与grpc server进行关联。mux是一个wrapper，主要为了用户在一个pluggable类型下定义了多个componet的情况进行隔离，每个有自己的实例。其逻辑比较简单，就是根据metadata中带的`*InstanceID*`来加载对应实例（上文说过这里会在runtime自动注入）这部分会纵向拉出来在后文中详细介绍。

```go
func WithPubSub(factory func() pubsub.PubSub) option {
   return func(cf *componentsOpts) {
      cf.useGrpcServer = append(cf.useGrpcServer, func(s *grpc.Server) {
         pubsub.Register(s, mux(factory))
      })
   }
}
```

我们看下MustRun执行了哪些操作。这里会调用Run方法，遇到错误就退出。

```go
// MustRun same as run but panics on error
func MustRun() {
	if err := Run(); err != nil {
		panic(err)
	}
}
```

Run根据环境变量获取socket文件夹，创建sock文件。每有一个withXXX就会创建一个，使用其名称作为文件名。然后针对每个注册的component调用runComponent来运行。

```go
// Run starts the component server.
func Run() error {
	socketFolder, ok := os.LookupEnv(unixSocketFolderPathEnvVar)
	if !ok {
		socketFolder, ok = os.LookupEnv(fallbackUnixSocketFolderPathEnvVar)
		if !ok {
			socketFolder = defaultSocketFolder
		}
	}
	if len(factories) == 0 {
		return ErrNoComponentsRegistered
	}
	done := make(chan struct{}, len(factories))
	abort := makeAbortChan(done)
	var cleanupGroup sync.WaitGroup

	for component := range factories {
		socket := filepath.Join(socketFolder, component+".sock")
		cleanupGroup.Add(1)
		go func(opts *componentsOpts) {
			err := runComponent(socket, opts, abort, &cleanupGroup)
			if err != nil {
				svcLogger.Errorf("aborting due to an error %v", err)
				done <- struct{}{}
			}
		}(factories[component])
	}

	select {
	case <-done:
		cleanupGroup.Wait()
		return nil
	case <-abort:
		cleanupGroup.Wait()
		return nil
	}
}
```

runComponent就是正常启动grpc server，然后会为其注册反射服务。

```go

func runComponent(socket string, opts *componentsOpts, abortChan chan struct{}, onFinish *sync.WaitGroup) error {
	// remove socket if it is already created.
	if err := os.Remove(socket); err != nil && !os.IsNotExist(err) {
		return err
	}

	svcLogger.Infof("using socket defined at '%s'", socket)

	lis, err := net.Listen("unix", socket)
	if err != nil {
		return err
	}

	defer lis.Close()

	server := grpc.NewServer()

	if err = opts.apply(server); err != nil {
		return err
	}
	go func() {
		defer onFinish.Done()
		<-abortChan
		lis.Close()
	}()
	// 将 grpc.Server 注册到反射服务中，这样在启动 gprc 反射服务后，那么就可以通过 reflection 包提供的反射服务查询 gRPC 服务及调用 gRPC 方法。
	reflection.Register(server)
	return server.Serve(lis)
}
```

#### pubsub

##### 包装

在上文中`example及初始化`部分略过了Register实际的处理。可以看到这里对传入的pubsub实例进行了代理包装，实现了GRPC server对应的方法，然后在对应的方法中获取组件实例（上面的mux方法），并调用实例对应的实际实现的逻辑方法。

```go
// Register the pubsub implementation for the component gRPC service.
func Register(server *grpc.Server, getInstance func(context.Context) PubSub) {
   pubsub := &pubsub{
      getInstance: getInstance,
   }
   proto.RegisterPubSubServer(server, pubsub)
}

type pubsub struct {
	proto.UnimplementedPubSubServer
	getInstance func(context.Context) PubSub
}
```

##### Init

```go
func (s *pubsub) Init(ctx context.Context, initReq *proto.PubSubInitRequest) (*proto.PubSubInitResponse, error) {
   return &proto.PubSubInitResponse{}, s.getInstance(ctx).Init(contribPubSub.Metadata{
      Base: contribMetadata.Base{Properties: initReq.Metadata.Properties},
   })
}
```

##### Ping

```go
func (s *pubsub) Ping(context.Context, *proto.PingRequest) (*proto.PingResponse, error) {
   return &proto.PingResponse{}, nil
}
```

##### Features

```go
func (s *pubsub) Features(ctx context.Context, _ *proto.FeaturesRequest) (*proto.FeaturesResponse, error) {
   features := &proto.FeaturesResponse{
      Features: internal.Map(s.getInstance(ctx).Features(), func(f contribPubSub.Feature) string {
         return string(f)
      }),
   }

   return features, nil
}
```

##### Publish

```go
func (s *pubsub) Publish(ctx context.Context, req *proto.PublishRequest) (*proto.PublishResponse, error) {
   return &proto.PublishResponse{}, s.getInstance(ctx).Publish(&contribPubSub.PublishRequest{
      Data:        req.Data,
      PubsubName:  req.PubsubName,
      Topic:       req.Topic,
      Metadata:    req.Metadata,
      ContentType: &req.ContentType,
   })
}
```

##### PullMessages

 从客户端接收消息拉取请求，获取客户端传入的topic，订阅topic，将消息发到客户端处理，并进行ack。

```go
func (s *pubsub) PullMessages(stream proto.PubSub_PullMessagesServer) error {
   // 第一条消息是要订阅的topic，然后component开始发送，后面的收到的就都是ack的消息了。
   fstMessage, err := stream.Recv()
   if err != nil {
      return err
   }

  // 拿到topic
   topic := fstMessage.GetTopic()

   if topic == nil {
      return ErrTopicNotSpecified
   }

   ctx, cancel := context.WithCancel(stream.Context())
   defer cancel()

  // 这里是对handler和ack循环的封装
   handler, startAckLoop := pullFor(stream)

   err = s.getInstance(ctx).Subscribe(ctx, contribPubSub.SubscribeRequest{
      Topic:    topic.Name,
      Metadata: topic.Metadata,
   }, handler)

   if err != nil {
      return err
   }

   return startAckLoop()
}
```

pullFor包含消息处理整体逻辑，就是调用handler往daprd发送，然后根据返回是否成功进行ack或者其他操作。

```go
// pullFor creates a message handler for the given stream.
func pullFor(stream proto.PubSub_PullMessagesServer) (pubsubHandler contribPubSub.Handler, acknLoop func() error) {
  // thread safe stream就是对收发消息加锁避免并发问题
   tfStream := internal.NewGRPCThreadSafeStream[proto.PullMessagesResponse, proto.PullMessagesRequest](stream)
  // ack管理助手
   ackManager := internal.NewAckManager[error]()
  // handler是业务处理wrapper，发出消息并等待消息的ack。
   return handler(tfStream, ackManager), func() error {
     // 循环读取dapr运行时发回的ack信息，将ack信息发送到对应channel中。
      return ackLoop(stream.Context(), tfStream, ackManager)
   }
}
```

此函数就是单个消息处理函数，发消息并等待异步ack，

```go
// handler build a pubsub handler using the given threadsafe stream and the ack manager.
func handler(tfStream internal.ThreadSafeStream[proto.PullMessagesResponse, proto.PullMessagesRequest], ackManager *internal.AcknowledgementManager[error]) contribPubSub.Handler {
   return func(ctx context.Context, contribMsg *contribPubSub.NewMessage) error {
     // 这里获取一个唯一msgID作为此次消息处理标识，在ackManager那里针对此标识创建pendingAck channel
      msgID, pendingAck, cleanup := ackManager.Get()
      defer cleanup()

      msg := &proto.PullMessagesResponse{
         Data:        contribMsg.Data,
         TopicName:   contribMsg.Topic,
         Metadata:    contribMsg.Metadata,
         ContentType: internal.ZeroValueIfNil(contribMsg.ContentType),
         Id:          msgID,
      }

      // in case of message can't be sent it does not mean that the sidecar didn't receive the message
      // it only means that the component wasn't able to receive the response back
      // it could leads in messages being acknowledged without even being pending first
      // we should ignore this since it will be probably retried by the underlying component.
     // 发送到dapr runtime，这里的发送是加锁避免并发的
      err := tfStream.Send(msg)
      if err != nil {
         return errors.Wrapf(err, "error when sending message %s to consumer on topic %s", msg.Id, msg.TopicName)
      }

      select {
        // 这里的err如果不出错的情况下是nil
      case err := <-pendingAck:
         return err
        // 这里说明如果要对单条消息做超时控制的话，要在调用handler的时候通过context控制。
      case <-ctx.Done():
         return ErrAckTimeout
      }
   }
}
```

这里获取一个唯一msgID作为此次消息处理标识，针对此标识创建pendingAck channel，以及返回清理函数。

```go
func (m *AcknowledgementManager[TAckResult]) Get() (messageID string, ackChan chan TAckResult, cleanup func()) {
   m.mu.Lock()
   defer m.mu.Unlock()
   msgID := uuid.New().String()

   ackChan = make(chan TAckResult, 1)
   m.pendingAcks[msgID] = ackChan

   return msgID, ackChan, func() {
      m.mu.Lock()
      delete(m.pendingAcks, msgID)
      close(ackChan)
      m.mu.Unlock()
   }
}
```

这个就是获取dapr runtime发过来的ack信息，然后发送到对应的channel上。

```go
// ackLoop starts an active ack loop reciving acks from client.
func ackLoop(streamCtx context.Context, tfStream internal.ThreadSafeStream[proto.PullMessagesResponse, proto.PullMessagesRequest], ackManager *internal.AcknowledgementManager[error]) error {
   for {
      if streamCtx.Err() != nil {
         return streamCtx.Err()
      }
      ack, err := tfStream.Recv()
      if err == io.EOF {
         return nil
      }

      if err != nil {
         return err
      }

      var ackError error

      if ack.AckError != nil {
         ackError = errors.New(ack.AckError.Message)
      }
     // 这里注意ackError可能是nil
      if err := ackManager.Ack(ack.AckMessageId, ackError); err != nil {
         pubsubLogger.Warnf("error %v when trying to notify ack", err)
      }
   }
}
```

就是在调用的时候尝试往对应的channel上发送消息，select中有两个注意点，一个在注释中已经说了，因为channel是有缓冲的，所以如果发消息发不进去的话首先一定是重复了，另外channel被关闭或者没有接收端在处理消息。所以这个地方的代码可以优化一下如果发不进去直接进default执行空动作就可以了。这里还有一个微小的风险是这里的c有在执行到的时候有被关闭的可能，虽然ack是顺序的，但是在Get函数中执行clean的时候会关闭channel。如果这里解锁之后发生切换恰巧执行超时ctx取消channel关闭，这里就可能panic。

```go
func (m *AcknowledgementManager[TAckResult]) Ack(messageID string, result TAckResult) error {
   m.mu.RLock()
   c, ok := m.pendingAcks[messageID]
   m.mu.RUnlock()

   if !ok {
      return fmt.Errorf("message %s not found or not specified", messageID)
   }

   select {
   // wait time for outstanding acks
   // that should be instantaneous as the channel is bufferized size 1
   // if this operation takes longer than the waitFunc (defaults to 1s)
   // it probably means that no consumer is waiting for the message ack
   // or it could be duplicated ack for the same message.
   case <-m.ackTimeoutFunc():
      return nil
   case c <- result:
      return nil
   }
}
```

##### 为什么包装类是newInstance函数而不是instance实例

在源码中我们可以看到pubsub的wrapper结构体中并不是一个pubsub的结构体而是创建一个pubsub的方法。在注册pubsub的时候也是传入的创建pubsub的工厂方法。而在proto rpc方法被调用时会通过`s.getInstance(ctx)`来获取实例，如果工厂类没有经过任何封装的话，那么每次都是一个新的实例。相关函数如下：

```go
// 
type pubsub struct {
	proto.UnimplementedPubSubServer
	getInstance func(context.Context) PubSub
}
// 这里注册时传入的工厂方法经过了mux的封装
func WithPubSub(factory func() pubsub.PubSub) option {
   return func(cf *componentsOpts) {
      cf.useGrpcServer = append(cf.useGrpcServer, func(s *grpc.Server) {
         pubsub.Register(s, mux(factory))
      })
   }
}
// 每次方法调用的时候都通过getInstance来获取实例。
func (s *pubsub) Init(ctx context.Context, initReq *proto.PubSubInitRequest) (*proto.PubSubInitResponse, error) {
   return &proto.PubSubInitResponse{}, s.getInstance(ctx).Init(contribPubSub.Metadata{
      Base: contribMetadata.Base{Properties: initReq.Metadata.Properties},
   })
}
```

那么就是在`mux`方法中缓存了实例，通过方法源码我们可以看到，通过map持有多个实例，通过锁来控制instance单例的创建。这里之所以不是用一个实例，而是通过mux包装后的方法来获取实例，是为了为不同的component instance创建不同的实例。

```go
func mux[TComponent any](new func() TComponent) func(context.Context) TComponent {
   instances := sync.Map{}
   firstLoad := sync.Mutex{}
   return func(ctx context.Context) TComponent {
      instanceID := "#default__instance#"
      metadata, ok := metadata.FromIncomingContext(ctx)
      if ok {
         instanceIDs := metadata.Get(metadataInstanceID)
         if len(instanceIDs) != 0 {
            instanceID = instanceIDs[0]
         }
      }
      instance, ok := instances.Load(instanceID)
      if !ok {
         firstLoad.Lock()
         defer firstLoad.Unlock()
         instance, ok = instances.Load(instanceID) // double check lock
         if ok {
            return instance.(TComponent)
         }
         instance = new()
         instances.Store(instanceID, instance)
      }
      return instance.(TComponent)
   }
}
```

在什么情况下会发生这种情况呢？在为application配置component的时候有多个component都是此Pluggable component类型的时候。例如下图所示：

![image](https://user-images.githubusercontent.com/5839364/193684998-5d891e78-4b32-4784-8091-206af4c5cf94.png)

### Injector

在启动一个具有Pluggable component的服务的时候，需要确保Pluggable component和daprd挂载了相同的UDS 文件夹，以便可以相互访问。

##### 挂载

`dapr.io/unix-domain-socket-path`



```go
func (i *injector) getPodPatchOperations(ar *v1.AdmissionReview,
   namespace, image, imagePullPolicy string, kubeClient kubernetes.Interface, daprClient scheme.Interface,
) (patchOps []sidecar.PatchOperation, err error) {
   req := ar.Request
   // ... 格式数据检查
   // 为pod设置uds的volume定义，整合volumeMounts给sidecar用
   volumeMounts := sidecar.GetVolumeMounts(pod)
   socketVolumeMount := sidecar.GetUnixDomainSocketVolumeMount(&pod)
   if socketVolumeMount != nil {
      volumeMounts = append(volumeMounts, *socketVolumeMount)
   }
   volumeMounts = append(volumeMounts, corev1.VolumeMount{
      Name:      sidecar.TokenVolumeName,
      MountPath: sidecar.TokenVolumeKubernetesMountPath,
      ReadOnly:  true,
   })

   // App containers和 Pluggable区分开来,以及根据`dapr.io/inject-pluggable-components`注解进行自动注入。这两个函数都比较关键会在后面介绍。
   appContainers, componentContainers, injectedComponentContainers, err := i.splitContainers(pod)
	if err != nil {
		return nil, err
	}
	componentPatchOps, componentsSocketVolumeMount := components.PatchOps(componentContainers, injectedComponentContainers, &pod)


   // 生成sidecar容器，注入volumeMounts
   sidecarContainer, err := sidecar.GetSidecarContainer(sidecar.ContainerConfig{
      //...
      VolumeMounts:                 volumeMounts,
      ComponentsSocketsVolumeMount: componentsSocketVolumeMount,
      RunAsNonRoot:                 i.config.GetRunAsNonRoot(),
   })
   if err != nil {
      return nil, err
   }

   // Create the list of patch operations
   patchOps = []sidecar.PatchOperation{}

  // patch sidecar 容器
   patchOps = append(patchOps,
      sidecar.PatchOperation{
         Op:    "add",
         Path:  sidecar.PatchPathContainers + "/-",
         Value: sidecarContainer,
      },
      sidecar.AddDaprEnvVarsToContainers(appContainers)...)
  // 为app容器添加socket volume mount
   patchOps = append(patchOps,
      sidecar.AddSocketVolumeMountToContainers(appContainers, socketVolumeMount)...)
  // 这里为 pod 添加 Pluggable 的 volume 以及给 component pod 添加 volume mount
   patchOps = append(patchOps, componentPatchOps...)
	// ...
   return patchOps, nil
}
```

`splitContainers`函数根据pod定义是否包含`dapr.io/inject-pluggable-components`注解来决定是否启用自动注入pluggable container。

```go

func (i *injector) splitContainers(pod corev1.Pod) (appContainers map[int]corev1.Container, componentContainers map[int]corev1.Container, injectedComponentContainers []corev1.Container, err error) {
	an := annotations.New(pod.Annotations)
	injectionEnabled := an.GetBoolOrDefault(annotations.KeyPluggableComponentsInjection, false)
	if injectionEnabled {
    // 列出namespace下的所有component
		componentsList, err, _ := namespaceFlight.Do(pod.Namespace, func() (any, error) {
			return i.daprClient.ComponentsV1alpha1().Components(pod.Namespace).List(metav1.ListOptions{})
		})
		if err != nil {
			return nil, nil, nil, fmt.Errorf("error when fetching components: %w", err)
		}
    // 自动注入pluggable container
		injectedComponentContainers = components.Injectable(an.GetString(annotations.KeyAppID), componentsList.(*v1alpha1.ComponentList).Items)
	}
  // pod注解`dapr.io/pluggable-components`中以`,`分割了pluggable component的container名称。将它们筛选出来就是`componentContainers`.
	appContainers, componentContainers = components.SplitContainers(pod)
	return
}
```

`Injectable`函数会筛选出来所有带有`dapr.io/component-container`注解的，限定app-id列表中包含自己，或者未限定的，将其`dapr.io/inject-pluggable-components`定义解析后自动注入进来。

```go
// Injectable parses the container definition from components annotations returning them as a list. Uses the appID to filter
// only the eligble components for such apps avoiding injecting containers that will not be used.
func Injectable(appID string, components []componentsapi.Component) []corev1.Container {
	componentContainers := make([]corev1.Container, 0)
	componentImages := make(map[string]bool, 0)
	// 遍历所有components
	for _, component := range components {
    // component是否包含dapr.io/component-container注解，这里定义了PluggableComponentContainer基本信息
		containerAsStr := component.Annotations[annotations.KeyPluggableComponentContainer]
		if containerAsStr == "" {
			continue
		}
    // 反序列化成k8s Container
		var container *corev1.Container
		if err := json.Unmarshal([]byte(containerAsStr), &container); err != nil {
			log.Warnf("could not unmarshal %s error: %v", component.Name, err)
			continue
		}
		// 如果已经处理过了，一个镜像只需要启动一个实例。因为一个component可能会包含了多个pluggable。这样如果每个component中都定义了这个，就需要去重，因为需且只需启动一份实例。
		if componentImages[container.Image] {
			continue
		}
		// 是否限定了app，如果限定了就需要包含自己才需要注入
		appScopped := len(component.Scopes) == 0
		for _, scoppedApp := range component.Scopes {
			if scoppedApp == appID {
				appScopped = true
				break
			}
		}
		// container加入定义中
		if appScopped {
			componentImages[container.Image] = true
			// if container name is not set, the component name will be used ensuring uniqueness
			if container.Name == "" {
				container.Name = component.Name
			}
			componentContainers = append(componentContainers, *container)
		}
	}

	return componentContainers
}
```

计算Patch信息，volume、VolumeMounts、环境变量等。

```go
// PatchOps returns the patch operations required to properly bootstrap the pluggable component and the respective volume mount for the sidecar.
func PatchOps(componentContainers map[int]corev1.Container, injectedContainers []corev1.Container, pod *corev1.Pod) ([]patcher.PatchOperation, *corev1.VolumeMount) {
	patches := make([]patcher.PatchOperation, 0)

	// check ...

  // 获取Pluggable Components Sockets文件夹，优先级为annotation、env、默认值。
	podAnnotations := annotations.New(pod.Annotations)
	mountPath := podAnnotations.GetString(annotations.KeyPluggableComponentsSocketsFolder)
	if mountPath == "" {
    // 尝试从`DAPR_COMPONENTS_SOCKETS_FOLDER`环境变量中取，还取不到就用默认值`/tmp/dapr-components-sockets`
		mountPath = pluggable.GetSocketFolderPath()
	}
	// 为pod添加socket 使用的volume
	volumePatch, sharedSocketVolumeMount := addSharedSocketVolume(mountPath, pod)
	patches = append(patches, volumePatch)
  // 将路径设置到`DAPR_COMPONENT_SOCKETS_FOLDER`环境变量
	componentsEnvVars := []corev1.EnvVar{{
		Name:  componentsUnixDomainSocketMountPathEnvVar,
		Value: sharedSocketVolumeMount.MountPath,
	}}

	for idx, container := range componentContainers {
    // 添加env和volumeMount，仅添加不与container本身定义中冲突的部分。
		patches = append(patches, patcher.GetEnvPatchOperations(container.Env, componentsEnvVars, idx)...)
		patches = append(patches, patcher.GetVolumeMountPatchOperations(container.VolumeMounts, []corev1.VolumeMount{sharedSocketVolumeMount}, idx)...)
	}

	podVolumes := make(map[string]bool, len(pod.Spec.Volumes))
	for _, volume := range pod.Spec.Volumes {
		podVolumes[volume.Name] = true
	}

	for _, container := range injectedContainers {
		container.Env = append(container.Env, componentsEnvVars...)
		// mount volume as empty dir by default.
		patches = append(patches, emptyVolumePatches(container, podVolumes, pod)...)
		container.VolumeMounts = append(container.VolumeMounts, sharedSocketVolumeMount)

		patches = append(patches, patcher.PatchOperation{
			Op:    "add",
			Path:  patcher.PatchPathContainers + "/-",
			Value: container,
		})
	}

	return patches, &sharedSocketVolumeMount
}
```

##### volume

添加共享socket volume到pod定义中。sharedSocketVolumeMount的添加在patch里。

```go
func addSharedSocketVolume(mountPath string, pod *corev1.Pod) (sidecar.PatchOperation, corev1.VolumeMount) {
	sharedSocketVolume := sharedComponentsSocketVolume()
	sharedSocketVolumeMount := sharedComponentsUnixSocketVolumeMount(mountPath)

	var volumePatch sidecar.PatchOperation

	if len(pod.Spec.Volumes) == 0 {
		volumePatch = sidecar.PatchOperation{
			Op:    "add",
			Path:  sidecar.PatchPathVolumes,
			Value: []corev1.Volume{sharedSocketVolume},
		}
	} else {
		volumePatch = sidecar.PatchOperation{
			Op:    "add",
      // 如果已经存在的话就是添加到一个已经存在的数组里
			Path:  sidecar.PatchPathVolumes + "/-",
			Value: sharedSocketVolume,
		}
	}
	// volume添加到pod
	pod.Spec.Volumes = append(pod.Spec.Volumes, sharedSocketVolume)
	return volumePatch, sharedSocketVolumeMount
}
```

Volume和VolumeMount定义

```go
// sharedComponentsSocketVolume creates a shared unix socket volume to be used by sidecar.
func sharedComponentsSocketVolume() corev1.Volume {
	return corev1.Volume{
		Name: componentsUnixDomainSocketVolumeName,//"dapr-components-unix-domain-socket"
		VolumeSource: corev1.VolumeSource{
			EmptyDir: &corev1.EmptyDirVolumeSource{},
		},
	}
}

// sharedComponentsUnixSocketVolumeMount creates a shared unix socket volume mount to be used by pluggable component.
func sharedComponentsUnixSocketVolumeMount(mountPath string) corev1.VolumeMount {
	return corev1.VolumeMount{
		Name:      componentsUnixDomainSocketVolumeName,//"dapr-components-unix-domain-socket"
		MountPath: mountPath,
	}
}
```































Tip: unix domain socket与Pluggable component共用同一个volume，因为Pluggable discovery在初始化阶段，然后才会初始化http和GRPC server，所以一起用应该是没有影响的。

参考

介绍文档https://docs.dapr.io/developing-applications/develop-components/pluggable-components/

overview https://docs.dapr.io/developing-applications/develop-components/pluggable-components/pluggable-components-overview/

Register a pluggable component https://docs.dapr.io/operations/components/pluggable-components-registration/

https://github.com/dapr/dapr/pull/5406/files

new injector pr https://github.com/dapr/dapr/pull/5935/files

pluggable component annotionshttps://github.com/dapr/dapr/issues/5668

初步提出prhttps://github.com/dapr/dapr/issues/3787

完善prhttps://github.com/dapr/dapr/issues/4925

https://docs.dapr.io/operations/components/pluggable-components-registration/

https://github.com/dapr/dapr/pull/4348

https://github.com/dapr/dapr/issues/5402

进行中pr

https://github.com/dapr/dapr/pull/6202 允许daprd等待/重试 pluggable组件的注册和初始化

