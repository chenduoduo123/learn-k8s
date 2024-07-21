# 1.资源监测与分析

主要就是前面分析的 `synchronize` 函数用于周期性(10S)检查节点的资源状态，确定是否需要触发驱逐。

# 2.阈值设置和策略定义

## 2.1 驱逐变量定义

* signalToResource： 就是当发生驱逐事件的时候，告诉你是由那些资源引起的驱逐

* 在 Kubernetes 中，当你查看日志或事件以了解资源驱逐（==eviction==）的情况时，通常显示的是资源的信号名，例如 `evictionapi.SignalMemoryAvailable`，而不是资源的类型名 `v1.ResourceMemory`

```go
	// map signals to resources (and vice-versa)
	signalToResource = map[evictionapi.Signal]v1.ResourceName{}
	signalToResource[evictionapi.SignalMemoryAvailable] = v1.ResourceMemory 
//这表示内存可用量的信号，用于触发基于内存压力的驱逐决策。
	signalToResource[evictionapi.SignalAllocatableMemoryAvailable] = v1.ResourceMemory
//这个信号通常表示调整后的可分配内存量，可能用于特定的容器或服务的内存分配。
	signalToResource[evictionapi.SignalImageFsAvailable] = v1.ResourceEphemeralStorage
//这个信号关联于镜像文件系统的可用存储空间，用于基于磁盘空间的驱逐策略。
	signalToResource[evictionapi.SignalImageFsInodesFree] = resourceInodes
//这涉及到文件系统的 inode 数量，inode 是文件系统内部用于存储文件信息的一个数据结构。
	signalToResource[evictionapi.SignalContainerFsAvailable] = v1.ResourceEphemeralStorage
//这个信号用于监控容器文件系统的可用存储空间。
	signalToResource[evictionapi.SignalContainerFsInodesFree] = resourceInodes
//关注的是容器文件系统的可用 inode 数。
	signalToResource[evictionapi.SignalNodeFsAvailable] = v1.ResourceEphemeralStorage
//这个信号用于监控节点级别的文件系统可用存储空间。
	signalToResource[evictionapi.SignalNodeFsInodesFree] = resourceInodes
//监控节点文件系统的 inode 数量。
	signalToResource[evictionapi.SignalPIDAvailable] = resourcePids
//用于监控可用的进程标识符（PID）数目，这对于管理系统上运行的进程数量是非常关键的。
```

* signalToNodeCondition：Node节点状态，例如，如果内存资源低（`SignalMemoryAvailable`），则节点状态会被标记为 `NodeMemoryPressure`，表明节点正面临内存压力。



```go
	// map eviction signals to node conditions
	signalToNodeCondition = map[evictionapi.Signal]v1.NodeConditionType{}
	signalToNodeCondition[evictionapi.SignalMemoryAvailable] = v1.NodeMemoryPressure
	signalToNodeCondition[evictionapi.SignalAllocatableMemoryAvailable] = v1.NodeMemoryPressure
	signalToNodeCondition[evictionapi.SignalImageFsAvailable] = v1.NodeDiskPressure
	signalToNodeCondition[evictionapi.SignalContainerFsAvailable] = v1.NodeDiskPressure
	signalToNodeCondition[evictionapi.SignalNodeFsAvailable] = v1.NodeDiskPressure
	signalToNodeCondition[evictionapi.SignalImageFsInodesFree] = v1.NodeDiskPressure
	signalToNodeCondition[evictionapi.SignalNodeFsInodesFree] = v1.NodeDiskPressure
	signalToNodeCondition[evictionapi.SignalContainerFsInodesFree] = v1.NodeDiskPressure
	signalToNodeCondition[evictionapi.SignalPIDAvailable] = v1.NodePIDPressure
```

*  `SignalMemoryAvailable` 和 `SignalAllocatableMemoryAvailable`

- **节点状态**: `NodeMemoryPressure`
- 描述:
    - 这两个信号都关联到 `NodeMemoryPressure` 状态。
    - `SignalMemoryAvailable` 代表节点上实际可用的内存量低于某个阈值。
    - `SignalAllocatableMemoryAvailable` 指的是节点上配置为可分配给 Pods 的内存量低于某个阈值。
- 驱逐触发原因:
    - 如果节点的可用内存或可分配内存过低，可能会导致性能降低或者节点无法为新的或现有的 Pods 分配内存，从而触发驱逐过程以释放资源。

* `SignalImageFsAvailable`, `SignalContainerFsAvailable`, `SignalNodeFsAvailable`

- **节点状态**: `NodeDiskPressure`
- 描述:
    - 这些信号代表不同类型的文件系统资源可用性低于阈值。
    - `SignalImageFsAvailable`: 镜像文件系统的可用空间不足。
    - `SignalContainerFsAvailable`: 容器运行时文件系统的可用空间不足。
    - `SignalNodeFsAvailable`: 节点级文件系统的可用空间不足。
- 驱逐触发原因:
    - 当文件系统空间不足时，节点可能无法正常运行或创建新容器，因此系统可能会尝试驱逐 Pods 以回收空间。

* `SignalImageFsInodesFree`, `SignalNodeFsInodesFree`, `SignalContainerFsInodesFree`

- **节点状态**: `NodeDiskPressure`
- 描述:
    - 这些信号与 inode 的使用相关，inode 是文件系统中用于存储文件元数据的数据结构。
    - `SignalImageFsInodesFree`: 镜像文件系统可用 inode 数量低。
    - `SignalNodeFsInodesFree`: 节点级文件系统可用 inode 数量低。
    - `SignalContainerFsInodesFree`: 容器运行时文件系统可用 inode 数量低。
- 驱逐触发原因:
    - 如果 inode 数量不足，将无法在文件系统中创建新文件，这可能会影响节点上运行的应用程序的正常运行。系统可能会驱逐 Pods 以释放 inode 和相关资源。

* `SignalPIDAvailable`

- **节点状态**: `NodePIDPressure`
- 描述:
    - 此信号表示节点上可用的进程标识符（PID）数量低于某个阈值。
- 驱逐触发原因:
    - 进程标识符用完时，新的进程无法被创建。这可能导致新的 Pods 无法启动或现有 Pods 无法创建新进程，从而影响应用的正常运行。系统可能会驱逐 Pods 以减少对 PID 的需求，释放已有的 PID 资源。

---

## 2.2 解析配置 ParseThresholdConfig()

* `解析硬阈值`，`解析软阈值`，`解析宽限期`，`以及最小回收设置`

## 2.3 makeSignalObservations生成信号

```go
// makeSignalObservations derives observations using the specified summary provider.
func makeSignalObservations(summary *statsapi.Summary) (signalObservations, statsFunc) {
	// build the function to work against for pod stats
	//从缓存数据种获取Pod的统计数据
	statsFunc := cachedStatsFunc(summary.Pods)
	// build an evaluation context for current eviction signals
	result := signalObservations{}
	//检查节点的内存统计信息。如果可用内存和工作集内存都不为 nil，则记录内存可用量和总容量。这些信息被存储在 result 中，对应的信号是 evictionapi.SignalMemoryAvailable。
	//WorkingSetBytes就是当前程序正在使用的物理内存，当前活跃且不可被换出（非交换或者不可再分页）的内存集合。
	if memory := summary.Node.Memory; memory != nil && memory.AvailableBytes != nil && memory.WorkingSetBytes != nil {
		result[evictionapi.SignalMemoryAvailable] = signalObservation{
			available: resource.NewQuantity(int64(*memory.AvailableBytes), resource.BinarySI),
			capacity:  resource.NewQuantity(int64(*memory.AvailableBytes+*memory.WorkingSetBytes), resource.BinarySI),
			time:      memory.Time,
		}
	}
	//这里尝试获取系统容器（特别是 Pods 的容器）的内存使用情况，如果可用内存和工作集内存都不为 nil，并构建相应的观察结果。如果成功，同样记录可用内存和总容量。
	//并与 evictionapi.SignalAllocatableMemoryAvailable 信号关联
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
	//检查节点的文件系统统计信息是否非空，检查是否具备可用字节和容量字节的数据，如果都不为空，则记录可用字节和容量字节。
	//并于evictionapi.SignalNodeFsAvailable信号关联
	if nodeFs := summary.Node.Fs; nodeFs != nil {
		if nodeFs.AvailableBytes != nil && nodeFs.CapacityBytes != nil {
			result[evictionapi.SignalNodeFsAvailable] = signalObservation{
				available: resource.NewQuantity(int64(*nodeFs.AvailableBytes), resource.BinarySI),
				capacity:  resource.NewQuantity(int64(*nodeFs.CapacityBytes), resource.BinarySI),
				time:      nodeFs.Time,
			}
		}
		// 检查是否具备inode的可用数和总数的数据，如果都不为空，则记录inode的可用数和总数。
		//并于SignalNodeFsInodesFree信号关联
		if nodeFs.InodesFree != nil && nodeFs.Inodes != nil {
			result[evictionapi.SignalNodeFsInodesFree] = signalObservation{
				available: resource.NewQuantity(int64(*nodeFs.InodesFree), resource.DecimalSI),
				capacity:  resource.NewQuantity(int64(*nodeFs.Inodes), resource.DecimalSI),
				time:      nodeFs.Time,
			}
		}
	}
	//即节点运行时的统计数据是否存在
	if summary.Node.Runtime != nil {
		//检查节点的镜像文件系统统计数据是否非空
		if imageFs := summary.Node.Runtime.ImageFs; imageFs != nil {
			//检查镜像文件系统，检查是否有可用字节和容量字节数据：
			if imageFs.AvailableBytes != nil && imageFs.CapacityBytes != nil {
				//记录可用空间和容量，对应的信号是SignalImageFsAvailable
				result[evictionapi.SignalImageFsAvailable] = signalObservation{
					available: resource.NewQuantity(int64(*imageFs.AvailableBytes), resource.BinarySI),
					capacity:  resource.NewQuantity(int64(*imageFs.CapacityBytes), resource.BinarySI),
					time:      imageFs.Time,
				}
			}
			//检查是否有 inode 的可用数和总数数据，记录inode的可用数和总数的数据
			//并于SignalImageFsInodesFree信号关联
			if imageFs.InodesFree != nil && imageFs.Inodes != nil {
				result[evictionapi.SignalImageFsInodesFree] = signalObservation{
					available: resource.NewQuantity(int64(*imageFs.InodesFree), resource.DecimalSI),
					capacity:  resource.NewQuantity(int64(*imageFs.Inodes), resource.DecimalSI),
					time:      imageFs.Time,
				}
			}
		}
		//检查节点的容器文件系统统计数据是否非空
		if containerFs := summary.Node.Runtime.ContainerFs; containerFs != nil {
			//检查容器文件系统，检查是否有可用字节和容量字节数据：
			if containerFs.AvailableBytes != nil && containerFs.CapacityBytes != nil {
				//记录可用空间和容量，对应的信号是SignalContainerFsAvailable
				result[evictionapi.SignalContainerFsAvailable] = signalObservation{
					available: resource.NewQuantity(int64(*containerFs.AvailableBytes), resource.BinarySI),
					capacity:  resource.NewQuantity(int64(*containerFs.CapacityBytes), resource.BinarySI),
					time:      containerFs.Time,
				}
			}
			//检查容器文件系统，是否有 inode 的可用数和总数数据，记录inode的可用数和总数的数据
			//对应的信号是SignalContainerFsInodesFree
			if containerFs.InodesFree != nil && containerFs.Inodes != nil {
				result[evictionapi.SignalContainerFsInodesFree] = signalObservation{
					available: resource.NewQuantity(int64(*containerFs.InodesFree), resource.DecimalSI),
					capacity:  resource.NewQuantity(int64(*containerFs.Inodes), resource.DecimalSI),
					time:      containerFs.Time,
				}
			}
		}
	}
	//如果节点的进程限制数据可用，计算并记录剩余可用的进程ID数量和总量
	//与 evictionapi.SignalPIDAvailable 信号关联。
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

## 2.4 thresholdsMet计算驱逐阈值

```go
// thresholdsMet returns the set of thresholds that were met independent of grace period
func thresholdsMet(thresholds []evictionapi.Threshold, observations signalObservations, enforceMinReclaim bool) []evictionapi.Threshold {
	//results存储已达到阈值的阈值列表。
	results := []evictionapi.Threshold{}
	//检查阈值对应的观测数据是否存在
	for i := range thresholds {
		threshold := thresholds[i]
		observed, found := observations[threshold.Signal]
		if !found {
			klog.InfoS("Eviction manager: no observation found for eviction signal", "signal", threshold.Signal)
			continue
		}
		// determine if we have met the specified threshold
		thresholdMet := false
		//计算阈值，如果阈值是2G之类的直接返回，如果是百分比（比如50%），会根据capacity假如100G，100Gx50%=50G
		quantity := evictionapi.GetThresholdQuantity(threshold.Value, observed.capacity)
		// if enforceMinReclaim is specified, we compare relative to value - minreclaim
		//如果设置了 enforceMinReclaim =true代表开启最小回收，false代表不开启， 并且定义了最小回收量（MinReclaim），就会计算两次，依照上面计算的结果50G，如果MinReclaim=5G，
		//50G-5G=45G，实际触发的阈值是45G
		if enforceMinReclaim && threshold.MinReclaim != nil {
			quantity.Add(*evictionapi.GetThresholdQuantity(*threshold.MinReclaim, observed.capacity))
		}
		//计算出驱逐的阈值与实际可用资源量进行比较；
		//-1 如果 quantity 小于 observed.available
		//0 如果两者相等
		//1 如果 quantity 大于 observed.available
		//这里 >0,代表驱逐大于实际资源，所以要驱逐
		thresholdResult := quantity.Cmp(*observed.available)
		switch threshold.Operator {
		case evictionapi.OpLessThan:
			thresholdMet = thresholdResult > 0
		}
		if thresholdMet {
			results = append(results, threshold)
		}
	}
	return results
}
```

## 2.5 mergeThresholds 驱逐阈值合并

驱逐阈值和最小驱逐阈值合并去重，避免无效驱逐

```go
// mergeThresholds will merge both threshold lists eliminating duplicates.
func mergeThresholds(inputsA []evictionapi.Threshold, inputsB []evictionapi.Threshold) []evictionapi.Threshold {
	results := inputsA
	for _, threshold := range inputsB {
		if !hasThreshold(results, threshold) {
			results = append(results, threshold)
		}
	}
	return results
}
```

## 2.6 nodeConditions通过驱逐阈值设置Node节点状态

只有三种:

* NodeMemoryPressure
* NodeDiskPressure
* NodePIDPressure

```go
// nodeConditions returns the set of node conditions associated with a threshold
func nodeConditions(thresholds []evictionapi.Threshold) []v1.NodeConditionType {
	results := []v1.NodeConditionType{}
	for _, threshold := range thresholds {
		if nodeCondition, found := signalToNodeCondition[threshold.Signal]; found {
			if !hasNodeCondition(results, nodeCondition) {
				results = append(results, nodeCondition)
			}
		}
	}
	return results
}
```

## 2.7 buildSignalToNodeReclaimFuncs不同信号驱逐回收策略

* 镜像文件系统就是存储列表Docker镜像的文件系统
* 容器文件系统是容器中使用的文件系统，它模拟了一个完整的文件系统，为容器内运行的应用程序提供文件存储和管理。

```go
// buildSignalToNodeReclaimFuncs returns reclaim functions associated with resources.
func buildSignalToNodeReclaimFuncs(imageGC ImageGC, containerGC ContainerGC, withImageFs bool, splitContainerImageFs bool) map[evictionapi.Signal]nodeReclaimFuncs {
	signalToReclaimFunc := map[evictionapi.Signal]nodeReclaimFuncs{}
	// usage of an imagefs is optional
	// 当存在独立的镜像文件系统，且容器和镜像使用相同的文件系统时：
	if withImageFs && !splitContainerImageFs {
		// with an imagefs, nodefs pressure should just delete logs
		//SignalNodeFsAvailable和SignalNodeFsInodesFree不做回收
		signalToReclaimFunc[evictionapi.SignalNodeFsAvailable] = nodeReclaimFuncs{}
		signalToReclaimFunc[evictionapi.SignalNodeFsInodesFree] = nodeReclaimFuncs{}
		// with an imagefs, imagefs pressure should delete unused images
		//镜像文件系统的可用空间压力和inode压力会触发删除未使用的容器和镜像的操作。
		signalToReclaimFunc[evictionapi.SignalImageFsAvailable] = nodeReclaimFuncs{containerGC.DeleteAllUnusedContainers, imageGC.DeleteUnusedImages}
		signalToReclaimFunc[evictionapi.SignalImageFsInodesFree] = nodeReclaimFuncs{containerGC.DeleteAllUnusedContainers, imageGC.DeleteUnusedImages}
		// usage of imagefs and container fs on separate disks
		// containers gc on containerfs pressure
		// image gc on imagefs pressure、
		// 当存在独立的镜像文件系统，且容器和镜像使用分离的文件系统时：
	} else if withImageFs && splitContainerImageFs {
		// 镜像文件系统的压力仅触发删除未使用的镜像操作。
		// with an imagefs, imagefs pressure should delete unused images
		signalToReclaimFunc[evictionapi.SignalImageFsAvailable] = nodeReclaimFuncs{imageGC.DeleteUnusedImages}
		signalToReclaimFunc[evictionapi.SignalImageFsInodesFree] = nodeReclaimFuncs{imageGC.DeleteUnusedImages}
		// with an split fs and imagefs, containerfs pressure should delete unused containers
		// 容器文件系统的压力触发删除未使用的容器操作。
		signalToReclaimFunc[evictionapi.SignalNodeFsAvailable] = nodeReclaimFuncs{containerGC.DeleteAllUnusedContainers}
		signalToReclaimFunc[evictionapi.SignalNodeFsInodesFree] = nodeReclaimFuncs{containerGC.DeleteAllUnusedContainers}
	} else {
		// 当没有独立的镜像文件系统时（即容器、镜像和节点共用同一个文件系统）：
		// without an imagefs, nodefs pressure should delete logs, and unused images
		// since imagefs, containerfs and nodefs share a common device, they share common reclaim functions
		// 所有类型的文件系统压力都会触发删除未使用的容器和镜像的操作。
		signalToReclaimFunc[evictionapi.SignalNodeFsAvailable] = nodeReclaimFuncs{containerGC.DeleteAllUnusedContainers, imageGC.DeleteUnusedImages}
		signalToReclaimFunc[evictionapi.SignalNodeFsInodesFree] = nodeReclaimFuncs{containerGC.DeleteAllUnusedContainers, imageGC.DeleteUnusedImages}
		signalToReclaimFunc[evictionapi.SignalImageFsAvailable] = nodeReclaimFuncs{containerGC.DeleteAllUnusedContainers, imageGC.DeleteUnusedImages}
		signalToReclaimFunc[evictionapi.SignalImageFsInodesFree] = nodeReclaimFuncs{containerGC.DeleteAllUnusedContainers, imageGC.DeleteUnusedImages}
	}
	return signalToReclaimFunc
}
```

## 2.8 localStorageEviction本地存储驱逐

如果开启了本地存储容量隔离，检查是否需要驱逐

1. 检查Pod使用的单个EmptyDir超出限制没有，超出驱逐，并跳过后续的检查
2. 检查Pod使用的所有EmptyDir超出限制没有，超出驱逐，并跳过后续的检查
3. 检查 Pod 中每个容器的临时存储使用情况。如果任何容器的使用超过了限制，整个 Pod 会被添加到驱逐列表。

```go
// localStorageEviction checks the EmptyDir volume usage for each pod and determine whether it exceeds the specified limit and needs
// to be evicted. It also checks every container in the pod, if the container overlay usage exceeds the limit, the pod will be evicted too.
func (m *managerImpl) localStorageEviction(pods []*v1.Pod, statsFunc statsFunc) []*v1.Pod {
	evicted := []*v1.Pod{}
	for _, pod := range pods {
		podStats, ok := statsFunc(pod)
		if !ok {
			continue
		}

		if m.emptyDirLimitEviction(podStats, pod) {
			evicted = append(evicted, pod)
			continue
		}

		if m.podEphemeralStorageLimitEviction(podStats, pod) {
			evicted = append(evicted, pod)
			continue
		}

		if m.containerEphemeralStorageLimitEviction(podStats, pod) {
			evicted = append(evicted, pod)
		}
	}

	return evicted
}
```

## 2.9 reclaimNodeLevelResources驱逐前先回收Node节点资源

* 就是驱逐前先回收**Node**节点的不需要的资源比如**未使用的容器和镜像**，使用的函数就是之前`buildSignalToNodeReclaimFuncs`设置的清理函数;然后在计算 需要驱逐不

```go
// reclaimNodeLevelResources attempts to reclaim node level resources.  returns true if thresholds were satisfied and no pod eviction is required.
func (m *managerImpl) reclaimNodeLevelResources(ctx context.Context, signalToReclaim evictionapi.Signal, resourceToReclaim v1.ResourceName) bool {
	//通过信号获取对应的资源回收函数
	nodeReclaimFuncs := m.signalToNodeReclaimFuncs[signalToReclaim]
	//尝试执行每一个回收函数
	for _, nodeReclaimFunc := range nodeReclaimFuncs {
		// attempt to reclaim the pressured resource.
		if err := nodeReclaimFunc(ctx); err != nil {
			klog.InfoS("Eviction manager: unexpected error when attempting to reduce resource pressure", "resourceName", resourceToReclaim, "err", err)
		}

	}
	//如果函数 > 0 ,从cadvisor获取详细信息，就是node, pod的资源统计信息-（
	if len(nodeReclaimFuncs) > 0 {
		summary, err := m.summaryProvider.Get(ctx, true)
		if err != nil {
			klog.ErrorS(err, "Eviction manager: failed to get summary stats after resource reclaim")
			return false
		}

		// make observations and get a function to derive pod usage stats relative to those observations.
		//生成新的信号
		observations, _ := makeSignalObservations(summary)
		debugLogObservations("observations after resource reclaim", observations)

		// evaluate all thresholds independently of their grace period to see if with
		// the new observations, we think we have met min reclaim goals
		//重新计算驱逐阈值，忽略宽限期的影响
		thresholds := thresholdsMet(m.config.Thresholds, observations, true)
		debugLogThresholdsWithObservation("thresholds after resource reclaim - ignoring grace period", thresholds, observations)
		//如果驱逐阈值=0，表示不需要驱逐，否则还需要驱逐
		if len(thresholds) == 0 {
			return true
		}
	}
	return false
}
```

## 2.10 buildSignalToRankFunc驱逐Pod排序

* `主要功能：根据节点配置资源，为各种资源信号（例如内存、磁盘空间、inode 等）分配适当的排序函数。这些资源信号用于监控节点上的资源使用情况，并在资源紧张时决定哪些 Pod 应该被优先驱逐。`

```go
// buildSignalToRankFunc returns ranking functions associated with resources
func buildSignalToRankFunc(withImageFs bool, imageContainerSplitFs bool) map[evictionapi.Signal]rankFunc {
	//将 evictionapi.Signal 类型的信号映射到相应的 rankFunc 排序函数：
	signalToRankFunc := map[evictionapi.Signal]rankFunc{
		evictionapi.SignalMemoryAvailable:            rankMemoryPressure,
		evictionapi.SignalAllocatableMemoryAvailable: rankMemoryPressure,
		evictionapi.SignalPIDAvailable:               rankPIDPressure,
	}
	// usage of an imagefs is optional
	// We have a dedicated Image filesystem (images and containers are on same disk)
	// then we assume it is just a separate imagefs
	//Image 和容器在同一磁盘上
	if withImageFs && !imageContainerSplitFs {
		// with an imagefs, nodefs pod rank func for eviction only includes logs and local volumes
		//这个函数只考虑日志和本地卷的文件系统统计信息，用于计算节点文件系统的压力。所关注的资源类型是 v1.ResourceEphemeralStorage（临时存储）。
		signalToRankFunc[evictionapi.SignalNodeFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsLogs, fsStatsLocalVolumeSource}, v1.ResourceEphemeralStorage)
		//它也只考虑日志和本地卷的文件系统统计信息，但所关注的资源类型是 resourceInodes（inode 数量）。
		signalToRankFunc[evictionapi.SignalNodeFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsLogs, fsStatsLocalVolumeSource}, resourceInodes)
		// with an imagefs, imagefs pod rank func for eviction only includes rootfs
		//它考虑 rootfs 和镜像的文件系统统计信息，计算 image 文件系统的压力，关注的资源类型是 v1.ResourceEphemeralStorage。
		signalToRankFunc[evictionapi.SignalImageFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot, fsStatsImages}, v1.ResourceEphemeralStorage)
		//它也考虑 rootfs 和镜像的文件系统统计信息，但关注的资源类型是 resourceInodes。
		signalToRankFunc[evictionapi.SignalImageFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsRoot, fsStatsImages}, resourceInodes)
		//表示容器文件系统的可用空间排序函数与 image 文件系统的相同。
		signalToRankFunc[evictionapi.SignalContainerFsAvailable] = signalToRankFunc[evictionapi.SignalImageFsAvailable]
		//表示容器文件系统的可用 inode 数量排序函数与 image 文件系统的相同。
		signalToRankFunc[evictionapi.SignalContainerFsInodesFree] = signalToRankFunc[evictionapi.SignalImageFsInodesFree]

		// If both imagefs and container fs are on separate disks
		// we want to track the writeable layer in containerfs signals.
	} else if withImageFs && imageContainerSplitFs {
		//Image 和容器在不同磁盘上
		// with an imagefs, nodefs pod rank func for eviction only includes logs and local volumes
		//会考虑以下文件系统统计类型：日志（fsStatsLogs）、本地卷（fsStatsLocalVolumeSource）和 root 文件系统（fsStatsRoot），关注的资源类型是临时存储（v1.ResourceEphemeralStorage）
		signalToRankFunc[evictionapi.SignalNodeFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsLogs, fsStatsLocalVolumeSource, fsStatsRoot}, v1.ResourceEphemeralStorage)
		//也会考虑日志、本地卷和 root 文件系统的统计信息，但关注的资源类型是 inode 数量（resourceInodes）。
		signalToRankFunc[evictionapi.SignalNodeFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsLogs, fsStatsLocalVolumeSource, fsStatsRoot}, resourceInodes)
		//表示容器文件系统的可用空间排序函数与节点文件系统相同。
		signalToRankFunc[evictionapi.SignalContainerFsAvailable] = signalToRankFunc[evictionapi.SignalNodeFsAvailable]
		//表示容器文件系统的可用 inode 数量排序函数与节点文件系统相同。
		signalToRankFunc[evictionapi.SignalContainerFsInodesFree] = signalToRankFunc[evictionapi.SignalNodeFsInodesFree]
		// with an imagefs, containerfs pod rank func for eviction only includes rootfs
		//会仅考虑镜像文件系统的统计信息（fsStatsImages），关注的资源类型是临时存储（v1.ResourceEphemeralStorage）。
		signalToRankFunc[evictionapi.SignalImageFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsImages}, v1.ResourceEphemeralStorage)
		//也会仅考虑镜像文件系统的统计信息，但关注的资源类型是 inode 数量（resourceInodes）。
		signalToRankFunc[evictionapi.SignalImageFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsImages}, resourceInodes)
		// If image fs is not on separate disk as root but container fs is
	} else {
		//不使用独立的 Image 文件系统
		// without an imagefs, nodefs pod rank func for eviction looks at all fs stats.
		// since imagefs and nodefs share a common device, they share common ranking functions.
		//会考虑所有文件系统统计类型，包括镜像（fsStatsImages）、root 文件系统（fsStatsRoot）、日志（fsStatsLogs）和本地卷（fsStatsLocalVolumeSource），关注的资源类型是临时存储（v1.ResourceEphemeralStorage）。
		signalToRankFunc[evictionapi.SignalNodeFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsImages, fsStatsRoot, fsStatsLogs, fsStatsLocalVolumeSource}, v1.ResourceEphemeralStorage)
		//也会考虑所有文件系统统计类型，但关注的资源类型是 inode 数量（resourceInodes）。
		signalToRankFunc[evictionapi.SignalNodeFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsImages, fsStatsRoot, fsStatsLogs, fsStatsLocalVolumeSource}, resourceInodes)
		//因为 image 文件系统和节点文件系统共享一个设备，所以它们使用相同的排序函数。这个排序函数会考虑所有文件系统统计类型，并关注临时存储（v1.ResourceEphemeralStorage）
		signalToRankFunc[evictionapi.SignalImageFsAvailable] = rankDiskPressureFunc([]fsStatsType{fsStatsImages, fsStatsRoot, fsStatsLogs, fsStatsLocalVolumeSource}, v1.ResourceEphemeralStorage)
		//考虑所有文件系统统计类型，关注 inode 数量（resourceInodes）。
		signalToRankFunc[evictionapi.SignalImageFsInodesFree] = rankDiskPressureFunc([]fsStatsType{fsStatsImages, fsStatsRoot, fsStatsLogs, fsStatsLocalVolumeSource}, resourceInodes)
		//表示容器文件系统的可用空间排序函数与节点文件系统相同。
		signalToRankFunc[evictionapi.SignalContainerFsAvailable] = signalToRankFunc[evictionapi.SignalNodeFsAvailable]
		//表示容器文件系统的可用 inode 数量排序函数与节点文件系统相同。
		signalToRankFunc[evictionapi.SignalContainerFsInodesFree] = signalToRankFunc[evictionapi.SignalNodeFsInodesFree]
	}
	return signalToRankFunc
}
```

## 2.11 evictPod驱逐Pod

```
func (m *managerImpl) evictPod(pod *v1.Pod, gracePeriodOverride int64, evictMsg string, annotations map[string]string, condition *v1.PodCondition) bool {
   // If the pod is marked as critical and static, and support for critical pod annotations is enabled,
   // do not evict such pods. Static pods are not re-admitted after evictions.
   // https://github.com/kubernetes/kubernetes/issues/40573 has more details.
   //不能驱逐关键Pod或者静态Pod，静态 Pod 在被驱逐后不会被重新接纳。
   //使用 IsCriticalPod 函数检查传入的 Pod 是否是关键 Pod
   if kubelettypes.IsCriticalPod(pod) {
      klog.ErrorS(nil, "Eviction manager: cannot evict a critical pod", "pod", klog.KObj(pod))
      return false
      //关键返回false不能驱逐
   }
   // record that we are evicting the pod
   m.recorder.AnnotatedEventf(pod, annotations, v1.EventTypeWarning, Reason, evictMsg)
   // this is a blocking call and should only return when the pod and its containers are killed.
   klog.V(3).InfoS("Evicting pod", "pod", klog.KObj(pod), "podUID", pod.UID, "message", evictMsg)
   //调用 killPodFunc 来杀死 Pod
   err := m.killPodFunc(pod, true, &gracePeriodOverride, func(status *v1.PodStatus) {
      status.Phase = v1.PodFailed
      status.Reason = Reason
      status.Message = evictMsg
      if condition != nil {
         podutil.UpdatePodCondition(status, condition)
      }
   })
   if err != nil {
      klog.ErrorS(err, "Eviction manager: pod failed to evict", "pod", klog.KObj(pod))
   } else {
      klog.InfoS("Eviction manager: pod is evicted successfully", "pod", klog.KObj(pod))
   }
   return true
}
```
