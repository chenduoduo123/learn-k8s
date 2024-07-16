### 1. 背景

上文分析到，所有的容器都是stop停了，但是没有清理。这个清理工作就是GC做的。在kubelet初始化的时createAndInitKubelet函数中，开启了gc 流程。接下里看看GC流程的处理逻辑。

```go
cmd/kubelet/app/server.go
createAndInitKubelet

k.StartGarbageCollection()
```

### 2. StartGarbageCollection

StartGarbageCollection逻辑如下：

（1） 开启一个协程进行container的GC处理。间隔时间1分钟，ContainerGCPeriod=1 min

（2）判断是否开启了镜像最大使用率100%和镜像的最大存活时间被禁用或设置为 0则不清理镜像

（3）如果开了image gc，协程进行imageManager.GarbageCollect处理。间隔时间5分钟，ImageGCPeriod=5 min

可以看到，kubelet的核心就是contianer Gc 和imageManager.GarbageCollect

```go
// StartGarbageCollection starts garbage collection threads.
func (kl *Kubelet) StartGarbageCollection() {
	//用于跟踪容器垃圾回收是否曾失败过
	loggedContainerGCFailure := false
	go wait.Until(func() {
		ctx := context.Background()
		//不停循环执行垃圾回收kl.containerGC.GarbageCollect，成功loggedContainerGCFailure=true，ContainerGCPeriod间隔一分钟执行一次
		if err := kl.containerGC.GarbageCollect(ctx); err != nil {
			klog.ErrorS(err, "Container garbage collection failed")
			kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.ContainerGCFailed, err.Error())
			loggedContainerGCFailure = true
		} else {
			var vLevel klog.Level = 4
			if loggedContainerGCFailure {
				vLevel = 1
				loggedContainerGCFailure = false
			}

			klog.V(vLevel).InfoS("Container garbage collection succeeded")
		}
	}, ContainerGCPeriod, wait.NeverStop)

	// when the high threshold is set to 100, and the max age is 0 (or the max age feature is disabled)
	// stub the image GC manager
	//如果镜像垃圾回收的阈值设置为 100% 且镜像的最大存活时间被禁用或设置为 0，则不启用镜像垃圾回收
	if kl.kubeletConfiguration.ImageGCHighThresholdPercent == 100 &&
		(!utilfeature.DefaultFeatureGate.Enabled(features.ImageMaximumGCAge) || kl.kubeletConfiguration.ImageMaximumGCAge.Duration == 0) {
		klog.V(2).InfoS("ImageGCHighThresholdPercent is set 100 and ImageMaximumGCAge is 0, Disable image GC")
		return
	}
	//判断镜像是否成功清理
	prevImageGCFailed := false
	beganGC := time.Now()
	go wait.Until(func() {
		ctx := context.Background()
		//调用kl.imageManager.GarbageCollect清理镜像，每隔5分钟
		if err := kl.imageManager.GarbageCollect(ctx, beganGC); err != nil {
			if prevImageGCFailed {
				klog.ErrorS(err, "Image garbage collection failed multiple times in a row")
				// Only create an event for repeated failures
				kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.ImageGCFailed, err.Error())
			} else {
				klog.ErrorS(err, "Image garbage collection failed once. Stats initialization may not have completed yet")
			}
			prevImageGCFailed = true
		} else {
			var vLevel klog.Level = 4
			if prevImageGCFailed {
				vLevel = 1
				prevImageGCFailed = false
			}

			klog.V(vLevel).InfoS("Image garbage collection succeeded")
		}
	}, ImageGCPeriod, wait.NeverStop)
}
```

### 3. container gc处理流程

#### 3.1 gc 参数设置

GCPolicy 结构如下：

**`MinAge time.Duration`**:

- 对应于 `--minimum-container-ttl-duration` 启动参数，这个字段指定了一个容器被垃圾回收前必须处于终止状态的最小时间。如果设置为 `0s`（默认值），则表示没有最小存活时间限制，容器一旦终止就可以立即被回收。

**`MaxPerPodContainer int`**:

- 对应于 `--maximum-dead-containers-per-container` 启动参数，这个字段设定了每个 Pod（具体到每个容器名称和UID的组合）可以保留的最大终止容器数量。如果设置为负数，则表示没有限制。

**`MaxContainers int`**:

- 对应于 `--maximum-dead-containers` 启动参数，这个字段指定了在整个节点上可以保留的终止容器的总数。如果设置为 `-1`（默认值），则表示没有数量限制。

```
// GCPolicy specifies a policy for garbage collecting containers.
type GCPolicy struct {
	// Minimum age at which a container can be garbage collected, zero for no limit.
	MinAge time.Duration

	// Max number of dead containers any single pod (UID, container name) pair is
	// allowed to have, less than zero for no limit.
	MaxPerPodContainer int

	// Max number of total dead containers, less than zero for no limit.
	MaxContainers int
}
```

#### 3.2 GarbageCollect

调用链为：GarbageCollect -> runtime.GarbageCollect -> containerGC.GarbageCollect

```go
func (cgc *realContainerGC) GarbageCollect() error {
	return cgc.runtime.GarbageCollect(cgc.policy, cgc.sourcesReadyProvider.AllReady(), false)
}

// GarbageCollect removes dead containers using the specified container gc policy.
func (m *kubeGenericRuntimeManager) GarbageCollect(gcPolicy kubecontainer.ContainerGCPolicy, allSourcesReady bool, evictNonDeletedPods bool) error {
	return m.containerGC.GarbageCollect(gcPolicy, allSourcesReady, evictNonDeletedPods)
}
```

#### 3.3 containerGC.GarbageCollect

1. 调用 `evictContainers` 方法清理可清除的容器。
2. 调用 `evictSandboxes` 方法来清理已经不包含任何容器的Sandbox
3. 调用 `evictPodLogsDirectories` 方法清理与Pod相关的日志目录

```go
// GarbageCollect removes dead containers using the specified container gc policy.
// Note that gc policy is not applied to sandboxes. Sandboxes are only removed when they are
// not ready and containing no containers.
//
// GarbageCollect consists of the following steps:
// * gets evictable containers which are not active and created more than gcPolicy.MinAge ago.
// * removes oldest dead containers for each pod by enforcing gcPolicy.MaxPerPodContainer.
// * removes oldest dead containers by enforcing gcPolicy.MaxContainers.
// * gets evictable sandboxes which are not ready and contains no containers.
// * removes evictable sandboxes.
func (cgc *containerGC) GarbageCollect(ctx context.Context, gcPolicy kubecontainer.GCPolicy, allSourcesReady bool, evictNonDeletedPods bool) error {
	ctx, otelSpan := cgc.tracer.Start(ctx, "Containers/GarbageCollect")
	defer otelSpan.End()
	errors := []error{}
	// Remove evictable containers
	if err := cgc.evictContainers(ctx, gcPolicy, allSourcesReady, evictNonDeletedPods); err != nil {
		errors = append(errors, err)
	}

	// Remove sandboxes with zero containers
	if err := cgc.evictSandboxes(ctx, evictNonDeletedPods); err != nil {
		errors = append(errors, err)
	}

	// Remove pod sandbox log directory
	if err := cgc.evictPodLogsDirectories(ctx, allSourcesReady); err != nil {
		errors = append(errors, err)
	}
	return utilerrors.NewAggregate(errors)
}
```

##### 3.2.1 移除需要驱逐的的containers

（1）得到需要所有需要驱逐的contaienrs，非running已经挂掉 MinAge分钟了

（2）驱逐contaienrs，检查MaxPerPodContainer是否超过了最大容器数，超过就驱逐

（3）根据MaxContainers参数，保留最新的MaxContainers个contaeinrs，其他的按照创建时间依次驱逐

```go
// evict all containers that are evictable
func (cgc *containerGC) evictContainers(ctx context.Context, gcPolicy kubecontainer.GCPolicy, allSourcesReady bool, evictNonDeletedPods bool) error {
	// Separate containers by evict units.
	evictUnits, err := cgc.evictableContainers(ctx, gcPolicy.MinAge)
	if err != nil {
		return err
	}

	// Remove deleted pod containers if all sources are ready.
	if allSourcesReady {
		for key, unit := range evictUnits {
			if cgc.podStateProvider.ShouldPodContentBeRemoved(key.uid) || (evictNonDeletedPods && cgc.podStateProvider.ShouldPodRuntimeBeRemoved(key.uid)) {
				cgc.removeOldestN(ctx, unit, len(unit)) // Remove all.
				delete(evictUnits, key)
			}
		}
	}

	// Enforce max containers per evict unit.
	if gcPolicy.MaxPerPodContainer >= 0 {
		cgc.enforceMaxContainersPerEvictUnit(ctx, evictUnits, gcPolicy.MaxPerPodContainer)
	}

	// Enforce max total number of containers.
	if gcPolicy.MaxContainers >= 0 && evictUnits.NumContainers() > gcPolicy.MaxContainers {
		// Leave an equal number of containers per evict unit (min: 1).
		numContainersPerEvictUnit := gcPolicy.MaxContainers / evictUnits.NumEvictUnits()
		if numContainersPerEvictUnit < 1 {
			numContainersPerEvictUnit = 1
		}
		cgc.enforceMaxContainersPerEvictUnit(ctx, evictUnits, numContainersPerEvictUnit)

		// If we still need to evict, evict oldest first.
		numContainers := evictUnits.NumContainers()
		if numContainers > gcPolicy.MaxContainers {
			flattened := make([]containerGCInfo, 0, numContainers)
			for key := range evictUnits {
				flattened = append(flattened, evictUnits[key]...)
			}
			sort.Sort(byCreated(flattened))

			cgc.removeOldestN(ctx, flattened, numContainers-gcPolicy.MaxContainers)
		}
	}
	return nil
}
```

##### 3.2.2 移除sandboxes

该函数逻辑如下：

（1）获取 node 上所有的 container以及所有的 sandboxes

（2）收集所有 container 的 PodSandboxId， 构建 sandboxes 与 pod 的对应关系并将其保存在 sandboxesByPodUID 中

（3）遍历 sandboxesByPod，若 sandboxes 所在的 pod 处于 deleted 状态或者被标记evictNonDeletedPods，则删除该 pod 中所有的 sandboxes ；如果Pod未被删除，通常会保留最新创建的sandbox；如果Pod已存在至少一个处于活跃状态的沙盒，则删除所有非活跃的沙盒。

```go
// evictSandboxes remove all evictable sandboxes. An evictable sandbox must
// meet the following requirements:
//  1. not in ready state
//  2. contains no containers.
//  3. belong to a non-existent (i.e., already removed) pod, or is not the
//     most recently created sandbox for the pod.
func (cgc *containerGC) evictSandboxes(ctx context.Context, evictNonDeletedPods bool) error {
	containers, err := cgc.manager.getKubeletContainers(ctx, true)
	if err != nil {
		return err
	}

	sandboxes, err := cgc.manager.getKubeletSandboxes(ctx, true)
	if err != nil {
		return err
	}

	// collect all the PodSandboxId of container
	sandboxIDs := sets.NewString()
	for _, container := range containers {
		sandboxIDs.Insert(container.PodSandboxId)
	}

	sandboxesByPod := make(sandboxesByPodUID, len(sandboxes))
	for _, sandbox := range sandboxes {
		podUID := types.UID(sandbox.Metadata.Uid)
		sandboxInfo := sandboxGCInfo{
			id:         sandbox.Id,
			createTime: time.Unix(0, sandbox.CreatedAt),
		}

		// Set ready sandboxes and sandboxes that still have containers to be active.
		if sandbox.State == runtimeapi.PodSandboxState_SANDBOX_READY || sandboxIDs.Has(sandbox.Id) {
			sandboxInfo.active = true
		}

		sandboxesByPod[podUID] = append(sandboxesByPod[podUID], sandboxInfo)
	}

	for podUID, sandboxes := range sandboxesByPod {
		if cgc.podStateProvider.ShouldPodContentBeRemoved(podUID) || (evictNonDeletedPods && cgc.podStateProvider.ShouldPodRuntimeBeRemoved(podUID)) {
			// Remove all evictable sandboxes if the pod has been removed.
			// Note that the latest dead sandbox is also removed if there is
			// already an active one.
			cgc.removeOldestNSandboxes(ctx, sandboxes, len(sandboxes))
		} else {
			// Keep latest one if the pod still exists.
			cgc.removeOldestNSandboxes(ctx, sandboxes, len(sandboxes)-1)
		}
	}
	return nil
}
```

##### 3.2.3 回收log Directories

该方法会回收所有可回收 pod 以及 container 的 log dir，其主要逻辑为： 

（1）首先回收 deleted 状态 pod logs dir，遍历 pod logs dir /var/log/pods，/var/log/pods 为 pod logs 的默认目录，pod logs dir 的格式为 /var/log/pods/NAMESPACE_NAME_UID，解析 pod logs dir 获取 pod uid，，确定其对应的Pod是否已经不再存在。如果Pod已被删除，那么删除该Pod的日志目录。"

（2）遍历 `/var/log/containers` 目录中的所有日志软链接。首先检查每个链接是否指向有效的日志文件。如果发现链接已断开（即日志文件不存在），则尝试解析出与该链接相关的容器ID。如果获取到容器ID并确定容器已经退出，那么删除这个软链接，以清理不再需要的日志引用。如果容器仍在运行，保留该链接，避免在日志轮换过程中意外删除有效的日志链接。

/var/log/containers 为 container log 的默认目录，其会软链接到 pod 的 log dir 下，例如： /var/log/containers/storage-provisioner_kube-system_storage-provisioner-acc8386e409dfb3cc01618cbd14c373d8ac6d7f0aaad9ced018746f31d0081e2.log -> /var/log/pods/kube-system_storage-provisioner_b448e496-eb5d-4d71-b93f-ff7ff77d2348/storage-provisioner/0.log

```go

// evictPodLogsDirectories evicts all evictable pod logs directories. Pod logs directories
// are evictable if there are no corresponding pods.
func (cgc *containerGC) evictPodLogsDirectories(ctx context.Context, allSourcesReady bool) error {
	osInterface := cgc.manager.osInterface
	podLogsDirectory := cgc.manager.podLogsDirectory
	if allSourcesReady {
		// Only remove pod logs directories when all sources are ready.
		dirs, err := osInterface.ReadDir(podLogsDirectory)
		if err != nil {
			return fmt.Errorf("failed to read podLogsDirectory %q: %w", podLogsDirectory, err)
		}
		for _, dir := range dirs {
			name := dir.Name()
			podUID := parsePodUIDFromLogsDirectory(name)
			if !cgc.podStateProvider.ShouldPodContentBeRemoved(podUID) {
				continue
			}
			klog.V(4).InfoS("Removing pod logs", "podUID", podUID)
			err := osInterface.RemoveAll(filepath.Join(podLogsDirectory, name))
			if err != nil {
				klog.ErrorS(err, "Failed to remove pod logs directory", "path", name)
			}
		}
	}

	// Remove dead container log symlinks.
	// TODO(random-liu): Remove this after cluster logging supports CRI container log path.
	logSymlinks, _ := osInterface.Glob(filepath.Join(legacyContainerLogsDir, fmt.Sprintf("*.%s", legacyLogSuffix)))
	for _, logSymlink := range logSymlinks {
		if _, err := osInterface.Stat(logSymlink); os.IsNotExist(err) {
			if containerID, err := getContainerIDFromLegacyLogSymlink(logSymlink); err == nil {
				resp, err := cgc.manager.runtimeService.ContainerStatus(ctx, containerID, false)
				if err != nil {
					// TODO: we should handle container not found (i.e. container was deleted) case differently
					// once https://github.com/kubernetes/kubernetes/issues/63336 is resolved
					klog.InfoS("Error getting ContainerStatus for containerID", "containerID", containerID, "err", err)
				} else {
					status := resp.GetStatus()
					if status == nil {
						klog.V(4).InfoS("Container status is nil")
						continue
					}
					if status.State != runtimeapi.ContainerState_CONTAINER_EXITED {
						// Here is how container log rotation works (see containerLogManager#rotateLatestLog):
						//
						// 1. rename current log to rotated log file whose filename contains current timestamp (fmt.Sprintf("%s.%s", log, timestamp))
						// 2. reopen the container log
						// 3. if #2 fails, rename rotated log file back to container log
						//
						// There is small but indeterministic amount of time during which log file doesn't exist (between steps #1 and #2, between #1 and #3).
						// Hence the symlink may be deemed unhealthy during that period.
						// See https://github.com/kubernetes/kubernetes/issues/52172
						//
						// We only remove unhealthy symlink for dead containers
						klog.V(5).InfoS("Container is still running, not removing symlink", "containerID", containerID, "path", logSymlink)
						continue
					}
				}
			} else {
				klog.V(4).InfoS("Unable to obtain container ID", "err", err)
			}
			err := osInterface.Remove(logSymlink)
			if err != nil {
				klog.ErrorS(err, "Failed to remove container log dead symlink", "path", logSymlink)
			} else {
				klog.V(4).InfoS("Removed symlink", "path", logSymlink)
			}
		}
	}
	return nil
}
```

### 4. Image Gc处理流程

该函数逻辑

（1）获取镜像列表，清理旧镜像，更新镜像列表

（2）若计算文件系统的当前使用率使用率大于 HighThresholdPercent，此时需要回收镜像

（3）调用 im.freeSpace 回收未使用的镜像信息

```go

func (im *realImageGCManager) GarbageCollect(ctx context.Context, beganGC time.Time) error {
	ctx, otelSpan := im.tracer.Start(ctx, "Images/GarbageCollect")
	defer otelSpan.End()
	//获取镜像列表
	freeTime := time.Now()
	images, err := im.imagesInEvictionOrder(ctx, freeTime)
	if err != nil {
		return err
	}
	//清理旧镜像,更新镜像列表
	images, err = im.freeOldImages(ctx, images, freeTime, beganGC)
	if err != nil {
		return err
	}

	// Get disk usage on disk holding images.
	//获取磁盘使用情况
	fsStats, _, err := im.statsProvider.ImageFsStats(ctx)
	if err != nil {
		return err
	}
	//计算磁盘容量和可用空间
	var capacity, available int64
	if fsStats.CapacityBytes != nil {
		capacity = int64(*fsStats.CapacityBytes)
	}
	if fsStats.AvailableBytes != nil {
		available = int64(*fsStats.AvailableBytes)
	}
	//校验空间合理性
	if available > capacity {
		klog.InfoS("Availability is larger than capacity", "available", available, "capacity", capacity)
		available = capacity
	}

	// Check valid capacity.
	//检查有效容量
	if capacity == 0 {
		err := goerrors.New("invalid capacity 0 on image filesystem")
		im.recorder.Eventf(im.nodeRef, v1.EventTypeWarning, events.InvalidDiskCapacity, err.Error())
		return err
	}

	// If over the max threshold, free enough to place us at the lower threshold.
	//释放空间以满足阈值
	usagePercent := 100 - int(available*100/capacity)
	if usagePercent >= im.policy.HighThresholdPercent {
		amountToFree := capacity*int64(100-im.policy.LowThresholdPercent)/100 - available
		klog.InfoS("Disk usage on image filesystem is over the high threshold, trying to free bytes down to the low threshold", "usage", usagePercent, "highThreshold", im.policy.HighThresholdPercent, "amountToFree", amountToFree, "lowThreshold", im.policy.LowThresholdPercent)
		freed, err := im.freeSpace(ctx, amountToFree, freeTime, images)
		if err != nil {
			return err
		}

		if freed < amountToFree {
			err := fmt.Errorf("Failed to garbage collect required amount of images. Attempted to free %d bytes, but only found %d bytes eligible to free.", amountToFree, freed)
			im.recorder.Eventf(im.nodeRef, v1.EventTypeWarning, events.FreeDiskSpaceFailed, err.Error())
			return err
		}
	}

	return nil
}
```

#### 4.1 freeSpace

该函数的主要逻辑： 

（1）对传入的镜像列表进行筛选，以确定哪些镜像可能被删除

（2）检查镜像是否可删除：

- 对每个镜像，检查其 `lastUsed` 时间是否在调用 `freeSpace` 函数时的 `freeTime` 之前。如果镜像在 `freeTime` 后还被使用过，则跳过该镜像，不进行删除。
- 检查镜像的 `firstDetected` 时间，确保镜像的存在时间超过了策略设定的最小年龄 (`MinAge`)。如果未满足这个条件，同样跳过删除。

（3）尝试删除镜像：

- 如果镜像满足上述条件（未在 `freeTime` 后被使用且年龄足够老），尝试通过 `freeImage` 方法删除该镜像，并将释放的空间加到 `spaceFreed` 变量。
- 如果删除过程中出现错误，将这些错误添加到 `deletionErrors` 列表。

（4）判断是否已释放足够空间：

- 在每次成功删除镜像后，检查已释放的空间是否达到了要求的 `bytesToFree`。如果已经达到或超过这个值，停止删除过程。

```go
// Tries to free bytesToFree worth of images on the disk.
//
// Returns the number of bytes free and an error if any occurred. The number of
// bytes freed is always returned.
// Note that error may be nil and the number of bytes free may be less
// than bytesToFree.
func (im *realImageGCManager) freeSpace(ctx context.Context, bytesToFree int64, freeTime time.Time, images []evictionInfo) (int64, error) {
	// Delete unused images until we've freed up enough space.
	var deletionErrors []error
	spaceFreed := int64(0)
	for _, image := range images {
		klog.V(5).InfoS("Evaluating image ID for possible garbage collection based on disk usage", "imageID", image.id, "runtimeHandler", image.imageRecord.runtimeHandlerUsedToPullImage)
		// Images that are currently in used were given a newer lastUsed.
		if image.lastUsed.Equal(freeTime) || image.lastUsed.After(freeTime) {
			klog.V(5).InfoS("Image ID was used too recently, not eligible for garbage collection", "imageID", image.id, "lastUsed", image.lastUsed, "freeTime", freeTime)
			continue
		}

		// Avoid garbage collect the image if the image is not old enough.
		// In such a case, the image may have just been pulled down, and will be used by a container right away.
		if freeTime.Sub(image.firstDetected) < im.policy.MinAge {
			klog.V(5).InfoS("Image ID's age is less than the policy's minAge, not eligible for garbage collection", "imageID", image.id, "age", freeTime.Sub(image.firstDetected), "minAge", im.policy.MinAge)
			continue
		}

		if err := im.freeImage(ctx, image, ImageGarbageCollectedTotalReasonSpace); err != nil {
			deletionErrors = append(deletionErrors, err)
			continue
		}
		spaceFreed += image.size

		if spaceFreed >= bytesToFree {
			break
		}
	}

	if len(deletionErrors) > 0 {
		return spaceFreed, fmt.Errorf("wanted to free %d bytes, but freed %d bytes space with errors in image deletion: %v", bytesToFree, spaceFreed, errors.NewAggregate(deletionErrors))
	}
	return spaceFreed, nil
}
```

在 Kubernetes 中，Kubelet 的垃圾回收机制不仅仅依赖于定时任务，还与 Pod 的生命周期事件紧密相关。这些事件可以包括但不限于：

1. **Pod 删除**：
    - 当 Pod 被删除时，其关联的容器将变为停止状态。根据设置的垃圾回收策略，这些容器可能会被标记为可回收。此时，垃圾回收过程会考虑这些容器，如果它们满足 MinAge 条件，则会被清理。
    - Pod 的日志和其他资源（如挂载的卷的某些内容）也可能需要在 Pod 删除后进行清理。
2. **Pod 更新**：
    - 更新操作可能会导致旧的容器停止并启动新的容器版本。旧的容器实例如果不再需要，将被垃圾回收处理。
    - 这同样适用于 Pod 的配置变更，如环境变量、配置卷更新等。
3. **系统资源压力**：
    - 当系统资源（如磁盘空间或内存）压力达到一定阈值时，Kubelet 的垃圾回收机制可以被触发以释放资源。这通常涉及更积极的资源回收策略，比如减少保留的死容器数量或清理未使用的镜像。

### 定时与事件驱动的结合

Kubelet 的垃圾回收不只是定时执行的简单任务，而是一个灵活的系统，可以响应多种触发条件：

- **定时任务**确保即使在低活动期也能定期清理资源。
- **事件驱动的触发**，如 Pod 的生命周期变更，确保在必要时可以立即响应资源管理需求。

### 效率与资源管理

在设计垃圾回收策略时，Kubernetes 尝试平衡效率与资源利用率：

- 通过只在资源确实需要时才触发清理操作，减少系统的整体资源消耗。
- 保证即使在资源需求剧增时，系统也能保持稳定运行，例如在 Pod 高速创建和销毁的场景下。