**本章重点：**

介绍kube-apiserver启动过程中第三个步骤-定义通用配置，包含如下配置：

![image-20240731163822025](C:\mynote\images4\image-20240731163822025.png)

# 1. 背景介绍

接上文分析，这里直接从Run函数开始分析。这部分主要从代码角度，进行 kube-apiserver第三个流程分析

（1）资源注册。

（2）Cobra命令行参数解析（返回run函數）

（3）创建APIServer通用配置

（4）创建APIExtensionsServer

（5）创建KubeAPIServer

（6）创建AggregatorServer

（7）启动HTTP服务。

（8）启动HTTPS服务

# 2.Run

>**Run**函数可以分为四个部分：
>
>`这个Run函数就是真正开始启动Api-server的函数`
>
>（1）NewConfig
>
>（2）config.Complete
>
>（3）CreateServerChain
>
>（4）PrepareRun

```go
// Run runs the specified APIServer.  This should never exit.
func Run(opts options.CompletedOptions, stopCh <-chan struct{}) error {
	// To help debugging, immediately log version
	klog.Infof("Version: %+v", version.Get())

	klog.InfoS("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))

	config, err := NewConfig(opts)
	if err != nil {
		return err
	}
	completed, err := config.Complete()
	if err != nil {
		return err
	}
	server, err := CreateServerChain(completed)
	if err != nil {
		return err
	}

	prepared, err := server.PrepareRun()
	if err != nil {
		return err
	}

	return prepared.Run(stopCh)
}

```

# 3. NewConfig

* 主要有4个函数：

    * CreateKubeAPIServerConfig:

        * `创建 Kubernetes API 服务器配置`

    * CreateAPIExtensionsConfig:

        * `创建APIExtensionsServer服务器配置`

    * NewDefaultAuthenticationInfoResolverWrapper:

        * `创建一个webhook.NewDefaultAuthenticationInfoResolverWrapper实列，用来管理ProxyTransport、EgressSelector、LoopbackClientConfig和TracerProvider`

        * **ProxyTransport**:

            - `controlPlane.ExtraConfig.ProxyTransport` 提供了一个 HTTP 传输层，这通常用于配置 API 服务器与其他服务（如 webhook 服务）通信时使用的网络代理或者其他传输层设置。

            **EgressSelector**:

            - `controlPlane.GenericConfig.EgressSelector` 是一个出口选择器配置，用于管理和优化从 API 服务器到其他服务（如云服务或内部服务）的出站连接。通过选择合适的网络出口路径，可以增强通信的安全性和效率。

            **LoopbackClientConfig**:

            - `controlPlane.GenericConfig.LoopbackClientConfig` 提供了一个内部的客户端配置，用于 API 服务器内部组件或服务间的通信。这种配置通常是优化的，因为它只在同一主机或本地网络上使用。

            **TracerProvider**:

            - `controlPlane.GenericConfig.TracerProvider` 是用于配置追踪提供者的，允许 API 服务器实现请求追踪或性能分析。这有助于监控和调试 API 请求的生命周期，从而提升性能和可靠性。

    * createAggregatorConfig:

        * `创建AggregatorServer服务器配置`

```go
// NewConfig creates all the resources for running kube-apiserver, but runs none of them.
func NewConfig(opts options.CompletedOptions) (*Config, error) {
	c := &Config{
		Options: opts,
	}
	//创建 Kubernetes API 服务器配置:
	//核心配置（controlPlane），以及相关的服务解析器（serviceResolver）和插件初始化器（pluginInitializer）
	controlPlane, serviceResolver, pluginInitializer, err := CreateKubeAPIServerConfig(opts)
	if err != nil {
		return nil, err
	}
	c.ControlPlane = controlPlane
	//创建APIExtensionsServer服务器配置
	apiExtensions, err := apiserver.CreateAPIExtensionsConfig(*controlPlane.GenericConfig, controlPlane.ExtraConfig.VersionedInformers, pluginInitializer, opts.CompletedOptions, opts.MasterCount,
		//创建一个webhook.NewDefaultAuthenticationInfoResolverWrapper实列，用来管理ProxyTransport、EgressSelector、LoopbackClientConfig和TracerProvider
		serviceResolver, webhook.NewDefaultAuthenticationInfoResolverWrapper(controlPlane.ExtraConfig.ProxyTransport, controlPlane.GenericConfig.EgressSelector, controlPlane.GenericConfig.LoopbackClientConfig, controlPlane.GenericConfig.TracerProvider))
	if err != nil {
		return nil, err
	}
	c.ApiExtensions = apiExtensions
	//创建AggregatorServer服务器配置
	aggregator, err := createAggregatorConfig(*controlPlane.GenericConfig, opts.CompletedOptions, controlPlane.ExtraConfig.VersionedInformers, serviceResolver, controlPlane.ExtraConfig.ProxyTransport, controlPlane.ExtraConfig.PeerProxy, pluginInitializer)
	if err != nil {
		return nil, err
	}
	c.Aggregator = aggregator

	return c, nil
}

```

## 3.1 CreateKubeAPIServerConfig

```go
// CreateKubeAPIServerConfig creates all the resources for running the API server, but runs none of them
func CreateKubeAPIServerConfig(opts options.CompletedOptions) (
	*controlplane.Config,
	aggregatorapiserver.ServiceResolver,
	[]admission.PluginInitializer,
	error,
) {
	//创建了一个网络代理传输配置，用于管理 API 服务器和其他服务
	proxyTransport := CreateProxyTransport()
	//根据完成的选项（opts.CompletedOptions）、相关的运行时方案和 OpenAPI 定义，构建了 API 服务器的通用配置(genericConfig)。这包括安全设置、API 路由、存储配置等。
	genericConfig, versionedInformers, storageFactory, err := controlplaneapiserver.BuildGenericConfig(
		opts.CompletedOptions,
		[]*runtime.Scheme{legacyscheme.Scheme, extensionsapiserver.Scheme, aggregatorscheme.Scheme},
		generatedopenapi.GetOpenAPIDefinitions,
	)
	if err != nil {
		return nil, nil, nil, err
	}
	//设置系统能力和监控指标:
	capabilities.Setup(opts.AllowPrivileged, opts.MaxConnectionBytesPerSec)
	opts.Metrics.Apply()
	serviceaccount.RegisterMetrics()
	//构建 API 服务器特定配置
	config := &controlplane.Config{
		GenericConfig: genericConfig,
		ExtraConfig: controlplane.ExtraConfig{
			APIResourceConfigSource: storageFactory.APIResourceConfigSource,
			StorageFactory:          storageFactory,
			EventTTL:                opts.EventTTL,
			KubeletClientConfig:     opts.KubeletConfig,
			EnableLogsSupport:       opts.EnableLogsHandler,
			ProxyTransport:          proxyTransport,

			ServiceIPRange:          opts.PrimaryServiceClusterIPRange,
			APIServerServiceIP:      opts.APIServerServiceIP,
			SecondaryServiceIPRange: opts.SecondaryServiceClusterIPRange,

			APIServerServicePort: 443,

			ServiceNodePortRange:      opts.ServiceNodePortRange,
			KubernetesServiceNodePort: opts.KubernetesServiceNodePort,

			EndpointReconcilerType: reconcilers.Type(opts.EndpointReconcilerType),
			MasterCount:            opts.MasterCount,

			ServiceAccountIssuer:        opts.ServiceAccountIssuer,
			ServiceAccountMaxExpiration: opts.ServiceAccountTokenMaxExpiration,
			ExtendExpiration:            opts.Authentication.ServiceAccounts.ExtendExpiration,

			VersionedInformers: versionedInformers,
		},
	}

	if utilfeature.DefaultFeatureGate.Enabled(features.UnknownVersionInteroperabilityProxy) {
		config.ExtraConfig.PeerEndpointLeaseReconciler, err = controlplaneapiserver.CreatePeerEndpointLeaseReconciler(*genericConfig, storageFactory)
		if err != nil {
			return nil, nil, nil, err
		}
		// build peer proxy config only if peer ca file exists
		if opts.PeerCAFile != "" {
			config.ExtraConfig.PeerProxy, err = controlplaneapiserver.BuildPeerProxy(versionedInformers, genericConfig.StorageVersionManager, opts.ProxyClientCertFile,
				opts.ProxyClientKeyFile, opts.PeerCAFile, opts.PeerAdvertiseAddress, genericConfig.APIServerID, config.ExtraConfig.PeerEndpointLeaseReconciler, config.GenericConfig.Serializer)
			if err != nil {
				return nil, nil, nil, err
			}
		}
	}
	//配置对等节点的客户端认证和请求头认证信息，这对于集群内部组件间的安全通信非常关键。
	clientCAProvider, err := opts.Authentication.ClientCert.GetClientCAContentProvider()
	if err != nil {
		return nil, nil, nil, err
	}
	config.ExtraConfig.ClusterAuthenticationInfo.ClientCA = clientCAProvider

	requestHeaderConfig, err := opts.Authentication.RequestHeader.ToAuthenticationRequestHeaderConfig()
	if err != nil {
		return nil, nil, nil, err
	}
	if requestHeaderConfig != nil {
		config.ExtraConfig.ClusterAuthenticationInfo.RequestHeaderCA = requestHeaderConfig.CAContentProvider
		config.ExtraConfig.ClusterAuthenticationInfo.RequestHeaderAllowedNames = requestHeaderConfig.AllowedClientNames
		config.ExtraConfig.ClusterAuthenticationInfo.RequestHeaderExtraHeaderPrefixes = requestHeaderConfig.ExtraHeaderPrefixes
		config.ExtraConfig.ClusterAuthenticationInfo.RequestHeaderGroupHeaders = requestHeaderConfig.GroupHeaders
		config.ExtraConfig.ClusterAuthenticationInfo.RequestHeaderUsernameHeaders = requestHeaderConfig.UsernameHeaders
	}

	// setup admission
	//设置 admission 控制
	admissionConfig := &kubeapiserveradmission.Config{
		ExternalInformers:    versionedInformers,
		LoopbackClientConfig: genericConfig.LoopbackClientConfig,
		CloudConfigFile:      opts.CloudProvider.CloudConfigFile,
	}
	serviceResolver := buildServiceResolver(opts.EnableAggregatorRouting, genericConfig.LoopbackClientConfig.Host, versionedInformers)
	pluginInitializers, err := admissionConfig.New(proxyTransport, genericConfig.EgressSelector, serviceResolver, genericConfig.TracerProvider)
	if err != nil {
		return nil, nil, nil, fmt.Errorf("failed to create admission plugin initializer: %v", err)
	}
	clientgoExternalClient, err := clientset.NewForConfig(genericConfig.LoopbackClientConfig)
	if err != nil {
		return nil, nil, nil, fmt.Errorf("failed to create real client-go external client: %w", err)
	}
	dynamicExternalClient, err := dynamic.NewForConfig(genericConfig.LoopbackClientConfig)
	if err != nil {
		return nil, nil, nil, fmt.Errorf("failed to create real dynamic external client: %w", err)
	}
	//应用 admission 配置
	err = opts.Admission.ApplyTo(
		genericConfig,
		versionedInformers,
		clientgoExternalClient,
		dynamicExternalClient,
		utilfeature.DefaultFeatureGate,
		pluginInitializers...)
	if err != nil {
		return nil, nil, nil, fmt.Errorf("failed to apply admission: %w", err)
	}

	if config.GenericConfig.EgressSelector != nil {
		// Use the config.GenericConfig.EgressSelector lookup to find the dialer to connect to the kubelet
		//使用 config.GenericConfig.EgressSelector 查找拨号器来连接到 kubelet
		config.ExtraConfig.KubeletClientConfig.Lookup = config.GenericConfig.EgressSelector.Lookup

		// Use the config.GenericConfig.EgressSelector lookup as the transport used by the "proxy" subresources.
		//使用 config.GenericConfig.EgressSelector 查找作为“代理”子资源使用的传输
		networkContext := egressselector.Cluster.AsNetworkContext()
		dialer, err := config.GenericConfig.EgressSelector.Lookup(networkContext)
		if err != nil {
			return nil, nil, nil, err
		}
		c := proxyTransport.Clone()
		c.DialContext = dialer
		config.ExtraConfig.ProxyTransport = c
	}

	// Load and set the public keys.
	//加密设置公钥
	var pubKeys []interface{}
	for _, f := range opts.Authentication.ServiceAccounts.KeyFiles {
		keys, err := keyutil.PublicKeysFromFile(f)
		if err != nil {
			return nil, nil, nil, fmt.Errorf("failed to parse key file %q: %v", f, err)
		}
		pubKeys = append(pubKeys, keys...)
	}
	config.ExtraConfig.ServiceAccountIssuerURL = opts.Authentication.ServiceAccounts.Issuers[0]
	config.ExtraConfig.ServiceAccountJWKSURI = opts.Authentication.ServiceAccounts.JWKSURI
	config.ExtraConfig.ServiceAccountPublicKeys = pubKeys

	return config, serviceResolver, pluginInitializers, nil
}

```

## 3.1.1 BuildGenericConfig初始化apiserver通用配置

> `主要生成和配置以下对象`：

**genericConfig (`genericapiserver.Config`)**:

- 这是核心的通用 API 服务器配置对象，包含了多种设置，如安全配置、审计策略、特性开关、存储配置等。

**versionedInformers (`clientgoinformers.SharedInformerFactory`)**:

- 这是 `client-go` 的 `SharedInformerFactory`，用于创建和管理 Kubernetes 资源的 informers。Informers 是用于监视、缓存和同步 Kubernetes 资源状态的工具，它们对于控制器和其他需要与 Kubernetes API 交互的组件来说是非常重要的。`versionedInformers` 提供了对各种资源的实时更新通知，有助于高效处理资源状态变化。。

**storageFactory (`serverstorage.DefaultStorageFactory`)**:

- 这是一个存储工厂，用于创建和配置 Kubernetes 资源的存储。它管理资源如何存储以及使用哪些后端（通常是 etcd）。

**clientgoExternalClient**:

- 这是一个用于与 Kubernetes API 交互的客户端，基于 loopback 配置（即服务器内部使用的客户端配置）。这个客户端用于 API 服务器内部的操作，如自身资源的访问和管理。

**EgressSelector**:

- 如果启用，该配置用于管理 API 服务器到其他服务的出站连接。它可以根据配置改变网络出口的选择策略。

**OpenAPI 和 OpenAPIV3 配置**:

- 这些配置用于生成和管理 API 文档。`OpenAPIConfig` 和 `OpenAPIV3Config` 为 API 服务器提供了描述其接口的机制，这对于客户端工具和开发者理解 API 是非常重要的。

**Authentication 和 Authorization 配置**:

- 这些配置负责 API 服务器的身份验证和授权机制。这包括配置如何验证请求者的身份和如何确定请求者是否有权进行特定的 API 调用。

**Audit**:

- 审计配置用于记录对 API 的操作，这对于安全、遵从性和故障诊断至关重要。

**AggregatedDiscoveryGroupManager**:

- 如果启用了聚合发现端点特性，这个管理器将用于处理聚合的资源组。

```go

// BuildGenericConfig takes the master server options and produces the genericapiserver.Config associated with it
func BuildGenericConfig(
	s controlplaneapiserver.CompletedOptions,
	schemes []*runtime.Scheme,
	getOpenAPIDefinitions func(ref openapicommon.ReferenceCallback) map[string]openapicommon.OpenAPIDefinition,
) (
	genericConfig *genericapiserver.Config,
	versionedInformers clientgoinformers.SharedInformerFactory,
	storageFactory *serverstorage.DefaultStorageFactory,

	lastErr error,
) {
	//生成genericConfig
	genericConfig = genericapiserver.NewConfig(legacyscheme.Codecs)
	genericConfig.MergedResourceConfig = controlplane.DefaultAPIResourceConfigSource()

	if lastErr = s.GenericServerRunOptions.ApplyTo(genericConfig); lastErr != nil {
		return
	}

	if lastErr = s.SecureServing.ApplyTo(&genericConfig.SecureServing, &genericConfig.LoopbackClientConfig); lastErr != nil {
		return
	}

	// Use protobufs for self-communication.
	// Since not every generic apiserver has to support protobufs, we
	// cannot default to it in generic apiserver and need to explicitly
	// set it in kube-apiserver.
	genericConfig.LoopbackClientConfig.ContentConfig.ContentType = "application/vnd.kubernetes.protobuf"
	// Disable compression for self-communication, since we are going to be
	// on a fast local network
	genericConfig.LoopbackClientConfig.DisableCompression = true

	kubeClientConfig := genericConfig.LoopbackClientConfig
	//生成clientgoExternalClient
	clientgoExternalClient, err := clientgoclientset.NewForConfig(kubeClientConfig)
	if err != nil {
		lastErr = fmt.Errorf("failed to create real external clientset: %v", err)
		return
	}
	//生成versionedInformers
	versionedInformers = clientgoinformers.NewSharedInformerFactory(clientgoExternalClient, 10*time.Minute)

	if lastErr = s.Features.ApplyTo(genericConfig, clientgoExternalClient, versionedInformers); lastErr != nil {
		return
	}
	if lastErr = s.APIEnablement.ApplyTo(genericConfig, controlplane.DefaultAPIResourceConfigSource(), legacyscheme.Scheme); lastErr != nil {
		return
	}
	//生成EgressSelector
	if lastErr = s.EgressSelector.ApplyTo(genericConfig); lastErr != nil {
		return
	}
	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerTracing) {
		if lastErr = s.Traces.ApplyTo(genericConfig.EgressSelector, genericConfig); lastErr != nil {
			return
		}
	}
	// wrap the definitions to revert any changes from disabled features
	getOpenAPIDefinitions = openapi.GetOpenAPIDefinitionsWithoutDisabledFeatures(getOpenAPIDefinitions)
	namer := openapinamer.NewDefinitionNamer(schemes...)
	//生成OpenAPI 和 OpenAPIV3 配置
	genericConfig.OpenAPIConfig = genericapiserver.DefaultOpenAPIConfig(getOpenAPIDefinitions, namer)
	genericConfig.OpenAPIConfig.Info.Title = "Kubernetes"
	genericConfig.OpenAPIV3Config = genericapiserver.DefaultOpenAPIV3Config(getOpenAPIDefinitions, namer)
	genericConfig.OpenAPIV3Config.Info.Title = "Kubernetes"
	//生成LongRunningFunc
	genericConfig.LongRunningFunc = filters.BasicLongRunningRequestCheck(
		sets.NewString("watch", "proxy"),
		sets.NewString("attach", "exec", "proxy", "log", "portforward"),
	)

	kubeVersion := version.Get()
	genericConfig.Version = &kubeVersion

	if genericConfig.EgressSelector != nil {
		s.Etcd.StorageConfig.Transport.EgressLookup = genericConfig.EgressSelector.Lookup
	}
	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerTracing) {
		s.Etcd.StorageConfig.Transport.TracerProvider = genericConfig.TracerProvider
	} else {
		s.Etcd.StorageConfig.Transport.TracerProvider = oteltrace.NewNoopTracerProvider()
	}

	storageFactoryConfig := kubeapiserver.NewStorageFactoryConfig()
	storageFactoryConfig.APIResourceConfig = genericConfig.MergedResourceConfig
	//生成storageFactory
	storageFactory, lastErr = storageFactoryConfig.Complete(s.Etcd).New()
	if lastErr != nil {
		return
	}
	if lastErr = s.Etcd.ApplyWithStorageFactoryTo(storageFactory, genericConfig); lastErr != nil {
		return
	}

	ctx := wait.ContextForChannel(genericConfig.DrainedNotify())

	// Authentication.ApplyTo requires already applied OpenAPIConfig and EgressSelector if present
	//生成Authentication
	if lastErr = s.Authentication.ApplyTo(ctx, &genericConfig.Authentication, genericConfig.SecureServing, genericConfig.EgressSelector, genericConfig.OpenAPIConfig, genericConfig.OpenAPIV3Config, clientgoExternalClient, versionedInformers, genericConfig.APIServerID); lastErr != nil {
		return
	}

	var enablesRBAC bool
	genericConfig.Authorization.Authorizer, genericConfig.RuleResolver, enablesRBAC, err = BuildAuthorizer(
		ctx,
		s,
		genericConfig.EgressSelector,
		genericConfig.APIServerID,
		versionedInformers,
	)
	if err != nil {
		lastErr = fmt.Errorf("invalid authorization config: %v", err)
		return
	}
	if s.Authorization != nil && !enablesRBAC {
		genericConfig.DisabledPostStartHooks.Insert(rbacrest.PostStartHookName)
	}
	//生成Audit
	lastErr = s.Audit.ApplyTo(genericConfig)
	if lastErr != nil {
		return
	}
	//生成AggregatedDiscoveryGroupManager
	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.AggregatedDiscoveryEndpoint) {
		genericConfig.AggregatedDiscoveryGroupManager = aggregated.NewResourceManager("apis")
	}

	return
}

```

#### 3.1.1.1 genericConfig 

>其中genericConfig.MergedResourceConfig用于设置启用/禁用GV（资源组、资源版本）及其Resource （资源）。如果未在命令行参数中指定启用/禁用的GV，则通过instance.DefaultAPIResourceConfigSource启用默认设置的GV及其资源。instance.DefaultAPIResourceConfigSource将启用资源版本为Stable和Beta的资源，默认不启用Alpha资源版本的资源。通过EnableVersions函数启用指定资源，而通过DisableVersions函数禁用指定资源，代码示例如下：
>
>**通过使用 `kubectl api-versions` 和 `kubectl api-resources` 命令，可以实时查看集群当前启用的 API 组和版本**

```go
  // 1. 生成 genericConfig,  用于决定k8s开启哪些资源
	genericConfig = genericapiserver.NewConfig(legacyscheme.Codecs)
	genericConfig.MergedResourceConfig = master.DefaultAPIResourceConfigSource()

pkg\controlplane\instance.go
var (
	// stableAPIGroupVersionsEnabledByDefault is a list of our stable versions.
	stableAPIGroupVersionsEnabledByDefault = []schema.GroupVersion{
		admissionregistrationv1.SchemeGroupVersion,
		apiv1.SchemeGroupVersion,
		appsv1.SchemeGroupVersion,
		authenticationv1.SchemeGroupVersion,
		authorizationapiv1.SchemeGroupVersion,
		autoscalingapiv1.SchemeGroupVersion,
		autoscalingapiv2.SchemeGroupVersion,
		batchapiv1.SchemeGroupVersion,
		certificatesapiv1.SchemeGroupVersion,
		coordinationapiv1.SchemeGroupVersion,
		discoveryv1.SchemeGroupVersion,
		eventsv1.SchemeGroupVersion,
		networkingapiv1.SchemeGroupVersion,
		nodev1.SchemeGroupVersion,
		policyapiv1.SchemeGroupVersion,
		rbacv1.SchemeGroupVersion,
		storageapiv1.SchemeGroupVersion,
		schedulingapiv1.SchemeGroupVersion,
		flowcontrolv1.SchemeGroupVersion,
	}

	// legacyBetaEnabledByDefaultResources is the list of beta resources we enable.  You may only add to this list
	// if your resource is already enabled by default in a beta level we still serve AND there is no stable API for it.
	// see https://github.com/kubernetes/enhancements/tree/master/keps/sig-architecture/3136-beta-apis-off-by-default
	// for more details.
	legacyBetaEnabledByDefaultResources = []schema.GroupVersionResource{
		flowcontrolv1beta3.SchemeGroupVersion.WithResource("flowschemas"),                 // deprecate in 1.29, remove in 1.32
		flowcontrolv1beta3.SchemeGroupVersion.WithResource("prioritylevelconfigurations"), // deprecate in 1.29, remove in 1.32
	}
	// betaAPIGroupVersionsDisabledByDefault is for all future beta groupVersions.
	betaAPIGroupVersionsDisabledByDefault = []schema.GroupVersion{
		admissionregistrationv1beta1.SchemeGroupVersion,
		authenticationv1beta1.SchemeGroupVersion,
		storageapiv1beta1.SchemeGroupVersion,
		flowcontrolv1beta1.SchemeGroupVersion,
		flowcontrolv1beta2.SchemeGroupVersion,
		flowcontrolv1beta3.SchemeGroupVersion,
	}

	// alphaAPIGroupVersionsDisabledByDefault holds the alpha APIs we have.  They are always disabled by default.
	alphaAPIGroupVersionsDisabledByDefault = []schema.GroupVersion{
		admissionregistrationv1alpha1.SchemeGroupVersion,
		apiserverinternalv1alpha1.SchemeGroupVersion,
		authenticationv1alpha1.SchemeGroupVersion,
		resourcev1alpha2.SchemeGroupVersion,
		certificatesv1alpha1.SchemeGroupVersion,
		networkingapiv1alpha1.SchemeGroupVersion,
		storageapiv1alpha1.SchemeGroupVersion,
		svmv1alpha1.SchemeGroupVersion,
	}
)

// DefaultAPIResourceConfigSource returns default configuration for an APIResource.
func DefaultAPIResourceConfigSource() *serverstorage.ResourceConfig {
	ret := serverstorage.NewResourceConfig()
	// NOTE: GroupVersions listed here will be enabled by default. Don't put alpha or beta versions in the list.
	ret.EnableVersions(stableAPIGroupVersionsEnabledByDefault...)

	// disable alpha and beta versions explicitly so we have a full list of what's possible to serve
	ret.DisableVersions(betaAPIGroupVersionsDisabledByDefault...)
	ret.DisableVersions(alphaAPIGroupVersionsDisabledByDefault...)

	// enable the legacy beta resources that were present before stopped serving new beta APIs by default.
	ret.EnableResources(legacyBetaEnabledByDefaultResources...)

	return ret
}
```

> apiserver 通过 --runtime-config 指定支持哪些内置资源。 一般都是api/all=true

```
    --runtime-config mapStringString
                A set of key=value pairs that enable or disable built-in APIs. Supported options are:
                v1=true|false for the core API group
                <group>/<version>=true|false for a specific API group and version (e.g. apps/v1=true)
                api/all=true|false controls all API versions
                api/ga=true|false controls all API versions of the form v[0-9]+
                api/beta=true|false controls all API versions of the form v[0-9]+beta[0-9]+
                api/alpha=true|false controls all API versions of the form v[0-9]+alpha[0-9]+
                api/legacy is deprecated, and will be removed in a future version
```

#### 3.1.1.2 OpenAPI和OpenAPIV3 

```gi
	//生成OpenAPI 和 OpenAPIV3 配置
	genericConfig.OpenAPIConfig = genericapiserver.DefaultOpenAPIConfig(getOpenAPIDefinitions, namer)
	genericConfig.OpenAPIConfig.Info.Title = "Kubernetes"
	genericConfig.OpenAPIV3Config = genericapiserver.DefaultOpenAPIV3Config(getOpenAPIDefinitions, namer)
	genericConfig.OpenAPIV3Config.Info.Title = "Kubernetes"
```

```go
// DefaultOpenAPIV3Config provides the default OpenAPIV3Config used to build the OpenAPI V3 spec
func DefaultOpenAPIV3Config(getDefinitions openapicommon.GetOpenAPIDefinitions, defNamer *apiopenapi.DefinitionNamer) *openapicommon.OpenAPIV3Config {
	defaultConfig := &openapicommon.OpenAPIV3Config{
		IgnorePrefixes: []string{},
		Info: &spec.Info{
			InfoProps: spec.InfoProps{
				Title: "Generic API Server",
			},
		},
		DefaultResponse: &spec3.Response{
			ResponseProps: spec3.ResponseProps{
				Description: "Default Response.",
			},
		},
		GetOperationIDAndTags: apiopenapi.GetOperationIDAndTags,
		GetDefinitionName:     defNamer.GetDefinitionName,
		GetDefinitions:        getDefinitions,
	}
	defaultConfig.Definitions = getDefinitions(func(name string) spec.Ref {
		defName, _ := defaultConfig.GetDefinitionName(name)
		return spec.MustCreateRef("#/components/schemas/" + openapicommon.EscapeJsonPointer(defName))
	})

	return defaultConfig
}
```

其中 generatedopenapi.GetOpenAPIDefinitions 定义了OpenAPIDefinition文件（OpenAPI定义文件），该文件由openapi-gen代码生成器自动生成。

#### 3.1.1.3 storageFactory 

>`storageFactory` 在 Kubernetes API 服务器中负责管理与后端存储（通常是 etcd）的所有交互。这包括数据的存储、检索、更新和删除操作。通过 `storageFactory`，Kubernetes 能够将集群的状态、配置和所有资源信息持久化到 etcd 中。这使得 Kubernetes 集群能够在节点重启或故障后恢复状态：

```go
	// StorageFactory存储（Etcd）配置
    storageFactoryConfig := kubeapiserver.NewStorageFactoryConfig()
	storageFactoryConfig.APIResourceConfig = genericConfig.MergedResourceConfig

	storageFactory, lastErr = storageFactoryConfig.Complete(s.Etcd).New()
	if lastErr != nil {
		return
	}
	if lastErr = s.Etcd.ApplyWithStorageFactoryTo(storageFactory, genericConfig); lastErr != nil {
		return
	}

// Complete completes the StorageFactoryConfig with provided etcdOptions returning completedStorageFactoryConfig.
// This method mutates the receiver (StorageFactoryConfig).  It must never mutate the inputs.
func (c *StorageFactoryConfig) Complete(etcdOptions *serveroptions.EtcdOptions) *completedStorageFactoryConfig {
	c.StorageConfig = etcdOptions.StorageConfig
	c.DefaultStorageMediaType = etcdOptions.DefaultStorageMediaType
	c.EtcdServersOverrides = etcdOptions.EtcdServersOverrides
	return &completedStorageFactoryConfig{c}
}
```

**实例化 StorageFactoryConfig**:

- `kubeapiserver.NewStorageFactoryConfig()` 函数初始化 `StorageFactoryConfig` 对象。这个配置对象包含了与 etcd 交互所需的所有配置信息。

**配置 StorageFactory**:

- `storageFactoryConfig.APIResourceConfig = genericConfig.MergedResourceConfig` 这一行将通用配置中的资源启用信息应用到存储配置中。这确保了只有启用的资源才会被存储和管理。

**完成 StorageFactory 配置**:

- `storageFactoryConfig.Complete(s.Etcd)` 方法接受 etcd 的配置选项并填充 `StorageFactoryConfig` 的其它必要字段，如存储媒体类型、etcd 服务器覆盖等。

**创建 StorageFactory**:

- `storageFactory, lastErr = storageFactoryConfig.Complete(s.Etcd).New()` 这一行基于完成的配置创建一个新的 `StorageFactory` 实例。

**应用 Etcd 配置**:

- `if lastErr = s.Etcd.ApplyWithStorageFactoryTo(storageFactory, genericConfig); lastErr != nil { return }` 这部分代码应用 etcd 的配置到 `StorageFactory`。这可能包

#### 3.1.1.4 clientgoExternalClient

* 生成K8s内部与kube-apiserver交互的客户端，如果没有指定RateLimiter的QPS和Burst，则会默认生成一个**限流器**
* RateLimiter:用于控制对 Kubernetes API 服务器的请求速率，以避免因过多的请求而导致的服务降级或 API 被限流。`QPS` 定义了每秒允许的最大请求数，而 `Burst` 定义了在短时间内允许的最大并发请求数。

```go
	//生成clientgoExternalClient
	clientgoExternalClient, err := clientgoclientset.NewForConfig(kubeClientConfig)
	if err != nil {
		lastErr = fmt.Errorf("failed to create real external clientset: %v", err)
		return
	}



// NewForConfig creates a new Clientset for the given config.
// If config's RateLimiter is not set and QPS and Burst are acceptable,
// NewForConfig will generate a rate-limiter in configShallowCopy.
// NewForConfig is equivalent to NewForConfigAndClient(c, httpClient),
// where httpClient was generated with rest.HTTPClientFor(c).
func NewForConfig(c *rest.Config) (*Clientset, error) {
	configShallowCopy := *c

	if configShallowCopy.UserAgent == "" {
		configShallowCopy.UserAgent = rest.DefaultKubernetesUserAgent()
	}

	// share the transport between all clients
	httpClient, err := rest.HTTPClientFor(&configShallowCopy)
	if err != nil {
		return nil, err
	}

	return NewForConfigAndClient(&configShallowCopy, httpClient)
}
```

#### 3.1.1.5 EgressSelector

* EgressSelector和APIServerTracing都是可开启选项：

* EgressSelector主要就是控制api-server出站流量访问外部服务的网络方式；
* APIServerTracing：主要就是跟踪Api-server请求，比如：可以跟踪api-server外部请求，分析性能和排查问题；

```go
	//生成EgressSelector
	if lastErr = s.EgressSelector.ApplyTo(genericConfig); lastErr != nil {
		return
	}
	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerTracing) {
		if lastErr = s.Traces.ApplyTo(genericConfig.EgressSelector, genericConfig); lastErr != nil {
			return
		}
	}
```

#### 3.1.1.6 Audit

* 通过将 `s.Audit.ApplyTo(genericConfig)` 集成到 Kubernetes API 服务器的启动流程中，确保了所有的 API 操作都会按照预设的审计策略被记录和监控。这是确保 Kubernetes 环境安全、合规并可审计的重要步骤。正确配置和管理审计功能对于维护 Kubernetes 集群的整体安全架构至关重要。

```go
	//生成Audit
	lastErr = s.Audit.ApplyTo(genericConfig)
	if lastErr != nil {
		return
	}
```

#### 3.1.1.7 AggregatedDiscoveryGroupManager

>AggregatedDiscoveryGroupManager负责将第三方api-server整合到/apis组下，它负责这些资源的整合、管理和发现。这使得第三方资源可以无缝地与 Kubernetes 原生资源一起使用。
>
>**AggregatorServer**：作为 API 聚合层，负责将外部独立运行的 API 服务器聚合到 Kubernetes 主 API 服务器之下。这种架构设计使得第三方资源和扩展可以作为原生 Kubernetes 资源被访问。
>
>**AggregatedDiscoveryGroupManager**：具体管理聚合 API 组的发现和展示，保证聚合 API 的可见性和访问性

```go
//生成AggregatedDiscoveryGroupManager	
if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.AggregatedDiscoveryEndpoint) {
		genericConfig.AggregatedDiscoveryGroupManager = aggregated.NewResourceManager("apis")
	}
```

#### 3.1.1.8 Authentication

>kube-apiserver作为Kubernetes集群的请求入口，接收组件与客户端的访问请求，每个请求都需要经过认证（Authentication）、授权（Authorization）及准入控制器（Admission Controller）3个阶段，之后才能真正地操作资源。
>
>kube-apiserver目前提供了9种认证机制，分别是BasicAuth、ClientCA、TokenAuth、BootstrapToken、RequestHeader、WebhookTokenAuth、Anonymous、OIDC、ServiceAccountAuth。每一种认证机制被实例化后会成为认证器（Authenticator），每一个认证器都被封装在http.Handler请求处理函数中，它们接收组件或客户端的请求并认证请求。kube-apiserver通过BuildAuthenticator函数实例化认证器，代码示例如下：

```go
	//认证
	if lastErr = s.Authentication.ApplyTo(ctx, &genericConfig.Authentication, genericConfig.SecureServing, genericConfig.EgressSelector, genericConfig.OpenAPIConfig, genericConfig.OpenAPIV3Config, clientgoExternalClient, versionedInformers, genericConfig.APIServerID); lastErr != nil {
		return
	}

	var enablesRBAC bool
	//授权
	genericConfig.Authorization.Authorizer, genericConfig.RuleResolver, enablesRBAC, err = BuildAuthorizer(
		ctx,
		s,
		genericConfig.EgressSelector,
		genericConfig.APIServerID,
		versionedInformers,
	)
```

## 3.2 Authentication认证

ApplyTo---->authenticatorConfig.New

`authenticatorConfig.New()` 方法创建一个认证器 (`authenticator`)

 **RequestHeader**

- 使用 `headerrequest.NewDynamicVerifyOptionsSecure` 方法创建请求头认证器（`requestHeaderAuthenticator`）。这用于代理认证，例如 API 聚合。

 **ClientCA (客户端证书认证)**

- `x509.NewDynamic` 创建用于客户端证书认证的认证器，允许通过客户端证书验证用户。

**TokenAuth**

- 如果配置了 token 文件（`config.TokenAuthFile`），使用 `newAuthenticatorFromTokenFile` 方法创建基于令牌文件的认证器。

 **ServiceAccountAuth**

- 使用 `newLegacyServiceAccountAuthenticator` 和 `newServiceAccountAuthenticator` 方法创建服务账户令牌认证器，这是 Kubernetes 内部服务账户使用的标准认证方式。

**BootstrapToken**

- 如果启用了引导令牌（`config.BootstrapToken`），使用 `config.BootstrapTokenAuthenticator` 添加到认证器中。

 **OIDC (OpenID Connect)**

- 通过 `newJWTAuthenticator` 方法创建基于 JWT 的 OIDC 认证器。

**WebhookTokenAuth**

- 如果配置了 Webhook 认证文件（`config.WebhookTokenAuthnConfigFile`），使用 `newWebhookTokenAuthenticator` 创建 Webhook 认证器。

 **Anonymous**

- 如果允许匿名访问（`config.Anonymous`），则添加一个匿名认证器。

**BasicAuth**

- 尽管在这个特定代码段中没有直接提及 BasicAuth 的创建，但它通常在 Kubernetes 中通过静态密码文件配置，并在相应的认证配置代码中处理。

authenticators中存放的是已启用的认证器列表。union.New函数将authenticators合并成一个authenticator认证器，实际上将认证器列表存放在union结构的Handlers []authenticator.Request对象中。当客户端请求到达kube-apiserver时，kube-apiserver会遍历认证器列表，尝试执行每个认证器，当有一个认证器返回true时，则认证成功。

```go
authenticator := union.New(authenticators...)
```

Authentication可以通过下面的参数配置，开启上诉的认证。例如：--authentication-token-webhook-config-file 指定认证的webhook配置。一般是和公司的权限认证相关。

```
Authentication flags:

      --anonymous-auth
                Enables anonymous requests to the secure port of the API server. Requests that are not rejected by another authentication method are treated as anonymous
                requests. Anonymous requests have a username of system:anonymous, and a group name of system:unauthenticated. (default true)
      --api-audiences strings
                Identifiers of the API. The service account token authenticator will validate that tokens used against the API are bound to at least one of these audiences.
                If the --service-account-issuer flag is configured and this flag is not, this field defaults to a single element list containing the issuer URL .
      --authentication-token-webhook-cache-ttl duration
                The duration to cache responses from the webhook token authenticator. (default 2m0s)
      --authentication-token-webhook-config-file string
                File with webhook configuration for token authentication in kubeconfig format. The API server will query the remote service to determine authentication for
                bearer tokens.
      --authentication-token-webhook-version string
                The API version of the authentication.k8s.io TokenReview to send to and expect from the webhook. (default "v1beta1")
      --client-ca-file string
                If set, any request presenting a client certificate signed by one of the authorities in the client-ca-file is authenticated with an identity corresponding
                to the CommonName of the client certificate.
      --enable-bootstrap-token-auth
                Enable to allow secrets of type 'bootstrap.kubernetes.io/token' in the 'kube-system' namespace to be used for TLS bootstrapping authentication.
      --oidc-ca-file string
                If set, the OpenID server's certificate will be verified by one of the authorities in the oidc-ca-file, otherwise the host's root CA set will be used.
      --oidc-client-id string
                The client ID for the OpenID Connect client, must be set if oidc-issuer-url is set.
      --oidc-groups-claim string
                If provided, the name of a custom OpenID Connect claim for specifying user groups. The claim value is expected to be a string or array of strings. This flag
                is experimental, please see the authentication documentation for further details.
      --oidc-groups-prefix string
                If provided, all groups will be prefixed with this value to prevent conflicts with other authentication strategies.
      --oidc-issuer-url string
                The URL of the OpenID issuer, only HTTPS scheme will be accepted. If set, it will be used to verify the OIDC JSON Web Token (JWT).
      --oidc-required-claim mapStringString
                A key=value pair that describes a required claim in the ID Token. If set, the claim is verified to be present in the ID Token with a matching value. Repeat
                this flag to specify multiple claims.
      --oidc-signing-algs strings
                Comma-separated list of allowed JOSE asymmetric signing algorithms. JWTs with a 'alg' header value not in this list will be rejected. Values are defined by
                RFC 7518 https://tools.ietf.org/html/rfc7518#section-3.1. (default [RS256])
      --oidc-username-claim string
                The OpenID claim to use as the user name. Note that claims other than the default ('sub') is not guaranteed to be unique and immutable. This flag is
                experimental, please see the authentication documentation for further details. (default "sub")
      --oidc-username-prefix string
                If provided, all usernames will be prefixed with this value. If not provided, username claims other than 'email' are prefixed by the issuer URL to avoid
                clashes. To skip any prefixing, provide the value '-'.
      --requestheader-allowed-names strings
                List of client certificate common names to allow to provide usernames in headers specified by --requestheader-username-headers. If empty, any client
                certificate validated by the authorities in --requestheader-client-ca-file is allowed.
      --requestheader-client-ca-file string
                Root certificate bundle to use to verify client certificates on incoming requests before trusting usernames in headers specified by
                --requestheader-username-headers. WARNING: generally do not depend on authorization being already done for incoming requests.
      --requestheader-extra-headers-prefix strings
                List of request header prefixes to inspect. X-Remote-Extra- is suggested.
      --requestheader-group-headers strings
                List of request headers to inspect for groups. X-Remote-Group is suggested.
      --requestheader-username-headers strings
                List of request headers to inspect for usernames. X-Remote-User is common.
      --service-account-issuer string
                Identifier of the service account token issuer. The issuer will assert this identifier in "iss" claim of issued tokens. This value is a string or URI.
      --service-account-key-file stringArray
                File containing PEM-encoded x509 RSA or ECDSA private or public keys, used to verify ServiceAccount tokens. The specified file can contain multiple keys,
                and the flag can be specified multiple times with different files. If unspecified, --tls-private-key-file is used. Must be specified when
                --service-account-signing-key is provided
      --service-account-lookup
                If true, validate ServiceAccount tokens exist in etcd as part of authentication. (default true)
      --service-account-max-token-expiration duration
                The maximum validity duration of a token created by the service account token issuer. If an otherwise valid TokenRequest with a validity duration larger
                than this value is requested, a token will be issued with a validity duration of this value.
      --token-auth-file string
                If set, the file that will be used to secure the secure port of the API server via token authentication.
```

## 3.3 Authorization授权

BuildAuthorizer-----> authorizationConfig.New

认证和授权的区别在于： 张三发来了一个删除pod的请求。 认证：证明张三是张三， 授权：张三是master，有权限删除这个pod。

在Kubernetes系统组件或客户端请求通过认证阶段之后，会来到授权阶段。kube-apiserver同样支持多种授权机制，并支持同时开启多个授权功能，客户端发起一个请求，在经过授权阶段时，只要有一个授权器通过则授权成功。kube-apiserver目前提供了6种授权机制，分别是AlwaysAllow、AlwaysDeny、Webhook、

Node、ABAC、RBAC。每一种授权机制被实例化后会成为授权器（Authorizer），每一个授权器都被封装在http.Handler请求处理函数中，它们接收组件或客户

端的请求并授权请求。kube-apiserver通过BuildAuthorizer函数实例化授权器，代码示例如下：

```go
//auth授权	
genericConfig.Authorization.Authorizer, genericConfig.RuleResolver, enablesRBAC, err = BuildAuthorizer(
		ctx,
		s,
		genericConfig.EgressSelector,
		genericConfig.APIServerID,
		versionedInformers,
	)
```

**Node 授权器**

- **作用**：Node授权器是为了确保节点只能访问它们自身相关的资源而设计的。它主要用于节点自我管理其资源和监控，确保节点不能越权访问不属于它的信息。
- 实现：
    - 使用节点信息的图结构 (`graph`) 来存储和管理节点和相关资源之间的关系。
    - 在图中注册各种 Kubernetes 资源的事件处理器，这些资源包括节点、Pods、持久卷和卷附件。
    - 如果启用了动态资源分配功能（`DynamicResourceAllocation`），还会处理资源切片。
    - 使用 `node.NewAuthorizer` 创建具体的 Node 授权器实例。

 **ABAC 授权器**

- **作用**：基于属性的访问控制（ABAC）提供了一种简单的声明性访问控制机制，通过属性（如用户、组、资源类型等）来决定是否允许访问。
- 实现：
    - 从指定的策略文件加载 ABAC 策略（`config.PolicyFile`）。
    - 使用 `abac.NewFromFile` 方法创建 ABAC 授权器。如果在加载文件或解析策略时出错，则返回错误。

**RBAC 授权器**

- **作用**：基于角色的访问控制（RBAC）是一种广泛使用的授权策略，通过角色（Role）和角色绑定（RoleBinding）来管理权限。
- 实现：
    - 使用 Kubernetes 的 API 访问角色和角色绑定资源。
    - 创建 RBAC 授权器实例，该实例利用这些角色和绑定来确定请求是否被允许。

>```
>重载时确保一致性，删除所有非 Webhook 类型的授权器类型标记
>```

```go
// New returns the right sort of union of multiple authorizer.Authorizer objects
// based on the authorizationMode or an error.
// stopCh is used to shut down config reload goroutines when the server is shutting down.
func (config Config) New(ctx context.Context, serverID string) (authorizer.Authorizer, authorizer.RuleResolver, error) {
      ....................................................
// Build and store authorizers which will persist across reloads
	for _, configuredAuthorizer := range config.AuthorizationConfiguration.Authorizers {
		seenTypes.Insert(configuredAuthorizer.Type)

		// Keep cases in sync with constant list in k8s.io/kubernetes/pkg/kubeapiserver/authorizer/modes/modes.go.
		switch configuredAuthorizer.Type {
		case authzconfig.AuthorizerType(modes.ModeNode):
			var slices resourcev1alpha2informers.ResourceSliceInformer
			if utilfeature.DefaultFeatureGate.Enabled(features.DynamicResourceAllocation) {
				slices = config.VersionedInformerFactory.Resource().V1alpha2().ResourceSlices()
			}
			node.RegisterMetrics()
			graph := node.NewGraph()
			node.AddGraphEventHandlers(
				graph,
				config.VersionedInformerFactory.Core().V1().Nodes(),
				config.VersionedInformerFactory.Core().V1().Pods(),
				config.VersionedInformerFactory.Core().V1().PersistentVolumes(),
				config.VersionedInformerFactory.Storage().V1().VolumeAttachments(),
				slices, // Nil check in AddGraphEventHandlers can be removed when always creating this.
			)
			r.nodeAuthorizer = node.NewAuthorizer(graph, nodeidentifier.NewDefaultNodeIdentifier(), bootstrappolicy.NodeRules())

		case authzconfig.AuthorizerType(modes.ModeABAC):
			var err error
			r.abacAuthorizer, err = abac.NewFromFile(config.PolicyFile)
			if err != nil {
				return nil, nil, err
			}
		case authzconfig.AuthorizerType(modes.ModeRBAC):
			r.rbacAuthorizer = rbac.New(
				&rbac.RoleGetter{Lister: config.VersionedInformerFactory.Rbac().V1().Roles().Lister()},
				&rbac.RoleBindingLister{Lister: config.VersionedInformerFactory.Rbac().V1().RoleBindings().Lister()},
				&rbac.ClusterRoleGetter{Lister: config.VersionedInformerFactory.Rbac().V1().ClusterRoles().Lister()},
				&rbac.ClusterRoleBindingLister{Lister: config.VersionedInformerFactory.Rbac().V1().ClusterRoleBindings().Lister()},
			)
		}
	}
    ...........................
}

```

authorizationConfig.New函数在实例化授权器的过程中，会根据--authorization-mode参数的配置信息（由flags命令行参数传入）决定是否启用授权方法，并对

启用的授权方法生成对应的HTTP Handler函数，最后将已启用的授权器合并到r.current数组对象中，代码示例如下：

```go
// BuildAuthorizer constructs the authorizer. If authorization is not set in s, it returns nil, nil, false, nil
func BuildAuthorizer(ctx context.Context, s controlplaneapiserver.CompletedOptions, egressSelector *egressselector.EgressSelector, apiserverID string, versionedInformers clientgoinformers.SharedInformerFactory) (authorizer.Authorizer, authorizer.RuleResolver, bool, error) {
	authorizationConfig, err := s.Authorization.ToAuthorizationConfig(versionedInformers)

	authorizer, ruleResolver, err := authorizationConfig.New(ctx, apiserverID)

	return authorizer, ruleResolver, enablesRBAC, err
}

	// Construct the authorizers / ruleResolvers for the given configuration
	authorizer, ruleResolver, err := r.newForConfig(r.initialConfig.AuthorizationConfiguration)
	if err != nil {
		return nil, nil, err
	}

	r.current.Store(&authorizerResolver{
		authorizer:   authorizer,
		ruleResolver: ruleResolver,
	})
```

authorizers中存放的是已启用的授权器列表，ruleResolvers中存放的是已启用的授权器规则解析器，实际上放入r.current，当客户端请求到达kube-apiserver时，kube-apiserver会遍历授权器列表，并按照顺序执行授权

```go
//r.current是一个安全的多线程存储结构
type reloadableAuthorizerResolver struct {	
	current atomic.Pointer[authorizerResolver]
}

type authorizerResolver struct {
	authorizer   authorizer.Authorizer
	ruleResolver authorizer.RuleResolver
}

```

器，排在前面的授权器具有更高的优先级（允许或拒绝请求）。客户端发起一个请求，在经过授权阶段时，只要有一个授权器通过，则授权成功。

**Kube-apiserver**通过authorization-mode指定支持哪几种授权模式。

**注意，这个是访问安全端口时候用到的，如果访问的是非安全端口，是不同通过授权验证的！！！**

```go
--authorization-mode strings
                Ordered list of plug-ins to do authorization on secure port. Comma-delimited list of: AlwaysAllow,AlwaysDeny,ABAC,Webhook,RBAC,Node. (default [AlwaysAllow])
```

## 3.4 Admission准入控制器配置

Kubernetes系统组件或客户端请求通过授权阶段之后，会来到准入控制器阶段，它会在认证和授权请求之后，对象被持久化之前(写入Etcd），拦截kube-apiserver的请求，

拦截后的请求进入准入控制器中处理，对请求的资源对象进行自定义（校验、修改或拒绝）等操作。kube-apiserver支持多种准入控制器机制，并支持同时开启多

个准入控制器功能，如果开启了多个准入控制器，则按照顺序执行准入控制器。

**AlwaysPullImages**

**CertificateApproval**

**CertificateSigning**

**CertificateSubjectRestriction**

**DefaultIngressClass**

**DefaultStorageClass**

**DefaultTolerationSeconds**

**LimitRanger**

**MutatingAdmissionWebhook**

**NamespaceLifecycle**

**PersistentVolumeClaimResize**

**PodSecurity**

**Priority**

**ResourceQuota**

**RuntimeClass**

**ServiceAccount**

**StorageObjectInUseProtection**

**TaintNodesByCondition**

**ValidatingAdmissionPolicy**

**ValidatingAdmissionWebhook**

```go
	admissionConfig := &kubeapiserveradmission.Config{
		ExternalInformers:    versionedInformers,
		LoopbackClientConfig: genericConfig.LoopbackClientConfig,
		CloudConfigFile:      opts.CloudProvider.CloudConfigFile,
	}
	serviceResolver := buildServiceResolver(opts.EnableAggregatorRouting, genericConfig.LoopbackClientConfig.Host, versionedInformers)
	pluginInitializers, err := admissionConfig.New(proxyTransport, genericConfig.EgressSelector, serviceResolver, genericConfig.TracerProvider)
	if err != nil {
		return nil, nil, nil, fmt.Errorf("failed to create admission plugin initializer: %v", err)
	}
	clientgoExternalClient, err := clientset.NewForConfig(genericConfig.LoopbackClientConfig)
	if err != nil {
		return nil, nil, nil, fmt.Errorf("failed to create real client-go external client: %w", err)
	}
	dynamicExternalClient, err := dynamic.NewForConfig(genericConfig.LoopbackClientConfig)
	if err != nil {
		return nil, nil, nil, fmt.Errorf("failed to create real dynamic external client: %w", err)
	}
	//应用 admission 配置
	err = opts.Admission.ApplyTo(
		genericConfig,
		versionedInformers,
		clientgoExternalClient,
		dynamicExternalClient,
		utilfeature.DefaultFeatureGate,
		pluginInitializers...)
	if err != nil {
		return nil, nil, nil, fmt.Errorf("failed to apply admission: %w", err)
	}
```

kube-apiserver在启动时注册所有准入控制器，准入控制器通过Plugins数据结构统一注册、存放、管理所有的准入控制器。Plugins数据结构如下：

```go
// Factory is a function that returns an Interface for admission decisions.
// The config parameter provides an io.Reader handler to the factory in
// order to load specific configurations. If no configuration is provided
// the parameter is nil.
type Factory func(config io.Reader) (Interface, error)

type Plugins struct {
	lock     sync.Mutex
	registry map[string]Factory
}
```

Plugins数据结构字段说明如下。

● registry：以键值对形式存放插件，key为准入控制器的名称，例如AlwaysPullImages、LimitRanger等；value为对应的准入控制器名称的代码实现。

● lock：用于保护registry字段的并发一致性。

其中Factory为准入控制器实现的接口定义，它接收准入控制器的config配置信息，通过--admission-control-config-

file参数指定准入控制器的配置文件，返回准入控制器的插件实现。Plugins数据结构提供了Register方法，为外部提供了准入控制器的注册方法。

kube-apiserver提供了20种准入控制器，kube-apiserver组件在启动时分别在两个位置注册它们，代码示例如下：

```go
// RegisterAllAdmissionPlugins registers all admission plugins
func RegisterAllAdmissionPlugins(plugins *admission.Plugins) {
	lifecycle.Register(plugins)
	validatingwebhook.Register(plugins)
	mutatingwebhook.Register(plugins)
	validatingadmissionpolicy.Register(plugins)
}

```

每个准入控制器都实现了Register方法，通过Register方法可以在Plugins数据结构中注册当前准入控制器。以AlwaysPullImages准入控制器为例，注册方法代码示例如下：

```go
// PluginName indicates name of admission plugin.
const PluginName = "ImagePolicyWebhook"
const ephemeralcontainers = "ephemeralcontainers"

// AuditKeyPrefix is used as the prefix for all audit keys handled by this
// pluggin. Some well known suffixes are listed below.
var AuditKeyPrefix = strings.ToLower(PluginName) + ".image-policy.k8s.io/"

const (
	// ImagePolicyFailedOpenKeySuffix in an annotation indicates the image
	// review failed open when the image policy webhook backend connection
	// failed.
	ImagePolicyFailedOpenKeySuffix string = "failed-open"

	// ImagePolicyAuditRequiredKeySuffix in an annotation indicates the pod
	// should be audited.
	ImagePolicyAuditRequiredKeySuffix string = "audit-required"
)

var (
	groupVersions = []schema.GroupVersion{v1alpha1.SchemeGroupVersion}
)

// Register registers a plugin
func Register(plugins *admission.Plugins) {
	plugins.Register(PluginName, func(config io.Reader) (admission.Interface, error) {
		newImagePolicyWebhook, err := NewImagePolicyWebhook(config)
		if err != nil {
			return nil, err
		}
		return newImagePolicyWebhook, nil
	})
}
```

> 这个是通过--enable-admission-plugins指定，例如：

```go

--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,ServiceAccount,ResourceQuota,DefaultStorageClass,Priority,MutatingAdmissionWebhook,ValidatingAdmissionWebhook

```



