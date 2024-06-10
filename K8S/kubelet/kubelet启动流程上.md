# 1.kubelet入口函数

```go
func main() {
    //初始化一个新的Kubelet命令，通常会设置运行Kubelet所需的所有必要配置、命令行参数和依赖项。
	command := app.NewKubeletCommand()
	//命令实际执行的地方。这个函数可能处理Kubelet的实际运行，处理请求处理、容器的生命周期管理等。
	code := cli.Run(command)
	os.Exit(code)
}
```

# 2. NewKubeletCommand函数

该函数整体逻辑如下：`当前不关心初始化，验证等逻辑，现在直接奔 Run(kubeletServer, kubeletDeps, stopCh) 去`

（1）初始化参数解析，初始化cleanFlagSet，kubeletFlags，kubeletConfig。这些都是初始化kubelet时要用到的

（2）定义一个cobra命令，这里核心就是 Run函数。Run函数的核心逻辑如下：

- 针对不规范的参数输入，或者help, version情况输出帮助或者版本信息
- 加载并验证 kubelet-config是否规范
- 更加kubelet-config生成kubeletServer和kubeletDeps，这个是生成kubelet的必要条件
- 运行kubelet，核心是Run函数

```go
func NewKubeletCommand() *cobra.Command {
	cleanFlagSet := pflag.NewFlagSet(componentKubelet, pflag.ContinueOnError)
	//设置标志集合，遇到错误不退出继续解析
	cleanFlagSet.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	//格式化命令行参数，处理像 --some-flag=value1,value2 这样的参数，将其拆分为多个单独的参数，例如将 --foo=bar 转换为 --foo bar
	kubeletFlags := options.NewKubeletFlags()
	/*
		创建一个包含kubelet启动标志的结构体
		return &KubeletFlags{
			ContainerRuntimeOptions: *NewContainerRuntimeOptions(),
			CertDirectory:           "/var/lib/kubelet/pki",         //kubelet使用的TLS证书存储的目录
			RootDirectory:           filepath.Clean(defaultRootDir), //指定kubelet的根目录，用于存放kubelet运行时需要的数据
			MaxContainerCount:       -1,                             //指定一个节点上可以存在的最大容器数量，-1通常表示没有限制
			MaxPerPodContainerCount: 1,                              //指定一个pod中可以存在的最大容器数量，默认为1
			MinimumGCAge:            metav1.Duration{Duration: 0},   //指定kubelet进行垃圾回收的最小时间间隔 ,0表示没有时间限制，垃圾收集可以立即执行。
			RegisterSchedulable:     true,                           //指定kubelet是否可以作为调度节点
			NodeLabels:              make(map[string]string),        //指定节点标签
		}
	*/
	kubeletConfig, err := options.NewKubeletConfiguration()
	//创建kubelet配置对象
	if err != nil {
		klog.ErrorS(err, "Failed to create a new kubelet configuration")
		os.Exit(1)
	}

	cmd := &cobra.Command{
		/*
			use 代表命令行名称kubelet
			long 代表命令行描述信息
			DisableFlagParsing：设置为true，意味着禁用cobra的自动标志解析。这是因为kubelet需要根据特定的优先级手动解析命令行参数。
			SilenceUsage：当命令出错时，不显示使用帮助信息。这通常用于减少错误输出中的干扰信息。
		*/
		Use: componentKubelet,
		Long: `The kubelet is the primary "node agent" that runs on each
node. It can register the node with the apiserver using one of: the hostname; a flag to
override the hostname; or specific logic for a cloud provider.

The kubelet works in terms of a PodSpec. A PodSpec is a YAML or JSON object
that describes a pod. The kubelet takes a set of PodSpecs that are provided through
various mechanisms (primarily through the apiserver) and ensures that the containers
described in those PodSpecs are running and healthy. The kubelet doesn't manage
containers which were not created by Kubernetes.

Other than from an PodSpec from the apiserver, there are two ways that a container
manifest can be provided to the Kubelet.

File: Path passed as a flag on the command line. Files under this path will be monitored
periodically for updates. The monitoring period is 20s by default and is configurable
via a flag.

HTTP endpoint: HTTP endpoint passed as a parameter on the command line. This endpoint
is checked every 20 seconds (also configurable with a flag).`,
		// The Kubelet has special flag parsing requirements to enforce flag precedence rules,
		// so we do all our parsing manually in Run, below.
		// DisableFlagParsing=true provides the full set of flags passed to the kubelet in the
		// `args` arg to Run, without Cobra's interference.
		DisableFlagParsing: true,
		SilenceUsage:       true,
		//rune核心启动函数
		RunE: func(cmd *cobra.Command, args []string) error {
			// initial flag parse, since we disable cobra's flag parsing
			if err := cleanFlagSet.Parse(args); err != nil {
				return fmt.Errorf("failed to parse kubelet flag: %w", err)
			}
			//解析命令行参数,由于DisableFlagParsing已启用，需要手动解析这些参数。
			// check if there are non-flag arguments in the command line
			cmds := cleanFlagSet.Args()
			if len(cmds) > 0 {
				return fmt.Errorf("unknown command %+s", cmds[0])
			}
			//检查是否有非标志参数（如子命令），如果有，返回错误。
			// short-circuit on help
			help, err := cleanFlagSet.GetBool("help")
			if err != nil {
				return errors.New(`"help" flag is non-bool, programmer error, please correct`)
			}
			if help {
				return cmd.Help()
			}
			//检查是否请求了帮助信息，如果是，则显示帮助信息并退出。
			// short-circuit on verflag
			verflag.PrintAndExitIfRequested()

			// set feature gates from initial flags-based config
			if err := utilfeature.DefaultMutableFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
				return fmt.Errorf("failed to set feature gates from initial flags-based config: %w", err)
			}

			// validate the initial KubeletFlags
			if err := options.ValidateKubeletFlags(kubeletFlags); err != nil {
				return fmt.Errorf("failed to validate kubelet flags: %w", err)
			}

			if cleanFlagSet.Changed("pod-infra-container-image") {
				klog.InfoS("--pod-infra-container-image will not be pruned by the image garbage collector in kubelet and should also be set in the remote runtime")
				_ = cmd.Flags().MarkDeprecated("pod-infra-container-image", "--pod-infra-container-image will be removed in 1.30. Image garbage collector will get sandbox image information from CRI.")
			}
			//设置特性门控。这些是从命令行参数中读取的功能标志，控制kubelet的特定功能是否启用。
			// 加载 kubelet 配置文件（如果提供）
			if len(kubeletFlags.KubeletConfigFile) > 0 {
				kubeletConfig, err = loadConfigFile(kubeletFlags.KubeletConfigFile)
				if err != nil {
					return fmt.Errorf("failed to load kubelet config file, path: %s, error: %w", kubeletFlags.KubeletConfigFile, err)
				}
			}
			// 如果设置了 --config-dir，则合并 kubelet 配置
			if len(kubeletFlags.KubeletDropinConfigDirectory) > 0 {
				if err := mergeKubeletConfigurations(kubeletConfig, kubeletFlags.KubeletDropinConfigDirectory); err != nil {
					return fmt.Errorf("failed to merge kubelet configs: %w", err)
				}
			}

			if len(kubeletFlags.KubeletConfigFile) > 0 || len(kubeletFlags.KubeletDropinConfigDirectory) > 0 {
				// We must enforce flag precedence by re-parsing the command line into the new object.
				// This is necessary to preserve backwards-compatibility across binary upgrades.
				// See issue #56171 for more details.
				if err := kubeletConfigFlagPrecedence(kubeletConfig, args); err != nil {
					return fmt.Errorf("failed to precedence kubeletConfigFlag: %w", err)
				}
				// update feature gates based on new config
				if err := utilfeature.DefaultMutableFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
					return fmt.Errorf("failed to set feature gates from initial flags-based config: %w", err)
				}
			}

			// Config and flags parsed, now we can initialize logging.
			logs.InitLogs()
			if err := logsapi.ValidateAndApplyAsField(&kubeletConfig.Logging, utilfeature.DefaultFeatureGate, field.NewPath("logging")); err != nil {
				return fmt.Errorf("initialize logging: %v", err)
			}
			cliflag.PrintFlags(cleanFlagSet)

			// We always validate the local configuration (command line + config file).
			// This is the default "last-known-good" config for dynamic config, and must always remain valid.
			if err := kubeletconfigvalidation.ValidateKubeletConfiguration(kubeletConfig, utilfeature.DefaultFeatureGate); err != nil {
				return fmt.Errorf("failed to validate kubelet configuration, error: %w, path: %s", err, kubeletConfig)
			}
			//验证kubelet配置的有效性。这是确保配置符合预期和安全要求的关键步骤。
			if (kubeletConfig.KubeletCgroups != "" && kubeletConfig.KubeReservedCgroup != "") && (strings.Index(kubeletConfig.KubeletCgroups, kubeletConfig.KubeReservedCgroup) != 0) {
				klog.InfoS("unsupported configuration:KubeletCgroups is not within KubeReservedCgroup")
			}
			// 从 kubeletFlags 和 kubeletConfig 构造一个 KubeletServer
			kubeletServer := &options.KubeletServer{
				KubeletFlags:         *kubeletFlags,
				KubeletConfiguration: *kubeletConfig,
			}

			// use kubeletServer to construct the default KubeletDeps
			kubeletDeps, err := UnsecuredDependencies(kubeletServer, utilfeature.DefaultFeatureGate)
			if err != nil {
				return fmt.Errorf("failed to construct kubelet dependencies: %w", err)
			}
			//构造kubelet的依赖关系。这包括网络、存储、监控等外部依赖，这是kubelet正常工作的基础。kubeletDeps就是kubelet运行所需要的环境依赖的结构体
			if err := checkPermissions(); err != nil {
				klog.ErrorS(err, "kubelet running with insufficient permissions")
			}

			// make the kubelet's config safe for logging
			config := kubeletServer.KubeletConfiguration.DeepCopy()
			for k := range config.StaticPodURLHeader {
				config.StaticPodURLHeader[k] = []string{"<masked>"}
			}
			// log the kubelet's config for inspection
			klog.V(5).InfoS("KubeletConfiguration", "configuration", klog.Format(config))

			// 创建一个用于接收系统信号的上下文，这使得kubelet可以优雅地响应如SIGTERM等终止信号。
			ctx := genericapiserver.SetupSignalContext()

			utilfeature.DefaultMutableFeatureGate.AddMetrics()
			//调用是为了确保相关的度量（metrics）被记录，这对于监控和分析kubelet的性能非常重要。
			// 运行kubelet，run函数是kubelet启动的核心
			return Run(ctx, kubeletServer, kubeletDeps, utilfeature.DefaultFeatureGate)
		},
	}

	// keep cleanFlagSet separate, so Cobra doesn't pollute it with the global flags
	kubeletFlags.AddFlags(cleanFlagSet)
	options.AddKubeletConfigFlags(cleanFlagSet, kubeletConfig)
	options.AddGlobalFlags(cleanFlagSet)
	cleanFlagSet.BoolP("help", "h", false, fmt.Sprintf("help for %s", cmd.Name()))

	// ugly, but necessary, because Cobra's default UsageFunc and HelpFunc pollute the flagset with global flags
	const usageFmt = "Usage:\n  %s\n\nFlags:\n%s"
	cmd.SetUsageFunc(func(cmd *cobra.Command) error {
		fmt.Fprintf(cmd.OutOrStderr(), usageFmt, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
		return nil
	})
	cmd.SetHelpFunc(func(cmd *cobra.Command, args []string) {
		fmt.Fprintf(cmd.OutOrStdout(), "%s\n\n"+usageFmt, cmd.Long, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
	})

	return cmd
}
```

# 3. Run(kubeletServer, kubeletDeps, stopCh) 函数

核心就是调用

```go
func Run(ctx context.Context, s *options.KubeletServer, kubeDeps *kubelet.Dependencies, featureGate featuregate.FeatureGate) error {
	// To help debugging, immediately log version
	klog.InfoS("Kubelet version", "kubeletVersion", version.Get())
	//记录Kubelet的版本
	klog.InfoS("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))
	//记录垃圾回收阈值GOGC，最大并行处理数（GOMAXPROCS），追踪回溯设置（GOTRACEDBACK）
	if err := initForOS(s.KubeletFlags.WindowsService, s.KubeletFlags.WindowsPriorityClass); err != nil {
		return fmt.Errorf("failed OS init: %w", err)
	}
	//initForOS函数针对特定的操作系统环境执行初始化操作。在这个例子中，如果Kubelet在Windows上运行，可能需要配置为Windows服务或设置进程优先级类别
	if err := run(ctx, s, kubeDeps, featureGate); err != nil {
		return fmt.Errorf("failed to run Kubelet: %w", err)
	}
	//run函数实际上执行了Kubelet的主操作逻辑。
	return nil
}
```

这里先看一下函数参数 s-kubeletServer和kubeletDeps是什么

## 3.1. KubeletServer

KubeletServer 就是配置参数

基本看参数名字和注释就能知道什么意思

```go
type KubeletFlags struct {
	KubeConfig          string
	BootstrapKubeconfig string

	// HostnameOverride is the hostname used to identify the kubelet instead
	// of the actual hostname.
	HostnameOverride string
	// NodeIP is IP address of the node.
	// If set, kubelet will use this IP address for the node.
	NodeIP string

	// Container-runtime-specific options.
	config.ContainerRuntimeOptions

	// certDirectory is the directory where the TLS certs are located.
	// If tlsCertFile and tlsPrivateKeyFile are provided, this flag will be ignored.
	CertDirectory string

	// cloudProvider is the provider for cloud services.
	// +optional
	CloudProvider string

	// cloudConfigFile is the path to the cloud provider configuration file.
	// +optional
	CloudConfigFile string

	// rootDirectory is the directory path to place kubelet files (volume
	// mounts,etc).
	RootDirectory string

	// The Kubelet will load its initial configuration from this file.
	// The path may be absolute or relative; relative paths are under the Kubelet's current working directory.
	// Omit this flag to use the combination of built-in default configuration values and flags.
	KubeletConfigFile string

	// kubeletDropinConfigDirectory is a path to a directory to specify dropins allows the user to optionally specify
	// additional configs to overwrite what is provided by default and in the KubeletConfigFile flag
	KubeletDropinConfigDirectory string

	// WindowsService should be set to true if kubelet is running as a service on Windows.
	// Its corresponding flag only gets registered in Windows builds.
	WindowsService bool

	// WindowsPriorityClass sets the priority class associated with the Kubelet process
	// Its corresponding flag only gets registered in Windows builds
	// The default priority class associated with any process in Windows is NORMAL_PRIORITY_CLASS. Keeping it as is
	// to maintain backwards compatibility.
	// Source: https://docs.microsoft.com/en-us/windows/win32/procthread/scheduling-priorities
	WindowsPriorityClass string

	// experimentalMounterPath is the path of mounter binary. Leave empty to use the default mount path
	ExperimentalMounterPath string
	// This flag, if set, will avoid including `EvictionHard` limits while computing Node Allocatable.
	// Refer to [Node Allocatable](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable) doc for more information.
	ExperimentalNodeAllocatableIgnoreEvictionThreshold bool
	// Node Labels are the node labels to add when registering the node in the cluster
	NodeLabels map[string]string
	// lockFilePath is the path that kubelet will use to as a lock file.
	// It uses this file as a lock to synchronize with other kubelet processes
	// that may be running.
	LockFilePath string
	// ExitOnLockContention is a flag that signifies to the kubelet that it is running
	// in "bootstrap" mode. This requires that 'LockFilePath' has been set.
	// This will cause the kubelet to listen to inotify events on the lock file,
	// releasing it and exiting when another process tries to open that file.
	ExitOnLockContention bool
	// DEPRECATED FLAGS
	// minimumGCAge is the minimum age for a finished container before it is
	// garbage collected.
	MinimumGCAge metav1.Duration
	// maxPerPodContainerCount is the maximum number of old instances to
	// retain per container. Each container takes up some disk space.
	MaxPerPodContainerCount int32
	// maxContainerCount is the maximum number of old instances of containers
	// to retain globally. Each container takes up some disk space.
	MaxContainerCount int32
	// registerSchedulable tells the kubelet to register the node as
	// schedulable. Won't have any effect if register-node is false.
	// DEPRECATED: use registerWithTaints instead
	RegisterSchedulable bool
	// This flag, if set, instructs the kubelet to keep volumes from terminated pods mounted to the node.
	// This can be useful for debugging volume related issues.
	KeepTerminatedPodVolumes bool
	// SeccompDefault enables the use of `RuntimeDefault` as the default seccomp profile for all workloads on the node.
	SeccompDefault bool
}
```

```go
// KubeletConfiguration contains the configuration for the Kubelet
type KubeletConfiguration struct {
	metav1.TypeMeta

	// enableServer enables Kubelet's secured server.
	// Note: Kubelet's insecure port is controlled by the readOnlyPort option.
	EnableServer bool
	// staticPodPath is the path to the directory containing local (static) pods to
	// run, or the path to a single static pod file.
	StaticPodPath string
	// podLogsDir is a custom root directory path kubelet will use to place pod's log files.
	// Default: "/var/log/pods/"
	// Note: it is not recommended to use the temp folder as a log directory as it may cause
	// unexpected behavior in many places.
	PodLogsDir string
	// syncFrequency is the max period between synchronizing running
	// containers and config
	SyncFrequency metav1.Duration
	// fileCheckFrequency is the duration between checking config files for
	// new data
	FileCheckFrequency metav1.Duration
	// httpCheckFrequency is the duration between checking http for new data
	HTTPCheckFrequency metav1.Duration
	// staticPodURL is the URL for accessing static pods to run
	StaticPodURL string
	// staticPodURLHeader is a map of slices with HTTP headers to use when accessing the podURL
	StaticPodURLHeader map[string][]string `datapolicy:"token"`
	// address is the IP address for the Kubelet to serve on (set to 0.0.0.0
	// for all interfaces)
	Address string
	// port is the port for the Kubelet to serve on.
	Port int32
	// readOnlyPort is the read-only port for the Kubelet to serve on with
	// no authentication/authorization (set to 0 to disable)
	ReadOnlyPort int32
	// volumePluginDir is the full path of the directory in which to search
	// for additional third party volume plugins.
	VolumePluginDir string
	// providerID, if set, sets the unique id of the instance that an external provider (i.e. cloudprovider)
	// can use to identify a specific node
	ProviderID string
	// tlsCertFile is the file containing x509 Certificate for HTTPS.  (CA cert,
	// if any, concatenated after server cert). If tlsCertFile and
	// tlsPrivateKeyFile are not provided, a self-signed certificate
	// and key are generated for the public address and saved to the directory
	// passed to the Kubelet's --cert-dir flag.
	TLSCertFile string
	// tlsPrivateKeyFile is the file containing x509 private key matching tlsCertFile
	TLSPrivateKeyFile string
	// TLSCipherSuites is the list of allowed cipher suites for the server.
	// Note that TLS 1.3 ciphersuites are not configurable.
	// Values are from tls package constants (https://golang.org/pkg/crypto/tls/#pkg-constants).
	TLSCipherSuites []string
	// TLSMinVersion is the minimum TLS version supported.
	// Values are from tls package constants (https://golang.org/pkg/crypto/tls/#pkg-constants).
	TLSMinVersion string
	// rotateCertificates enables client certificate rotation. The Kubelet will request a
	// new certificate from the certificates.k8s.io API. This requires an approver to approve the
	// certificate signing requests.
	RotateCertificates bool
	// serverTLSBootstrap enables server certificate bootstrap. Instead of self
	// signing a serving certificate, the Kubelet will request a certificate from
	// the certificates.k8s.io API. This requires an approver to approve the
	// certificate signing requests. The RotateKubeletServerCertificate feature
	// must be enabled.
	ServerTLSBootstrap bool
	// authentication specifies how requests to the Kubelet's server are authenticated
	Authentication KubeletAuthentication
	// authorization specifies how requests to the Kubelet's server are authorized
	Authorization KubeletAuthorization
	// registryPullQPS is the limit of registry pulls per second.
	// Set to 0 for no limit.
	RegistryPullQPS int32
	// registryBurst is the maximum size of bursty pulls, temporarily allows
	// pulls to burst to this number, while still not exceeding registryPullQPS.
	// Only used if registryPullQPS > 0.
	RegistryBurst int32
	// eventRecordQPS is the maximum event creations per second. If 0, there
	// is no limit enforced.
	EventRecordQPS int32
	// eventBurst is the maximum size of a burst of event creations, temporarily
	// allows event creations to burst to this number, while still not exceeding
	// eventRecordQPS. Only used if eventRecordQPS > 0.
	EventBurst int32
	// enableDebuggingHandlers enables server endpoints for log collection
	// and local running of containers and commands
	EnableDebuggingHandlers bool
	// enableContentionProfiling enables block profiling, if enableDebuggingHandlers is true.
	EnableContentionProfiling bool
	// healthzPort is the port of the localhost healthz endpoint (set to 0 to disable)
	HealthzPort int32
	// healthzBindAddress is the IP address for the healthz server to serve on
	HealthzBindAddress string
	// oomScoreAdj is The oom-score-adj value for kubelet process. Values
	// must be within the range [-1000, 1000].
	OOMScoreAdj int32
	// clusterDomain is the DNS domain for this cluster. If set, kubelet will
	// configure all containers to search this domain in addition to the
	// host's search domains.
	ClusterDomain string
	// clusterDNS is a list of IP addresses for a cluster DNS server. If set,
	// kubelet will configure all containers to use this for DNS resolution
	// instead of the host's DNS servers.
	ClusterDNS []string
	// streamingConnectionIdleTimeout is the maximum time a streaming connection
	// can be idle before the connection is automatically closed.
	StreamingConnectionIdleTimeout metav1.Duration
	// nodeStatusUpdateFrequency is the frequency that kubelet computes node
	// status. If node lease feature is not enabled, it is also the frequency that
	// kubelet posts node status to master. In that case, be cautious when
	// changing the constant, it must work with nodeMonitorGracePeriod in nodecontroller.
	NodeStatusUpdateFrequency metav1.Duration
	// nodeStatusReportFrequency is the frequency that kubelet posts node
	// status to master if node status does not change. Kubelet will ignore this
	// frequency and post node status immediately if any change is detected. It is
	// only used when node lease feature is enabled.
	NodeStatusReportFrequency metav1.Duration
	// nodeLeaseDurationSeconds is the duration the Kubelet will set on its corresponding Lease.
	NodeLeaseDurationSeconds int32
	// ImageMinimumGCAge is the minimum age for an unused image before it is
	// garbage collected.
	ImageMinimumGCAge metav1.Duration
	// ImageMaximumGCAge is the maximum age an image can be unused before it is garbage collected.
	// The default of this field is "0s", which disables this field--meaning images won't be garbage
	// collected based on being unused for too long.
	ImageMaximumGCAge metav1.Duration
	// imageGCHighThresholdPercent is the percent of disk usage after which
	// image garbage collection is always run. The percent is calculated as
	// this field value out of 100.
	ImageGCHighThresholdPercent int32
	// imageGCLowThresholdPercent is the percent of disk usage before which
	// image garbage collection is never run. Lowest disk usage to garbage
	// collect to. The percent is calculated as this field value out of 100.
	ImageGCLowThresholdPercent int32
	// How frequently to calculate and cache volume disk usage for all pods
	VolumeStatsAggPeriod metav1.Duration
	// KubeletCgroups is the absolute name of cgroups to isolate the kubelet in
	KubeletCgroups string
	// SystemCgroups is absolute name of cgroups in which to place
	// all non-kernel processes that are not already in a container. Empty
	// for no container. Rolling back the flag requires a reboot.
	SystemCgroups string
	// CgroupRoot is the root cgroup to use for pods.
	// If CgroupsPerQOS is enabled, this is the root of the QoS cgroup hierarchy.
	CgroupRoot string
	// Enable QoS based Cgroup hierarchy: top level cgroups for QoS Classes
	// And all Burstable and BestEffort pods are brought up under their
	// specific top level QoS cgroup.
	CgroupsPerQOS bool
	// driver that the kubelet uses to manipulate cgroups on the host (cgroupfs or systemd)
	CgroupDriver string
	// CPUManagerPolicy is the name of the policy to use.
	// Requires the CPUManager feature gate to be enabled.
	CPUManagerPolicy string
	// CPUManagerPolicyOptions is a set of key=value which 	allows to set extra options
	// to fine tune the behaviour of the cpu manager policies.
	// Requires  both the "CPUManager" and "CPUManagerPolicyOptions" feature gates to be enabled.
	CPUManagerPolicyOptions map[string]string
	// CPU Manager reconciliation period.
	// Requires the CPUManager feature gate to be enabled.
	CPUManagerReconcilePeriod metav1.Duration
	// MemoryManagerPolicy is the name of the policy to use.
	// Requires the MemoryManager feature gate to be enabled.
	MemoryManagerPolicy string
	// TopologyManagerPolicy is the name of the policy to use.
	TopologyManagerPolicy string
	// TopologyManagerScope represents the scope of topology hint generation
	// that topology manager requests and hint providers generate.
	// Default: "container"
	// +optional
	TopologyManagerScope string
	// TopologyManagerPolicyOptions is a set of key=value which allows to set extra options
	// to fine tune the behaviour of the topology manager policies.
	// Requires  both the "TopologyManager" and "TopologyManagerPolicyOptions" feature gates to be enabled.
	TopologyManagerPolicyOptions map[string]string
	// Map of QoS resource reservation percentages (memory only for now).
	// Requires the QOSReserved feature gate to be enabled.
	QOSReserved map[string]string
	// runtimeRequestTimeout is the timeout for all runtime requests except long running
	// requests - pull, logs, exec and attach.
	RuntimeRequestTimeout metav1.Duration
	// hairpinMode specifies how the Kubelet should configure the container
	// bridge for hairpin packets.
	// Setting this flag allows endpoints in a Service to loadbalance back to
	// themselves if they should try to access their own Service. Values:
	//   "promiscuous-bridge": make the container bridge promiscuous.
	//   "hairpin-veth":       set the hairpin flag on container veth interfaces.
	//   "none":               do nothing.
	// Generally, one must set --hairpin-mode=hairpin-veth to achieve hairpin NAT,
	// because promiscuous-bridge assumes the existence of a container bridge named cbr0.
	HairpinMode string
	// maxPods is the number of pods that can run on this Kubelet.
	MaxPods int32
	// The CIDR to use for pod IP addresses, only used in standalone mode.
	// In cluster mode, this is obtained from the master.
	PodCIDR string
	// The maximum number of processes per pod.  If -1, the kubelet defaults to the node allocatable pid capacity.
	PodPidsLimit int64
	// ResolverConfig is the resolver configuration file used as the basis
	// for the container DNS resolution configuration.
	ResolverConfig string
	// RunOnce causes the Kubelet to check the API server once for pods,
	// run those in addition to the pods specified by static pod files, and exit.
	RunOnce bool
	// cpuCFSQuota enables CPU CFS quota enforcement for containers that
	// specify CPU limits
	CPUCFSQuota bool
	// CPUCFSQuotaPeriod sets the CPU CFS quota period value, cpu.cfs_period_us, defaults to 100ms
	CPUCFSQuotaPeriod metav1.Duration
	// maxOpenFiles is Number of files that can be opened by Kubelet process.
	MaxOpenFiles int64
	// nodeStatusMaxImages caps the number of images reported in Node.Status.Images.
	NodeStatusMaxImages int32
	// contentType is contentType of requests sent to apiserver.
	ContentType string
	// kubeAPIQPS is the QPS to use while talking with kubernetes apiserver
	KubeAPIQPS int32
	// kubeAPIBurst is the burst to allow while talking with kubernetes
	// apiserver
	KubeAPIBurst int32
	// serializeImagePulls when enabled, tells the Kubelet to pull images one at a time.
	SerializeImagePulls bool
	// MaxParallelImagePulls sets the maximum number of image pulls in parallel.
	MaxParallelImagePulls *int32
	// Map of signal names to quantities that defines hard eviction thresholds. For example: {"memory.available": "300Mi"}.
	// Some default signals are Linux only: nodefs.inodesFree
	EvictionHard map[string]string
	// Map of signal names to quantities that defines soft eviction thresholds.  For example: {"memory.available": "300Mi"}.
	EvictionSoft map[string]string
	// Map of signal names to quantities that defines grace periods for each soft eviction signal. For example: {"memory.available": "30s"}.
	EvictionSoftGracePeriod map[string]string
	// Duration for which the kubelet has to wait before transitioning out of an eviction pressure condition.
	EvictionPressureTransitionPeriod metav1.Duration
	// Maximum allowed grace period (in seconds) to use when terminating pods in response to a soft eviction threshold being met.
	EvictionMaxPodGracePeriod int32
	// Map of signal names to quantities that defines minimum reclaims, which describe the minimum
	// amount of a given resource the kubelet will reclaim when performing a pod eviction while
	// that resource is under pressure. For example: {"imagefs.available": "2Gi"}
	EvictionMinimumReclaim map[string]string
	// podsPerCore is the maximum number of pods per core. Cannot exceed MaxPods.
	// If 0, this field is ignored.
	PodsPerCore int32
	// enableControllerAttachDetach enables the Attach/Detach controller to
	// manage attachment/detachment of volumes scheduled to this node, and
	// disables kubelet from executing any attach/detach operations
	EnableControllerAttachDetach bool
	// protectKernelDefaults, if true, causes the Kubelet to error if kernel
	// flags are not as it expects. Otherwise the Kubelet will attempt to modify
	// kernel flags to match its expectation.
	ProtectKernelDefaults bool
	// If true, Kubelet creates the KUBE-IPTABLES-HINT chain in iptables as a hint to
	// other components about the configuration of iptables on the system.
	MakeIPTablesUtilChains bool
	// iptablesMasqueradeBit formerly controlled the creation of the KUBE-MARK-MASQ
	// chain.
	// Deprecated: no longer has any effect.
	IPTablesMasqueradeBit int32
	// iptablesDropBit formerly controlled the creation of the KUBE-MARK-DROP chain.
	// Deprecated: no longer has any effect.
	IPTablesDropBit int32
	// featureGates is a map of feature names to bools that enable or disable alpha/experimental
	// features. This field modifies piecemeal the built-in default values from
	// "k8s.io/kubernetes/pkg/features/kube_features.go".
	FeatureGates map[string]bool
	// Tells the Kubelet to fail to start if swap is enabled on the node.
	FailSwapOn bool
	// memorySwap configures swap memory available to container workloads.
	// +featureGate=NodeSwap
	// +optional
	MemorySwap MemorySwapConfiguration
	// A quantity defines the maximum size of the container log file before it is rotated. For example: "5Mi" or "256Ki".
	ContainerLogMaxSize string
	// Maximum number of container log files that can be present for a container.
	ContainerLogMaxFiles int32
	// Maximum number of concurrent log rotation workers to spawn for processing the log rotation
	// requests
	ContainerLogMaxWorkers int32
	// Interval at which the container logs are monitored for rotation
	ContainerLogMonitorInterval metav1.Duration
	// ConfigMapAndSecretChangeDetectionStrategy is a mode in which config map and secret managers are running.
	ConfigMapAndSecretChangeDetectionStrategy ResourceChangeDetectionStrategy
	// A comma separated allowlist of unsafe sysctls or sysctl patterns (ending in `*`).
	// Unsafe sysctl groups are `kernel.shm*`, `kernel.msg*`, `kernel.sem`, `fs.mqueue.*`, and `net.*`.
	// These sysctls are namespaced but not allowed by default.
	// For example: "`kernel.msg*,net.ipv4.route.min_pmtu`"
	// +optional
	AllowedUnsafeSysctls []string
	// kernelMemcgNotification if enabled, the kubelet will integrate with the kernel memcg
	// notification to determine if memory eviction thresholds are crossed rather than polling.
	KernelMemcgNotification bool

	/* the following fields are meant for Node Allocatable */

	// A set of ResourceName=ResourceQuantity (e.g. cpu=200m,memory=150G,ephemeral-storage=1G,pid=100) pairs
	// that describe resources reserved for non-kubernetes components.
	// Currently only cpu, memory and local ephemeral storage for root file system are supported.
	// See http://kubernetes.io/docs/user-guide/compute-resources for more detail.
	SystemReserved map[string]string
	// A set of ResourceName=ResourceQuantity (e.g. cpu=200m,memory=150G,ephemeral-storage=1G,pid=100) pairs
	// that describe resources reserved for kubernetes system components.
	// Currently only cpu, memory and local ephemeral storage for root file system are supported.
	// See http://kubernetes.io/docs/user-guide/compute-resources for more detail.
	KubeReserved map[string]string
	// This flag helps kubelet identify absolute name of top level cgroup used to enforce `SystemReserved` compute resource reservation for OS system daemons.
	// Refer to [Node Allocatable](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable) doc for more information.
	SystemReservedCgroup string
	// This flag helps kubelet identify absolute name of top level cgroup used to enforce `KubeReserved` compute resource reservation for Kubernetes node system daemons.
	// Refer to [Node Allocatable](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable) doc for more information.
	KubeReservedCgroup string
	// This flag specifies the various Node Allocatable enforcements that Kubelet needs to perform.
	// This flag accepts a list of options. Acceptable options are `pods`, `system-reserved` & `kube-reserved`.
	// Refer to [Node Allocatable](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable) doc for more information.
	EnforceNodeAllocatable []string
	// This option specifies the cpu list reserved for the host level system threads and kubernetes related threads.
	// This provide a "static" CPU list rather than the "dynamic" list by system-reserved and kube-reserved.
	// This option overwrites CPUs provided by system-reserved and kube-reserved.
	ReservedSystemCPUs string
	// The previous version for which you want to show hidden metrics.
	// Only the previous minor version is meaningful, other values will not be allowed.
	// The format is <major>.<minor>, e.g.: '1.16'.
	// The purpose of this format is make sure you have the opportunity to notice if the next release hides additional metrics,
	// rather than being surprised when they are permanently removed in the release after that.
	ShowHiddenMetricsForVersion string
	// Logging specifies the options of logging.
	// Refer [Logs Options](https://github.com/kubernetes/component-base/blob/master/logs/options.go) for more information.
	Logging logsapi.LoggingConfiguration
	// EnableSystemLogHandler enables /logs handler.
	EnableSystemLogHandler bool
	// EnableSystemLogQuery enables the node log query feature on the /logs endpoint.
	// EnableSystemLogHandler has to be enabled in addition for this feature to work.
	// +featureGate=NodeLogQuery
	// +optional
	EnableSystemLogQuery bool
	// ShutdownGracePeriod specifies the total duration that the node should delay the shutdown and total grace period for pod termination during a node shutdown.
	// Defaults to 0 seconds.
	// +featureGate=GracefulNodeShutdown
	// +optional
	ShutdownGracePeriod metav1.Duration
	// ShutdownGracePeriodCriticalPods specifies the duration used to terminate critical pods during a node shutdown. This should be less than ShutdownGracePeriod.
	// Defaults to 0 seconds.
	// For example, if ShutdownGracePeriod=30s, and ShutdownGracePeriodCriticalPods=10s, during a node shutdown the first 20 seconds would be reserved for gracefully terminating normal pods, and the last 10 seconds would be reserved for terminating critical pods.
	// +featureGate=GracefulNodeShutdown
	// +optional
	ShutdownGracePeriodCriticalPods metav1.Duration
	// ShutdownGracePeriodByPodPriority specifies the shutdown grace period for Pods based
	// on their associated priority class value.
	// When a shutdown request is received, the Kubelet will initiate shutdown on all pods
	// running on the node with a grace period that depends on the priority of the pod,
	// and then wait for all pods to exit.
	// Each entry in the array represents the graceful shutdown time a pod with a priority
	// class value that lies in the range of that value and the next higher entry in the
	// list when the node is shutting down.
	ShutdownGracePeriodByPodPriority []ShutdownGracePeriodByPodPriority
	// ReservedMemory specifies a comma-separated list of memory reservations for NUMA nodes.
	// The parameter makes sense only in the context of the memory manager feature. The memory manager will not allocate reserved memory for container workloads.
	// For example, if you have a NUMA0 with 10Gi of memory and the ReservedMemory was specified to reserve 1Gi of memory at NUMA0,
	// the memory manager will assume that only 9Gi is available for allocation.
	// You can specify a different amount of NUMA node and memory types.
	// You can omit this parameter at all, but you should be aware that the amount of reserved memory from all NUMA nodes
	// should be equal to the amount of memory specified by the node allocatable features(https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable).
	// If at least one node allocatable parameter has a non-zero value, you will need to specify at least one NUMA node.
	// Also, avoid specifying:
	// 1. Duplicates, the same NUMA node, and memory type, but with a different value.
	// 2. zero limits for any memory type.
	// 3. NUMAs nodes IDs that do not exist under the machine.
	// 4. memory types except for memory and hugepages-<size>
	ReservedMemory []MemoryReservation
	// EnableProfiling enables /debug/pprof handler.
	EnableProfilingHandler bool
	// EnableDebugFlagsHandler enables/debug/flags/v handler.
	EnableDebugFlagsHandler bool
	// SeccompDefault enables the use of `RuntimeDefault` as the default seccomp profile for all workloads.
	SeccompDefault bool
	// MemoryThrottlingFactor specifies the factor multiplied by the memory limit or node allocatable memory
	// when setting the cgroupv2 memory.high value to enforce MemoryQoS.
	// Decreasing this factor will set lower high limit for container cgroups and put heavier reclaim pressure
	// while increasing will put less reclaim pressure.
	// See https://kep.k8s.io/2570 for more details.
	// Default: 0.9
	// +featureGate=MemoryQoS
	// +optional
	MemoryThrottlingFactor *float64
	// registerWithTaints are an array of taints to add to a node object when
	// the kubelet registers itself. This only takes effect when registerNode
	// is true and upon the initial registration of the node.
	// +optional
	RegisterWithTaints []v1.Taint
	// registerNode enables automatic registration with the apiserver.
	// +optional
	RegisterNode bool

	// Tracing specifies the versioned configuration for OpenTelemetry tracing clients.
	// See https://kep.k8s.io/2832 for more details.
	// +featureGate=KubeletTracing
	// +optional
	Tracing *tracingapi.TracingConfiguration

	// LocalStorageCapacityIsolation enables local ephemeral storage isolation feature. The default setting is true.
	// This feature allows users to set request/limit for container's ephemeral storage and manage it in a similar way
	// as cpu and memory. It also allows setting sizeLimit for emptyDir volume, which will trigger pod eviction if disk
	// usage from the volume exceeds the limit.
	// This feature depends on the capability of detecting correct root file system disk usage. For certain systems,
	// such as kind rootless, if this capability cannot be supported, the feature LocalStorageCapacityIsolation should be
	// disabled. Once disabled, user should not set request/limit for container's ephemeral storage, or sizeLimit for emptyDir.
	// +optional
	LocalStorageCapacityIsolation bool

	// ContainerRuntimeEndpoint is the endpoint of container runtime.
	// unix domain sockets supported on Linux while npipes and tcp endpoints are supported for windows.
	// Examples:'unix:///path/to/runtime.sock', 'npipe:////./pipe/runtime'
	ContainerRuntimeEndpoint string

	// ImageServiceEndpoint is the endpoint of container image service.
	// If not specified the default value is ContainerRuntimeEndpoint
	// +optional
	ImageServiceEndpoint string
}

```

## 3.2. Dependencies

kubeletDeps就是kubelet运行所需要的环境依赖的结构体

```
type Dependencies struct {
	Options []Option

	// Injected Dependencies
	Auth                      server.AuthInterface
	CAdvisorInterface         cadvisor.Interface
	Cloud                     cloudprovider.Interface
	ContainerManager          cm.ContainerManager
	EventClient               v1core.EventsGetter
	HeartbeatClient           clientset.Interface
	OnHeartbeatFailure        func()
	KubeClient                clientset.Interface
	Mounter                   mount.Interface
	HostUtil                  hostutil.HostUtils
	OOMAdjuster               *oom.OOMAdjuster
	OSInterface               kubecontainer.OSInterface
	PodConfig                 *config.PodConfig
	ProbeManager              prober.Manager
	Recorder                  record.EventRecorder
	Subpather                 subpath.Interface
	TracerProvider            trace.TracerProvider
	VolumePlugins             []volume.VolumePlugin
	DynamicPluginProber       volume.DynamicPluginProber
	TLSOptions                *server.TLSOptions
	RemoteRuntimeService      internalapi.RuntimeService
	RemoteImageService        internalapi.ImageManagerService
	PodStartupLatencyTracker  util.PodStartupLatencyTracker
	NodeStartupLatencyTracker util.NodeStartupLatencyTracker
	// remove it after cadvisor.UsingLegacyCadvisorStats dropped.
	useLegacyCadvisorStats bool
}
```

# 4.func run(ctx context.Context, s *options.KubeletServer, kubeDeps *kubelet.Dependencies, featureGate featuregate.FeatureGate) (err error) 

1. **设置功能开关**:
    - 设置基于初始`KubeletServer`的全局特性开关。
    - 验证初始`KubeletServer`配置是否有效。
2. **文件锁和系统兼容性检查**:
    - 如果指定了锁文件路径并开启了锁文件争用退出选项，尝试获取文件锁。
    - 检查MemoryQoS特性是否在cgroups v1上启用，并发出警告，因为此特性只与Linux上的cgroups v2兼容。
3. **依赖项和客户端初始化**:
    - 判断是否为独立模式（无API服务器连接），并相应配置客户端。
    - 根据配置决定是否初始化云提供商。
    - 初始化认证模块。
    - 获取或生成节点名和各种Kubernetes客户端（如事件、心跳等）。
4. **容器运行环境配置**:
    - 初始化cgroup配置，获取或设置cgroup驱动。
    - 初始化和配置cAdvisor接口，用于容器监控。
    - 配置并初始化Container Manager，管理资源限制、QoS等。
5. **Kubelet运行与健康检查**:
    - 启动Kubelet服务，处理一次性运行或持续运行的逻辑。
    - 设置和启动健康检查端点。
    - 如果是Systemd环境，发送启动完成信号。
6. **事件记录和性能跟踪**:
    - 设置事件记录器。
    - 初始化Pod启动和节点启动延迟跟踪器。
7. **OOM调整和清理**:
    - 应用OOM得分调整，以管理系统内存溢出情况。
8. **运行时服务和特性门控配置**:
    - 进行容器运行时服务预初始化。
    - 根据特性门控配置额外的运行时选项，如CPU管理策略。

此函数涵盖了Kubelet的启动和配置的多个方面，包括资源管理、客户端初始化、安全性和稳定性配置，以及与系统其他部分的交互。

```go
func run(ctx context.Context, s *options.KubeletServer, kubeDeps *kubelet.Dependencies, featureGate featuregate.FeatureGate) (err error) {
	//设置功能开关和验证配置
	// Set global feature gates based on the value on the initial KubeletServer
	err = utilfeature.DefaultMutableFeatureGate.SetFromMap(s.KubeletConfiguration.FeatureGates)
	if err != nil {
		return err
	}
	// validate the initial KubeletServer (we set feature gates first, because this validation depends on feature gates)
	if err := options.ValidateKubeletServer(s); err != nil {
		return err
	}

	// Warn if MemoryQoS enabled with cgroups v1
	if utilfeature.DefaultFeatureGate.Enabled(features.MemoryQoS) &&
		!isCgroup2UnifiedMode() {
		klog.InfoS("Warning: MemoryQoS feature only works with cgroups v2 on Linux, but enabled with cgroups v1")
	}
	// Obtain Kubelet Lock File
	if s.ExitOnLockContention && s.LockFilePath == "" {
		return errors.New("cannot exit on lock file contention: no lock file specified")
	}
	//文件锁获取和系统兼容性检查
	done := make(chan struct{})
	if s.LockFilePath != "" {
		klog.InfoS("Acquiring file lock", "path", s.LockFilePath)
		if err := flock.Acquire(s.LockFilePath); err != nil {
			return fmt.Errorf("unable to acquire file lock on %q: %w", s.LockFilePath, err)
		}
		if s.ExitOnLockContention {
			klog.InfoS("Watching for inotify events", "path", s.LockFilePath)
			if err := watchForLockfileContention(s.LockFilePath, done); err != nil {
				return err
			}
		}
	}

	// Register current configuration with /configz endpoint
	err = initConfigz(&s.KubeletConfiguration)
	if err != nil {
		klog.ErrorS(err, "Failed to register kubelet configuration with configz")
	}

	if len(s.ShowHiddenMetricsForVersion) > 0 {
		metrics.SetShowHidden()
	}

	// About to get clients and such, detect standaloneMode
	standaloneMode := true
	if len(s.KubeConfig) > 0 {
		standaloneMode = false
	}
	// 依赖项和客户端初始化
	if kubeDeps == nil {
		kubeDeps, err = UnsecuredDependencies(s, featureGate)
		if err != nil {
			return err
		}
	}

	if kubeDeps.Cloud == nil {
		if !cloudprovider.IsExternal(s.CloudProvider) {
			cloudprovider.DeprecationWarningForProvider(s.CloudProvider)
			cloud, err := cloudprovider.InitCloudProvider(s.CloudProvider, s.CloudConfigFile)
			if err != nil {
				return err
			}
			if cloud != nil {
				klog.V(2).InfoS("Successfully initialized cloud provider", "cloudProvider", s.CloudProvider, "cloudConfigFile", s.CloudConfigFile)
			}
			kubeDeps.Cloud = cloud
		}
	}

	hostName, err := nodeutil.GetHostname(s.HostnameOverride)
	if err != nil {
		return err
	}
	nodeName, err := getNodeName(kubeDeps.Cloud, hostName)
	if err != nil {
		return err
	}

	// if in standalone mode, indicate as much by setting all clients to nil
	switch {
	case standaloneMode:
		kubeDeps.KubeClient = nil
		kubeDeps.EventClient = nil
		kubeDeps.HeartbeatClient = nil
		klog.InfoS("Standalone mode, no API client")

	case kubeDeps.KubeClient == nil, kubeDeps.EventClient == nil, kubeDeps.HeartbeatClient == nil:
		clientConfig, onHeartbeatFailure, err := buildKubeletClientConfig(ctx, s, kubeDeps.TracerProvider, nodeName)
		if err != nil {
			return err
		}
		if onHeartbeatFailure == nil {
			return errors.New("onHeartbeatFailure must be a valid function other than nil")
		}
		kubeDeps.OnHeartbeatFailure = onHeartbeatFailure

		kubeDeps.KubeClient, err = clientset.NewForConfig(clientConfig)
		if err != nil {
			return fmt.Errorf("failed to initialize kubelet client: %w", err)
		}

		// make a separate client for events
		eventClientConfig := *clientConfig
		eventClientConfig.QPS = float32(s.EventRecordQPS)
		eventClientConfig.Burst = int(s.EventBurst)
		kubeDeps.EventClient, err = v1core.NewForConfig(&eventClientConfig)
		if err != nil {
			return fmt.Errorf("failed to initialize kubelet event client: %w", err)
		}

		// make a separate client for heartbeat with throttling disabled and a timeout attached
		heartbeatClientConfig := *clientConfig
		heartbeatClientConfig.Timeout = s.KubeletConfiguration.NodeStatusUpdateFrequency.Duration
		// The timeout is the minimum of the lease duration and status update frequency
		leaseTimeout := time.Duration(s.KubeletConfiguration.NodeLeaseDurationSeconds) * time.Second
		if heartbeatClientConfig.Timeout > leaseTimeout {
			heartbeatClientConfig.Timeout = leaseTimeout
		}

		heartbeatClientConfig.QPS = float32(-1)
		kubeDeps.HeartbeatClient, err = clientset.NewForConfig(&heartbeatClientConfig)
		if err != nil {
			return fmt.Errorf("failed to initialize kubelet heartbeat client: %w", err)
		}
	}

	if kubeDeps.Auth == nil {
		auth, runAuthenticatorCAReload, err := BuildAuth(nodeName, kubeDeps.KubeClient, s.KubeletConfiguration)
		if err != nil {
			return err
		}
		kubeDeps.Auth = auth
		runAuthenticatorCAReload(ctx.Done())
	}

	if err := kubelet.PreInitRuntimeService(&s.KubeletConfiguration, kubeDeps); err != nil {
		return err
	}

	// Get cgroup driver setting from CRI
	//这部分配置容器运行环境，设置cgroup驱动，初始化cAdvisor接口用于监控容器性能。
	if utilfeature.DefaultFeatureGate.Enabled(features.KubeletCgroupDriverFromCRI) {
		if err := getCgroupDriverFromCRI(ctx, s, kubeDeps); err != nil {
			return err
		}
	}

	var cgroupRoots []string
	nodeAllocatableRoot := cm.NodeAllocatableRoot(s.CgroupRoot, s.CgroupsPerQOS, s.CgroupDriver)
	cgroupRoots = append(cgroupRoots, nodeAllocatableRoot)
	kubeletCgroup, err := cm.GetKubeletContainer(s.KubeletCgroups)
	if err != nil {
		klog.InfoS("Failed to get the kubelet's cgroup. Kubelet system container metrics may be missing.", "err", err)
	} else if kubeletCgroup != "" {
		cgroupRoots = append(cgroupRoots, kubeletCgroup)
	}

	if s.RuntimeCgroups != "" {
		// RuntimeCgroups is optional, so ignore if it isn't specified
		cgroupRoots = append(cgroupRoots, s.RuntimeCgroups)
	}

	if s.SystemCgroups != "" {
		// SystemCgroups is optional, so ignore if it isn't specified
		cgroupRoots = append(cgroupRoots, s.SystemCgroups)
	}

	if kubeDeps.CAdvisorInterface == nil {
		imageFsInfoProvider := cadvisor.NewImageFsInfoProvider(s.ContainerRuntimeEndpoint)
		kubeDeps.CAdvisorInterface, err = cadvisor.New(imageFsInfoProvider, s.RootDirectory, cgroupRoots, cadvisor.UsingLegacyCadvisorStats(s.ContainerRuntimeEndpoint), s.LocalStorageCapacityIsolation)
		if err != nil {
			return err
		}
	}
	// 事件记录和性能跟踪
	// Setup event recorder if required.
	makeEventRecorder(ctx, kubeDeps, nodeName)

	if kubeDeps.ContainerManager == nil {
		if s.CgroupsPerQOS && s.CgroupRoot == "" {
			klog.InfoS("--cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /")
			s.CgroupRoot = "/"
		}

		machineInfo, err := kubeDeps.CAdvisorInterface.MachineInfo()
		if err != nil {
			return err
		}
		reservedSystemCPUs, err := getReservedCPUs(machineInfo, s.ReservedSystemCPUs)
		if err != nil {
			return err
		}
		if reservedSystemCPUs.Size() > 0 {
			// at cmd option validation phase it is tested either --system-reserved-cgroup or --kube-reserved-cgroup is specified, so overwrite should be ok
			klog.InfoS("Option --reserved-cpus is specified, it will overwrite the cpu setting in KubeReserved and SystemReserved", "kubeReserved", s.KubeReserved, "systemReserved", s.SystemReserved)
			if s.KubeReserved != nil {
				delete(s.KubeReserved, "cpu")
			}
			if s.SystemReserved == nil {
				s.SystemReserved = make(map[string]string)
			}
			s.SystemReserved["cpu"] = strconv.Itoa(reservedSystemCPUs.Size())
			klog.InfoS("After cpu setting is overwritten", "kubeReserved", s.KubeReserved, "systemReserved", s.SystemReserved)
		}

		kubeReserved, err := parseResourceList(s.KubeReserved)
		if err != nil {
			return fmt.Errorf("--kube-reserved value failed to parse: %w", err)
		}
		systemReserved, err := parseResourceList(s.SystemReserved)
		if err != nil {
			return fmt.Errorf("--system-reserved value failed to parse: %w", err)
		}
		var hardEvictionThresholds []evictionapi.Threshold
		// If the user requested to ignore eviction thresholds, then do not set valid values for hardEvictionThresholds here.
		if !s.ExperimentalNodeAllocatableIgnoreEvictionThreshold {
			hardEvictionThresholds, err = eviction.ParseThresholdConfig([]string{}, s.EvictionHard, nil, nil, nil)
			if err != nil {
				return err
			}
		}
		experimentalQOSReserved, err := cm.ParseQOSReserved(s.QOSReserved)
		if err != nil {
			return fmt.Errorf("--qos-reserved value failed to parse: %w", err)
		}

		var cpuManagerPolicyOptions map[string]string
		if utilfeature.DefaultFeatureGate.Enabled(features.CPUManagerPolicyOptions) {
			cpuManagerPolicyOptions = s.CPUManagerPolicyOptions
		} else if s.CPUManagerPolicyOptions != nil {
			return fmt.Errorf("CPU Manager policy options %v require feature gates %q, %q enabled",
				s.CPUManagerPolicyOptions, features.CPUManager, features.CPUManagerPolicyOptions)
		}

		var topologyManagerPolicyOptions map[string]string
		if utilfeature.DefaultFeatureGate.Enabled(features.TopologyManagerPolicyOptions) {
			topologyManagerPolicyOptions = s.TopologyManagerPolicyOptions
		} else if s.TopologyManagerPolicyOptions != nil {
			return fmt.Errorf("topology manager policy options %v require feature gates %q enabled",
				s.TopologyManagerPolicyOptions, features.TopologyManagerPolicyOptions)
		}
		if utilfeature.DefaultFeatureGate.Enabled(features.NodeSwap) {
			if !isCgroup2UnifiedMode() && s.MemorySwap.SwapBehavior == kubelettypes.LimitedSwap {
				// This feature is not supported for cgroupv1 so we are failing early.
				return fmt.Errorf("swap feature is enabled and LimitedSwap but it is only supported with cgroupv2")
			}
			if !s.FailSwapOn && s.MemorySwap.SwapBehavior == "" {
				// This is just a log because we are using a default of NoSwap.
				klog.InfoS("NoSwap is set due to memorySwapBehavior not specified", "memorySwapBehavior", s.MemorySwap.SwapBehavior, "FailSwapOn", s.FailSwapOn)
			}
		}
		//NewContainerManager负责初始化容器管理器，配置节点的资源管理策略，例如CPU和内存资源的分配、cgroup配置以及QoS（服务质量）策畔。
		kubeDeps.ContainerManager, err = cm.NewContainerManager(
			kubeDeps.Mounter,
			kubeDeps.CAdvisorInterface,
			cm.NodeConfig{
				NodeName:              nodeName,
				RuntimeCgroupsName:    s.RuntimeCgroups,
				SystemCgroupsName:     s.SystemCgroups,
				KubeletCgroupsName:    s.KubeletCgroups,
				KubeletOOMScoreAdj:    s.OOMScoreAdj,
				CgroupsPerQOS:         s.CgroupsPerQOS,
				CgroupRoot:            s.CgroupRoot,
				CgroupDriver:          s.CgroupDriver,
				KubeletRootDir:        s.RootDirectory,
				ProtectKernelDefaults: s.ProtectKernelDefaults,
				NodeAllocatableConfig: cm.NodeAllocatableConfig{
					KubeReservedCgroupName:   s.KubeReservedCgroup,
					SystemReservedCgroupName: s.SystemReservedCgroup,
					EnforceNodeAllocatable:   sets.New(s.EnforceNodeAllocatable...),
					KubeReserved:             kubeReserved,
					SystemReserved:           systemReserved,
					ReservedSystemCPUs:       reservedSystemCPUs,
					HardEvictionThresholds:   hardEvictionThresholds,
				},
				QOSReserved:                             *experimentalQOSReserved,
				CPUManagerPolicy:                        s.CPUManagerPolicy,
				CPUManagerPolicyOptions:                 cpuManagerPolicyOptions,
				CPUManagerReconcilePeriod:               s.CPUManagerReconcilePeriod.Duration,
				ExperimentalMemoryManagerPolicy:         s.MemoryManagerPolicy,
				ExperimentalMemoryManagerReservedMemory: s.ReservedMemory,
				PodPidsLimit:                            s.PodPidsLimit,
				EnforceCPULimits:                        s.CPUCFSQuota,
				CPUCFSQuotaPeriod:                       s.CPUCFSQuotaPeriod.Duration,
				TopologyManagerPolicy:                   s.TopologyManagerPolicy,
				TopologyManagerScope:                    s.TopologyManagerScope,
				TopologyManagerPolicyOptions:            topologyManagerPolicyOptions,
			},
			s.FailSwapOn,
			kubeDeps.Recorder,
			kubeDeps.KubeClient,
		)

		if err != nil {
			return err
		}
	}

	if kubeDeps.PodStartupLatencyTracker == nil {
		kubeDeps.PodStartupLatencyTracker = kubeletutil.NewPodStartupLatencyTracker()
	}

	if kubeDeps.NodeStartupLatencyTracker == nil {
		kubeDeps.NodeStartupLatencyTracker = kubeletutil.NewNodeStartupLatencyTracker()
	}
	//OOM调整和清理
	// TODO(vmarmol): Do this through container config.
	oomAdjuster := kubeDeps.OOMAdjuster
	if err := oomAdjuster.ApplyOOMScoreAdj(0, int(s.OOMScoreAdj)); err != nil {
		klog.InfoS("Failed to ApplyOOMScoreAdj", "err", err)
	}
	// Kubelet运行与健康检查
	if err := RunKubelet(s, kubeDeps, s.RunOnce); err != nil {
		return err
	}

	if s.HealthzPort > 0 {
		mux := http.NewServeMux()
		healthz.InstallHandler(mux)
		go wait.Until(func() {
			err := http.ListenAndServe(net.JoinHostPort(s.HealthzBindAddress, strconv.Itoa(int(s.HealthzPort))), mux)
			if err != nil {
				klog.ErrorS(err, "Failed to start healthz server")
			}
		}, 5*time.Second, wait.NeverStop)
	}

	if s.RunOnce {
		return nil
	}
	//系统通知和清理逻辑
	// If systemd is used, notify it that we have started
	go daemon.SdNotify(false, "READY=1")

	select {
	case <-done:
		break
	case <-ctx.Done():
		break
	}

	return nil
}
```

# 5. RunKubelet

**获取主机名并初始化事件记录器**：

- 使用 `nodeutil.GetHostname` 获取节点的主机名。如果 `HostnameOverride` 配置项被设置，则使用它作为主机名。
- 初始化事件记录器（Event Recorder），这是一个关键组件，用于将节点事件发送到 Kubernetes 的 API server。这对于系统监控和事件跟踪非常重要。

**设置 Kubelet 的权限**：

- 调用 `capabilities.Initialize` 来设置 kubelet 的运行权限。此设置确保 kubelet 在需要时可以执行特权操作，如直接访问硬件等。

**配置 Kubelet 的工作目录**：

- 设置 kubelet 的根目录，通常位于 `/var/lib/kubelet`。此目录用于存放 kubelet 运行时数据，如容器运行信息、配置文件等。
- 使用 `credentialprovider.SetPreferredDockercfgPath` 来设置 Docker 配置文件的路径，这对于容器镜像的拉取权限验证至关重要。

**创建并初始化 Kubelet 实例**：

- 调用 `createAndInitKubelet` 函数，传入配置和依赖参数，初始化一个 kubelet 实例。这个过程包括设置运行时环境、容器运行时配置、节点特有的配置等。
- 检查是否存在 Pod 配置源 (`PodConfig`)，这是 kubelet 获取 Pod 信息的配置接口。如果不存在，将报错退出。

**根据运行模式启动 Kubelet**：

- 如果设置了 

    ```
    runOnce
    ```

     参数，kubelet 会在启动后处理一次 Pod 更新，然后退出。这通常用于测试或临时场景。

    - 在 runOnce 模式下，kubelet 创建所需的目录结构，监听 Pod 更新信息。一旦接收到 Pod 数据，就启动相应的 Pod 并返回它们的状态信息，然后退出。

- 如果没有设置 

    ```
    runOnce
    ```

    ，kubelet 将以服务（server）模式运行：

    - 调用 `startKubeite` 函数启动 kubelet 主循环。在这个模式下，kubeite 不断监听 Pod 配置的更新，管理 Pod 的生命周期（创建、更新、删除等）。

**日志记录**：

- 使用 `klog` 记录关键步骤和可能的错误信息，帮助管理员监控 kubelet 的活动和诊断潜在问题。

到这里关注两个点：

- 第一：createAndInitKubelet函数做了什么
- 第二：跟进startKubelet的逻辑

```go
func RunKubelet(kudeServer *options.KubeletServer, kubeDeps *kubelet.Dependencies, runOnce bool) error {
    // 尝试获取主机名，如果指定了 HostnameOverride 参数则使用该值
    hostname, err := nodeutil.GetHostname(kubeServer.HostnameOverride)
    if err != nil {
        return err
    }

    // 从云提供商获取节点名，如果没有云环境则使用主机名
    nodeName, err := getNodeName(kubeDeps.Cloud, hostname)
    if err != nil {
        return err
    }

    // 检查是否有主机名覆盖
    hostnameOverridden := len(kubeServer.HostnameOverride) > 0

    // 设置事件记录器，用于记录关键系统事件到 Kubernetes API
    makeEventRecorder(context.TODO(), kubeDeps, nodeName)

    // 解析和验证节点 IP 地址
    nodeIPs, err := nodeutil.ParseNodeIPArgument(kubeServer.NodeIP, kubeServer.CloudProvider)
    if err != nil {
        return fmt.Errorf("bad --node-ip %q: %v", kubeServer.NodeIP, err)
    }

    // 初始化 kubelet 权限设置，允许运行特权容器
    capabilities.Initialize(capabilities.Capabilities{
        AllowPrivileged: true,
    })

    // 设置 Docker 配置文件的首选路径
    credentialprovider.SetPreferredDockercfgPath(kubeServer.RootDirectory)
    klog.V(2).InfoS("Using root directory", "path", kubeServer.RootDirectory)

    // 确保操作系统接口实例化，用于与宿主机系统交互
    if kubeDeps.OSInterface == nil {
        kubeDeps.OSInterface = kubecontainer.RealOS{}
    }

    // 创建并初始化 kubelet 实例
    k, err := createAndInitKubelet(kubeServer,
        kubeDeps,
        hostname,
        hostnameOverridden,
        nodeName,
        nodeIPs)
    if err != nil {
        return fmt.Errorf("failed to create kubelet: %w", err)
    }

    // 检查 Pod 配置源是否存在，这是 kubelet 获取 Pod 信息的方式
    if kubeDeps.PodConfig == nil {
        return fmt.Errorf("failed to create kubelet, pod source config was nil")
    }
    podCfg := kubeDeps.PodConfig

    // 设置系统资源限制，如最大打开文件数
    if err := rlimit.SetNumFiles(uint64(kubeServer.MaxOpenFiles)); err != nil {
        klog.ErrorS(err, "Failed to set rlimit on max file handles")
    }

    // 根据运行模式处理 Pod 更新并退出或持续运行
    if runOnce {
        if _, err := k.RunOnce(podCfg.Updates()); err != nil {
            return fmt.Errorf("runonce failed: %w", err)
        }
        klog.InfoS("Started kubelet as runonce")
    } else {
        startKubelet(k, podCfg, &kubeServer.KubeletConfiguration, kubeDeps, kubeServer.EnableServer)
        klog.InfoS("Started kubelet")
    }
    return nil
}
```

## 5.1. CreateAndInitKubelet 函数

主要逻辑：

（1）根据各种配置生成 NewMainKubelet，并且初始化各种manager，例如livenessManager,statusManager,podManager等等

（2）BirthCry ,往apiserver发送一个启动 kubelet的事件

```
// BirthCry sends an event that the kubelet has started up.
func (kl *Kubelet) BirthCry() {
	// Make an event that kubelet restarted.
	kl.recorder.Eventf(kl.nodeRef, v1.EventTypeNormal, events.StartingKubelet, "Starting kubelet.")
}
```

（3）启动垃圾回收，具体就是后台启动多个协程，进行container, image的垃圾回收。

```go
func createAndInitKubelet(kubeServer *options.KubeletServer,
	kubeDeps *kubelet.Dependencies,
	hostname string,
	hostnameOverridden bool,
	nodeName types.NodeName,
	nodeIPs []net.IP) (k kubelet.Bootstrap, err error) {
	// TODO: block until all sources have delivered at least one update to the channel, or break the sync loop
	// up into "per source" synchronizations

	k, err = kubelet.NewMainKubelet(&kubeServer.KubeletConfiguration,
		kubeDeps,
		&kubeServer.ContainerRuntimeOptions,
		hostname,
		hostnameOverridden,
		nodeName,
		nodeIPs,
		kubeServer.ProviderID,
		kubeServer.CloudProvider,
		kubeServer.CertDirectory,
		kubeServer.RootDirectory,
		kubeServer.PodLogsDir,
		kubeServer.ImageCredentialProviderConfigFile,
		kubeServer.ImageCredentialProviderBinDir,
		kubeServer.RegisterNode,
		kubeServer.RegisterWithTaints,
		kubeServer.AllowedUnsafeSysctls,
		kubeServer.ExperimentalMounterPath,
		kubeServer.KernelMemcgNotification,
		kubeServer.ExperimentalNodeAllocatableIgnoreEvictionThreshold,
		kubeServer.MinimumGCAge,
		kubeServer.MaxPerPodContainerCount,
		kubeServer.MaxContainerCount,
		kubeServer.RegisterSchedulable,
		kubeServer.KeepTerminatedPodVolumes,
		kubeServer.NodeLabels,
		kubeServer.NodeStatusMaxImages,
		kubeServer.KubeletFlags.SeccompDefault || kubeServer.KubeletConfiguration.SeccompDefault)
	if err != nil {
		return nil, err
	}

	k.BirthCry()

	k.StartGarbageCollection()

	return k, nil
}
```

## 5.2 NewMainKubelet

**NewMainKubelet 实例化一个 kubelet 对象，并对 kubelet 内部各个 component 进行初始化工作:**

- containerGC // 容器的垃圾回收
- statusManager // pod 状态的管理
- imageManager // 镜像的管路
- probeManager // 容器健康检测
- gpuManager // GPU 的支持
- PodCache // Pod 缓存的管理
- secretManager // secret 资源的管理
- configMapManager // configMap 资源的管理
- InitNetworkPlugin // 网络插件的初始化
- PodManager // 对 pod 的管理, e.g., CRUD
- makePodSourceConfig // pod 元数据的来源 (FILE, URL, api-server)
- diskSpaceManager // 磁盘空间的管理
- ContainerRuntime // 容器运行时的选择(docker 或 rkt)

## 5.3 startKubelet

CreateAndInitKubele之后，马上就是startKubelet了，这里核心调用的了 kubelet.Run函数

主要逻辑如下：

1. 检查 logserver 以及 apiserver 是否可用

2. 如有 cloud provider 配置， 则启动`cloudResourceSyncManager`，将请求发送给 cloud provider

3. 启动 volumeManager，VolumeManager运行一组异步循环，这些循环根据在此节点上调度的Pod来确定需要附加/装入/卸载/分离的卷。

4. 调用 `kubelet.syncNodeStatus`同步 node 状态，如果从上次同步起有任何更改 或 经过了足够的时间，它将节点状态同步到主节点，并在必要时先注册kubelet。

5. 调用 `kubelet.updateRuntimeUp`，updateRuntimeUp调用容器运行时状态回调，在容器运行时首次出现时初始化依赖于运行时的模块，如果状态检查失败，则返回错误。 如果状态检查确定，则在kubelet runtimeState中更新容器运行时正常运行时间。

6. 开启循环，同步 iptables 规则（*但是在源码中无任何操作*）

7. 启动一个用于“杀死pod” 的 goroutine，如果尚未使用其他goroutine，则podKiller会从通道(`podKillingCh`)中接收到一个pod，然后启动goroutine杀死他

8. 启动 statusManager 和 probeManager （都是无限循环的同步机制），statusManager 与 apiserver 同步pod状态；probeManager 管理并接收 container 探针。

9. 启动 runtimeClass manager，默认是docker

    > runtimeClass 是 K8s 的一个 api 对象，可以通过定义 runtimeClass 实现 K8s 对接不同的 容器运行时。

10. 启动 pleg （pod lifecycle event generator），用于生成 pod 相关的 event。

至此，kubelet 整体的启动流程完毕，进入无限循环中，实时同步不同组件的状态。同时也对端口进行监听，响应 http 请求。

```
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *kubelet.Dependencies, enableServer bool) {
	// start the kubelet
	go wait.Until(func() {
		k.Run(podCfg.Updates())
	}, 0, wait.NeverStop)

	// start the kubelet server
	if enableServer {
		go k.ListenAndServe(net.ParseIP(kubeCfg.Address), uint(kubeCfg.Port), kubeDeps.TLSOptions, kubeDeps.Auth, kubeCfg.EnableDebuggingHandlers, kubeCfg.EnableContentionProfiling)

	}
	if kubeCfg.ReadOnlyPort > 0 {
		go k.ListenAndServeReadOnly(net.ParseIP(kubeCfg.Address), uint(kubeCfg.ReadOnlyPort))
	}
}
```



```
// Run starts the kubelet reacting to config updates
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	if kl.logServer == nil {
		kl.logServer = http.StripPrefix("/logs/", http.FileServer(http.Dir("/var/log/")))
	}
	if kl.kubeClient == nil {
		klog.Warning("No api server defined - no node status update will be sent.")
	}

	// Start the cloud provider sync manager
	if kl.cloudResourceSyncManager != nil {
		go kl.cloudResourceSyncManager.Run(wait.NeverStop)
	}

	if err := kl.initializeModules(); err != nil {
		kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.KubeletSetupFailed, err.Error())
		klog.Fatal(err)
	}

	// Start volume manager
	go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)

	if kl.kubeClient != nil {
		// Start syncing node status immediately, this may set up things the runtime needs to run.
		go wait.Until(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, wait.NeverStop)
		go kl.fastStatusUpdateOnce()

		// start syncing lease
		go kl.nodeLeaseController.Run(wait.NeverStop)
	}
	go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)

	// Set up iptables util rules
	if kl.makeIPTablesUtilChains {
		kl.initNetworkUtil()
	}

	// Start a goroutine responsible for killing pods (that are not properly
	// handled by pod workers).
	go wait.Until(kl.podKiller, 1*time.Second, wait.NeverStop)

	// Start component sync loops.
	kl.statusManager.Start()
	kl.probeManager.Start()

	// Start syncing RuntimeClasses if enabled.
	if kl.runtimeClassManager != nil {
		kl.runtimeClassManager.Start(wait.NeverStop)
	}

	// Start the pod lifecycle event generator.
	kl.pleg.Start()
	kl.syncLoop(updates, kl)
}
```



# 6. 总结

1. kubelet采用[Cobra](https://github.com/spf13/cobra)命令行框架和[pflag](https://github.com/spf13/pflag)参数解析框架，和apiserver、scheduler、controller-manager形成统一的代码风格。
2. `kubernetes/cmd/kubelet`部分主要对运行参数进行定义和解析，初始化和构造相关的依赖组件（主要在`kubeDeps`结构体中），并没有kubelet运行的详细逻辑，该部分位于`kubernetes/pkg/kubelet`模块。
3. cmd部分调用流程如下：`Main-->NewKubeletCommand-->Run(kubeletServer, kubeletDeps, stopCh)-->run(s *options.KubeletServer, kubeDeps ..., stopCh ...)--> RunKubelet(s, kubeDeps, s.RunOnce)-->startKubelet-->k.Run(podCfg.Updates())-->pkg/kubelet`。同时`RunKubelet(s, kubeDeps, s.RunOnce)-->CreateAndInitKubelet-->kubelet.NewMainKubelet-->pkg/kubelet`。
4. 整体而言，到目前为止都是进行初始化工作。初始化kubelete各种控制器，然后运行kl.syncLoop(updates, kl)处理Pod

另一种描述：

1. 初始化模块，其实就是运行`imageManager`、`serverCertificateManager`、`oomWatcher`、`resourceAnalyzer`。
2. 运行各种manager，大部分以常驻goroutine的方式运行，其中包括`volumeManager`、`statusManager`等。
3. 执行处理变更的循环函数`syncLoop`，对pod的生命周期进行管理。
