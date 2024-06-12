### 1.背景

从上文中知道，kubelet 监听了apiserver, 静态file,url等pod资源。然后送到 configCh channel中去。

但是在处理configCh channel的时候，确实 **ADD**，**UPDATE**， **REMOVE**等状态变化。这个肯定是经过转换的。

所以本章节就是了解kubelte到底是如何监听处理apiserver等来源的Pod

### 2. makePodSourceConfig

这之前的分析中，在NewMainKubelet函数中有一个重要的步骤, 就是makePodSourceConfig。

```go
if kubeDeps.PodConfig == nil {
		var err error
		kubeDeps.PodConfig, err = makePodSourceConfig(kubeCfg, kubeDeps, nodeName, bootstrapCheckpointPath)
		if err != nil {
			return nil, err
		}
	}
```

makePodSourceConfig的核心逻辑如下：

1. 如果StaticPodPath不为空，调用NewSourceFile监听该目录下定义的Pod的yaml可能是多个yaml,当文件发生变化时，文件源会通过`cfg.Channel`通道发送更新给 Pod 配置系统。
2. 如果配置StaticPodURL不为空，监听这个url获取pod的配置, 配置更新通过`cfg.Channel`通道传送
3. 如果kubeclient是一个有效的客户端，从apiserver监听pod， 通过`cfg.Channel`通道发送从 API 服务器获取的配置更新。

```go
func makePodSourceConfig(kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *Dependencies, nodeName types.NodeName, nodeHasSynced func() bool) (*config.PodConfig, error) {
	//初始化一个 HTTP 头部 http.Header 类型的变量 manifestURLHeader，这将用于后续处理 HTTP 请求。
	manifestURLHeader := make(http.Header)
	//这一段代码检查 kubeCfg.StaticPodURLHeader 是否有内容。如果有，它会遍历所有的键值对，
	//把这些头部信息添加到 manifestURLHeader 中。这对于通过 HTTP 请求加载 Pod 配置非常有用，可以包括认证或其他需要的信息。
	if len(kubeCfg.StaticPodURLHeader) > 0 {
		for k, v := range kubeCfg.StaticPodURLHeader {
			for i := range v {
				manifestURLHeader.Add(k, v[i])
			}
		}
	}

	// source of all configuration
	cfg := config.NewPodConfig(config.PodConfigNotificationIncremental, kubeDeps.Recorder, kubeDeps.PodStartupLatencyTracker)

	// TODO:  it needs to be replaced by a proper context in the future
	ctx := context.TODO()

	//如果StaticPodPath不为空，调用NewSourceFile监听该目录下定义的Pod的yaml可能是多个yaml,当文件发生变化时，文件源会通过`cfg.Channel`通道发送更新给 Pod 配置系统。
	if kubeCfg.StaticPodPath != "" {
		klog.InfoS("Adding static pod path", "path", kubeCfg.StaticPodPath)
		config.NewSourceFile(kubeCfg.StaticPodPath, nodeName, kubeCfg.FileCheckFrequency.Duration, cfg.Channel(ctx, kubetypes.FileSource))
	}

	// 如果配置StaticPodURL不为空，监听这个url获取pod的配置, 配置更新通过`cfg.Channel`通道传送
	if kubeCfg.StaticPodURL != "" {
		klog.InfoS("Adding pod URL with HTTP header", "URL", kubeCfg.StaticPodURL, "header", manifestURLHeader)
		config.NewSourceURL(kubeCfg.StaticPodURL, manifestURLHeader, nodeName, kubeCfg.HTTPCheckFrequency.Duration, cfg.Channel(ctx, kubetypes.HTTPSource))
	}
	//如果kubeclient是一个有效的客户端，从apiserver监听pod， 通过`cfg.Channel`通道发送从 API 服务器获取的配置更新
	if kubeDeps.KubeClient != nil {
		klog.InfoS("Adding apiserver pod source")
		config.NewSourceApiserver(kubeDeps.KubeClient, nodeName, nodeHasSynced, cfg.Channel(ctx, kubetypes.ApiserverSource))
	}
	return cfg, nil
}
```

### 3. PodConfig 结构体介绍

```go
type PodConfig struct {
	pods *podStorage //存储和管理所有 Pod 配置的状态
	mux  *mux        //多路复用和分发从各种配置源接收到的更新

	// the channel of denormalized changes passed to listeners
	updates chan kubetypes.PodUpdate //用于传递给监听者归一化的 Pod 更新。这个通道允许异步传输和处理配置更新

	// contains the list of all configured sources
	sourcesLock sync.Mutex
	sources     sets.String  // 记录所有配置的来源，比如api，url，静态文件
}

// NewPodConfig creates an object that can merge many configuration sources into a stream
// of normalized updates to a pod configuration.
func NewPodConfig(mode PodConfigNotificationMode, recorder record.EventRecorder, startupSLIObserver podStartupSLIObserver) *PodConfig {
	// 创建一个具有 50 个缓冲空间的通道，用于存放 Pod 更新。缓冲区的大小允许在处理时存储多个更新，避免阻塞
	updates := make(chan kubetypes.PodUpdate, 50)
	//创建一个新的 Pod 存储实例，该实例将处理接收到的更新，并根据指定的模式和监视器对它们进行管理
	storage := newPodStorage(updates, mode, recorder, startupSLIObserver)
	//初始化 PodConfig 结构体实例，分配存储和多路复用器，并设置更新通道和源集。
	podConfig := &PodConfig{
		pods:    storage,
		mux:     newMux(storage),
		updates: updates,
		sources: sets.String{},
	}
	//	//返回 podConfig 实例
	return podConfig
}
```

这个 `podStorage` 结构体作为 Kubelet 配置系统的核心部分，负责管理 Pod 的配置状态和处理来自不同配置源的更新。通过将更新通过通道发送，它可以确保配置更新是异步和解耦的，增加系统的响应性和灵活性。

```go
type podStorage struct {
	podLock sync.RWMutex
	// map of source name to pod uid to pod reference
	pods map[string]map[types.UID]*v1.Pod
	mode PodConfigNotificationMode

	// ensures that updates are delivered in strict order
	// on the updates channel
	updateLock sync.Mutex
	updates    chan<- kubetypes.PodUpdate

	// contains the set of all sources that have sent at least one SET
	sourcesSeenLock sync.RWMutex
	sourcesSeen     sets.String  // 用于标记某个来源的 Pod 配置是否已被处理过

	// the EventRecorder to use
	recorder record.EventRecorder

	startupSLIObserver podStartupSLIObserver
}



// TODO: PodConfigNotificationMode could be handled by a listener to the updates channel
// in the future, especially with multiple listeners.
// TODO: allow initialization of the current state of the store with snapshotted version.
func newPodStorage(updates chan<- kubetypes.PodUpdate, mode PodConfigNotificationMode, recorder record.EventRecorder, startupSLIObserver podStartupSLIObserver) *podStorage {
	return &podStorage{
		pods:               make(map[string]map[types.UID]*v1.Pod),
		mode:               mode,
		updates:            updates,
		sourcesSeen:        sets.String{},
		recorder:           recorder,
		startupSLIObserver: startupSLIObserver,
	}
}
```

`小小总结`：

* PodConfig

`PodConfig` 作为多路复用体（Mux），其主要职责是将来自不同源的 Pod 配置更新统一管理，并按一定的顺序传递给监听者。这包括来自文件、URL 或直接从 Kubernetes API 服务器的更新。`PodConfig` 内部使用 `podStorage` 来存储和管理这些更新。`podStorage` 包含一个按命名空间组织的 Pod 映射，以及一个 `updates` 通道，用于接收和处理 Pod 更新。

* Updates 通道

`PodConfig` 和 `podStorage` 都依赖于 `updates` 通道来接收从不同数据源发来的更新。这个通道的作用是确保 Pod 更新能够被集中处理，并按照一定的逻辑顺序（例如优先级）分发给系统的其他部分。

* Source Update Functions（如 newSourceApiserverFromLW）

`newSourceApiserverFromLW` 函数是用来监听来自 Kubernetes API 服务器的 Pod 更新事件的一个具体实现。类似的还有处理静态文件和 URL 更新的函数。这些函数的共同点是它们都将更新事件封装成 `kubetypes.PodUpdate` 类型，然后通过相应的 `updates` 通道发送。

* Merge 操作

在 `podStorage` 中，`Merge` 方法是处理接收到的每个来源的 Pod 更新的关键。无论更新是从 API 服务器、文件系统还是其他 HTTP URL 来的，`Merge` 方法都会根据更新的类型（如增加、删除或修改 Pod）来更新内部存储的状态。这确保了 `PodConfig` 内部状态的一致性和最新性。

* 关于newSourceApiserverFromLW和处理静态文件和 URL 更新的函数，与makePodSourceConfig函数的关系
    * newSourceApiserverFromLW和处理静态文件和 URL 更新的函数只是将这三种发送到updates这个通道，makePodSourceConfig初始化PodConfig实列，而是负责设置和整合多个数据源（就是那三种），以确保 `PodConfig` 能够接收和处理来自这些不同来源的更新。

```go
// newSourceApiserverFromLW holds creates a config source that watches and pulls from the apiserver.
func newSourceApiserverFromLW(lw cache.ListerWatcher, updates chan<- interface{}) {
	send := func(objs []interface{}) {
		var pods []*v1.Pod
		for _, o := range objs {
			pods = append(pods, o.(*v1.Pod))
		}
		updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.ApiserverSource}
	}
	r := cache.NewReflector(lw, &v1.Pod{}, cache.NewUndeltaStore(send, cache.MetaNamespaceKeyFunc), 0)
	go r.Run(wait.NeverStop)
}
```

### 4.Merge

而podStorage.Merge对每个来源的数据进行了统一的处理。进行合并。

Merge函数的核心就是调用 s.merge 整理处理 add/updates/del/remove等等的pods。

```go
// Merge normalizes a set of incoming changes from different sources into a map of all Pods
// and ensures that redundant changes are filtered out, and then pushes zero or more minimal
// updates onto the update channel.  Ensures that updates are delivered in order.
func (s *podStorage) Merge(source string, change interface{}) error {\
	//使用互斥锁 updateLock 来保证在更新存储的过程中数据的一致性和线程安全。defer 关键字确保方法结束时释放锁，无论是通过正常返回还是由于错误提前退出。
	s.updateLock.Lock()
	defer s.updateLock.Unlock()
	//检查当前源（source）是否已经被此存储处理过。这个信息存储在 sourcesSeen 集合中，用于追踪哪些数据源的更新已经被处理。
	seenBefore := s.sourcesSeen.Has(source)
	//调用 merge 方法处理接收到的更新，该方法根据更新的类型（新增、更新、删除等）分别归类处理，并返回各类更新的集合。
	adds, updates, deletes, removes, reconciles := s.merge(source, change)
	//检查这是否是第一次从该源接收到更新。如果之前未见过，且现在标记为已见，那么这是第一次处理该源的数据。
	firstSet := !seenBefore && s.sourcesSeen.Has(source)

	// deliver update notifications
	//根据 podStorage 的通知模式（s.mode），决定如何发布更新通知。PodConfigNotificationMode 定义了几种不同的通知策略。
	switch s.mode {
	//如果模式设置为增量更新，则对每种类型的更新分别处理。
	case PodConfigNotificationIncremental:
		//如果有 Pods 被移除，将这些更新发送到 updates 通道。
		if len(removes.Pods) > 0 {
			s.updates <- *removes
		}
		//对于新增、更新、删除的 Pods，如果有相关的数据，同样将这些更新发送到 updates 通道。
		if len(adds.Pods) > 0 {
			s.updates <- *adds
		}
		if len(updates.Pods) > 0 {
			s.updates <- *updates
		}
		if len(deletes.Pods) > 0 {
			s.updates <- *deletes
		}
		//如果是第一次从该源接收更新，并且没有任何新增、更新或删除的 Pods，发送一个空的更新通知。这表示数据源已准备好，但没有数据变动。
		if firstSet && len(adds.Pods) == 0 && len(updates.Pods) == 0 && len(deletes.Pods) == 0 {
			// Send an empty update when first seeing the source and there are
			// no ADD or UPDATE or DELETE pods from the source. This signals kubelet that
			// the source is ready.
			s.updates <- *adds
		}
		// Only add reconcile support here, because kubelet doesn't support Snapshot update now.
		//处理调和类型的更新（通常用于状态同步或错误修正）
		if len(reconciles.Pods) > 0 {
			s.updates <- *reconciles
		}
	//这个模式结合了快照和增量更新。在这种模式下，根据变化类型选择性地发送完整的快照或者只发送变更。
	case PodConfigNotificationSnapshotAndUpdates:
		//如果有 Pods 被添加或移除，或者是首次从这个源接收更新，那么发送一个包含所有 Pods 当前状态的快照。这个快照是通过 s.mergedState() 方法生成的，它返回当前所有 Pods 的状态。
		if len(removes.Pods) > 0 || len(adds.Pods) > 0 || firstSet {
			s.updates <- kubetypes.PodUpdate{Pods: s.mergedState().([]*v1.Pod), Op: kubetypes.SET, Source: source}
		}
		//如果有 Pods 被更新或删除，这些具体的更新会被单独发送。这样可以确保接收者明确知道哪些 Pods 有变化，而不需要处理整个集群的快照。
		if len(updates.Pods) > 0 {
			s.updates <- *updates
		}
		if len(deletes.Pods) > 0 {
			s.updates <- *deletes
		}
	//这个模式仅发送完整的快照，不区分变化类型。
	case PodConfigNotificationSnapshot:
		//无论是添加、更新、删除还是移除 Pods，或是首次从这个源接收数据，都会发送一个包含所有当前 Pods 状态的快照。这种模式适用于需要完整视图的情况，比如系统初始化或重大状态变更后的同步。
		if len(updates.Pods) > 0 || len(deletes.Pods) > 0 || len(adds.Pods) > 0 || len(removes.Pods) > 0 || firstSet {
			s.updates <- kubetypes.PodUpdate{Pods: s.mergedState().([]*v1.Pod), Op: kubetypes.SET, Source: source}
		}
		//这个部分处理未知的通知模式或者未被明确指定的模式。
	case PodConfigNotificationUnknown:
		//如果 s.mode 匹配到未知模式或任何未显式处理的模式，程序将触发一个 panic，表明发生了一个不应该出现的配置或编程错误。
		fallthrough
	default:
		panic(fmt.Sprintf("unsupported PodConfigNotificationMode: %#v", s.mode))
	}

	return nil
}
```

### 5. s.merge

```go

func (s *podStorage) merge(source string, change interface{}) (adds, updates, deletes, removes, reconciles *kubetypes.PodUpdate) {
	s.podLock.Lock()
	defer s.podLock.Unlock()
	//这五行初始化五个Pod切片，用于存储将要进行的不同类型的Pod操作。
	addPods := []*v1.Pod{}
	updatePods := []*v1.Pod{}
	deletePods := []*v1.Pod{}
	removePods := []*v1.Pod{}
	reconcilePods := []*v1.Pod{}
	//这里检查特定来源的Pod缓存是否存在。如果不存在，就初始化一个新的映射。
	pods := s.pods[source]
	if pods == nil {
		pods = make(map[types.UID]*v1.Pod)
	}

	// updatePodFunc is the local function which updates the pod cache *oldPods* with new pods *newPods*.
	// After updated, new pod will be stored in the pod cache *pods*.
	// Notice that *pods* and *oldPods* could be the same cache.
	//这行定义了一个匿名函数 updatePodsFunc，它接收三个参数：newPods 是一个Pod数组，包含最新的Pod信息；
	//oldPods 是一个从UID到Pod的映射，代表旧的Pod状态；pods 是一个映射，可能是更新过的 oldPods 或是全新的映射，用于存储当前的Pod状态。
	updatePodsFunc := func(newPods []*v1.Pod, oldPods, pods map[types.UID]*v1.Pod) {
		//这行调用 filterInvalidPods 函数，传入新的Pods、来源和记录器，用于筛选出有效的Pods。无效的Pods可能由于数据问题或与特定来源不匹配而被过滤掉。
		filtered := filterInvalidPods(newPods, source, s.recorder)
		//这行开始遍历 filtered 中的每个Pod (ref)，进行进一步的处理。
		for _, ref := range filtered {
			// Annotate the pod with the source before any comparison.
			//这两行首先检查Pod的注解是否初始化，如果没有，则初始化它。之后，将Pod的来源信息添加到注解中，用于标识Pod的配置来源
			//通过pod Annotations表明pod的来源（apiserver/url/file）
			if ref.Annotations == nil {
				ref.Annotations = make(map[string]string)
			}
			ref.Annotations[kubetypes.ConfigSourceAnnotationKey] = source
			// ignore static pods
			//这两行检查如果Pod不是静态Pod，则记录该Pod在观察列表中的观察时间。静态Pods通常由节点上的配置文件直接管理，而非通过API服务器。
			if !kubetypes.IsStaticPod(ref) {
				s.startupSLIObserver.ObservedPodOnWatch(ref, time.Now())
			}
			//这行代码检查基于UID的Pod (ref) 是否在 oldPods 映射中存在。如果存在，existing 变量将引用找到的Pod，而 found 将为 true。
			if existing, found := oldPods[ref.UID]; found {
				//这行代码将 existing（已存在的Pod）赋值给 pods 映射中对应的UID键。这确保了 pods 映射保持最新状态，指向旧的Pod实例
				pods[ref.UID] = existing
				//这行调用 checkAndUpdatePod 函数，传入两个参数：existing（当前缓存中的Pod）和 ref（新接收到的Pod）。这个函数比较这两个Pod的状态，确定是否需要进行更新、协调或优雅删除操作。它返回三个布尔值，分别代表是否需要执行这些操作。
				needUpdate, needReconcile, needGracefulDelete := checkAndUpdatePod(existing, ref)
				//如果 needUpdate 为 true，则将 existing Pod 添加到 updatePods 列表中，表示需要更新这个Pod。
				if needUpdate {
					updatePods = append(updatePods, existing)
				} else if needReconcile {
					//如果 needReconcile 为 true，则将 existing Pod 添加到 reconcilePods 列表中，表示这个Pod需要进行一些状态协调。
					reconcilePods = append(reconcilePods, existing)
				} else if needGracefulDelete {
					//如果 needGracefulDelete 为 true，则将 existing Pod 添加到 deletePods 列表中，表示这个Pod需要被优雅地删除。
					deletePods = append(deletePods, existing)
				}
				continue
			}
			//如果在旧的Pods映射中没有找到相应的UID，这意味着这是一个全新的Pod。这行代码记录Pod首次出现的时间，将其加入到当前的Pods映射中，并添加到添加Pods列表中。
			recordFirstSeenTime(ref)
			pods[ref.UID] = ref
			addPods = append(addPods, ref)
		}
	}
	//将 change 转换为 kubetypes.PodUpdate 类型，这是类型断言操作，如果 change 不是这个类型，将导致 panic
	update := change.(kubetypes.PodUpdate)
	//update.Op 表示更新操作的类型
	switch update.Op {
		//	这段代码处理三个操作：添加（ADD）、更新（UPDATE）和删除（DELETE）。
	case kubetypes.ADD, kubetypes.UPDATE, kubetypes.DELETE:
		if update.Op == kubetypes.ADD {
			klog.V(4).InfoS("Adding new pods from source", "source", source, "pods", klog.KObjSlice(update.Pods))
		} else if update.Op == kubetypes.DELETE {
			klog.V(4).InfoS("Gracefully deleting pods from source", "source", source, "pods", klog.KObjSlice(update.Pods))
		} else {
			klog.V(4).InfoS("Updating pods from source", "source", source, "pods", klog.KObjSlice(update.Pods))
		}
		//调用 updatePodsFunc 函数，传入 update.Pods（新Pods列表）、pods（旧Pods缓存），以进行添加、更新或删除操作
		updatePodsFunc(update.Pods, pods, pods)
	//处理 REMOVE 操作，即从缓存中移除 Pods，并记录日志信息。
	case kubetypes.REMOVE:
		klog.V(4).InfoS("Removing pods from source", "source", source, "pods", klog.KObjSlice(update.Pods))
		//遍历 update.Pods 中的每个 Pod，如果在 pods 中找到对应的 UID，则从 pods 中删除该Pod，并将其添加到 removePods 列表中。如果没有找到对应的 UID，则无需操作。
		for _, value := range update.Pods {
			if existing, found := pods[value.UID]; found {
				// this is a delete
				delete(pods, value.UID)
				removePods = append(removePods, existing)
				continue
			}
			// this is a no-op
		}
	//处理 SET 操作，即完全重置 pods 映射，并记录日志信息。
	case kubetypes.SET:
		klog.V(4).InfoS("Setting pods for source", "source", source)
		//调用 markSourceSet 方法，标记该来源的Pod集合已被设置
		s.markSourceSet(source)
		// Clear the old map entries by just creating a new map
		//保存旧的 pods 映射到 oldPods，然后创建一个新的 pods 映射，用于存储更新后的Pods。
		oldPods := pods
		pods = make(map[types.UID]*v1.Pod)
		//保存旧的 pods 映射到 oldPods，然后创建一个新的 pods 映射，用于存储更新后的Pods。
		updatePodsFunc(update.Pods, oldPods, pods)
		//遍历 oldPods，将那些没有出现在新的 pods 映射中的 UID 对应的Pods添加到 removePods 列表中。
		for uid, existing := range oldPods {
			if _, found := pods[uid]; !found {
				// this is a delete
				removePods = append(removePods, existing)
			}
		}
	//对于任何不匹配上述类型的操作，记录收到无效更新类型的日志信息。
	default:
		klog.InfoS("Received invalid update type", "type", update)
	//
	}

	s.pods[source] = pods
	//这些行将计算出的添加、更新、删除、移除和协调操作的Pod列表分别赋值给返回值。这些返回的 PodUpdate 结构体将被用于外部处理，例如发送到其他组件或记录日志。
	adds = &kubetypes.PodUpdate{Op: kubetypes.ADD, Pods: copyPods(addPods), Source: source}
	updates = &kubetypes.PodUpdate{Op: kubetypes.UPDATE, Pods: copyPods(updatePods), Source: source}
	deletes = &kubetypes.PodUpdate{Op: kubetypes.DELETE, Pods: copyPods(deletePods), Source: source}
	removes = &kubetypes.PodUpdate{Op: kubetypes.REMOVE, Pods: copyPods(removePods), Source: source}
	reconciles = &kubetypes.PodUpdate{Op: kubetypes.RECONCILE, Pods: copyPods(reconcilePods), Source: source}

	return adds, updates, deletes, removes, reconciles
}
```

#### 5.1 case

* 通道断言`kubetypes.PodUpdate`，如果是ADD，update，delete就调用updatePodsFunc把pod加入add，updatePods，delete，reconcilePods的切片中

* 如果是`kubetypes.remove`，就便利for通道获取的所有update.pods，用update.pods的UID和pods的UID进行对比（pods就是podstorge里面sourcesSeen的集合，代表那些pod处理过了，`也可以理解为旧的pod`），找到就把它从pods里面删除，再把这个pod加入要删除的removePods的集合中，返回给Merge

* 这里重点关注`kubetypes.SET`,因为**API 服务器**、**静态文件** 和 **URL** 都代表整个 Pod 列表的完整状态。每次获取新的配置时，假设来源提供的 Pod 列表是该来源的最新完整状态。因此，任何变更都会被视为该来源的新的完整配置，而不是部分更新。markSourceSet（）方法把这个source加入代表已处理，调用updatePodsFunc更新处理全部pod，然后便利oldPods，updatePodsFunc会过滤然后更新缓存中的pod最新状态存在pods里面，然后oldpods中不存在与pods的里面话，就加入remove切片中

* 切片格式

    ```go
    addPods := []*v1.Pod{}
    updatePods := []*v1.Pod{}
    deletePods := []*v1.Pod{}
    removePods := []*v1.Pod{}
    reconcilePods := []*v1.Pod{}	
    
    adds = &kubetypes.PodUpdate{Op: kubetypes.ADD, Pods: copyPods(addPods), Source: source}
    updates = &kubetypes.PodUpdate{Op: kubetypes.UPDATE, Pods: copyPods(updatePods), Source: source}
    deletes = &kubetypes.PodUpdate{Op: kubetypes.DELETE, Pods: copyPods(deletePods), Source: source}
    removes = &kubetypes.PodUpdate{Op: kubetypes.REMOVE, Pods: copyPods(removePods), Source: source}
    reconciles = &kubetypes.PodUpdate{Op: kubetypes.RECONCILE, Pods: copyPods(reconcilePods), Source: source}
    ```

```go
	//将 change 转换为 kubetypes.PodUpdate 类型，这是类型断言操作，如果 change 不是这个类型，将导致 panic
	update := change.(kubetypes.PodUpdate)
	//update.Op 表示更新操作的类型
	switch update.Op {
		//	这段代码处理三个操作：添加（ADD）、更新（UPDATE）和删除（DELETE）。
	case kubetypes.ADD, kubetypes.UPDATE, kubetypes.DELETE:
		if update.Op == kubetypes.ADD {
			klog.V(4).InfoS("Adding new pods from source", "source", source, "pods", klog.KObjSlice(update.Pods))
		} else if update.Op == kubetypes.DELETE {
			klog.V(4).InfoS("Gracefully deleting pods from source", "source", source, "pods", klog.KObjSlice(update.Pods))
		} else {
			klog.V(4).InfoS("Updating pods from source", "source", source, "pods", klog.KObjSlice(update.Pods))
		}
		//调用 updatePodsFunc 函数，传入 update.Pods（新Pods列表）、pods（旧Pods缓存），以进行添加、更新或删除操作
		updatePodsFunc(update.Pods, pods, pods)
	//处理 REMOVE 操作，即从缓存中移除 Pods，并记录日志信息。
	case kubetypes.REMOVE:
		klog.V(4).InfoS("Removing pods from source", "source", source, "pods", klog.KObjSlice(update.Pods))
		//遍历 update.Pods 中的每个 Pod，如果在 pods 中找到对应的 UID，则从 pods 中删除该Pod，并将其添加到 removePods 列表中。如果没有找到对应的 UID，则无需操作。
		for _, value := range update.Pods {
			if existing, found := pods[value.UID]; found {
				// this is a delete
				delete(pods, value.UID)
				removePods = append(removePods, existing)
				continue
			}
			// this is a no-op
		}
	//处理 SET 操作，即完全重置 pods 映射，并记录日志信息。
	case kubetypes.SET:
		klog.V(4).InfoS("Setting pods for source", "source", source)
		//调用 markSourceSet 方法，标记该来源的Pod集合已被设置
		s.markSourceSet(source)
		// Clear the old map entries by just creating a new map
		//保存旧的 pods 映射到 oldPods，然后创建一个新的 pods 映射，用于存储更新后的Pods。
		oldPods := pods
		pods = make(map[types.UID]*v1.Pod)
		//保存旧的 pods 映射到 oldPods，然后创建一个新的 pods 映射，用于存储更新后的Pods。
		updatePodsFunc(update.Pods, oldPods, pods)
		//遍历 oldPods，将那些没有出现在新的 pods 映射中的 UID 对应的Pods添加到 removePods 列表中。
		for uid, existing := range oldPods {
			if _, found := pods[uid]; !found {
				// this is a delete
				removePods = append(removePods, existing)
			}
		}
	//对于任何不匹配上述类型的操作，记录收到无效更新类型的日志信息。
	default:
		klog.InfoS("Received invalid update type", "type", update)
	//
	}
```

#### 5.2 updatePodsFunc

* 根据NodeName去重
* 通过pod Annotations表明pod的来源（apiserver/url/file）
* 检查是否静态pod，静态pod记录在观察列表
* 检查最新的update.pods是否存在缓存pods，存在的话就把本地pods指向最新的update.pods，就是更新旧的pod实列
* 检查缓存pod和最新pod如果是一个pod，但是状态不是一样，就加入reconcilePods队列
* 如果新pod有DeletionTimestamp，那就是需要needGracefulDelete, 加入deletePods
* 否则就是update
* 最后检查如果在旧的Pods映射中没有找到相应的UID，这意味着这是一个全新的Pod。这行代码记录Pod首次出现的时间，将其加入到当前的Pods映射中，并添加到添加Pods列表中

```go
	updatePodsFunc := func(newPods []*v1.Pod, oldPods, pods map[types.UID]*v1.Pod) {
		//根据updates.pods的NodeName字段和source就是已经处理过的Pod，过滤掉已经处理过的Pod
		filtered := filterInvalidPods(newPods, source, s.recorder)
		//这行开始遍历 filtered 中的每个Pod (名称)，进行进一步的处理。
		for _, ref := range filtered {
			// Annotate the pod with the source before any comparison.
			//这两行首先检查Pod的注解是否初始化，如果没有，则初始化它。之后，将Pod的来源信息添加到注解中，用于标识Pod的配置来源
			//通过pod Annotations表明pod的来源（apiserver/url/file）
			if ref.Annotations == nil {
				ref.Annotations = make(map[string]string)
			}
			ref.Annotations[kubetypes.ConfigSourceAnnotationKey] = source
			// ignore static pods
			//这两行检查如果Pod不是静态Pod，则记录该Pod在观察列表中的观察时间。静态Pods通常由节点上的配置文件直接管理，而非通过API服务器。
			if !kubetypes.IsStaticPod(ref) {
				s.startupSLIObserver.ObservedPodOnWatch(ref, time.Now())
			}
			//检查过滤掉的pod是否存在于oldPods中，如果存在，则将缓存中的pod指向最新旧的pod实列
			if existing, found := oldPods[ref.UID]; found {
				//将缓存中的pod指向最新的pod实列
				pods[ref.UID] = existing
				//这行调用 checkAndUpdatePod 函数，传入两个参数：existing（当前缓存中的Pod）和 ref（新接收到的Pod）。这个函数比较这两个Pod的状态，确定是否需要进行更新、协调或优雅删除操作。它返回三个布尔值，分别代表是否需要执行这些操作。
				needUpdate, needReconcile, needGracefulDelete := checkAndUpdatePod(existing, ref)
				//如果 needUpdate 为 true，则将 existing Pod 添加到 updatePods 列表中，表示需要更新这个Pod。
				if needUpdate {
					updatePods = append(updatePods, existing)
				} else if needReconcile {
					//如果 needReconcile 为 true，则将 existing Pod 添加到 reconcilePods 列表中，把pod的期望状态和实际状态一致
					reconcilePods = append(reconcilePods, existing)
				} else if needGracefulDelete {
					//如果 needGracefulDelete 为 true，则将 existing Pod 添加到 deletePods 列表中，表示这个Pod需要被优雅地删除。
					deletePods = append(deletePods, existing)
				}
				continue
			}
			//如果在旧的Pods映射中没有找到相应的UID，这意味着这是一个全新的Pod。这行代码记录Pod首次出现的时间，将其加入到当前的Pods映射中，并添加到添加Pods列表中。
			recordFirstSeenTime(ref)
			pods[ref.UID] = ref
			addPods = append(addPods, ref)
		}
	}
```

#### 5.3checkAndUpdatePod

参数：existing是缓存的pod, ref是update.pod

逻辑：

（1）如果本地pod和新pod处理除了状态外，其他都一样，那就是Reconcile（就是让pod期望状态和实际状态一致）

（2）如果新pod有DeletionTimestamp，那就是需要needGracefulDelete

（3）否则就是update

```go
func checkAndUpdatePod(existing, ref *v1.Pod) (needUpdate, needReconcile, needGracefulDelete bool) {

	// 1. this is a reconcile
	// TODO: it would be better to update the whole object and only preserve certain things
	//       like the source annotation or the UID (to ensure safety)
	//检查现有 Pod 和update.Pod 是否在语义上有差异，即判断两个 Pod 是否相同
	if !podsDifferSemantically(existing, ref) {
		// this is not an update
		// Only check reconcile when it is not an update, because if the pod is going to
		// be updated, an extra reconcile is unnecessary
		//如果是同一个pod，但是状态不相同，则将现有 Pod 的状态更新为update.Pod的状态，并设置 needReconcile 为 true，表示需要进行协调操作。
		if !reflect.DeepEqual(existing.Status, ref.Status) {
			// Pod with changed pod status needs reconcile, because kubelet should
			// be the source of truth of pod status.
			existing.Status = ref.Status
			needReconcile = true
		}
		return
	}

	// Overwrite the first-seen time with the existing one. This is our own
	// internal annotation, there is no need to update.
	//将update.Pod 的第一次出现时间更新为现有 Pod 的第一次出现时间，保持一致性，无需更新。
	ref.Annotations[kubetypes.ConfigFirstSeenAnnotationKey] = existing.Annotations[kubetypes.ConfigFirstSeenAnnotationKey]
	//将现有 Pod 的规范、标签、删除时间戳、删除优雅期限和状态更新为update.Pod 的相应字段，并调用 updateAnnotations 函数更新注解信息。
	existing.Spec = ref.Spec
	existing.Labels = ref.Labels
	existing.DeletionTimestamp = ref.DeletionTimestamp
	existing.DeletionGracePeriodSeconds = ref.DeletionGracePeriodSeconds
	existing.Status = ref.Status
	updateAnnotations(existing, ref)

	// 2. this is an graceful delete
	//如果update.Pod 的删除时间戳不为空，则设置 needGracefulDelete 为 true，表示需要进行优雅删除操作；
	//否则，设置 needUpdate 为 true，表示需要进行更新操作。
	if ref.DeletionTimestamp != nil {
		needGracefulDelete = true
	} else {
		// 3. this is an update
		needUpdate = true
	}

	return
}
```

`完整总结`：==kubelet监听pod变化，如何把三种不同来源的事件转换成add，delete，remove，update等==

1. **启动监听**：在 Kubelet 启动时，确实会通过一个循环来监听 `configCh` 通道，这个通道用于接收来自不同数据源的 Pod 更新事件。
2. **事件类型**：监听的事件包括 Pod 的添加、更新和删除等。这些事件是从各种配置源（如 API 服务器、静态文件和 URL）收集而来。

* `数据源处理与更新发送`

1. **数据源初始化**：`newSourceApiserverFromLW` 和处理静态文件及 URL 的相关函数负责初始化这些数据源的监听，并在检测到变化时，将 Pod 更新封装成 `kubetypes.PodUpdate` 类型后发送到 `updates` 通道。
2. **配置数据源处理**：`makePodSourceConfig` 函数在 Kubelet 启动时被调用，用于初始化 `NewPodConfig`。这个配置决定了如何处理来自不同源的数据，即如何从 `updates` 通道读取并响应更新。

* `更新合并与通知模式`

1. **合并逻辑**：`Merge` 方法内部调用 `s.merge`，后者负责实际的合并逻辑，即如何将从数据源接收到的更新分为不同的类别（添加、删除、更新等），然后再加入==add==，==delete==，==update==，==remove==，==reconciles==事件中。
2. **处理更新**：`Merge` 根据 `PodConfig` 实例配置的通知模式（`mode`），将这些更新分类后分别发送。这可以是发送具体的增量更新（增加、删除、更新），或者在某些模式下发送一个包含所有当前状态的快照。

* `通知模式`（还有一个未知模式）

1. **增量通知**：在 `PodConfigNotificationIncremental` 模式下，仅相关的变更（如新增、删除）会被发送。
2. **快照与更新**：在 `PodConfigNotificationSnapshotAndUpdates` 模式下，除了具体变更外，还可能发送完整的 Pod 状态快照，特别是在初次接收数据源时或当发生重大变更时。
3. **完整快照**：`PodConfigNotificationSnapshot` 模式会在任何更新发生时发送整个集群的 Pod 快照，确保接收方的状态完全同步。

* `syncLoop` 处理

1. **直接处理**：最后，所有这些通过 `updates` 通道接收到的更新事件（新增、更新、删除等）都会被 `syncLoop` 直接处理。`syncLoop` 是 Kubelet 的核心事件处理循环，它负责实际应用这些更新到 Kubernetes 集群中，如启动或停止容器、更新容器状态等。