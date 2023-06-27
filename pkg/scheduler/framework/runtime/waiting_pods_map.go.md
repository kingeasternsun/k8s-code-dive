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

首先我们思考一下，如果要我们自己实现一个这样的功能，需要有个机制通知framework某个waitingPod可以去Bind或失败，应该用什么方法。在golang中，首选的是利用channel，还可以用sync.Condition， 在k8s中就是使用channel来进行通知。

https://sourcegraph.com/github.com/kubernetes/kubernetes@v1.21.5/-/blob/pkg/scheduler/framework/runtime/waiting_pods_map.go?L73:6&popover=pinned

```go
// waitingPod represents a pod waiting in the permit phase.
type waitingPod struct {
	pod            *v1.Pod // 对应在Permit阶段返回framework.Wait的Pod
	pendingPlugins map[string]*time.Timer // 记录是哪些plugin判定这个Pod为framework.Wait, 以及不同plugin的Wait超时时间
	s              chan *framework.Status // framework.WaitOnPermit就是阻塞等待在这个channel上，根据s上接受的Status决定是执行绑定还是回退
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
由于在Permit阶段中，可能多个plugin都会判定这个Pod为Wait，所以newWaitingPod中的第二个参数pluginsMaxWaitTime是一个map，除了记录判定Pod为Wait的plugin列表，同时还会记录每个plugin对应的超时时间。然后在创建newWaitingPod时，基于这些信息为每个plugin创建一个Timer (`time.AfterFunc`)，在超时后调用`Reject`方法来把这个Pod设置为失败从而通知到framework。


下面这两个方法比较简单
https://sourcegraph.com/github.com/kubernetes/kubernetes@v1.21.5/-/blob/pkg/scheduler/framework/runtime/waiting_pods_map.go?L111:22&popover=pinned
```go
// GetPod returns a reference to the waiting pod.
func (w *waitingPod) GetPod() *v1.Pod {
	return w.pod
}
```

https://sourcegraph.com/github.com/kubernetes/kubernetes@v1.21.5/-/blob/pkg/scheduler/framework/runtime/waiting_pods_map.go?L116:22&popover=pinned

```go
// GetPendingPlugins returns a list of pending permit plugin's name.
func (w *waitingPod) GetPendingPlugins() []string {
	w.mu.RLock()
	defer w.mu.RUnlock()
	plugins := make([]string, 0, len(w.pendingPlugins))
	for p := range w.pendingPlugins {
		plugins = append(plugins, p)
	}

	return plugins
}
```


https://sourcegraph.com/github.com/kubernetes/kubernetes@v1.21.5/-/blob/pkg/scheduler/framework/runtime/waiting_pods_map.go?L130:22&popover=pinned
```go
// Allow declares the waiting pod is allowed to be scheduled by plugin pluginName.
// If this is the last remaining plugin to allow, then a success signal is delivered
// to unblock the pod.
func (w *waitingPod) Allow(pluginName string) {
	w.mu.Lock()
	defer w.mu.Unlock()
	if timer, exist := w.pendingPlugins[pluginName]; exist {
		timer.Stop()
		delete(w.pendingPlugins, pluginName)
	}

	// Only signal success status after all plugins have allowed
	if len(w.pendingPlugins) != 0 {
		return
	}

	// The select clause works as a non-blocking send.
	// If there is no receiver, it's a no-op (default case).
	select {
	case w.s <- framework.NewStatus(framework.Success, ""):
	default:
	}
}
```
由于Pod可能是被多个plugin判断为Wait的，所以当一个plugin判断这个Pod为Allow后，就将这个plugin从pendingPlugins移除，只有所有的plugin都Allow（也就是pendingPlugins为空）后才能判断这个Pod为Success，然后用 non-blocking 的方式发送结果到 w.s

https://sourcegraph.com/github.com/kubernetes/kubernetes@v1.21.5/-/blob/pkg/scheduler/framework/runtime/waiting_pods_map.go?L152:22&popover=pinned
```go
// Reject declares the waiting pod unschedulable.
func (w *waitingPod) Reject(pluginName, msg string) {
	w.mu.RLock()
	defer w.mu.RUnlock()
	for _, timer := range w.pendingPlugins {
		timer.Stop()
	}

	// The select clause works as a non-blocking send.
	// If there is no receiver, it's a no-op (default case).
	select {
	case w.s <- framework.NewStatus(framework.Unschedulable, msg).WithFailedPlugin(pluginName):
	default:
	}
}
```
任何一个plugin判断这个Pod失败就意味这个Pod无法完成调度，就可以用 non-blocking 的方式发送结果到 w.s， framework收到后就不会对Pod绑定，执行相关回退操作