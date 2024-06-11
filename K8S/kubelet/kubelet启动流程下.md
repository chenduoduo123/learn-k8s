### 1. 背景



书接上文，在Kubelet.Run函数中，通过pleg.Start和kl.syncLoop来处理pod，所有本文从这里开始，从源码角度了解pod创建的整个过程。

```
kl.pleg.Start()
kl.syncLoop(updates, kl)
```



### 2. pleg.Start

relist的逻辑如下：

 **加锁和解锁**: 使用 `g.relistLock.Lock()` 和 `defer g.relistLock.Unlock()` 来确保在同一时间内只有一个 `Relist` 进程在执行，防止数据竞争。

**记录开始时间和间隔**: 方法开始时记录当前时间，并在方法结束时记录执行的持续时间和自上次 `Relist` 以来的时间间隔，用于监控和性能分析。

**获取所有 Pods**: 调用 `g.runtime.GetPods(ctx, true)` 获取当前所有的 Pod。这里的 `true` 参数表示获取包括终止但仍未清除的 Pod。

**更新 Pod 列表和生成事件**: 使用从容器运行时获取的 Pods 列表，与内部记录的 Pod 列表进行比较，为发生变化的 Pod 生成相应的生命周期事件。

**处理缓存**: 如果启用了缓存 (`g.cacheEnabled()` 返回 `true`)，则对有更新的 Pod 进行缓存更新。这包拲调用 `updateCache` 方法来检查 Pod 的最新状态并更新缓存。

**发送事件**: 遍历生成的事件，并通过 `g.eventChannel` 发送。如果事件通道已满，则丢弃该事件并记录。

**处理容器死亡事件**: 对于每个容器死亡事件，记录容器的退出代码。这需要从缓存中获取最新的 Pod 状态来查找对应的退出代码。

**重新检查失败的 Pods**: 如果有 Pod 在更新缓存时失败了，它们将被记录并在下一次 `Relist` 调用时重新检查。

**更新缓存的时间戳**: 在所有 Pod 都被正确地更新在缓存之后，更新缓存的时间戳。

这里需要注意的是：查看generateEvents函数，可以发现只有当新的container状态为plegContainerUnknown，才会产生ContainerChanged。这就是发送eventa为什么会跳过ContainerChanged的原因。因为plegContainerUnknown代表未知，就是kubelet和容器运行时交互遇到了困难或者发生了错误

```go
func (g *GenericPLEG) Relist() {
	//使用互斥锁确保在同一时间内只有一个 Relist 进程在执行，避免数据竞争。
	g.relistLock.Lock()
	defer g.relistLock.Unlock()
	//创建一个背景上下文 ctx，并记录日志表明开始了一次 relisting 操作。
	ctx := context.Background()
	klog.V(5).InfoS("GenericPLEG: Relisting")
	//检查上次 relist 的时间，并更新 relist 间隔的监控指标。
	//有两个时间，一个是内部记录relist的时间，一个是暴露监控指标的时间，这个函数开始先更新指标监控的上次的relist时间，然后记录当relist当前时间，最后通过defer更新relist内部记录时间
	if lastRelistTime := g.getRelistTime(); !lastRelistTime.IsZero() {
		metrics.PLEGRelistInterval.Observe(metrics.SinceInSeconds(lastRelistTime))
	}
	//记录当前时间戳，用于计算此次 relist 操作的持续时间，并在函数返回前通过 defer 更新 relist 持续时间的监控指标。
	timestamp := g.clock.Now()
	defer func() {
		metrics.PLEGRelistDuration.Observe(metrics.SinceInSeconds(timestamp))
	}()

	// Get all the pods.
	//调用运行时接口获取所有的 Pods（包括已终止但尚未清理的 Pods）。如果调用失败，记录错误并返回。
	podList, err := g.runtime.GetPods(ctx, true)
	if err != nil {
		klog.ErrorS(err, "GenericPLEG: Unable to retrieve pods")
		return
	}
	//更新内部记录的最后一次 relist 的时间戳。
	g.updateRelistTime(timestamp)
	//将获取到的 Pod 列表转换为内部使用的格式，并更新运行中的 Pod 和容器的监控指标。同时，设置当前的 Pod 记录。
	pods := kubecontainer.Pods(podList)
	// update running pod and container count
	//updateRunningPodAndContainerMetrics便利podlist，然后取出单个pod中的所有容器，用map[string容器状态列如：running，stop]int，统计pod里面的running，stop容器数量
	//更新到指标监控中，然后便利pod里面Sandbox 容器，如果Sandbox的状态字段是ContainerStateRunning，就更新Pod 总数的监控指标
	updateRunningPodAndContainerMetrics(pods)
	g.podRecords.setCurrent(pods)

	// Compare the old and the current pods, and generate events.
	//初始化一个用于存储按 Pod ID 分类的事件列表的字典。
	eventsByPodID := map[types.UID][]*PodLifecycleEvent{}
	//遍历内部记录的所有 Pods，比边它们的旧状态和新状态，计算出不同的生命周期事件（如容器启动、停止等）。
	for pid := range g.podRecords {
		oldPod := g.podRecords.getOld(pid)
		//传入pid返回旧的pod
		pod := g.podRecords.getCurrent(pid)
		//传入pid返回现在的pod
		// Get all containers in the old and the new pod.
		//getContainersFromPods返回一个pod中所有容器，包括旧的和新的
		allContainers := getContainersFromPods(oldPod, pod)
		//便利所有容器，包含新的和旧的
		for _, container := range allContainers {
			//调用computeEvents计算从旧的容器状态到新的容器状态的生命周期事件
			events := computeEvents(oldPod, pod, &container.ID)
			for _, e := range events {
				//使用updateEvents把生命周期事件添加到eventsByPodID
				updateEvents(eventsByPodID, e)
			}
		}
	}
	//如果启用了缓存，初始化一个字典来记录需要重新检查的 Pods。
	//kubelet启用缓存就是使用一个本地缓存来存储更新pod状态，不用直接和容器运行时交互，这样可以减少网络通信的次数。即使与容器运行时的通信暂时中断，Kubelet 也可以使用缓存中的数据继续其操作。
	//缓存会定期和容器运行时同步状态
	var needsReinspection map[types.UID]*kubecontainer.Pod
	if g.cacheEnabled() {
		needsReinspection = make(map[types.UID]*kubecontainer.Pod)
	}

	// If there are events associated with a pod, we should update the
	// podCache.
	//遍历所有生成的事件，更新缓存并处理需要重新检查的 Pods。对每个有事件的 Pod，更新其在内部记录中的状态，并将事件发送到事件通道。
	for pid, events := range eventsByPodID {
		//返回旧的pod
		pod := g.podRecords.getCurrent(pid)
		if g.cacheEnabled() {
			// updateCache() will inspect the pod and update the cache. If an
			// error occurs during the inspection, we want PLEG to retry again
			// in the next relist. To achieve this, we do not update the
			// associated podRecord of the pod, so that the change will be
			// detect again in the next relist.
			// TODO: If many pods changed during the same relist period,
			// inspecting the pod and getting the PodStatus to update the cache
			// serially may take a while. We should be aware of this and
			// parallelize if needed.
			if err, updated := g.updateCache(ctx, pod, pid); err != nil {
				// Rely on updateCache calling GetPodStatus to log the actual error.
				klog.V(4).ErrorS(err, "PLEG: Ignoring events for pod", "pod", klog.KRef(pod.Namespace, pod.Name))

				// make sure we try to reinspect the pod during the next relisting
				//如果更新失败，就是pod为空或者获取这个pod失败，就把这个pod放到需要重新检查的列表中
				needsReinspection[pid] = pod
				// 使用 continue 跳过后续的代码，继续处理下一个 Pod
				continue
			} else {
				// this pod was in the list to reinspect and we did so because it had events, so remove it
				// from the list (we don't want the reinspection code below to inspect it a second time in
				// this relist execution)
				//成功执行updateCache，就从podsToReinspect中删除这个pod，podsToReinspect是需要被重新检查的pod列表,podsToReinspect是全局的
				delete(g.podsToReinspect, pid)
				//如果启用了 Evented PLEG 特性，代码会进一步检查 updated 标志：
				if utilfeature.DefaultFeatureGate.Enabled(features.EventedPLEG) {
					//updated 返回 false，则使用 continue 跳过当前循环的剩余部分
					if !updated {
						continue
						//跳转到下面的代码中去
					}
				}
			}
		}
		// Update the internal storage and send out the events.
		//更新内部记录（如内存中的状态数据），以反映最新的 Pod 状态
		g.podRecords.update(pid)

		// Map from containerId to exit code; used as a temporary cache for lookup
		//这里初始化了一个映射 containerExitCode，它的作用是临时存储容器的退出代码。该映射将容器 ID 映射到对应的退出代码。
		containerExitCode := make(map[string]int)
		//这段代码遍历所有捕获到的事件。如果事件类型是 ContainerChanged，这通常表示容器状态有变化但不是终结状态，因此这些事件被跳过，不进行进一步处理。
		for i := range events {
			// Filter out events that are not reliable and no other components use yet.
			if events[i].Type == ContainerChanged {
				continue
			}
			//如果事件类型不是ContainerChanged就发送到g.eventRequestChannel。如果事件通道已满，无法发送更多事件，将增加丢弃事件的计数，并记录错误日志。这是为了防止程序阻塞在一个满的通道上。
			select {
			case g.eventChannel <- events[i]:
			default:
				metrics.PLEGDiscardEvents.Inc()
				klog.ErrorS(nil, "Event channel is full, discard this relist() cycle event")
			}
			// Log exit code of containers when they finished in a particular event
			//这段代码处理 ContainerDied 事件，首次出现时，从缓存中获取更新的 Pod 状态并填充 containerExitCode 映射。这样可以在后续处理中快速查找容器的退出代码。
			if events[i].Type == ContainerDied {
				// Fill up containerExitCode map for ContainerDied event when first time appeared
				//containerExitCode== 0，代表containerExitCode这个map是空的。就需要从缓存中获取pod状态，并填充containerExitCode映射。
				//如果!=0,代表containerExitCode这个map已经填充了，不需要再从缓存中获取pod状态。
				if len(containerExitCode) == 0 && pod != nil && g.cache != nil {
					// Get updated podStatus
					status, err := g.cache.Get(pod.ID)
					if err == nil {
						for _, containerStatus := range status.ContainerStatuses {
							containerExitCode[containerStatus.ID.ID] = containerStatus.ExitCode
						}
					}
				}
				//如果能成功获取容器的退出代码，这里将记录相关的日志信息，包括容器的 ID、Pod 的 ID 和容器的退出代码。这对于诊断容器退出的原因非常有用。
				if containerID, ok := events[i].Data.(string); ok {
					if exitCode, ok := containerExitCode[containerID]; ok && pod != nil {
						klog.V(2).InfoS("Generic (PLEG): container finished", "podID", pod.ID, "containerID", containerID, "exitCode", exitCode)
					}
				}
			}
		}
	}
	//如果开启了缓存，检查podsToReinspect是否为空，不为空就还有需要检查的pod，重新用updateCache检查，如果还有问题就继续加入needsReinspection继续检查的map中
	if g.cacheEnabled() {
		// reinspect any pods that failed inspection during the previous relist
		if len(g.podsToReinspect) > 0 {
			klog.V(5).InfoS("GenericPLEG: Reinspecting pods that previously failed inspection")
			for pid, pod := range g.podsToReinspect {
				if err, _ := g.updateCache(ctx, pod, pid); err != nil {
					// Rely on updateCache calling GetPodStatus to log the actual error.
					klog.V(5).ErrorS(err, "PLEG: pod failed reinspection", "pod", klog.KRef(pod.Namespace, pod.Name))
					needsReinspection[pid] = pod
				}
			}
		}

		// Update the cache timestamp.  This needs to happen *after*
		// all pods have been properly updated in the cache.
		//更新缓存时间戳。必须是需要缓存中需要检查的pod都更新完了才能更新时间戳
		g.cache.UpdateTime(timestamp)
	}

	// make sure we retain the list of pods that need reinspecting the next time relist is called
	//确保下次调用relist的时候，podsToReinspect=需要检查的pod
	g.podsToReinspect = needsReinspection
}
```

从generateEvents可以看出来，这里产生的event和k8s中的event不同。这里指的是PodLifecycleEvent。

**容器状态处理**:

- `plegContainerRunning`: 如果容器的新状态是运行中，生成一个 `ContainerStarted` 事件。
- `plegContainerExited`: 如果容器已经退出，生成一个 `ContainerDied` 事件。
- `plegContainerUnknown`: 如果容器状态未知，生成一个 `ContainerChanged` 事件。
- `plegContainerNonExistent`: 如果容器不存在：
    - 如果旧状态是已退出，生成一个 `ContainerRemoved` 事件，表示容器之前已经报告过退出。
    - 否则，生成一个 `ContainerDied` 和一个 `ContainerRemoved` 事件，表示容器已经停止并且被移除。

```go
func generateEvents(podID types.UID, cid string, oldState, newState plegContainerState) []*PodLifecycleEvent {
	//如果容器的新状态与旧状态相同，说明没有发生状态变化，因此不生成任何事件，直接返回 nil。
    if newState == oldState {
		return nil
	}

	klog.V(4).InfoS("GenericPLEG", "podUID", podID, "containerID", cid, "oldState", oldState, "newState", newState)
	switch newState {
	case plegContainerRunning:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerStarted, Data: cid}}
	case plegContainerExited:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerDied, Data: cid}}
	case plegContainerUnknown:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerChanged, Data: cid}}
	case plegContainerNonExistent:
		switch oldState {
		case plegContainerExited:
			// We already reported that the container died before.
			return []*PodLifecycleEvent{{ID: podID, Type: ContainerRemoved, Data: cid}}
		default:
			return []*PodLifecycleEvent{{ID: podID, Type: ContainerDied, Data: cid}, {ID: podID, Type: ContainerRemoved, Data: cid}}
		}
	default:
		panic(fmt.Sprintf("unrecognized container state: %v", newState))
	}
}
```



* updateCache更新kubelet本地缓存

```go
func (g *GenericPLEG) updateCache(ctx context.Context, pod *kubecontainer.Pod, pid types.UID) (error, bool) {
	//如果传入的 pod 为 nil，说明 Pod 在当前 relist 过程中不存在，可能是被删除了。这时候会从缓存里删除该 Pod 的状态，并返回 nil 和 true 表示成功删除。
	if pod == nil {
		// The pod is missing in the current relist. This means that
		// the pod has no visible (active or inactive) containers.
		klog.V(4).InfoS("PLEG: Delete status for pod", "podUID", string(pid))
		g.cache.Delete(pid)
		return nil, true
	}
	//使用锁 podnet. CacheMutex 确保缓存更新操作的线程安全，使用 defer 来确保函数结束时释放锁。
	g.podCacheMutex.Lock()
	defer g.podCacheMutex.Unlock()
	//获取当前时间戳，这个时间戳用于记录状态更新的时间。
	timestamp := g.clock.Now()
	//调用容器运行时接口获取最新的 Pod 状态。这里可能返回错误，如果发生错误则后续处理。
	status, err := g.runtime.GetPodStatus(ctx, pod.ID, pod.Name, pod.Namespace)
	if err != nil {
		// nolint:logcheck // Not using the result of klog.V inside the
		// if branch is okay, we just use it to determine whether the
		// additional "podStatus" key and its value should be added.
		//根据日志等级失败或者成功记录或者不记录日志。
		if klog.V(6).Enabled() {
			klog.ErrorS(err, "PLEG: Write status", "pod", klog.KRef(pod.Namespace, pod.Name), "podStatus", status)
		} else {
			klog.ErrorS(err, "PLEG: Write status", "pod", klog.KRef(pod.Namespace, pod.Name))
		}
	} else {
		if klogV := klog.V(6); klogV.Enabled() {
			klogV.InfoS("PLEG: Write status", "pod", klog.KRef(pod.Namespace, pod.Name), "podStatus", status)
		} else {
			klog.V(4).InfoS("PLEG: Write status", "pod", klog.KRef(pod.Namespace, pod.Name))
		}
		// Preserve the pod IP across cache updates if the new IP is empty.
		// When a pod is torn down, kubelet may race with PLEG and retrieve
		// a pod status after network teardown, but the kubernetes API expects
		// the completed pod's IP to be available after the pod is dead.
		//从缓存中获取并更新 Pod 的 IP 地址。这是因为在 Pod 终止后，可能还需要保留其 IP 地址，尽管网络已经被拆除。
		status.IPs = g.getPodIPs(pid, status)
	}

	// When we use Generic PLEG only, the PodStatus is saved in the cache without
	// any validation of the existing status against the current timestamp.
	// This works well when there is only Generic PLEG setting the PodStatus in the cache however,
	// if we have multiple entities, such as Evented PLEG, while trying to set the PodStatus in the
	// cache we may run into the racy timestamps given each of them were to calculate the timestamps
	// in their respective execution flow. While Generic PLEG calculates this timestamp and gets
	// the PodStatus, we can only calculate the corresponding timestamp in
	// Evented PLEG after the event has been received by the Kubelet.
	// For more details refer to:
	// https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/3386-kubelet-evented-pleg#timestamp-of-the-pod-status
	//如果启用了事件驱动的 PLEG 特性并且正在使用中，将使用从事件中获取的时间戳来更新缓存，以保证时间的一致性。
	if utilfeature.DefaultFeatureGate.Enabled(features.EventedPLEG) && isEventedPLEGInUse() && status != nil {
		timestamp = status.TimeStamp
	}
	//最后，将获取的 Pod 状态更新到缓存中，并返回操作结果。如果更新成功，返回的错误为 nil，布尔值为 true 表示缓存已更新。
	return err, g.cache.Set(pod.ID, status, err, timestamp)
}

```

### 3.syncLoop

从函数的注释了解到：syncLoop是核心的同步逻辑。它监听file, apiserver, and http三个channel的变化，然后进行期望状态和当前状态的同步。

syncLoop核心是调用了`syncLoopIteration`的函数来执行更具体的监控pod变化的循环。

**日志记录**：

- 在同步循环开始时，记录启动日志，以便跟踪。

**定时器设置**：

- `syncTicker`: 每秒触发一次，用于定期同步 Pod 状态。
- `housekeepingTicker`: 按设定的 `housekeepingPeriod` 触发，用于执行系统维护任务。
- 使用 `defer` 关键字确保定时器在函数退出时停止。

**PLEG 监听**：

- 通过 `pleg.Watch()` 方法获得 PLEG 事件通道 `plegCh`，用于监听 Pod 生命周期事件。

**错误处理与指数回退**：

- 在每次循环开始时检查运行时错误。如果存在，则记录错误并应用指数回退机制（使用 `time.Sleep`），延迟下一次同步尝试。

**DNS 配置检查**：

- 如果 DNS 配置器已设置并指定了解析配置，检查 `resolv.conf` 中的限制。

**主循环**：

- 无限循环，直到接收到终止信号。
- 在循环中，先更新监控状态，然后调用 `syncLoopIteration` 来处理实际的同步任务。
- 使用来自 `syncTicker`、`housekeepingTicker` 和 `plegCh` 的事件驱动同步操作。
- 如果 `syncLoopIteration` 返回 `false`，则终止循环。

```go
func (kl *Kubelet) syncLoop(ctx context.Context, updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	//记录日志信息，表明 Kubelet 的主同步循环已经启动。
	klog.InfoS("Starting kubelet main sync loop")
	// The syncTicker wakes up kubelet to checks if there are any pod workers
	// that need to be sync'd. A one-second period is sufficient because the
	// sync interval is defaulted to 10s.
	//创建一个每秒触发一次的定时器 syncTicker，用于周期性地触发检查操作。使用 defer 来确保在 syncLoop 方法结束时停止这个定时器。
	syncTicker := time.NewTicker(time.Second)
	defer syncTicker.Stop()
	//创建另一个定时器 housekeepingTicker，用于进行系统维护相关操作。housekeepingPeriod 是一个预定义的维护周期。同样使用 defer 来确保定时器被停止。
	housekeepingTicker := time.NewTicker(housekeepingPeriod)
	defer housekeepingTicker.Stop()
	//调用 kl.pleg.Watch() 来获取 PLEG (Pod Lifecycle Event Generator) 的事件通道 plegCh。PLEG 负责监听 Pod 的生命周期事件，并生成相应的事件通道。
	plegCh := kl.pleg.Watch()
	//定义了几个常量，用于配置错误重试策略中的指数回退机制。
	const (
		base   = 100 * time.Millisecond
		max    = 5 * time.Second
		factor = 2
	)
	//初始化重试延迟 duration 为基本值 base。
	duration := base
	// Responsible for checking limits in resolv.conf
	// The limits do not have anything to do with individual pods
	// Since this is called in syncLoop, we don't need to call it anywhere else
	//如果 DNS 配置器 kl.dnsConfigurer 存在并且已配置 ResolverConfig，则检查 resolv.conf 中的限制。这通常与 DNS 相关的配置项限制有关。
	if kl.dnsConfigurer != nil && kl.dnsConfigurer.ResolverConfig != "" {
		kl.dnsConfigurer.CheckLimitsForResolvConf()
	}

	for {
		//检查运行时状态中是否有错误。如果有错误，记录错误并进行指数回退，然后继续下一个循环迭代。
		if err := kl.runtimeState.runtimeErrors(); err != nil {
			klog.ErrorS(err, "Skipping pod synchronization")
			// exponential backoff
			time.Sleep(duration)
			duration = time.Duration(math.Min(float64(max), factor*float64(duration)))
			continue
		}
		// reset backoff if we have a success
		//如果没有错误，重置延迟 duration 为基础值 base。
		duration = base
		//在监视器中存储当前时间，表示一个同步循环迭代的开始。
		kl.syncLoopMonitor.Store(kl.clock.Now())
		//执行一个同步循环迭代。如果 syncLoopIteration 方法返回 false，则终止循环。
		if !kl.syncLoopIteration(ctx, updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		//再次在监视器中存储当前时间，表示同步循环迭代的结束。
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
}
```

#### 3.1 syncLoopIteration 相关channel介绍

`syncLoopIteration`主要通过几种`channel`来对不同类型的事件进行监听并处理。其中包括：`configCh`、`plegCh`、`syncCh`、`houseKeepingCh`、`livenessManager.Updates()`。

`syncLoopIteration`实际执行了pod的操作，此部分设置了几种不同的channel:

- `configCh`：将配置更改的pod分派给事件类型的相应处理程序回调。
- `plegCh`：更新runtime缓存，同步pod。
- `syncCh`：同步所有等待同步的pod。
- `houseKeepingCh`：触发清理pod。
- `livenessManager.Updates()`：对失败的pod或者liveness检查失败的pod进行sync操作。

1. configCh

- **来源**：`configCh` 通道来自于 `makePodSourceConfig` 函数，该函数负责从不同的配置源创建和配置 Pod 更新事件的通道。这些配置源可能包括==静态文件==(目录）、==API== 服务器或指定的 ==URL==。
- **作用**：`configCh` 用于接收关于 Pod 配置变更的通知，如 Pod 的添加、更新、删除等。这些通知触发 Kubelet 去处理这些变更，确保 Pod 的状态与配置同步。

2. plegCh

- **来源**：`plegCh` 通道直接来自于 PLEG (Pod Lifecycle Event Generator) 组件。PLEG 通过调用 `pleg.Watch()` 方法获取，该方法返回一个事件通道。
- **作用**：`plegCh` 用于接收底层容器运行时的事件，例如容器启动、停止、重启。这些事件反映了 Pod 的生命周期变化，需要由 Kubelet 处理以保持系统状态的一致性。

3. syncCh

- **来源**：`syncCh` 通道由 `time.NewTicker` 创建，通常设定为每秒触发一次。
- **作用**：`syncCh` 用于定期触发 Pod 的同步操作。这保证了即使没有外部事件触发，Kubelet 也会定期检查并同步所有 Pod 的状态，保持集群的健康和反应灵敏。

4. housekeepingCh

- **来源**：`housekeepingCh` 也是由 `time.NewTicker` 创建，触发频率通常比 `syncCh` 慢，例如每两秒一次。
- **作用**：`housekeepingCh` 用于触发清理任务，如资源回收和状态检查等。这有助于维持系统的清洁和高效运行。

5. livenessManager.Updates

- **来源**：这个通道来自于 Kubelet 中的 liveness probe 管理器，该管理器定期检查 Pod 的存活探针。
- **作用**：当 liveness probe 失败时，该通道会接收到更新事件。这些事件会触发相关 Pod 的同步操作，可能包括重启容器来恢复服务。

### 4.syncLoopIteration源码分析

* `configCh`

`kubetypes.ADD`

- **日志记录**：使用 `klog.V(2).InfoS` 记录事件信息，包括操作类型（ADD），来源以及受影响的 Pods。
- **操作说明**：当 Kubelet 重启后，会把所有现有的 Pods 当作新 Pods 处理，这些 Pods 将进入准入控制过程中，可能会被拒绝。(==新创建一个pod也是这个函数==)
- **处理函数**：调用 `handler.HandlePodAdditions` 方法添加这些 Pods。这通常涉及到初始化 Pod 的各种资源，如网络、存储等，并将 Pod 状态同步到期望状态。

`kubetypes.UPDATE`

- **日志记录**：记录更新操作的详细信息。
- **处理函数**：调用 `handler.HandlePodUpdates` 方法更新 Pods。这通常发生在 Pods 的配置变更时，如环境变量、镜像版本更新等。

kubetypes.REMOVE

- **日志记录**：记录移除操作的详细信息。
- **处理函数**：调用 `handler.HandlePodRemoves` 方法删除 Pods。这通常是清理 Pods 相关的资源和状态。

`kubetypes.RECONCILE`

- **日志记录**：记录协调操作的详细信息，此日志级别较高（V(4)）。
- **处理函数**：调用 `handler.HandlePodReconcile` 方法协调 Pods。协调操作确保 Pods 的当前状态与集群的期望状态一致，用于处理可能由于外部因素导致的状态偏差。

`kubetypes.DELETE`

- **日志记录**：记录删除操作的详细信息。
- **操作说明**：删除操作被视为更新操作，因为 Kubernetes 支持优雅删除，即允许 Pods 在消失前完成某些清理任务。
- **处理函数**：调用 `handler.HandlePodUpdates` 处理优雅删除过程。

`kubetypes.SET`

- **日志记录**：记录不支持的操作类型，并提示错误。
- **操作说明**：当前 Kubelet 不支持快照更新类型，因此如果接收到此类操作会记录错误。

`default`

- **日志记录**：如果接收到无效的操作类型，记录错误。

`其他`

- **标记源就绪**：无论哪种类型的更新，完成后都会调用 `kl.sourcesReady.AddSource(u.Source)` 方法标记该源为已就绀。这是为了跟踪所有数据源的就绪状态，确保从所有配置源接收的数据都已处理。

```go
	case u, open := <-configCh:
		// Update from a config source; dispatch it to the right handler
		// callback.
		if !open {
			klog.ErrorS(nil, "Update channel is closed, exiting the sync loop")
			return false
		}

		switch u.Op {
		case kubetypes.ADD:
			klog.V(2).InfoS("SyncLoop ADD", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			// After restarting, kubelet will get all existing pods through
			// ADD as if they are new pods. These pods will then go through the
			// admission process and *may* be rejected. This can be resolved
			// once we have checkpointing.
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE:
			klog.V(2).InfoS("SyncLoop UPDATE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.REMOVE:
			klog.V(2).InfoS("SyncLoop REMOVE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			handler.HandlePodRemoves(u.Pods)
		case kubetypes.RECONCILE:
			klog.V(4).InfoS("SyncLoop RECONCILE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE:
			klog.V(2).InfoS("SyncLoop DELETE", "source", u.Source, "pods", klog.KObjSlice(u.Pods))
			// DELETE is treated as a UPDATE because of graceful deletion.
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.SET:
			// TODO: Do we want to support this?
			klog.ErrorS(nil, "Kubelet does not support snapshot update")
		default:
			klog.ErrorS(nil, "Invalid operation type received", "operation", u.Op)
		}

		kl.sourcesReady.AddSource(u.Source)
```

* `plegCh`
    * 检查事件是否值得同步,只要不是 pleg.ContainerRemoved都同步
    * handler.HandlePodSyncs同步pod状态，更新，重启等
    * 如果容器状态时pleg.ContainerDied，清理，删除容器

```go
	case e := <-plegCh:
		//isSyncPodWorthy检查事件是否值得同步,只要不是 pleg.ContainerRemoved都同步
		if isSyncPodWorthy(e) {
			// PLEG event for a pod; sync it.
			//通过 kl.podManager.GetPodByUID(e.ID) 尝试根据事件中的 Pod UID 获取 Pod 对象
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				klog.V(2).InfoS("SyncLoop (PLEG): event for pod", "pod", klog.KObj(pod), "event", e)
				//handler.HandlePodSyncs 方法来同步 Pod。这可能涉及更新 Pod 状态、重启容器等操作。
				handler.HandlePodSyncs([]*v1.Pod{pod})
			} else {
				// If the pod no longer exists, ignore the event.
				//失败记录日志
				klog.V(4).InfoS("SyncLoop (PLEG): pod does not exist, ignore irrelevant event", "event", e)
			}
		}
		//pleg.ContainerDied（表示容器已经死亡），从事件的 Data 字段获取容器 ID，并调用 kl.cleanUpContainersInPod 方法清理对应的容器。这个清理操作可能包括释放资源、删除容器等后续步骤。
		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
```

* `syncCh`

**触发机制**：来自一个定时器，通常设定为每秒触发一次。

**功能**：用于定期同步所有等待同步的 Pods。

**流程**：

- `getPodsToSync` 方法调用以获取需要同步的 Pod 列表。
- 如果没有 Pods 需要同步，退出当前迭代。
- 日志记录同步的 Pods 的数量和细节。
- 通过 `handler.HandlePodSyncs` 方法处理这些 Pods 的同步。

```go
	case <-syncCh:
		// Sync pods waiting for sync
		podsToSync := kl.getPodsToSync()
		if len(podsToSync) == 0 {
			break
		}
		klog.V(4).InfoS("SyncLoop (SYNC) pods", "total", len(podsToSync), "pods", klog.KObjSlice(podsToSync))
		handler.HandlePodSyncs(podsToSync)
```

* `livenessManager.Updates`

case update := <-kl.livenessManager.Updates():

- **来源**：来自存活性管理器，监控 Pod 的存活探针。
- **功能**：处理探针检测失败的事件。
- 流程：
    - 判断探测结果，如果失败，则调用 `handleProbeSync` 方法来处理不健康的状态，可能涉及重启容器等操作。

case update := <-kl.readinessManager.Updates():

- **来源**：来自就绪性管理器，监控 Pod 的就绪探针。
- **功能**：更新容器的就绪状态。
- 流程：
    - 判断探测结果，如果成功，则设置容器状态为就绪。
    - 根据探测结果，调用 `handleProbeSync` 方法更新 Pod 状态或处理相关事件。

case update := <-kl.startupManager.Updates():

- **来源**：来自启动管理器，监控 Pod 启动序列。
- **功能**：更新容器的启动状态。
- 流程：
    - 判断探测结果，如果成功，则标记容器为已启动。
    - 调用 `handleProbeSync` 方法处理启动状态相关的逻辑。

```go
	case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			handleProbeSync(kl, update, handler, "liveness", "unhealthy")
		}
	case update := <-kl.readinessManager.Updates():
		ready := update.Result == proberesults.Success
		kl.statusManager.SetContainerReadiness(update.PodUID, update.ContainerID, ready)

		status := ""
		if ready {
			status = "ready"
		}
		handleProbeSync(kl, update, handler, "readiness", status)
	case update := <-kl.startupManager.Updates():
		started := update.Result == proberesults.Success
		kl.statusManager.SetContainerStartup(update.PodUID, update.ContainerID, started)

		status := "unhealthy"
		if started {
			status = "started"
		}
		handleProbeSync(kl, update, handler, "startup", status)
```

* `housekeepingCh`

case <-housekeepingCh:

- **触发机制**：由定时器控制，通常设定为每几秒触发一次。
- **功能**：执行定期的系统维护任务，如清理无用的资源。
- 流程：
    - 检查所有数据源是否就绪，如果未就绪，跳过此次清理。
    - 如果就绪，记录开始时间，执行清理任务。
    - 计算执行时长，如果超过预期，记录警告日志。
    - 完成后记录维护结束和持续时间。

```go
	case <-housekeepingCh:
		if !kl.sourcesReady.AllReady() {
			// If the sources aren't ready or volume manager has not yet synced the states,
			// skip housekeeping, as we may accidentally delete pods from unready sources.
			klog.V(4).InfoS("SyncLoop (housekeeping, skipped): sources aren't ready yet")
		} else {
			start := time.Now()
			klog.V(4).InfoS("SyncLoop (housekeeping)")
			if err := handler.HandlePodCleanups(ctx); err != nil {
				klog.ErrorS(err, "Failed cleaning pods")
			}
			duration := time.Since(start)
			if duration > housekeepingWarningDuration {
				klog.ErrorS(fmt.Errorf("housekeeping took too long"), "Housekeeping took longer than expected", "expected", housekeepingWarningDuration, "actual", duration.Round(time.Millisecond))
			}
			klog.V(4).InfoS("SyncLoop (housekeeping) end", "duration", duration.Round(time.Millisecond))
		}
	}
```

