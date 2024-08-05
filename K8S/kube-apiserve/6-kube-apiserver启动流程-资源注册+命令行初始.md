**本章重点：**

（1）kube-apiserver启动过程中，前两个步骤：资源注册和命令行解析

在kube-apiserver组件启动过程中，代码逻辑可分为9个步骤，分别介绍如下。

（1）资源注册。

（2）Cobra命令行参数解析。

（3）创建APIServer通用配置。

（4）创建APIExtensionsServer。

（5）创建KubeAPIServer。

（6）创建AggregatorServer。

（7）启动HTTP服务。

（8）启动HTTPS服务

对应的流程图如下：

![image-20240730135042958](C:\mynote\images4\image-20240730135042958.png)

# 1. 资源注册

kube-apiserver组件启动后的第一件事情是将Kubernetes所支持的资源注册到Scheme资源注册表中，这样后面启动的逻辑才能够从Scheme资源注册表中拿到资源信息并启动和运行APIExtensionsServer、KubeAPIServer、AggregatorServer这3种服务。资源的注册过程并不是通过函数调用触发的，而是通过Go语言的导入（import）和初始化（init）机制触发的。导入和初始化机制如下图所示。

![image-20240729152959443](C:\mynote\images4\image-20240729152959443.png)

kube-apiserver的资源注册过程就利用了 import 和 init机制，代码示例如下：

在 kube-apiserver 入口函数所在的文件中 cmd\kube-apiserver\app\server.go （NewAPIServerCommand 函数就在这个文件中）

```go
	"k8s.io/kubernetes/pkg/api/legacyscheme"
	"k8s.io/kubernetes/pkg/controlplane"
```

kube-apiserver导入了legacyscheme和controlplane包。kube-apiserver资源注册分为两步：第1步，初始化Scheme资源注册表；第2步，注册Kubernetes所支持的资源。

**初始化资源注册表 scheme**

因为 server.go import了 legacyscheme包，所以 legacyscheme包里面的 var会被初始化。

而在legacyscheme包中，就初始化了 scheme表。

```go
package legacyscheme

import (
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/serializer"
)

// Scheme is the default instance of runtime.Scheme to which types in the Kubernetes API are already registered.
// NOTE: If you are copying this file to start a new api group, STOP! Copy the
// extensions group instead. This Scheme is special and should appear ONLY in
// the api group, unless you really know what you're doing.
// TODO(lavalamp): make the above error impossible.
var Scheme = runtime.NewScheme()

// Codecs provides access to encoding and decoding for the scheme
var Codecs = serializer.NewCodecFactory(Scheme)

// ParameterCodec handles versioning of objects that are converted to query parameters.
var ParameterCodec = runtime.NewParameterCodec(Scheme)
```


同样 server.go import了 controlplane包。所以 controlplane包引入的 包也会 运行 init var等过程。

在 controlplane包中，有一个 pkg\controlplane\import_known_versions.go 文件。这个文件 引入了包，但是啥都没干。这样的目的，就是 完成 引入包的 init, var。

```go
package controlplane

import (
	// These imports are the API groups the API server will support.
	_ "k8s.io/kubernetes/pkg/apis/admission/install"
	_ "k8s.io/kubernetes/pkg/apis/admissionregistration/install"
	_ "k8s.io/kubernetes/pkg/apis/apiserverinternal/install"
	_ "k8s.io/kubernetes/pkg/apis/apps/install"
	_ "k8s.io/kubernetes/pkg/apis/authentication/install"
	_ "k8s.io/kubernetes/pkg/apis/authorization/install"
	_ "k8s.io/kubernetes/pkg/apis/autoscaling/install"
	_ "k8s.io/kubernetes/pkg/apis/batch/install"
	_ "k8s.io/kubernetes/pkg/apis/certificates/install"
	_ "k8s.io/kubernetes/pkg/apis/coordination/install"
	_ "k8s.io/kubernetes/pkg/apis/core/install"
	_ "k8s.io/kubernetes/pkg/apis/discovery/install"
	_ "k8s.io/kubernetes/pkg/apis/events/install"
	_ "k8s.io/kubernetes/pkg/apis/extensions/install"
	_ "k8s.io/kubernetes/pkg/apis/flowcontrol/install"
	_ "k8s.io/kubernetes/pkg/apis/imagepolicy/install"
	_ "k8s.io/kubernetes/pkg/apis/networking/install"
	_ "k8s.io/kubernetes/pkg/apis/node/install"
	_ "k8s.io/kubernetes/pkg/apis/policy/install"
	_ "k8s.io/kubernetes/pkg/apis/rbac/install"
	_ "k8s.io/kubernetes/pkg/apis/resource/install"
	_ "k8s.io/kubernetes/pkg/apis/scheduling/install"
	_ "k8s.io/kubernetes/pkg/apis/storage/install"
	_ "k8s.io/kubernetes/pkg/apis/storagemigration/install"
)

```

随便拿一个为例，例如k8s.io/kubernetes/pkg/apis/core/install 包

```go
package install

import (
	"k8s.io/apimachinery/pkg/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	"k8s.io/kubernetes/pkg/api/legacyscheme"
	"k8s.io/kubernetes/pkg/apis/core"
	"k8s.io/kubernetes/pkg/apis/core/v1"
)

func init() {
	Install(legacyscheme.Scheme)
}

// Install registers the API group and adds types to a scheme
func Install(scheme *runtime.Scheme) {
    //注册内部版本
	utilruntime.Must(core.AddToScheme(scheme))
    //注册外部版本
	utilruntime.Must(v1.AddToScheme(scheme))
	utilruntime.Must(scheme.SetVersionPriority(v1.SchemeGroupVersion))
}

```

所以上面的一个 utilruntime.Must(core.AddToScheme(scheme))

就将pod,podlist等等都注册到了scheme中去。

```go
package core

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
)

// GroupName is the group name use in this package
const GroupName = ""

// SchemeGroupVersion is group version used to register these objects
var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: runtime.APIVersionInternal}

// Kind takes an unqualified kind and returns a Group qualified GroupKind
func Kind(kind string) schema.GroupKind {
	return SchemeGroupVersion.WithKind(kind).GroupKind()
}

// Resource takes an unqualified resource and returns a Group qualified GroupResource
func Resource(resource string) schema.GroupResource {
	return SchemeGroupVersion.WithResource(resource).GroupResource()
}

var (
	// SchemeBuilder object to register various known types
	SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)

	// AddToScheme represents a func that can be used to apply all the registered
	// funcs in a scheme
	AddToScheme = SchemeBuilder.AddToScheme
)

func addKnownTypes(scheme *runtime.Scheme) error {
	if err := scheme.AddIgnoredConversionType(&metav1.TypeMeta{}, &metav1.TypeMeta{}); err != nil {
		return err
	}
	scheme.AddKnownTypes(SchemeGroupVersion,
		&Pod{},
		&PodList{},
		&PodStatusResult{},
		&PodTemplate{},
		&PodTemplateList{},
		&ReplicationControllerList{},
		&ReplicationController{},
		&ServiceList{},
		&Service{},
		&ServiceProxyOptions{},
		&NodeList{},
		&Node{},
		&NodeProxyOptions{},
		&Endpoints{},
		&EndpointsList{},
		&Binding{},
		&Event{},
		&EventList{},
		&List{},
		&LimitRange{},
		&LimitRangeList{},
		&ResourceQuota{},
		&ResourceQuotaList{},
		&Namespace{},
		&NamespaceList{},
		&ServiceAccount{},
		&ServiceAccountList{},
		&Secret{},
		&SecretList{},
		&PersistentVolume{},
		&PersistentVolumeList{},
		&PersistentVolumeClaim{},
		&PersistentVolumeClaimList{},
		&PodAttachOptions{},
		&PodLogOptions{},
		&PodExecOptions{},
		&PodPortForwardOptions{},
		&PodProxyOptions{},
		&ComponentStatus{},
		&ComponentStatusList{},
		&SerializedReference{},
		&RangeAllocation{},
		&ConfigMap{},
		&ConfigMapList{},
	)

	return nil
}
```

这里可以看出来，init中 就注册了 core资源。在上述代码中，core.AddToScheme函数注册了core资源组内部版本的资源，v1.AddToScheme函数注册了core资源

组外部版本的资源，scheme.SetVersionPriority函数注册了资源组的版本顺序。如果有多个资源版本，排在最前面的为资源首选版本。

提示：除将KubeAPIServer（API核心服务）注册至legacyscheme.Scheme资源注册表以外，还需要了解APIExtensionsServer和AggregatorServer资源注册过程。

● 将APIExtensionsServer （API扩展服务）注册至extensionsapiserver.Scheme资源注册表，注册过程定义在vendor/k8s.io/apiextensions-apiserver/pkg/apiserver/apiserver.go中。

● 将AggregatorServer（API聚合服务）注册至aggregatorscheme.Scheme资源注册表，注册过程定义在vendor/k8s.io/kube-aggregator/pkg/apiserver/scheme/scheme.go中。

# 2. Cobra命令行参数解析

## 2.1. 入口函数 main->NewAPIServerCommand

cmd\kube-apiserver\apiserver.go

```go
func main() {
	command := app.NewAPIServerCommand()
	code := cli.Run(command)
	os.Exit(code)
}
```

**NewAPIServerCommand**

```go

// NewAPIServerCommand creates a *cobra.Command object with default parameters

	// 1. 定义 NewServerRunOptions。就是定义所有参数的结构体对象。详见：2.2
	s := options.NewServerRunOptions()
	cmd := &cobra.Command{
		Use: "kube-apiserver",
		Long: `The Kubernetes API server validates and configures data
for the api objects which include pods, services, replicationcontrollers, and
others. The API Server services REST operations and provides the frontend to the
cluster's shared state through which all other components interact.`,

		// stop printing usage when the command errors
		SilenceUsage: true,
		PersistentPreRunE: func(*cobra.Command, []string) error {
			// silence client-go warnings.
			// kube-apiserver loopback clients should not log self-issued warnings.
			rest.SetDefaultWarningHandler(rest.NoWarnings{})
			return nil
		},
		RunE: func(cmd *cobra.Command, args []string) error {
			verflag.PrintAndExitIfRequested()
			fs := cmd.Flags()

			// Activate logging as soon as possible, after that
			// show flags with the final logging configuration.
			if err := logsapi.ValidateAndApply(s.Logs, utilfeature.DefaultFeatureGate); err != nil {
				return err
			}
			cliflag.PrintFlags(fs)

			// set default options
			completedOptions, err := s.Complete()
			if err != nil {
				return err
			}

			// validate options
			if errs := completedOptions.Validate(); len(errs) != 0 {
				return utilerrors.NewAggregate(errs)
			}
			// add feature enablement metrics
			utilfeature.DefaultMutableFeatureGate.AddMetrics()
			return Run(completedOptions, genericapiserver.SetupSignalHandler())
		},
		Args: func(cmd *cobra.Command, args []string) error {
			for _, arg := range args {
				if len(arg) > 0 {
					return fmt.Errorf("%q does not take any arguments, got %q", cmd.CommandPath(), args)
				}
			}
			return nil
		},
	}
	// 3. 初始化cmd的 flagset。这里就是定义一个空的，以 kube-apserver为名字的 flagset。详见2.3
	fs := cmd.Flags()

	// 4. 绑定 kube-apiserver各个组件（etcd,CloudProvider等等），详见 2.4
	namedFlagSets := s.Flags()
	verflag.AddFlags(namedFlagSets.FlagSet("global"))
	globalflag.AddGlobalFlags(namedFlagSets.FlagSet("global"), cmd.Name(), logs.SkipLoggingConfigurationFlags())
	options.AddCustomGlobalFlags(namedFlagSets.FlagSet("generic"))

	// 5. 将各个组件的 flagset加入，fs中。这样fs就有了 kube-apiserver，以及各个组件的flagset了。
	for _, f := range namedFlagSets.FlagSets {
		fs.AddFlagSet(f)
	}

	cols, _, _ := term.TerminalSize(cmd.OutOrStdout())
	cliflag.SetUsageAndHelpFunc(cmd, namedFlagSets, cols)

	return cmd
}
```

## 2.2 options.NewServerRunOptions()

```go
// NewServerRunOptions creates a new ServerRunOptions object with default parameters
func NewServerRunOptions() *ServerRunOptions {
	s := ServerRunOptions{
        //配置APIserver常规配置
		Options:       controlplaneapiserver.NewOptions(),
        //云服务配置
		CloudProvider: kubeoptions.NewCloudProviderOptions(),

		Extra: Extra{
            //lease
			EndpointReconcilerType: string(reconcilers.LeaseEndpointReconcilerType),
			KubeletConfig: kubeletclient.KubeletClientConfig{
                //10250， kubelet 通信使用的主端口号
				Port:         ports.KubeletPort,
                //10255， kubelet 的只读端口
				ReadOnlyPort: ports.KubeletReadOnlyPort,
				PreferredAddressTypes: []string{
					// --override-hostname
                    //如果节点报告了主机名，优先使用
					string(api.NodeHostName),

					// internal, preferring DNS if reported
                    //如果内部 DNS 名称可用，次优先选择
					string(api.NodeInternalDNS),
                    //内部 IP 地址，如果内部 DNS 不可用时使用
					string(api.NodeInternalIP),

					// external, preferring DNS if reported
                    //外部 DNS 名称，通常用于跨云或跨区域通信
					string(api.NodeExternalDNS),
                    //外部 IP 地址，作为最后的备选项
					string(api.NodeExternalIP),
				},
                //设置了 HTTP 请求的超时时间
				HTTPTimeout: time.Duration(5) * time.Second,
			},
            //服务节点端口范围的默认值
			ServiceNodePortRange: kubeoptions.DefaultServiceNodePortRange,
            		//集群中预期的 master 节点数量
			MasterCount:          1,
		},
	}

	return &s
}
```

> NewOptions常规配置的实列化Etcd 
>
> 1. **GenericServerRunOptions**：通用服务器运行选项，提供服务器运行的基本设置，如绑定地址、端口等。
> 2. **Etcd**：用于配置 Etcd 存储后端，包括默认的路径前缀和存储媒介。
> 3. **SecureServing**：配置安全服务选项，包括 TLS 证书和监听端口等安全相关的设置。
> 4. **Audit**：审计选项，用于配置审计日志的生成和存储。
> 5. **Features**：功能选项，用于启用或禁用 Kubernetes 的特定功能。
> 6. **Admission**：准入控制选项，管理准入控制插件，这些插件在资源被创建或更新时进行拦截处理。
> 7. **Authentication**：身份验证选项，配置API服务器的身份验证机制。
> 8. **Authorization**：授权选项，配置API服务器的授权机制。
> 9. **APIEnablement**：API 启用选项，用于控制哪些 API 组和版本在服务器上可用。
> 10. **EgressSelector**：出口选择器选项，配置 API 服务器到外部服务的通信路由。
> 11. **Metrics**：度量选项，用于配置度量数据的收集和报告。
> 12. **Logs**：日志选项，用于配置日志记录的细节。
> 13. **Traces**：跟踪选项，用于配置 API 服务器的请求跟踪。
>
> 此外，还有一些特定的配置：
>
> - **EnableLogsHandler**：是否启用日志处理程序。
> - **EventTTL**：事件对象的生存时间，配置为 1 小时。
> - **AggregatorRejectForwardingRedirects**：聚合层是否拒绝重定向和转发的请求。

* 这里面有些还有默认初始化值，就不一 一展开了

```go
// NewOptions creates a new ServerRunOptions object with default parameters
func NewOptions() *Options {
	s := Options{
		GenericServerRunOptions: genericoptions.NewServerRunOptions(),
		Etcd:                    genericoptions.NewEtcdOptions(storagebackend.NewDefaultConfig(kubeoptions.DefaultEtcdPathPrefix, nil)),
		SecureServing:           kubeoptions.NewSecureServingOptions(),
		Audit:                   genericoptions.NewAuditOptions(),
		Features:                genericoptions.NewFeatureOptions(),
		Admission:               kubeoptions.NewAdmissionOptions(),
		Authentication:          kubeoptions.NewBuiltInAuthenticationOptions().WithAll(),
		Authorization:           kubeoptions.NewBuiltInAuthorizationOptions(),
		APIEnablement:           genericoptions.NewAPIEnablementOptions(),
		EgressSelector:          genericoptions.NewEgressSelectorOptions(),
		Metrics:                 metrics.NewOptions(),
		Logs:                    logs.NewOptions(),
		Traces:                  genericoptions.NewTracingOptions(),

		EnableLogsHandler:                   true,
		EventTTL:                            1 * time.Hour,
		AggregatorRejectForwardingRedirects: true,
	}

	// Overwrite the default for storage data format.
	s.Etcd.DefaultStorageMediaType = "application/vnd.kubernetes.protobuf"

	return &s
}
```

> 只展示etcd

```go
func NewEtcdOptions(backendConfig *storagebackend.Config) *EtcdOptions {
	options := &EtcdOptions{
		StorageConfig:           *backendConfig,
		DefaultStorageMediaType: "application/json",
		DeleteCollectionWorkers: 1,
		EnableGarbageCollection: true,
		EnableWatchCache:        true,
		DefaultWatchCacheSize:   100,
	}
	options.StorageConfig.CountMetricPollPeriod = time.Minute
	return options
}

```

## 2.3 cmd.Flags()

就是定义一个 kube-apiserver 为名字的 flagset。

```
// Flags returns the complete FlagSet that applies
// to this command (local and persistent declared here and by all parents).
func (c *Command) Flags() *flag.FlagSet {
	if c.flags == nil {
		c.flags = flag.NewFlagSet(c.Name(), flag.ContinueOnError)
		if c.flagErrorBuf == nil {
			c.flagErrorBuf = new(bytes.Buffer)
		}
		c.flags.SetOutput(c.flagErrorBuf)
	}

	return c.flags
}
```

### 2.3.1 C.Name

Command.Name = Command.Use。所以这里就是 c.Name = "kube-apiserver"

```
// Name returns the command's name: the first word in the use line.
func (c *Command) Name() string {
	name := c.Use
	i := strings.Index(name, " ")
	if i >= 0 {
		name = name[:i]
	}
	return name
}
```

### 2.3.2 NewFlagSet

就是返回一个 flagset对象。

```go
// NewFlagSet returns a new, empty flag set with the specified name,
// error handling property and SortFlags set to true.
func NewFlagSet(name string, errorHandling ErrorHandling) *FlagSet {
	f := &FlagSet{
		name:          name,
		errorHandling: errorHandling,
		argsLenAtDash: -1,
		interspersed:  true,
		SortFlags:     true,
	}
	return f
}

```

```go
// A FlagSet represents a set of defined flags.
type FlagSet struct {
	// Usage is the function called when an error occurs while parsing flags.
	// The field is a function (not a method) that may be changed to point to
	// a custom error handler.
	Usage func()

	// SortFlags is used to indicate, if user wants to have sorted flags in
	// help/usage messages.
	SortFlags bool

	// ParseErrorsWhitelist is used to configure a whitelist of errors
	ParseErrorsWhitelist ParseErrorsWhitelist

	name              string
	parsed            bool
	actual            map[NormalizedName]*Flag
	orderedActual     []*Flag
	sortedActual      []*Flag
	formal            map[NormalizedName]*Flag
	orderedFormal     []*Flag
	sortedFormal      []*Flag
	shorthands        map[byte]*Flag
	args              []string // arguments after flags
	argsLenAtDash     int      // len(args) when a '--' was located when parsing, or -1 if no --
	errorHandling     ErrorHandling
	output            io.Writer // nil means stderr; use out() accessor
	interspersed      bool      // allow interspersed option/non-option args
	normalizeNameFunc func(f *FlagSet, name string) NormalizedName

	addedGoFlagSets []*goflag.FlagSet
}
```

通过打印日志发现，这个时候都还是空的。

```go
I0127 10:51:41.053661    5612 flag.go:1209] zxtest f.name is kube-apiserver
I0127 10:51:41.053666    5612 flag.go:1210] zxtest f.actual is map[]
I0127 10:51:41.053677    5612 flag.go:1211] zxtest f.orderedActual is []
I0127 10:51:41.053684    5612 flag.go:1212] zxtest f.sortedActual is []
I0127 10:51:41.053689    5612 flag.go:1213] zxtest f.formal is map[]
I0127 10:51:41.053695    5612 flag.go:1214] zxtest f.orderedFormal is []
I0127 10:51:41.053702    5612 flag.go:1215] zxtest f.sortedFormal is []
I0127 10:51:41.053708    5612 flag.go:1216] zxtest f.shorthands is map[]
I0127 10:51:41.053714    5612 flag.go:1217] zxtest f.args is []
```

## 2.4 s.Flags()

s.Flags就是让 结构体的参数 和启动时输入的参数进行一个绑定。

fss apiserverflag.NamedFlagSets是一个 map，对应 [key] flagSet

可以认为是：     

```go
fss = {
    “etcd”， flagset1,

    "secure serving", flagset2,
 
} 
```

```go
// Flags returns flags for a specific APIServer by section name
func (s *ServerRunOptions) Flags() (fss cliflag.NamedFlagSets) {
	s.Options.AddFlags(&fss)
	s.CloudProvider.AddFlags(fss.FlagSet("cloud provider"))

	// Note: the weird ""+ in below lines seems to be the only way to get gofmt to
	// arrange these text blocks sensibly. Grrr.
	fs := fss.FlagSet("misc")

	fs.BoolVar(&s.AllowPrivileged, "allow-privileged", s.AllowPrivileged,
		"If true, allow privileged containers. [default=false]")

	fs.StringVar(&s.EndpointReconcilerType, "endpoint-reconciler-type", s.EndpointReconcilerType,
		"Use an endpoint reconciler ("+strings.Join(reconcilers.AllTypes.Names(), ", ")+") master-count is deprecated, and will be removed in a future version.")

	// See #14282 for details on how to test/try this option out.
	// TODO: remove this comment once this option is tested in CI.
	fs.IntVar(&s.KubernetesServiceNodePort, "kubernetes-service-node-port", s.KubernetesServiceNodePort, ""+
		"If non-zero, the Kubernetes master service (which apiserver creates/maintains) will be "+
		"of type NodePort, using this as the value of the port. If zero, the Kubernetes master "+
		"service will be of type ClusterIP.")

	fs.StringVar(&s.ServiceClusterIPRanges, "service-cluster-ip-range", s.ServiceClusterIPRanges, ""+
		"A CIDR notation IP range from which to assign service cluster IPs. This must not "+
		"overlap with any IP ranges assigned to nodes or pods. Max of two dual-stack CIDRs is allowed.")

	fs.Var(&s.ServiceNodePortRange, "service-node-port-range", ""+
		"A port range to reserve for services with NodePort visibility.  This must not overlap with the ephemeral port range on nodes.  "+
		"Example: '30000-32767'. Inclusive at both ends of the range.")

	// Kubelet related flags:
	fs.StringSliceVar(&s.KubeletConfig.PreferredAddressTypes, "kubelet-preferred-address-types", s.KubeletConfig.PreferredAddressTypes,
		"List of the preferred NodeAddressTypes to use for kubelet connections.")

	fs.UintVar(&s.KubeletConfig.Port, "kubelet-port", s.KubeletConfig.Port,
		"DEPRECATED: kubelet port.")
	fs.MarkDeprecated("kubelet-port", "kubelet-port is deprecated and will be removed.")

	fs.UintVar(&s.KubeletConfig.ReadOnlyPort, "kubelet-read-only-port", s.KubeletConfig.ReadOnlyPort,
		"DEPRECATED: kubelet read only port.")
	fs.MarkDeprecated("kubelet-read-only-port", "kubelet-read-only-port is deprecated and will be removed.")

	fs.DurationVar(&s.KubeletConfig.HTTPTimeout, "kubelet-timeout", s.KubeletConfig.HTTPTimeout,
		"Timeout for kubelet operations.")

	fs.StringVar(&s.KubeletConfig.TLSClientConfig.CertFile, "kubelet-client-certificate", s.KubeletConfig.TLSClientConfig.CertFile,
		"Path to a client cert file for TLS.")

	fs.StringVar(&s.KubeletConfig.TLSClientConfig.KeyFile, "kubelet-client-key", s.KubeletConfig.TLSClientConfig.KeyFile,
		"Path to a client key file for TLS.")

	fs.StringVar(&s.KubeletConfig.TLSClientConfig.CAFile, "kubelet-certificate-authority", s.KubeletConfig.TLSClientConfig.CAFile,
		"Path to a cert file for the certificate authority.")

	fs.IntVar(&s.MasterCount, "apiserver-count", s.MasterCount,
		"The number of apiservers running in the cluster, must be a positive number. (In use when --endpoint-reconciler-type=master-count is enabled.)")
	fs.MarkDeprecated("apiserver-count", "apiserver-count is deprecated and will be removed in a future version.")

	return fss
}

```

map中如果没有就新建一个

```go
// FlagSet returns the flag set with the given name and adds it to the
// ordered name list if it is not in there yet.
func (nfs *NamedFlagSets) FlagSet(name string) *pflag.FlagSet {
	if nfs.FlagSets == nil {
		nfs.FlagSets = map[string]*pflag.FlagSet{}
	}
	if _, ok := nfs.FlagSets[name]; !ok {
		flagSet := pflag.NewFlagSet(name, pflag.ExitOnError)
		flagSet.SetNormalizeFunc(pflag.CommandLine.GetNormalizeFunc())
		if nfs.NormalizeNameFunc != nil {
			flagSet.SetNormalizeFunc(nfs.NormalizeNameFunc)
		}
		nfs.FlagSets[name] = flagSet
		nfs.Order = append(nfs.Order, name)
	}
	return nfs.FlagSets[name]
}
```

还是以etcd为例，这里就是将 EtcdOptions结构体中的一个一个变量和 输入的参数进行绑定。

```go

// AddFlags adds flags related to etcd storage for a specific APIServer to the specified FlagSet
func (s *EtcdOptions) AddFlags(fs *pflag.FlagSet) {
	if s == nil {
		return
	}

	fs.StringSliceVar(&s.EtcdServersOverrides, "etcd-servers-overrides", s.EtcdServersOverrides, ""+
		"Per-resource etcd servers overrides, comma separated. The individual override "+
		"format: group/resource#servers, where servers are URLs, semicolon separated. "+
		"Note that this applies only to resources compiled into this server binary. ")

	fs.StringVar(&s.DefaultStorageMediaType, "storage-media-type", s.DefaultStorageMediaType, ""+
		"The media type to use to store objects in storage. "+
		"Some resources or storage backends may only support a specific media type and will ignore this setting. "+
		"Supported media types: [application/json, application/yaml, application/vnd.kubernetes.protobuf]")
	fs.IntVar(&s.DeleteCollectionWorkers, "delete-collection-workers", s.DeleteCollectionWorkers,
		"Number of workers spawned for DeleteCollection call. These are used to speed up namespace cleanup.")

	fs.BoolVar(&s.EnableGarbageCollection, "enable-garbage-collector", s.EnableGarbageCollection, ""+
		"Enables the generic garbage collector. MUST be synced with the corresponding flag "+
		"of the kube-controller-manager.")

	fs.BoolVar(&s.EnableWatchCache, "watch-cache", s.EnableWatchCache,
		"Enable watch caching in the apiserver")

	fs.IntVar(&s.DefaultWatchCacheSize, "default-watch-cache-size", s.DefaultWatchCacheSize,
		"Default watch cache size. If zero, watch cache will be disabled for resources that do not have a default watch size set.")

	fs.MarkDeprecated("default-watch-cache-size",
		"watch caches are sized automatically and this flag will be removed in a future version")

	fs.StringSliceVar(&s.WatchCacheSizes, "watch-cache-sizes", s.WatchCacheSizes, ""+
		"Watch cache size settings for some resources (pods, nodes, etc.), comma separated. "+
		"The individual setting format: resource[.group]#size, where resource is lowercase plural (no version), "+
		"group is omitted for resources of apiVersion v1 (the legacy core API) and included for others, "+
		"and size is a number. This option is only meaningful for resources built into the apiserver, "+
		"not ones defined by CRDs or aggregated from external servers, and is only consulted if the "+
		"watch-cache is enabled. The only meaningful size setting to supply here is zero, which means to "+
		"disable watch caching for the associated resource; all non-zero values are equivalent and mean "+
		"to not disable watch caching for that resource")

	fs.StringVar(&s.StorageConfig.Type, "storage-backend", s.StorageConfig.Type,
		"The storage backend for persistence. Options: 'etcd3' (default).")

	fs.StringSliceVar(&s.StorageConfig.Transport.ServerList, "etcd-servers", s.StorageConfig.Transport.ServerList,
		"List of etcd servers to connect with (scheme://ip:port), comma separated.")

	fs.StringVar(&s.StorageConfig.Prefix, "etcd-prefix", s.StorageConfig.Prefix,
		"The prefix to prepend to all resource paths in etcd.")

	fs.StringVar(&s.StorageConfig.Transport.KeyFile, "etcd-keyfile", s.StorageConfig.Transport.KeyFile,
		"SSL key file used to secure etcd communication.")

	fs.StringVar(&s.StorageConfig.Transport.CertFile, "etcd-certfile", s.StorageConfig.Transport.CertFile,
		"SSL certification file used to secure etcd communication.")

	fs.StringVar(&s.StorageConfig.Transport.TrustedCAFile, "etcd-cafile", s.StorageConfig.Transport.TrustedCAFile,
		"SSL Certificate Authority file used to secure etcd communication.")

	fs.StringVar(&s.EncryptionProviderConfigFilepath, "encryption-provider-config", s.EncryptionProviderConfigFilepath,
		"The file containing configuration for encryption providers to be used for storing secrets in etcd")

	fs.BoolVar(&s.EncryptionProviderConfigAutomaticReload, "encryption-provider-config-automatic-reload", s.EncryptionProviderConfigAutomaticReload,
		"Determines if the file set by --encryption-provider-config should be automatically reloaded if the disk contents change. "+
			"Setting this to true disables the ability to uniquely identify distinct KMS plugins via the API server healthz endpoints.")

	fs.DurationVar(&s.StorageConfig.CompactionInterval, "etcd-compaction-interval", s.StorageConfig.CompactionInterval,
		"The interval of compaction requests. If 0, the compaction request from apiserver is disabled.")

	fs.DurationVar(&s.StorageConfig.CountMetricPollPeriod, "etcd-count-metric-poll-period", s.StorageConfig.CountMetricPollPeriod, ""+
		"Frequency of polling etcd for number of resources per type. 0 disables the metric collection.")

	fs.DurationVar(&s.StorageConfig.DBMetricPollInterval, "etcd-db-metric-poll-interval", s.StorageConfig.DBMetricPollInterval,
		"The interval of requests to poll etcd and update metric. 0 disables the metric collection")

	fs.DurationVar(&s.StorageConfig.HealthcheckTimeout, "etcd-healthcheck-timeout", s.StorageConfig.HealthcheckTimeout,
		"The timeout to use when checking etcd health.")

	fs.DurationVar(&s.StorageConfig.ReadycheckTimeout, "etcd-readycheck-timeout", s.StorageConfig.ReadycheckTimeout,
		"The timeout to use when checking etcd readiness")

	fs.Int64Var(&s.StorageConfig.LeaseManagerConfig.ReuseDurationSeconds, "lease-reuse-duration-seconds", s.StorageConfig.LeaseManagerConfig.ReuseDurationSeconds,
		"The time in seconds that each lease is reused. A lower value could avoid large number of objects reusing the same lease. Notice that a too small value may cause performance problems at storage layer.")
}
```

# 3.Run()真正的参数解析

>RUN------->run----->Execute-------->ExecuteC------->Command execute
>
>cmd.execute 的大体流程如下：
>
>（1）解析参数
>
>（2）判断cmd是否设置了 run, runE函数。没有就直接返回
>
>（3）运行设置的初始化函数，preRun
>
>（4）运行RunE,或者Run函数。

```go
// Run provides the common boilerplate code around executing a cobra command.
// For example, it ensures that logging is set up properly. Logging
// flags get added to the command line if not added already. Flags get normalized
// so that help texts show them with hyphens. Underscores are accepted
// as alternative for the command parameters.
//
// Run tries to be smart about how to print errors that are returned by the
// command: before logging is known to be set up, it prints them as plain text
// to stderr. This covers command line flag parse errors and unknown commands.
// Afterwards it logs them. This covers runtime errors.
//
// Commands like kubectl where logging is not normally part of the runtime output
// should use RunNoErrOutput instead and deal with the returned error themselves.
func Run(cmd *cobra.Command) int {
	//解析命令行参数，logsInitialized代表日志是否初始化，err代表解析命令行参数的错误
	if logsInitialized, err := run(cmd); err != nil {
		// If the error is about flag parsing, then printing that error
		// with the decoration that klog would add ("E0923
		// 23:02:03.219216 4168816 run.go:61] unknown shorthand flag")
		// is less readable. Using klog.Fatal is even worse because it
		// dumps a stack trace that isn't about the error.
		//
		// But if it is some other error encountered at runtime, then
		// we want to log it as error, at least in most commands because
		// their output is a log event stream.
		//
		// We can distinguish these two cases depending on whether
		// we got to logs.InitLogs() above.
		//
		// This heuristic might be problematic for command line
		// tools like kubectl where the output is carefully controlled
		// and not a log by default. They should use RunNoErrOutput
		// instead.
		//
		// The usage of klog is problematic also because we don't know
		// whether the command has managed to configure it. This cannot
		// be checked right now, but may become possible when the early
		// logging proposal from
		// https://github.com/kubernetes/enhancements/pull/3078
		// ("contextual logging") is implemented.
		//没有初始化，标准输出错误
		if !logsInitialized {
			fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		} else {
			//已经初始化，记录错误
			klog.ErrorS(err, "command failed")
		}
		//失败
		return 1
	}
	//成功
	return 0
}
```

## 3.1 run

```go

func run(cmd *cobra.Command) (logsInitialized bool, err error) {
	rand.Seed(time.Now().UnixNano())
	defer logs.FlushLogs()

	cmd.SetGlobalNormalizationFunc(cliflag.WordSepNormalizeFunc)

	// When error printing is enabled for the Cobra command, a flag parse
	// error gets printed first, then optionally the often long usage
	// text. This is very unreadable in a console because the last few
	// lines that will be visible on screen don't include the error.
	//
	// The recommendation from #sig-cli was to print the usage text, then
	// the error. We implement this consistently for all commands here.
	// However, we don't want to print the usage text when command
	// execution fails for other reasons than parsing. We detect this via
	// the FlagParseError callback.
	//
	// Some commands, like kubectl, already deal with this themselves.
	// We don't change the behavior for those.
	if !cmd.SilenceUsage {
		cmd.SilenceUsage = true
		cmd.SetFlagErrorFunc(func(c *cobra.Command, err error) error {
			// Re-enable usage printing.
			c.SilenceUsage = false
			return err
		})
	}

	// In all cases error printing is done below.
	cmd.SilenceErrors = true

	// This is idempotent.
	logs.AddFlags(cmd.PersistentFlags())

	// Inject logs.InitLogs after command line parsing into one of the
	// PersistentPre* functions.
	switch {
	case cmd.PersistentPreRun != nil:
		pre := cmd.PersistentPreRun
		cmd.PersistentPreRun = func(cmd *cobra.Command, args []string) {
			logs.InitLogs()
			logsInitialized = true
			pre(cmd, args)
		}
	case cmd.PersistentPreRunE != nil:
		pre := cmd.PersistentPreRunE
		cmd.PersistentPreRunE = func(cmd *cobra.Command, args []string) error {
			logs.InitLogs()
			logsInitialized = true
			return pre(cmd, args)
		}
	default:
		cmd.PersistentPreRun = func(cmd *cobra.Command, args []string) {
			logs.InitLogs()
			logsInitialized = true
		}
	}

	err = cmd.Execute()
	return
}

```

## 3.2 Execute

```gp
// Execute uses the args (os.Args[1:] by default)
// and run through the command tree finding appropriate matches
// for commands and then corresponding flags.
func (c *Command) Execute() error {
	_, err := c.ExecuteC()
	return err
}

```

## 3.3  ExecuteC

```
// ExecuteC executes the command.
func (c *Command) ExecuteC() (cmd *Command, err error) {
   if c.ctx == nil {
      c.ctx = context.Background()
   }

   // Regardless of what command execute is called on, run on Root only
   if c.HasParent() {
      return c.Root().ExecuteC()
   }

   // windows hook
   //如果定义了 preExecHookFn（一个预执行钩子函数），则在执行任何操作之前调用它
   if preExecHookFn != nil {
      preExecHookFn(c)
   }
   //初始化默认的帮助和自动补全命令
   // initialize help at the last point to allow for user overriding
   c.InitDefaultHelpCmd()
   // initialize completion at the last point to allow for user overriding
   c.InitDefaultCompletionCmd()

   // Now that all commands have been created, let's make sure all groups
   // are properly created also
   //检查和初始化命令组，这通常用于管理相关命令的分组
   c.checkCommandGroups()
   //将命令对象 c 的参数列表赋值给局部变量 args
   args := c.args

   // Workaround FAIL with "go test -v" or "cobra.test -test.v", see #155
   if c.args == nil && filepath.Base(os.Args[0]) != "cobra.test" {
      args = os.Args[1:]
   }

   // initialize the hidden command to be used for shell completion
   c.initCompleteCmd(args)

   var flags []string
   if c.TraverseChildren {
      cmd, flags, err = c.Traverse(args)
   } else {
      cmd, flags, err = c.Find(args)
   }
   if err != nil {
      // If found parse to a subcommand and then failed, talk about the subcommand
      if cmd != nil {
         c = cmd
      }
      if !c.SilenceErrors {
         c.PrintErrln("Error:", err.Error())
         c.PrintErrf("Run '%v --help' for usage.\n", c.CommandPath())
      }
      return c, err
   }

   cmd.commandCalledAs.called = true
   if cmd.commandCalledAs.name == "" {
      cmd.commandCalledAs.name = cmd.Name()
   }

   // We have to pass global context to children command
   // if context is present on the parent command.
   if cmd.ctx == nil {
      cmd.ctx = c.ctx
   }

   err = cmd.execute(flags)
   if err != nil {
      // Always show help if requested, even if SilenceErrors is in
      // effect
      if errors.Is(err, flag.ErrHelp) {
         cmd.HelpFunc()(cmd, args)
         return cmd, nil
      }

      // If root command has SilenceErrors flagged,
      // all subcommands should respect it
      if !cmd.SilenceErrors && !c.SilenceErrors {
         c.PrintErrln("Error:", err.Error())
      }

      // If root command has SilenceUsage flagged,
      // all subcommands should respect it
      if !cmd.SilenceUsage && !c.SilenceUsage {
         c.Println(cmd.UsageString())
      }
   }
   return cmd, err
}
```

## 3.4 Command execute



```go

func (c *Command) execute(a []string) (err error) {
	if c == nil {
		return fmt.Errorf("Called Execute() on a nil Command")
	}

	if len(c.Deprecated) > 0 {
		c.Printf("Command %q is deprecated, %s\n", c.Name(), c.Deprecated)
	}

	// initialize help and version flag at the last point possible to allow for user
	// overriding
	c.InitDefaultHelpFlag()
	c.InitDefaultVersionFlag()
	// 1. 解析参数
	err = c.ParseFlags(a)
	if err != nil {
		return c.FlagErrorFunc()(c, err)
	}

	// If help is called, regardless of other flags, return we want help.
	// Also say we need help if the command isn't runnable.
	helpVal, err := c.Flags().GetBool("help")
	if err != nil {
		// should be impossible to get here as we always declare a help
		// flag in InitDefaultHelpFlag()
		c.Println("\"help\" flag declared as non-bool. Please correct your code")
		return err
	}

	if helpVal {
		return flag.ErrHelp
	}

	// for back-compat, only add version flag behavior if version is defined
	if c.Version != "" {
		versionVal, err := c.Flags().GetBool("version")
		if err != nil {
			c.Println("\"version\" flag declared as non-bool. Please correct your code")
			return err
		}
		if versionVal {
			err := tmpl(c.OutOrStdout(), c.VersionTemplate(), c)
			if err != nil {
				c.Println(err)
			}
			return err
		}
	}
	// 2.判断cmd是否设置了 run, runE函数。
	if !c.Runnable() {
		return flag.ErrHelp
	}
	// 3.运行设置的初始化函数
	c.preRun()

	defer c.postRun()

	argWoFlags := c.Flags().Args()
	if c.DisableFlagParsing {
		argWoFlags = a
	}

	if err := c.ValidateArgs(argWoFlags); err != nil {
		return err
	}
    // 4. 开始运行 Run，或者RunE函数。可以看出来，RunE函数的优先级是大于Run的。
	for p := c; p != nil; p = p.Parent() {
		if p.PersistentPreRunE != nil {
			if err := p.PersistentPreRunE(c, argWoFlags); err != nil {
				return err
			}
			break
		} else if p.PersistentPreRun != nil {
			p.PersistentPreRun(c, argWoFlags)
			break
		}
	}
	if c.PreRunE != nil {
		if err := c.PreRunE(c, argWoFlags); err != nil {
			return err
		}
	} else if c.PreRun != nil {
		c.PreRun(c, argWoFlags)
	}

	if err := c.ValidateRequiredFlags(); err != nil {
		return err
	}
	if err := c.ValidateFlagGroups(); err != nil {
		return err
	}

	if c.RunE != nil {
		if err := c.RunE(c, argWoFlags); err != nil {
			return err
		}
	} else {
		c.Run(c, argWoFlags)
	}
	if c.PostRunE != nil {
		if err := c.PostRunE(c, argWoFlags); err != nil {
			return err
		}
	} else if c.PostRun != nil {
		c.PostRun(c, argWoFlags)
	}
	for p := c; p != nil; p = p.Parent() {
		if p.PersistentPostRunE != nil {
			if err := p.PersistentPostRunE(c, argWoFlags); err != nil {
				return err
			}
			break
		} else if p.PersistentPostRun != nil {
			p.PersistentPostRun(c, argWoFlags)
			break
		}
	}

	return nil
}
```

总结

（1）这里主要就是利用corba工具，进行初始化。options.NewServerRunOptions 函数将 corba和 kube-apiserver的参数进行了解耦。

（2）Execute()里面才会真正的进行参数解析，所以Run函数外面的都是没有解析的值。打印出来确实都是默认值。Run函数里面的都是参数解析完的。

# 4 RunE

这个是NewAPIServerCommand中定义的RunE函数。

```go
		RunE: func(cmd *cobra.Command, args []string) error {
			//该函数检查命令行参数中是否包含版本标志（如 --version）。如果包含，它会打印版本信息并退出程序。
			verflag.PrintAndExitIfRequested()
			fs := cmd.Flags()

			// Activate logging as soon as possible, after that
			// show flags with the final logging configuration.
			if err := logsapi.ValidateAndApply(s.Logs, utilfeature.DefaultFeatureGate); err != nil {
				return err
			}
			cliflag.PrintFlags(fs)

			// set default options
			//成配置选项的初始化和验证
			completedOptions, err := s.Complete()
			if err != nil {
				return err
			}

			// validate options
			//验证完成的选项是否正确
			if errs := completedOptions.Validate(); len(errs) != 0 {
				return utilerrors.NewAggregate(errs)
			}
			// add feature enablement metrics
			//添加功能启用的度量指标
			utilfeature.DefaultMutableFeatureGate.AddMetrics()
			//调用 Run 函数，并传入完成的选项和信号处理器。Run 函数是实际执行 API 服务器的主要逻辑，genericapiserver.SetupSignalHandler() 用于处理系统信号（如终止信号）以便优雅地关闭服务器。
			return Run(completedOptions, genericapiserver.SetupSignalHandler())
```

# 5.总结

kube-apiserver启动过程分为8个步骤。这里先分析到前两个

（1）资源注册

（2）Cobra命令行参数解析

通过这个分析，了解到了apiserver是如何感知pod，deploy等内置资源的存在