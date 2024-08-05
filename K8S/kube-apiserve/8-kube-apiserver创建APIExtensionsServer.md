**本章重点：**分析第四个流程，创建APIExtensionsServer

kube-apiserver整体启动流程如下：

（1）资源注册。

（2）Cobra命令行参数解析

（3）创建APIServer通用配置

（4）创建APIExtensionsServer

（5）创建KubeAPIServer

（6）创建AggregatorServer

（7）启动HTTP服务。

（8）启动HTTPS服务

---

# 1.APIExtensionsServer的通用配置

>APIExtensionsServer的配置其实就是kube-apiserver通用的配置，然后+一个ExtraConfig，ExtraConfig主要是CRDRESTOptionsGetter、MasterCount、AuthResolverWrapper、ServiceResolver

# 2.创建APIExtensionsServer

CreateServerChain------->ApiExtensions.New

## 2.1 ApiExtensions.New

```go
// New returns a new instance of CustomResourceDefinitions from the given config.
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*CustomResourceDefinitions, error) {
	//创建一个名为 apiextensions-apiserver 的通用 API 服务器
	genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
	if err != nil {
		return nil, err
	}

	// hasCRDInformerSyncedSignal is closed when the CRD informer this server uses has been fully synchronized.
	// It ensures that requests to potential custom resource endpoints while the server hasn't installed all known HTTP paths get a 503 error instead of a 404
	//创建一个信号通道，用于指示 CRD 信息同步是否完成
	hasCRDInformerSyncedSignal := make(chan struct{})
	//在服务器上注册一个完成信号，当 CRD informer 同步完成时关闭通道
	if err := genericServer.RegisterMuxAndDiscoveryCompleteSignal("CRDInformerHasNotSynced", hasCRDInformerSyncedSignal); err != nil {
		return nil, err
	}
	//创建一个包含通用服务器的 CRD 服务器实例，实例化该对象后才能注册CRD下的资源。
	s := &CustomResourceDefinitions{
		GenericAPIServer: genericServer,
	}

	apiResourceConfig := c.GenericConfig.MergedResourceConfig
	///配置 API 组信息
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(apiextensions.GroupName, Scheme, metav1.ParameterCodec, Codecs)
	storage := map[string]rest.Storage{}
	// customresourcedefinitions
	//条件性启用 CRD 存储
	if resource := "customresourcedefinitions"; apiResourceConfig.ResourceEnabled(v1.SchemeGroupVersion.WithResource(resource)) {
		customResourceDefinitionStorage, err := customresourcedefinition.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
		if err != nil {
			return nil, err
		}
		storage[resource] = customResourceDefinitionStorage
		storage[resource+"/status"] = customresourcedefinition.NewStatusREST(Scheme, customResourceDefinitionStorage)
	}
	if len(storage) > 0 {
		apiGroupInfo.VersionedResourcesStorageMap[v1.SchemeGroupVersion.Version] = storage
	}
	//安装 API 组
	if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
		return nil, err
	}

	crdClient, err := clientset.NewForConfig(s.GenericAPIServer.LoopbackClientConfig)
	if err != nil {
		// it's really bad that this is leaking here, but until we can fix the test (which I'm pretty sure isn't even testing what it wants to test),
		// we need to be able to move forward
		return nil, fmt.Errorf("failed to create clientset: %v", err)
	}
	s.Informers = externalinformers.NewSharedInformerFactory(crdClient, 5*time.Minute)

	delegateHandler := delegationTarget.UnprotectedHandler()
	if delegateHandler == nil {
		delegateHandler = http.NotFoundHandler()
	}

	versionDiscoveryHandler := &versionDiscoveryHandler{
		discovery: map[schema.GroupVersion]*discovery.APIVersionHandler{},
		delegate:  delegateHandler,
	}
	groupDiscoveryHandler := &groupDiscoveryHandler{
		discovery: map[string]*discovery.APIGroupHandler{},
		delegate:  delegateHandler,
	}
	establishingController := establish.NewEstablishingController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
	crdHandler, err := NewCustomResourceDefinitionHandler(
		versionDiscoveryHandler,
		groupDiscoveryHandler,
		s.Informers.Apiextensions().V1().CustomResourceDefinitions(),
		delegateHandler,
		c.ExtraConfig.CRDRESTOptionsGetter,
		c.GenericConfig.AdmissionControl,
		establishingController,
		c.ExtraConfig.ServiceResolver,
		c.ExtraConfig.AuthResolverWrapper,
		c.ExtraConfig.MasterCount,
		s.GenericAPIServer.Authorizer,
		c.GenericConfig.RequestTimeout,
		time.Duration(c.GenericConfig.MinRequestTimeout)*time.Second,
		apiGroupInfo.StaticOpenAPISpec,
		c.GenericConfig.MaxRequestBodyBytes,
	)
	if err != nil {
		return nil, err
	}
	s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", crdHandler)
	s.GenericAPIServer.Handler.NonGoRestfulMux.HandlePrefix("/apis/", crdHandler)
	s.GenericAPIServer.RegisterDestroyFunc(crdHandler.destroy)

	aggregatedDiscoveryManager := genericServer.AggregatedDiscoveryGroupManager
	if aggregatedDiscoveryManager != nil {
		aggregatedDiscoveryManager = aggregatedDiscoveryManager.WithSource(aggregated.CRDSource)
	}
	discoveryController := NewDiscoveryController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), versionDiscoveryHandler, groupDiscoveryHandler, aggregatedDiscoveryManager)
	namingController := status.NewNamingConditionController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
	nonStructuralSchemaController := nonstructuralschema.NewConditionController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
	apiApprovalController := apiapproval.NewKubernetesAPIApprovalPolicyConformantConditionController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
	finalizingController := finalizer.NewCRDFinalizer(
		s.Informers.Apiextensions().V1().CustomResourceDefinitions(),
		crdClient.ApiextensionsV1(),
		crdHandler,
	)

	s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-informers", func(context genericapiserver.PostStartHookContext) error {
		s.Informers.Start(context.StopCh)
		return nil
	})
	//启动CRDcontroller
	s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-controllers", func(context genericapiserver.PostStartHookContext) error {
		// OpenAPIVersionedService and StaticOpenAPISpec are populated in generic apiserver PrepareRun().
		// Together they serve the /openapi/v2 endpoint on a generic apiserver. A generic apiserver may
		// choose to not enable OpenAPI by having null openAPIConfig, and thus OpenAPIVersionedService
		// and StaticOpenAPISpec are both null. In that case we don't run the CRD OpenAPI controller.
		if s.GenericAPIServer.StaticOpenAPISpec != nil {
			if s.GenericAPIServer.OpenAPIVersionedService != nil {
				openapiController := openapicontroller.NewController(s.Informers.Apiextensions().V1().CustomResourceDefinitions())
				go openapiController.Run(s.GenericAPIServer.StaticOpenAPISpec, s.GenericAPIServer.OpenAPIVersionedService, context.StopCh)
			}

			if s.GenericAPIServer.OpenAPIV3VersionedService != nil {
				openapiv3Controller := openapiv3controller.NewController(s.Informers.Apiextensions().V1().CustomResourceDefinitions())
				go openapiv3Controller.Run(s.GenericAPIServer.OpenAPIV3VersionedService, context.StopCh)
			}
		}

		go namingController.Run(context.StopCh)
		go establishingController.Run(context.StopCh)
		go nonStructuralSchemaController.Run(5, context.StopCh)
		go apiApprovalController.Run(5, context.StopCh)
		go finalizingController.Run(5, context.StopCh)

		discoverySyncedCh := make(chan struct{})
		go discoveryController.Run(context.StopCh, discoverySyncedCh)
		select {
		case <-context.StopCh:
		case <-discoverySyncedCh:
		}

		return nil
	})
	// we don't want to report healthy until we can handle all CRDs that have already been registered.  Waiting for the informer
	// to sync makes sure that the lister will be valid before we begin.  There may still be races for CRDs added after startup,
	// but we won't go healthy until we can handle the ones already present.
	s.GenericAPIServer.AddPostStartHookOrDie("crd-informer-synced", func(context genericapiserver.PostStartHookContext) error {
		return wait.PollImmediateUntil(100*time.Millisecond, func() (bool, error) {
			if s.Informers.Apiextensions().V1().CustomResourceDefinitions().Informer().HasSynced() {
				close(hasCRDInformerSyncedSignal)
				return true, nil
			}
			return false, nil
		}, context.StopCh)
	})

	return s, nil
}

```

## 2.2 创建GenericAPIServer

```go
	//创建一个名为 apiextensions-apiserver 的通用 API 服务器
	genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
	if err != nil {
		return nil, err
	}

```

> 在创建另外两个apiserver的时候，都用到了这个，最后再统一分析。

## 2.3 实例化CustomResourceDefinitions

```
s := &CustomResourceDefinitions{
		GenericAPIServer: genericServer,
	}
	

// CustomResourceDefinitions进行了另外的封装
type CustomResourceDefinitions struct {
	GenericAPIServer *genericapiserver.GenericAPIServer

	// provided for easier embedding
	Informers internalinformers.SharedInformerFactory
}
```

>APIExtensionsServer（API扩展服务）通过CustomResourceDefinitions对象进行管理，实例化该对象后才能注册APIExtensionsServer下的资源。

## 2.4 配置APIGroupInfo

```go
	///配置 API 组信息
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(apiextensions.GroupName, Scheme, metav1.ParameterCodec, Codecs)

// Info about an API group.
type APIGroupInfo struct {
	PrioritizedVersions []schema.GroupVersion
	// Info about the resources in this group. It's a map from version to resource to the storage.
    存储资源与资源存储对象的对应关系，用于installRest的时候用
	VersionedResourcesStorageMap map[string]map[string]rest.Storage
	// OptionsExternalVersion controls the APIVersion used for common objects in the
	// schema like api.Status, api.DeleteOptions, and metav1.ListOptions. Other implementors may
	// define a version "v1beta1" but want to use the Kubernetes "v1" internal objects.
	// If nil, defaults to groupMeta.GroupVersion.
	// TODO: Remove this when https://github.com/kubernetes/kubernetes/issues/19018 is fixed.
	OptionsExternalVersion *schema.GroupVersion
	// MetaGroupVersion defaults to "meta.k8s.io/v1" and is the scheme group version used to decode
	// common API implementations like ListOptions. Future changes will allow this to vary by group
	// version (for when the inevitable meta/v2 group emerges).
	MetaGroupVersion *schema.GroupVersion

	// Scheme includes all of the types used by this group and how to convert between them (or
	// to convert objects from outside of this group that are accepted in this API).
	// TODO: replace with interfaces
	Scheme *runtime.Scheme
	// NegotiatedSerializer controls how this group encodes and decodes data
	NegotiatedSerializer runtime.NegotiatedSerializer
	// ParameterCodec performs conversions for query parameters passed to API calls
	ParameterCodec runtime.ParameterCodec

	// StaticOpenAPISpec is the spec derived from the definitions of all resources installed together.
	// It is set during InstallAPIGroups, InstallAPIGroup, and InstallLegacyAPIGroup.
	StaticOpenAPISpec map[string]*spec.Schema
}
```

APIGroupInfo对象用于描述资源组信息，其中该对象的VersionedResourcesStorageMap字段用于存储资源与资源存储对象的对应关系，其表现形式为map[string]map[string]rest.Storage（即<资源版本>/<资源>/<资源存储对象>）;

例如CustomResourceDefinitions资源与资源存储对象的映射关系是v1beta1/customresourcedefinitions/customResourceDefintionStorage。

在实例化APIGroupInfo对象后，完成其资源与资源存储对象的映射，APIExtensionsServer会先判断apiextensions.k8s.io/v1资源

组/资源版本是否已启用，如果其已启用，则将该资源组、资源版本下的资源与资源存储对象进行映射并存储至APIGroupInfo对象的

VersionedResourcesStorageMap字段中。每个资源（包括子资源）都通过类似于NewREST的函数创建资源存储对象（即

RESTStorage）。kube-apiserver将RESTStorage封装成HTTP Handler函数，资源存储对象以RESTful的方式运行，一个RESTStorage对

象负责一个资源的增、删、改、查操作。当操作CustomResourceDefinitions资源数据时，通过对应的RESTStorage资源存储对象与

genericregistry.Store进行交互。



**提示：** 一个资源组对应一个APIGroupInfo对象，每个资源（包括子资源）对应一个资源存储对象。

admisionregistration.k8s.io就是一个group。

```go
[root@k8s-master ~]# kubectl api-versions    表现形式为<group>/<version>.
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
apps/v1beta1
apps/v1beta2
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1beta1
events.k8s.io/v1beta1
extensions/v1beta1
networking.k8s.io/v1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```

## 2.5 InstallAPIGroup安装APIGroup

```go
	//安装 API 组
	if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
		return nil, err
	}
```

InstallAPIGroup注册APIGroupInfo的过程非常重要，将APIGroupInfo对象中的<资源组>/<资源版本>/<资源>/<子资源>（包括资源存储

对象）注册到APIExtensionsServerHandler函数。其过程是遍历APIGroupInfo，将<资源组>/<资源版本>/<资源名称>映射到HTTP PATH

请求路径，通过InstallREST函数将资源存储对象作为资源的Handlers方法，最后使用go-restful的ws.Route将定义好的请求路径和

Handlers方法添加路由到go-restful中。整个过程为InstallAPIGroup→s.InstallAPIGroups→InstallAPIGroups---->installAPIResources---->InstallREST，代码示例如下：

```go
// InstallREST registers the REST handlers (storage, watch, proxy and redirect) into a restful Container.
// It is expected that the provided path root prefix will serve all operations. Root MUST NOT end
// in a slash.
func (g *APIGroupVersion) InstallREST(container *restful.Container) ([]apidiscoveryv2.APIResourceDiscovery, []*storageversion.ResourceInfo, error) {
	prefix := path.Join(g.Root, g.GroupVersion.Group, g.GroupVersion.Version)
	installer := &APIInstaller{
		group:             g,
		prefix:            prefix,
		minRequestTimeout: g.MinRequestTimeout,
	}

	apiResources, resourceInfos, ws, registrationErrors := installer.Install()
	versionDiscoveryHandler := discovery.NewAPIVersionHandler(g.Serializer, g.GroupVersion, staticLister{apiResources})
	versionDiscoveryHandler.AddToWebService(ws)
	container.Add(ws)
	aggregatedDiscoveryResources, err := ConvertGroupVersionIntoToDiscovery(apiResources)
	if err != nil {
		registrationErrors = append(registrationErrors, err)
	}
	return aggregatedDiscoveryResources, removeNonPersistedResources(resourceInfos), utilerrors.NewAggregate(registrationErrors)
}
```

InstallREST函数接收restful.Container指针对象。安装过程分为4步，分别介绍如下。

（1）prefix定义了HTTP PATH请求路径，其表现形式为//（即/apis/apiextensions.k8s.io/v1）。

（2）实例化APIInstaller安装器。

（3）在installer.Install安装器内部创建一个go-restful WebService，然后通过a.registerResourceHandlers函数，为资源注册对应的

Handlers方法（即资源存储对象Resource Storage），完成资源与资源Handlers方法的绑定并为go-restfulWebService添加该路由。

（4）最后通过container.Add函数将WebService添加到go-restful Container中。APIExtensionsServer负责管理apiextensions.k8s.io资

源组下的所有资源，该资源有v1beta1版本。通过访问http://127.0.0.1:8080/apis/apiextensions.k8s.io/v1获得该资源/子资源的详细信

息，命令示例如下：

```
# curl http://127.0.0.1:8080/apis/apiextensions.k8s.io/v1  ##需要替换成实际的ip,port
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "apiextensions.k8s.io/v1",
  "resources": [
    {
      "name": "customresourcedefinitions",
      "singularName": "",
      "namespaced": false,
      "kind": "CustomResourceDefinition",
      "verbs": [
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ],
      "shortNames": [
        "crd",
        "crds"
      ],
      "storageVersionHash": "jfWCUB31mvA="
    },
    {
      "name": "customresourcedefinitions/status",
      "singularName": "",
      "namespaced": false,
      "kind": "CustomResourceDefinition",
      "verbs": [
        "get",
        "patch",
        "update"
      ]
    }
  ]
}
```





```
查看具体某个crd的信息。 khchecks.comcast.gihub.io就是一个crd
#curl http://127.0.0.1:8080/apis/apiextensions.k8s.io/v1/customresourcedefinitions/khchecks.comcast.gihub.io
{
  "kind": "CustomResourceDefinition",
  "apiVersion": "apiextensions.k8s.io/v1",
  "metadata": {
    "name": "khchecks.comcast.github.io",
    "selfLink": "/apis/apiextensions.k8s.io/v1/customresourcedefinitions/khchecks.comcast.github.io",
    "uid": "e115a743-5c79-4aa9-8b16-016b852e9c46",
    "resourceVersion": "197092612",
    "generation": 1,
    "creationTimestamp": "2021-04-13T10:00:14Z"
  },
  "spec": {
    "group": "comcast.github.io",
    "names": {
      "plural": "khchecks",
      "singular": "khcheck",
      "shortNames": [
        "khc"
      ],
      "kind": "KuberhealthyCheck",
      "listKind": "KuberhealthyCheckList"
    },
    "scope": "Namespaced",
    "versions": [
      {
        "name": "v1",
        "served": true,
        "storage": true
      }
    ],
    "conversion": {
      "strategy": "None"
    },
    "preserveUnknownFields": true
  },
  "status": {
    "conditions": [
      {
        "type": "NamesAccepted",
        "status": "True",
        "lastTransitionTime": "2021-04-13T10:00:14Z",
        "reason": "NoConflicts",
        "message": "no conflicts found"
      },
      {
        "type": "Established",
        "status": "True",
        "lastTransitionTime": "2021-04-13T10:00:19Z",
        "reason": "InitialNamesAccepted",
        "message": "the initial names have been accepted"
      }
    ],
    "acceptedNames": {
      "plural": "khchecks",
      "singular": "khcheck",
      "shortNames": [
        "khc"
      ],
      "kind": "KuberhealthyCheck",
      "listKind": "KuberhealthyCheckList"
    },
    "storedVersions": [
      "v1"
    ]
  }
}
```

## 2.6 启动crdController

```go
		go namingController.Run(context.StopCh)
		go establishingController.Run(context.StopCh)
		go nonStructuralSchemaController.Run(5, context.StopCh)
		go apiApprovalController.Run(5, context.StopCh)
		go finalizingController.Run(5, context.StopCh)
```

**`namingController`**:

- 负责处理和验证 Kubernetes 资源（如 CRDs）的命名规范。
- 启动命令: `go namingController.Run(context.StopCh)`

**`establishingController`**:

- 负责将 CRDs 的状态更新为 “Established”，这意味着 CRD 已准备好被用于创建相应的自定义资源。
- 启动命令: `go establishingController.Run(context.StopCh)`

**`nonStructuralSchemaController`**:

- 负责检查 CRDs 的模式是否符合 Kubernetes 的结构化模式要求。
- 启动命令: `go nonStructuralSchemaController.Run(5, context.StopCh)`

**`apiApprovalController`**:

- 负责处理 CRDs 的 API 审批流程，确保符合群组和版本的管理政策。
- 启动命令: `go apiApprovalController.Run(5, context.StopCh)`

**`finalizingController`**:

- 负责在删除 CRD 时执行必要的清理工作，比如处理 finalizers。
- 启动命令: `go finalizingController.Run(5, context.StopCh)`

> namingController.Run
>
> ### 步骤 1: 检查并处理 CRD 的删除
>
> - **检索 CRD**: 通过 `crdLister.Get(key)` 方法尝试获取指定的 CRD。
> - **处理 CRD 删除**: 如果返回 `IsNotFound` 错误，说明 CRD 已被删除。这时，函数会调用 `requeueAllOtherGroupCRDs(key)` 重新排队该组中的所有其他 CRD，以重新考虑它们的命名情况。
>
> ### 步骤 2: 跳过无需更新的情况
>
> - **检查是否需要更新**：通过比较 `Spec.Names` 和 `Status.AcceptedNames`。如果它们相等，说明没有必要更新状态，因此函数将直接返回。
>
> ### 步骤 3: 更新 CRD 状态
>
> - **计算新的命名和条件**：通过调用 `calculateNamesAndConditions` 计算出应接受的命名和相关条件。
> - **检查是否需要更改状态**：如果新计算的 `AcceptedNames` 和 `NamesAccepted` 条件与当前状态相同，则不进行更新。
> - **更新 CRD 状态**：如果状态需要更新，会创建 CRD 的副本并更新其 `AcceptedNames` 和相关条件，然后调用 `UpdateStatus` 将更改推送到 API 服务器。
>
> ### 步骤 4: 后续处理
>
> - **处理更新冲突**：如果在更新状态时遇到 `NotFound` 或 `Conflict` 错误，将返回 nil，因为 CRD 可能在同时被其他操作更改或删除。
> - **更新 Mutation Cache**：成功更新后，将新的 CRD 状态添加到 mutation cache 中。
> - **重新排队同组其他 CRD**：更新成功后，再次调用 `requeueAllOtherGroupCRDs(key)` 为了确保同组的其他 CRD 能考虑到可能由于命名冲突释放的名字。

```go

func (c *NamingConditionController) sync(key string) error {
	inCustomResourceDefinition, err := c.crdLister.Get(key)
	if apierrors.IsNotFound(err) {
		// CRD was deleted and has freed its names.
		// Reconsider all other CRDs in the same group.
		if err := c.requeueAllOtherGroupCRDs(key); err != nil {
			return err
		}
		return nil
	}
	if err != nil {
		return err
	}

	// Skip checking names if Spec and Status names are same.
	if equality.Semantic.DeepEqual(inCustomResourceDefinition.Spec.Names, inCustomResourceDefinition.Status.AcceptedNames) {
		return nil
	}

	acceptedNames, namingCondition, establishedCondition := c.calculateNamesAndConditions(inCustomResourceDefinition)

	// nothing to do if accepted names and NamesAccepted condition didn't change
	if reflect.DeepEqual(inCustomResourceDefinition.Status.AcceptedNames, acceptedNames) &&
		apiextensionshelpers.IsCRDConditionEquivalent(&namingCondition, apiextensionshelpers.FindCRDCondition(inCustomResourceDefinition, apiextensionsv1.NamesAccepted)) {
		return nil
	}

	crd := inCustomResourceDefinition.DeepCopy()
	crd.Status.AcceptedNames = acceptedNames
	apiextensionshelpers.SetCRDCondition(crd, namingCondition)
	apiextensionshelpers.SetCRDCondition(crd, establishedCondition)

	updatedObj, err := c.crdClient.CustomResourceDefinitions().UpdateStatus(context.TODO(), crd, metav1.UpdateOptions{})
	if apierrors.IsNotFound(err) || apierrors.IsConflict(err) {
		// deleted or changed in the meantime, we'll get called again
		return nil
	}
	if err != nil {
		return err
	}

	// if the update was successful, go ahead and add the entry to the mutation cache
	c.crdMutationCache.Mutation(updatedObj)

	// we updated our status, so we may be releasing a name.  When this happens, we need to rekick everything in our group
	// if we fail to rekick, just return as normal.  We'll get everything on a resync
	if err := c.requeueAllOtherGroupCRDs(key); err != nil {
		return err
	}

	return nil
}
```

>establishingController.Run
>
>### 步骤 1: 检索 CRD
>
>- **获取 CRD**：通过 `crdLister.Get(key)` 尝试从本地缓存中获取指定的 CRD。
>- **处理找不到的 CRD**：如果 CRD 不存在（返回 `IsNotFound` 错误），函数返回 `nil`，表示没有找到这个 CRD，无需进一步处理。
>- **处理其他错误**：如果在获取 CRD 时出现其他错误（如 API 调用失败），则直接返回这个错误。
>
>### 步骤 2: 检查 CRD 状态
>
>- **检查条件**：检查 CRD 的 `NamesAccepted` 条件是否为真，且 `Established` 条件也为真。如果这两个条件都满足，说明没有必要进一步更新 CRD 的状态，因此函数返回 `nil`。
>
>### 步骤 3: 更新 CRD 条件
>
>- **复制和设置 CRD 条件**：如果 CRD 需要被标记为 `Established`，函数会创建 CRD 的一个深拷贝，并设置相应的 `Established` 条件。
>- **构造 `Established` 条件**：设置 `Established` 条件的类型为 `Established`，状态为 `True`，原因为 `"InitialNamesAccepted"`，并附加消息 `"the initial names have been accepted"`。
>
>### 步骤 4: 推送更新到 API 服务器
>
>- **更新 CRD 状态**：使用 `crdClient.CustomResourceDefinitions().UpdateStatus()` 将更新后的 CRD 状态推送到 Kubernetes API 服务器。
>
>- 处理更新中的错误
>
>    ：
>
>    - **处理找不到或冲突错误**：如果在更新过程中 CRD 被删除或有冲突（返回 `IsNotFound` 或 `IsConflict` 错误），函数返回 `nil`，因为 CRD 可能已被删除或已被其他操作更改，所以这个更新操作无效。
>    - **处理其他错误**：如果更新 CRD 状态时遇到其他类型的错误，则返回这个错误。

```go

// sync is used to turn CRDs into the Established state.
func (ec *EstablishingController) sync(key string) error {
	cachedCRD, err := ec.crdLister.Get(key)
	if apierrors.IsNotFound(err) {
		return nil
	}
	if err != nil {
		return err
	}

	if !apiextensionshelpers.IsCRDConditionTrue(cachedCRD, apiextensionsv1.NamesAccepted) ||
		apiextensionshelpers.IsCRDConditionTrue(cachedCRD, apiextensionsv1.Established) {
		return nil
	}

	crd := cachedCRD.DeepCopy()
	establishedCondition := apiextensionsv1.CustomResourceDefinitionCondition{
		Type:    apiextensionsv1.Established,
		Status:  apiextensionsv1.ConditionTrue,
		Reason:  "InitialNamesAccepted",
		Message: "the initial names have been accepted",
	}
	apiextensionshelpers.SetCRDCondition(crd, establishedCondition)

	// Update server with new CRD condition.
	_, err = ec.crdClient.CustomResourceDefinitions().UpdateStatus(context.TODO(), crd, metav1.UpdateOptions{})
	if apierrors.IsNotFound(err) || apierrors.IsConflict(err) {
		// deleted or changed in the meantime, we'll get called again
		return nil
	}
	if err != nil {
		return err
	}

	return nil
}

```

>nonStructuralSchemaController.Run
>
>### 步骤 1: 检索 CRD
>
>- **获取 CRD**: 尝试通过 `crdLister.Get(key)` 获取指定的 CRD。
>- **处理未找到的错误**: 如果 CRD 不存在（即 `IsNotFound` 错误），函数返回 `nil`，表示无需进一步操作。
>- **处理其他错误**: 如果在获取 CRD 时出现其他错误，则直接返回这个错误。
>
>### 步骤 2: 避免重复计算
>
>- **锁定和检查**：使用锁来保证线程安全地访问 `lastSeenGeneration` 映射，检查当前 CRD 的版本号（Generation）是否已被处理。
>- **跳过处理**: 如果该 CRD 的当前版本已处理过，且没有新的变更（即版本号未变），则返回 `nil` 以避免重复处理。
>
>### 步骤 3: 检查条件
>
>- **计算新条件**: 使用 `calculateCondition` 函数基于 CRD 的当前状态计算新的条件。
>- **比较新旧条件**: 获取 CRD 当前的 `NonStructuralSchema` 条件，并与新计算的条件进行比较。
>- **跳过更新**: 如果新旧条件相同（状态、原因、消息均未变），则不进行任何操作，直接返回 `nil`。
>
>### 步骤 4: 更新 CRD 状态
>
>- **深拷贝并更新条件**: 对 CRD 进行深拷贝并根据新条件更新其状态。如果新条件不存在，则从 CRD 中移除对应的条件；如果新条件存在，则设置或更新该条件。
>- **推送更新**: 将更新后的 CRD 状态通过 `crdClient.CustomResourceDefinitions().UpdateStatus()` 推送到 Kubernetes API。
>- **处理更新中的错误**: 如果更新过程中 CRD 被删除或存在版本冲突（`IsNotFound` 或 `IsConflict` 错误），则返回 `nil`，因为这种情况下将重新调用该函数。
>
>### 步骤 5: 缓存处理结果
>
>- **更新缓存**: 为了避免同一版本的重复更新，将 CRD 的名称和新的版本号存入 `lastSeenGeneration` 映射中。

```go

func (c *ConditionController) sync(key string) error {
	inCustomResourceDefinition, err := c.crdLister.Get(key)
	if apierrors.IsNotFound(err) {
		return nil
	}
	if err != nil {
		return err
	}

	// avoid repeated calculation for the same generation
	c.lastSeenGenerationLock.Lock()
	lastSeen, seenBefore := c.lastSeenGeneration[inCustomResourceDefinition.Name]
	c.lastSeenGenerationLock.Unlock()
	if seenBefore && inCustomResourceDefinition.Generation <= lastSeen {
		return nil
	}

	// check old condition
	cond := calculateCondition(inCustomResourceDefinition)
	old := apiextensionshelpers.FindCRDCondition(inCustomResourceDefinition, apiextensionsv1.NonStructuralSchema)

	if cond == nil && old == nil {
		return nil
	}
	if cond != nil && old != nil && old.Status == cond.Status && old.Reason == cond.Reason && old.Message == cond.Message {
		return nil
	}

	// update condition
	crd := inCustomResourceDefinition.DeepCopy()
	if cond == nil {
		apiextensionshelpers.RemoveCRDCondition(crd, apiextensionsv1.NonStructuralSchema)
	} else {
		cond.LastTransitionTime = metav1.NewTime(time.Now())
		apiextensionshelpers.SetCRDCondition(crd, *cond)
	}

	_, err = c.crdClient.CustomResourceDefinitions().UpdateStatus(context.TODO(), crd, metav1.UpdateOptions{})
	if apierrors.IsNotFound(err) || apierrors.IsConflict(err) {
		// deleted or changed in the meantime, we'll get called again
		return nil
	}
	if err != nil {
		return err
	}

	// store generation in order to avoid repeated updates for the same generation (and potential
	// fights of API server in HA environments).
	c.lastSeenGenerationLock.Lock()
	defer c.lastSeenGenerationLock.Unlock()
	c.lastSeenGeneration[crd.Name] = crd.Generation

	return nil
}

```

>apiApprovalController.Run
>
>### 步骤 1: 检索 CRD
>
>- **获取 CRD**：尝试通过 `crdLister.Get(key)` 获取指定的 CRD。
>- **处理未找到的错误**：如果 CRD 不存在（即 `IsNotFound` 错误），函数返回 `nil`，表示无需进一步操作。
>- **处理其他错误**：如果在获取 CRD 时出现其他错误，则直接返回这个错误。
>
>### 步骤 2: 避免重复处理
>
>- **锁定和检查**：使用锁来安全地访问 `lastSeenProtectedAnnotation` 映射，检查 CRD 的保护注解值是否已被处理。
>- **跳过处理**：如果注解值未变（即之前已处理过），则不进行任何操作，返回 `nil`。
>
>### 步骤 3: 检查和更新条件
>
>- **计算新条件**：使用 `calculateCondition` 函数基于 CRD 的当前状态计算新的条件。
>- **比较新旧条件**：获取 CRD 当前的 `KubernetesAPIApprovalPolicyConformant` 条件，并与新计算的条件进行比较。
>- **跳过无需更新的情况**：如果新旧条件相同（状态、原因、消息均未变），则不进行任何操作，直接返回 `nil`。
>
>### 步骤 4: 更新 CRD 状态
>
>- **深拷贝并设置条件**：对 CRD 进行深拷贝并根据新条件更新其状态。
>- **推送更新**：将更新后的 CRD 状态通过 `crdClient.CustomResourceDefinitions().UpdateStatus()` 推送到 Kubernetes API。
>- **处理更新中的错误**：如果更新过程中 CRD 被删除或存在版本冲突（`IsNotFound` 或 `IsConflict` 错误），则返回 `nil`，因为这种情况下将重新调用该函数。
>
>### 步骤 5: 缓存处理结果
>
>- **更新缓存**：为了避免对同一注解值的重复更新，将 CRD 的名称和保护注解值存入 `lastSeenProtectedAnnotation` 映射中。

```go

func (c *KubernetesAPIApprovalPolicyConformantConditionController) sync(key string) error {
	inCustomResourceDefinition, err := c.crdLister.Get(key)
	if apierrors.IsNotFound(err) {
		return nil
	}
	if err != nil {
		return err
	}

	// avoid repeated calculation for the same annotation
	protectionAnnotationValue := inCustomResourceDefinition.Annotations[apiextensionsv1.KubeAPIApprovedAnnotation]
	c.lastSeenProtectedAnnotationLock.Lock()
	lastSeen, seenBefore := c.lastSeenProtectedAnnotation[inCustomResourceDefinition.Name]
	c.lastSeenProtectedAnnotationLock.Unlock()
	if seenBefore && protectionAnnotationValue == lastSeen {
		return nil
	}

	// check old condition
	cond := calculateCondition(inCustomResourceDefinition)
	if cond == nil {
		// because group is immutable, if we have no condition now, we have no need to remove a condition.
		return nil
	}
	old := apihelpers.FindCRDCondition(inCustomResourceDefinition, apiextensionsv1.KubernetesAPIApprovalPolicyConformant)

	// don't attempt a write if all the condition details are the same
	if old != nil && old.Status == cond.Status && old.Reason == cond.Reason && old.Message == cond.Message {
		// no need to update annotation because we took no action.
		return nil
	}

	// update condition
	crd := inCustomResourceDefinition.DeepCopy()
	apihelpers.SetCRDCondition(crd, *cond)

	_, err = c.crdClient.CustomResourceDefinitions().UpdateStatus(context.TODO(), crd, metav1.UpdateOptions{})
	if apierrors.IsNotFound(err) || apierrors.IsConflict(err) {
		// deleted or changed in the meantime, we'll get called again
		return nil
	}
	if err != nil {
		return err
	}

	// store annotation in order to avoid repeated updates for the same annotation (and potential
	// fights of API server in HA environments).
	c.lastSeenProtectedAnnotationLock.Lock()
	defer c.lastSeenProtectedAnnotationLock.Unlock()
	c.lastSeenProtectedAnnotation[crd.Name] = protectionAnnotationValue

	return nil
}

```

>finalizingController.Run
>
>### 步骤 1: 检索 CRD
>
>- **获取 CRD**: 通过 `crdLister.Get(key)` 尝试获取指定的 CRD。
>- **处理未找到或其他错误**: 如果 CRD 不存在（即 `IsNotFound` 错误）或遇到其他错误，函数相应地返回 `nil` 或错误。
>
>### 步骤 2: 判断是否需要处理终结器
>
>- **检查终结条件**: 检查 CRD 是否有删除时间戳 (`DeletionTimestamp`) 和是否包含特定的终结器 (`CustomResourceCleanupFinalizer`)。
>- **无需处理**: 如果没有设置删除时间戳或不包含终结器，则无需进一步操作，函数返回 `nil`。
>
>### 步骤 3: 更新终结状态
>
>- **更新状态**: 对 CRD 进行深拷贝，设置状态为 `Terminating`，表示正在进行清理。
>- **推送状态更新**: 将更新后的 CRD 状态通过 `crdClient.CustomResourceDefinitions().UpdateStatus()` 推送到 Kubernetes API。
>- **处理更新中的错误**: 如果在更新过程中 CRD 被删除或存在版本冲突，返回 `nil` 并等待下次调用。
>
>### 步骤 4: 删除实例
>
>- **判断是否跳过删除**: 检查 CRD 的实例是否与内置资源重叠，如果是，则设置相应的状态并跳过删除。
>- **删除实例**: 如果 CRD 已被标记为 `Established`，调用 `deleteInstances` 方法删除所有关联的自定义资源实例。
>- **更新删除状态**: 更新 CRD 的终结状态，如果删除过程中发生错误，再次更新状态并返回错误。
>
>### 步骤 5: 移除终结器
>
>- **移除终结器**: 从 CRD 中移除 `CustomResourceCleanupFinalizer` 终结器。
>- **推送最终状态更新**: 将最终状态更新推送到 Kubernetes API。
>- **处理最终更新中的错误**: 如果在最终更新过程中 CRD 被删除或存在版本冲突，返回 `nil`；如果有其他错误，返回该错误。

```go

func (c *CRDFinalizer) sync(key string) error {
	cachedCRD, err := c.crdLister.Get(key)
	if apierrors.IsNotFound(err) {
		return nil
	}
	if err != nil {
		return err
	}

	// no work to do
	if cachedCRD.DeletionTimestamp.IsZero() || !apiextensionshelpers.CRDHasFinalizer(cachedCRD, apiextensionsv1.CustomResourceCleanupFinalizer) {
		return nil
	}

	crd := cachedCRD.DeepCopy()

	// update the status condition.  This cleanup could take a while.
	apiextensionshelpers.SetCRDCondition(crd, apiextensionsv1.CustomResourceDefinitionCondition{
		Type:    apiextensionsv1.Terminating,
		Status:  apiextensionsv1.ConditionTrue,
		Reason:  "InstanceDeletionInProgress",
		Message: "CustomResource deletion is in progress",
	})
	crd, err = c.crdClient.CustomResourceDefinitions().UpdateStatus(context.TODO(), crd, metav1.UpdateOptions{})
	if apierrors.IsNotFound(err) || apierrors.IsConflict(err) {
		// deleted or changed in the meantime, we'll get called again
		return nil
	}
	if err != nil {
		return err
	}

	// Now we can start deleting items.  We should use the REST API to ensure that all normal admission runs.
	// Since we control the endpoints, we know that delete collection works. No need to delete if not established.
	if OverlappingBuiltInResources()[schema.GroupResource{Group: crd.Spec.Group, Resource: crd.Spec.Names.Plural}] {
		// Skip deletion, explain why, and proceed to remove the finalizer and delete the CRD
		apiextensionshelpers.SetCRDCondition(crd, apiextensionsv1.CustomResourceDefinitionCondition{
			Type:    apiextensionsv1.Terminating,
			Status:  apiextensionsv1.ConditionFalse,
			Reason:  "OverlappingBuiltInResource",
			Message: "instances overlap with built-in resources in storage",
		})
	} else if apiextensionshelpers.IsCRDConditionTrue(crd, apiextensionsv1.Established) {
		cond, deleteErr := c.deleteInstances(crd)
		apiextensionshelpers.SetCRDCondition(crd, cond)
		if deleteErr != nil {
			if _, err = c.crdClient.CustomResourceDefinitions().UpdateStatus(context.TODO(), crd, metav1.UpdateOptions{}); err != nil {
				utilruntime.HandleError(err)
			}
			return deleteErr
		}
	} else {
		apiextensionshelpers.SetCRDCondition(crd, apiextensionsv1.CustomResourceDefinitionCondition{
			Type:    apiextensionsv1.Terminating,
			Status:  apiextensionsv1.ConditionFalse,
			Reason:  "NeverEstablished",
			Message: "resource was never established",
		})
	}

	apiextensionshelpers.CRDRemoveFinalizer(crd, apiextensionsv1.CustomResourceCleanupFinalizer)
	_, err = c.crdClient.CustomResourceDefinitions().UpdateStatus(context.TODO(), crd, metav1.UpdateOptions{})
	if apierrors.IsNotFound(err) || apierrors.IsConflict(err) {
		// deleted or changed in the meantime, we'll get called again
		return nil
	}
	return err
}
```

