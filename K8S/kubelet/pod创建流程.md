### 1. 背景

本章节分析kubelet创建pod的详细过程。在 上一章节中了解到。kubelet通过configCh这个channel获取了来自apiserver/file/url的pod。并且调用了 HandlePodAdditions进行了处理。

### 2. HandlePodAdditions

**时间记录**：记录处理开始的时间，这对于性能监控和日志记录是有帮助的。

**排序操作**：对接收到的 Pod 列表按创建时间进行排序，确保按照创建顺序处理 Pod。这有助于维持处理的一致性和预测性。

**特性控制**：检查“就地 Pod 垂直伸缩”特性是否启用，这个特性允许 Kubelet 在不重启 Pod 的情况下动态调整其资源限制。如果启用了该功能，会通过锁定机制保护资源调整过程中的数据完整性。

**Pod 管理**：

- **Pod 添加**：将新的或更新的 Pod 添加到内部管理系统中。
- **镜像 Pod 处理**：对于静态 Pod，Kubelet 会创建一个镜像 Pod 来在 Kubernetes 系统中表示该 Pod。该函数检查 Pod 是否为镜像 Pod，并相应处理。
- **Pod 终止检查**：在执行任何资源分配或状态更新之前，检查 Pod 是否已被标记为终止，避免对即将删除的 Pod 进行处理。

**资源适配性检查**：

- **资源分配更新**：如果启用了垂直伸缩，会根据当前的资源使用情况更新 Pod 的资源分配。
- **准入控制**：通过 `canAdmitPod` 函数检查 Pod 是否可以在当前节点上运行，根据资源限制、安全策略和调度约束进行决定。如果 Pod 不符合条件，则会被拒绝。

**资源分配记录**：对于通过准入检查的新 Pod，记录其资源分配状态，这对于系统的恢复和资源跟踪非常重要。

**并发处理**：使用 `podWorkers` 来并发处理 Pod 的创建和更新任务。这包括调用容器运行时来启动或更新容器，同步 Pod 的状态，并处理与 Pod 生命周期相关的事件。

```go
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
	//这行代码获取当前的时间，用于记录处理 Pod 添加操作的开始时间
	start := kl.clock.Now()
	//对传入的 pods 数组进行排序，基于 Pod 的创建时间，确保处理顺序是按照 Pod 创建的时间顺序进行
	sort.Sort(sliceutils.PodsByCreationTime(pods))
	//如果启用了“就地 Pod 垂直伸缩”功能，则通过锁定和解锁 podResizeMutex 来保证在调整 Pod 资源时的线程安全性
	if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {
		kl.podResizeMutex.Lock()
		defer kl.podResizeMutex.Unlock()
	}
	//遍历每个 Pod 来进行处理
	for _, pod := range pods {
		//从 podManager 获取当前 Kubelet 管理的所有 Pod 的列表
		existingPods := kl.podManager.GetPods()
		// Always add the pod to the pod manager. Kubelet relies on the pod
		// manager as the source of truth for the desired state. If a pod does
		// not exist in the pod manager, it means that it has been deleted in
		// the apiserver and no action (other than cleanup) is required.
		//将新的 Pod 添加到 Kubelet 的内部 Pod 管理器中
		kl.podManager.AddPod(pod)
		//检查当前 Pod 是否是一个镜像 Pod（由 Kubelet 自动创建的，用于表示静态 Pod 的状态）。
		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		//如果是并且pod存在就是！=nil就更新这个pod
		if wasMirror {
			//pod不存在跳出循环
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
			kl.podWorkers.UpdatePod(UpdatePodOptions{
				Pod:        pod,
				MirrorPod:  mirrorPod,
				UpdateType: kubetypes.SyncPodUpdate,
				StartTime:  start,
			})
			continue
		}

		// Only go through the admission process if the pod is not requested
		// for termination by another part of the kubelet. If the pod is already
		// using resources (previously admitted), the pod worker is going to be
		// shutting it down. If the pod hasn't started yet, we know that when
		// the pod worker is invoked it will also avoid setting up the pod, so
		// we simply avoid doing any work.
		//检查这个 Pod 是否已经被请求终止处理。如果是，则不进行进一步处理。
		if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
			// We failed pods that we rejected, so activePods include all admitted
			// pods that are alive.
			//从现有的 Pod 列表中筛选出活跃的 Pod
			activePods := kl.filterOutInactivePods(existingPods)
			//检查是否启用了Pod 垂直扩展功能
			if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {
				// To handle kubelet restarts, test pod admissibility using AllocatedResources values
				// (for cpu & memory) from checkpoint store. If found, that is the source of truth.
				//对传入的 Pod 对象进行深拷贝，以避免在后续操作中对原对象产生副作用
				podCopy := pod.DeepCopy()
				//更新 Pod 的容器资源分配。这可能与就地 Pod 垂直扩展功能有关，确保 Pod 的资源分配是最新的
				kl.updateContainerResourceAllocation(podCopy)

				// Check if we can admit the pod; if not, reject it.
				//调用 canAdmitPod 函数检查是否可以准入这个 Pod。这涉及到资源的可用性、配额、调度等多种因素
				if ok, reason, message := kl.canAdmitPod(activePods, podCopy); !ok {
					//如果 Pod 不能被准入，则调用 rejectPod 函数拒绝这个 Pod，并给出拒绝的原因和消息
					kl.rejectPod(pod, reason, message)
					continue
				}
				// For new pod, checkpoint the resource values at which the Pod has been admitted
				//如果 Pod 是新的并且通过了准入检查，此代码行会尝试设置 Pod 的资源分配状态。SetPodAllocation 用于记录 Pod 被准入时的资源配置，有助于系统恢复和状态跟踪。
				if err := kl.statusManager.SetPodAllocation(podCopy); err != nil {
					//TODO(vinaykul,InPlacePodVerticalScaling): Can we recover from this in some way? Investigate
					klog.ErrorS(err, "SetPodAllocation failed", "pod", klog.KObj(pod))
				}
			} else {
				// Check if we can admit the pod; if not, reject it.
				//如果没有启动pod垂直扩展功能，调用 canAdmitPod 函数检查是否可以准入这个 Pod。
				if ok, reason, message := kl.canAdmitPod(activePods, pod); !ok {
					//如果 Pod 不能被准入，则拒绝这个 Pod
					kl.rejectPod(pod, reason, message)
					continue
				}
			}
		}
		//将 Pod 的处理任务提交给 podWorkers，这是一个并发处理机制，用于同步 Pod 状态，比如创建或更新容器
		kl.podWorkers.UpdatePod(UpdatePodOptions{
			Pod:        pod,
			MirrorPod:  mirrorPod,
			UpdateType: kubetypes.SyncPodCreate,
			StartTime:  start,
		})
	}
}
```

##### 2.1.1 podWorkers.UpdatePod

**podWorkers**结构体如下，核心：

* podUpdates:有变化的pod列表
* podSyncStatuses：记录每个 Pod 的同步状态的列表，包括 syncing, terminating, terminated, 和 evicted 状态。
* 存储所有 Pod 的容器状态信息的缓存

```go
type podWorkers struct {
    podLock: sync.Mutex
      - 保护 podWorkers 结构中所有字段的互斥锁，确保并发操作的线程安全。

    podsSynced: bool
      - 表示所有 Pod 的工作协程是否至少同步过一次，确保所有工作 Pod 都已通过 UpdatePod() 启动。

    podUpdates: map[types.UID]chan struct{}
      - 追踪每个 Pod 的 goroutine 更新通道。发送消息至此通道将触发处理对应 Pod 的 pendingUpdate。

    podSyncStatuses: map[types.UID]*podSyncStatus
      - 记录每个 Pod 的同步状态（如 syncing, terminating, terminated, evicted）。

    startedStaticPodsByFullname: map[string]types.UID
      - 追踪已经启动的静态 Pod 的 UID，通过 Pod 的完整名称映射。

    waitingToStartStaticPodsByFullname: map[string][]types.UID
      - 追踪等待启动的静态 Pod 的 UID 列表，通过 Pod 的完整名称映射。

    workQueue: queue.WorkQueue
      - 工作队列，用于管理和调度待处理任务。

    podSyncer: podSyncer
      - 定义如何同步 Pod 的期望状态的函数，需要确保线程安全。

    workerChannelFn: func(uid types.UID, in chan struct{}) (out <-chan struct{})
      - 用于测试的函数，允许在单元测试中施加通道通信延迟。

    recorder: record.EventRecorder
      - 用于记录 Kubernetes 事件的记录器。

    backOffPeriod: time.Duration
      - 当发生同步错误时的退避时间，用于控制重试逻辑。

    resyncInterval: time.Duration
      - 下一次同步前的等待时间，用于控制同步频率。

    podCache: kubecontainer.Cache
      - 存储所有 Pod 的容器状态信息的缓存，用于快速检索 Pod 状态。

    clock: clock.PassiveClock
      - 主要用于测试，以模拟时间相关操作。
}
```

UpdatePod的核心逻辑是：

`最重要就是检查podUpdates这个通道，里面有没有新传入的pod，没有的话就为这个podUpdates新建一个消息为1的通道，然后加入podUpdates，启动携程，podUpdates就是pod.chan的map，每个pod的更新，删除，添加等事件都在map[types.UID]podUpdates种`

1. ### . 初始化和环境设置

    - **确认 Pod 类型**：区分是完整的 Pod 还是仅有运行时状态的 Pod（如孤立 Pod）。这是为了确定只有运行时信息的 Pod 只能执行终止操作。
    - **锁定操作**：通过互斥锁保证在修改 podWorkers 内部状态时的线程安全。

    ### 2. 状态检查和首次同步

    - **检查 Pod 状态是否已知**：如果是首次同步该 Pod，会初始化其状态。这是因为首次处理一个 Pod 时需要建立基本的跟踪信息。
    - **处理终止状态的 Pod**：如果 Pod 已经处于终止状态（无论是正常结束还是失败），检查其在 Pod 缓存中的状态，以确保在 Kubelet 重启后，这些 Pod 不会被错误地视为活跃 Pod。

    ### 3. 更新逻辑

    - **处理运行时 Pod**：如果是由运行时状态触发的更新（没有完整的 Pod 配置），则尝试使用之前的或活跃的更新作为当前操作的基础。这确保了即使在缺少完整配置的情况下，也能正确处理 Pod 的生命周期事件。
    - **决定 Pod 操作**：基于 Pod 的当前状态和请求类型（创建、更新、终止），确定是设置新的 Pod、终止现有的 Pod 还是忽略操作。

    ### 4. 终止和重启处理

    - **终止逻辑**：如果检测到 Pod 需要终止（例如，有删除时间戳、状态为失败或成功），则设置相应的终止标记和时间。
    - **重启处理**：对于正在终止但请求重启的 Pod，设置重启请求标记，这通常发生在静态 Pod 或由于某些稀有条件导致的同一 UID 的重复创建。

    ### 5. 同步更新和通知

    - **启动 Pod 工作线程**：如果当前没有为该 Pod 运行的工作线程，就启动一个新的 goroutine。这保证了每个 Pod 的生命周期管理都是并行和独立的。
    - **传递更新通知**：通过 Pod 的更新通道发送通知，触发对应的工作线程处理最新的更新。使用非阻塞发送保证调用不会因为通道已满而延迟。
    - **处理紧急取消**：如果在处理过程中发现需要紧急停止当前操作（如宽限期被缩短或 Pod 被标记为终止），则调用取消函数中断当前同步。

```go
// UpdatePod carries a configuration change or termination state to a pod. A pod is either runnable,
// terminating, or terminated, and will transition to terminating if: deleted on the apiserver,
// discovered to have a terminal phase (Succeeded or Failed), or evicted by the kubelet.
func (p *podWorkers) UpdatePod(options UpdatePodOptions) {
	// Handle when the pod is an orphan (no config) and we only have runtime status by running only
	// the terminating part of the lifecycle. A running pod contains only a minimal set of information
	// about the pod
	//这段代码主要判断pod是运行时 Pod还是常规pod，运行时pod不是完整pod，一般只能终止
	//标识是否为运行时pod
	var isRuntimePod bool
	var uid types.UID
	var name, ns string
	// 判断pod是否存在，存在在进行下面的if逻辑，否则就是不存在，从options.Pod.Name提取UID.命名空间和名称
	if runningPod := options.RunningPod; runningPod != nil {
		// 如果没有Pod对象，只能进行终止操作，判断类型是不是SyncPodKill，如果不是代表是正常的pod，直接返回；
		//如果是SyncPodKill，提取runningPod.Name提取UID.命名空间和名称，isRuntimePod = true代表这是一个需要终止的pod
		if options.Pod == nil {
			// the sythetic pod created here is used only as a placeholder and not tracked
			if options.UpdateType != kubetypes.SyncPodKill {
				klog.InfoS("Pod update is ignored, runtime pods can only be killed", "pod", klog.KRef(runningPod.Namespace, runningPod.Name), "podUID", runningPod.ID, "updateType", options.UpdateType)
				return
			}
			uid, ns, name = runningPod.ID, runningPod.Namespace, runningPod.Name
			isRuntimePod = true
		} else {
			//如果有Pod对象，忽略该运行的pod，从 options.Pod 中提取 UID、命名空间和名称
			options.RunningPod = nil
			uid, ns, name = options.Pod.UID, options.Pod.Namespace, options.Pod.Name
			klog.InfoS("Pod update included RunningPod which is only valid when Pod is not specified", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
		}
	} else {
		uid, ns, name = options.Pod.UID, options.Pod.Namespace, options.Pod.Name
		//	//从options.Pod.Name提取UID.命名空间和名称
	}

	p.podLock.Lock()
	defer p.podLock.Unlock()

	// decide what to do with this pod - we are either setting it up, tearing it down, or ignoring it
	var firstTime bool
	now := p.clock.Now()
	//检查podSyncStatuses是否存在当前pod的状态
	status, ok := p.podSyncStatuses[uid]
	// 如果没有ok就为false，进行第一次同步
	if !ok {
		klog.V(4).InfoS("Pod is being synced for the first time", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
		//标记为第一次同步该Pod
		firstTime = true
		//初始化podSyncStatus对象
		status = &podSyncStatus{
			syncedAt: now,                                      //同步時間
			fullname: kubecontainer.BuildPodFullName(name, ns), //pod,名字和命名空间
		}
		// if this pod is being synced for the first time, we need to make sure it is an active pod
		//如果该 Pod 非空 (options.Pod != nil) 且状态为失败 (PodFailed) 或成功 (PodSucceeded)，则执行以下检查。
		if options.Pod != nil && (options.Pod.Status.Phase == v1.PodFailed || options.Pod.Status.Phase == v1.PodSucceeded) {
			// Check to see if the pod is not running and the pod is terminal; if this succeeds then record in the podWorker that it is terminated.
			// This is needed because after a kubelet restart, we need to ensure terminal pods will NOT be considered active in Pod Admission. See http://issues.k8s.io/105523
			// However, `filterOutInactivePods`, considers pods that are actively terminating as active. As a result, `IsPodKnownTerminated()` needs to return true and thus `terminatedAt` needs to be set.
			//尝试从 Pod 缓存中获取当前 Pod 的状态。
			if statusCache, err := p.podCache.Get(uid); err == nil {
				//因为当kubelet
				if isPodStatusCacheTerminal(statusCache) {
					//检查从缓存获取的状态是否表示 Pod 已终止。可能pod状态已经终止
					// At this point we know:
					// (1) The pod is terminal based on the config source.
					// (2) The pod is terminal based on the runtime cache.
					// This implies that this pod had already completed `SyncTerminatingPod` sometime in the past. The pod is likely being synced for the first time due to a kubelet restart.
					// These pods need to complete SyncTerminatedPod to ensure that all resources are cleaned and that the status manager makes the final status updates for the pod.
					// As a result, set finished: false, to ensure a Terminated event will be sent and `SyncTerminatedPod` will run.
					status = &podSyncStatus{
						terminatedAt:       now,
						terminatingAt:      now,
						syncedAt:           now,
						startedTerminating: true,
						finished:           false, //pod已经终止，但是还没执行清理操作，以便后续可以发送终止事件并执行清理操作。
						fullname:           kubecontainer.BuildPodFullName(name, ns),
					}
				}
			}
		}
		p.podSyncStatuses[uid] = status
		//将初始化或更新后的 Pod 状态保存回 podSyncStatuses 字典中，以便后续跟踪和管理。
	}

	// RunningPods represent an unknown pod execution and don't contain pod spec information
	// sufficient to perform any action other than termination. If we received a RunningPod
	// after a real pod has already been provided, use the most recent spec instead. Also,
	// once we observe a runtime pod we must drive it to completion, even if we weren't the
	// ones who started it.
	pod := options.Pod
	//处理完整pod的更新
	if isRuntimePod {
		//表示状态记录已经观察到了运行时的 Pod
		//options.RunningPod 代表仅有pod运行时的信息，options.Pod 代表完整的pod信息，进行状态更新时需要完整pod信息
		status.observedRuntime = true
		switch {
		//如果有待处理的更新，并且更新中包含了Pod对象；就options.Pod=更新中包含的Pod对象，同时清除 options.RunningPod
		case status.pendingUpdate != nil && status.pendingUpdate.Pod != nil:
			pod = status.pendingUpdate.Pod
			options.Pod = pod
			options.RunningPod = nil
			//如果有活跃更新，并且更新中包含 Pod 对像；就options.Pod=更新中包含的Pod对象，同时清除 options.RunningPod
		case status.activeUpdate != nil && status.activeUpdate.Pod != nil:
			pod = status.activeUpdate.Pod
			options.Pod = pod
			options.RunningPod = nil
			//如果没有匹配的更新，将继续使用从 RunningPod.ToAPIPod() 方法得到的 Pod 对象作为 pod 的值，但是 options.Pod 将保持为 nil。
		default:
			// we will continue to use RunningPod.ToAPIPod() as pod here, but
			// options.Pod will be nil and other methods must handle that appropriately.
			pod = options.RunningPod.ToAPIPod()
		}
	}

	// When we see a create update on an already terminating pod, that implies two pods with the same UID were created in
	// close temporal proximity (usually static pod but it's possible for an apiserver to extremely rarely do something
	// similar) - flag the sync status to indicate that after the pod terminates it should be reset to "not running" to
	// allow a subsequent add/update to start the pod worker again. This does not apply to the first time we see a pod,
	// such as when the kubelet restarts and we see already terminated pods for the first time.
	// 如果在Pod正在终止过程中收到创建更新，标记需要重新启动
	if !firstTime && status.IsTerminationRequested() {
		if options.UpdateType == kubetypes.SyncPodCreate {
			status.restartRequested = true
			klog.V(4).InfoS("Pod is terminating but has been requested to restart with same UID, will be reconciled later", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			return
		}
	}

	// once a pod is terminated by UID, it cannot reenter the pod worker (until the UID is purged by housekeeping)
	//// 如果Pod已经完成了处理，不再接受更新
	if status.IsFinished() {
		klog.V(4).InfoS("Pod is finished processing, no further updates", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
		return
	}

	// check for a transition to terminating
	// 检查Pod是否需要开始终止过程
	var becameTerminating bool
	//如果没有请求终止
	if !status.IsTerminationRequested() {
		switch {
		case isRuntimePod:
			//如果时容器运行时，终止pod
			klog.V(4).InfoS("Pod is orphaned and must be torn down", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			status.deleted = true
			status.terminatingAt = now
			becameTerminating = true
		case pod.DeletionTimestamp != nil:
			//如果Pod已经标记为删除，则终止pod
			klog.V(4).InfoS("Pod is marked for graceful deletion, begin teardown", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			status.deleted = true
			status.terminatingAt = now
			becameTerminating = true
		case pod.Status.Phase == v1.PodFailed, pod.Status.Phase == v1.PodSucceeded:
			//如果Pod已经完成，则终止pod
			klog.V(4).InfoS("Pod is in a terminal phase (success/failed), begin teardown", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			status.terminatingAt = now
			becameTerminating = true
		case options.UpdateType == kubetypes.SyncPodKill:
			//如果更新事件是SyncPodKill，并且设置了Evict，设置驱逐状态；否则pod正在被删除
			if options.KillPodOptions != nil && options.KillPodOptions.Evict {
				klog.V(4).InfoS("Pod is being evicted by the kubelet, begin teardown", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
				status.evicted = true
			} else {
				klog.V(4).InfoS("Pod is being removed by the kubelet, begin teardown", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			}
			//无论是否删除，都将 status.terminatingAt 设置为当前时间并标记 becameTerminating。
			status.terminatingAt = now
			becameTerminating = true
		}
	}

	// once a pod is terminating, all updates are kills and the grace period can only decrease
	//  更新终止期限的逻辑处理
	var wasGracePeriodShortened bool
	switch {
	// Pod 的状态已被标记为完全终止tatus.IsTerminated(),如果返回true，代表pod等待清理
	case status.IsTerminated():
		// A terminated pod may still be waiting for cleanup - if we receive a runtime pod kill request
		// due to housekeeping seeing an older cached version of the runtime pod simply ignore it until
		// after the pod worker completes.
		//如果收到的清理请求是基于过时缓存，则忽略它
		if isRuntimePod {
			klog.V(3).InfoS("Pod is waiting for termination, ignoring runtime-only kill until after pod worker is fully terminated", "pod", klog.KRef(ns, name), "podUID", uid, "updateType", options.UpdateType)
			return
		}
		//如果存在 KillPodOptions，如果存在CompletedCh通道，则关闭通道
		if options.KillPodOptions != nil {
			if ch := options.KillPodOptions.CompletedCh; ch != nil {
				close(ch)
			}
		}
		//代表 Pod 的终止处理已经完成
		options.KillPodOptions = nil
	//如果 Pod 正在请求终止（status.IsTerminationRequested() 返回 true）
	case status.IsTerminationRequested():
		if options.KillPodOptions == nil {
			//KillPodOptions为nil，则初始化一个新的实例
			options.KillPodOptions = &KillPodOptions{}
		}
		//如果 KillPodOptions 存在 CompletedCh，如果通道非空，则将此通道添加到 status.notifyPostTerminating 列表中。这个列表用于在 Pod 终止后发送通知给所有注册的监听者。
		if ch := options.KillPodOptions.CompletedCh; ch != nil {
			status.notifyPostTerminating = append(status.notifyPostTerminating, ch)
		}
		//如果 KillPodOptions 存在 PodStatusFunc，如果通道非空，则将此通道添加到 status.statusPostTerminating 列表中。这个列表保存在 Pod 终止过程中需要执行的所有状态更新函数。。
		if fn := options.KillPodOptions.PodStatusFunc; fn != nil {
			status.statusPostTerminating = append(status.statusPostTerminating, fn)
		}
		//函数计算 Pod 的有效宽限期
		gracePeriod, gracePeriodShortened := calculateEffectiveGracePeriod(status, pod, options.KillPodOptions)
		//标识宽限期是否在本次操作中被缩短。
		wasGracePeriodShortened = gracePeriodShortened
		status.gracePeriod = gracePeriod
		// always set the grace period for syncTerminatingPod so we don't have to recalculate,
		// will never be zero.
		//直接指定 Pod 终止时应使用的宽限期
		options.KillPodOptions.PodTerminationGracePeriodSecondsOverride = &gracePeriod

	default:
		// KillPodOptions is not valid for sync actions outside of the terminating phase
		//对于非终止阶段的 Pod 更新，如果 KillPodOptions 存在，则关闭通道
		if options.KillPodOptions != nil {
			if ch := options.KillPodOptions.CompletedCh; ch != nil {
				close(ch)
			}
			////代表 Pod 的终止处理已经完成
			options.KillPodOptions = nil
		}
	}

	// start the pod worker goroutine if it doesn't exist
	//检查是否已存在该Pod的更新通道
	podUpdates, exists := p.podUpdates[uid]
	//如果不存在，创建一个缓冲大小为1的通道。这样做是为了避免发送更新时阻塞UpdatePod方法。
	if !exists {
		// buffer the channel to avoid blocking this method
		podUpdates = make(chan struct{}, 1)
		p.podUpdates[uid] = podUpdates

		// 确保静态pod按照UpdatePod接收它们的顺序启动
		if kubetypes.IsStaticPod(pod) {
			p.waitingToStartStaticPodsByFullname[status.fullname] =
				append(p.waitingToStartStaticPodsByFullname[status.fullname], uid)
		}

		// allow testing of delays in the pod update channel
		//允许测试pod更新通道中的延迟
		var outCh <-chan struct{}
		if p.workerChannelFn != nil {
			outCh = p.workerChannelFn(uid, podUpdates)
		} else {
			outCh = podUpdates
		}

		// spawn a pod worker
		//为每个Pod启动一个goroutine来处理更新。podWorkerLoop是处理Pod生命周期事件的循环
		go func() {
			// TODO: this should be a wait.Until with backoff to handle panics, and
			// accept a context for shutdown
			defer runtime.HandleCrash()
			defer klog.V(3).InfoS("Pod worker has stopped", "podUID", uid)
			p.podWorkerLoop(uid, outCh)
		}()
	}

	// measure the maximum latency between a call to UpdatePod and when the pod worker reacts to it
	// by preserving the oldest StartTime
	//测量UpdatePod调用到工作协程响应的最大延迟
	if status.pendingUpdate != nil && !status.pendingUpdate.StartTime.IsZero() && status.pendingUpdate.StartTime.Before(options.StartTime) {
		options.StartTime = status.pendingUpdate.StartTime
	}

	// notify the pod worker there is a pending update
	// 通知工作协程有一个待处理的更新
	status.pendingUpdate = &options
	status.working = true
	klog.V(4).InfoS("Notifying pod of pending update", "pod", klog.KRef(ns, name), "podUID", uid, "workType", status.WorkType())
	select {
	case podUpdates <- struct{}{}:
	default:
	}
	//如果Pod开始终止过程或宽限期被缩短，并且存在取消函数cancelFn，则调用该函数取消当前同步。这通常发生在Pod状态有重大变化时，需要重新评估Pod的处理逻辑。
	if (becameTerminating || wasGracePeriodShortened) && status.cancelFn != nil {
		klog.V(3).InfoS("Cancelling current pod sync", "pod", klog.KRef(ns, name), "podUID", uid, "workType", status.WorkType())
		status.cancelFn()
		return
	}
}
```

#### 2.1.2  podWorkerLoop

**监听更新**: 通过 `for range podUpdates` 无限循环监听来自 `podUpdates` 通道的信号。每次信号表示 Pod 有更新需要处理。

**初始化同步**: 使用 `p.startPodSync` 函数初始化同步流程，该函数检查 Pod 是否有待处理的更新，以及 Pod 是否处于可以启动的状态。

**状态和错误处理**:

- **获取状态**: 根据 Pod 的更新类型，可能需要从 Pod 生命周期事件生成器（PLEG）获取最新的 Pod 状态。
- **处理终止**: 如果 Pod 正在终止，会直接调用适当的终止处理函数，如 `SyncTerminatingRuntimePod` 或 `SyncTerminatingPod`。
- **常规同步**: 对于非终止的更新，调用 `SyncPod` 函数，该函数负责处理 Pod 的创建、更新等。

**错误和状态转换处理**:

- 检查同步操作中是否出现错误，如上下文取消或同步失败，然后根据错误类型进行相应处理。
- 根据 Pod 的终止状态决定是否关闭工作器或标记状态转换。

**完成处理**:

- 使用 `completeWork` 函数处理同步后的清理工作，如重试机制或终止后续操作。
- 记录操作持续时间，用于性能监控。

**性能监控**: 使用 Prometheus 监控工具记录从同步开始到结束的时间，帮助监测和优化 Pod 同步性能。

```go
func (p *podWorkers) podWorkerLoop(podUID types.UID, podUpdates <-chan struct{}) {
	//用来记录最后一次同步操作的时间
	var lastSyncTime time.Time
	//无线循环监听podUpdates，每当有新的信号（空结构体）发送到这个通道时，循环就会进行一次迭代
	for range podUpdates {
		ctx, update, canStart, canEverStart, ok := p.startPodSync(podUID)
		// If we had no update waiting, it means someone initialized the channel without filling out pendingUpdate.
		//ok代表获取到更新操作，如果没有跳出循环
		if !ok {
			continue
		}
		// If the pod was terminated prior to the pod being allowed to start, we exit the loop.
		//如果 canEverStart 为 false，表示此 Pod 永远不应该开始同步
		if !canEverStart {
			return
		}
		// If the pod is not yet ready to start, continue and wait for more updates.
		//如果当前不应该开始同步（canStart 为 false），则跳过当前循环迭代，等待下一次更新
		if !canStart {
			continue
		}
		//从更新事件种获取pod UID和信息
		podUID, podRef := podUIDAndRefForUpdate(update.Options)

		klog.V(4).InfoS("Processing pod event", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)
		//标记 Pod 是否进入终止状态
		var isTerminal bool
		err := func() error {
			// The worker is responsible for ensuring the sync method sees the appropriate
			// status updates on resyncs (the result of the last sync), transitions to
			// terminating (no wait), or on terminated (whatever the most recent state is).
			// Only syncing and terminating can generate pod status changes, while terminated
			// pods ensure the most recent status makes it to the api server.
			var status *kubecontainer.PodStatus
			var err error
			switch {
			//如果状态是正在运行的，无需处理，跳过更新
			case update.Options.RunningPod != nil:
				// when we receive a running pod, we don't need status at all because we are
				// guaranteed to be terminating and we skip updates to the pod
			default:
				// wait until we see the next refresh from the PLEG via the cache (max 2s)
				// TODO: this adds ~1s of latency on all transitions from sync to terminating
				//  to terminated, and on all termination retries (including evictions). We should
				//  improve latency by making the pleg continuous and by allowing pod status
				//  changes to be refreshed when key events happen (killPod, sync->terminating).
				//  Improving this latency also reduces the possibility that a terminated
				//  container's status is garbage collected before we have a chance to update the
				//  API server (thus losing the exit code).
				// 从 Pod 缓存中获取状态，该缓存由 PLEG（Pod Lifecycle Event Generator）维护，它定期刷新 Pod 状态。
				//  这里等待直到从 PLEG 通过缓存获取的信息比 lastSyncTime 更新，以确保处理的是最新状态。
				status, err = p.podCache.GetNewerThan(update.Options.Pod.UID, lastSyncTime)

				if err != nil {
					// This is the legacy event thrown by manage pod loop all other events are now dispatched
					// from syncPodFn
					p.recorder.Eventf(update.Options.Pod, v1.EventTypeWarning, events.FailedSync, "error determining status: %v", err)
					return err
				}
			}

			// Take the appropriate action (illegal phases are prevented by UpdatePod)
			switch {
			// 如果工作类型是处理已终止的 Pod，调用 SyncTerminatedPod 方法，它处理清理和资源释放。
			case update.WorkType == TerminatedPod:
				err = p.podSyncer.SyncTerminatedPod(ctx, update.Options.Pod, status)
			// 如果工作类型是正在终止，处理 Pod 的终止逻辑，可能涉及优雅停机。
			case update.WorkType == TerminatingPod:
				var gracePeriod *int64
				if opt := update.Options.KillPodOptions; opt != nil {
					gracePeriod = opt.PodTerminationGracePeriodSecondsOverride
				}
				podStatusFn := p.acknowledgeTerminating(podUID)

				// if we only have a running pod, terminate it directly
				// 如果只有运行时 Pod，直接终止它，不需要详细的 Pod 状态。
				if update.Options.RunningPod != nil {
					err = p.podSyncer.SyncTerminatingRuntimePod(ctx, update.Options.RunningPod)
				} else {
					// 否则，处理常规的终止逻辑。
					err = p.podSyncer.SyncTerminatingPod(ctx, update.Options.Pod, status, gracePeriod, podStatusFn)
				}
			// 处理常规的 Pod 同步任务，该方法根据 Pod 的当前状态和期望状态进行同步操作，可以处理创建、更新等。
			default:
				isTerminal, err = p.podSyncer.SyncPod(ctx, update.Options.UpdateType, update.Options.Pod, update.Options.MirrorPod, status)
			}

			lastSyncTime = p.clock.Now()
			return err
		}()

		var phaseTransition bool
		switch {
		//如果上下文被取消，可能是因为外部中断如操作系统信号。
		case err == context.Canceled:
			// when the context is cancelled we expect an update to already be queued
			klog.V(2).InfoS("Sync exited with context cancellation error", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)
		//// 如果在同步过程中发生错误，记录错误并跳过此次同步。
		case err != nil:
			// we will queue a retry
			klog.ErrorS(err, "Error syncing pod, skipping", "pod", podRef, "podUID", podUID)
		//// 如果工作类型为 TerminatedPod，调用 completeTerminated 完成终止处理，并记录性能指标。
		case update.WorkType == TerminatedPod:
			// we can shut down the worker
			p.completeTerminated(podUID)
			if start := update.Options.StartTime; !start.IsZero() {
				metrics.PodWorkerDuration.WithLabelValues("terminated").Observe(metrics.SinceInSeconds(start))
			}
			klog.V(4).InfoS("Processing pod event done", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)
			return
		// 如果工作类型为 TerminatingPod，根据是否存在 RunningPod 来处理终止逻辑。
		case update.WorkType == TerminatingPod:
			// pods that don't exist in config don't need to be terminated, other loops will clean them up
			if update.Options.RunningPod != nil {
				p.completeTerminatingRuntimePod(podUID)
				if start := update.Options.StartTime; !start.IsZero() {
					metrics.PodWorkerDuration.WithLabelValues(update.Options.UpdateType.String()).Observe(metrics.SinceInSeconds(start))
				}
				klog.V(4).InfoS("Processing pod event done", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)
				return
			}
			// otherwise we move to the terminating phase
			p.completeTerminating(podUID)
			phaseTransition = true
		//如果 Pod 状态为终端，调用 completeSync 完成同步并设置状态转换。
		case isTerminal:
			// if syncPod indicated we are now terminal, set the appropriate pod status to move to terminating
			klog.V(4).InfoS("Pod is terminal", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)
			p.completeSync(podUID)
			phaseTransition = true
		}

		// queue a retry if necessary, then put the next event in the channel if any
		//在每种情况处理完毕后，调用 completeWork 来处理后续逻辑（如重试），并记录事件完成的度量信息。
		p.completeWork(podUID, phaseTransition, err)
		if start := update.Options.StartTime; !start.IsZero() {
			metrics.PodWorkerDuration.WithLabelValues(update.Options.UpdateType.String()).Observe(metrics.SinceInSeconds(start))
		}
		klog.V(4).InfoS("Processing pod event done", "pod", podRef, "podUID", podUID, "updateType", update.WorkType)
	}
}
```

#### 2.1.3 SyncPod

**跟踪和记录**：函数开始时，会启动一个跟踪段（trace span），这有助于监控函数的执行并记录重要的信息，如 Pod 的 UID、名称和更新类型等。

**延迟测量**：如果这是第一次同步这个 Pod，会记录从 Kubelet 首次看到这个 Pod 到开始处理它的时间，这有助于了解 Pod 启动的延迟。

**生成和更新 Pod 状态**：基于 Pod 的当前运行信息和 Kubelet 的状态管理器，生成 Pod 的最终 API 状态，并对比当前状态，进行必要的更新。

**判断 Pod 运行条件**：

- 如果 Pod 由于资源限制或策略限制不能运行，会更新 Pod 的状态为 Pending，并尝试停止正在运行的容器。
- 如果 Pod 需要网络但网络插件未就绪，也会阻止 Pod 运行。

**资源和安全管理**：

- 确保 Kubelet 知道 Pod 使用的所有密钥和配置映射。
- 为 Pod 创建或更新资源控制组（cgroups），这对于资源管理非常关键。

**容器运行时同步**：调用容器运行时的 `SyncPod` 方法来实际启动、停止或重启容器，以确保 Pod 的容器状态符合最新的规范。

**健康检查**：为 Pod 添加健康检查探针，如活跃度（liveness）和就绪度（readiness）探针，以监控容器的健康状态。

**处理动态资源调整**：如果启用了 Pod 的资源垂直扩缩容特性，并且检测到相关的资源调整需求，会处理这些调整，确保 Pod 的资源配置符合最新需求。

**错误处理和状态更新**：

- 如果在同步过程中发生错误，会根据错误类型判断是否需要重试。
- 更新内部的原因缓存，记录为何 Pod 无法进入期望的状态。

**清理和关闭**：在函数结束时，关闭跟踪段，并返回最终的同步结果，表示是否成功同步 Pod。

`对于创建pod来说`：、

**管理资源和权限**：

- 确认并注册 Pod 使用的密钥和配置映射，确保容器可以访问必要的配置和密钥信息。
- 为 Pod 创建和配置资源控制组（cgroups），这是资源管理的关键步骤，确保 Pod 按照预定的资源限制运行。

**容器运行时准备**：调用容器运行时的 `SyncPod` 方法，负责实际的容器创建和启动。这包括根据 Pod 的定义启动新容器、设置网络和执行初始命令等。

**设置健康检查探针**：为 Pod 配置必要的健康检查探针，如活跃度和就绪度探针，这些探针定期检查容器的健康状态，以确保服务的可靠性。

**创建和管理数据目录**：

- 为 Pod 创建必要的数据目录，包括日志和持久化数据存储目录。这些目录对于容器的日常运行和数据持久化至关重要。
- 确保所有定义在 Pod 规范中的卷被正确挂载到指定的目录，这可能包括外部存储卷的挂载和配置。

```go
func (kl *Kubelet) SyncPod(ctx context.Context, updateType kubetypes.SyncPodType, pod, mirrorPod *v1.Pod, podStatus *kubecontainer.PodStatus) (isTerminal bool, err error) {
	//使用 OpenTelemetry 库开始一个新的跟踪段（span），用于监控和记录 syncPod 函数的执行。跟踪包括 Pod 的 UID、名称、命名空间以及更新类型等属性，以便于追踪问题和性能分析。
	ctx, otelSpan := kl.tracer.Start(ctx, "syncPod", trace.WithAttributes(
		semconv.K8SPodUIDKey.String(string(pod.UID)),
		attribute.String("k8s.pod", klog.KObj(pod).String()),
		semconv.K8SPodNameKey.String(pod.Name),
		attribute.String("k8s.pod.update_type", updateType.String()),
		semconv.K8SNamespaceNameKey.String(pod.Namespace),
	))
	klog.V(4).InfoS("SyncPod enter", "pod", klog.KObj(pod), "podUID", pod.UID)
	defer func() {
		klog.V(4).InfoS("SyncPod exit", "pod", klog.KObj(pod), "podUID", pod.UID, "isTerminal", isTerminal)
		otelSpan.End()
	}()

	// Latency measurements for the main workflow are relative to the
	// first time the pod was seen by kubelet.
	//Pod 的注解中是否记录了首次被 Kubelet 见到的时间，如果有，将其转换为时间格式。这个时间用于后续计算 Pod 从创建到开始处理的延迟
	var firstSeenTime time.Time
	if firstSeenTimeStr, ok := pod.Annotations[kubetypes.ConfigFirstSeenAnnotationKey]; ok {
		firstSeenTime = kubetypes.ConvertToTimestamp(firstSeenTimeStr).Get()
	}

	// Record pod worker start latency if being created
	// TODO: make pod workers record their own latencies
	//如果更新类型是创建 Pod（SyncPodCreate），则测量并记录从 Kubelet 首次看到 Pod 到开始同步这个 Pod 的时间。如果首次见到时间没有记录，则记录一个警告日志
	if updateType == kubetypes.SyncPodCreate {
		if !firstSeenTime.IsZero() {
			// This is the first time we are syncing the pod. Record the latency
			// since kubelet first saw the pod if firstSeenTime is set.
			metrics.PodWorkerStartDuration.Observe(metrics.SinceInSeconds(firstSeenTime))
		} else {
			klog.V(3).InfoS("First seen time not recorded for pod",
				"podUID", pod.UID,
				"pod", klog.KObj(pod))
		}
	}

	// Generate final API pod status with pod and status manager status
	//调用 generateAPIPodStatus 方法生成 Pod 的最终 API 状态。这个状态包括 Pod 的所有容器的状态，以及 Pod 自身的状态。
	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus, false)
	// The pod IP may be changed in generateAPIPodStatus if the pod is using host network. (See #24576)
	// TODO(random-liu): After writing pod spec into container labels, check whether pod is using host network, and
	// set pod IP to hostIP directly in runtime.GetPodStatus
	//从podapi提取Ip地址到podStatus
	podStatus.IPs = make([]string, 0, len(apiPodStatus.PodIPs))
	for _, ipInfo := range apiPodStatus.PodIPs {
		podStatus.IPs = append(podStatus.IPs, ipInfo.IP)
	}
	//如果是旧版本使用apiPodStatus.PodIP，而不是apiPodStatus.PodIPs字段
	if len(podStatus.IPs) == 0 && len(apiPodStatus.PodIP) > 0 {
		podStatus.IPs = []string{apiPodStatus.PodIP}
	}

	// If the pod is terminal, we don't need to continue to setup the pod
	//如果 Pod 的阶段是成功完成（PodSucceeded）或失败（PodFailed），则更新 Pod 状态，并标记为终端状态，返回终端状态和无错误
	if apiPodStatus.Phase == v1.PodSucceeded || apiPodStatus.Phase == v1.PodFailed {
		kl.statusManager.SetPodStatus(pod, apiPodStatus)
		//isTerminal代表pod生命周期已经终止，不会在更新了，可以清理资源
		isTerminal = true
		return isTerminal, nil
	}

	// If the pod should not be running, we request the pod's containers be stopped. This is not the same
	// as termination (we want to stop the pod, but potentially restart it later if soft admission allows
	// it later). Set the status and phase appropriately
	// 检查Pod是否允许运行
	runnable := kl.canRunPod(pod)
	//如果 runnable.Admit 为 false，表示 Pod 不允许运行
	if !runnable.Admit {
		// Pod is not runnable; and update the Pod and Container statuses to why.
		//如果 Pod 的当前阶段不是 Failed 或 Succeeded，则将其阶段更新为 Pending
		if apiPodStatus.Phase != v1.PodFailed && apiPodStatus.Phase != v1.PodSucceeded {
			apiPodStatus.Phase = v1.PodPending
		}
		//像runnable提供具体原因和信息也会被记录
		apiPodStatus.Reason = runnable.Reason
		apiPodStatus.Message = runnable.Message
		// Waiting containers are not creating.
		//2个for循环便利初始化容器设备状态和常规容器状态，如果State.Waiting字段不为空设置为Blocked
		const waitingReason = "Blocked"
		for _, cs := range apiPodStatus.InitContainerStatuses {
			if cs.State.Waiting != nil {
				cs.State.Waiting.Reason = waitingReason
			}
		}
		for _, cs := range apiPodStatus.ContainerStatuses {
			if cs.State.Waiting != nil {
				cs.State.Waiting.Reason = waitingReason
			}
		}
	}

	// Record the time it takes for the pod to become running
	// since kubelet first saw the pod if firstSeenTime is set.
	//记录从 kubelet 第一次看到 Pod 到 Pod 变成 Running 的时间
	existingStatus, ok := kl.statusManager.GetPodStatus(pod.UID)
	if !ok || existingStatus.Phase == v1.PodPending && apiPodStatus.Phase == v1.PodRunning &&
		!firstSeenTime.IsZero() {
		metrics.PodStartDuration.Observe(metrics.SinceInSeconds(firstSeenTime))
	}
	// 更新Pod状态
	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	// Pods that are not runnable must be stopped - return a typed error to the pod worker
	// 检查Pod是否允许运行
	if !runnable.Admit {
		klog.V(2).InfoS("Pod is not runnable and must have running containers stopped", "pod", klog.KObj(pod), "podUID", pod.UID, "message", runnable.Message)
		var syncErr error
		//将podStatus转换成runtime格式，为停止容器做准备
		p := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
		//kill掉pod
		if err := kl.killPod(ctx, pod, p, nil); err != nil {
			//如果在尝试停止Pod的过程中遇到错误，并且这个错误不是因为操作被中断（如超时或取消操作）
			if !wait.Interrupted(err) {
				//记录警告事件
				kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToKillPod, "error killing pod: %v", err)
				syncErr = fmt.Errorf("error killing pod: %w", err)
				//utilruntime.HandleError处理错误
				utilruntime.HandleError(syncErr)
			}
		} else {
			//代表容器正常停止
			// There was no error killing the pod, but the pod cannot be run.
			// Return an error to signal that the sync loop should back off.
			syncErr = fmt.Errorf("pod cannot be run: %v", runnable.Message)
		}
		return false, syncErr
	}

	// If the network plugin is not ready, only start the pod if it uses the host network
	// 检查网络插件是否准备好，如果未准备好且Pod不使用宿主网络，则返回错误
	if err := kl.runtimeState.networkErrors(); err != nil && !kubecontainer.IsHostNetworkPod(pod) {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.NetworkNotReady, "%s: %v", NetworkNotReadyErrorMsg, err)
		return false, fmt.Errorf("%s: %v", NetworkNotReadyErrorMsg, err)
	}

	// ensure the kubelet knows about referenced secrets or configmaps used by the pod
	// 确保Kubelet知道Pod使用的密钥和配置映射
	if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
		if kl.secretManager != nil {
			kl.secretManager.RegisterPod(pod)
		}
		if kl.configMapManager != nil {
			kl.configMapManager.RegisterPod(pod)
		}
	}

	// Create Cgroups for the pod and apply resource parameters
	// to them if cgroups-per-qos flag is enabled.
	// 码初始化了一个新的Pod容器管理器实例，用来管理Pod的cgroups（资源控制组）
	pcm := kl.containerManager.NewPodContainerManager()
	// If pod has already been terminated then we need not create
	// or update the pod's cgroup
	// TODO: once context cancellation is added this check can be removed
	//检查是否已经为该Pod请求了终止。如果没有，它将继续进行Pod的设置或更新。
	if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
		// When the kubelet is restarted with the cgroups-per-qos
		// flag enabled, all the pod's running containers
		// should be killed intermittently and brought back up
		// under the qos cgroup hierarchy.
		// Check if this is the pod's first sync
		//检查当前是否为Pod的首次同步。通过检查所有容器的状态来确定，如果任一容器已经在运行状态，那么firstSync变量将被设置为false
		firstSync := true
		for _, containerStatus := range apiPodStatus.ContainerStatuses {
			if containerStatus.State.Running != nil {
				firstSync = false
				break
			}
		}
		// Don't kill containers in pod if pod's cgroups already
		// exists or the pod is running for the first time
		//检查当前是否为Pod的首次同步。通过检查所有容器的状态来确定，如果任一容器已经在运行状态，那么firstSync变量将被设置为false
		podKilled := false
		if !pcm.Exists(pod) && !firstSync {
			p := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
			if err := kl.killPod(ctx, pod, p, nil); err == nil {
				if wait.Interrupted(err) {
					return false, err
				}
				podKilled = true
			} else {
				klog.ErrorS(err, "KillPod failed", "pod", klog.KObj(pod), "podStatus", podStatus)
			}
		}
		// Create and Update pod's Cgroups
		// Don't create cgroups for run once pod if it was killed above
		// The current policy is not to restart the run once pods when
		// the kubelet is restarted with the new flag as run once pods are
		// expected to run only once and if the kubelet is restarted then
		// they are not expected to run again.
		// We don't create and apply updates to cgroup if its a run once pod and was killed above
		//这部分代码用于创建或更新Pod的cgroups。如果Pod未被杀死（或者被杀死但重启策略不是Never），且Pod的cgroups还不存在，那么会尝试创建或更新这些cgroups。如果在更新过程中遇到错误，会记录相应的事件并返回错误。
		if !(podKilled && pod.Spec.RestartPolicy == v1.RestartPolicyNever) {
			if !pcm.Exists(pod) {
				if err := kl.containerManager.UpdateQOSCgroups(); err != nil {
					klog.V(2).InfoS("Failed to update QoS cgroups while syncing pod", "pod", klog.KObj(pod), "err", err)
				}
				if err := pcm.EnsureExists(pod); err != nil {
					kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToCreatePodContainer, "unable to ensure pod container exists: %v", err)
					return false, fmt.Errorf("failed to ensure that the pod: %v cgroups exist and are correctly applied: %v", pod.UID, err)
				}
			}
		}
	}

	// Create Mirror Pod for Static Pod if it doesn't already exist
	//判断是否为静态pod
	if kubetypes.IsStaticPod(pod) {
		deleted := false
		//如果存在镜像Pod且该Pod已被标记为删除或不再反映静态Pod的状态，则删除该镜像Pod
		if mirrorPod != nil {
			if mirrorPod.DeletionTimestamp != nil || !kubepod.IsMirrorPodOf(mirrorPod, pod) {
				// The mirror pod is semantically different from the static pod. Remove
				// it. The mirror pod will get recreated later.
				klog.InfoS("Trying to delete pod", "pod", klog.KObj(pod), "podUID", mirrorPod.ObjectMeta.UID)
				podFullName := kubecontainer.GetPodFullName(pod)
				var err error
				deleted, err = kl.mirrorPodClient.DeleteMirrorPod(podFullName, &mirrorPod.ObjectMeta.UID)
				if deleted {
					klog.InfoS("Deleted mirror pod because it is outdated", "pod", klog.KObj(mirrorPod))
				} else if err != nil {
					klog.ErrorS(err, "Failed deleting mirror pod", "pod", klog.KObj(mirrorPod))
				}
			}
		}
		//如果没有镜像Pod或旧的镜像Pod已被删除，尝试创建一个新的镜像Pod以在API server中反映静态Pod的存在和状态。
		if mirrorPod == nil || deleted {
			node, err := kl.GetNode()
			if err != nil {
				klog.V(4).ErrorS(err, "No need to create a mirror pod, since failed to get node info from the cluster", "node", klog.KRef("", string(kl.nodeName)))
			} else if node.DeletionTimestamp != nil {
				klog.V(4).InfoS("No need to create a mirror pod, since node has been removed from the cluster", "node", klog.KRef("", string(kl.nodeName)))
			} else {
				klog.V(4).InfoS("Creating a mirror pod for static pod", "pod", klog.KObj(pod))
				if err := kl.mirrorPodClient.CreateMirrorPod(pod); err != nil {
					klog.ErrorS(err, "Failed creating a mirror pod for", "pod", klog.KObj(pod))
				}
			}
		}
	}

	// Make data directories for the pod
	//为Pod创建必要的数据目录，如日志和容器数据目录。如果创建失败，记录警告事件并返回错误。
	if err := kl.makePodDataDirs(pod); err != nil {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToMakePodDataDirectories, "error making pod data directories: %v", err)
		klog.ErrorS(err, "Unable to make pod data directories for pod", "pod", klog.KObj(pod))
		return false, err
	}

	// Wait for volumes to attach/mount
	//等待所有定义在Pod中的卷被正确挂载。这是启动容器前的必要步骤，如果挂载失败，则记录警告并返回错误。
	if err := kl.volumeManager.WaitForAttachAndMount(ctx, pod); err != nil {
		if !wait.Interrupted(err) {
			kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedMountVolume, "Unable to attach or mount volumes: %v", err)
			klog.ErrorS(err, "Unable to attach or mount volumes for pod; skipping pod", "pod", klog.KObj(pod))
		}
		return false, err
	}

	// Fetch the pull secrets for the pod
	//获取Pod需要的所有镜像拉取秘钥，这些秘钥用于从私有容器镜像仓库拉取镜像
	pullSecrets := kl.getPullSecretsForPod(pod)

	// Ensure the pod is being probed
	//为Pod配置健康检查探针，如liveness和readiness探针，这些探针帮助Kubelet确定Pod内容器的健康状态
	kl.probeManager.AddPod(pod)
	//如果启用了Pod的垂直扩缩容功能（In-Place Pod Vertical Scaling），检查并处理Pod的资源调整请求。这通常发生在Pod运行时基于实际负载动态调整其资源配置
	if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {
		// Handle pod resize here instead of doing it in HandlePodUpdates because
		// this conveniently retries any Deferred resize requests
		// TODO(vinaykul,InPlacePodVerticalScaling): Investigate doing this in HandlePodUpdates + periodic SyncLoop scan
		//     See: https://github.com/kubernetes/kubernetes/pull/102884#discussion_r663160060
		if kl.podWorkers.CouldHaveRunningContainers(pod.UID) && !kubetypes.IsStaticPod(pod) {
			pod = kl.handlePodResourcesResize(pod)
		}
	}

	// TODO(#113606): use cancellation from the incoming context parameter, which comes from the pod worker.
	// Currently, using cancellation from that context causes test failures. To remove this WithoutCancel,
	// any wait.Interrupted errors need to be filtered from result and bypass the reasonCache - cancelling
	// the context for SyncPod is a known and deliberate error, not a generic error.
	// Use WithoutCancel instead of a new context.TODO() to propagate trace context
	// Call the container runtime's SyncPod callback
	sctx := context.WithoutCancel(ctx)
	//调用容器运行时的SyncPod方法来同步Pod状态，包括启动新容器、停止旧容器、重启失败容器等
	result := kl.containerRuntime.SyncPod(sctx, pod, podStatus, pullSecrets, kl.backOff)
	kl.reasonCache.Update(pod.UID, result)
	//处理容器运行时返回的同步结果。如果同步过程中出现错误，根据错误类型决定是否重试
	if err := result.Error(); err != nil {
		// Do not return error if the only failures were pods in backoff
		for _, r := range result.SyncResults {
			if r.Error != kubecontainer.ErrCrashLoopBackOff && r.Error != images.ErrImagePullBackOff {
				// Do not record an event here, as we keep all event logging for sync pod failures
				// local to container runtime, so we get better errors.
				return false, err
			}
		}

		return false, nil
	}
	//如果Pod正在进行资源调整，并且启用了相关特性，则更新Pod状态缓存，以确保状态是最新的
	if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) && isPodResizeInProgress(pod, &apiPodStatus) {
		// While resize is in progress, periodically call PLEG to update pod cache
		runningPod := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
		if err, _ := kl.pleg.UpdateCache(&runningPod, pod.UID); err != nil {
			klog.ErrorS(err, "Failed to update pod cache", "pod", klog.KObj(pod))
			return false, err
		}
	}

	return false, nil
}
```

#### 2.1.4 containerRuntime.SyncPod

该函数是最核心的功能，函数的主要功能就是，输入Pod期望的状态。然后调用cri 对容器执行对应的操作，已达到期望状态。

具体流程如下：

（1）调用computePodActions 判断pod状态。主要是完善这个结构

```
	changes := podActions{
		KillPod:           createPodSandbox,     // bool值 判断是否需要kill pod
		CreateSandbox:     createPodSandbox,     // bool值 判断是否需要创建sandbox
		SandboxID:         sandboxID,            // string, sandboxId
		Attempt:           attempt,              // int，sandbox创建次数，每次+1
		ContainersToStart: []int{},              // int，第几个容器需要start 
		ContainersToKill:  make(map[kubecontainer.ContainerID]containerToKillInfo),   //哪写容器需要kill
	}
```

如果判断CreateSandbox是否为真，获取pod引用，出现问题记录，CreateSandbox != nil 就需要杀死并重新创建；如果CreateSandbox = "" 就代表需要新建。

（2）如果KillPod为true，终止pod，如果需要创建新的沙盒，就清理容器初始化数据

（3）如果KillPod为false，代表没必要终止整个pod，只需要停止不应该继续运行的容器，便利ContainersToKill容器列表，对每个容器进行停止操作，然后清理已终止的初始化容器垃圾回收

（4）获取podSandboxID，然后判断odContainerChanges.CreateSandbox==true，开始创建沙盒，将沙盒创建结果同步到createSandboxResult中，如果启用了动态资源分配，分配资源，失败记录事件；尝试createPodSandbox创建沙盒，发生错误记录日志，并沙盒创建失败事件指标+1，标记开始的同步结果createSandboxResult为失败，尝试为pod创建引用来触发Kubernetes 事件，如果创建失败，触发FailedCreatePodSandBox的warning警告事件，如果没有错误就是创建成功；从容器运行时服务获取Pod沙盒的状态，发生错误记录一个警告事件，表明无法获取沙盒状态，检查 resp.GetStatus() == nil，为空记录同步失败返回，如果Pod不是使用主机网络，更新Pod的IP地址,生成Sandbox配置，定义一个启动各种容器start的函数

（5）for循环启动odContainerChanges.EphemeralContainersToStart临时容器列表，如果启用了Sidecar 容器功能，就启动全部初始化容器，如果标注为可重启的失败了跳过，失败一个就终止整个pod初始化；没有启用Sidecar 容器功能，就只需要启动一个初始化容器，成功记录日志返回；

（6）如果允许在位Pod垂直缩放，更新容器资源，在不停止pod的情况下，更新CPU,内存

（7）启动所有常规容器

```go
// SyncPod syncs the running pod into the desired pod by executing following steps:
//
//  1. Compute sandbox and container changes.
//  2. Kill pod sandbox if necessary.
//  3. Kill any containers that should not be running.
//  4. Create sandbox if necessary.
//  5. Create ephemeral containers.
//  6. Create init containers.
//  7. Resize running containers (if InPlacePodVerticalScaling==true)
//  8. Create normal containers.

func (m *kubeGenericRuntimeManager) SyncPod(ctx context.Context, pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, backOff *flowcontrol.Backoff) (result kubecontainer.PodSyncResult) {
	// Step 1: Compute sandbox and container changes.
	//判断当前Pod状态和期望的Pod状态是否一致，计算出需要执行的动作，动作：是否需要创建新的沙盒、杀死现有容器、创建新容器等。
	podContainerChanges := m.computePodActions(ctx, pod, podStatus)
	klog.V(3).InfoS("computePodActions got for pod", "podActions", podContainerChanges, "pod", klog.KObj(pod))
	//判断CreateSandbox字段是否为真，如果为真，则创建新的沙盒。
	if podContainerChanges.CreateSandbox {
		//获取pod引用
		ref, err := ref.GetReference(legacyscheme.Scheme, pod)
		//出现错误记录错误
		if err != nil {
			klog.ErrorS(err, "Couldn't make a ref to pod", "pod", klog.KObj(pod))
		}
		//如果SandboxID不为空，Sandbox需要重新创建；如果为空，则创建新的沙盒。
		if podContainerChanges.SandboxID != "" {
			m.recorder.Eventf(ref, v1.EventTypeNormal, events.SandboxChanged, "Pod sandbox changed, it will be killed and re-created.")
		} else {
			klog.V(4).InfoS("SyncPod received new pod, will create a sandbox for it", "pod", klog.KObj(pod))
		}
	}

	// Step 2: Kill the pod if the sandbox has changed.
	//如果KillPod为真，表明需要终止并重建 Pod 的 Sandbox（沙盒）
	if podContainerChanges.KillPod {
		//如果如果KillPod为真，CreateSandbox也为真，终止现有的 Sandbox 并启动一个新的。这通常发生在 Pod 的网络配置或其他基础设置改变时。
		if podContainerChanges.CreateSandbox {
			klog.V(4).InfoS("Stopping PodSandbox for pod, will start new one", "pod", klog.KObj(pod))
		} else {
			//如果如果KillPod为真，CreateSandbox为假，则表示因为 Pod 中的所有其他容器都已停止运行，需要停止 Sandbox。
			klog.V(4).InfoS("Stopping PodSandbox for pod, because all other containers are dead", "pod", klog.KObj(pod))
		}
		//使用killPodWithSyncResult 终止pod，并返回结果
		killResult := m.killPodWithSyncResult(ctx, pod, kubecontainer.ConvertPodStatusToRunningPod(m.runtimeName, podStatus), nil)
		result.AddPodSyncResult(killResult)
		//终止操作出现问题记录问题立即返回，终止后续操作
		if killResult.Error() != nil {
			klog.ErrorS(killResult.Error(), "killPodWithSyncResult failed")
			return
		}
		//如果需要创建新的沙盒，purgeInitContainers清理初始化容器数据，为 Pod 创建新的运行环境做准备
		if podContainerChanges.CreateSandbox {
			m.purgeInitContainers(ctx, pod, podStatus)
		}
	} else {
		// Step 3: kill any running containers in this pod which are not to keep.
		//如果没必要终止整个pod，只需要终止不应该继续运行的容器，ContainersToKill就是不应该继续运行的容器列表
		for containerID, containerInfo := range podContainerChanges.ContainersToKill {
			klog.V(3).InfoS("Killing unwanted container for pod", "containerName", containerInfo.name, "containerID", containerID, "pod", klog.KObj(pod))
			killContainerResult := kubecontainer.NewSyncResult(kubecontainer.KillContainer, containerInfo.name)
			//操作都记录在result种
			result.AddSyncResult(killContainerResult)
			//使用killContainer对每个容器进行停止操作
			if err := m.killContainer(ctx, pod, containerID, containerInfo.name, containerInfo.message, containerInfo.reason, nil, nil); err != nil {
				//如果终止容器时发生错误，记录错误信息并立即返回，停止后续所有操作
				killContainerResult.Fail(kubecontainer.ErrKillContainer, err.Error())
				klog.ErrorS(err, "killContainer for pod failed", "containerName", containerInfo.name, "containerID", containerID, "pod", klog.KObj(pod))
				return
			}
		}
	}

	// Keep terminated init containers fairly aggressively controlled
	// This is an optimization because container removals are typically handled
	// by container garbage collector.
	//清理已经终止的初始化容器
	m.pruneInitContainersBeforeStart(ctx, pod, podStatus)

	// We pass the value of the PRIMARY podIP and list of podIPs down to
	// generatePodSandboxConfig and generateContainerConfig, which in turn
	// passes it to various other functions, in order to facilitate functionality
	// that requires this value (hosts file and downward API) and avoid races determining
	// the pod IP in cases where a container requires restart but the
	// podIP isn't in the status manager yet. The list of podIPs is used to
	// generate the hosts file.
	//
	// We default to the IPs in the passed-in pod status, and overwrite them if the
	// sandbox needs to be (re)started.
	//把podStatus的IP字段保存到podIPs切片种
	var podIPs []string
	if podStatus != nil {
		podIPs = podStatus.IPs
	}

	// Step 4: Create a sandbox for the pod if necessary.
	//获取沙盒ID
	podSandboxID := podContainerChanges.SandboxID
	//如果CreateSandbox为true
	if podContainerChanges.CreateSandbox {
		var msg string
		var err error
		//记录日志准备开始创建沙盒
		klog.V(4).InfoS("Creating PodSandbox for pod", "pod", klog.KObj(pod))
		// 增加创建沙盒的度量计数
		metrics.StartedPodsTotal.Inc()
		// 创建沙盒的同步结果，并加入到结果列表中
		createSandboxResult := kubecontainer.NewSyncResult(kubecontainer.CreatePodSandbox, format.Pod(pod))
		result.AddSyncResult(createSandboxResult)

		// ConvertPodSysctlsVariableToDotsSeparator converts sysctl variable
		// in the Pod.Spec.SecurityContext.Sysctls slice into a dot as a separator.
		// runc uses the dot as the separator to verify whether the sysctl variable
		// is correct in a separate namespace, so when using the slash as the sysctl
		// variable separator, runc returns an error: "sysctl is not in a separate kernel namespace"
		// and the podSandBox cannot be successfully created. Therefore, before calling runc,
		// we need to convert the sysctl variable, the dot is used as a separator to separate the kernel namespace.
		// When runc supports slash as sysctl separator, this function can no longer be used.
		//处理系统调控变量，将斜线分隔的变量转换为点分隔，以符合运行时的要求
		sysctl.ConvertPodSysctlsVariableToDotsSeparator(pod.Spec.SecurityContext)

		// Prepare resources allocated by the Dynammic Resource Allocation feature for the pod
		// 如果启用了动态资源分配功能，准备动态分配的资源
		if utilfeature.DefaultFeatureGate.Enabled(features.DynamicResourceAllocation) {
			if err := m.runtimeHelper.PrepareDynamicResources(pod); err != nil {
				ref, referr := ref.GetReference(legacyscheme.Scheme, pod)
				if referr != nil {
					klog.ErrorS(referr, "Couldn't make a ref to pod", "pod", klog.KObj(pod))
					return
				}
				// 记录准备动态资源失败的事件
				m.recorder.Eventf(ref, v1.EventTypeWarning, events.FailedPrepareDynamicResources, "Failed to prepare dynamic resources: %v", err)
				klog.ErrorS(err, "Failed to prepare dynamic resources", "pod", klog.KObj(pod))
				return
			}
		}
		// 尝试创建Pod沙盒，并处理可能的错误
		podSandboxID, msg, err = m.createPodSandbox(ctx, pod, podContainerChanges.Attempt)
		if err != nil {
			// createPodSandbox can return an error from CNI, CSI,
			// or CRI if the Pod has been deleted while the POD is
			// being created. If the pod has been deleted then it's
			// not a real error.
			//
			// SyncPod can still be running when we get here, which
			// means the PodWorker has not acked the deletion.
			//创建发生错误，记录日志并返回
			if m.podStateProvider.IsPodTerminationRequested(pod.UID) {
				klog.V(4).InfoS("Pod was deleted and sandbox failed to be created", "pod", klog.KObj(pod), "podUID", pod.UID)
				return
			}
			//记录沙盒创建失败的事件
			metrics.StartedPodsErrorsTotal.Inc()
			//标记创建沙盒的结果为失败，并记录日志
			createSandboxResult.Fail(kubecontainer.ErrCreatePodSandbox, msg)
			klog.ErrorS(err, "CreatePodSandbox for pod failed", "pod", klog.KObj(pod))
			//尝试为pod创建引用来触发Kubernetes 事件，如果创建失败，触发FailedCreatePodSandBox的warning警告事件
			ref, referr := ref.GetReference(legacyscheme.Scheme, pod)
			if referr != nil {
				klog.ErrorS(referr, "Couldn't make a ref to pod", "pod", klog.KObj(pod))
			}
			m.recorder.Eventf(ref, v1.EventTypeWarning, events.FailedCreatePodSandBox, "Failed to create pod sandbox: %v", err)
			return
		}
		//创建成功
		klog.V(4).InfoS("Created PodSandbox for pod", "podSandboxID", podSandboxID, "pod", klog.KObj(pod))

		// 请求从容器运行时服务获取Pod沙盒的状态
		resp, err := m.runtimeService.PodSandboxStatus(ctx, podSandboxID, false)
		if err != nil {
			//  如果请求沙盒状态失败，尝试获取Pod的引用
			ref, referr := ref.GetReference(legacyscheme.Scheme, pod)
			// 如果获取引用失败，记录错误日志
			if referr != nil {
				klog.ErrorS(referr, "Couldn't make a ref to pod", "pod", klog.KObj(pod))
			}
			//  记录一个警告事件，表明无法获取沙盒状态
			m.recorder.Eventf(ref, v1.EventTypeWarning, events.FailedStatusPodSandBox, "Unable to get pod sandbox status: %v", err)
			// 记录一个错误日志，并标记同步结果为失败，然后返回，不再继续后续步骤
			klog.ErrorS(err, "Failed to get pod sandbox status; Skipping pod", "pod", klog.KObj(pod))
			result.Fail(err)
			return
		}
		// 检查响应中的状态是否为空，如果为空，则记录同步失败并返回
		if resp.GetStatus() == nil {
			result.Fail(errors.New("pod sandbox status is nil"))
			return
		}

		// If we ever allow updating a pod from non-host-network to
		// host-network, we may use a stale IP.
		// 如果Pod不是使用主机网络，更新Pod的IP地址
		// 这是因为Pod的网络环境可能发生变化，特别是在沙盒重新创建的情况下
		if !kubecontainer.IsHostNetworkPod(pod) {
			// Overwrite the podIPs passed in the pod status, since we just started the pod sandbox.
			// 覆盖传入的Pod状态中的IP地址，因为我们刚刚启动了沙盒
			podIPs = m.determinePodSandboxIPs(pod.Namespace, pod.Name, resp.GetStatus())
			// 记录确定的新IP地址
			klog.V(4).InfoS("Determined the ip for pod after sandbox changed", "IPs", podIPs, "pod", klog.KObj(pod))
		}
	}

	// the start containers routines depend on pod ip(as in primary pod ip)
	// instead of trying to figure out if we have 0 < len(podIPs)
	// everytime, we short circuit it here
	// 为接下来的容器启动准备
	podIP := ""
	if len(podIPs) != 0 {
		podIP = podIPs[0]
	}

	// Get podSandboxConfig for containers to start.
	configPodSandboxResult := kubecontainer.NewSyncResult(kubecontainer.ConfigPodSandbox, podSandboxID)
	result.AddSyncResult(configPodSandboxResult)
	// 生成沙盒配置，如果生成失败则记录错误并返回。
	podSandboxConfig, err := m.generatePodSandboxConfig(pod, podContainerChanges.Attempt)
	if err != nil {
		message := fmt.Sprintf("GeneratePodSandboxConfig for pod %q failed: %v", format.Pod(pod), err)
		klog.ErrorS(err, "GeneratePodSandboxConfig for pod failed", "pod", klog.KObj(pod))
		configPodSandboxResult.Fail(kubecontainer.ErrConfigPodSandbox, message)
		return
	}

	// Helper containing boilerplate common to starting all types of containers.
	// typeName is a description used to describe this type of container in log messages,
	// currently: "container", "init container" or "ephemeral container"
	// metricLabel is the label used to describe this type of container in monitoring metrics.
	// currently: "container", "init_container" or "ephemeral_container"
	// 定义一个辅助函数，用于启动各类型容器，例如常规容器、初始容器和临时容器
	start := func(ctx context.Context, typeName, metricLabel string, spec *startSpec) error {
		startContainerResult := kubecontainer.NewSyncResult(kubecontainer.StartContainer, spec.container.Name)
		result.AddSyncResult(startContainerResult)
		//  检查容器是否处于退避状态,就是一直发生错误多次重启，backoff就是暂时停止重启
		//// pod: 待检查的Pod对象
		//// container: 待检查的容器对象
		//// podStatus: Pod的容器状态信息
		//// backOff: 退避控制对象
		isInBackOff, msg, err := m.doBackOff(pod, spec.container, podStatus, backOff)
		if isInBackOff {
			startContainerResult.Fail(err, msg)
			klog.V(4).InfoS("Backing Off restarting container in pod", "containerType", typeName, "container", spec.container, "pod", klog.KObj(pod))
			return err
		}
		// 监控指标
		metrics.StartedContainersTotal.WithLabelValues(metricLabel).Inc()
		if sc.HasWindowsHostProcessRequest(pod, spec.container) {
			metrics.StartedHostProcessContainersTotal.WithLabelValues(metricLabel).Inc()
		}
		klog.V(4).InfoS("Creating container in pod", "containerType", typeName, "container", spec.container, "pod", klog.KObj(pod))
		// NOTE (aramase) podIPs are populated for single stack and dual stack clusters. Send only podIPs.
		// 启动容器
		if msg, err := m.startContainer(ctx, podSandboxID, podSandboxConfig, spec, pod, podStatus, pullSecrets, podIP, podIPs); err != nil {
			// startContainer() returns well-defined error codes that have reasonable cardinality for metrics and are
			// useful to cluster administrators to distinguish "server errors" from "user errors".
			// 记录容器启动失败的详细信息。
			metrics.StartedContainersErrorsTotal.WithLabelValues(metricLabel, err.Error()).Inc()
			//发丝只有podIPs的信息
			if sc.HasWindowsHostProcessRequest(pod, spec.container) {
				metrics.StartedHostProcessContainersErrorsTotal.WithLabelValues(metricLabel, err.Error()).Inc()
			}
			startContainerResult.Fail(err, msg)
			// known errors that are logged in other places are logged at higher levels here to avoid
			// repetitive log spam
			// 根据错误类型记录不同级别的日志。
			switch {
			case err == images.ErrImagePullBackOff:
				klog.V(3).InfoS("Container start failed in pod", "containerType", typeName, "container", spec.container, "pod", klog.KObj(pod), "containerMessage", msg, "err", err)
			default:
				utilruntime.HandleError(fmt.Errorf("%v %+v start failed in pod %v: %v: %s", typeName, spec.container, format.Pod(pod), err, msg))
			}
			return err
		}

		return nil
	}

	// Step 5: start ephemeral containers
	// These are started "prior" to init containers to allow running ephemeral containers even when there
	// are errors starting an init container. In practice init containers will start first since ephemeral
	// containers cannot be specified on pod creation.
	// 启动每一个临时容器，podContainerChanges.EphemeralContainersToStart临时容器列表
	for _, idx := range podContainerChanges.EphemeralContainersToStart {
		start(ctx, "ephemeral container", metrics.EphemeralContainer, ephemeralContainerStartSpec(&pod.Spec.EphemeralContainers[idx]))
	}
	// 判断是否启用了Sidecar 容器，注意这里取反！真正开启的时候是true，但是！取反执行else；相反是执行if
	if !utilfeature.DefaultFeatureGate.Enabled(features.SidecarContainers) {
		// Step 6: start the init container.
		// 启动一个初始化容器
		if container := podContainerChanges.NextInitContainerToStart; container != nil {
			// Start the next init container.
			//  启动初始化容器
			if err := start(ctx, "init container", metrics.InitContainer, containerStartSpec(container)); err != nil {
				return
			}

			// Successfully started the container; clear the entry in the failure
			// 容器启动成功，记录日志
			klog.V(4).InfoS("Completed init container for pod", "containerName", container.Name, "pod", klog.KObj(pod))
		}
	} else {
		// Step 6: start init containers.
		// 启动初始化容器，podContainerChanges.InitContainersToStart是初始化容器列表
		for _, idx := range podContainerChanges.InitContainersToStart {
			container := &pod.Spec.InitContainers[idx]
			// Start the next init container.
			// 启动全部初始化容器，成功就记录日志
			if err := start(ctx, "init container", metrics.InitContainer, containerStartSpec(container)); err != nil {
				if types.IsRestartableInitContainer(container) {
					//  如果是标记可重启的初始化容器失败，则跳过当前容器，继续其他容器
					klog.V(4).InfoS("Failed to start the restartable init container for the pod, skipping", "initContainerName", container.Name, "pod", klog.KObj(pod))
					continue
				}
				// 初始化容器启动失败，终止整个Pod的初始化
				klog.V(4).InfoS("Failed to initialize the pod, as the init container failed to start, aborting", "initContainerName", container.Name, "pod", klog.KObj(pod))
				return
			}

			// Successfully started the container; clear the entry in the failure
			klog.V(4).InfoS("Completed init container for pod", "containerName", container.Name, "pod", klog.KObj(pod))
		}
	}

	// Step 7: For containers in podContainerChanges.ContainersToUpdate[CPU,Memory] list, invoke UpdateContainerResources
	// 如果允许在位Pod垂直缩放，更新容器资源，在不停止pod的情况下
	if isInPlacePodVerticalScalingAllowed(pod) {
		if len(podContainerChanges.ContainersToUpdate) > 0 || podContainerChanges.UpdatePodResources {
			m.doPodResizeAction(pod, podStatus, podContainerChanges, result)
		}
	}

	// Step 8: start containers in podContainerChanges.ContainersToStart.
	//  启动所有需要启动的容器
	for _, idx := range podContainerChanges.ContainersToStart {
		start(ctx, "container", metrics.Container, containerStartSpec(&pod.Spec.Containers[idx]))
	}

	return
}

// If a container is still in backoff, the function will return a brief backoff error and
// a detailed error message.
func (m *kubeGenericRuntimeManager) doBackOff(pod *v1.Pod, container *v1.Container, podStatus *kubecontainer.PodStatus, backOff *flowcontrol.Backoff) (bool, string, error) {
	var cStatus *kubecontainer.Status
	for _, c := range podStatus.ContainerStatuses {
		if c.Name == container.Name && c.State == kubecontainer.ContainerStateExited {
			cStatus = c
			break
		}
	}

	if cStatus == nil {
		return false, "", nil
	}

	klog.V(3).InfoS("Checking backoff for container in pod", "containerName", container.Name, "pod", klog.KObj(pod))
	// Use the finished time of the latest exited container as the start point to calculate whether to do back-off.
	ts := cStatus.FinishedAt
	// backOff requires a unique key to identify the container.
	key := getStableKey(pod, container)
	if backOff.IsInBackOffSince(key, ts) {
		if containerRef, err := kubecontainer.GenerateContainerRef(pod, container); err == nil {
			m.recorder.Eventf(containerRef, v1.EventTypeWarning, events.BackOffStartContainer,
				fmt.Sprintf("Back-off restarting failed container %s in pod %s", container.Name, format.Pod(pod)))
		}
		err := fmt.Errorf("back-off %s restarting failed container=%s pod=%s", backOff.Get(key), container.Name, format.Pod(pod))
		klog.V(3).InfoS("Back-off restarting failed container", "err", err.Error())
		return true, err.Error(), kubecontainer.ErrCrashLoopBackOff
	}

	backOff.Next(key, ts)
	return false, "", nil
}
```

#### 2.1.5 总结

再次梳理总结一下pod创建过程：有pod创建，会别kubelet监听，然后通过`podWorkerLoop`-> `syncPod`-> `Runtime.SyncPod`

（1）更新pod的status，比如status.startTime， containerStatus等等

（2）调用containerRuntime.SyncPod 完成 创建sandbox, 启动容器等过程

（3）后面容器启动好了后，会触发pleg channel的update，这个后面在分析

### 3. pod创建过程详细分析

#### 3.1 创建sandbox过程做了什么工作

##### 3.1.1 createPodSandbox

在SyncPod中的m.createPodSandbox 调用了 createPodSandbox

kubeGenericRuntimeManager.createPodSandbox核心逻辑如下：

（1）创建logdir。 默认是 /var/log/pods/<NAMESPACE>_<NAME>_<UID>/

（2）调用runtimeService.RunPodSandbox创建sandbox

```go
// createPodSandbox creates a pod sandbox and returns (podSandBoxID, message, error).
func (m *kubeGenericRuntimeManager) createPodSandbox(ctx context.Context, pod *v1.Pod, attempt uint32) (string, string, error) {
	// 根据 pod 规范和尝试次数生成沙盒配置。
	podSandboxConfig, err := m.generatePodSandboxConfig(pod, attempt)
	if err != nil {
		message := fmt.Sprintf("Failed to generate sandbox config for pod %q: %v", format.Pod(pod), err)
		klog.ErrorS(err, "Failed to generate sandbox config for pod", "pod", klog.KObj(pod))
		return "", message, err
	}

	// Create pod logs directory
	// 创建存储 pod 日志的目录。/var/log/pods/<NAMESPACE>_<NAME>_<UID>/
	err = m.osInterface.MkdirAll(podSandboxConfig.LogDirectory, 0755)
	if err != nil {
		message := fmt.Sprintf("Failed to create log directory for pod %q: %v", format.Pod(pod), err)
		klog.ErrorS(err, "Failed to create log directory for pod", "pod", klog.KObj(pod))
		return "", message, err
	}
	// 如果定义了运行时类管理器，确定沙盒的适当运行时处理器。runc, gVisor, 和 KataContainers
	runtimeHandler := ""
	if m.runtimeClassManager != nil {
		runtimeHandler, err = m.runtimeClassManager.LookupRuntimeHandler(pod.Spec.RuntimeClassName)
		if err != nil {
			message := fmt.Sprintf("Failed to create sandbox for pod %q: %v", format.Pod(pod), err)
			return "", message, err
		}
		if runtimeHandler != "" {
			klog.V(2).InfoS("Running pod with runtime handler", "pod", klog.KObj(pod), "runtimeHandler", runtimeHandler)
		}
	}
	// 使用运行时服务运行沙盒。
	podSandBoxID, err := m.runtimeService.RunPodSandbox(ctx, podSandboxConfig, runtimeHandler)
	if err != nil {
		message := fmt.Sprintf("Failed to create sandbox for pod %q: %v", format.Pod(pod), err)
		klog.ErrorS(err, "Failed to create sandbox for pod", "pod", klog.KObj(pod))
		return "", message, err
	}
	// 如果沙盒创建成功，返回沙盒的唯一标识符。
	return podSandBoxID, "", nil
}
```

##### 3.1.2 runtimeService.RunPodSandbox

这里主要调用`runtimeClient.RunPodSandbox`

```go
// RunPodSandbox creates and starts a pod-level sandbox. Runtimes should ensure
// the sandbox is in ready state.
func (r *remoteRuntimeService) RunPodSandbox(ctx context.Context, config *runtimeapi.PodSandboxConfig, runtimeHandler string) (string, error) {
	// Use 2 times longer timeout for sandbox operation (4 mins by default)
	// TODO: Make the pod sandbox timeout configurable.
	timeout := r.timeout * 2

	klog.V(10).InfoS("[RemoteRuntimeService] RunPodSandbox", "config", config, "runtimeHandler", runtimeHandler, "timeout", timeout)

	ctx, cancel := context.WithTimeout(ctx, timeout)
	defer cancel()

	resp, err := r.runtimeClient.RunPodSandbox(ctx, &runtimeapi.RunPodSandboxRequest{
		Config:         config,
		RuntimeHandler: runtimeHandler,
	})

	if err != nil {
		klog.ErrorS(err, "RunPodSandbox from runtime service failed")
		return "", err
	}

	podSandboxID := resp.PodSandboxId

	if podSandboxID == "" {
		errorMessage := fmt.Sprintf("PodSandboxId is not set for sandbox %q", config.Metadata)
		err := errors.New(errorMessage)
		klog.ErrorS(err, "RunPodSandbox failed")
		return "", err
	}

	klog.V(10).InfoS("[RemoteRuntimeService] RunPodSandbox Response", "podSandboxID", podSandboxID)

	return podSandboxID, nil
}
```

##### 3.1.3 runtimeClient.RunPodSandbox

这个也是个接口，在 k8s.io/cri-api/pkg/apis/runtime/v1alpha2/api.pb.go 中实现。并且指定了该服务的Handler是_RuntimeService_RunPodSandbox_Handler

```go
func (c *runtimeServiceClient) RunPodSandbox(ctx context.Context, in *RunPodSandboxRequest, opts ...grpc.CallOption) (*RunPodSandboxResponse, error) {
	out := new(RunPodSandboxResponse)
	err := c.cc.Invoke(ctx, "/runtime.v1.RuntimeService/RunPodSandbox", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

```

_RuntimeService_RunPodSandbox_Handler调用了srv.(RuntimeServiceServer).RunPodSandbox处理。

```go
func _RuntimeService_RunPodSandbox_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(RunPodSandboxRequest)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(RuntimeServiceServer).RunPodSandbox(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/runtime.v1.RuntimeService/RunPodSandbox",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(RuntimeServiceServer).RunPodSandbox(ctx, req.(*RunPodSandboxRequest))
	}
	return interceptor(ctx, in, info, handler)
}

```

##### 3.1.4 srv.(RuntimeServiceServer).RunPodSandbox

这个也是接口，主要是针对不同的容器运行时。这里分析的是containred，还有cri-o这里就不分析了，所以是如下的函数。

该函数逻辑为：

**获取请求中的配置信息**

- 从请求中提取沙盒配置，并生成沙盒唯一ID和名称。

**保留沙盒名称**

- 确保沙盒名称唯一，避免并发创建相同的沙盒名称。

**获取OCI运行时配置**

- 获取和生成运行时选项，并设置到沙盒信息中。

**创建初始内部沙盒对象**

- 初始化沙盒对象，并尝试将其保存到沙盒存储中。

**设置网络命名空间**

- 如果不是主机网络且没有启用用户命名空间，创建网络命名空间并设置网络路径。

**设置网络**

- 调用 `setupPodNetwork` 函数为沙盒设置网络。

**启动沙盒**

- 调用沙盒服务的 `CreateSandbox` 和 `StartSandbox` 方法，启动沙盒并获取控制器。

**处理用户命名空间**

- 如果启用了用户命名空间且未使用主机网络，根据沙盒的进程ID创建网络命名空间并设置路径。

**更新沙盒状态**

- 更新沙盒状态为就绪，并将沙盒添加到沙盒存储中。

**生成并发送事件**

- 生成并发送沙盒创建和启动事件。

**监控沙盒退出**

- 启动沙盒退出监控，确保在接收到任务退出事件时沙盒已经在存储中。

**返回响应**

- 成功执行所有操作后，返回包含沙盒ID的响应。

`为什么要判断两次网络`？

两次判断网络的原因在于不同的命名空间设置方式：

- **第一次判断**处理普通的网络命名空间创建和设置。
- **第二次判断**处理启用用户命名空间后的网络命名空间设置，因为用户命名空间的网络命名空间需要由运行时基于沙盒的进程ID创建。

==普通的网络命名空间和用户命名空间 不能共存==

`yaml对比`

是的，普通的网络命名空间和用户命名空间后的网络命名空间设置只能存在一个。根据配置，系统会选择其中一种进行设置。我们可以通过 Kubernetes Pod 的 YAML 配置来启用这两种不同的网络命名空间设置。

`启用普通的网络命名空间`

普通的网络命名空间是默认设置，只要不启用用户命名空间即可。下面是一个简单的 Pod YAML 配置示例，表示启用普通的网络命名空间：

```
yaml复制代码apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mycontainer
    image: busybox
    command: ["sh", "-c", "echo Hello Kubernetes! && sleep 3600"]
  # 默认情况下，不启用用户命名空间
```

`启用用户命名空间`

要启用用户命名空间，可以在 Pod 的安全上下文中进行配置，特别是设置 `namespaceOptions` 的 `usernsMode` 为 `POD`。以下是一个示例：

```
yaml复制代码apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mycontainer
    image: busybox
    command: ["sh", "-c", "echo Hello Kubernetes! && sleep 3600"]
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    namespaceOptions:
      usernsMode: POD  # 启用用户命名空间
```

在这个配置中，`namespaceOptions` 的 `usernsMode` 设置为 `POD`，启用了用户命名空间。在这种情况下，网络命名空间将通过 `netns.NewNetNSFromPID` 基于沙盒的进程ID创建。

`配置对比`

- **普通网络命名空间**: 不需要额外配置，使用默认设置。
- **用户命名空间**: 需要在 `securityContext` 中设置 `namespaceOptions` 的 `usernsMode` 为 `POD`。\

如果是OCI运行时类型是==Pod==沙盒模式，会尝试记载Sandbox的容器；记住如果不是POD模式，那么Sandbox的容器在之前的`c.sandboxService.StartSandbox`就启动了

```go
	//如果OCI运行时类型是Pod沙盒模式，尝试加载对应的容器。如果加载失败，返回错误。加载成功后，将容器实例关联到沙盒对象。
	// TODO: get rid of this. sandbox object should no longer have Container field.
	if ociRuntime.Sandboxer == string(criconfig.ModePodSandbox) {
		container, err := c.client.LoadContainer(ctx, id)
		if err != nil {
			return nil, fmt.Errorf("failed to load container %q for sandbox: %w", id, err)
		}
		sandbox.Container = container
	}
```

* 真正的Sandbox的启动，其实是c.nri.RunPodSandbox(ctx, &sandbox)，因为前面的是先设置好网络，然后`c.sandboxService.StartSandbox`启动Sandbox容器，然后才是c.nri.RunPodSandbox启动真正设置好网络和启动好容器的Sandbox

下面还要分析几个重要函数，都是通过初始化GRPC实列化controller然后获得方法调用：

* 设置网络：**setupPodNetwork**
* 启动Sandbox：**Start**

```go
// RunPodSandbox creates and starts a pod-level sandbox. Runtimes should ensure
// the sandbox is in ready state.
func (c *criService) RunPodSandbox(ctx context.Context, r *runtime.RunPodSandboxRequest) (_ *runtime.RunPodSandboxResponse, retErr error) {
	//这里获取请求中的配置信息
	config := r.GetConfig()
	log.G(ctx).Debugf("Sandbox config %+v", config)

	// Generate unique id and name for the sandbox and reserve the name.
	//生成了沙盒的唯一ID
	id := util.GenerateID()
	//尝试获取元数据
	metadata := config.GetMetadata()
	if metadata == nil {
		return nil, errors.New("sandbox config must include metadata")
	}
	//根据元数据生成沙盒名称
	name := makeSandboxName(metadata)
	log.G(ctx).WithField("podsandboxid", id).Debugf("generated id for sandbox name %q", name)

	// cleanupErr records the last error returned by the critical cleanup operations in deferred functions,
	// like CNI teardown and stopping the running sandbox task.
	// If cleanup is not completed for some reason, the CRI-plugin will leave the sandbox
	// in a not-ready state, which can later be cleaned up by the next execution of the kubelet's syncPod workflow.
	//定义一个清理函数返回的错误
	var cleanupErr error

	// Reserve the sandbox name to avoid concurrent `RunPodSandbox` request starting the
	// same sandbox.
	//保留沙盒名称，避免并发创建;
	if err := c.sandboxNameIndex.Reserve(name, id); err != nil {
		return nil, fmt.Errorf("failed to reserve sandbox name %q: %w", name, err)
	}
	//如果保留失败，释放沙盒名称
	defer func() {
		// Release the name if the function returns with an error.
		// When cleanupErr != nil, the name will be cleaned in sandbox_remove.
		if retErr != nil && cleanupErr == nil {
			c.sandboxNameIndex.ReleaseByName(name)
		}
	}()

	var (
		err         error
		sandboxInfo = sb.Sandbox{ID: id}
	)
	//获取OCI运行时配置
	ociRuntime, err := c.config.GetSandboxRuntime(config, r.GetRuntimeHandler())
	if err != nil {
		return nil, fmt.Errorf("unable to get OCI runtime for sandbox %q: %w", id, err)
	}
	//设置sandboxInfo中的运行时信息，包括运行时的类型和沙盒器
	sandboxInfo.Runtime.Name = ociRuntime.Type
	sandboxInfo.Sandboxer = ociRuntime.Sandboxer
	//记录开始处理运行时的时间，并尝试生成运行时选项
	runtimeStart := time.Now()
	// Retrieve runtime options
	runtimeOpts, err := criconfig.GenerateRuntimeOptions(ociRuntime)
	if err != nil {
		return nil, fmt.Errorf("failed to generate sandbox runtime options: %w", err)
	}
	//如果运行时选项非空，则将这些选项序列化存储到sandboxInfo.Runtime.Options中。如果序列化失败，则返回错误
	if runtimeOpts != nil {
		sandboxInfo.Runtime.Options, err = typeurl.MarshalAny(runtimeOpts)
		if err != nil {
			return nil, fmt.Errorf("failed to marshal runtime options: %w", err)
		}
	}

	// Save sandbox name
	//在sandboxInfo中添加一个标签，标记沙盒的名称。
	sandboxInfo.AddLabel("name", name)

	// Create initial internal sandbox object.
	//创建一个新的沙盒对象，设置其元数据和状态。状态初始设置为未知，创建时间为当前时间。
	sandbox := sandboxstore.NewSandbox(
		sandboxstore.Metadata{
			ID:             id,
			Name:           name,
			Config:         config,
			RuntimeHandler: r.GetRuntimeHandler(),
		},
		sandboxstore.Status{
			State:     sandboxstore.StateUnknown,
			CreatedAt: time.Now().UTC(),
		},
	)
	sandbox.Sandboxer = ociRuntime.Sandboxer
	//尝试在沙盒存储中创建沙盒信息。如果创建失败，返回错误
	if _, err := c.client.SandboxStore().Create(ctx, sandboxInfo); err != nil {
		return nil, fmt.Errorf("failed to save sandbox metadata: %w", err)
	}
	//设置一个defer函数，用于在函数结束时，如果有错误返回且没有其他清理错误，执行沙盒信息的删除操作
	defer func() {
		if retErr != nil && cleanupErr == nil {
			cleanupErr = c.client.SandboxStore().Delete(ctx, id)
		}
	}()
	//用于在发生清理错误时将沙盒状态标记为未知并尝试将其重新加入沙盒存储。如果加入失败，则记录错误。
	defer func() {
		// Put the sandbox into sandbox store when some resources fail to be cleaned.
		if retErr != nil && cleanupErr != nil {
			log.G(ctx).WithError(cleanupErr).Errorf("encountered an error cleaning up failed sandbox %q, marking sandbox state as SANDBOX_UNKNOWN", id)
			//重新加入沙盒存储
			if err := c.sandboxStore.Add(sandbox); err != nil {
				log.G(ctx).WithError(err).Errorf("failed to add sandbox %+v into store", sandbox)
			}
		}
	}()

	// XXX: What we really want here is to call controller.Platform() and then check
	// platform.OS, but that is only populated after controller.Create() and that needs to be
	// done later (uses sandbox.NSPath that we will set just _after_ this).
	// So, lets check for the Linux section on the config, if that is populated, we assume the
	// platform is linux.
	// This is a hack, we should improve the controller interface to return the platform
	// earlier. But should work fine for this specific use.
	//这部分代码检查是否启用了用户命名空间（user namespace）。它首先获取Linux配置，并检查是否存在用户命名空间的设置，如果设置为POD模式，则启用用户命名空间。
	userNsEnabled := false
	if linux := config.GetLinux(); linux != nil {
		usernsOpts := linux.GetSecurityContext().GetNamespaceOptions().GetUsernsOptions()
		if usernsOpts != nil && usernsOpts.GetMode() == runtime.NamespaceMode_POD {
			userNsEnabled = true
		}
	}

	// Setup the network namespace if host networking wasn't requested.
	//如果沙盒不使用主机网络且未启用用户命名空间
	if !hostNetwork(config) && !userNsEnabled {
		// XXX: We do c&p of this code later for the podNetwork && userNsEnabled case too.
		// We can't move this to a function, as the defer calls need to be executed if other
		// errors are returned in this function. So, we would need more refactors to move
		// this code to a function and the idea was to not change the current code for
		// !userNsEnabled case, therefore doing it would defeat the purpose.
		//
		// The difference between the cases is the use of netns.NewNetNS() vs
		// netns.NewNetNSFromPID().
		//
		// To simplify this, in the future, we should just remove this case (podNetwork &&
		// !userNsEnabled) and just keep the other case (podNetwork && userNsEnabled).
		netStart := time.Now()
		// If it is not in host network namespace then create a namespace and set the sandbox
		// handle. NetNSPath in sandbox metadata and NetNS is non empty only for non host network
		// namespaces. If the pod is in host network namespace then both are empty and should not
		// be used.
		var netnsMountDir = "/var/run/netns"
		if c.config.NetNSMountsUnderStateDir {
			netnsMountDir = filepath.Join(c.config.StateDir, "netns")
		}
		//创建新的网络命名空间
		sandbox.NetNS, err = netns.NewNetNS(netnsMountDir)
		if err != nil {
			return nil, fmt.Errorf("failed to create network namespace for sandbox %q: %w", id, err)
		}
		// Update network namespace in the store, which is used to generate the container's spec
		sandbox.NetNSPath = sandbox.NetNS.GetPath()
		//发生错误时删除创建的网络命名空间。
		defer func() {
			// Remove the network namespace only if all the resource cleanup is done
			if retErr != nil && cleanupErr == nil {
				if cleanupErr = sandbox.NetNS.Remove(); cleanupErr != nil {
					log.G(ctx).WithError(cleanupErr).Errorf("Failed to remove network namespace %s for sandbox %q", sandbox.NetNSPath, id)
					return
				}
				//更新网络命名空间路径
				sandbox.NetNSPath = ""
			}
		}()
		//添加扩展信息。发生错误返回
		if err := sandboxInfo.AddExtension(podsandbox.MetadataKey, &sandbox.Metadata); err != nil {
			return nil, fmt.Errorf("unable to save sandbox %q to store: %w", id, err)
		}
		// Save sandbox metadata to store
		//更新沙盒信息，发生错误返回
		if sandboxInfo, err = c.client.SandboxStore().Update(ctx, sandboxInfo, "extensions"); err != nil {
			return nil, fmt.Errorf("unable to update extensions for sandbox %q: %w", id, err)
		}

		// Define this defer to teardownPodNetwork prior to the setupPodNetwork function call.
		// This is because in setupPodNetwork the resource is allocated even if it returns error, unlike other resource
		// creation functions.
		//清除网络，主要是为了setupPodNetwork函数调用，因为setupPodNetwork失败也会分配资源
		defer func() {
			// Remove the network namespace only if all the resource cleanup is done.
			if retErr != nil && cleanupErr == nil {
				deferCtx, deferCancel := util.DeferContext()
				defer deferCancel()
				// Teardown network if an error is returned.
				if cleanupErr = c.teardownPodNetwork(deferCtx, sandbox); cleanupErr != nil {
					log.G(ctx).WithError(cleanupErr).Errorf("Failed to destroy network for sandbox %q", id)
				}

			}
		}()

		// Setup network for sandbox.
		// Certain VM based solutions like clear containers (Issue containerd/cri-containerd#524)
		// rely on the assumption that CRI shim will not be querying the network namespace to check the
		// network states such as IP.
		// In future runtime implementation should avoid relying on CRI shim implementation details.
		// In this case however caching the IP will add a subtle performance enhancement by avoiding
		// calls to network namespace of the pod to query the IP of the veth interface on every
		// SandboxStatus request.
		//调用setupPodNetwork设置网络
		if err := c.setupPodNetwork(ctx, &sandbox); err != nil {
			return nil, fmt.Errorf("failed to setup network for sandbox %q: %w", id, err)
		}
		sandboxCreateNetworkTimer.UpdateSince(netStart)
	}
	//尝试创建沙盒，使用沙盒信息，配置选项和网络空间
	if err := sandboxInfo.AddExtension(podsandbox.MetadataKey, &sandbox.Metadata); err != nil {
		return nil, fmt.Errorf("unable to save sandbox %q to store: %w", id, err)
	}

	// Save sandbox metadata to store
	//更新沙盒存储信息，包含之前的扩展
	if sandboxInfo, err = c.client.SandboxStore().Update(ctx, sandboxInfo, "extensions"); err != nil {
		return nil, fmt.Errorf("unable to update extensions for sandbox %q: %w", id, err)
	}
	//尝试创建沙盒，使用沙盒信息、配置选项和网络命名空间路径
	if err := c.sandboxService.CreateSandbox(ctx, sandboxInfo, sb.WithOptions(config), sb.WithNetNSPath(sandbox.NetNSPath)); err != nil {
		return nil, fmt.Errorf("failed to create sandbox %q: %w", id, err)
	}
	//尝试启动沙盒
	ctrl, err := c.sandboxService.StartSandbox(ctx, sandbox.Sandboxer, id)
	if err != nil {
		var cerr podsandbox.CleanupErr
		if errors.As(err, &cerr) {
			cleanupErr = fmt.Errorf("failed to cleanup sandbox: %w", cerr)

			// Strip last error as cleanup error to handle separately
			//果错误类型为CleanupErr，则单独处理
			if merr, ok := err.(interface{ Unwrap() []error }); ok {
				if errs := merr.Unwrap(); len(errs) > 0 {
					err = errs[0]
				}
			}
		}
		return nil, fmt.Errorf("failed to start sandbox %q: %w", id, err)
	}
	//如果控制器返回的地址不为空，则将这个地址和版本信息设置为沙盒的端点
	if ctrl.Address != "" {
		sandbox.Endpoint = sandboxstore.Endpoint{
			Version: ctrl.Version,
			Address: ctrl.Address,
		}
	}
	//再次更新沙盒信息以包括任何新的扩展信息
	if sandboxInfo, err = c.client.SandboxStore().Update(ctx, sandboxInfo, "extensions"); err != nil {
		return nil, fmt.Errorf("unable to update extensions for sandbox %q: %w", id, err)
	}
	//如果启用了用户命名空间而没有使用主机网络
	if !hostNetwork(config) && userNsEnabled {
		// If userns is enabled, then the netns was created by the OCI runtime
		// on controller.Start(). The OCI runtime needs to create the netns
		// because, if userns is in use, the netns needs to be owned by the
		// userns. So, let the OCI runtime just handle this for us.
		// If the netns is not owned by the userns several problems will happen.
		// For instance, the container will lack permission (even if
		// capabilities are present) to modify the netns or, even worse, the OCI
		// runtime will fail to mount sysfs:
		//      https://github.com/torvalds/linux/commit/7dc5dbc879bd0779924b5132a48b731a0bc04a1e#diff-4839664cd0c8eab716e064323c7cd71fR1164
		//
		// Note we do this after controller.Start(), as before that we
		// can't get the PID for the sandbox that we need for the netns.
		// Doing a controller.Status() call before that fails (can't
		// find the sandbox) so we can't get the PID.
		netStart := time.Now()

		// If it is not in host network namespace then create a namespace and set the sandbox
		// handle. NetNSPath in sandbox metadata and NetNS is non empty only for non host network
		// namespaces. If the pod is in host network namespace then both are empty and should not
		// be used.
		//基于沙盒的进程ID创建一个新的网络命名空间，并设置相应的路径
		var netnsMountDir = "/var/run/netns"
		if c.config.NetNSMountsUnderStateDir {
			netnsMountDir = filepath.Join(c.config.StateDir, "netns")
		}

		sandbox.NetNS, err = netns.NewNetNSFromPID(netnsMountDir, ctrl.Pid)
		if err != nil {
			return nil, fmt.Errorf("failed to create network namespace for sandbox %q: %w", id, err)
		}

		// Update network namespace in the store, which is used to generate the container's spec
		sandbox.NetNSPath = sandbox.NetNS.GetPath()
		//如果创建失败，返回错误。另外，使用defer在出现错误时清理网络命名空间
		defer func() {
			// Remove the network namespace only if all the resource cleanup is done
			if retErr != nil && cleanupErr == nil {
				if cleanupErr = sandbox.NetNS.Remove(); cleanupErr != nil {
					log.G(ctx).WithError(cleanupErr).Errorf("Failed to remove network namespace %s for sandbox %q", sandbox.NetNSPath, id)
					return
				}
				sandbox.NetNSPath = ""
			}
		}()
		//添加扩展，错误返回
		if err := sandboxInfo.AddExtension(podsandbox.MetadataKey, &sandbox.Metadata); err != nil {
			return nil, fmt.Errorf("unable to save sandbox %q to store: %w", id, err)
		}
		// Save sandbox metadata to store
		//更新沙盒，错误返回
		if sandboxInfo, err = c.client.SandboxStore().Update(ctx, sandboxInfo, "extensions"); err != nil {
			return nil, fmt.Errorf("unable to update extensions for sandbox %q: %w", id, err)
		}

		// Define this defer to teardownPodNetwork prior to the setupPodNetwork function call.
		// This is because in setupPodNetwork the resource is allocated even if it returns error, unlike other resource
		// creation functions.
		//还是setup的清理函数，因为setupPodNetwork函数在返回错误时也会分配资源
		defer func() {
			// Remove the network namespace only if all the resource cleanup is done.
			if retErr != nil && cleanupErr == nil {
				deferCtx, deferCancel := util.DeferContext()
				defer deferCancel()
				// Teardown network if an error is returned.
				if cleanupErr = c.teardownPodNetwork(deferCtx, sandbox); cleanupErr != nil {
					log.G(ctx).WithError(cleanupErr).Errorf("Failed to destroy network for sandbox %q", id)
				}

			}
		}()

		// Setup network for sandbox.
		// Certain VM based solutions like clear containers (Issue containerd/cri-containerd#524)
		// rely on the assumption that CRI shim will not be querying the network namespace to check the
		// network states such as IP.
		// In future runtime implementation should avoid relying on CRI shim implementation details.
		// In this case however caching the IP will add a subtle performance enhancement by avoiding
		// calls to network namespace of the pod to query the IP of the veth interface on every
		// SandboxStatus request.
		//调用setupPodNetwork设置网络
		if err := c.setupPodNetwork(ctx, &sandbox); err != nil {
			return nil, fmt.Errorf("failed to setup network for sandbox %q: %w", id, err)
		}
		sandboxCreateNetworkTimer.UpdateSince(netStart)
	}
	//如果OCI运行时类型是Pod沙盒模式，尝试加载对应的容器。如果加载失败，返回错误。加载成功后，将容器实例关联到沙盒对象。
	// TODO: get rid of this. sandbox object should no longer have Container field.
	if ociRuntime.Sandboxer == string(criconfig.ModePodSandbox) {
		container, err := c.client.LoadContainer(ctx, id)
		if err != nil {
			return nil, fmt.Errorf("failed to load container %q for sandbox: %w", id, err)
		}
		sandbox.Container = container
	}
	//从控制器获取标签，如果没有标签，则初始化一个空的标签映射。
	labels := ctrl.Labels
	if labels == nil {
		labels = map[string]string{}
	}
	//从标签中获取SELinux标签并设置到沙盒的处理标签上。
	sandbox.ProcessLabel = labels["selinux_label"]
	//调用网络资源接口（NRI）来运行沙盒。如果运行失败，返回错误
	err = c.nri.RunPodSandbox(ctx, &sandbox)
	if err != nil {
		return nil, fmt.Errorf("NRI RunPodSandbox failed: %w", err)
	}
	//设置一个延迟执行函数，如果在执行过程中发生错误，将调用NRI来移除沙盒。
	defer func() {
		if retErr != nil {
			deferCtx, deferCancel := util.DeferContext()
			defer deferCancel()
			c.nri.RemovePodSandbox(deferCtx, &sandbox)
		}
	}()
	//更新沙盒的状态，设置进程ID、状态为就绪、创建时间。如果状态更新失败，返回错误。
	if err := sandbox.Status.Update(func(status sandboxstore.Status) (sandboxstore.Status, error) {
		// Set the pod sandbox as ready after successfully start sandbox container.
		status.Pid = ctrl.Pid
		status.State = sandboxstore.StateReady
		status.CreatedAt = ctrl.CreatedAt
		return status, nil
	}); err != nil {
		return nil, fmt.Errorf("failed to update sandbox status: %w", err)
	}

	// Add sandbox into sandbox store in INIT state .
	//将沙盒添加到沙盒存储中。如果添加失败，返回错误。
	if err := c.sandboxStore.Add(sandbox); err != nil {
		return nil, fmt.Errorf("failed to add sandbox %+v into store: %w", sandbox, err)
	}

	// Send CONTAINER_CREATED event with both ContainerId and SandboxId equal to SandboxId.
	// Note that this has to be done after sandboxStore.Add() because we need to get
	// SandboxStatus from the store and include it in the event.
	//生成并发送容器创建事件。事件类型为“容器已创建
	c.generateAndSendContainerEvent(ctx, id, id, runtime.ContainerEventType_CONTAINER_CREATED_EVENT)
	//调用沙盒服务的WaitSandbox方法等待沙盒退出。如果等待失败，返回错误。
	exitCh, err := c.sandboxService.WaitSandbox(util.NamespacedContext(), sandbox.Sandboxer, id)
	if err != nil {
		return nil, fmt.Errorf("failed to wait sandbox %s: %v", id, err)
	}

	// start the monitor after adding sandbox into the store, this ensures
	// that sandbox is in the store, when event monitor receives the TaskExit event.
	//
	// TaskOOM from containerd may come before sandbox is added to store,
	// but we don't care about sandbox TaskOOM right now, so it is fine.
	//开始监控沙盒退出。这保证在接收到任务退出事件时，沙盒已经在存储中。
	c.startSandboxExitMonitor(context.Background(), id, exitCh)
	//生成并发送容器启动事件。事件类型为“容器已启动”
	// Send CONTAINER_STARTED event with ContainerId equal to SandboxId.
	c.generateAndSendContainerEvent(ctx, id, id, runtime.ContainerEventType_CONTAINER_STARTED_EVENT)
	//更新创建沙盒的运行时计时器，以记录创建过程所需的时间
	sandboxRuntimeCreateTimer.WithValues(labels["oci_runtime_type"]).UpdateSince(runtimeStart)
	//成功执行所有操作后，返回包含沙盒ID的响应。
	return &runtime.RunPodSandboxResponse{PodSandboxId: id}, nil
}
```

##### 3.1.5 setupPodNetwork

**初始化变量**

- 初始化沙盒ID、配置、网络命名空间路径、网络插件等变量。

**检查网络插件是否初始化**

- 检查 `netPlugin` 是否为空，如果为空则返回错误。

**启用回环接口**

- 如果启用了内部回环接口，尝试在指定的网络命名空间中启用回环接口。

**获取 CNI 命名空间选项**

- 调用 `cniNamespaceOpts` 函数获取 CNI 命名空间选项。

**开始 CNI 设置**

- 记录日志，表明开始 CNI 设置过程。
- 根据配置选择以串行或并行方式调用网络插件的 `Setup` 方法来设置网络。

**记录和更新网络设置操作的指标**

- 记录网络设置操作的次数和耗时，如果有错误也记录错误操作次数。

**记录 CNI 结果**

- 调用 `logDebugCNIResult` 函数来记录 CNI 操作的结果。

**检查默认接口是否有 IP 配置**

- 检查 CNI 操作的结果中是否包含默认网络接口的 IP 配置，如果有则选择主IP和附加IP，保存到沙盒对象中。

**返回结果或错误**

- 如果没有找到网络信息，则返回错误。

==具体初始化根据CNI插件实现==

```go
// setupPodNetwork setups up the network for a pod
func (c *criService) setupPodNetwork(ctx context.Context, sandbox *sandboxstore.Sandbox) error {
	/*
		id：沙盒的唯一标识符。
		config：沙盒的配置信息。
		path：沙盒的网络命名空间路径。
		netPlugin：获取与运行时处理器关联的网络插件。  比如自己写的CNI或者Calico，Flannel
		err：用于捕获错误。
		result：用于存储CNI（容器网络接口）操作的结果。
	*/
	var (
		id        = sandbox.ID
		config    = sandbox.Config
		path      = sandbox.NetNSPath
		netPlugin = c.getNetworkPlugin(sandbox.RuntimeHandler)
		err       error
		result    *cni.Result
	)
	//则表示网络插件未初始化，返回错误
	if netPlugin == nil {
		return errors.New("cni config not initialized")
	}
	//如果启用了回环接口
	if c.config.UseInternalLoopback {
		//尝试在指定的网络命名空间中启用回环接口
		err := c.bringUpLoopback(path)
		if err != nil {
			return fmt.Errorf("unable to set lo to up: %w", err)
		}
	}
	//调用cniNamespaceOpts函数获取CNI命名空间选项
	opts, err := cniNamespaceOpts(id, config)
	if err != nil {
		return fmt.Errorf("get cni namespace options: %w", err)
	}
	//记录日志，表明开始CNI（容器网络接口）的设置过程。
	log.G(ctx).WithField("podsandboxid", id).Debugf("begin cni setup")
	netStart := time.Now()
	//配置选择以串行或并行方式调用网络插件的Setup方法来设置网络
	if c.config.CniConfig.NetworkPluginSetupSerially {
		result, err = netPlugin.SetupSerially(ctx, id, path, opts...)
	} else {
		result, err = netPlugin.Setup(ctx, id, path, opts...)
	}
	//指标（metric）来记录网络设置操作的次数+1
	networkPluginOperations.WithValues(networkSetUpOp).Inc()
	//记录网络设置操作的耗时
	networkPluginOperationsLatency.WithValues(networkSetUpOp).UpdateSince(netStart)
	if err != nil {
		//记录网络错误操作+1
		networkPluginOperationsErrors.WithValues(networkSetUpOp).Inc()
		return err
	}
	//调用logDebugCNIResult函数来记录CNI（容器网络接口）操作的结果
	logDebugCNIResult(ctx, id, result)
	// Check if the default interface has IP config
	//检查CNI操作的结果中是否包含默认网络接口的IP配置，并且至少包含一个IP配置
	if configs, ok := result.Interfaces[defaultIfName]; ok && len(configs.IPConfigs) > 0 {
		//selectPodIPs选择Pod的主IP地址和其他附加IP地址（包含IPV4和IPV6），保存在沙盒对象中
		sandbox.IP, sandbox.AdditionalIPs = selectPodIPs(ctx, configs.IPConfigs, c.config.IPPreference)
		sandbox.CNIResult = result
		return nil
	}
	//返回错误
	return fmt.Errorf("failed to find network info for sandbox %q", id)
}

```

#### 3.1.5 Start

**清理函数定义**

- 定义延迟执行的清理函数，以便在函数返回前进行资源清理。

**获取沙盒实例**

- 通过 `id` 获取沙盒实例 `podSandbox`，并获取元数据 `metadata`。

**获取沙盒镜像**

- 调用 `getSandboxImageName` 获取沙盒容器的镜像名称。
- 确保沙盒镜像存在，调用 `ensureImageExists` 进行镜像检查和下载。
- 将镜像信息转换为 containerd 格式，调用 `toContainerdImage`。

**获取 OCI 运行时配置**

- 获取适合当前沙盒的 OCI 运行时配置，调用 `GetSandboxRuntime`。

**创建沙盒容器规范**

- 生成沙盒容器的规范，包括容器的网络设置、挂载点、环境变量等，调用 `sandboxContainerSpec`。
- 修改 SELinux 标签，调用 `modifyProcessLabel`。
- 如果沙盒配置为特权模式，清空 SELinux 标签。

**生成容器规范选项**

- 生成容器创建时应用的额外规范选项，调用 `sandboxContainerSpecOpts`。

**创建快照和容器**

- 创建快照选项，调用 `sandboxSnapshotterOpts`。
- 创建新的 containerd 容器，调用 `client.NewContainer`。

**创建沙盒根目录**

- 创建沙盒的根目录和临时性目录，调用 `os.MkdirAll`。

**设置沙盒文件**

- 设置沙盒所需的文件，调用 `setupSandboxFiles`。

**获取容器信息**

- 获取容器信息，包括时间戳，调用 `container.Info`。

**创建并启动沙盒任务**

- 创建新任务，调用 `container.NewTask`。
- 等待沙盒任务的结束，调用 `task.Wait`。
- 启动沙盒容器任务，调用 `task.Start`。

**NRI 沙盒创建**

- 创建 NRI 客户端，调用 `nri.New`。
- 如果 NRI 客户端存在，调用 `nric.InvokeWithSandbox` 触发自定义逻辑。

**更新沙盒状态**

- 更新沙盒的状态，包括进程ID、状态和创建时间，调用 `podSandbox.Status.Update`。

**返回结果并启动退出监控**

- 设置返回对象的属性，并启动一个 goroutine 来等待沙盒容器任务退出，调用 `waitSandboxExit`。

```go
// Start creates resources required for the sandbox and starts the sandbox.  If an error occurs, Start attempts to tear
// down the created resources.  If an error occurs while tearing down resources, a zero-valued response is returned
// alongside the error.  If the teardown was successful, a nil response is returned with the error.
// TODO(samuelkarp) Determine whether this error indication is reasonable to retain once controller.Delete is implemented.
func (c *Controller) Start(ctx context.Context, id string) (cin sandbox.ControllerInstance, retErr error) {
	var cleanupErr error
	//清理函数，用于在函数返回前执行，失败的时候执行
	defer func() {
		if retErr != nil && cleanupErr != nil {
			log.G(ctx).WithField("id", id).WithError(cleanupErr).Errorf("failed to fully teardown sandbox resources after earlier error: %s", retErr)
			retErr = errors.Join(retErr, CleanupErr{cleanupErr})
		}
	}()
	//过 id 获取沙盒实例
	podSandbox := c.store.Get(id)
	if podSandbox == nil {
		return cin, fmt.Errorf("unable to find pod sandbox with id %q: %w", id, errdefs.ErrNotFound)
	}
	//从podSandbox获取元数据
	metadata := podSandbox.Metadata
	//提取沙盒的配置，并初始化一个空的标签映射。这些标签可能在后续操作中用于标记容器或其他资源。
	var (
		config = metadata.Config
		labels = map[string]string{}
	)
	//调用 getSandboxImageName 方法获取沙盒容器所需的镜像名称
	sandboxImage := c.getSandboxImageName()
	// Ensure sandbox container image snapshot.
	//确保所需的沙盒镜像存在。如果镜像不存在，可能需要下载或构建。
	image, err := c.ensureImageExists(ctx, sandboxImage, config, metadata.RuntimeHandler)
	if err != nil {
		return cin, fmt.Errorf("failed to get sandbox image %q: %w", sandboxImage, err)
	}
	//将镜像信息转换为 containerd 可以使用的格式
	containerdImage, err := c.toContainerdImage(ctx, *image)
	if err != nil {
		return cin, fmt.Errorf("failed to get image from containerd %q: %w", image.ID, err)
	}
	//从配置中获取适合当前沙盒的 OCI 运行时配置。RuntimeHandler 可能是指定使用特定的运行时环境。
	ociRuntime, err := c.config.GetSandboxRuntime(config, metadata.RuntimeHandler)
	if err != nil {
		return cin, fmt.Errorf("failed to get sandbox runtime: %w", err)
	}
	log.G(ctx).WithField("podsandboxid", id).Debugf("use OCI runtime %+v", ociRuntime)
	//将运行时类型添加到标签中，以便在创建容器时使用
	labels["oci_runtime_type"] = ociRuntime.Type

	// Create sandbox container.
	// NOTE: sandboxContainerSpec SHOULD NOT have side
	// effect, e.g. accessing/creating files, so that we can test
	// it safely.
	//c.sandboxContainerSpec 方法创建沙盒容器的规范，包括容器的网络设置、挂载点、环境变量等。如果生成失败，则返回错误。
	spec, err := c.sandboxContainerSpec(id, config, &image.ImageSpec.Config, metadata.NetNSPath, ociRuntime.PodAnnotations)
	if err != nil {
		return cin, fmt.Errorf("failed to generate sandbox container spec: %w", err)
	}
	log.G(ctx).WithField("podsandboxid", id).Debugf("sandbox container spec: %#+v", spew.NewFormatter(spec))
	// SELinux 标签保存到标签映射中，以便以后可能使用
	metadata.ProcessLabel = spec.Process.SelinuxLabel
	defer func() {
		if retErr != nil {
			selinux.ReleaseLabel(metadata.ProcessLabel)
		}
	}()
	labels["selinux_label"] = metadata.ProcessLabel

	// handle any KVM based runtime
	//可能对进程标签进行修改，具体取决于 OCI 运行时的类型
	if err := modifyProcessLabel(ociRuntime.Type, spec); err != nil {
		return cin, err
	}
	//如果沙盒配置为特权模式，将清空 SELinux 标签，因为特权容器通常不受 SELinux 策略的限制
	if config.GetLinux().GetSecurityContext().GetPrivileged() {
		// If privileged don't set selinux label, but we still record the MCS label so that
		// the unused label can be freed later.
		spec.Process.SelinuxLabel = ""
	}

	// Generate spec options that will be applied to the spec later.
	//这段代码生成容器创建时应用的额外规范选项，比如安全设置、资源限制等。如果生成失败，则返回错误。
	specOpts, err := c.sandboxContainerSpecOpts(config, &image.ImageSpec.Config)
	if err != nil {
		return cin, fmt.Errorf("failed to generate sandbox container spec options: %w", err)
	}
	//合并沙盒配置中的标签和镜像规范中的标签，生成最终的容器标签
	sandboxLabels := buildLabels(config.Labels, image.ImageSpec.Config.Labels, crilabels.ContainerKindSandbox)
	//创建快照切片，其实包含一个过滤标签选项
	snapshotterOpt := []snapshots.Opt{snapshots.WithLabels(snapshots.FilterInheritedLabels(config.Annotations))}
	//获取额外的快照选项
	extraSOpts, err := sandboxSnapshotterOpts(config)
	if err != nil {
		return cin, err
	}
	//将额外的快照选项合并到快照切片中
	snapshotterOpt = append(snapshotterOpt, extraSOpts...)
	//创建一个新的 containerd 容器选项的切片
	opts := []containerd.NewContainerOpts{
		containerd.WithSnapshotter(c.imageService.RuntimeSnapshotter(ctx, ociRuntime)),
		customopts.WithNewSnapshot(id, containerdImage, snapshotterOpt...),
		containerd.WithSpec(spec, specOpts...),
		containerd.WithContainerLabels(sandboxLabels),
		containerd.WithContainerExtension(crilabels.SandboxMetadataExtension, &metadata),
		containerd.WithRuntime(ociRuntime.Type, podSandbox.Runtime.Options),
	}
	//创建容器实列
	container, err := c.client.NewContainer(ctx, id, opts...)
	if err != nil {
		return cin, fmt.Errorf("failed to create containerd container: %w", err)
	}
	podSandbox.Container = container
	//延迟清理函数
	defer func() {
		if retErr != nil && cleanupErr == nil {
			deferCtx, deferCancel := ctrdutil.DeferContext()
			defer deferCancel()
			if cleanupErr = container.Delete(deferCtx, containerd.WithSnapshotCleanup); cleanupErr != nil {
				log.G(ctx).WithError(cleanupErr).Errorf("Failed to delete containerd container %q", id)
			}
			podSandbox.Container = nil
		}
	}()

	// Create sandbox container root directories.
	sandboxRootDir := c.getSandboxRootDir(id)
	if err := c.os.MkdirAll(sandboxRootDir, 0755); err != nil {
		return cin, fmt.Errorf("failed to create sandbox root directory %q: %w",
			sandboxRootDir, err)
	}
	defer func() {
		if retErr != nil && cleanupErr == nil {
			// Cleanup the sandbox root directory.
			if cleanupErr = c.os.RemoveAll(sandboxRootDir); cleanupErr != nil {
				log.G(ctx).WithError(cleanupErr).Errorf("Failed to remove sandbox root directory %q",
					sandboxRootDir)
			}
		}
	}()
	//创建沙盒的根目录
	volatileSandboxRootDir := c.getVolatileSandboxRootDir(id)
	if err := c.os.MkdirAll(volatileSandboxRootDir, 0755); err != nil {
		return cin, fmt.Errorf("failed to create volatile sandbox root directory %q: %w",
			volatileSandboxRootDir, err)
	}
	//清理函数，发生错误将尝试删除创建的根目录
	defer func() {
		if retErr != nil && cleanupErr == nil {
			// Cleanup the volatile sandbox root directory.
			if cleanupErr = c.os.RemoveAll(volatileSandboxRootDir); cleanupErr != nil {
				log.G(ctx).WithError(cleanupErr).Errorf("Failed to remove volatile sandbox root directory %q",
					volatileSandboxRootDir)
			}
		}
	}()

	// Setup files required for the sandbox.
	//创建沙盒的零时性目录
	if err = c.setupSandboxFiles(id, config); err != nil {
		return cin, fmt.Errorf("failed to setup sandbox files: %w", err)
	}
	//清理函数，发生错误将尝试删除创建的临时目录
	defer func() {
		if retErr != nil && cleanupErr == nil {
			if cleanupErr = c.cleanupSandboxFiles(id, config); cleanupErr != nil {
				log.G(ctx).WithError(cleanupErr).Errorf("Failed to cleanup sandbox files in %q",
					sandboxRootDir)
			}
		}
	}()

	// Update sandbox created timestamp.
	//码从 container 对象中获取容器的信息，这可能包括容器的配置、状态和其他元数据。包括时间戳
	info, err := container.Info(ctx)
	if err != nil {
		return cin, fmt.Errorf("failed to get sandbox container info: %w", err)
	}

	// Create sandbox task in containerd.
	log.G(ctx).Tracef("Create sandbox container (id=%q, name=%q).", id, metadata.Name)
	//创建新任务的选项数组
	var taskOpts []containerd.NewTaskOpts
	//如果为 OCI 运行时指定了一个路径 (ociRuntime.Path), 这个路径将作为一个选项被加入到任务选项中。这允许配置任务使用特定的运行时执行环境，而不是默认的环境
	if ociRuntime.Path != "" {
		taskOpts = append(taskOpts, containerd.WithRuntimePath(ociRuntime.Path))
	}

	// We don't need stdio for sandbox container.
	//创建了一个新的任务，它将运行沙盒容器
	task, err := container.NewTask(ctx, containerdio.NullIO, taskOpts...)
	if err != nil {
		return cin, fmt.Errorf("failed to create containerd task: %w", err)
	}
	//清理函数，清理任务
	defer func() {
		if retErr != nil && cleanupErr == nil {
			deferCtx, deferCancel := ctrdutil.DeferContext()
			defer deferCancel()
			// Cleanup the sandbox container if an error is returned.
			if _, err := task.Delete(deferCtx, WithNRISandboxDelete(id), containerd.WithProcessKill); err != nil && !errdefs.IsNotFound(err) {
				log.G(ctx).WithError(err).Errorf("Failed to delete sandbox container %q", id)
				cleanupErr = err
			}
		}
	}()

	// wait is a long running background request, no timeout needed.
	//调用task.Wait等待沙盒容器任务的结束，并返回一个通道，exitCh等待任务结束的信号
	exitCh, err := task.Wait(ctrdutil.NamespacedContext())
	if err != nil {
		return cin, fmt.Errorf("failed to wait for sandbox container task: %w", err)
	}
	//创建一个新的NRI客户端。NRI是用于在创建、更新或删除容器时插入自定义逻辑的接口。
	nric, err := nri.New()
	if err != nil {
		return cin, fmt.Errorf("unable to create nri client: %w", err)
	}
	//如果NRI客户端创建成功，将构建一个NRI沙盒对象并调用InvokeWithSandbox，以便在创建沙盒的过程中触发自定义逻辑。
	//这可能涉及到资源分配、安全配置或其他插件指定的任务。如果调用失败，函数返回错误。
	if nric != nil {
		nriSB := &nri.Sandbox{
			ID:     id,
			Labels: config.Labels,
		}
		if _, err := nric.InvokeWithSandbox(ctx, task, v1.Create, nriSB); err != nil {
			return cin, fmt.Errorf("nri invoke: %w", err)
		}
	}
	//这行代码启动沙盒容器的任务。
	if err := task.Start(ctx); err != nil {
		return cin, fmt.Errorf("failed to start sandbox container task %q: %w", id, err)
	}
	//这部分代码获取任务的进程ID，并更新沙盒的状态，包括进程ID、状态和创建时间。如果状态更新失败，将返回错误。
	pid := task.Pid()
	if err := podSandbox.Status.Update(func(status sandboxstore.Status) (sandboxstore.Status, error) {
		status.Pid = pid
		status.State = sandboxstore.StateReady
		status.CreatedAt = info.CreatedAt
		return status, nil
	}); err != nil {
		return cin, fmt.Errorf("failed to update status of pod sandbox %q: %w", id, err)
	}
	//这些行设置返回对象cin的各种属性，包括沙盒ID、进程ID、创建时间和标签。
	cin.SandboxID = id
	cin.Pid = task.Pid()
	cin.CreatedAt = info.CreatedAt
	cin.Labels = labels
	//启动一个goroutine来等待沙盒容器任务退出，并更新沙盒的状态。
	go func() {
		if err := c.waitSandboxExit(ctrdutil.NamespacedContext(), podSandbox, exitCh); err != nil {
			log.G(context.Background()).Warnf("failed to wait pod sandbox exit %v", err)
		}
	}()

	return
}
```

#### 3.1.6 总结

创建sandbox就是 启动了pause容器，同时通过cni 创建了网络资源

### 3.2 start init/业务容器做了什么操作

#### 3.2.1 start

start init/业务容器其实都是一样的，只不过是顺序不同而言。先init 容器，再业务容器。具体是调用start函数。

```go
	// 定义一个辅助函数，用于启动各类型容器，例如常规容器、初始容器和临时容器
	start := func(ctx context.Context, typeName, metricLabel string, spec *startSpec) error {
		startContainerResult := kubecontainer.NewSyncResult(kubecontainer.StartContainer, spec.container.Name)
		result.AddSyncResult(startContainerResult)
		//  检查容器是否处于退避状态,就是一直发生错误多次重启，backoff就是暂时停止重启
		//// pod: 待检查的Pod对象
		//// container: 待检查的容器对象
		//// podStatus: Pod的容器状态信息
		//// backOff: 退避控制对象
		isInBackOff, msg, err := m.doBackOff(pod, spec.container, podStatus, backOff)
		if isInBackOff {
			startContainerResult.Fail(err, msg)
			klog.V(4).InfoS("Backing Off restarting container in pod", "containerType", typeName, "container", spec.container, "pod", klog.KObj(pod))
			return err
		}
		// 监控指标
		metrics.StartedContainersTotal.WithLabelValues(metricLabel).Inc()
		if sc.HasWindowsHostProcessRequest(pod, spec.container) {
			metrics.StartedHostProcessContainersTotal.WithLabelValues(metricLabel).Inc()
		}
		klog.V(4).InfoS("Creating container in pod", "containerType", typeName, "container", spec.container, "pod", klog.KObj(pod))
		// NOTE (aramase) podIPs are populated for single stack and dual stack clusters. Send only podIPs.
		// 启动容器
		if msg, err := m.startContainer(ctx, podSandboxID, podSandboxConfig, spec, pod, podStatus, pullSecrets, podIP, podIPs); err != nil {
			// startContainer() returns well-defined error codes that have reasonable cardinality for metrics and are
			// useful to cluster administrators to distinguish "server errors" from "user errors".
			// 记录容器启动失败的详细信息。
			metrics.StartedContainersErrorsTotal.WithLabelValues(metricLabel, err.Error()).Inc()
			//发丝只有podIPs的信息
			if sc.HasWindowsHostProcessRequest(pod, spec.container) {
				metrics.StartedHostProcessContainersErrorsTotal.WithLabelValues(metricLabel, err.Error()).Inc()
			}
			startContainerResult.Fail(err, msg)
			// known errors that are logged in other places are logged at higher levels here to avoid
			// repetitive log spam
			// 根据错误类型记录不同级别的日志。
			switch {
			case err == images.ErrImagePullBackOff:
				klog.V(3).InfoS("Container start failed in pod", "containerType", typeName, "container", spec.container, "pod", klog.KObj(pod), "containerMessage", msg, "err", err)
			default:
				utilruntime.HandleError(fmt.Errorf("%v %+v start failed in pod %v: %v: %s", typeName, spec.container, format.Pod(pod), err, msg))
			}
			return err
		}

		return nil
	}
```

#### 3.2.2 startContainer

1. 拉取镜像 (Pull the image)

- **目的**：确保容器镜像在本地存在。
- 操作：
    - 检查并获取 `RuntimeClass`（如果启用了 `RuntimeClassInImageCriAPI` 特性）。
    - 调用 `EnsureImageExists` 确保指定的容器镜像存在于本地。如果镜像不存在，进行拉取操作。
- 错误处理：
    - 如果拉取镜像失败，记录相应的事件并返回错误信息。

2. 创建容器 (Create the container)

- **目的**：根据配置和镜像创建一个新的容器实例。
- 操作：
    - 确定容器的重启次数 (`restartCount`)。
    - 生成容器配置 (`generateContainerConfig`)。
    - 执行内部生命周期钩子函数 `PreCreateContainer`。
    - 调用 `CreateContainer` 在运行时中创建容器。
    - 执行内部生命周期钩子函数 `PreStartContainer`。
    - 记录容器创建事件。
- 错误处理：
    - 如果任意一步失败，记录相应的事件并返回错误信息。

3. 启动容器 (Start the container)

- **目的**：启动已经创建的容器实例。
- 操作：
    - 调用 `StartContainer` 启动容器。
    - 记录容器启动事件。
    - 创建符号链接以支持旧版集群日志记录。
- 错误处理：
    - 如果启动容器失败，记录相应的事件并返回错误信息。

4. 执行启动后钩子函数 (Run the post start lifecycle hooks)

- **目的**：在容器启动后执行特定的生命周期钩子函数。
- 操作：
    - 如果容器定义了 `PostStart` 钩子函数，调用 `Run` 执行该钩子函数。
- 错误处理：
    - 如果钩子函数执行失败，记录相应的事件并尝试终止容器，返回错误信息。

```go
// startContainer starts a container and returns a message indicates why it is failed on error.
// It starts the container through the following steps:
// * pull the image
// * create the container
// * start the container
// * run the post start lifecycle hooks (if applicable)
func (m *kubeGenericRuntimeManager) startContainer(ctx context.Context, podSandboxID string, podSandboxConfig *runtimeapi.PodSandboxConfig, spec *startSpec, pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, podIP string, podIPs []string) (string, error) {
	container := spec.container

	// Step 1: pull the image.
	// 拉取镜像
	// If RuntimeClassInImageCriAPI feature gate is enabled, pass runtimehandler
	// information for the runtime class specified. If not runtime class is
	// specified, then pass ""
	// 如果启用了 RuntimeClassInImageCriAPI 特性开关，则传递指定运行时类的信息。如果没有指定运行时类，则传递空字符串
	podRuntimeHandler := ""
	var err error
	if utilfeature.DefaultFeatureGate.Enabled(features.RuntimeClassInImageCriAPI) {
		if pod.Spec.RuntimeClassName != nil && *pod.Spec.RuntimeClassName != "" {
			podRuntimeHandler, err = m.runtimeClassManager.LookupRuntimeHandler(pod.Spec.RuntimeClassName)
			if err != nil {
				msg := fmt.Sprintf("Failed to lookup runtimeHandler for runtimeClassName %v", pod.Spec.RuntimeClassName)
				return msg, err
			}
		}
	}

	// 确保镜像存在，如果镜像不存在，则进行拉取
	imageRef, msg, err := m.imagePuller.EnsureImageExists(ctx, pod, container, pullSecrets, podSandboxConfig, podRuntimeHandler)
	if err != nil {
		s, _ := grpcstatus.FromError(err)
		m.recordContainerEvent(pod, container, "", v1.EventTypeWarning, events.FailedToCreateContainer, "Error: %v", s.Message())
		return msg, err
	}

	// Step 2: create the container.
	// 创建容器
	// For a new container, the RestartCount should be 0
	// 对于一个新的容器，RestartCount 应为 0
	restartCount := 0
	containerStatus := podStatus.FindContainerStatusByName(container.Name)
	if containerStatus != nil {
		restartCount = containerStatus.RestartCount + 1
	} else {
		// The container runtime keeps state on container statuses and
		// what the container restart count is. When nodes are rebooted
		// some container runtimes clear their state which causes the
		// restartCount to be reset to 0. This causes the logfile to
		// start at 0.log, which either overwrites or appends to the
		// already existing log.
		// 
		// We are checking to see if the log directory exists, and find
		// the latest restartCount by checking the log name -
		// {restartCount}.log - and adding 1 to it.
		// 容器运行时会保留容器状态及其重启次数。当节点重新启动时，有些容器运行时会清除其状态，导致重启计数重置为 0。这会导致日志文件从 0.log 开始，要么覆盖，要么附加到已存在的日志文件中。
		// 我们检查日志目录是否存在，并通过检查日志名称（{restartCount}.log）找到最新的重启计数，并加 1。
		logDir := BuildContainerLogsDirectory(m.podLogsDirectory, pod.Namespace, pod.Name, pod.UID, container.Name)
		restartCount, err = calcRestartCountByLogDir(logDir)
		if err != nil {
			klog.InfoS("Cannot calculate restartCount from the log directory", "logDir", logDir, "err", err)
			restartCount = 0
		}
	}

	// 获取目标 ID
	target, err := spec.getTargetID(podStatus)
	if err != nil {
		s, _ := grpcstatus.FromError(err)
		m.recordContainerEvent(pod, container, "", v1.EventTypeWarning, events.FailedToCreateContainer, "Error: %v", s.Message())
		return s.Message(), ErrCreateContainerConfig
	}

	// 生成容器配置
	containerConfig, cleanupAction, err := m.generateContainerConfig(ctx, container, pod, restartCount, podIP, imageRef, podIPs, target)
	if cleanupAction != nil {
		defer cleanupAction()
	}
	if err != nil {
		s, _ := grpcstatus.FromError(err)
		m.recordContainerEvent(pod, container, "", v1.EventTypeWarning, events.FailedToCreateContainer, "Error: %v", s.Message())
		return s.Message(), ErrCreateContainerConfig
	}

	// 执行内部生命周期钩子函数（创建前）
	err = m.internalLifecycle.PreCreateContainer(pod, container, containerConfig)
	if err != nil {
		s, _ := grpcstatus.FromError(err)
		m.recordContainerEvent(pod, container, "", v1.EventTypeWarning, events.FailedToCreateContainer, "Internal PreCreateContainer hook failed: %v", s.Message())
		return s.Message(), ErrPreCreateHook
	}

	// 创建容器
	containerID, err := m.runtimeService.CreateContainer(ctx, podSandboxID, containerConfig, podSandboxConfig)
	if err != nil {
		s, _ := grpcstatus.FromError(err)
		m.recordContainerEvent(pod, container, containerID, v1.EventTypeWarning, events.FailedToCreateContainer, "Error: %v", s.Message())
		return s.Message(), ErrCreateContainer
	}

	// 执行内部生命周期钩子函数（启动前）
	err = m.internalLifecycle.PreStartContainer(pod, container, containerID)
	if err != nil {
		s, _ := grpcstatus.FromError(err)
		m.recordContainerEvent(pod, container, containerID, v1.EventTypeWarning, events.FailedToStartContainer, "Internal PreStartContainer hook failed: %v", s.Message())
		return s.Message(), ErrPreStartHook
	}

	// 记录容器创建事件
	m.recordContainerEvent(pod, container, containerID, v1.EventTypeNormal, events.CreatedContainer, fmt.Sprintf("Created container %s", container.Name))

	// Step 3: start the container.
	// 启动容器
	err = m.runtimeService.StartContainer(ctx, containerID)
	if err != nil {
		s, _ := grpcstatus.FromError(err)
		m.recordContainerEvent(pod, container, containerID, v1.EventTypeWarning, events.FailedToStartContainer, "Error: %v", s.Message())
		return s.Message(), kubecontainer.ErrRunContainer
	}

	// 记录容器启动事件
	m.recordContainerEvent(pod, container, containerID, v1.EventTypeNormal, events.StartedContainer, fmt.Sprintf("Started container %s", container.Name))

	// Symlink container logs to the legacy container log location for cluster logging
	// support.
	// TODO(random-liu): Remove this after cluster logging supports CRI container log path.
	// 创建容器日志的符号链接，以支持集群日志记录
	containerMeta := containerConfig.GetMetadata()
	sandboxMeta := podSandboxConfig.GetMetadata()
	legacySymlink := legacyLogSymlink(containerID, containerMeta.Name, sandboxMeta.Name, sandboxMeta.Namespace)
	containerLog := filepath.Join(podSandboxConfig.LogDirectory, containerConfig.LogPath)
	// 仅在容器日志路径存在时创建旧版符号链接（或错误不为 IsNotExist）。因为如果容器日志路径不存在，只会创建悬空的旧版符号链接。此悬空的旧版符号链接稍后会被容器 GC 删除，因此根本没有意义创建它。
	if _, err := m.osInterface.Stat(containerLog); !os.IsNotExist(err) {
		if err := m.osInterface.Symlink(containerLog, legacySymlink); err != nil {
			klog.ErrorS(err, "Failed to create legacy symbolic link", "path", legacySymlink, "containerID", containerID, "containerLogPath", containerLog)
		}
	}

	// Step 4: execute the post start hook.
	// 执行启动后钩子函数
	if container.Lifecycle != nil && container.Lifecycle.PostStart != nil {
		kubeContainerID := kubecontainer.ContainerID{
			Type: m.runtimeName,
			ID:   containerID,
		}
		msg, handlerErr := m.runner.Run(ctx, kubeContainerID, pod, container, container.Lifecycle.PostStart)
		if handlerErr != nil {
			klog.ErrorS(handlerErr, "Failed to execute PostStartHook", "pod", klog.KObj(pod), "podUID", pod.UID, "containerName", container.Name, "containerID", kubeContainerID.String())
			// do not record the message in the event so that secrets won't leak from the server.
			// 不记录消息在事件中，以免泄露服务器的秘密
			m.recordContainerEvent(pod, container, kubeContainerID.ID, v1.EventTypeWarning, events.FailedPostStartHook, "PostStartHook failed")
			if err := m.killContainer(ctx, pod, kubeContainerID, container.Name, "FailedPostStartHook", reasonFailedPostStartHook, nil, nil); err != nil {
				klog.ErrorS(err, "Failed to kill container", "pod", klog.KObj(pod), "podUID", pod.UID, "containerName", container.Name, "containerID", kubeContainerID.String())
			}
			return msg, ErrPostStartHook
		}
	}

	return "", nil
}

```

### 4.总结



到这里为止，Pod在kubelet的创建流程就清楚了。了解整个过程会对排查问题，优化调度有所帮助。

比如看到pod有开始拉取业务容器image的动作时，可以确定，网络初始化是已经完成了，如果这个时候发现pod没有podip，那肯定就是有问题的。

这里再次初略总结一下整个流程：

（1）kubelet监听pod的创建，然后进行处理

（2）更新Pod应有的status，然后调用containerRuntime.SyncPod 完成 创建sandbox, 启动容器等过程, 以达到pod的期望状态

（3）先创建了pod 的目录包括 log，volume目录等等。然后先启动sandbox，启动这个后会完成网络的初始化

（4）成功启动sandbox后，在依次启动Init容器，业务容器。启动的过程是先拉取镜像，然后再create container， start contaienr
