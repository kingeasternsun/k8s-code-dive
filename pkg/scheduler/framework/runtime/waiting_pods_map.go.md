https://sourcegraph.com/github.com/kubernetes/kubernetes@v1.21.5/-/blob/pkg/scheduler/framework/runtime/waiting_pods_map.go


# waitingPodsMap
https://sourcegraph.com/github.com/kubernetes/kubernetes@v1.21.5/-/blob/pkg/scheduler/framework/runtime/waiting_pods_map.go?L30:6&popover=pinned
```go
// waitingPodsMap a thread-safe map used to maintain pods waiting in the permit phase.
type waitingPodsMap struct {
	pods map[types.UID]*waitingPod
	mu   sync.RWMutex
}
```
`waitingPodsMap`是一个线程并发安全的map，用来存储在k8s framework调度器`permit`阶段判断Wait后处于waiting的Pod,
这个类型的结构比较简单，通过一个读写锁 `mu   sync.RWMutex` 加一个基础的map `pods map[types.UID]*waitingPod`实现，其中map的Key的类型是types.UID, 值类型是 *waitingPod

# waitingPod
https://sourcegraph.com/github.com/kubernetes/kubernetes@v1.21.5/-/blob/pkg/scheduler/framework/runtime/waiting_pods_map.go?L73:6&popover=pinned

```go
// waitingPod represents a pod waiting in the permit phase.
type waitingPod struct {
	pod            *v1.Pod
	pendingPlugins map[string]*time.Timer
	s              chan *framework.Status
	mu             sync.RWMutex
}
```

先看初始化方法 https://sourcegraph.com/github.com/kubernetes/kubernetes@v1.21.5/-/blob/pkg/scheduler/framework/runtime/waiting_pods_map.go?L83:6&popover=pinned
```go

// newWaitingPod returns a new waitingPod instance.
func newWaitingPod(pod *v1.Pod, pluginsMaxWaitTime map[string]time.Duration) *waitingPod {
	wp := &waitingPod{
		pod: pod,
		// Allow() and Reject() calls are non-blocking. This property is guaranteed
		// by using non-blocking send to this channel. This channel has a buffer of size 1
		// to ensure that non-blocking send will not be ignored - possible situation when
		// receiving from this channel happens after non-blocking send.
		s: make(chan *framework.Status, 1),
	}

	wp.pendingPlugins = make(map[string]*time.Timer, len(pluginsMaxWaitTime))
	// The time.AfterFunc calls wp.Reject which iterates through pendingPlugins map. Acquire the
	// lock here so that time.AfterFunc can only execute after newWaitingPod finishes.
	wp.mu.Lock()
	defer wp.mu.Unlock()
	for k, v := range pluginsMaxWaitTime {
		plugin, waitTime := k, v
		wp.pendingPlugins[plugin] = time.AfterFunc(waitTime, func() {
			msg := fmt.Sprintf("rejected due to timeout after waiting %v at plugin %v",
				waitTime, plugin)
			wp.Reject(plugin, msg)
		})
	}

	return wp
}
```