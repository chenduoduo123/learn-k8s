* 删除流程和上一章的流程大概一样，很多代码细节就不展示了，看上章

* configCh和plegCh最终都是调用podWorkerLoop，只是两种更新的类型不一样，结构体也不一样，configCh主要是外部配置变化，plegCh是生命周期变化

    

    ```go
    type UpdatePodOptions struct {
    	// The type of update (create, update, sync, kill).更新事件类型
    	UpdateType kubetypes.SyncPodType
    	// 开始时间戳
    	StartTime time.Time
    	// Pod 更新对象
    	Pod *v1.Pod
    	// MirrorPod is the mirror pod if Pod is a static pod. Optional when UpdateType
    	// is kill or terminated.
    	MirrorPod *v1.Pod
    	// RunningPod is a runtime pod that is no longer present in config. Required
    	// if Pod is nil, ignored if Pod is set.
    	RunningPod *kubecontainer.Pod
    	// KillPodOptions is used to override the default termination behavior of the
    	// pod or to update the pod status after an operation is completed. Since a
    	// pod can be killed for multiple reasons, PodStatusFunc is invoked in order
    	// and later kills have an opportunity to override the status (i.e. a preemption
    	// may be later turned into an eviction).
    	KillPodOptions *KillPodOptions
    }
    
    
    PLEG事件
    ContainerStarted：当容器启动时生成的事件。这表明一个或多个 Pod 中的容器已经开始运行。
    
    ContainerDied：当容器停止运行时生成的事件。这可能是因为容器自然结束、发生错误、或被显式杀死。
    
    ContainerKilled：当容器因资源限制（如内存或 CPU 超限）被杀死时生成的事件。这通常是由 kubelet 的资源管理决策触发。
    
    ContainerRemoved：当容器从系统中移除（例如在清理过程中）时生成的事件。
    
    PodSync：这个事件不是由 PLEG 直接生成的，但是 PLEG 生成的其他事件可能导致 Pod 同步操作。这通常涉及到根据 Pod 的最新状态来启动或停止容器、应用配置更新等。
    
    PodStarted：当 Pod 中的所有容器都已启动并且 Pod 被认为是 "运行中" 时，生成此事件。
    
    PodStopped：当 Pod 中的所有容器都已停止时生成的事件。
    ```

* 不管是configCh还是plegCh最后kubelet进行更新的类型都只有`create`, `update`, `sync`, `kill`

# 1.configCh的Update

**更新 Pod 状态**：通过 `podManager.UpdatePod` 方法，更新 Kubelet 内部存储的 Pod 相关信息。这个步骤确保 Kubelet 的内部状态与从外部接收到的更新信息同步。

**处理镜像 Pod**：对于每个更新的 Pod，使用 `podManager.GetPodAndMirrorPod` 方法尝试获取与之关联的镜像 Pod。镜像 Pod 是 API Server 上的 Pod 的本地镜像，主要用于确保 Kubelet 和 API Server 之间的状态一致性。

**跳过无效的镜像 Pod**：如果更新的 Pod 是一个镜像 Pod 但找不到原始 Pod（可能已被删除），则跳过对该镜像 Pod 的处理。

**提交更新任务**：对每个有效的 Pod 和它的镜像 Pod（如果存在），构造一个 `UpdatePodOptions` 结构并提交给 `podWorkers`。`podWorkers` 负责并行处理这些更新任务，包括同步 Pod 的状态、重启容器等操作。

```go
func (kl *Kubelet) HandlePodUpdates(pods []*v1.Pod) {
    // 记录函数开始执行的时间
    start := kl.clock.Now()

    // 遍历所有传入的 Pods
    for _, pod := range pods {
        // 使用 Pod 管理器更新内部存储的 Pod 状态
        kl.podManager.UpdatePod(pod)

        // 获取 Pod 及其镜像 Pod（如果存在）。如果 Pod 是一个镜像 Pod，返回它和它的原 Pod。
        pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(pod)

        // 如果当前处理的 Pod 是一个镜像 Pod
        if wasMirror {
            // 如果原 Pod 不存在（可能已经被删除），则跳过这次更新
            if pod == nil {
                klog.V(2).InfoS("Unable to find pod for mirror pod, skipping", "mirrorPod", klog.KObj(mirrorPod), "mirrorPodUID", mirrorPod.UID)
                continue
            }
        }

        // 将更新任务提交到 Pod 工作队列中去处理
        kl.podWorkers.UpdatePod(UpdatePodOptions{
            Pod:        pod,          // 正在更新的 Pod
            MirrorPod:  mirrorPod,    // 镜像 Pod（如果存在）
            UpdateType: kubetypes.SyncPodUpdate, // 更新类型，这里是同步更新
            StartTime:  start,        // 任务的开始时间
        })
    }
}
```

* HandlePodUpdates-----> podWorkers.UpdatePod----->podWorkerLoop----->SyncPod----->containerRuntime.SyncPod
* HandlePodUpdates调用podWorkers.UpdatePod为每个pod启用一个携程来处理create, update, sync, kill，podWorkers.UpdatePod里面本身有处理pod更新的逻辑；podWorkerLoop主要接收信号，调用SyncPod，podWorkerLoop本身也有pod的更新逻辑；SyncPod主要生成最终pod的api状态和更新 Pod 状态管理器，根据POD期望状态，启停相关容器，调整POD资源，和配置一样；containerRuntime.SyncPod,输入Pod期望的状态。然后调用cri 对容器执行对应的操作，已达到期望状态

# 2.plegCh的Update

主要是relist方法，前面有讲

* HandlePodSyncs------>podWorkers.UpdatePod-------->podWorkerLoop----->SyncPod------->containerRuntime.SyncPod

* HandlePodSyncs调用podWorkers.UpdatePod为每个pod启用一个携程来处理create, update, sync, kill，podWorkers.UpdatePod里面本身有处理pod更新的逻辑；podWorkerLoop主要接收信号，调用SyncPod，podWorkerLoop本身也有pod的更新逻辑；SyncPod主要生成最终pod的api状态和更新 Pod 状态管理器，根据POD期望状态，启停相关容器，调整POD资源，和配置一样；containerRuntime.SyncPod,输入Pod期望的状态。然后调用cri 对容器执行对应的操作，已达到期望状态
