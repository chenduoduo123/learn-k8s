# 1.探针

**存活探针（Liveness Probe）**

- 作用：检测容器是否存活。如果探针失败，Kubelet 会认为容器处于不健康状态，并将其重启。

**就绪探针（Readiness Probe）**

- 作用：检测容器是否已经准备好接收流量。如果探针失败，Kubelet 会将该容器从服务负载均衡器的后端列表中移除

**启动探针（Startup Probe）**

- 作用：检测容器是否成功启动。如果探针失败，Kubelet 会认为容器启动失败，并将其重启。这个探针主要用于延迟检查容器的启动状态，避免在启动时间较长时被误判为失败。

```go
	case update := <-kl.livenessManager.Updates():
		//proberesults.Failure表示器不健康
		if update.Result == proberesults.Failure {
			handleProbeSync(kl, update, handler, "liveness", "unhealthy")
		}
		//就緒探針
	case update := <-kl.readinessManager.Updates():
		//根据探针结果是否为 proberesults.Success，将 ready 设置为 true 或 false。
		ready := update.Result == proberesults.Success
		kl.statusManager.SetContainerReadiness(update.PodUID, update.ContainerID, ready)
		//设置状态字符串 status，如果探针结果为成功，则设置为 "ready"。
		status := ""
		if ready {
			status = "ready"
		}
		handleProbeSync(kl, update, handler, "readiness", status)
		//启动探針
	case update := <-kl.startupManager.Updates():
		started := update.Result == proberesults.Success
		//根据探针结果是否为 proberesults.Success，将 started 设置为 true 或 false。
		kl.statusManager.SetContainerStartup(update.PodUID, update.ContainerID, started)
		//设置状态字符串 status，默认是 "unhealthy"，如果探针结果为成功，则设置为 "started"。
		status := "unhealthy"
		if started {
			status = "started"
		}
		handleProbeSync(kl, update, handler, "startup", status)
```

handleProbeSync----->HandlePodSyncs------------>podWorkers.UpdatePod----->podWorkerLoop----->SyncPod

handleProbeSync如果Pod已经不存在，忽略更新；HandlePodSyncs调用podWorkers.UpdatePod为每个pod启用一个携程来处理create, update, sync, kill，podWorkerLoop主要接收信号，调用SyncPod

#### SyncPod添加健康检查探针

在这段代码中，探针的添加通过以下代码实现：

```
// Ensure the pod is being probed
kl.probeManager.AddPod(pod)
```

`kl.probeManager.AddPod(pod)` 方法会为给定的Pod配置健康检查探针。探针包括存活探针、就绪探针和启动探针。这些探针会帮助Kubelet确定Pod中的容器是否健康并且准备好接受流量。

#### 处理探针结果的影响

虽然代码中没有直接处理探针结果的逻辑，但探针结果会影响Pod的状态，并且这些状态会在 `SyncPod` 函数中被处理。例如，如果探针结果表明容器不健康，Kubelet会相应地终止和重启容器。这些逻辑在以下部分中有所体现：

```
go复制代码// 如果 Pod 的阶段是成功完成（PodSucceeded）或失败（PodFailed），则更新 Pod 状态，并标记为终端状态，返回终端状态和无错误
if apiPodStatus.Phase == v1.PodSucceeded || apiPodStatus.Phase == v1.PodFailed {
	kl.statusManager.SetPodStatus(pod, apiPodStatus)
	isTerminal = true
	return isTerminal, nil
}

// 检查Pod是否允许运行
runnable := kl.canRunPod(pod)
if !runnable.Admit {
	if apiPodStatus.Phase != v1.PodFailed && apiPodStatus.Phase != v1.PodSucceeded {
		apiPodStatus.Phase = v1.PodPending
	}
	apiPodStatus.Reason = runnable.Reason
	apiPodStatus.Message = runnable.Message
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
```

在这些部分中，如果探针结果表明Pod已成功完成或失败，Kubelet会更新Pod的状态。如果Pod不允许运行（例如，由于探针失败导致Pod被标记为不健康），Kubelet会阻止Pod运行并更新其状态。

---

# 2.housekeepingCh

* 执行定期的系统维护任务，如清理无用的资源

HandlePodCleanups

1. 同步Pod的最新信息
2. 清理不再运行的 Pods 的探针
3. 清除已知 Pod 列表中不存在的孤立 Pod 状态
4. 清理用户命名空间和Pod目录
5. 删除孤立的镜像 Pod（API server）
6. 如果由Pod UID重复或者同步错误，就进行重启或者创建
7. 处理删除但未运行的 Pods和处理终止且未知的 Pods，然后清理
8. 清理容器运行时Pods，就是 Docker、containerd中有记录，但是kubelet中没有记录的Pods
9. 清除cgroup和回退

### QoS cgroups

在 Kubernetes 中，QoS（Quality of Service）cgroups 是一种利用 Linux cgroup（控制组）功能来实现资源管理的方法。Kubernetes 为每个 Pod 分配一个 QoS 类别，基于它们的资源请求和限制，分为以下三种：

- **Guaranteed**：如果 Pod 中所有容器的 CPU 和内存资源都有明确的请求和限制，并且它们相等。
- **Burstable**：如果 Pod 中任何容器有 CPU 或内存的请求或限制。
- **BestEffort**：如果 Pod 中的容器没有提出任何资源请求或限制

### Pod 的状态进展是单调的

Pod 的生命周期状态（Phase）是单向且单调递增的，这意味着一旦 Pod 进入了一个最终状态（如 `Succeeded` 或 `Failed`），它就不应该再改变状态。在 Kubernetes 中，Pod 的生命周期状态包括：

- **Pending**：Pod 已被创建，但某些条件尚未满足（如容器镜像尚未完全下载）。
- **Running**：Pod 已被绑定到一个节点，所有容器都已创建，至少有一个容器正在运行。
- **Succeeded**：Pod 中的所有容器都正常运行完毕，并且已经退出。
- **Failed**：Pod 中的一个或多个容器已经失败（退出状态非零）。
- **Unknown**：因为某些原因，Pod 的状态不能被确定。

```go
// HandlePodCleanups performs a series of cleanup work, including terminating
// pod workers, killing unwanted pods, and removing orphaned volumes/pod
// directories. No config changes are sent to pod workers while this method
// is executing which means no new pods can appear. After this method completes
// the desired state of the kubelet should be reconciled with the actual state
// in the pod worker and other pod-related components.
//
// This function is executed by the main sync loop, so it must execute quickly
// and all nested calls should be asynchronous. Any slow reconciliation actions
// should be performed by other components (like the volume manager). The duration
// of this call is the minimum latency for static pods to be restarted if they
// are updated with a fixed UID (most should use a dynamic UID), and no config
// updates are delivered to the pod workers while this method is running.
func (kl *Kubelet) HandlePodCleanups(ctx context.Context) error {
	// The kubelet lacks checkpointing, so we need to introspect the set of pods
	// in the cgroup tree prior to inspecting the set of pods in our pod manager.
	// this ensures our view of the cgroup tree does not mistakenly observe pods
	// that are added after the fact...
	var (
		cgroupPods map[types.UID]cm.CgroupName
		err        error
	)
	//检查并获取 cgroup 中的 Pod 信息（如果启用了 QoS cgroups）
	if kl.cgroupsPerQOS {
		pcm := kl.containerManager.NewPodContainerManager()
		cgroupPods, err = pcm.GetAllPodsFromCgroups()
		if err != nil {
			return fmt.Errorf("failed to get list of pods that still exist on cgroup mounts: %v", err)
		}
	}
	//从 Pod 管理器中获取所有的 Pod，包括镜像 Pod 和孤立的镜像 Pod 全名
	allPods, mirrorPods, orphanedMirrorPodFullnames := kl.podManager.GetPodsAndMirrorPods()

	// Pod phase progresses monotonically. Once a pod has reached a final state,
	// it should never leave regardless of the restart policy. The statuses
	// of such pods should not be changed, and there is no need to sync them.
	/*
				容器在死亡后立即被移除：如果容器一旦停止运行就立即被删除，Kubelet 可能无法生成正确的状态或进行正确的过滤。这可能导致状态同步问题，
			因为如果容器的信息已经从系统中删除，Kubelet 可能无法检索到足够的信息来准确报告 Pod 的状态。

				Kubelet 在写入终止状态之前重启：如果 Kubelet 在将 Pod 的终止状态写入到 API 服务器前发生了重启，这个已终止的 Pod 可能会被再次启动，
			即使按照 API 服务器的视角，这个 Pod 并未被视为已终止。这是因为 Pod 的终止信息未被成功持久化，导致状态丢失。
			未来潜在的解决办法：
		Kubelet 可以定期保存其当前状态到一个持久化存储。这样，即使 Kubelet 进程崩溃或重启，它也可以从最后的检查点恢复，
	*/
	// TODO: the logic here does not handle two cases:
	//   1. If the containers were removed immediately after they died, kubelet
	//      may fail to generate correct statuses, let alone filtering correctly.
	//   2. If kubelet restarted before writing the terminated status for a pod
	//      to the apiserver, it could still restart the terminated pod (even
	//      though the pod was not considered terminated by the apiserver).
	// These two conditions could be alleviated by checkpointing kubelet.

	// Stop the workers for terminated pods not in the config source
	klog.V(3).InfoS("Clean up pod workers for terminated pods")
	//同步已知的 Pod 到 Pod 工作器中，确保 Pod 工作器的状态与 Kubelet 的当前视图一致,已终止的Pod就不
	workingPods := kl.podWorkers.SyncKnownPods(allPods)

	// Reconcile: At this point the pod workers have been pruned to the set of
	// desired pods. Pods that must be restarted due to UID reuse, or leftover
	// pods from previous runs, are not known to the pod worker.

	allPodsByUID := make(map[types.UID]*v1.Pod)
	for _, pod := range allPods {
		allPodsByUID[pod.UID] = pod
	}

	// Identify the set of pods that have workers, which should be all pods
	// from config that are not terminated, as well as any terminating pods
	// that have already been removed from config. Pods that are terminating
	// will be added to possiblyRunningPods, to prevent overly aggressive
	// cleanup of pod cgroups.
	stringIfTrue := func(t bool) string {
		if t {
			return "true"
		}
		return ""
	}
	runningPods := make(map[types.UID]sets.Empty)
	possiblyRunningPods := make(map[types.UID]sets.Empty)
	for uid, sync := range workingPods {
		switch sync.State {
		case SyncPod:
			runningPods[uid] = struct{}{}
			possiblyRunningPods[uid] = struct{}{}
		case TerminatingPod:
			possiblyRunningPods[uid] = struct{}{}
		default:
		}
	}

	// Retrieve the list of running containers from the runtime to perform cleanup.
	// We need the latest state to avoid delaying restarts of static pods that reuse
	// a UID.
	if err := kl.runtimeCache.ForceUpdateIfOlder(ctx, kl.clock.Now()); err != nil {
		klog.ErrorS(err, "Error listing containers")
		return err
	}
	runningRuntimePods, err := kl.runtimeCache.GetPods(ctx)
	if err != nil {
		klog.ErrorS(err, "Error listing containers")
		return err
	}
	//清理不再运行的 Pods 的探针，避免无效的资源消耗。
	// Stop probing pods that are not running
	klog.V(3).InfoS("Clean up probes for terminated pods")
	kl.probeManager.CleanupPods(possiblyRunningPods)
	//清除已知 Pod 列表中不存在的孤立 Pod 状态，维护状态管理器的准确性。
	// Remove orphaned pod statuses not in the total list of known config pods
	klog.V(3).InfoS("Clean up orphaned pod statuses")
	kl.removeOrphanedPodStatuses(allPods, mirrorPods)

	// Remove orphaned pod user namespace allocations (if any).
	// 清理用户命名空间
	klog.V(3).InfoS("Clean up orphaned pod user namespace allocations")
	if err = kl.usernsManager.CleanupOrphanedPodUsernsAllocations(allPods, runningRuntimePods); err != nil {
		klog.ErrorS(err, "Failed cleaning up orphaned pod user namespaces allocations")
	}

	// Remove orphaned volumes from pods that are known not to have any
	// containers. Note that we pass all pods (including terminated pods) to
	// the function, so that we don't remove volumes associated with terminated
	// but not yet deleted pods.
	// TODO: this method could more aggressively cleanup terminated pods
	// in the future (volumes, mount dirs, logs, and containers could all be
	// better separated)
	//清理孤立的Pod目录
	klog.V(3).InfoS("Clean up orphaned pod directories")
	err = kl.cleanupOrphanedPodDirs(allPods, runningRuntimePods)
	if err != nil {
		// We want all cleanup tasks to be run even if one of them failed. So
		// we just log an error here and continue other cleanup tasks.
		// This also applies to the other clean up tasks.
		klog.ErrorS(err, "Failed cleaning up orphaned pod directories")
	}

	// Remove any orphaned mirror pods (mirror pods are tracked by name via the
	// pod worker)
	//删除孤立的镜像 Pod
	klog.V(3).InfoS("Clean up orphaned mirror pods")
	for _, podFullname := range orphanedMirrorPodFullnames {
		if !kl.podWorkers.IsPodForMirrorPodTerminatingByFullName(podFullname) {
			_, err := kl.mirrorPodClient.DeleteMirrorPod(podFullname, nil)
			if err != nil {
				klog.ErrorS(err, "Encountered error when deleting mirror pod", "podName", podFullname)
			} else {
				klog.V(3).InfoS("Deleted mirror pod", "podName", podFullname)
			}
		}
	}

	// After pruning pod workers for terminated pods get the list of active pods for
	// metrics and to determine restarts.
	activePods := kl.filterOutInactivePods(allPods)
	allRegularPods, allStaticPods := splitPodsByStatic(allPods)
	activeRegularPods, activeStaticPods := splitPodsByStatic(activePods)
	metrics.DesiredPodCount.WithLabelValues("").Set(float64(len(allRegularPods)))
	metrics.DesiredPodCount.WithLabelValues("true").Set(float64(len(allStaticPods)))
	metrics.ActivePodCount.WithLabelValues("").Set(float64(len(activeRegularPods)))
	metrics.ActivePodCount.WithLabelValues("true").Set(float64(len(activeStaticPods)))
	metrics.MirrorPodCount.Set(float64(len(mirrorPods)))

	// At this point, the pod worker is aware of which pods are not desired (SyncKnownPods).
	// We now look through the set of active pods for those that the pod worker is not aware of
	// and deliver an update. The most common reason a pod is not known is because the pod was
	// deleted and recreated with the same UID while the pod worker was driving its lifecycle (very
	// very rare for API pods, common for static pods with fixed UIDs). Containers that may still
	// be running from a previous execution must be reconciled by the pod worker's sync method.
	// We must use active pods because that is the set of admitted pods (podManager includes pods
	// that will never be run, and statusManager tracks already rejected pods).
	//检查是否有 Pods 因为 UID 重用而需要重启。这一步很关键，特别是对于静态 Pods
	//对于静态Pod来说他在Api server是镜像Pod，如果状态同步错误或者UID重用都会UpdateType: kubetypes.SyncPodCreate,来重启或者创建Pod
	var restartCount, restartCountStatic int
	for _, desiredPod := range activePods {
		if _, knownPod := workingPods[desiredPod.UID]; knownPod {
			continue
		}

		klog.V(3).InfoS("Pod will be restarted because it is in the desired set and not known to the pod workers (likely due to UID reuse)", "podUID", desiredPod.UID)
		isStatic := kubetypes.IsStaticPod(desiredPod)
		pod, mirrorPod, wasMirror := kl.podManager.GetPodAndMirrorPod(desiredPod)
		if pod == nil || wasMirror {
			klog.V(2).InfoS("Programmer error, restartable pod was a mirror pod but activePods should never contain a mirror pod", "podUID", desiredPod.UID)
			continue
		}
		//SyncPodCreate來 Pod 将被创建或重启
		kl.podWorkers.UpdatePod(UpdatePodOptions{
			UpdateType: kubetypes.SyncPodCreate,
			Pod:        pod,
			MirrorPod:  mirrorPod,
		})

		// the desired pod is now known as well
		workingPods[desiredPod.UID] = PodWorkerSync{State: SyncPod, HasConfig: true, Static: isStatic}
		if isStatic {
			// restartable static pods are the normal case
			restartCountStatic++
		} else {
			// almost certainly means shenanigans, as API pods should never have the same UID after being deleted and recreated
			// unless there is a major API violation
			restartCount++
		}
	}
	metrics.RestartedPodTotal.WithLabelValues("true").Add(float64(restartCountStatic))
	metrics.RestartedPodTotal.WithLabelValues("").Add(float64(restartCount))

	// Complete termination of deleted pods that are not runtime pods (don't have
	// running containers), are terminal, and are not known to pod workers.
	// An example is pods rejected during kubelet admission that have never
	// started before (i.e. does not have an orphaned pod).
	// Adding the pods with SyncPodKill to pod workers allows to proceed with
	// force-deletion of such pods, yet preventing re-entry of the routine in the
	// next invocation of HandlePodCleanups.
	//终止特殊Pod，然后使用UpdateType: kubetypes.SyncPodKill进行清理
	//处理删除但未运行的 Pods：这指的是那些已被删除（在 API 中不存在），但实际上在运行时（如 Docker 或其他容器运行时）中并没有运行容器的 Pods。
	//处理终止且未知的 Pods：这些 Pods 可能因为在 kubelet 准入阶段被拒绝而从未启动过，也没有留下“孤儿”Pod 的痕迹（即在节点上没有相关容器或数据残留）。
	//防止重复处理：通过添加这些 Pods 到 Pod 工作器并标记为 SyncPodKill，可以确保在下一次调用 HandlePodCleanups 时不会再次进入此清理流程。
	for _, pod := range kl.filterTerminalPodsToDelete(allPods, runningRuntimePods, workingPods) {
		klog.V(3).InfoS("Handling termination and deletion of the pod to pod workers", "pod", klog.KObj(pod), "podUID", pod.UID)
		kl.podWorkers.UpdatePod(UpdatePodOptions{
			UpdateType: kubetypes.SyncPodKill,
			Pod:        pod,
		})
	}

	// Finally, terminate any pods that are observed in the runtime but not present in the list of
	// known running pods from config. If we do terminate running runtime pods that will happen
	// asynchronously in the background and those will be processed in the next invocation of
	// HandlePodCleanups.
	//清理容器运行时Pods，就是 Docker、containerd中有记录，但是kubelet中没有记录的Pods
	var orphanCount int
	for _, runningPod := range runningRuntimePods {
		// If there are orphaned pod resources in CRI that are unknown to the pod worker, terminate them
		// now. Since housekeeping is exclusive to other pod worker updates, we know that no pods have
		// been added to the pod worker in the meantime. Note that pods that are not visible in the runtime
		// but which were previously known are terminated by SyncKnownPods().
		_, knownPod := workingPods[runningPod.ID]
		if !knownPod {
			one := int64(1)
			killPodOptions := &KillPodOptions{
				PodTerminationGracePeriodSecondsOverride: &one,
			}
			klog.V(2).InfoS("Clean up containers for orphaned pod we had not seen before", "podUID", runningPod.ID, "killPodOptions", killPodOptions)
			kl.podWorkers.UpdatePod(UpdatePodOptions{
				UpdateType:     kubetypes.SyncPodKill,
				RunningPod:     runningPod,
				KillPodOptions: killPodOptions,
			})

			// the running pod is now known as well
			workingPods[runningPod.ID] = PodWorkerSync{State: TerminatingPod, Orphan: true}
			orphanCount++
		}
	}
	metrics.OrphanedRuntimePodTotal.Add(float64(orphanCount))

	// Now that we have recorded any terminating pods, and added new pods that should be running,
	// record a summary here. Not all possible combinations of PodWorkerSync values are valid.
	//统计Pod状态，更新指标
	counts := make(map[PodWorkerSync]int)
	for _, sync := range workingPods {
		counts[sync]++
	}
	//Static静态Pod；Orphan孤儿Pod；HasConfig有配置
	for validSync, configState := range map[PodWorkerSync]string{
		{HasConfig: true, Static: true}:                "desired",
		{HasConfig: true, Static: false}:               "desired",
		{Orphan: true, HasConfig: true, Static: true}:  "orphan",
		{Orphan: true, HasConfig: true, Static: false}: "orphan",
		{Orphan: true, HasConfig: false}:               "runtime_only",
	} {
		for _, state := range []PodWorkerState{SyncPod, TerminatingPod, TerminatedPod} {
			validSync.State = state
			count := counts[validSync]
			delete(counts, validSync)
			staticString := stringIfTrue(validSync.Static)
			if !validSync.HasConfig {
				staticString = "unknown"
			}
			metrics.WorkingPodCount.WithLabelValues(state.String(), configState, staticString).Set(float64(count))
		}
	}
	if len(counts) > 0 {
		// in case a combination is lost
		klog.V(3).InfoS("Programmer error, did not report a kubelet_working_pods metric for a value returned by SyncKnownPods", "counts", counts)
	}

	// Remove any cgroups in the hierarchy for pods that are definitely no longer
	// running (not in the container runtime).
	//如果启用了基于 QoS 的 cgroups，此代码块负责清理那些孤立的 Pod cgroups，即那些不再与任何运行中的 Pods 关联的 cgroups。
	//通过调用 cleanupOrphanedPodCgroups 方法，确保不再需要的资源被回收，维护系统资源的整洁性和效率。
	if kl.cgroupsPerQOS {
		pcm := kl.containerManager.NewPodContainerManager()
		klog.V(3).InfoS("Clean up orphaned pod cgroups")
		kl.cleanupOrphanedPodCgroups(pcm, cgroupPods, possiblyRunningPods)
	}

	// Cleanup any backoff entries.
	kl.backOff.GC()
	return nil
}
```

### Kubernetes 垃圾回收的关键组件：

1. **容器垃圾回收**：停止并删除所有不再需要的容器实例。这包括由于 Pod 生命周期结束、配置更新或系统重调度导致的容器终止。
2. **Pod sandbox 清理**：每个 Pod 都在一个特定的 sandbox 环境中运行，这是隔离 Pod 内部各个容器的环境。当 Pod 不再需要时，相关的 sandbox 环境也会被清理。
3. **文件系统目录清理**：删除与已终止 Pods 相关联的文件系统上的目录，如日志文件、容器数据等。
4. **镜像清理**：删除不再需要的容器镜像，尤其是在磁盘空间紧张时，以释放存储资源。

### `HandlePodCleanups` 函数与垃圾回收的结合

`HandlePodCleanups` 函数在这个垃圾回收过程中扮演了一个特别的角色，它专注于清理与 Pod 相关的各类资源。其具体职责包括：

- **终止不需要的 Pod 工作器**：对于已经在 API 服务器中标记为删除的 Pods，确保它们在节点上的实际资源也被释放。
- **清理孤立的 Pod 资源**：对于那些不再有任何 Kubernetes 对象引用的 Pods（比如由于异常情况造成的孤立 Pods），清理它们占用的资源，包括 cgroups、用户命名空间、Pod 文件系统目录等。
- **删除孤立的镜像 Pods**：清理那些不再对应任何 API Pod 的镜像 Pods，这些通常是由节点上的 Kubelet 直接创建的。
- **回收探针资源**：停止对已终止或不再需要监控的 Pods 进行健康检查。
- **同步容器运行时的状态**：确保容器运行时的状态与期望状态一致，包括停止那些不应该运行的容器。