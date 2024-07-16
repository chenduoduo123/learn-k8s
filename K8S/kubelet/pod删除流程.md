# 1.背景

当一个pod删除时，client端向apiserver发送请求，apiserver将pod的deletionTimestamp打上时间。kubelet watch到该事件，开始处理。所以一开始 kubele监听到的其实是 update事件。

所以通过分析kubelet delete其实也是分析了 apiserver更新pod流程。所以不单独写 apiserver更新pod，kubelet的流程处理了。

HandlePodUpdates-----> podWorkers.UpdatePod----->podWorkerLoop----->SyncPod

```go
//KillPodOptions是pod终止结构体

type KillPodOptions struct {
	//一个可选字段，用于指定 Pod 终止时的宽限期。如果设置了这个字段，Kubelet 会使用这个宽限期而不是 Pod 自己的宽限期。
    PodTermination  GracePeriodSecondsOverride *int64

    // 是一个通道，表示终止操作完成时的通知机制。如果存在这个通道，在 Pod 终止操作完成后，Kubelet 会关闭这个通道，通知等待终止操作完成的其他部分。
    CompletedCh chan struct{}

    // 用于提供 Pod 的状态更新。如果指定了这个函数，在 Pod 被终止时，Kubelet 会调用这个函数来设置 Pod 的状态。
    PodStatusFunc func() v1.PodStatus

    //指示这个终止操作是否是由于驱逐引起的
    Evict bool
 
}

```

podWorkers.UpdatePod函数不会立即删除POD，因为POD打上deletionTimestamp时间，不代表Pod里面的容器已经全部停止运行了，

因为status.IsTerminationRequested判断podSyncStatus结构体中的是否terminatingAt是非零，并不是有了DeletionTimestamp就会认为是Terminated状态，而是有DeletionTimestamp且所有的容器不在运行了才是Terminated状态

```go
type podSyncStatus struct {
	ctx context.Context
	// 取消当前 pod 同步操作的函数
	cancelFn context.CancelFunc
	//  Pod 的名称和命名空间，唯一标识一个 Pod
	fullname string
	//表示是否有待处理或正在处理的 pod 更新
	working bool
	//待处理的 pod 更新状态
	pendingUpdate *UpdatePodOptions
	//当前正在处理的 pod 状态
	activeUpdate *UpdatePodOptions   
	//记录 pod 被工作协程首次处理的时间
	syncedAt time.Time
	// 记录 pod 实际启动的时间
	startedAt time.Time
	// 记录 pod 请求终止的时间。在 pod 工作协程开始终止之前可能已经设置。
	terminatingAt time.Time
	//  记录所有运行中的容器停止的时间，标志着 pod 完全终止。
	terminatedAt time.Time
	//请求的宽限期，一旦 terminatingAt 非零
	gracePeriod int64
	// : pod 终止后发送通知的通道列表
	notifyPostTerminating []chan<- struct{}
	// 与终止请求相关的状态变更函数列表
	statusPostTerminating []PodStatusFunc

	// pod 工作协程已观察到终止请求的标志
	startedTerminating bool
	// Pod 是否在 API Server 上被标记为删除或没有配置（之前已被删除）。
	deleted bool
	// pod 是否由于驱逐而被终止
	evicted bool
	// pod 工作协程完成同步操作的标志
	finished bool
	//  pod 工作协程被告知 pod 在被终止后需要重新启动
	restartRequested bool
	// pod 是否在运行时被观察到
	observedRuntime bool
}
```

podWorkers.UpdatePod其实主要做了两件事

1. 处理 Pod 的终止状态

    ```go
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
    ```

2. 处理终止请求

```go
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
		//如果 KillPodOptions 存在 PodStatusFunc，如果通道非空，则将此通道添加到 status.statusPostTerminating 列表中。这个列表保存在 Pod 终止过程中需要执行的所有状态更新函数。
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
```

podWorkers.UpdatePod除了被标记status.IsFinished不会发送到podUpdates通道，其他终止情况都会发送到podWorkerLoop

`podWorkerLoop`如果更新类型是`TerminatedPod`调用**SyncTerminatedPod**清理，如果是`TerminatingPod`调用**acknowledgeTerminating**优雅停机，如果是`TerminatingPod且Pod运行时`调用**SyncTerminatingRuntimePod**终止，如果是`TerminatingPod且是常规Pod`调用**SyncTerminatingPod**终止

 `SyncTerminatedPod`

- **用途**：当 Pod 已经进入终止状态时调用此方法。
- **功能**：完成所有清理工作，包括释放资源和最终状态更新。这是在 Pod 的所有容器都已停止后进行的，目的是确保所有资源都被正确清理。

 `acknowledgeTerminating`

- **用途**：此方法不直接调用同步函数，而是用于处理正在终止但尚未完全停止的 Pod。
- **功能**：主要用于管理和确认 Pod 正在终止的状态。这通常涉及设置一些内部标志以指示 Pod 正在终止过程中，有助于控制优雅停机的流程。

`SyncTerminatingRuntimePod`

- **用途**：用于处理只有运行时信息的 Pod 的终止，这种情况通常发生在 Pod 的配置已从 API 服务器删除，但容器仍在运行的情况下。
- **功能**：直接与容器运行时交互，终止所有仍在运行的容器，不依赖于 Kubernetes 的完整 Pod 对象。这是一种比较快速直接的终止方式。

`SyncTerminatingPod`

- **用途**：用于终止那些仍有完整 Pod 配置信息的 Pod。
- **功能**：优雅地停止 Pod 中的容器，尊重设置的宽限期，并进行必要的状态更新。与 `SyncTerminatingRuntimePod` 不同，这种方法依赖于完整的 Pod 对象来执行更细粒度的控制，如宽限期的处理。

---



* `理解删除流程`

- **设置 `deletionTimestamp`**: 当通过 `kubectl delete pod` 或其他 API 调用请求删除 Pod 时，API 服务器会为该 Pod 设置一个 `deletionTimestamp`，这是 Pod 开始被删除的标志。

    **Kubelet 的响应**: Kubelet 定期同步并检查其管理的每个 Pod。当它发现某个 Pod 有 `deletionTimestamp` 时，将根据 Pod 的当前状态启动适当的终止过程：

    - **如果 Pod 容器仍在运行**：Kubelet 会首先调用 `acknowledgeTerminating` 函数来标记 Pod 正在终止，这个函数还会返回一个可能用于更新 Pod 状态的 `PodStatusFunc`。接着，Kubelet 将继续调用 `SyncTerminatingPod`（针对完整管理的 Pod）或 `SyncTerminatingRuntimePod`（针对仅由容器运行时管理的 Pod，比如裸容器）来执行容器的停止操作。
    - **如果 Pod 已停止所有容器运行**：则直接调用 `SyncTerminatedPod` 来处理最后的清理工作，如卸载卷和更新最终状态。

    **`SyncPod` 的角色**:

    - **默认情况下**，当 Pod 需要更新或响应健康检查失败时，`SyncPod` 会被调用。这包括启动、停止或重启容器以匹配 Pod 的定义状态。如果 Pod 被标记为删除且仍有运行中的容器，`SyncPod` 还会根据需要停止这些容器。
    - 如果 Pod 不能在节点上运行（例如由于资源限制或策略限制），`SyncPod` 也会尝试停止所有相关容器，并通过事件记录和状态更新向用户报告原因。

    **处理终止过程**： `podWorkerLoop` 在这个过程中扮演着关键角色，它通过监听 `podUpdates` 通道来调度和执行上述终止操作。这个循环确保所有终止相关的任务按计划执行，直到 Pod 的资源被完全清理和回收。

    **清理和状态更新**： 在 Pod 的终止过程中，Kubelet 会确保所有相关资源如网络和存储被适当地释放。此外，它会通过状态管理器更新 Pod 的最终状态，标记为 `Terminated`，并确保所有终止信息准确反映在 Kubernetes 的 API 中。

    * ==这里面有个很关键的地方就是他判断是switch case，所以update通道可能先传过来TerminatingPod调用SyncTerminatingPod处理完毕在传来TerminatedPod事件调用SyncTerminatedPod==

---

podWorkerLoop里面的4个清理方法更适合于深入理解 Pod 如何被优雅地停止，以及在停止过程中如何处理资源清理和状态更新

SyncPod里面的涵盖Pod各个生命周期事件，包括Kill等

## 1.1 SyncTerminatedPod

* 主要就是卸载卷，清除secret或configmap，清除cgroup，释放命名空间（如果启用的话）

```go
func (kl *Kubelet) SyncTerminatedPod(ctx context.Context, pod *v1.Pod, podStatus *kubecontainer.PodStatus) error {
	ctx, otelSpan := kl.tracer.Start(ctx, "syncTerminatedPod", trace.WithAttributes(
		semconv.K8SPodUIDKey.String(string(pod.UID)),
		attribute.String("k8s.pod", klog.KObj(pod).String()),
		semconv.K8SPodNameKey.String(pod.Name),
		semconv.K8SNamespaceNameKey.String(pod.Namespace),
	))
	defer otelSpan.End()
	klog.V(4).InfoS("SyncTerminatedPod enter", "pod", klog.KObj(pod), "podUID", pod.UID)
	defer klog.V(4).InfoS("SyncTerminatedPod exit", "pod", klog.KObj(pod), "podUID", pod.UID)

	// generate the final status of the pod
	// TODO: should we simply fold this into TerminatePod? that would give a single pod update
	//生成 Pod 的最终 API 状态。这一步是整合当前 Pod 的运行状态（如容器的健康、运行状态等）到 Kubernetes API 所用的格式。
	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus, true)
	//在内部状态管理器中更新 Pod 状态，这样 Kubernetes API 便可获取到最新的状态信息
	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	// volumes are unmounted after the pod worker reports ShouldPodRuntimeBeRemoved (which is satisfied
	// before syncTerminatedPod is invoked)
	//等待直到所有挂载的卷被卸载
	if err := kl.volumeManager.WaitForUnmount(ctx, pod); err != nil {
		return err
	}
	klog.V(4).InfoS("Pod termination unmounted volumes", "pod", klog.KObj(pod), "podUID", pod.UID)
	//keepTerminatedPodVolumes为true就清理pod之前挂载所有的卷，
	if !kl.keepTerminatedPodVolumes {
		// This waiting loop relies on the background cleanup which starts after pod workers respond
		// true for ShouldPodRuntimeBeRemoved, which happens after `SyncTerminatingPod` is completed.
		if err := wait.PollUntilContextCancel(ctx, 100*time.Millisecond, true, func(ctx context.Context) (bool, error) {
			volumesExist := kl.podVolumesExist(pod.UID)
			if volumesExist {
				klog.V(3).InfoS("Pod is terminated, but some volumes have not been cleaned up", "pod", klog.KObj(pod), "podUID", pod.UID)
			}
			return !volumesExist, nil
		}); err != nil {
			return err
		}
		klog.V(3).InfoS("Pod termination cleaned up volume paths", "pod", klog.KObj(pod), "podUID", pod.UID)
	}

	// After volume unmount is complete, let the secret and configmap managers know we're done with this pod
	//对于使用了secret或configmap的 Pod，从相应的管理器中注销，以释放相关资源。
	if kl.secretManager != nil {
		kl.secretManager.UnregisterPod(pod)
	}
	if kl.configMapManager != nil {
		kl.configMapManager.UnregisterPod(pod)
	}

	// Note: we leave pod containers to be reclaimed in the background since dockershim requires the
	// container for retrieving logs and we want to make sure logs are available until the pod is
	// physically deleted.

	// remove any cgroups in the hierarchy for pods that are no longer running.
	//如果启用了按 QoS 分类的 cgroups，移除与 Pod 关联的 cgroup。这有助于确保资源限制被正确清除。
	if kl.cgroupsPerQOS {
		pcm := kl.containerManager.NewPodContainerManager()
		name, _ := pcm.GetPodContainerName(pod)
		if err := pcm.Destroy(name); err != nil {
			return err
		}
		klog.V(4).InfoS("Pod termination removed cgroups", "pod", klog.KObj(pod), "podUID", pod.UID)
	}
	//释放与 Pod 关联的用户命名空间（如果使用了用户命名空间的功能）
	kl.usernsManager.Release(pod.UID)
	// 在状态管理器中标记 Pod 为已终止，这意味着 Pod 不再需要状态更新
	// mark the final pod status
	kl.statusManager.TerminatePod(pod)
	klog.V(4).InfoS("Pod is terminated and will need no more status updates", "pod", klog.KObj(pod), "podUID", pod.UID)

	return nil
}
```

## 1.2 acknowledgeTerminating

* 主要是设置startedTerminating = true代表终止流程已经启动，返回statusPostTerminating清理函数

```
func (p *podWorkers) acknowledgeTerminating(podUID types.UID) PodStatusFunc {
   p.podLock.Lock()
   defer p.podLock.Unlock()
   //获取指定 UID 的 Pod 的状态
   status, ok := p.podSyncStatuses[podUID]
   if !ok {
      return nil
   }
   //如果terminatingAt非0且startedTerminating未被设置，那么status.startedTerminating = true，代表终止流程已经启动
   if !status.terminatingAt.IsZero() && !status.startedTerminating {
      klog.V(4).InfoS("Pod worker has observed request to terminate", "podUID", podUID)
      status.startedTerminating = true
   }
   //statusPostTerminating是清理函数的列表，如果不为0，代表最新的函数已经被添加，直接返回
   if l := len(status.statusPostTerminating); l > 0 {
      return status.statusPostTerminating[l-1]
   }
   return nil
}
```



## 1.3 SyncTerminatingRuntimePod

* 主要就是用killPod强制杀死容器，包括孤儿容器（Api server已经没有信息，但是还在运行）
* `这里初始化gracePeriod=1，代表优雅删除，因为1其实等于=0，其实可以看作强制删除，注释未来可能实现改成0强制删除`
* killPod下面的syncPod的流程里面会讲

```go

// SyncTerminatingRuntimePod is expected to terminate running containers in a pod that we have no
// configuration for. Once this method returns without error, any remaining local state can be safely
// cleaned up by background processes in each subsystem. Unlike syncTerminatingPod, we lack
// knowledge of the full pod spec and so cannot perform lifecycle related operations, only ensure
// that the remnant of the running pod is terminated and allow garbage collection to proceed. We do
// not update the status of the pod because with the source of configuration removed, we have no
// place to send that status.
func (kl *Kubelet) SyncTerminatingRuntimePod(_ context.Context, runningPod *kubecontainer.Pod) error {
	// TODO(#113606): connect this with the incoming context parameter, which comes from the pod worker.
	// Currently, using that context causes test failures.
	ctx := context.Background()
	//将内部容器 Pod 的表示转换为 Kubernetes API 中使用的 Pod 结构
	pod := runningPod.ToAPIPod()
	klog.V(4).InfoS("SyncTerminatingRuntimePod enter", "pod", klog.KObj(pod), "podUID", pod.UID)
	defer klog.V(4).InfoS("SyncTerminatingRuntimePod exit", "pod", klog.KObj(pod), "podUID", pod.UID)

	// we kill the pod directly since we have lost all other information about the pod.
	klog.V(4).InfoS("Orphaned running pod terminating without grace period", "pod", klog.KObj(pod), "podUID", pod.UID)
	// TODO: this should probably be zero, to bypass any waiting (needs fixes in container runtime)
	gracePeriod := int64(1)
	//调用 killPod 方法尝试终止 Pod
	if err := kl.killPod(ctx, pod, *runningPod, &gracePeriod); err != nil {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToKillPod, "error killing pod: %v", err)
		// there was an error killing the pod, so we return that error directly
		utilruntime.HandleError(err)
		return err
	}
	klog.V(4).InfoS("Pod termination stopped all running orphaned containers", "pod", klog.KObj(pod), "podUID", pod.UID)
	return nil
}
```

`gracePeriod` 在 Kubernetes 中是一个重要的概念，用于指定当请求终止（删除）一个 Pod 时，Pod 中的容器可以在完全被强制停止之前运行的最长时间。这个参数是以秒为单位的，并允许容器在被终止前有足够的时间来进行适当的清理操作，如关闭打开的文件、完成网络请求、保存状态等，以安全和优雅地关闭。

`宽限期的使用情况包括`：

1. **优雅终止**：
    - 当一个 Pod 被标记为删除时，Kubernetes API 会立即停止发送新的请求到该 Pod（比如在服务中的 Pod），并通知 Pod 的容器需要终止。如果指定了宽限期，Pod 内的应用可以利用这段时间来处理正在进行的任务或者执行必要的清理工作。
2. **配置删除请求**：
    - 用户或管理员可以在发送删除请求时指定宽限期。如果没有指定，将使用 Pod 定义中指定的默认宽限期。如果在 Pod 定义中也未指定，默认通常是 30 秒。
3. **强制和立即终止**：
    - 如果宽限期被设置为 `0`，Kubernetes 会立即强制终止容器，这种情况通常用于紧急情况，例如节点维护需要快速回收资源时。

`宽限期的实现方式`：

在 Kubernetes 的容器终止过程中，`gracePeriod` 是如何工作的：

- **发送终止信号**：首先，容器的主进程（PID 1）会接收到一个 SIGTERM 信号，这是一个通知进程应当清理并关闭自身的信号。
- **等待过程**：容器接收到 SIGTERM 信号后，它有 `gracePeriod` 指定的时间来正常关闭。如果在这段时间内容器成功退出，Kubernetes 会认为这个终止过程是正常的。
- **发送 SIGKILL**：如果容器在宽限期结束后仍然运行，Kubernetes 将发送 SIGKILL 信号，这会立即终止容器进程。

通过这种方式，`gracePeriod` 提供了一个平衡点，既保证了系统管理的需要（如释放资源和维护），又尽可能地减少对正在运行的应用程序的打扰。

## 1.4 SyncTerminatingPod

**生成并更新 Pod 状态**：

- 调用 `generateAPIPodStatus` 来生成 Pod 的当前状态，并通过状态管理器更新这一状态。

**终止探针检查**：

- 调用 `probeManager.StopLivenessAndStartup` 停止对 Pod 的存活性和启动性探测。

**调用 killPod 方法**：

- 使用追踪上下文、Pod 信息、运行中的 Pod 状态和宽限期调用 `killPod` 来终止所有容器。==killPod在这里可能是优雅删除，也可能是强制删除==，要看 `KillPodOptions` 中的 `PodTerminationGracePeriodSecondsOverride` 是否被设置

**检查容器状态**：

- 获取最新的容器状态以验证所有容器是否已经停止。如果发现有容器仍在运行，返回错误。

**清理资源**：

- 根据需要执行资源清理，如动态资源分配的回收。

**最终状态更新**：

- 在所有容器停止后，再次更新 Pod 的状态，以反映容器的终止状态。

```go
// SyncTerminatingPod is expected to terminate all running containers in a pod. Once this method
// returns without error, the pod is considered to be terminated and it will be safe to clean up any
// pod state that is tied to the lifetime of running containers. The next method invoked will be
// SyncTerminatedPod. This method is expected to return with the grace period provided and the
// provided context may be cancelled if the duration is exceeded. The method may also be interrupted
// with a context cancellation if the grace period is shortened by the user or the kubelet (such as
// during eviction). This method is not guaranteed to be called if a pod is force deleted from the
// configuration and the kubelet is restarted - SyncTerminatingRuntimePod handles those orphaned
// pods.
func (kl *Kubelet) SyncTerminatingPod(_ context.Context, pod *v1.Pod, podStatus *kubecontainer.PodStatus, gracePeriod *int64, podStatusFn func(*v1.PodStatus)) error {
	// TODO(#113606): connect this with the incoming context parameter, which comes from the pod worker.
	// Currently, using that context causes test failures.
	ctx, otelSpan := kl.tracer.Start(context.Background(), "syncTerminatingPod", trace.WithAttributes(
		semconv.K8SPodUIDKey.String(string(pod.UID)),
		attribute.String("k8s.pod", klog.KObj(pod).String()),
		semconv.K8SPodNameKey.String(pod.Name),
		semconv.K8SNamespaceNameKey.String(pod.Namespace),
	))
	defer otelSpan.End()
	klog.V(4).InfoS("SyncTerminatingPod enter", "pod", klog.KObj(pod), "podUID", pod.UID)
	defer klog.V(4).InfoS("SyncTerminatingPod exit", "pod", klog.KObj(pod), "podUID", pod.UID)
	//生成pod api状态，如果podStatusFn不为空，则调用podStatusFn修改Pod状态
	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus, false)
	if podStatusFn != nil {
		podStatusFn(&apiPodStatus)
	}
	//通过statusManager更新状态
	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	if gracePeriod != nil {
		klog.V(4).InfoS("Pod terminating with grace period", "pod", klog.KObj(pod), "podUID", pod.UID, "gracePeriod", *gracePeriod)
	} else {
		klog.V(4).InfoS("Pod terminating with grace period", "pod", klog.KObj(pod), "podUID", pod.UID, "gracePeriod", nil)
	}
	//停止对 Pod 的存活性和启动性探测
	kl.probeManager.StopLivenessAndStartup(pod)
	//如果是运行时pod，调用killPod停止所有容器
	p := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
	if err := kl.killPod(ctx, pod, p, gracePeriod); err != nil {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToKillPod, "error killing pod: %v", err)
		// there was an error killing the pod, so we return that error directly
		utilruntime.HandleError(err)
		return err
	}

	// Once the containers are stopped, we can stop probing for liveness and readiness.
	// TODO: once a pod is terminal, certain probes (liveness exec) could be stopped immediately after
	//   the detection of a container shutdown or (for readiness) after the first failure. Tracked as
	//   https://github.com/kubernetes/kubernetes/issues/107894 although may not be worth optimizing.
	//从探测管理器中移除了指定的 Pod
	kl.probeManager.RemovePod(pod)

	// Guard against consistency issues in KillPod implementations by checking that there are no
	// running containers. This method is invoked infrequently so this is effectively free and can
	// catch race conditions introduced by callers updating pod status out of order.
	// TODO: have KillPod return the terminal status of stopped containers and write that into the
	//  cache immediately
	//请求容器运行时获取最终的 Pod 状态
	podStatus, err := kl.containerRuntime.GetPodStatus(ctx, pod.UID, pod.Name, pod.Namespace)
	if err != nil {
		klog.ErrorS(err, "Unable to read pod status prior to final pod termination", "pod", klog.KObj(pod), "podUID", pod.UID)
		return err
	}
	var runningContainers []string
	type container struct {
		Name       string
		State      string
		ExitCode   int
		FinishedAt string
	}
	var containers []container
	klogV := klog.V(4)
	klogVEnabled := klogV.Enabled()
	//便利podStatus.ContainerStatuses 列表，s是每个容器的状态信息
	for _, s := range podStatus.ContainerStatuses {
		//如果是kubecontainer.ContainerStateRunning 代表容器正在运行 ，添加到runningContainers
		if s.State == kubecontainer.ContainerStateRunning {
			runningContainers = append(runningContainers, s.ID.String())
		}
		if klogVEnabled {
			containers = append(containers, container{Name: s.Name, State: string(s.State), ExitCode: s.ExitCode, FinishedAt: s.FinishedAt.UTC().Format(time.RFC3339Nano)})
		}
	}
	if klogVEnabled {
		sort.Slice(containers, func(i, j int) bool { return containers[i].Name < containers[j].Name })
		klog.V(4).InfoS("Post-termination container state", "pod", klog.KObj(pod), "podUID", pod.UID, "containers", containers)
	}
	//如果使用killpod之后，runningContainers > 0 代表还有容器在运行，违反CRI规范，返回错误
	if len(runningContainers) > 0 {
		return fmt.Errorf("detected running containers after a successful KillPod, CRI violation: %v", runningContainers)
	}

	// NOTE: resources must be unprepared AFTER all containers have stopped
	// and BEFORE the pod status is changed on the API server
	// to avoid race conditions with the resource deallocation code in kubernetes core.
	//检查是否启用了动态资源分配特性，如果启用，则执行相关资源的清理工作
	if utilfeature.DefaultFeatureGate.Enabled(features.DynamicResourceAllocation) {
		if err := kl.UnprepareDynamicResources(pod); err != nil {
			return err
		}
	}

	// Compute and update the status in cache once the pods are no longer running.
	// The computation is done here to ensure the pod status used for it contains
	// information about the container end states (including exit codes) - when
	// SyncTerminatedPod is called the containers may already be removed.
	//生成pod最新状态，并且在statusManager更新
	apiPodStatus = kl.generateAPIPodStatus(pod, podStatus, true)
	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	// we have successfully stopped all containers, the pod is terminating, our status is "done"
	klog.V(4).InfoS("Pod termination stopped all running containers", "pod", klog.KObj(pod), "podUID", pod.UID)

	return nil
}
```

---

# 2.SyncPod

SyncPod里面关于POd删除的生命周期函数调用  kl.killPod(ctx, pod, p, nil);

kl.killPod-----> kl.containerRuntime.KillPod------->killPodWithSyncResult------->killContainer

`停止正常容器`killPodWithSyncResult-------->killContainer----->remoteRuntimeService StopContainer------>*runtimeServiceClient StopContainer----------->*criService StopContainer(GRPC) ------> criService stopContainer

`停止sandbox`killPodWithSyncResult-------->------->runtimeService.StopPodSandbox------->runtimeClient.StopPodSandbox------->criService) StopPodSandbox(GRPC)----->riService stopPodSandbox

## 2.1 kl.killPod

* ==注意SyncPod里面的Pod删除是优雅删除==，不是强制删除，因为SyncPod  kl.killPod(ctx, pod, p, gracePeriodOverride=nil)，==gracePeriodOverride= nil==；

```go
// KillPod kills all the containers of a pod. Pod may be nil, running pod must not be.
// gracePeriodOverride if specified allows the caller to override the pod default grace period.
// only hard kill paths are allowed to specify a gracePeriodOverride in the kubelet in order to not corrupt user data.
// it is useful when doing SIGKILL for hard eviction scenarios, or max grace period during soft eviction scenarios.
func (m *kubeGenericRuntimeManager) KillPod(ctx context.Context, pod *v1.Pod, runningPod kubecontainer.Pod, gracePeriodOverride *int64) error {
	err := m.killPodWithSyncResult(ctx, pod, runningPod, gracePeriodOverride)
	return err.Error()
}
```

## 2.2 killPodWithSyncResult 

killPodWithSyncResult核心是：

（1）调用killContainersWithSyncResult 函数删除业务容器

（2）调用StopPodSandbox 函数停止sandbox容器。这里有一点，这里只是停止sandbox，说清理工作会由gc做

```go
// killPodWithSyncResult kills a runningPod and returns SyncResult.
// Note: The pod passed in could be *nil* when kubelet restarted.
func (m *kubeGenericRuntimeManager) killPodWithSyncResult(ctx context.Context, pod *v1.Pod, runningPod kubecontainer.Pod, gracePeriodOverride *int64) (result kubecontainer.PodSyncResult) {
	//调用 killContainersWithSyncResult 方法终止 Pod 中的所有容器，并获取操作结果。这些结果保存在 killContainerResults 中
	killContainerResults := m.killContainersWithSyncResult(ctx, pod, runningPod, gracePeriodOverride)
	for _, containerResult := range killContainerResults {
		//环遍历所有容器的终止结果，并将它们添加到最终的 Pod 同步结果 (result) 中
		result.AddSyncResult(containerResult)
	}

	// stop sandbox, the sandbox will be removed in GarbageCollect
	//创建一个新的同步结果实例 killSandboxResult，用于记录沙盒（Pod 级的资源和环境）的终止操作，并将其添加到综合结果中
	killSandboxResult := kubecontainer.NewSyncResult(kubecontainer.KillPodSandbox, runningPod.ID)
	result.AddSyncResult(killSandboxResult)
	// Stop all sandboxes belongs to same pod
	//遍历属于该 Pod 的所有Sandbox，并尝试停止它们。如果停止沙盒失败且错误不是因为沙盒未找到，那么会在 killSandboxResult 中记录失败原因
	for _, podSandbox := range runningPod.Sandboxes {
		if err := m.runtimeService.StopPodSandbox(ctx, podSandbox.ID.ID); err != nil && !crierror.IsNotFound(err) {
			killSandboxResult.Fail(kubecontainer.ErrKillPodSandbox, err.Error())
			klog.ErrorS(nil, "Failed to stop sandbox", "podSandboxID", podSandbox.ID)
		}
	}

	return
}

```

* killContainersWithSyncResult

这里就是异步调用m.killContainer，kill掉所有容器

`并发执行`

- 函数为每个容器启动一个独立的协程，允许并行终止容器。这种并发执行模式提高了处理速度，特别是在处理包含多个容器的大型 Pods 时。

`结果同步`

- 使用 Go 语言的 `sync.WaitGroup` 来同步所有协程，确保主函数只在所有容器都尝试终止之后继续执行。
- 使用通道 (`channel`) 收集每个容器终止操作的结果，允许主线程在所有协程完成后统一处理这些结果。

`容器终止逻辑`

- 对每个容器调用 `killContainer` 方法进行终止，该方法支持传入宽限期覆盖（`gracePeriodOverride`），允许根据具体情况调整容器的宽限期。
- 如果启用了 `SidecarContainers` 特性，还会根据终止顺序（`terminationOrdering`）决定各容器的终止顺序，优先终止非边车容器。

```go

// killContainersWithSyncResult kills all pod's containers with sync results.
func (m *kubeGenericRuntimeManager) killContainersWithSyncResult(ctx context.Context, pod *v1.Pod, runningPod kubecontainer.Pod, gracePeriodOverride *int64) (syncResults []*kubecontainer.SyncResult) {
	//containerResults用于存放每个容器终止操作的结果。通道的容量设置为 runningPod.Containers 的长度
	containerResults := make(chan *kubecontainer.SyncResult, len(runningPod.Containers))
	//WaitGroup，用于同步所有并发执行的容器终止操作
	wg := sync.WaitGroup{}
	//将WaitGroup的计数器设置为容器数量
	wg.Add(len(runningPod.Containers))
	var termOrdering *terminationOrdering
	// we only care about container termination ordering if the sidecars feature is enabled
	//检查是否启用SidecarContainers，如果启用了把容器顺序添加到termOrdering中，用于控制容器终止顺序，一般让Sidecar在正常容器之后终止
	if utilfeature.DefaultFeatureGate.Enabled(features.SidecarContainers) {
		var runningContainerNames []string
		for _, container := range runningPod.Containers {
			runningContainerNames = append(runningContainerNames, container.Name)
		}
		termOrdering = newTerminationOrdering(pod, runningContainerNames)
	}
	//对每个容器启动一个协程进行终止操作
	for _, container := range runningPod.Containers {
		go func(container *kubecontainer.Container) {
			// utilruntime.HandleCrash用于恢复可能的 panic，避免协程崩溃
			defer utilruntime.HandleCrash()
			defer wg.Done()
			//将容器终止操作结果写入containerResults
			killContainerResult := kubecontainer.NewSyncResult(kubecontainer.KillContainer, container.Name)
			//使用killContainer终止容器
			if err := m.killContainer(ctx, pod, container.ID, container.Name, "", reasonUnknown, gracePeriodOverride, termOrdering); err != nil {
				killContainerResult.Fail(kubecontainer.ErrKillContainer, err.Error())
				// Use runningPod for logging as the pod passed in could be *nil*.
				klog.ErrorS(err, "Kill container failed", "pod", klog.KRef(runningPod.Namespace, runningPod.Name), "podUID", runningPod.ID,
					"containerName", container.Name, "containerID", container.ID)
			}
			//将容器终止操作结果写入containerResults
			containerResults <- killContainerResult
		}(container)
	}
	//等待所有容器终止操作完成后，关闭结果通道
	wg.Wait()
	close(containerResults)
	//遍历通道中的所有结果，并将它们收集到 syncResults 列表中，随后返回这个列表。这样，调用者可以了解每个容器终止操作的详细结果
	for containerResult := range containerResults {
		syncResults = append(syncResults, containerResult)
	}
	return
}
```

## 2.3 killContainer

* killContainersWithSyncResult为每个容器启动一个携程killContainer删除容器，并发异步执行，注意如果ordering != nil && gracePeriod > 0代表启用顺序干掉容器，`ordering.waitForTurn(containerName, gracePeriod)` 是一个可能会阻塞的调用，大概意思就是killContainer每个都是并发执行的，但是会根据容器Kill顺序，就是比如第2的先完成了，但是他也会等第1的完成了在完成。特别是Sidecar需要在某些之后完成清理

**容器规格获取**：

- 如果 `pod` 对象非空，函数尝试从 `pod` 中获取指定容器的规格 (`containerSpec`)。如果无法获取，将返回错误。
- 如果 `pod` 对象为空，函数尝试从容器的标签恢复 `pod` 和容器的规格。

**记录容器事件**：

- 无论容器是否有特定的消息要记录，都会在 Kubernetes 事件系统中记录一个 "Killing" 事件。

**宽限期处理**：

- 根据容器的规格和可能的覆盖值设置宽限期 (`gracePeriod`)。
- 如果提供了宽限期覆盖，将使用该值覆盖默认宽限期。

**执行预停止钩子**：

- 如果容器规格中定义了预停止钩子 (`PreStop`)，并且还有足够的宽限期，会先执行这个钩子。执行完毕后，可能会调整剩余的宽限期。

**容器终止顺序**：

- 如果定义了容器的终止顺序（对边车容器特性支持时尤其重要），则会按此顺序等待容器的轮到终止。

**保证最小宽限期**：

- 确保给予容器至少最小宽限期（例如 2 秒），即使在调整后的宽限期小于这个值时也是如此。

**停止容器**：

- 调用运行时服务的 `StopContainer` 方法来停止容器，传入容器 ID 和计算后的宽限期。如果操作失败，会记录错误并返回。

**处理终止顺序状态**：

- 如果有终止顺序管理器，更新容器的终止状态。

```go
// killContainer kills a container through the following steps:
// * Run the pre-stop lifecycle hooks (if applicable).
// * Stop the container.
func (m *kubeGenericRuntimeManager) killContainer(ctx context.Context, pod *v1.Pod, containerID kubecontainer.ContainerID, containerName string, message string, reason containerKillReason, gracePeriodOverride *int64, ordering *terminationOrdering) error {
	var containerSpec *v1.Container
	//先尝试获取将要终止的容器的规格 (containerSpec)。如果 pod 非空，它会从 pod 中查找匹配的容器规格；如果找不到，将返回错误
	if pod != nil {
		if containerSpec = kubecontainer.GetContainerSpec(pod, containerName); containerSpec == nil {
			return fmt.Errorf("failed to get containerSpec %q (id=%q) in pod %q when killing container for reason %q",
				containerName, containerID.String(), format.Pod(pod), message)
		}
		//如果 pod 为空，表示可能是在 kubelet 重启后丢失了上下文，此时它会尝试通过容器的标签恢复 pod 和容器的规格。
	} else {
		// Restore necessary information if one of the specs is nil.
		restoredPod, restoredContainer, err := m.restoreSpecsFromContainerLabels(ctx, containerID)
		if err != nil {
			return err
		}
		pod, containerSpec = restoredPod, restoredContainer
	}

	// From this point, pod and container must be non-nil.
	//根据 pod 和容器的规格设置宽限期 (gracePeriod)
	gracePeriod := setTerminationGracePeriod(pod, containerSpec, containerName, containerID, reason)
	//如果没有特定的消息提供，将生成一个默认消息并记录一个事件，表明正在终止容器
	if len(message) == 0 {
		message = fmt.Sprintf("Stopping container %s", containerSpec.Name)
	}
	m.recordContainerEvent(pod, containerSpec, containerID.ID, v1.EventTypeNormal, events.KillingContainer, message)
	//如果提供了宽限期覆盖 (gracePeriodOverride)，将使用这个覆盖值
	if gracePeriodOverride != nil {
		gracePeriod = *gracePeriodOverride
		klog.V(3).InfoS("Killing container with a grace period override", "pod", klog.KObj(pod), "podUID", pod.UID,
			"containerName", containerName, "containerID", containerID.String(), "gracePeriod", gracePeriod)
	}

	// Run the pre-stop lifecycle hooks if applicable and if there is enough time to run it
	//PreStop 钩子是一个容器生命周期事件，用于在容器终止之前执行特定的任务或清理工作
	//PreStop 钩子的执行时间从容器的宽限期中扣除。这意味着，如果钩子执行花费了一定的时间，那么容器实际上在接收 SIGTERM 信号后剩余的时间会减少
	//如果containerSpec定义了生命周期 Lifecycle和PreStop，并且gracePeriod > 0，那么运行PreStop钩子
	if containerSpec.Lifecycle != nil && containerSpec.Lifecycle.PreStop != nil && gracePeriod > 0 {
		gracePeriod = gracePeriod - m.executePreStopHook(ctx, pod, containerID, containerSpec, gracePeriod)
	}

	// if we care about termination ordering, then wait for this container's turn to exit if there is
	// time remaining
	//如果存在容器终止顺序管理 (ordering)，并且还有剩余的宽限期，会根据终止顺序等待其它容器先行终止
	if ordering != nil && gracePeriod > 0 {
		// grace period is only in seconds, so the time we've waited gets truncated downward
		gracePeriod -= int64(ordering.waitForTurn(containerName, gracePeriod))
	}
	// always give containers a minimal shutdown window to avoid unnecessary SIGKILLs
	//确保最终的宽限期不低于最小值，以避免立即发送 SIGKILL
	if gracePeriod < minimumGracePeriodInSeconds {
		gracePeriod = minimumGracePeriodInSeconds
	}

	klog.V(2).InfoS("Killing container with a grace period", "pod", klog.KObj(pod), "podUID", pod.UID,
		"containerName", containerName, "containerID", containerID.String(), "gracePeriod", gracePeriod)
	//调用StopContainer容器运行时服务终止容器
	err := m.runtimeService.StopContainer(ctx, containerID.ID, gracePeriod)
	if err != nil && !crierror.IsNotFound(err) {
		klog.ErrorS(err, "Container termination failed with gracePeriod", "pod", klog.KObj(pod), "podUID", pod.UID,
			"containerName", containerName, "containerID", containerID.String(), "gracePeriod", gracePeriod)
		return err
	}
	klog.V(3).InfoS("Container exited normally", "pod", klog.KObj(pod), "podUID", pod.UID,
		"containerName", containerName, "containerID", containerID.String())
	//如果使用了终止顺序管理，更新终止状态
	if ordering != nil {
		ordering.containerTerminated(containerName)
	}

	return nil
}
```

## 2.4 r *remoteRuntimeService StopContainer

```go
// StopContainer stops a running container with a grace period (i.e., timeout).
func (r *remoteRuntimeService) StopContainer(ctx context.Context, containerID string, timeout int64) (err error) {
	klog.V(10).InfoS("[RemoteRuntimeService] StopContainer", "containerID", containerID, "timeout", timeout)
	// Use timeout + default timeout (2 minutes) as timeout to leave extra time
	// for SIGKILL container and request latency.
	t := r.timeout + time.Duration(timeout)*time.Second
	ctx, cancel := context.WithTimeout(ctx, t)
	defer cancel()

	r.logReduction.ClearID(containerID)

	if _, err := r.runtimeClient.StopContainer(ctx, &runtimeapi.StopContainerRequest{
		ContainerId: containerID,
		Timeout:     timeout,
	}); err != nil {
		klog.ErrorS(err, "StopContainer from runtime service failed", "containerID", containerID)
		return err
	}
	klog.V(10).InfoS("[RemoteRuntimeService] StopContainer Response", "containerID", containerID)

	return nil
}


```

## 2.5 c *runtimeServiceClient StopContainer

```go

func (c *runtimeServiceClient) StopContainer(ctx context.Context, in *StopContainerRequest, opts ...grpc.CallOption) (*StopContainerResponse, error) {
	out := new(StopContainerResponse)
	err := c.cc.Invoke(ctx, "/runtime.v1.RuntimeService/StopContainer", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```

* `这里分析的是containred源码，还有cri-o这里就不分析了，都是通过初始化GRPC实列化controller然后获得方法调用`

## 2.6 c *criService) StopContainer

* 这是GRPC调用

```go
// StopContainer stops a running container with a grace period (i.e., timeout).
func (c *criService) StopContainer(ctx context.Context, r *runtime.StopContainerRequest) (*runtime.StopContainerResponse, error) {
	start := time.Now()
	// Get container config from container store.
	//从容器存储中获取与请求中的容器ID对应的容器对象
	container, err := c.containerStore.Get(r.GetContainerId())
	if err != nil {
		return nil, fmt.Errorf("an error occurred when try to find container %q: %w", r.GetContainerId(), err)
	}
	//调用stopContainer方法停止容器
	if err := c.stopContainer(ctx, container, time.Duration(r.GetTimeout())*time.Second); err != nil {
		return nil, err
	}
	//尝试获取与容器相关联的沙盒
	sandbox, err := c.sandboxStore.Get(container.SandboxID)
	//获取成功，传递整个沙盒对象，失败传递容器对象
	//当StopContainer方法在Containerd的CRI服务中调用时，将沙盒对象和容器对象传递给NRI的目的是使NRI能够根据这些对象执行一系列的操作或勾子（hooks）
	//资源清理,状态更新,安全审计,通知其他系统组件,执行自定义逻辑
	if err != nil {
		err = c.nri.StopContainer(ctx, nil, &container)
	} else {
		err = c.nri.StopContainer(ctx, &sandbox, &container)
	}
	if err != nil {
		log.G(ctx).WithError(err).Error("NRI failed to stop container")
	}

	i, err := container.Container.Info(ctx)
	if err != nil {
		return nil, fmt.Errorf("get container info: %w", err)
	}

	containerStopTimer.WithValues(i.Runtime.Name).UpdateSince(start)
	//传递空对象代表操作成功
	return &runtime.StopContainerResponse{}, nil
}

```

## 2.7c *criService) stopContainer

> 容器状态检查

- **判断状态**：首先检查容器是否处于"运行中"（`CONTAINER_RUNNING`）或"未知"（`CONTAINER_UNKNOWN`）状态。如果不是这两种状态，则不进行任何操作并返回，因为没有必要停止一个已经停止的容器。

> 获取容器任务

- **任务获取**：尝试获取容器关联的任务（代表容器主进程的执行实体）。如果找不到任务且容器状态不是“未知”，则返回错误。对于未知状态的容器，执行特定的清理操作。

> 处理未知状态

- **未知状态处理**：为处于未知状态的容器设置退出处理程序。如果在等待任务时发生错误，进行清理操作。

> 停止信号的确定和发送

- **停止信号的获取**：如果容器未指定停止信号，则尝试从关联的镜像配置中获取。如果镜像不存在或获取失败，使用默认的SIGTERM信号。
- **信号发送**：使用`task.Kill`发送解析后的停止信号。如果容器已有一个超时大于0的停止请求在处理，将不会重复发送信号。

> 等待和强制停止

- **超时监控**：设置一个超时上下文来监控容器的停止过程。如果在指定的超时期限内容器未能停止，将发送SIGKILL信号强制停止容器。
- **事件监控**：等待容器停止的事件被系统监控捕获。如果监控过程中发生错误，将返回错误。

> 容器停止信号的处理细节

- 当容器的停止信号未显式指定时：
    - **从镜像获取信号**：首先尝试从容器使用的镜像的配置中获取停止信号。
    - **处理镜像不可用的情况**：如果因为镜像被删除或其他原因无法获取停止信号，将使用默认的`SIGTERM`。
    - **信号解析**：将字符串形式的信号名解析为系统可以识别的信号值。
    - **防重复发送机制**：使用原子操作确保在存在多个停止请求时，停止信号只被发送一次。

```go

// stopContainer stops a container based on the container metadata.
func (c *criService) stopContainer(ctx context.Context, container containerstore.Container, timeout time.Duration) error {
	id := container.ID
	sandboxID := container.SandboxID

	// Return without error if container is not running. This makes sure that
	// stop only takes real action after the container is started.
	// 判断容器是否在“运行”或“未知”状态。如果不是这两种状态，则记录信息并返回，不执行停止操作
	state := container.Status.Get().State()
	if state != runtime.ContainerState_CONTAINER_RUNNING &&
		state != runtime.ContainerState_CONTAINER_UNKNOWN {
		log.G(ctx).Infof("Container to stop %q must be in running or unknown state, current state %q",
			id, criContainerStateToString(state))
		return nil
	}

	//尝试获取容器的任务（执行实体）。如果任务不存在且状态不是“未知”，则返回错误。对于未知状态，执行清理操作。
	task, err := container.Container.Task(ctx, nil)
	if err != nil {
		if !errdefs.IsNotFound(err) {
			return fmt.Errorf("failed to get task for container %q: %w", id, err)
		}
		// Don't return for unknown state, some cleanup needs to be done.
		if state == runtime.ContainerState_CONTAINER_UNKNOWN {
			return c.cleanupUnknownContainer(ctx, id, container, sandboxID)
		}
		return nil
	}

	// Handle unknown state.
	//对于处于未知状态的容器，设置一个退出处理程序。如果等待任务失败，执行清理操作。
	if state == runtime.ContainerState_CONTAINER_UNKNOWN {
		// Start an exit handler for containers in unknown state.
		waitCtx, waitCancel := context.WithCancel(ctrdutil.NamespacedContext())
		defer waitCancel()
		exitCh, err := task.Wait(waitCtx)
		if err != nil {
			if !errdefs.IsNotFound(err) {
				return fmt.Errorf("failed to wait for task for %q: %w", id, err)
			}
			return c.cleanupUnknownContainer(ctx, id, container, sandboxID)
		}

		exitCtx, exitCancel := context.WithCancel(context.Background())
		stopCh := c.startContainerExitMonitor(exitCtx, id, task.Pid(), exitCh)
		defer func() {
			exitCancel()
			// This ensures that exit monitor is stopped before
			// `Wait` is cancelled, so no exit event is generated
			// because of the `Wait` cancellation.
			<-stopCh
		}()
	}

	// We only need to kill the task. The event handler will Delete the
	// task from containerd after it handles the Exited event.
	//如果指定了超时，发送SIGTERM或其他配置的停止信号。如果已经发送过信号，则跳过此步骤
	if timeout > 0 {
		stopSignal := "SIGTERM"
		if container.StopSignal != "" {
			stopSignal = container.StopSignal
		} else {
			// The image may have been deleted, and the `StopSignal` field is
			// just introduced to handle that.
			// However, for containers created before the `StopSignal` field is
			// introduced, still try to get the stop signal from the image config.
			// If the image has been deleted, logging an error and using the
			// default SIGTERM is still better than returning error and leaving
			// the container unstoppable. (See issue #990)
			// TODO(random-liu): Remove this logic when containerd 1.2 is deprecated.
			//如果 container.StopSignal为空，从容器的镜像配置中获取停止信号
			image, err := c.GetImage(container.ImageRef)
			//如果出现错误，使用默认的SIGTERM信号
			if err != nil {
				if !errdefs.IsNotFound(err) {
					return fmt.Errorf("failed to get image %q: %w", container.ImageRef, err)
				}
				//如果出现错误，使用默认的SIGTERM信号
				log.G(ctx).Warningf("Image %q not found, stop container with signal %q", container.ImageRef, stopSignal)
			} else {
				//如果没有错误，进入分支，如果image.ImageSpec.Config.StopSignal有值用它更新stopSignal
				if image.ImageSpec.Config.StopSignal != "" {
					stopSignal = image.ImageSpec.Config.StopSignal
				}
			}
		}
		//signal.ParseSignal函数将信号名称转换为相应的信号数值
		sig, err := signal.ParseSignal(stopSignal)
		if err != nil {
			return fmt.Errorf("failed to parse stop signal %q: %w", stopSignal, err)
		}
		//使用原子操作atomic.CompareAndSwapUint32来确保停止信号只发送一次。这是通过检查IsStopSignaledWithTimeout标志来实现的，该标志确保即使有多个停止请求，也只在第一次请求时发送信号
		var sswt bool
		if container.IsStopSignaledWithTimeout == nil {
			log.G(ctx).Infof("unable to ensure stop signal %v was not sent twice to container %v", sig, id)
			sswt = true
		} else {
			sswt = atomic.CompareAndSwapUint32(container.IsStopSignaledWithTimeout, 0, 1)
		}
		//如果sswt为真，即信号未被发送过，使用task.Kill方法向容器发送停止信号。如果发送失败，返回错误。
		if sswt {
			log.G(ctx).Infof("Stop container %q with signal %v", id, sig)
			if err = task.Kill(ctx, sig); err != nil && !errdefs.IsNotFound(err) {
				return fmt.Errorf("failed to stop container %q: %w", id, err)
			}
		} else {
			log.G(ctx).Infof("Skipping the sending of signal %v to container %q because a prior stop with timeout>0 request already sent the signal", sig, id)
		}
		//使用一个超时上下文sigTermCtx来等待容器响应停止信号。如果容器在超时前停止，返回成功；如果超时，尝试使用SIGKILL。
		sigTermCtx, sigTermCtxCancel := context.WithTimeout(ctx, timeout)
		defer sigTermCtxCancel()
		err = c.waitContainerStop(sigTermCtx, container)
		if err == nil {
			// Container stopped on first signal no need for SIGKILL
			return nil
		}
		// If the parent context was cancelled or exceeded return immediately
		if ctx.Err() != nil {
			return ctx.Err()
		}
		// sigTermCtx was exceeded. Send SIGKILL
		log.G(ctx).Debugf("Stop container %q with signal %v timed out", id, sig)
	}

	log.G(ctx).Infof("Kill container %q", id)
	//如果容器没有在规定的超时时间内停止，发送SIGKILL信号强制终止容器
	if err = task.Kill(ctx, syscall.SIGKILL); err != nil && !errdefs.IsNotFound(err) {
		return fmt.Errorf("failed to kill container %q: %w", id, err)
	}

	// Wait for a fixed timeout until container stop is observed by event monitor.
	//待事件监控观察到容器停止。如果在等待过程中发生错误，返回错误。
	err = c.waitContainerStop(ctx, container)
	if err != nil {
		return fmt.Errorf("an error occurs during waiting for container %q to be killed: %w", id, err)
	}
	return nil
}
```

---

## 2.8 stopPodSandbox

1. stopContainer上面解释的方法停止属于sandbox的容器（这里其实是第二次清理，因为killPodWithSyncResult 调用stopContainer已经停止了常规容器）
2. 使用stopSandboxContainer停止pasue轻量级别的容器，stopSandboxContainer是通过初始化controller最后调用stop方法调用的
3. 通知nri Sandbox已清理
4. 清理网络IP，网桥设置等

```go

func (c *criService) stopPodSandbox(ctx context.Context, sandbox sandboxstore.Sandbox) error {
	// Use the full sandbox id.
	//获取沙盒 ID
	id := sandbox.ID

	// Stop all containers inside the sandbox. This terminates the container forcibly,
	// and container may still be created, so production should not rely on this behavior.
	// TODO(random-liu): Introduce a state in sandbox to avoid future container creation.
	//列出所有容器，选出所有属于sandbox的容器，然后调用stopContainer停止容器
	stop := time.Now()
	containers := c.containerStore.List()
	for _, container := range containers {
		if container.SandboxID != id {
			continue
		}
		// Forcibly stop the container. Do not use `StopContainer`, because it introduces a race
		// if a container is removed after list.
		if err := c.stopContainer(ctx, container, 0); err != nil {
			return fmt.Errorf("failed to stop container %q: %w", container.ID, err)
		}
	}

	// Only stop sandbox container when it's running or unknown.
	//只有当沙盒的状态是StateReady或者StateUnknown使用StopSandbox停止沙盒
	state := sandbox.Status.Get().State
	if state == sandboxstore.StateReady || state == sandboxstore.StateUnknown {
		if err := c.sandboxService.StopSandbox(ctx, sandbox.Sandboxer, id); err != nil {
			// Log and ignore the error if controller already removed the sandbox
			if errdefs.IsNotFound(err) {
				log.G(ctx).Warnf("sandbox %q is not found when stopping it", id)
			} else {
				return fmt.Errorf("failed to stop sandbox %q: %w", id, err)
			}
		}
	}

	sandboxRuntimeStopTimer.WithValues(sandbox.RuntimeHandler).UpdateSince(stop)

	err := c.nri.StopPodSandbox(ctx, &sandbox)
	if err != nil {
		log.G(ctx).WithError(err).Errorf("NRI sandbox stop notification failed")
	}

	// Teardown network for sandbox.
	//如果沙盒有关联的网络命名空间(NetNS)，则执行网络清理操作
	//括销毁沙盒的网络配置并尝试移除网络命名空间
	if sandbox.NetNS != nil {
		netStop := time.Now()
		// Use empty netns path if netns is not available. This is defined in:
		// https://github.com/containernetworking/cni/blob/v0.7.0-alpha1/SPEC.md
		if closed, err := sandbox.NetNS.Closed(); err != nil {
			return fmt.Errorf("failed to check network namespace closed: %w", err)
		} else if closed {
			sandbox.NetNSPath = ""
		}
		if err := c.teardownPodNetwork(ctx, sandbox); err != nil {
			return fmt.Errorf("failed to destroy network for sandbox %q: %w", id, err)
		}
		if err := sandbox.NetNS.Remove(); err != nil {
			return fmt.Errorf("failed to remove network namespace for sandbox %q: %w", id, err)
		}
		sandboxDeleteNetwork.UpdateSince(netStop)
	}

	log.G(ctx).Infof("TearDown network for sandbox %q successfully", id)

	return nil
}
```

## 2.9 controller stopSandboxContainer

 **获取容器任务和状态检查**:

- 通过 `container.Task(ctx, nil)` 尝试获取沙盒容器的任务表示。任务代表了容器的执行实体，用于后续的操作如等待退出和发送终止信号。
- 如果任务获取失败且错误不是“未找到”，则返回错误。
- 如果容器状态未知（`StateUnknown`），并且任务未找到，会调用 `cleanupUnknownSandbox` 来进行异常状态下的清理操作。

**处理未知状态的容器**:

- 对于状态未知的容器，设置一个取消控制的上下文 `waitCtx`，并调用 `task.Wait(waitCtx)` 监听容器的退出事件。
- 如果在等待过程中发生错误，并且错误类型不是“未找到”，则执行相应的错误处理。如果是“未找到”，则进行清理。

**并发监控容器退出**:

- 使用 `go` 关键字启动一个新的 goroutine 来异步执行 `c.waitSandboxExit` 函数，该函数负责等待和处理容器的退出过程。
- `exitCh` 通道用于从 `task.Wait` 接收容器任务退出的信号。
- 使用 `exitCtx` 和 `exitCancel` 来管理退出监控的生命周期，确保在适当的时候能取消等待操作。

**强制终止容器**:

- 使用 `syscall.SIGKILL` 强制停止沙盒容器。这是在容器没有响应正常停止请求时的后备措施。
- 确保即使在最坚固的容器也能被终止。

**同步等待和资源释放**:

- `defer` 语句用于确保在函数退出前取消退出监控上下文，并等待异步监控操作完成（通过 `<-stopCh` 确保 `go` 函数已经结束）。
- 最后，调用 `podSandbox.Wait(ctx)` 确保容器彻底停止，并处理任何剩余的同步问题。

```go

// stopSandboxContainer kills the sandbox container.
// `task.Delete` is not called here because it will be called when
// the event monitor handles the `TaskExit` event.
func (c *Controller) stopSandboxContainer(ctx context.Context, podSandbox *types.PodSandbox) error {
	id := podSandbox.ID
	container := podSandbox.Container
	state := podSandbox.Status.Get().State
	//首先尝试获取沙盒容器的任务表示（container.Task）
	task, err := container.Task(ctx, nil)
	//任务未找到，则返回错误
	if err != nil {
		if !errdefs.IsNotFound(err) {
			return fmt.Errorf("failed to get pod sandbox container: %w", err)
		}
		// Don't return for unknown state, some cleanup needs to be done.
		//如果任务未找到且容器状态未知清理他
		if state == sandboxstore.StateUnknown {
			return cleanupUnknownSandbox(ctx, id, podSandbox)
		}
		return nil
	}

	// Handle unknown state.
	// The cleanup logic is the same with container unknown state.
	//如果沙盒容器状态未知
	if state == sandboxstore.StateUnknown {
		// Start an exit handler for sandbox container in unknown state.
		waitCtx, waitCancel := context.WithCancel(ctrdutil.NamespacedContext())
		defer waitCancel()
		//task.Wait(waitCtx)用来等待容器主进程结束
		//task 是沙盒容器的代表，Wait 方法将返回一个通道（exitCh），该通道将在任务结束时接收到信号。
		exitCh, err := task.Wait(waitCtx)
		//如果不是因为未找到报错，返回报错，如果是未找到清理沙盒
		if err != nil {
			if !errdefs.IsNotFound(err) {
				return fmt.Errorf("failed to wait for task: %w", err)
			}
			return cleanupUnknownSandbox(ctx, id, podSandbox)
		}

		exitCtx, exitCancel := context.WithCancel(context.Background())
		stopCh := make(chan struct{})
		//启动一个携程来执行waitSandboxExit，waitSandboxExit监控沙盒的退出过程，exitCh 通道用于从 task.Wait 接收容器退出的通知。
		go func() {
			defer close(stopCh)
			err := c.waitSandboxExit(exitCtx, podSandbox, exitCh)
			if err != nil && err != context.Canceled && err != context.DeadlineExceeded {
				log.G(ctx).WithError(err).Errorf("Failed to wait sandbox exit %+v", err)
			}
		}()
		//等待 goroutine 结束（通过接收 stopCh 的关闭
		defer func() {
			exitCancel()
			// This ensures that exit monitor is stopped before
			// `Wait` is cancelled, so no exit event is generated
			// because of the `Wait` cancellation.
			<-stopCh
		}()
	}

	// Kill the pod sandbox container.
	//使用 SIGKILL 信号强制停止沙盒容器
	if err = task.Kill(ctx, syscall.SIGKILL); err != nil && !errdefs.IsNotFound(err) {
		return fmt.Errorf("failed to kill pod sandbox container: %w", err)
	}
	//等待容器终止
	_, err = podSandbox.Wait(ctx)
	return err
}
```

## 3.0 killPodWithSyncResult 的操作步骤解析

1. **停止常规容器**：

    - 通常开始于停止 Pod 中所有运行的业务容器（包括主容器和 sidecar 容器）。这通过调用 `killContainer` 来实现，其中 `killContainer` 通常会进一步调用 `stopContainer`。
    - `stopContainer` 负责发送停止信号给容器，并确保容器在规定时间内退出。如果容器没有在超时期限内停止，它将强制终止容器。

2. **调用 StopPodSandbox**：

    - 在所有业务容器被停止后，`killPodWithSyncResult` 将调用 `StopPodSandbox`。这是用来停止整个 Pod 的网络和命名空间环境，通常由 pause 容器维持。

    - **StopPodSandbox**

         的执行通常包括两个主要步骤：

        - **先停止所有业务容器**：尽管这些容器应该已经在前面的步骤中被停止，`StopPodSandbox` 再次确保所有容器均已停止是为了防止任何可能的遗漏或异常情况。
        - **停止 pause 容器**：最后停止 Pod 的 pause 容器，这实际上会导致 Pod 的网络和 PID 命名空间被销毁。

3. **资源清理**：

    - 在 pause 容器停止之后，相关的网络配置和可能的存储资源也会被清理，这通常涉及到 CNI 插件或 Kubernetes 的存储接口来执行。

`理解整个流程`

- **分层和顺序性**：整个流程体现了 Kubernetes 对 Pod 生命周期管理的分层和顺序性处理。首先确保所有用户层面的容器已经被适当地停止，然后才清理维持 Pod 基本运行环境的 pause 容器及其相关资源。
- **安全性和完整性**：通过这种方式，Kubernetes 确保了在停止 Pod 的过程中，所有的资源都被正确管理和释放，减少了资源泄露的风险，并保证了集群资源的有效利用和安全。

# 3 Pod是如何被刪除的

HandlePodSyncs------>podWorkers.UpdatePod-------->podWorkerLoop----->SyncPod

* SetPodStatus

```
func (m *manager) SetPodStatus(pod *v1.Pod, status v1.PodStatus) {
   m.podStatusesLock.Lock()
   defer m.podStatusesLock.Unlock()

   // Make sure we're caching a deep copy.
   //使用 status.DeepCopy() 创建 Pod 状态的深拷贝
   status = *status.DeepCopy()

   // Force a status update if deletion timestamp is set. This is necessary
   // because if the pod is in the non-running state, the pod worker still
   // needs to be able to trigger an update and/or deletion.
   //如果设置了删除时间戳，无论 Pod 当前的状态如何，都将强制更新其状态
   m.updateStatusInternal(pod, status, pod.DeletionTimestamp != nil, false)
}
```

* updateStatusInternal

    ==主要更新podStatus的状态，然后往这个podStatusChannel发送状态==

1. `v1.ContainersReady`

- **作用**: 指示 Pod 中所有容器是否已准备就绪（即已经启动并且运行无异常）。
- **更新时机**: 当容器的就绪状态发生变化时更新。

2. `v1.PodReady`

- **作用**: 表示 Pod 作为整体是否处于就绪状态。只有当所有容器都处于就绪状态，且满足 Pod 的其他就绪标准（如服务就绪）时，这一条件才会被标记为 true。
- **更新时机**: 当 Pod 的整体就绪状态发生变化时更新。

3. `v1.PodInitialized`

- **作用**: 指示 Pod 的所有 Init 容器是否已成功完成初始化。
- **更新时机**: 当所有 Init 容器运行完成后，此状态被更新。

4. `v1.PodReadyToStartContainers`

- **作用**: 这是一个较新的条件，用来标示 Pod 是否准备好启动非初始化容器。
- **更新时机**: 当 Pod 中的 Init 容器全部完成且 Pod 准备好启动主容器时更新。

5. `v1.PodScheduled`

- **作用**: 表示 Pod 已经被调度到一个节点上并且由 kubelet 开始进行管理。
- **更新时机**: 当调度决定已经确定，并且 Pod 被绑定到指定节点上时更新。

6. `v1.DisruptionTarget` (如果启用了 PodDisruptionConditions 功能)

- **作用**: 这是一个与 PodDisruptionBudget (PDB) 相关的条件，用于表示 Pod 是否被标记为受 PDB 保护的干扰目标。
- **更新时机**: 当 Pod 的干扰状态发生变化时更新，这通常与集群中的 PDB 设置相关。

```go
// updateStatusInternal updates the internal status cache, and queues an update to the api server if
// necessary.
// This method IS NOT THREAD SAFE and must be called from a locked function.
func (m *manager) updateStatusInternal(pod *v1.Pod, status v1.PodStatus, forceUpdate, podIsFinished bool) {
	var oldStatus v1.PodStatus
	//这段代码首先尝试从内部缓存中获取 Pod 的旧状态。
	cachedStatus, isCached := m.podStatuses[pod.UID]
	//如果 Pod 状态已缓存（isCached），则从缓存中取出。
	if isCached {
		oldStatus = cachedStatus.status
		// TODO(#116484): Also assign terminal phase to static pods.
		if !kubetypes.IsStaticPod(pod) {
			if cachedStatus.podIsFinished && !podIsFinished {
				klog.InfoS("Got unexpected podIsFinished=false, while podIsFinished=true in status cache, programmer error.", "pod", klog.KObj(pod))
				podIsFinished = true
			}
		}
		//如果状态未缓存且 Pod 是镜像 Pod（mirrorPod），则从镜像 Pod 中获取状态。
	} else if mirrorPod, ok := m.podManager.GetMirrorPodByPod(pod); ok {
		oldStatus = mirrorPod.Status
		//如果都不满足，则使用 Pod 的当前状态作为旧状态。
	} else {
		oldStatus = pod.Status
	}

	// Check for illegal state transition in containers
	//checkContainerStateTransition 函数来检查容器状态是否有非法转换
	if err := checkContainerStateTransition(&oldStatus, &status, &pod.Spec); err != nil {
		klog.ErrorS(err, "Status update on pod aborted", "pod", klog.KObj(pod))
		return
	}
	//这一系列 updateLastTransitionTime 调用用于更新不同条件的 LastTransitionTime，这是跟踪状态变化的重要时间戳。
	// Set ContainersReadyCondition.LastTransitionTime.
	updateLastTransitionTime(&status, &oldStatus, v1.ContainersReady)

	// Set ReadyCondition.LastTransitionTime.
	updateLastTransitionTime(&status, &oldStatus, v1.PodReady)

	// Set InitializedCondition.LastTransitionTime.
	updateLastTransitionTime(&status, &oldStatus, v1.PodInitialized)

	// Set PodReadyToStartContainersCondition.LastTransitionTime.
	updateLastTransitionTime(&status, &oldStatus, v1.PodReadyToStartContainers)

	// Set PodScheduledCondition.LastTransitionTime.
	updateLastTransitionTime(&status, &oldStatus, v1.PodScheduled)

	if utilfeature.DefaultFeatureGate.Enabled(features.PodDisruptionConditions) {
		// Set DisruptionTarget.LastTransitionTime.
		updateLastTransitionTime(&status, &oldStatus, v1.DisruptionTarget)
	}
	//确保 StartTime 在更新过程中保持不变。如果已经设置，则使用旧的 StartTime；如果未设置，则初始化为当前时间
	// ensure that the start time does not change across updates.
	if oldStatus.StartTime != nil && !oldStatus.StartTime.IsZero() {
		status.StartTime = oldStatus.StartTime
	} else if status.StartTime.IsZero() {
		// if the status has no start time, we need to set an initial time
		now := metav1.Now()
		status.StartTime = &now
	}

	normalizeStatus(pod, &status)

	// Perform some more extensive logging of container termination state to assist in
	// debugging production races (generally not needed).
	//这部分代码在详细日志级别下，记录容器的当前和上一个终止状态，帮助调试
	if klogV := klog.V(5); klogV.Enabled() {
		var containers []string
		for _, s := range append(append([]v1.ContainerStatus(nil), status.InitContainerStatuses...), status.ContainerStatuses...) {
			var current, previous string
			switch {
			case s.State.Running != nil:
				current = "running"
			case s.State.Waiting != nil:
				current = "waiting"
			case s.State.Terminated != nil:
				current = fmt.Sprintf("terminated=%d", s.State.Terminated.ExitCode)
			default:
				current = "unknown"
			}
			switch {
			case s.LastTerminationState.Running != nil:
				previous = "running"
			case s.LastTerminationState.Waiting != nil:
				previous = "waiting"
			case s.LastTerminationState.Terminated != nil:
				previous = fmt.Sprintf("terminated=%d", s.LastTerminationState.Terminated.ExitCode)
			default:
				previous = "<none>"
			}
			containers = append(containers, fmt.Sprintf("(%s state=%s previous=%s)", s.Name, current, previous))
		}
		sort.Strings(containers)
		klogV.InfoS("updateStatusInternal", "version", cachedStatus.version+1, "podIsFinished", podIsFinished, "pod", klog.KObj(pod), "podUID", pod.UID, "containers", strings.Join(containers, " "))
	}

	// The intent here is to prevent concurrent updates to a pod's status from
	// clobbering each other so the phase of a pod progresses monotonically.
	//检查如果状态没有变化且没有强制更新标志，则忽略本次更新
	if isCached && isPodStatusByKubeletEqual(&cachedStatus.status, &status) && !forceUpdate {
		klog.V(3).InfoS("Ignoring same status for pod", "pod", klog.KObj(pod), "status", status)
		return
	}

	newStatus := versionedPodStatus{
		status:        status,
		version:       cachedStatus.version + 1,
		podName:       pod.Name,
		podNamespace:  pod.Namespace,
		podIsFinished: podIsFinished,
	}

	// Multiple status updates can be generated before we update the API server,
	// so we track the time from the first status update until we retire it to
	// the API.
	//更新内部状态缓存，并尝试通过 m.podStatusChannel 发送状态更新信号。
	//如果通道已满，表示已有状态更新在处理中，本次更新将等待下一轮批处理。
	if cachedStatus.at.IsZero() {
		newStatus.at = time.Now()
	} else {
		newStatus.at = cachedStatus.at
	}

	m.podStatuses[pod.UID] = newStatus

	select {
	case m.podStatusChannel <- struct{}{}:
	default:
		// there's already a status update pending
	}
}
```

* statusManager.start

在Kubelet.Run的时候, statusManager.start了起来

```
// Start component sync loops.
	kl.statusManager.Start()
	kl.probeManager.Start()
```

Start函数的核心就是用一个协程处理podStatusChannel和syncTicker的数据，syncTicker是每十秒更新一次全量更新Pod，podStatusChannel是单次Pod状态

```g0

func (m *manager) Start() {
	// Initialize m.state to no-op state checkpoint manager
	m.state = state.NewNoopStateCheckpoint()

	// Create pod allocation checkpoint manager even if client is nil so as to allow local get/set of AllocatedResources & Resize
	//如果启用了 InPlacePodVerticalScaling 特性，会创建一个新的状态检查点管理器来支持 Pod 的资源分配和调整大小的状态持久化。
	if utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) {
		stateImpl, err := state.NewStateCheckpoint(m.stateFileDirectory, podStatusManagerStateFile)
		if err != nil {
			// This is a crictical, non-recoverable failure.
			klog.ErrorS(err, "Could not initialize pod allocation checkpoint manager, please drain node and remove policy state file")
			panic(err)
		}
		m.state = stateImpl
	}

	// Don't start the status manager if we don't have a client. This will happen
	// on the master, where the kubelet is responsible for bootstrapping the pods
	// of the master components.
	//检查 m.kubeClient 是否为 nil。如果是 nil，则不启动状态管理器。这通常在 kubelet 运行在 master 节点时发生，kubelet 只负责初始化 master 组件的 Pod，而不与 API 服务器同步状态
	if m.kubeClient == nil {
		klog.InfoS("Kubernetes client is nil, not starting status manager")
		return
	}

	klog.InfoS("Starting to sync pod status with apiserver")

	//nolint:staticcheck // SA1015 Ticker can leak since this is only called once and doesn't handle termination.
	//创建一个定时器，每个 syncPeriod 发送一次信号
	syncTicker := time.NewTicker(syncPeriod).C

	// syncPod and syncBatch share the same go routine to avoid sync races.
	// 在一个独立的协程中无限循环监听两个通道
	go wait.Forever(func() {
		for {
			select {
			case <-m.podStatusChannel: // 监听来自 Pod 状态更新的通道
				klog.V(4).InfoS("Syncing updated statuses")
				m.syncBatch(false) // 只同步更新的状态
			case <-syncTicker: // 监听定时器通道
				klog.V(4).InfoS("Syncing all statuses")
				m.syncBatch(true) // 执行全量同步
			}
		}
	}, 0)
}
```

* syncBatch

ALL 代表同步全量还是单次

静态pod由kubelet 进程管理，而不是通过 Kubernetes API 服务器。镜像Pod为了让 API 服务器能够显示和跟踪这些静态 Pod 的状态，kubelet 会在 API 服务器上创建一个对应的镜像 Pod

**清理孤立的版本**： 如果 `all` 为 `true`，则遍历 `m.apiStatusVersions` 并删除那些既不存在于 `m.podStatuses` 也不存在于 `mirrorToPod` 中的 UID。

**状态更新检查**： 遍历 `m.podStatuses`，通过 `podToMirror` 将 Pod UID 转换为状态 UID。对每个 Pod，检查其是否需要更新状态：

- 如果 `all` 为 `false`，仅在有新状态更新时触发更新。
- 如果 `m.apiStatusVersions[uidOfStatus] >= status.version` 表示状态版本已经是最新的，则跳过更新。

**状态更新或协调**： 使用 `needsUpdate` 和 `needsReconcile` 方法判断是否需要更新或协调状态：

- `needsUpdate` 返回 `true` 时，将该 Pod 状态添加到 `updatedStatuses` 列表中。
- `needsReconcile` 返回 `true` 时，删除 `m.apiStatusVersions` 中的对应条目以强制更新 Pod 状态，并将该 Pod 状态添加到 `updatedStatuses` 列表中。

**同步状态和记录日志**： 遍历 `updatedStatuses`，调用 `syncPod` 方法同步每个需要更新的 Pod 状态，并记录日志信息。

**返回值**： 返回尝试同步的 Pod 状态数量

```go
// syncBatch syncs pods statuses with the apiserver. Returns the number of syncs
// attempted for testing.
func (m *manager) syncBatch(all bool) int {
	//定义了一个内部类型 podSync，用来存储 Pod 的 UID、状态 UID 和版本化的 Pod 状态信息。
	type podSync struct {
		podUID    types.UID
		statusUID kubetypes.MirrorPodUID
		status    versionedPodStatus
	}
	//声明了一个切片 updatedStatuses 用于存储需要更新的 Pod 状态。
	//调用 m.podManager.GetUIDTranslations() 获取 Pod 与其镜像 Pod 之间的 UID 转换关系。
	var updatedStatuses []podSync
	podToMirror, mirrorToPod := m.podManager.GetUIDTranslations()
	//使用匿名函数和读锁 RLock 保护临界区，以防止并发访问 m.podStatuses
	func() { // Critical section
		m.podStatusesLock.RLock()
		defer m.podStatusesLock.RUnlock()

		// Clean up orphaned versions.
		//如果 all 为 true，则清理孤立的版本：遍历 m.apiStatusVersions是API服务器中POd的UID映射，删除那些既不存在于 m.podStatuses 也不存在于 mirrorToPod 中的 UID。
		if all {
			for uid := range m.apiStatusVersions {
				_, hasPod := m.podStatuses[types.UID(uid)]
				_, hasMirror := mirrorToPod[uid]
				if !hasPod && !hasMirror {
					delete(m.apiStatusVersions, uid)
				}
			}
		}

		// Decide which pods need status updates.
		//m.podStatuses 是一个映射表，存储了每个 Pod 的 UID 及其对应的状态
		for uid, status := range m.podStatuses {
			// translate the pod UID (source) to the status UID (API pod) -
			// static pods are identified in source by pod UID but tracked in the
			// API via the uid of the mirror pod
			//uidOfStatus 为当前 Pod 的 UID
			uidOfStatus := kubetypes.MirrorPodUID(uid)
			//podToMirror 是一个映射，包含 Pod UID 与镜像 Pod UID 之间的关系
			if mirrorUID, ok := podToMirror[kubetypes.ResolvedPodUID(uid)]; ok {
				//如果 mirrorUID 为空，说明这个静态 Pod 没有对应的镜像 Pod，则跳过这个 Pod 的状态更新。
				if mirrorUID == "" {
					klog.V(5).InfoS("Static pod does not have a corresponding mirror pod; skipping",
						"podUID", uid,
						"pod", klog.KRef(status.podNamespace, status.podName))
					continue
				}
				//否则，将 uidOfStatus 设置为 mirrorUID，即镜像 Pod 的 UID。
				//因为Api服务器静态Pod就是镜像Pod
				uidOfStatus = mirrorUID
			}

			// if a new status update has been delivered, trigger an update, otherwise the
			// pod can wait for the next bulk check (which performs reconciliation as well)
			//如果 all 为 false，则仅在有新的状态更新时触发更新；如果 all 为 true，则跳过这一部分，继续执行下一步检查
			//如果m.apiStatusVersions[uidOfStatus] >= status.version 代表静态pod版本大于APi服务器的版本，不需要更新
			if !all {
				if m.apiStatusVersions[uidOfStatus] >= status.version {
					continue
				}
				//如果有新的状态更新，则将该状态添加到 updatedStatuses 列表中
				updatedStatuses = append(updatedStatuses, podSync{uid, uidOfStatus, status})
				continue
			}

			// Ensure that any new status, or mismatched status, or pod that is ready for
			// deletion gets updated. If a status update fails we retry the next time any
			// other pod is updated.
			//needsUpdate判断当前uidOfStatus是否需要更新
			if m.needsUpdate(types.UID(uidOfStatus), status) {
				//如果 needsUpdate 返回 true，表示该 Pod 需要更新状态，将其添加到 updatedStatuses 列表中
				updatedStatuses = append(updatedStatuses, podSync{uid, uidOfStatus, status})
				//如果 needsUpdate 返回 false，使用needsReconcile进一步检查是否需要协调状态。
				//needsReconcile 方法通常用于检查状态是否不一致，或者是否需要执行协调操作以确保状态同步。
			} else if m.needsReconcile(uid, status.status) {
				// Delete the apiStatusVersions here to force an update on the pod status
				// In most cases the deleted apiStatusVersions here should be filled
				// soon after the following syncPod() [If the syncPod() sync an update
				// successfully].
				//needsReconcile返回true，代表需要，首先删除 m.apiStatusVersions 中的条目，以强制在下一次状态同步时更新 Pod 的状态。
				//然后将该 Pod 添加到 updatedStatuses 列表中。
				delete(m.apiStatusVersions, uidOfStatus)
				updatedStatuses = append(updatedStatuses, podSync{uid, uidOfStatus, status})
			}
		}
	}()
	//遍历 updatedStatuses，调用 syncPod 方法同步每个需要更新的 Pod 状态，并记录日志。
	for _, update := range updatedStatuses {
		klog.V(5).InfoS("Sync pod status", "podUID", update.podUID, "statusUID", update.statusUID, "version", update.status.version)
		m.syncPod(update.podUID, update.status)
	}
	//返回尝试同步的 Pod 状态数量
	return len(updatedStatuses)
}
```

* syncPod

  **获取 Pod**：
  
  - 使用 `kubeClient` 从 API 服务器获取指定命名空间和名称的 Pod。如果找不到 Pod 或遇到其他错误，函数将打印相应的日志并返回。
  
  **检查 Pod 的存在性**：
  
  - 如果查询结果显示 Pod 不存在（`IsNotFound(err)`），则记录日志并忽略状态更新。
  - 如果存在其他类型的错误，也记录日志并返回。
  
  **UID 校验**：
  
  - 通过 `TranslatePodUID` 方法将 Pod 的 UID 转换，并检查是否与传入的 UID 相匹配。如果不匹配（表明 Pod 可能被删除后重新创建），则删除该 Pod 的旧状态并返回。
  
  **状态合并与更新**：
  
  - 将 API 服务器上的 Pod 状态与传入的状态合并。
  - 使用 `PatchPodStatus` 方法尝试更新 Pod 状态。如果状态未更改，则记录状态已是最新的。如果更新成功，记录成功的日志，并更新本地 Pod 对象。
  
  **状态更新延迟测量**：
  
  - 如果提供的状态包含时间戳，测量并记录从状态生成到服务器更新的时间。
  
  **内部状态版本更新**：
  
  - 更新内部跟踪的状态版本号。
  
  **Pod 删除处理**：
  
  - 检查 Pod 是否可以被删除（基于其删除时间戳、是否为镜像 Pod、Pod 状态和是否已完成终止）。如果可以删除，则设置删除选项（无宽限期，并设置删除的 UID 前提条件）并请求 API 服务器删除 Pod。
  - 如果删除请求成功，记录 Pod 已从 etcd 完全终止和删除的日志，并清理 Pod 状态。
  
   

```go

// syncPod syncs the given status with the API server. The caller must not hold the status lock.
func (m *manager) syncPod(uid types.UID, status versionedPodStatus) {
	// TODO: make me easier to express from client code
	//从 Kubernetes API 服务器获取指定命名空间和名称的 Pod
	pod, err := m.kubeClient.CoreV1().Pods(status.podNamespace).Get(context.TODO(), status.podName, metav1.GetOptions{})
	//如果找不到错误，即Pod不存在
	if errors.IsNotFound(err) {
		klog.V(3).InfoS("Pod does not exist on the server",
			"podUID", uid,
			"pod", klog.KRef(status.podNamespace, status.podName))
		// If the Pod is deleted the status will be cleared in
		// RemoveOrphanedStatuses, so we just ignore the update here.
		return
	}
	//检查是否存在其他类型的错误
	if err != nil {
		//说明无法获取 Pod 状态
		klog.InfoS("Failed to get status for pod",
			"podUID", uid,
			"pod", klog.KRef(status.podNamespace, status.podName),
			"err", err)
		return
	}
	//说明无法获取 Pod 状态
	translatedUID := m.podManager.TranslatePodUID(pod.UID)
	// Type convert original uid just for the purpose of comparison.
	//检查转换后的 UID 是否有效，并且是否与请求的 UID 不匹配
	if len(translatedUID) > 0 && translatedUID != kubetypes.ResolvedPodUID(uid) {
		//记录日志，说明 Pod 已被删除并重新创建，跳过状态更新。
		klog.V(2).InfoS("Pod was deleted and then recreated, skipping status update",
			"pod", klog.KObj(pod),
			"oldPodUID", uid,
			"podUID", translatedUID)
		//删除pod的旧状态
		m.deletePodStatus(uid)
		return
	}
	//合并当前 Pod 状态与新的状态
	mergedStatus := mergePodStatus(pod.Status, status.status, m.podDeletionSafety.PodCouldHaveRunningContainers(pod))
	//更新 API 服务器上的 Pod 状态
	newPod, patchBytes, unchanged, err := statusutil.PatchPodStatus(context.TODO(), m.kubeClient, pod.Namespace, pod.Name, pod.UID, pod.Status, mergedStatus)
	klog.V(3).InfoS("Patch status for pod", "pod", klog.KObj(pod), "podUID", uid, "patch", string(patchBytes))
	//检查更新是否有错误，有错误返回
	if err != nil {
		klog.InfoS("Failed to update status for pod", "pod", klog.KObj(pod), "err", err)
		return
	}
	//检查状态是否未改变
	if unchanged {
		//记录状态已经是最新的
		klog.V(3).InfoS("Status for pod is up-to-date", "pod", klog.KObj(pod), "statusVersion", status.version)
		//如果状态有更新，记录最新的状态
	} else {
		klog.V(3).InfoS("Status for pod updated successfully", "pod", klog.KObj(pod), "statusVersion", status.version, "status", mergedStatus)
		//更新本地的 Pod 对象
		pod = newPod
		// We pass a new object (result of API call which contains updated ResourceVersion)
		//记录 Pod 状态更新的延迟
		m.podStartupLatencyHelper.RecordStatusUpdated(pod)
	}

	// measure how long the status update took to propagate from generation to update on the server
	//检查状态的时间戳是否设置
	if status.at.IsZero() {
		klog.V(3).InfoS("Pod had no status time set", "pod", klog.KObj(pod), "podUID", uid, "version", status.version)
	} else {
		//如果时间戳已设置，计算状态更新耗时，记录状态更新耗时的指标
		duration := time.Since(status.at).Truncate(time.Millisecond)
		metrics.PodStatusSyncDuration.Observe(duration.Seconds())
	}
	//更新内部状态版本记录
	m.apiStatusVersions[kubetypes.MirrorPodUID(pod.UID)] = status.version

	// We don't handle graceful deletion of mirror pods.
	//检查 Pod 是否可以被删除
	if m.canBeDeleted(pod, status.status, status.podIsFinished) {
		//设置删除选项，包括宽限期和删除前提条件
		deleteOptions := metav1.DeleteOptions{
			GracePeriodSeconds: new(int64),
			// Use the pod UID as the precondition for deletion to prevent deleting a
			// newly created pod with the same name and namespace.
			Preconditions: metav1.NewUIDPreconditions(string(pod.UID)),
		}
		//发起删除 Pod 的请求
		err = m.kubeClient.CoreV1().Pods(pod.Namespace).Delete(context.TODO(), pod.Name, deleteOptions)
		if err != nil {
			klog.InfoS("Failed to delete status for pod", "pod", klog.KObj(pod), "err", err)
			return
		}
		klog.V(3).InfoS("Pod fully terminated and removed from etcd", "pod", klog.KObj(pod))
		//删除 Pod 状态
		m.deletePodStatus(uid)
	}
}
```

* canBeDeleted前置条件：强制删除GracePeriodSeconds: new(int64)初始化是0，并且Uid不能重复
* 删除条件
    * pod不能是镜像pod，并且有DeletionTimestamp
    * Pod状态必须是Failed 或 Succeeded
    * podIsFinished必须=true

```go
func (m *manager) canBeDeleted(pod *v1.Pod, status v1.PodStatus, podIsFinished bool) bool {
	//pod不能没有删除时间戳或者是镜像pod，不能删除，因为镜像pod由kubelet管理，不是Api服务器
	if pod.DeletionTimestamp == nil || kubetypes.IsMirrorPod(pod) {
		return false
	}
	// Delay deletion of pods until the phase is terminal, based on pod.Status
	// which comes from pod manager.
	//这行代码检查 Pod 的状态是否处于终止阶段（例如 Failed 或 Succeeded）。如果不是，执行内部的代码。
	if !podutil.IsPodPhaseTerminal(pod.Status.Phase) {
		//如果 Pod 状态不是终止阶段，记录一条日志，说明 Pod 删除被延迟，并展示 Pod 当前的阶段和本地记录的阶段
		// For debugging purposes we also log the kubelet's local phase, when the deletion is delayed.
		klog.V(3).InfoS("Delaying pod deletion as the phase is non-terminal", "phase", pod.Status.Phase, "localPhase", status.Phase, "pod", klog.KObj(pod), "podUID", pod.UID)
		return false
	}
	// If this is an update completing pod termination then we know the pod termination is finished.
	//检查 podIsFinished 参数，如果为真，表示 Pod 已经完成了终止过程。
	if podIsFinished {
		klog.V(3).InfoS("The pod termination is finished as SyncTerminatedPod completes its execution", "phase", pod.Status.Phase, "localPhase", status.Phase, "pod", klog.KObj(pod), "podUID", pod.UID)
		return true
	}
	return false
}
```

总结：容器停止清理完毕后，通过pleg的同步，在判断容器停止后，最终会往apiserver发送一个 强制删除pod的请求。这个时候apiserver才会往etcd调用删除pod的操作。

# 4 kubelet监听到删除pod操作后做了什么操作

* HandlePodRemoves

1. 删除本地Pod状态podManager.RemovePod
2. 如果是镜像pod且原Pod不存在，跳过；如果镜像Pod并且原始Pod存在UpdatePod更新
3. 普通Pod直接删除 kl.deletePod

```go
func (kl *Kubelet) HandlePodRemoves(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
		//这个方法从 Kubelet 的内部管理结构中移除指定的 Pod。这是处理 Pod 删除的第一步，确保 Pod 从 Kubelet 的本地状态中被清除
		kl.podManager.RemovePod(pod)
		//这个方法用于获取 Pod 及其对应的镜像 Pod（如果存在的话）
		//pod, mirrorPod, wasMirror 表示原始 Pod，镜像 Pod，以及是否存在镜像 Pod 的布尔值
		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)
		//如果镜像Pod存在
		if wasMirror {
			//如果原始 Pod 不存在	，跳过
			if pod == nil {
				klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
				continue
			}
			//如果原始 Pod 存在，更新Pod
			kl.podWorkers.UpdatePod(UpdatePodOptions{
				Pod:        pod,
				MirrorPod:  mirrorPod,
				UpdateType: kubetypes.SyncPodUpdate,
				StartTime:  start,
			})
			continue
		}

		// Deletion is allowed to fail because the periodic cleanup routine
		// will trigger deletion again.
		//如果是普通pod，直接删除
		if err := kl.deletePod(pod); err != nil {
			klog.V(2).InfoS("Failed to delete pod", "pod", klog.KObj(pod), "err", err)
		}
	}
}
```

* podManager.RemovePod

```go
func (pm *basicManager) RemovePod(pod *v1.Pod) {
	updateMetrics(pod, nil)
	pm.lock.Lock()
	defer pm.lock.Unlock()
	podFullName := kubecontainer.GetPodFullName(pod)
	// It is safe to type convert here due to the IsMirrorPod guard.
	if kubetypes.IsMirrorPod(pod) {
		mirrorPodUID := kubetypes.MirrorPodUID(pod.UID) //取镜像 Pod 的 UID。
		delete(pm.mirrorPodByUID, mirrorPodUID)         //字典中删除此镜像 Pod 的 UID 键
		delete(pm.mirrorPodByFullName, podFullName)     //字典中删除此镜像 Pod 的全名键
		delete(pm.translationByUID, mirrorPodUID)       //删除与此镜像 Pod UID 相关的条目
	} else {
		delete(pm.podByUID, kubetypes.ResolvedPodUID(pod.UID)) // 删除此 Pod 的 UID 键。
		delete(pm.podByFullName, podFullName)                  //删除此 Pod 的全名键
	}
}
```

* 调用podWorkers.UpdatePod的发送SyncPodKill

```go
func (kl *Kubelet) deletePod(pod *v1.Pod) error {
	//pod对象为空返回错误
	if pod == nil {
		return fmt.Errorf("deletePod does not allow nil pod")
	}
	//检查所有 Pod 数据来源是否已经就绪
	if !kl.sourcesReady.AllReady() {
		// If the sources aren't ready, skip deletion, as we may accidentally delete pods
		// for sources that haven't reported yet.
		return fmt.Errorf("skipping delete because sources aren't ready yet")
	}
	klog.V(3).InfoS("Pod has been deleted and must be killed", "pod", klog.KObj(pod), "podUID", pod.UID)
	//更新pod，更新类型是SyncPodKill
	kl.podWorkers.UpdatePod(UpdatePodOptions{
		Pod:        pod,
		UpdateType: kubetypes.SyncPodKill,
	})
	// We leave the volume/directory cleanup to the periodic cleanup routine.
	return nil
}
```

* UpdatePod

如果 `options.UpdateType` 为 `kubetypes.SyncPodKill`，此方法会检查是否设置了终止相关的选项 (`KillPodOptions`)。

如果设置了终止时间戳或 Pod 状态为失败或成功，会更新 Pod 的状态为终止中。

根据 `KillPodOptions` 中可能指定的宽限期和其他终止选项，调整 Pod 的终止过程。

---

# 5.删除流程完整分析

首先`configCh`和`plegCh`都是==>podWorkers.UpdatePod------------>podWorkerLoop----->SyncPod==

当一个`Pod`被标记删除，`API Server` 会设置 Pod 的 `DeletionTimestamp`

`configCh`首先收到一个`Update`事件，**kl *Kubelet) HandlePodUpdates**首先同步本地的kubelet状态（`包含静态Pod和常规Po`d），如果是镜像Pod并且本地Pod为空跳过，然后进入==podWorkers.UpdatePod==，

`podWorkers`结构体主要有podCache的kubelet中的容器信息缓存，podUpdates有变化的pod列表，podSyncStatuses：记录每个 Pod 的同步状态的列表，包括 syncing, terminating, terminated, 和 evicted 状态

==UpdateType: kubetypes.SyncPodUpdate代表这是一次常规更新事件==

* `UpdatePod` :
    * 正常Pod进行更新流程，运行时Pod终止流程
    * DeletionTimestamp！=nil，标记为删除；状态 (`PodFailed`, `PodSucceeded`)代表进入正在终止状态；更新是SyncPodKill也是代表进入正在终止状态
    * 如果已经完全终止，直接进入podUpdates调用podWorkerLoop处理；如果是正在终止，计算一个宽限期，用于优雅停机

* `podWorkerLoop`:
    * 和PLEG本地的缓存进行对比，获取信息比上次的lastSyncTime更新就不用更新
    * ==update.WorkType == TerminatedPod==，调用SyncTerminatedPod：主要就是卸载卷，清除secret或configmap，清除cgroup，释放命名空间（如果启用的话），调用statusManager.SetPodStatus更新Api服务状态，就是把etcd里面的数据删除
    * ==update.WorkType == TerminatingPod==
    * 如果设置了gracePeriod，设置`gracePeriod`为宽限期；调用`acknowledgeTerminating`。主要是设置startedTerminating = true代表终止流程已经启动，返回statusPostTerminating清理函数
        * 如果是运行时Pod，调用`SyncTerminatingRuntimePod`，主要就是掉用killPod强制杀死容器，包括孤儿容器（Api server已经没有信息，但是还在运行），这里`这里初始化gracePeriod=1，代表优雅删除，因为1其实等于=0，其实可以看作强制删除，注释未来可能实现改成0强制删除`
        * 如果时完整Pod，调用`SyncTerminatingPod`，generateAPIPodStatus生成Pod的当前状态，通过statusManager.SetPodStatus更新Api服务状态，就是把etcd里面的数据删除，停止对 Pod 的存活性和启动性探测，`killPod` 来停止容器容器（这里可能时优雅停机，也可能是强制删除），清理资源

* Sync里面也有调用killPod是优雅停机，Sync里面的killPod可能在健康检查失败或 Pod 不能在节点上运行（例如由于资源限制或策略限制者）或者更新回退，错误更新等下调用，逻辑和下面的KillPod一样
* `killPod`主要调用`killContainersWithSyncResult`停止业务容器和`StopPodSandbox` 停止`Sandbox`容器
* killContainersWithSyncResult就不显示整体调用逻辑包含多个函数的重要逻辑：启用了Sidecar容器，按照终止顺序来终止容器，并发执行，如果2提前终止了它会等待1在返回，如果容器不是（CONTAINER_RUNNING）或者（CONTAINER_UNKNOWN`）状态直接返回，尝试获取容器关联的任务的主题如果找不到且未知执行清理；如果容器未指定停止信号，则尝试从关联的镜像配置中获取。如果镜像不存在或获取失败，使用默认的SIGTERM信号
* StopPodSandbox：使用上面的方法清理常规容器，这里是二次清理（因为killPodWithSyncResult 调用stopContainer已经停止了常规容器），使用stopSandboxContainer停止pasue轻量级别的容器。通知nri Sandbox已清理，清理网络IP，网桥设置等
* stopSandboxContainer：停止pause容器：尝试获取沙盒容器的任务，未找到报错，状态未知的容器调用task.Wait(waitCtx)监听退出事件，如果没有“未找到”，则进行清理；最后使用 `syscall.SIGKILL` 强制停止沙盒容器

我的理解如果TerminatedPod就直接进行了，就直接在Api和etcd中删除了；但是如果是正在停止的容器就会调用killPod，他会执行停止容器的动作，当有Stop操作的时候，我的理解是PLEG会触发一次更新：

HandlePodSyncs------>podWorkers.UpdatePod-------->podWorkerLoop----->SyncPod前面已经讲过就不讲了

* SetPodStatus 主要是设置了删除时间戳，就必须更新

* updateStatusInternal

    主要更新podStatus的状态，然后往这个podStatusChannel发送状态

1. `v1.ContainersReady`

2. `v1.PodReady`

3. `v1.PodInitialized`

4. `v1.PodReadyToStartContainers`

5. `v1.PodScheduled`

6. `v1.DisruptionTarget` 

* 在Kubelet.Run的时候, statusManager.start了起来

```
// Start component sync loops.
	kl.statusManager.Start()
	kl.probeManager.Start()
```

Start函数的核心就是用一个协程处理podStatusChannel和syncTicker的数据，syncTicker是每十秒更新一次全量更新Pod，podStatusChannel是单次Pod状态

* `syncBatch`:主要是Pod version+1，一些方法判断值不值得更新
* `syncPod`：主要就是删除Api的Pod状态信息和etcd

这个我的理解是由PLEG监听到的Pod生命周期事件引发的Api删除

* 关于`remove`事件，就是`update`的时候发送`SyncPodKill`事件，和上面的流程进入正在终止的Pod流程一样

关于update事件转换成delete再转换成remove分析：

是因为你的statusManager会上报给etcd信息。上报后etcd更新，informer的watch就会监听到，监听到会判断类型，然后会触发kubelet的UPDATE/RECONCILE/DELETE/REMOVE

apply一个就是触发add，然后containercreate状态下因为spec没变status变了，就会触发reconcile。调谐完成后就正常运行了，等你delete一个pod的时候，会触发delete，然后回到上面的上报，会再次watch到然后delete和reconcile，彻底删除后还会收到然触发remove
