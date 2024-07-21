# 1.evictionManager

* 主要是监控节点资源状态、评估驱逐阈值、执行驱逐决策等

## 2. initializeRuntimeDependentModules

省略kubelet->run->updateRuntimeUp，直接从initializeRuntimeDependentModules开发分析。

initializeRuntimeDependentModules的核心逻辑就是启动evictionManager和其他相关的组件。该函数只在kubelet运行时启动一次。

这里关注的核心函数：

（1）启动cadvisor

（2）启动containerManager

（3）启动evictionManager

因为evictionManager需要的数据是来源于，cadvisor的，所以必须等cadvisor启动完后在启动evictionManager

```go
// initializeRuntimeDependentModules will initialize internal modules that require the container runtime to be up.
func (kl *Kubelet) initializeRuntimeDependentModules() {
  // 1. 启动cadvisor
	if err := kl.cadvisor.Start(); err != nil {
		// Fail kubelet and rely on the babysitter to retry starting kubelet.
		// TODO(random-liu): Add backoff logic in the babysitter
		klog.Fatalf("Failed to start cAdvisor %v", err)
	}

	// trigger on-demand stats collection once so that we have capacity information for ephemeral storage.
	// ignore any errors, since if stats collection is not successful, the container manager will fail to start below.
	kl.StatsProvider.GetCgroupStats("/", true)
	// Start container manager.
	node, err := kl.getNodeAnyWay()
	if err != nil {
		// Fail kubelet and rely on the babysitter to retry starting kubelet.
		klog.Fatalf("Kubelet failed to get node info: %v", err)
	}
	
	// 2.启动containerManager
	// containerManager must start after cAdvisor because it needs filesystem capacity information
	if err := kl.containerManager.Start(node, kl.GetActivePods, kl.sourcesReady, kl.statusManager, kl.runtimeService); err != nil {
		// Fail kubelet and rely on the babysitter to retry starting kubelet.
		klog.Fatalf("Failed to start ContainerManager %v", err)
	}
	
	// 3.启动evictionManager
	// eviction manager must start after cadvisor because it needs to know if the container runtime has a dedicated imagefs
	kl.evictionManager.Start(kl.StatsProvider, kl.GetActivePods, kl.podResourcesAreReclaimed, evictionMonitoringPeriod)

	...
}
```

## 3.evictionManager.Start

核心逻辑如下 

（1）是否利用kernel memcg notification机制。默认是否，可以通过--kernel-memcg-notification参数开启

kubelet 定期通过 cadvisor 接口采集节点内存使用数据，当节点短时间内内存使用率突增，此时 kubelet 无法感知到也不会有 MemoryPressure 相关事件，但依然会调用 OOMKiller 停止容器。可以通过为 kubelet 配置 `--kernel-memcg-notification` 参数启用 memcg api，当触发 memory 使用率阈值时 memcg 会主动进行通知；

memcg 主动通知的功能是 cgroup 中已有的，kubelet 会在 `/sys/fs/cgroup/memory/cgroup.event_control` 文件中写入 memory.available 的阈值，而阈值与 inactive_file 文件的大小有关系，kubelet 也会定期更新阈值，当 memcg 使用率达到配置的阈值后会主动通知 kubelet，kubelet 通过 epoll 机制来接收通知。

这个暂时先了解一下，不做深入。

（2）循环调用synchronize，waitForPodsCleanup来驱逐清理pod。循环间隔是10s，monitoringInterval默认10s

```go
// Start starts the control loop to observe and response to low compute resources.
func (m *managerImpl) Start(diskInfoProvider DiskInfoProvider, podFunc ActivePodsFunc, podCleanedUpFunc PodCleanedUpFunc, monitoringInterval time.Duration) {
	thresholdHandler := func(message string) {
		klog.InfoS(message)
		m.synchronize(diskInfoProvider, podFunc)
	}
	if m.config.KernelMemcgNotification {
		for _, threshold := range m.config.Thresholds {
			if threshold.Signal == evictionapi.SignalMemoryAvailable || threshold.Signal == evictionapi.SignalAllocatableMemoryAvailable {
				notifier, err := NewMemoryThresholdNotifier(threshold, m.config.PodCgroupRoot, &CgroupNotifierFactory{}, thresholdHandler)
				if err != nil {
					klog.InfoS("Eviction manager: failed to create memory threshold notifier", "err", err)
				} else {
					go notifier.Start()
					m.thresholdNotifiers = append(m.thresholdNotifiers, notifier)
				}
			}
		}
	}
	// start the eviction manager monitoring
	go func() {
		for {
			evictedPods, err := m.synchronize(diskInfoProvider, podFunc)
			if evictedPods != nil && err == nil {
				klog.InfoS("Eviction manager: pods evicted, waiting for pod to be cleaned up", "pods", klog.KObjSlice(evictedPods))
				m.waitForPodsCleanup(podCleanedUpFunc, evictedPods)
			} else {
				if err != nil {
					klog.ErrorS(err, "Eviction manager: failed to synchronize")
				}
				time.Sleep(monitoringInterval)
				time.Sleep(monitoringInterval)
			}
		}
	}()
}
```

### 3.1 synchronize

核心逻辑：

（1）得到该节点所有activePods(得到所有Pod然后去掉了status.Phase == v1.PodFailed || status.Phase == v1.PodSucceeded || (pod.DeletionTimestamp != nil && notRunning(status.ContainerStatuses)))

（2）从cadvisor获取详细信息，就是node, pod的资源统计信息-（重要环节）

（3）从统计数据中获得节点资源的使用情况observations

（4）将资源实际使用量和资源容量进行比较，最终得到阈值结构体对象的列表。举例来说就是，我设置了pid, mem, fs三个thresholds,但是通过观察，可能就是mem这一个限制达到了驱逐阈值

（5）再加上最小强制回收值(防止反复驱逐)。算出来最终的哪些限制到阈值了。可以参[最小强制回收](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/#minimum-eviction-reclaim)_

（6）记录每个限制第一次驱逐时间，因为软驱逐会有时间容忍，所以对于软驱逐而言，过来容忍期还是超了阈值，这个时候就要驱逐()（如果启用了本地存储容量隔离，则检查是否需要进行本地存储驱逐）

（7）回收节点级的资源，如果回收的资源足够的话，直接返回，不需要驱逐正在运行中的pod

（8）对不同阈值驱逐场景下pod有不同的排序，比如如果是mem驱逐，就是按照req limit的qos进行排序驱逐

（9）按照排序后的结果每次驱逐一个pod,判断是硬性驱逐还是软性驱逐，annotations包含了为什么被驱逐，启用了 PodDisruptionConditions 功能，则创建一个 Pod 条件对象，标记当前 Pod 为驱逐目标。

```go
// synchronize is the main control loop that enforces eviction thresholds.
// Returns the pod that was killed, or nil if no pod was killed.
func (m *managerImpl) synchronize(diskInfoProvider DiskInfoProvider, podFunc ActivePodsFunc) ([]*v1.Pod, error) {
	ctx := context.Background()
	// if we have nothing to do, just return
	//检查是否有设定的驱逐阈值。如果没有任何阈值，并且未启用本地存储隔离功能，则函数直接返回，不执行任何操作。
	thresholds := m.config.Thresholds
	if len(thresholds) == 0 && !m.localStorageCapacityIsolation {
		return nil, nil
	}

	klog.V(3).InfoS("Eviction manager: synchronize housekeeping")
	// build the ranking functions (if not yet known)
	// TODO: have a function in cadvisor that lets us know if global housekeeping has completed
	if m.dedicatedImageFs == nil {
		hasImageFs, splitDiskError := diskInfoProvider.HasDedicatedImageFs(ctx)
		if splitDiskError != nil {
			klog.ErrorS(splitDiskError, "Eviction manager: failed to get HasDedicatedImageFs")
			return nil, fmt.Errorf("eviction manager: failed to get HasDedicatedImageFs: %v", splitDiskError)
		}
		m.dedicatedImageFs = &hasImageFs
		splitContainerImageFs := m.containerGC.IsContainerFsSeparateFromImageFs(ctx)

		// If we are a split filesystem but the feature is turned off
		// we should return an error.
		// This is a bad state.
		if !utilfeature.DefaultFeatureGate.Enabled(features.KubeletSeparateDiskGC) && splitContainerImageFs {
			splitDiskError := fmt.Errorf("KubeletSeparateDiskGC is turned off but we still have a split filesystem")
			return nil, splitDiskError
		}
		thresholds, err := UpdateContainerFsThresholds(m.config.Thresholds, hasImageFs, splitContainerImageFs)
		m.config.Thresholds = thresholds
		if err != nil {
			klog.ErrorS(err, "eviction manager: found conflicting containerfs eviction. Ignoring.")
		}
		m.splitContainerImageFs = &splitContainerImageFs
		m.signalToRankFunc = buildSignalToRankFunc(hasImageFs, splitContainerImageFs)
		m.signalToNodeReclaimFuncs = buildSignalToNodeReclaimFuncs(m.imageGC, m.containerGC, hasImageFs, splitContainerImageFs)
	}

	klog.V(3).InfoS("FileSystem detection", "DedicatedImageFs", m.dedicatedImageFs, "SplitImageFs", m.splitContainerImageFs)
	// 1. 得到该节点所有activePods(得到所有Pod然后去掉了status.Phase == v1.PodFailed || status.Phase == v1.PodSucceeded || (pod.DeletionTimestamp != nil && notRunning(status.ContainerStatuses)))
	activePods := podFunc()
	updateStats := true
	// 2. 从cadvisor获取详细信息，就是node, pod的资源统计信息-（重要环节）
	summary, err := m.summaryProvider.Get(ctx, updateStats)
	if err != nil {
		klog.ErrorS(err, "Eviction manager: failed to get summary stats")
		return nil, nil
	}
	// 之前内核notify相关，一般不开启，这里忽略
	if m.clock.Since(m.thresholdsLastUpdated) > notifierRefreshInterval {
		m.thresholdsLastUpdated = m.clock.Now()
		for _, notifier := range m.thresholdNotifiers {
			if err := notifier.UpdateThreshold(summary); err != nil {
				klog.InfoS("Eviction manager: failed to update notifier", "notifier", notifier.Description(), "err", err)
			}
		}
	}

	// make observations and get a function to derive pod usage stats relative to those observations.
	// 3. 从统计数据中获得节点资源的使用情况observations
	observations, statsFunc := makeSignalObservations(summary)
	debugLogObservations("observations", observations)

	// determine the set of thresholds met independent of grace period
	// 4. 将资源实际使用量和资源容量进行比较，最终得到阈值结构体对象的列表。举例来说就是，我设置了pid, mem, fs三个thresholds,但是通过观察，可能就是mem这一个限制达到了驱逐阈值
	thresholds = thresholdsMet(thresholds, observations, false) //这个false就是代表立即比较，函数宽限期
	debugLogThresholdsWithObservation("thresholds - ignoring grace period", thresholds, observations)

	// determine the set of thresholds previously met that have not yet satisfied the associated min-reclaim
	// 5. 加上enforceMinReclaim最小强制回收资源值。大概意思就是比如3个阈值，只有一个达到标准，在thresholdsMet检查那些没有
	//达到最小回收量，和之前达到触发的阈值进行合并，得到一个综合的驱逐决策依据
	//避免无效驱逐
	if len(m.thresholdsMet) > 0 {
		thresholdsNotYetResolved := thresholdsMet(m.thresholdsMet, observations, true)
		thresholds = mergeThresholds(thresholds, thresholdsNotYetResolved)
	}
	debugLogThresholdsWithObservation("thresholds - reclaim not satisfied", thresholds, observations)

	// 6.记录每个限制第一次驱逐时间，因为软驱逐会有时间容忍，所以对于软驱逐而言，过来容忍期还是超了阈值，这个时候就要驱逐
	// track when a threshold was first observed
	now := m.clock.Now()
	thresholdsFirstObservedAt := thresholdsFirstObservedAt(thresholds, m.thresholdsFirstObservedAt, now)

	// the set of node conditions that are triggered by currently observed thresholds
	nodeConditions := nodeConditions(thresholds)
	if len(nodeConditions) > 0 {
		klog.V(3).InfoS("Eviction manager: node conditions - observed", "nodeCondition", nodeConditions)
	}

	// track when a node condition was last observed
	nodeConditionsLastObservedAt := nodeConditionsLastObservedAt(nodeConditions, m.nodeConditionsLastObservedAt, now)

	// node conditions report true if it has been observed within the transition period window
	nodeConditions = nodeConditionsObservedSince(nodeConditionsLastObservedAt, m.config.PressureTransitionPeriod, now)
	if len(nodeConditions) > 0 {
		klog.V(3).InfoS("Eviction manager: node conditions - transition period not met", "nodeCondition", nodeConditions)
	}

	// determine the set of thresholds we need to drive eviction behavior (i.e. all grace periods are met)
	thresholds = thresholdsMetGracePeriod(thresholdsFirstObservedAt, now)
	debugLogThresholdsWithObservation("thresholds - grace periods satisfied", thresholds, observations)

	// update internal state
	m.Lock()
	m.nodeConditions = nodeConditions
	m.thresholdsFirstObservedAt = thresholdsFirstObservedAt
	m.nodeConditionsLastObservedAt = nodeConditionsLastObservedAt
	m.thresholdsMet = thresholds

	// determine the set of thresholds whose stats have been updated since the last sync
	thresholds = thresholdsUpdatedStats(thresholds, observations, m.lastObservations)
	debugLogThresholdsWithObservation("thresholds - updated stats", thresholds, observations)

	m.lastObservations = observations
	m.Unlock()
	//如果启用了本地存储容量隔离，则检查是否需要进行本地存储驱逐
	// evict pods if there is a resource usage violation from local volume temporary storage
	// If eviction happens in localStorageEviction function, skip the rest of eviction action
	if m.localStorageCapacityIsolation {
		if evictedPods := m.localStorageEviction(activePods, statsFunc); len(evictedPods) > 0 {
			return evictedPods, nil
		}
	}

	if len(thresholds) == 0 {
		klog.V(3).InfoS("Eviction manager: no resources are starved")
		return nil, nil
	}

	// rank the thresholds by eviction priority
	sort.Sort(byEvictionPriority(thresholds))
	thresholdToReclaim, resourceToReclaim, foundAny := getReclaimableThreshold(thresholds)
	if !foundAny {
		return nil, nil
	}
	klog.InfoS("Eviction manager: attempting to reclaim", "resourceName", resourceToReclaim)

	// record an event about the resources we are now attempting to reclaim via eviction
	m.recorder.Eventf(m.nodeRef, v1.EventTypeWarning, "EvictionThresholdMet", "Attempting to reclaim %s", resourceToReclaim)

	// 7.回收节点级的资源，如果回收的资源足够的话，直接返回，不需要驱逐正在运行中的pod
	// check if there are node-level resources we can reclaim to reduce pressure before evicting end-user pods.
	if m.reclaimNodeLevelResources(ctx, thresholdToReclaim.Signal, resourceToReclaim) {
		klog.InfoS("Eviction manager: able to reduce resource pressure without evicting pods.", "resourceName", resourceToReclaim)
		return nil, nil
	}

	klog.InfoS("Eviction manager: must evict pod(s) to reclaim", "resourceName", resourceToReclaim)

	// rank the pods for eviction
	rank, ok := m.signalToRankFunc[thresholdToReclaim.Signal]
	if !ok {
		klog.ErrorS(nil, "Eviction manager: no ranking function for signal", "threshold", thresholdToReclaim.Signal)
		return nil, nil
	}

	// the only candidates viable for eviction are those pods that had anything running.
	if len(activePods) == 0 {
		klog.ErrorS(nil, "Eviction manager: eviction thresholds have been met, but no pods are active to evict")
		return nil, nil
	}

	// rank the running pods for eviction for the specified resource
	// 8.对不同阈值驱逐场景下pod有不同的排序，比如如果是mem驱逐，就是按照req limit的qos进行排序驱逐
	rank(activePods, statsFunc)

	klog.InfoS("Eviction manager: pods ranked for eviction", "pods", klog.KObjSlice(activePods))

	//record age of metrics for met thresholds that we are using for evictions.
	for _, t := range thresholds {
		timeObserved := observations[t.Signal].time
		if !timeObserved.IsZero() {
			metrics.EvictionStatsAge.WithLabelValues(string(t.Signal)).Observe(metrics.SinceInSeconds(timeObserved.Time))
		}
	}
	// 9.按照排序后的结果每次驱逐一个pod,判断是硬性驱逐还是软性驱逐，annotations包含了为什么被驱逐，启用了 PodDisruptionConditions 功能，则创建一个 Pod 条件对象，标记当前 Pod 为驱逐目标。
	//PodDisruptionConditions就是中断，可以追踪调试驱逐的Pod
	// we kill at most a single pod during each eviction interval
	for i := range activePods {
		pod := activePods[i]
		gracePeriodOverride := int64(0)
		if !isHardEvictionThreshold(thresholdToReclaim) {
			gracePeriodOverride = m.config.MaxPodGracePeriodSeconds
		}
		message, annotations := evictionMessage(resourceToReclaim, pod, statsFunc, thresholds, observations)
		var condition *v1.PodCondition
		if utilfeature.DefaultFeatureGate.Enabled(features.PodDisruptionConditions) {
			condition = &v1.PodCondition{
				Type:    v1.DisruptionTarget,
				Status:  v1.ConditionTrue,
				Reason:  v1.PodReasonTerminationByKubelet,
				Message: message,
			}
		}
		if m.evictPod(pod, gracePeriodOverride, message, annotations, condition) {
			metrics.Evictions.WithLabelValues(string(thresholdToReclaim.Signal)).Inc()
			return []*v1.Pod{pod}, nil
		}
	}
	klog.InfoS("Eviction manager: unable to evict any pods from the node")
	return nil, nil
}
```

#### 3.1.1 summaryProvider.Get(ctx, updateStats)

可以看到，这里核心就是从cadvisor算出2个数据，nodeStats 和podStats。

```go

func (sp *summaryProviderImpl) Get(ctx context.Context, updateStats bool) (*statsapi.Summary, error) {
.............................
	nodeStats := statsapi.NodeStats{
		NodeName:         node.Name,
		CPU:              rootStats.CPU,
		Memory:           rootStats.Memory,
		Swap:             rootStats.Swap,
		Network:          networkStats,
		StartTime:        sp.systemBootTime,
		Fs:               rootFsStats,
		Runtime:          &statsapi.RuntimeStats{ContainerFs: containerFsStats, ImageFs: imageFsStats},
		Rlimit:           rlimit,
		SystemContainers: sp.GetSystemContainersStats(nodeConfig, podStats, updateStats),
	}
	summary := statsapi.Summary{
		Node: nodeStats,
		Pods: podStats,
	}
	return &summary, nil
}

```

以mem为例：

可以看到，pods的memlimit, RSSBytes,UsageBytes等信息都统计在内,都是从cadvisor采集的

```
if info.Spec.HasMemory && cstat.Memory != nil {
		pageFaults := cstat.Memory.ContainerData.Pgfault
		majorPageFaults := cstat.Memory.ContainerData.Pgmajfault
		memoryStats = &statsapi.MemoryStats{
			Time:            metav1.NewTime(cstat.Timestamp),
			UsageBytes:      &cstat.Memory.Usage,
			WorkingSetBytes: &cstat.Memory.WorkingSet,
			RSSBytes:        &cstat.Memory.RSS,
			PageFaults:      &pageFaults,
			MajorPageFaults: &majorPageFaults,
		}
		// availableBytes = memory limit (if known) - workingset
		if !isMemoryUnlimited(info.Spec.Memory.Limit) {
			availableBytes := info.Spec.Memory.Limit - cstat.Memory.WorkingSet
			memoryStats.AvailableBytes = &availableBytes
		}
```

#### 3.1.2 makeSignalObservations

构造的Observation就是

```go
// makeSignalObservations derives observations using the specified summary provider.
func makeSignalObservations(summary *statsapi.Summary) (signalObservations, statsFunc) {
	// build the function to work against for pod stats
	statsFunc := cachedStatsFunc(summary.Pods)
	// build an evaluation context for current eviction signals
	result := signalObservations{}

	if memory := summary.Node.Memory; memory != nil && memory.AvailableBytes != nil && memory.WorkingSetBytes != nil {
		result[evictionapi.SignalMemoryAvailable] = signalObservation{
			available: resource.NewQuantity(int64(*memory.AvailableBytes), resource.BinarySI),
			capacity:  resource.NewQuantity(int64(*memory.AvailableBytes+*memory.WorkingSetBytes), resource.BinarySI),
			time:      memory.Time,
		}
	}
	if allocatableContainer, err := getSysContainer(summary.Node.SystemContainers, statsapi.SystemContainerPods); err != nil {
		klog.ErrorS(err, "Eviction manager: failed to construct signal", "signal", evictionapi.SignalAllocatableMemoryAvailable)
	} else {
		if memory := allocatableContainer.Memory; memory != nil && memory.AvailableBytes != nil && memory.WorkingSetBytes != nil {
			result[evictionapi.SignalAllocatableMemoryAvailable] = signalObservation{
				available: resource.NewQuantity(int64(*memory.AvailableBytes), resource.BinarySI),
				capacity:  resource.NewQuantity(int64(*memory.AvailableBytes+*memory.WorkingSetBytes), resource.BinarySI),
				time:      memory.Time,
			}
		}
	}
	if nodeFs := summary.Node.Fs; nodeFs != nil {
		if nodeFs.AvailableBytes != nil && nodeFs.CapacityBytes != nil {
			result[evictionapi.SignalNodeFsAvailable] = signalObservation{
				available: resource.NewQuantity(int64(*nodeFs.AvailableBytes), resource.BinarySI),
				capacity:  resource.NewQuantity(int64(*nodeFs.CapacityBytes), resource.BinarySI),
				time:      nodeFs.Time,
			}
		}
		if nodeFs.InodesFree != nil && nodeFs.Inodes != nil {
			result[evictionapi.SignalNodeFsInodesFree] = signalObservation{
				available: resource.NewQuantity(int64(*nodeFs.InodesFree), resource.DecimalSI),
				capacity:  resource.NewQuantity(int64(*nodeFs.Inodes), resource.DecimalSI),
				time:      nodeFs.Time,
			}
		}
	}
	if summary.Node.Runtime != nil {
		if imageFs := summary.Node.Runtime.ImageFs; imageFs != nil {
			if imageFs.AvailableBytes != nil && imageFs.CapacityBytes != nil {
				result[evictionapi.SignalImageFsAvailable] = signalObservation{
					available: resource.NewQuantity(int64(*imageFs.AvailableBytes), resource.BinarySI),
					capacity:  resource.NewQuantity(int64(*imageFs.CapacityBytes), resource.BinarySI),
					time:      imageFs.Time,
				}
			}
			if imageFs.InodesFree != nil && imageFs.Inodes != nil {
				result[evictionapi.SignalImageFsInodesFree] = signalObservation{
					available: resource.NewQuantity(int64(*imageFs.InodesFree), resource.DecimalSI),
					capacity:  resource.NewQuantity(int64(*imageFs.Inodes), resource.DecimalSI),
					time:      imageFs.Time,
				}
			}
		}
		if containerFs := summary.Node.Runtime.ContainerFs; containerFs != nil {
			if containerFs.AvailableBytes != nil && containerFs.CapacityBytes != nil {
				result[evictionapi.SignalContainerFsAvailable] = signalObservation{
					available: resource.NewQuantity(int64(*containerFs.AvailableBytes), resource.BinarySI),
					capacity:  resource.NewQuantity(int64(*containerFs.CapacityBytes), resource.BinarySI),
					time:      containerFs.Time,
				}
			}
			if containerFs.InodesFree != nil && containerFs.Inodes != nil {
				result[evictionapi.SignalContainerFsInodesFree] = signalObservation{
					available: resource.NewQuantity(int64(*containerFs.InodesFree), resource.DecimalSI),
					capacity:  resource.NewQuantity(int64(*containerFs.Inodes), resource.DecimalSI),
					time:      containerFs.Time,
				}
			}
		}
	}
	if rlimit := summary.Node.Rlimit; rlimit != nil {
		if rlimit.NumOfRunningProcesses != nil && rlimit.MaxPID != nil {
			available := int64(*rlimit.MaxPID) - int64(*rlimit.NumOfRunningProcesses)
			result[evictionapi.SignalPIDAvailable] = signalObservation{
				available: resource.NewQuantity(available, resource.DecimalSI),
				capacity:  resource.NewQuantity(int64(*rlimit.MaxPID), resource.DecimalSI),
				time:      rlimit.Time,
			}
		}
	}
	return result, statsFunc
}
```

### 3.2 waitForPodsCleanup



waitForPodsCleanup逻辑很简单, 就是调用PodResourcesAreReclaimed清理容器volume， cgroup资源

```
func (m *managerImpl) waitForPodsCleanup(podCleanedUpFunc PodCleanedUpFunc, pods []*v1.Pod) {
	timeout := m.clock.NewTimer(podCleanupTimeout)
	defer timeout.Stop()
	ticker := m.clock.NewTicker(podCleanupPollFreq)
	defer ticker.Stop()
	for {
		select {
		case <-timeout.C():
			klog.InfoS("Eviction manager: timed out waiting for pods to be cleaned up", "pods", klog.KObjSlice(pods))
			return
		case <-ticker.C():
			for i, pod := range pods {
				if !podCleanedUpFunc(pod) {
					break
				}
				if i == len(pods)-1 {
					klog.InfoS("Eviction manager: pods successfully cleaned up", "pods", klog.KObjSlice(pods))
					return
				}
			}
		}
	}
}

```





```
// PodResourcesAreReclaimed returns true if all required node-level resources that a pod was consuming have
// been reclaimed by the kubelet.  Reclaiming resources is a prerequisite to deleting a pod from the API server.
func (kl *Kubelet) PodResourcesAreReclaimed(pod *v1.Pod, status v1.PodStatus) bool {
	if !notRunning(status.ContainerStatuses) {
		// We shouldn't delete pods that still have running containers
		klog.V(3).Infof("Pod %q is terminated, but some containers are still running", format.Pod(pod))
		return false
	}
	// pod's containers should be deleted
	runtimeStatus, err := kl.podCache.Get(pod.UID)
	if err != nil {
		klog.V(3).Infof("Pod %q is terminated, Error getting runtimeStatus from the podCache: %s", format.Pod(pod), err)
		return false
	}
	if len(runtimeStatus.ContainerStatuses) > 0 {
		var statusStr string
		for _, status := range runtimeStatus.ContainerStatuses {
			statusStr += fmt.Sprintf("%+v ", *status)
		}
		klog.V(3).Infof("Pod %q is terminated, but some containers have not been cleaned up: %s", format.Pod(pod), statusStr)
		return false
	}
	if kl.podVolumesExist(pod.UID) && !kl.keepTerminatedPodVolumes {
		// We shouldn't delete pods whose volumes have not been cleaned up if we are not keeping terminated pod volumes
		klog.V(3).Infof("Pod %q is terminated, but some volumes have not been cleaned up", format.Pod(pod))
		return false
	}
	if kl.kubeletConfiguration.CgroupsPerQOS {
		pcm := kl.containerManager.NewPodContainerManager()
		if pcm.Exists(pod) {
			klog.V(3).Infof("Pod %q is terminated, but pod cgroup sandbox has not been cleaned up", format.Pod(pod))
			return false
		}
	}
	return true
}
```

## 4. 总结



kubelet驱逐整体是比较明确的，就是每10s进行一次判断，如果超过了阈值就驱逐。

使用上可以参考[官方的文档](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/reserve-compute-resources/)。但是官方文档有个错误在于，memory.available是包含system-reserved，kube-reserved这些的，它指的是宿主可以用的资源。

+ 意思就是比如Node节点总内存30G，system-reserved=20G为K8S系统保留的，不包含Pod节点分配，所以要驱逐的话就是system-reserved+viction-hard=必须加上20G

举个例子：

这样是基本上不可能触发mem驱逐的。因为这个驱逐条件是宿主可用的资源小于2Gi, 但是给系统保留了20Gi，所以很难因为pod mem压力大而实现驱逐。反而会因为pod 使用mem过大，超过limit，会触发oom而不是驱逐。

```
--system-reserved=cpu=2000m,memory=20Gi --eviction-hard=memory.available<2Gi,nodefs.available<1Mi,nodefs.inodesFree<1
```



可用这样设置，就是当宿主可用资源小于25G的时候进行驱逐。这样的设置给了pod 5Gi的空间。当pod可用资源只剩下5Gi的时候，先驱逐，而不是oom。

```
--system-reserved=cpu=2000m,memory=20Gi --eviction-hard=memory.available<25Gi,nodefs.available<1Mi,nodefs.inodesFree<1
```



但是需要注意：oom是个系统概率，驱逐时10s的延迟概念。

当pod只剩下5Gi空间可用时，如果10s内pod使用的mem超过5G，oom会先发出来。

当pod只剩下5Gi空间可用时，如果10s内pod使用的mem不超过5G，驱逐会先发出来。



代码详见：

SignalMemoryAvailable直接就是mem threshold。设置多少就是多少，包含了system-reserved，kube-reserved

```
// hardEvictionReservation returns a resourcelist that includes reservation of resources based on hard eviction thresholds.
func hardEvictionReservation(thresholds []evictionapi.Threshold, capacity v1.ResourceList) v1.ResourceList {
	if len(thresholds) == 0 {
		return nil
	}
	ret := v1.ResourceList{}
	for _, threshold := range thresholds {
		if threshold.Operator != evictionapi.OpLessThan {
			continue
		}
		switch threshold.Signal {
		case evictionapi.SignalMemoryAvailable:
			memoryCapacity := capacity[v1.ResourceMemory]
			value := evictionapi.GetThresholdQuantity(threshold.Value, &memoryCapacity)
			ret[v1.ResourceMemory] = *value
		case evictionapi.SignalNodeFsAvailable:
			storageCapacity := capacity[v1.ResourceEphemeralStorage]
			value := evictionapi.GetThresholdQuantity(threshold.Value, &storageCapacity)
			ret[v1.ResourceEphemeralStorage] = *value
		}
	}
	return ret
}
```

