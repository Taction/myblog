---
title: "Dapr中的mtls源码分析介绍"
linkTitle: "Dapr mtls"
type: docs
date: 2022-12-13T19:35:55+08:00
draft: true
---

TLDR；本文主要从源码方面来介绍dapr中的mtls流程。包括ca、证书生成、轮换。

daprd是dapr sidecar，会作为一个单独的container，由injector自动注入。sentry是dapr的ca中心。daprd与sentry之间默认启用mtls，daprd通过访问sentry来申请工作负载证书。当开启mtls时，daprd与daprd之间的相互访问也通过mtls来认证。

### daprd

由于dapr cli的二进制名字为dapr，为了防止歧义，在文中将以daprd来代指dapr sidecar，以cli来代指dapr cli。

daprd初始化时从flag中读取是否允许mtls，其主要逻辑代码如下：

```go
func FromFlags() (*DaprRuntime, error) {
 enableMTLS := flag.Bool("enable-mtls", false, "Enables automatic mTLS for daprd to daprd communication channels")
 runtimeConfig := NewRuntimeConfig(NewRuntimeConfigOpts{
		ID:                           *appID,
		Mode:                         *mode,
		MTLSEnabled:                  *enableMTLS,
	})
 if *enableMTLS || *mode == string(modes.KubernetesMode) {
		runtimeConfig.CertChain, err = security.GetCertChain()
		if err != nil {
			return nil, err
		}
	}
}
```

如果开启mtls就从环境变量中读取证书配置，加载到runtimeConfig中。关于环境变量中证书信息是哪里来的，我们放到后文中介绍。证书读取到内存过程结束，这个证书不是daprd的工作负载证书，而是用于向sentry申请工作负载证书时进行通信的mtls信息。

```go
func GetCertChain() (*credentials.CertChain, error) {
	trustAnchors := os.Getenv(sentryConsts.TrustAnchorsEnvVar)
	if trustAnchors == "" {
		return nil, errors.Errorf("couldn't find trust anchors in environment variable %s", sentryConsts.TrustAnchorsEnvVar)
	}
	cert := os.Getenv(sentryConsts.CertChainEnvVar)
	if cert == "" {
		return nil, errors.Errorf("couldn't find cert chain in environment variable %s", sentryConsts.CertChainEnvVar)
	}
	key := os.Getenv(sentryConsts.CertKeyEnvVar)
	if cert == "" {
		return nil, errors.Errorf("couldn't find cert key in environment variable %s", sentryConsts.CertKeyEnvVar)
	}
	return &credentials.CertChain{
		RootCA: []byte(trustAnchors),
		Cert:   []byte(cert),
		Key:    []byte(key),
	}, nil
}
```

接下来就是sidecar的初始化，在daprd的initRuntime初始化方法中调用了`establishSecurity`方法，可以看到daprd利用环境变量中获得的信息构建了`Authenticator`，将其保存到runtime中并且赋值给了内部grpc。这个`Authenticator`负责具体的证书申请、内存存储逻辑。

```go
func (a *DaprRuntime) establishSecurity(sentryAddress string) error {
   if !a.runtimeConfig.mtlsEnabled {
      log.Info("mTLS is disabled. Skipping certificate request and tls validation")
      return nil
   }
   if sentryAddress == "" {
      return errors.New("sentryAddress cannot be empty")
   }
   log.Info("mTLS enabled. creating sidecar authenticator")

   auth, err := security.GetSidecarAuthenticator(sentryAddress, a.runtimeConfig.CertChain)
   if err != nil {
      return err
   }
   a.authenticator = auth
   a.grpc.SetAuthenticator(auth)

   log.Info("authenticator created")

   diag.DefaultMonitoring.MTLSInitCompleted()
   return nil
}
```

#### server部分

主要分为初始化和证书轮换两部分。

```go
func (a *DaprRuntime) startGRPCInternalServer(api grpc.API, port int) error {
   // Since GRPCInteralServer is encrypted & authenticated, it is safe to listen on *
   serverConf := a.getNewServerConfig([]string{""}, port)
   server := grpc.NewInternalServer(api, serverConf, a.globalConfig.Spec.TracingSpec, a.globalConfig.Spec.MetricSpec, a.authenticator, a.proxy)
   if err := server.StartNonBlocking(); err != nil {
      return err
   }
   a.apiClosers = append(a.apiClosers, server)

   return nil
}
```

在生成一个new grpcserver的时候，主要就是中间件处理和tls证书处理。证书的具体的生成逻辑在`generateWorkloadCert`函数中，即向sentry发送请求来申请一张证书。注意tlsConfig中的GetCertificate，这个是服务端根据client hello请求来决定返回的服务端证书。这个函数非常有用，比如可以用来支持证书轮换或者服务端具有多个不同域名等。每次建立tcp连接都会触发这个函数，所以当证书有改变时，新的连接获取的就是新的证书。单纯对于mtls 来说，tlsconfig中另外两个比较有用的函数是`VerifyPeerCertificate`和`VerifyConnection`这两个函数可以针对证书做一些自定义的检查，比如根据证书中的某些信息校验是否与其中断连接.然后注意这里启动了一个协程来执行证书轮换函数`startWorkloadCertRotation`,具体逻辑会放在后面讲。

```go
func (s *server) getGRPCServer() (*grpcGo.Server, error) {
   opts := s.getMiddlewareOptions()
   if s.maxConnectionAge != nil {
      opts = append(opts, grpcGo.KeepaliveParams(keepalive.ServerParameters{MaxConnectionAge: *s.maxConnectionAge}))
   }

   if s.authenticator != nil {
      err := s.generateWorkloadCert()
      if err != nil {
         return nil, err
      }

      //nolint:gosec
      tlsConfig := tls.Config{
         ClientCAs:  s.signedCert.TrustChain,
         ClientAuth: tls.RequireAndVerifyClientCert,
         GetCertificate: func(*tls.ClientHelloInfo) (*tls.Certificate, error) {
            return &s.tlsCert, nil
         },
      }
      ta := credentials.NewTLS(&tlsConfig)

      opts = append(opts, grpcGo.Creds(ta))
      go s.startWorkloadCertRotation()
   }

	// ... 拼接各项配置
   return grpcGo.NewServer(opts...), nil
}
```

generateWorkloadCert负责具体的证书生成逻辑，就是调用authenticator的CreateSignedWorkloadCert函数，获取一张被sentry签名过的证书，解析并存储。初始化server和证书轮换时都会调用这个函数。

```go
func (s *server) generateWorkloadCert() error {
   s.logger.Info("sending workload csr request to sentry")
   signedCert, err := s.authenticator.CreateSignedWorkloadCert(s.config.AppID, s.config.NameSpace, s.config.TrustDomain)
   if err != nil {
      return errors.Wrap(err, "error from authenticator CreateSignedWorkloadCert")
   }
   s.logger.Info("certificate signed successfully")

   tlsCert, err := tls.X509KeyPair(signedCert.WorkloadCert, signedCert.PrivateKeyPem)
   if err != nil {
      return errors.Wrap(err, "error creating x509 Key Pair")
   }

   s.signedCert = signedCert
   s.tlsCert = tlsCert
   s.signedCertDuration = signedCert.Expiry.Sub(time.Now().UTC())
   return nil
}
```

证书申请：以下函数忽略了无关细节以及错误处理，整体来说就是生成csr根据配置中的tls信息访问sentry获取签名后证书。

```go
// CreateSignedWorkloadCert returns a signed workload certificate, the PEM encoded private key
// And the duration of the signed cert.
func (a *authenticator) CreateSignedWorkloadCert(id, namespace, trustDomain string) (*SignedCertificate, error) {
   // 通过genCSRFunc函数生成一个Certificate Signing Request，以及一个私钥 private key in pem format
   csrb, pkPem, err := a.genCSRFunc(id)
   certPem := pem.EncodeToMemory(&pem.Block{Type: certType, Bytes: csrb})
  // 根据a.certChainPem, a.keyPem, TLSServerName, a.trustAnchors这些信息生成访问sentry的的tls config，使用tls创建grpc的连接，并请求对csr进行签名。
   config, err := daprCredentials.TLSConfigFromCertAndKey(a.certChainPem, a.keyPem, TLSServerName, a.trustAnchors)
   conn, err := grpc.Dial(
      a.sentryAddress,
      grpc.WithTransportCredentials(credentials.NewTLS(config)),
      grpc.WithUnaryInterceptor(unaryClientInterceptor))
   c := sentryv1pb.NewCAClient(conn)

   resp, err := c.SignCertificate(context.Background(),
      &sentryv1pb.SignCertificateRequest{
         CertificateSigningRequest: certPem,
         Id:                        getSentryIdentifier(id),
         Token:                     getToken(),
         TrustDomain:               trustDomain,
         Namespace:                 namespace,
      }, grpcRetry.WithMax(sentryMaxRetries), grpcRetry.WithPerRetryTimeout(sentrySignTimeout))

   signedCert := &SignedCertificate{
      WorkloadCert:  workloadCert,
      PrivateKeyPem: pkPem,
      Expiry:        expiry,
      TrustChain:    trustChain,
   }
   return signedCert, nil
}
```

证书轮换的逻辑很简单，就是开启一个定时器，加锁检查证书有效时间除以证书总有效期是否已经不足百分之三十，如果低于这个值，就申请一张新的证书即可。

```go
func (s *server) startWorkloadCertRotation() {
   s.logger.Infof("starting workload cert expiry watcher. current cert expires on: %s", s.signedCert.Expiry.String())

   ticker := time.NewTicker(certWatchInterval)

   for range ticker.C {
      s.renewMutex.Lock()
     // 判断证书剩余有效期是否已经不足百分之三十
      renew := shouldRenewCert(s.signedCert.Expiry, s.signedCertDuration)
      if renew {
         s.logger.Info("renewing certificate: requesting new cert and restarting gRPC server")

         err := s.generateWorkloadCert()
         if err != nil {
            s.logger.Errorf("error starting server: %s", err)
            s.renewMutex.Unlock()
            continue
         }
         diag.DefaultMonitoring.MTLSWorkLoadCertRotationCompleted()
      }
      s.renewMutex.Unlock()
   }
}
```

#### client部分

看完了服务端处理，我们再来看一下客户端处理。忽略无关代码，可以看到当mtls开启的时候，建立连接会先获取一张证书，这张证书跟服务端是同一张证书，因此到期之前服务端会轮换，所以不用考虑有效期问题。然后以这张证书构建客户端的tls配置。

```go
func (g *Manager) connectRemote(
   parentCtx context.Context,
   address string,
   id string,
   namespace string,
   customOpts ...grpc.DialOption,
) (conn *grpc.ClientConn, err error) {
   if g.auth != nil {
      signedCert := g.auth.GetCurrentSignedCert()
      var cert tls.Certificate
      cert, err = tls.X509KeyPair(signedCert.WorkloadCert, signedCert.PrivateKeyPem)
      if err != nil {
         return nil, errors.Errorf("error loading x509 Key Pair: %s", err)
      }

      var serverName string
      if id != "cluster.local" {
         serverName = id + "." + namespace + ".svc.cluster.local"
      }

      //nolint:gosec
      ta := credentials.NewTLS(&tls.Config{
         ServerName:   serverName,
         Certificates: []tls.Certificate{cert},
         RootCAs:      signedCert.TrustChain,
      })
      opts = append(opts, grpc.WithTransportCredentials(ta))
   } else {
      opts = append(opts, grpc.WithTransportCredentials(insecure.NewCredentials()))
   }
   conn, err = grpc.DialContext(ctx, dialPrefix+address, opts...)
   return conn, nil
}
```

### injector

injector为pod注入daprd容器。由于daprd会访问sentry申请工作负载证书，所以在注入容器时，injector同时会在其环境变量添加对sentry进行访问的tls证书信息。通过`getPodPatchOperations`函数来创建patch 操作，将daprd容器注入。主要逻辑为获取tls信息，使用tls信息以及其他配置信息生成daprd的container信息。

```go
func (i *injector) getPodPatchOperations(ar *v1.AdmissionReview,
	namespace, image, imagePullPolicy string, kubeClient kubernetes.Interface, daprClient scheme.Interface,
) (patchOps []sidecar.PatchOperation, err error) {
  // ...
  // 首先判断是否需要注入sidecar
  an := sidecar.Annotations(pod.Annotations)
	if !an.GetBoolOrDefault(annotations.KeyEnabled, false) || sidecar.PodContainsSidecarContainer(&pod) {
		return nil, nil
	}
  // ...
  // 获取tls配置以及其他配置，生成sidecar的container
	trustAnchors, certChain, certKey := sidecar.GetTrustAnchorsAndCertChain(context.TODO(), kubeClient, namespace)
	// Get the sidecar container
	sidecarContainer, err := sidecar.GetSidecarContainer(sidecar.ContainerConfig{
		MTLSEnabled:                  mTLSEnabled(daprClient),
    CertChain:                    certChain,
		CertKey:                      certKey,
		TrustAnchors:                 trustAnchors,
    // ...
	})
  // ...
  // 添加sidecar container、添加dapr 端口env信息到app containers、添加sock volume挂载（从app annotation中解析路径，用于app与daprd通过socket通信）
  return patchOps, nil
}
```

首先来看获取tls信息的逻辑，这部分非常简单，就是从k8s的secrets中读取进来就可以了。

```go
// 通过具体的获取tls配置函数可以看到，具体的内容是从k8s的secrets中获取的。
// GetTrustAnchorsAndCertChain returns the trust anchor and certs.
func GetTrustAnchorsAndCertChain(ctx context.Context, kubeClient kubernetes.Interface, namespace string) (string, string, string) {
	secret, err := kubeClient.CoreV1().Secrets(namespace).Get(ctx, sentryConsts.TrustBundleK8sSecretName, metav1.GetOptions{})
	if err != nil {
		return "", "", ""
	}

	rootCert := secret.Data[credentials.RootCertFilename]
	certChain := secret.Data[credentials.IssuerCertFilename]
	certKey := secret.Data[credentials.IssuerKeyFilename]
	return string(rootCert), string(certChain), string(certKey)
}
```

来看生成daprd的container信息部分，去掉跟tls无关的设置以及错误处理，就是生成一个daprd的k8s container信息，然后将tls信息放到其环境变量配置中。

```go
func GetSidecarContainer(cfg ContainerConfig) (*corev1.Container, error) {
 // ...
 // 生成sidecar的container信息
 container := &corev1.Container{
		Name:            SidecarContainerName,
		Image:           cfg.DaprSidecarImage,
		Ports: ports,
		Args:  append(cmd, args...),
		Env: []corev1.EnvVar{
		},
	}
	// ...
	// 将证书信息放到环境变量里，添加是否允许mtls的启动命令flag。这里可以看到，不管是否允许mtls，都生成证书放环境变量里了
 container.Env = append(container.Env,
		corev1.EnvVar{
			Name:  sentryConsts.TrustAnchorsEnvVar,
			Value: cfg.TrustAnchors,
		},
		corev1.EnvVar{
			Name:  sentryConsts.CertChainEnvVar,
			Value: cfg.CertChain,
		},
		corev1.EnvVar{
			Name:  sentryConsts.CertKeyEnvVar,
			Value: cfg.CertKey,
		},
		corev1.EnvVar{
			Name:  "SENTRY_LOCAL_IDENTITY",
			Value: cfg.Identity,
		},
	)
	if cfg.MTLSEnabled {
		container.Args = append(container.Args, "--enable-mtls")
	}
	return container, nil
}
```

### sentry

从上面可以看到injector在注入证书信息的时候是从环境变量中读取mtls信息并设置到daprd的环境变量中，那么这些信息是哪里来的呢？daprd的工作负载证书又是如何签署的呢？答案就是在sentry这个服务中，这是dapr实现的一个自身的ca中心。

#### 启动

可以看到启动就是做了一堆参数和控制信号的操作，然后new一个ca实例，并Start。

```go
func main() {
   // 解析读取各种flag和配置
	 ca := sentry.NewSentryCA()
   go func() {
      // Restart the server when the issuer credentials change
      // 监听证书有变化就重启ca
   }()

   // Start the health server in background
   // ...

   // Start the server in background
   err = ca.Start(runCtx, config)
   if err != nil {
      log.Fatalf("failed to restart sentry server: %s", err)
   }

   // Watch for changes in the watchDir
   // This also blocks until runCtx is canceled
   fswatcher.Watch(runCtx, watchDir, issuerEvent)

   shutdownDuration := 5 * time.Second
   log.Infof("allowing %s for graceful shutdown to complete", shutdownDuration)
   <-time.After(shutdownDuration)
}
```

Start的逻辑比较简单，主要就是两个函数的调用，首先调用createCAServer初始化ca信息和mtls访问信息，然后调用run启动。

```go
// Start the server in background.
func (s *sentry) Start(ctx context.Context, conf config.SentryConfig) error {
   // If the server is already running, return an error
   select {
   case s.running <- true:
   default:
      return errors.New("CertificateAuthority server is already running")
   }

   // Create the CA server
   // ca server主要就是验证签名，进行签名；同时会存储访问自身的mtls信息，就是个工具类。在createCAServer的时候会调用LoadOrStoreTrustBundle来初始化或者加载
   s.conf = conf
   certAuth, v := s.createCAServer()

   // Start the server in background
   // 新建协程启动server
   s.ctx, s.cancel = context.WithCancel(ctx)
   go s.run(certAuth, v)

   // Wait 100ms to ensure a clean startup
   time.Sleep(100 * time.Millisecond)

   return nil
}
```

先来看启动函数就是创建server，然后run。在run的时候会通过`tlsServerOption`设置服务端tls。通过tls配置可以看到访问sentry是需要客户端证书的。

```go
// Runs the CA server.
// This method blocks until the server is shut down.
func (s *sentry) run(certAuth ca.CertificateAuthority, v identity.Validator) {
   s.server = server.NewCAServer(certAuth, v)

   // In background, watch for the root certificate's expiration
   // 每小时检查root ca证书是不是快要过期了
   go watchCertExpiry(s.ctx, certAuth)

   // Watch for context cancelation to stop the server
	 // ...
  
   // Start the server; this is a blocking call
   log.Infof("sentry certificate authority is running, protecting y'all")
   serverRunErr := s.server.Run(s.conf.Port, certAuth.GetCACertBundle())
   if serverRunErr != nil {
      log.Fatalf("error starting gRPC server: %s", serverRunErr)
   }
}
```

sentry tls的ClientAuth设置为了`RequireAndVerifyClientCert`，意味着访问sentry需要客户端证书。且sentry的服务端证书是短生命周期的自动轮换（因为手握ca随时可以签一张新的）。

```go
func (s *server) tlsServerOption(trustBundler ca.TrustRootBundler) grpc.ServerOption {
   cp := trustBundler.GetTrustAnchors()

   //nolint:gosec
   config := &tls.Config{
      ClientCAs: cp,
      // Require cert verification
      ClientAuth: tls.RequireAndVerifyClientCert,
      GetCertificate: func(*tls.ClientHelloInfo) (*tls.Certificate, error) {
         if s.certificate == nil || needsRefresh(s.certificate, serverCertExpiryBuffer) {
            cert, err := s.getServerCertificate()
            if err != nil {
               monitoring.ServerCertIssueFailed("server_cert")
               log.Error(err)
               return nil, errors.Wrap(err, "failed to get TLS server certificate")
            }
            s.certificate = cert
         }
         return s.certificate, nil
      },
   }
   return grpc.Creds(credentials.NewTLS(config))
}
```

#### 设置访问自身的证书信息

然后我们来看createCAServer函数，这个是在上文的start函数中调用的。加载或者生成自签名ca证书，以及生成用于访问自身的mtls的客户端证书信息。在这个过程中，如果是新生成的证书会被存储起来。

```go
// Loads the trust anchors and issuer certs, then creates a new CA.
func (s *sentry) createCAServer() (ca.CertificateAuthority, identity.Validator) {
   // Create CA
   certAuth, authorityErr := ca.NewCertificateAuthority(s.conf)
   // Load the trust bundle
   // 初始化用于通信的ca及证书信息
   trustStoreErr := certAuth.LoadOrStoreTrustBundle()
   if trustStoreErr != nil {
      log.Fatalf("error loading trust root bundle: %s", trustStoreErr)
   }
   
   // Create identity validator
   v, validatorErr := s.createValidator()
   if validatorErr != nil {
      log.Fatalf("error creating validator: %s", validatorErr)
   }
   log.Info("validator created")

   return certAuth, v
}
```

这个函数的作用就是拿到`*trustRootBundle`，这里面存储了自签名ca及用于访问自身的client证书信息，首先判断是否需要重新生成证书，如果不需要就直接加载，如果需要就生成证书（生成证书时会进行存储）。同时在上文创建server时，tls配置中ClientCAs也是从这里创建的`*trustRootBundle`中取的。

```go
func (c *defaultCA) validateAndBuildTrustBundle() (*trustRootBundle, error) {
   var (
      issuerCreds     *certs.Credentials
      rootCertBytes   []byte
      issuerCertBytes []byte
   )
   // certs exist on disk or getting created, load them when ready
   if !shouldCreateCerts(c.config) {
      // ...
      // 这里就是加载已经存在的证书
      rootCertBytes = certChain.RootCA
      issuerCertBytes = certChain.Cert
   } else {
      // create self signed root and issuer certs
      issuerCreds, rootCertBytes, issuerCertBytes, err = c.generateRootAndIssuerCerts()
   }
   // load trust anchors
   trustAnchors, err := certs.CertPoolFromPEM(rootCertBytes)
   return &trustRootBundle{
      issuerCreds:   issuerCreds,
      trustAnchors:  trustAnchors,
      trustDomain:   c.config.TrustDomain,
      rootCertPem:   rootCertBytes,
      issuerCertPem: issuerCertBytes,
   }, nil
}
```

这里就是获取一张自签名的root ca证书，并且用这张证书签发一张用于通信的ca证书，签发的这张证书就是其他组件连接sentry是需要的客户端证书。有效期皆为一年。由于不同机器时钟可能不一致，AllowedClockSkew为可容忍的时钟漂移时间大小（就是在生成证书时设置notbefore为当前时间减去这个值，以防止其他机器比自己慢，认为证书还没到生效时间）。然后存储起来。

```go
func (c *defaultCA) generateRootAndIssuerCerts() (*certs.Credentials, []byte, []byte, error) {
   rootKey, err := certs.GenerateECPrivateKey()
   if err != nil {
      return nil, nil, nil, err
   }
   certsCredentials, rootCertPem, issuerCertPem, issuerKeyPem, err := GetNewSelfSignedCertificates(
      rootKey, selfSignedRootCertLifetime, c.config.AllowedClockSkew)
   if err != nil {
      return nil, nil, nil, err
   }
   // store credentials so that next time sentry restarts it'll load normally
   err = certs.StoreCredentials(c.config, rootCertPem, issuerCertPem, issuerKeyPem)
   if err != nil {
      return nil, nil, nil, err
   }

   return certsCredentials, rootCertPem, issuerCertPem, nil
}
```

#### mtls证书存储

`certs.StoreCredentials`会根据不同的运行环境进行存储，如果是运行在k8s模式下就会存在secrets中。这个secrets就是injector在给应用注入sidecar container的时候取的secrets，从这里开始就跟前面介绍的内容衔接上了。如果非k8s模式就存储在文件中，这里就不详细介绍了。

```go
func storeKubernetes(rootCertPem, issuerCertPem, issuerCertKey []byte) error {
   kubeClient, err := kubernetes.GetClient()
   if err != nil {
      return err
   }

   namespace := getNamespace()
   secret := &v1.Secret{
      Data: map[string][]byte{
         credentials.RootCertFilename:   rootCertPem,
         credentials.IssuerCertFilename: issuerCertPem,
         credentials.IssuerKeyFilename:  issuerCertKey,
      },
      ObjectMeta: metav1.ObjectMeta{
         Name:      consts.TrustBundleK8sSecretName,
         Namespace: namespace,
      },
      Type: v1.SecretTypeOpaque,
   }

   // We update and not create because sentry expects a secret to already exist
   _, err = kubeClient.CoreV1().Secrets(namespace).Update(context.TODO(), secret, metav1.UpdateOptions{})
   if err != nil {
      return errors.Wrap(err, "failed saving secret to kubernetes")
   }
   return nil
}
```

#### 证书签署

上面看到了sentry启动过程中对于证书的初始化，以及对于自身server证书的设置。接下来看下daprd在启动时请求的证书签署接口的逻辑。这个接口主要逻辑是接收来自csr，解析后填充部分信息生成一个待签名的证书，然后使用ca证书进行签署，并将签署后的证书返回给请求方。这个函数中有两个地方需要注意一下，一个是验证负载身份，一个是进行证书签名。

```go
func (s *server) SignCertificate(ctx context.Context, req *sentryv1pb.SignCertificateRequest) (*sentryv1pb.SignCertificateResponse, error) {
   // csr参数获取及解析
   csrPem := req.GetCertificateSigningRequest()
   csr, err := certs.ParsePemCSR(csrPem)
   err = s.certAuth.ValidateCSR(csr)
   // 如果处于k8s模式下验证工作负载身份，否则此验证函数目前什么都没做
   err = s.validator.Validate(req.GetId(), req.GetToken(), req.GetNamespace())
   // 调用SignCSR来生成签名后的证书
   identity := identity.NewBundle(csr.Subject.CommonName, req.GetNamespace(), req.GetTrustDomain())
   signed, err := s.certAuth.SignCSR(csrPem, csr.Subject.CommonName, identity, -1, false)
   
	 // 组装证书及签署它的证书链
   certPem := signed.CertPEM
   issuerCert := s.certAuth.GetCACertBundle().GetIssuerCertPem()
   rootCert := s.certAuth.GetCACertBundle().GetRootCertPem()
   certPem = append(certPem, issuerCert...)
   if len(rootCert) > 0 {
      certPem = append(certPem, rootCert...)
   }

	 // 返回证书、证书链、过期时间
   resp := &sentryv1pb.SignCertificateResponse{
      WorkloadCertificate:    certPem,
      TrustChainCertificates: [][]byte{issuerCert, rootCert},
      ValidUntil:             expiry,
   }
   return resp, nil
}
```

由csr生成证书的逻辑，首先进行数据校验，然后生成并签署证书。

```go
func (c *defaultCA) SignCSR(csrPem []byte, subject string, identity *identity.Bundle, ttl time.Duration, isCA bool) (*SignedCertificate, error) {
   c.issuerLock.RLock()
   defer c.issuerLock.RUnlock()

   // 根据请求参数和自身配置决定证书有效期
   certLifetime := ttl
   if certLifetime.Seconds() <= 0 {
      certLifetime = c.config.WorkloadCertTTL
   }
   certLifetime += c.config.AllowedClockSkew
	 
   // 获取用于签名的ca证书以及其私钥
   signingCert := c.bundle.issuerCreds.Certificate
   signingKey := c.bundle.issuerCreds.PrivateKey

   // 将csr由字节流解析成内存模式
   cert, err := certs.ParsePemCSR(csrPem)
   if err != nil {
      return nil, errors.Wrap(err, "error parsing csr pem")
   }
   // 根据csr和一起其他配置生成一张已经签名的证书
   crtb, err := csr.GenerateCSRCertificate(cert, subject, identity, signingCert, cert.PublicKey, signingKey, certLifetime, c.config.AllowedClockSkew, isCA)
   if err != nil {
      return nil, errors.Wrap(err, "error signing csr")
   }
   // 将证书由字节数组转化为内存模式
   csrCert, err := x509.ParseCertificate(crtb)
   if err != nil {
      return nil, errors.Wrap(err, "error parsing cert")
   }
	 // 对证书进行pem格式编码
   certPem := pem.EncodeToMemory(&pem.Block{
      Type:  certs.BlockTypeCertificate,
      Bytes: crtb,
   })
   return &SignedCertificate{
      Certificate: csrCert,
      CertPEM:     certPem,
   }, nil
}
```

k8s模式下验证请求者身份，daprd在请求时会附带k8s生成并挂载进去的service account token，其中附带了用户的身份。sentry会向k8s校验token合法性，然后将token对应的用户信息与daprd发送请求时附带的id信息进行校验，以确定当前请求是否合法，保证调用自己的服务是经过授权的。

注：在目前的版本中没有看到为每个容器创建service account，因此pod挂载的是此namespace下的名为default的service account。

```go
func (v *validator) Validate(id, token, namespace string) error {
   
   tokenReview := &kauthapi.TokenReview{
      Spec: kauthapi.TokenReviewSpec{
         Token:     token,
         Audiences: audiences,
      },
   }

tr: // TODO: Remove once the NoDefaultTokenAudience feature is finalized
	 // 一般来说prts是system:serviceaccount:namespace:default进行：分割后的值，id为namespace:default。其中namespace为实际的namespace。
   prts, err := v.executeTokenReview(tokenReview)
   if err != nil {
      return err
   }

   if len(prts) != 4 || prts[0] != "system" {
      return fmt.Errorf("%s: provided token is not a properly structured service account token", errPrefix)
   }
   podSa := prts[3]
   podNs := prts[2]

   if namespace != "" {
      if podNs != namespace {
         return fmt.Errorf("%s: namespace mismatch. received namespace: %s", errPrefix, namespace)
      }
   }

   if id != podNs+":"+podSa {
      return fmt.Errorf("%s: token/id mismatch. received id: %s", errPrefix, id)
   }
   return nil
}
```

### 集群证书轮换

在sentry启动时，我们可以看到，有证书信息存在就不会重新生成了。那么这个一年的证书同样会过期，一旦过期，整个集群都将瘫痪（如果开启了mtls）。那么在这个证书快要到期的时候同样是需要轮换的。dapr是如何做的呢？按照上面的代码来看，其实轮换也比较简单，生成一份新的ca和client证书，然后存到对应的位置，重启sentry。这样的话就会存在一个问题，sidecar环境变量里的是老的信息，就没法访问sentry了。那么就需要在生成新的配置的时候，要么在一定时间内老的还能用，要么就把集群里的重新刷一遍。带着这些思考我们来看dapr的实现方案：

集群证书轮换是cli的功能，通过[官方文档](https://docs.dapr.io/reference/cli/dapr-mtls/dapr-mtls-renew-certificate/)我们可以看到通过指令`dapr mtls renew-certificate [flags]`可以进行证书轮换。同时有flag支持设置各项配置以及对控制面组件进行重启。（这里的dapr命令实际上执行的是dapr的cli，它的二进制名字就是dapr，而sidecar的二进制名字为daprd）

来看命令`dapr mtls renew-certificate`执行的具体代码，可以看到此命令目前仅针对k8s做具体的轮换逻辑。首先判断用户是否设置了证书信息，如果设置了，说明用户想轮换为用户自行生成的证书。然后判断用户是否提供了私钥，如果提供私钥说明需要用此私钥生成root ca证书。最后如果都没有提供就自动生成所有的证书及私钥。去掉参数检查和异常处理，关键逻辑如下，轮换逻辑通过函数`RenewCertificate`这里是对不同的情况判断以及传递不同的参数。

```go
func(cmd *cobra.Command, args []string) {
   var err error
   pkFlag := cmd.Flags().Lookup("private-key").Changed
   rootcertFlag := cmd.Flags().Lookup("ca-root-certificate").Changed
   issuerKeyFlag := cmd.Flags().Lookup("issuer-private-key").Changed
   issuerCertFlag := cmd.Flags().Lookup("issuer-public-certificate").Changed
   if kubernetesMode {
     // 是否设置了证书信息
      if rootcertFlag || issuerKeyFlag || issuerCertFlag {
         err = kubernetes.RenewCertificate(kubernetes.RenewCertificateParams{
            RootCertificateFilePath:   caRootCertificateFile,
            IssuerCertificateFilePath: issuerPublicCertificateFile,
            IssuerPrivateKeyFilePath:  issuerPrivateKeyFile,
         })
      // 是否提供了私钥信息
      } else if pkFlag {
         err = kubernetes.RenewCertificate(kubernetes.RenewCertificateParams{
            RootPrivateKeyFilePath: privateKey,
         })
      // 都没有提供就全部自行生成
      } else {
         print.InfoStatusEvent(os.Stdout, "generating fresh certificates")
         err = kubernetes.RenewCertificate(kubernetes.RenewCertificateParams{
         })
      }
   }
},
```

RenewCertificate执行真正的轮换逻辑，这个函数及其子函数主要执行的逻辑为：加载或者生成证书信息，获取helm client，使用证书信息生成helm更新的请求，更新dapr的helm。执行到这里，新的证书信息已经被更新到了k8s的secrets中。但是所有的daprd以及控制面中的配置以及内存中的信息都还是旧的。

```go
func RenewCertificate(conf RenewCertificateParams) error {
  // 如果提供了证书信息就加载
	if conf.RootCertificateFilePath != "" && conf.IssuerCertificateFilePath != "" && conf.IssuerPrivateKeyFilePath != "" {
		rootCertBytes, issuerCertBytes, issuerKeyBytes, err = parseCertificateFiles(conf.RootCertificateFilePath, conf.IssuerCertificateFilePath, conf.IssuerPrivateKeyFilePath)
	} else {
    // 没有提供证书信息就生成一份新的
		rootCertBytes, issuerCertBytes, issuerKeyBytes, err = GenerateNewCertificates(conf.RootPrivateKeyFilePath)
	}
  // 轮换
	err = renewCertificate(rootCertBytes, issuerCertBytes, issuerKeyBytes, conf.Timeout, conf.ImageVariant)
	return nil
}

// 生成helm更新数据，然后应用更新
func renewCertificate(rootCert, issuerCert, issuerKey []byte, timeout uint, imageVariant string) error {
	// 数据准备以及helm client的生成
	// Get the control plane version from daprversion(1.x.x-mariner), if image variant is provided.
	// Here, imageVariant is used only to extract the actual control plane version,
	// and do some validation on top of that.
	if imageVariant != "" {
		daprVersion, daprImageVariant = utils.GetVersionAndImageVariant(daprVersion)
	}
	helmConf, err := helmConfig(status[0].Namespace)
	daprChart, err := daprChart(daprVersion, helmConf)
	upgradeClient := helm.NewUpgrade(helmConf)
	// Reuse the existing helm configuration values i.e. tags, registry, etc.
	upgradeClient.ReuseValues = true

	// 生成helm更新的参数
	vals, err := createHelmParamsForNewCertificates(string(rootCert), string(issuerCert), string(issuerKey))
	
  // 进行更新操作
	chart, err := GetDaprHelmChartName(helmConf)
	if _, err = upgradeClient.Run(chart, daprChart, vals); err != nil {
		return err
	}
	return nil
}

/// 将ca、issuerCert、issuerKey赋值到helm values对应的字段上
func createHelmParamsForNewCertificates(ca, issuerCert, issuerKey string) (map[string]interface{}, error) {
	chartVals := map[string]interface{}{}
	args := []string{}
	// 参数组装
	if ca != "" && issuerCert != "" && issuerKey != "" {
		args = append(args, fmt.Sprintf("dapr_sentry.tls.root.certPEM=%s", ca),
			fmt.Sprintf("dapr_sentry.tls.issuer.certPEM=%s", issuerCert),
			fmt.Sprintf("dapr_sentry.tls.issuer.keyPEM=%s", issuerKey),
		)
	} else {
		return nil, fmt.Errorf("parameters not found")
	}
	// 将字段解析为helm认可的格式
	for _, v := range args {
		if err := strvals.ParseInto(v, chartVals); err != nil {
			return nil, err
		}
	}
	return chartVals, nil
}
```

在上面函数运行完成后，会执行以下函数，打印证书过期时间，以及如果重启flag设置过的话，重启dapr的控制面，包括sentry、operator和placement。重启命令通过exec kubectl完成。

```go
func(cmd *cobra.Command, args []string) {
   // 打印过期是时间
   expiry, err := kubernetes.Expiry()
   if err != nil {
      logErrorAndExit(err)
   }
   print.SuccessStatusEvent(os.Stdout,
      fmt.Sprintf("Certificate rotation is successful! Your new certicate is valid through %s", expiry.Format(time.RFC1123)))

   if restartDaprServices {
     // 重启控制面
      restartControlPlaneService()
      if err != nil {
         print.FailureStatusEvent(os.Stdout, err.Error())
         os.Exit(1)
      }
   }
}
```

注：dapr正在考虑新的mtls支持方案。

### 参考

https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/authentication-resources/token-review-v1/

https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/service-accounts-admin/

https://cloudnative.to/blog/istio-certificate/

