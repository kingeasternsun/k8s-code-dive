

[NewFramework](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L245:6&popover=pinned)
```go
// NewFramework initializes plugins given the configuration and the registry.
func NewFramework(r Registry, profile *config.KubeSchedulerProfile, opts ...Option) (framework.Framework, error) {
	options := defaultFrameworkOptions()
	for _, opt := range opts {
		opt(&options)
	}

	f := &frameworkImpl{
		registry:              r,
		snapshotSharedLister:  options.snapshotSharedLister,
		pluginNameToWeightMap: make(map[string]int),
		waitingPods:           newWaitingPodsMap(),
		clientSet:             options.clientSet,
		eventRecorder:         options.eventRecorder,
		informerFactory:       options.informerFactory,
		metricsRecorder:       options.metricsRecorder,
		runAllFilters:         options.runAllFilters,
		extenders:             options.extenders,
		PodNominator:          options.podNominator,
		parallelizer: options.parallelizer,
	}

	if profile == nil {
		return f, nil
	}

	f.profileName = profile.SchedulerName
	if profile.Plugins == nil {
		return f, nil
	}

	// get needed plugins from config
	// 从配置文件读取的配置信息中，获取Enabled的plugin列表
	pg := f.pluginsNeeded(profile.Plugins)

	// 获取不同plugin所使用的的参数
	pluginConfig := make(map[string]runtime.Object, len(profile.PluginConfig))
	for i := range profile.PluginConfig {
		name := profile.PluginConfig[i].Name
		if _, ok := pluginConfig[name]; ok {
			return nil, fmt.Errorf("repeated config for plugin %s", name)
		}
		pluginConfig[name] = profile.PluginConfig[i].Args
	}
	outputProfile := config.KubeSchedulerProfile{
		SchedulerName: f.profileName,
		Plugins:       profile.Plugins,
		PluginConfig:  make([]config.PluginConfig, 0, len(pg)),
	}

	pluginsMap := make(map[string]framework.Plugin)
	var totalPriority int64
	for name, factory := range r {
		// initialize only needed plugins.
		// r 里面保存的不同plugin的实例化函数,key就是plugin名字，factory就是创建plugin实例的NewXXX函数
		if _, ok := pg[name]; !ok {
			continue
		}

		args, err := getPluginArgsOrDefault(pluginConfig, name)
		if err != nil {
			return nil, fmt.Errorf("getting args for Plugin %q: %w", name, err)
		}
		if args != nil {
			outputProfile.PluginConfig = append(outputProfile.PluginConfig, config.PluginConfig{
				Name: name,
				Args: args,
			})
		}

		// 完成plugin实例的创建
		p, err := factory(args, f)
		if err != nil {
			return nil, fmt.Errorf("initializing plugin %q: %w", name, err)
		}
		pluginsMap[name] = p

		// Update ClusterEventMap in place.
		// 注册不同plugin所关联的一些Event事件
		fillEventToPluginMap(p, options.clusterEventMap)

		// a weight of zero is not permitted, plugins can be disabled explicitly
		// when configured.
		f.pluginNameToWeightMap[name] = int(pg[name].Weight)
		if f.pluginNameToWeightMap[name] == 0 {
			f.pluginNameToWeightMap[name] = 1
		}
		// Checks totalPriority against MaxTotalScore to avoid overflow
		if int64(f.pluginNameToWeightMap[name])*framework.MaxNodeScore > framework.MaxTotalScore-totalPriority {
			return nil, fmt.Errorf("total score of Score plugins could overflow")
		}
		totalPriority += int64(f.pluginNameToWeightMap[name]) * framework.MaxClusterScore
	}

	// 将上面创建完成的plugin实例填充到e.slicePtr也就是frameworkImpl.preFilterPlugins , frameworkImpl.XXXPlugins中
	for _, e := range f.getExtensionPoints(profile.Plugins) {
		if err := updatePluginList(e.slicePtr, e.plugins, pluginsMap); err != nil {
			return nil, err
		}
	}

	// Verifying the score weights again since Plugin.Name() could return a different
	// value from the one used in the configuration.
	for _, scorePlugin := range f.scorePlugins {
		if f.pluginNameToWeightMap[scorePlugin.Name()] == 0 {
			return nil, fmt.Errorf("score plugin %q is not configured with weight", scorePlugin.Name())
		}
	}

	if len(f.queueSortPlugins) == 0 {
		return nil, fmt.Errorf("no queue sort plugin is enabled")
	}
	if len(f.queueSortPlugins) > 1 {
		return nil, fmt.Errorf("only one queue sort plugin can be enabled")
	}
	if len(f.bindPlugins) == 0 {
		return nil, fmt.Errorf("at least one bind plugin is needed")
	}

	// Configurator 提供
	// https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/factory.go?L144:20&popover=pinned
	// 最终对应到 https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/cmd/kube-scheduler/app/server.go?L334:13&popover=pinned
	if options.captureProfile != nil {
		if len(outputProfile.PluginConfig) != 0 {
			sort.Slice(outputProfile.PluginConfig, func(i, j int) bool {
				return outputProfile.PluginConfig[i].Name < outputProfile.PluginConfig[j].Name
			})
		} else {
			outputProfile.PluginConfig = nil
		}
		options.captureProfile(outputProfile)
	}

	return f, nil
}

```

[fillEventToPluginMap](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L370:6&popover=pinned)
```go
func fillEventToPluginMap(p framework.Plugin, eventToPlugins map[framework.ClusterEvent]sets.String) {
	ext, ok := p.(framework.EnqueueExtensions)
	if !ok {
		// If interface EnqueueExtensions is not implemented, register the default events
		// to the plugin. This is to ensure backward compatibility.
		registerClusterEvents(p.Name(), eventToPlugins, allClusterEvents)
		return
	}

	// ext.EventsToRegister 返回这个plugin所关注的一些集群Event，这些Event发生的时候可能会导致一些之前无法调度的pod变为可调度
	events := ext.EventsToRegister()
	// It's rare that a plugin implements EnqueueExtensions but returns nil.
	// We treat it as: the plugin is not interested in any event, and hence pod failed by that plugin
	// cannot be moved by any regular cluster event.
	if len(events) == 0 {
		klog.InfoS("Plugin's EventsToRegister() returned nil", "plugin", p.Name())
		return
	}
	// The most common case: a plugin implements EnqueueExtensions and returns non-nil result.
	registerClusterEvents(p.Name(), eventToPlugins, events)
}

```

[updatePluginList](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L420:6&popover=pinned)
```go
// 利用 reflect 机制，从而将创建后的不同类型的plugin实例分别加入到 frameworkImpl.queueSortPlugins frameworkImpl.preFilterPlugins 等列表中
func updatePluginList(pluginList interface{}, pluginSet config.PluginSet, pluginsMap map[string]framework.Plugin) error {
	plugins := reflect.ValueOf(pluginList).Elem()
	pluginType := plugins.Type().Elem()
	set := sets.NewString()
	for _, ep := range pluginSet.Enabled {
		pg, ok := pluginsMap[ep.Name]
		if !ok {
			return fmt.Errorf("%s %q does not exist", pluginType.Name(), ep.Name)
		}

		if !reflect.TypeOf(pg).Implements(pluginType) {
			return fmt.Errorf("plugin %q does not extend %s plugin", ep.Name, pluginType.Name())
		}

		if set.Has(ep.Name) {
			return fmt.Errorf("plugin %q already registered as %q", ep.Name, pluginType.Name())
		}

		set.Insert(ep.Name)

		newPlugins := reflect.Append(plugins, reflect.ValueOf(pg))
		plugins.Set(newPlugins)
	}
	return nil
}
```

[QueueSortFunc](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L447:25&popover=pinned)
```go
// QueueSortFunc returns the function to sort pods in scheduling queue
// 用于在调度队列中对待调度的pod进行排序
func (f *frameworkImpl) QueueSortFunc() framework.LessFunc {
	if f == nil {
		// If frameworkImpl is nil, simply keep their order unchanged.
		// NOTE: this is primarily for tests.
		return func(_, _ *framework.QueuedPodInfo) bool { return false }
	}

	if len(f.queueSortPlugins) == 0 {
		panic("No QueueSort plugin is registered in the frameworkImpl.")
	}

	// Only one QueueSort plugin can be enabled.
	// 可以看到如果有多个排序plugin，也只有第一个排序plugin生效，
	return f.queueSortPlugins[0].Less
}
```
[RunPreFilterPlugins](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L466:25&popover=pinned)
```go
// RunPreFilterPlugins runs the set of configured PreFilter plugins. It returns
// *Status and its code is set to non-success if any of the plugins returns
// anything but Success. If a non-success status is returned, then the scheduling
// cycle is aborted.
// 只要任意一个preFilterPlugin返回不成功，就认定失败，这个调度周期就结束
func (f *frameworkImpl) RunPreFilterPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod) (status *framework.Status) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(preFilter, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()
	for _, pl := range f.preFilterPlugins {
		status = f.runPreFilterPlugin(ctx, pl, state, pod)
		if !status.IsSuccess() {
			status.SetFailedPlugin(pl.Name())
			if status.IsUnschedulable() {
				return status
			}
			return framework.AsStatus(fmt.Errorf("running PreFilter plugin %q: %w", pl.Name(), status.AsError())).WithFailedPlugin(pl.Name())
		}
	}

	return nil
}
```

[RunFilterPlugins](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L569:25&popover=pinned)
```go
// RunFilterPlugins runs the set of configured Filter plugins for pod on
// the given node. If any of these plugins doesn't return "Success", the
// given node is not suitable for running pod.
// Meanwhile, the failure message and status are set for the given node.
// 只要有任意一个FilterPlugin没有返回Success，并且错误不是UnSchedulable，就判断失败，调度周期结束
// 如果返回Unschedulable错误，且f.runAllFilters为true的话，还会再去试后面的FilterPlugin
// 如果返回Unschedulable错误，且f.runAllFilters为false的话，就判断失败，调度周期结束
func (f *frameworkImpl) RunFilterPlugins(
	ctx context.Context,
	state *framework.CycleState,
	pod *v1.Pod,
	nodeInfo *framework.NodeInfo,
) framework.PluginToStatus {
	statuses := make(framework.PluginToStatus)
	for _, pl := range f.filterPlugins {
		pluginStatus := f.runFilterPlugin(ctx, pl, state, pod, nodeInfo)
		if !pluginStatus.IsSuccess() {
			if !pluginStatus.IsUnschedulable() {
				// Filter plugins are not supposed to return any status other than
				// Success or Unschedulable.
				errStatus := framework.AsStatus(fmt.Errorf("running %q filter plugin: %w", pl.Name(), pluginStatus.AsError())).WithFailedPlugin(pl.Name())
				return map[string]*framework.Status{pl.Name(): errStatus}
			}
			pluginStatus.SetFailedPlugin(pl.Name())
			statuses[pl.Name()] = pluginStatus
			if !f.runAllFilters {
				// Exit early if we don't need to run all filters.
				return statuses
			}
		}
	}

	return statuses
}
```

[RunPostFilterPlugins](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L609:25&popover=pinned)
```go
// RunPostFilterPlugins runs the set of configured PostFilter plugins until the first
// Success or Error is met, otherwise continues to execute all plugins.
// 这里和别的都不一样，如果一个PostFilterPlugin返回 Unschedulable的错误，就会再执行下一个PostFilterPlugin，直到遇到Success或非Unschedulable的错误就返回
func (f *frameworkImpl) RunPostFilterPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, filteredNodeStatusMap framework.NodeToStatusMap) (_ *framework.PostFilterResult, status *framework.Status) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(postFilter, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()

	statuses := make(framework.PluginToStatus)
	for _, pl := range f.postFilterPlugins {
		r, s := f.runPostFilterPlugin(ctx, pl, state, pod, filteredNodeStatusMap)
		if s.IsSuccess() {
			return r, s
		} else if !s.IsUnschedulable() {
			// Any status other than Success or Unschedulable is Error.
			return nil, framework.AsStatus(s.AsError())
		}
		statuses[pl.Name()] = s
	}

	return nil, statuses.Merge()
}
```

[RunPreScorePlugins](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L725:25&popover=pinned)
```go
// RunPreScorePlugins runs the set of configured pre-score plugins. If any
// of these plugins returns any status other than "Success", the given pod is rejected.
// 任何一个PreScorePlugin没有返回Success，这个Pod就认定失败，调度周期结束
func (f *frameworkImpl) RunPreScorePlugins(
	ctx context.Context,
	state *framework.CycleState,
	pod *v1.Pod,
	nodes []*v1.Node,
) (status *framework.Status) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(preScore, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()
	for _, pl := range f.preScorePlugins {
		status = f.runPreScorePlugin(ctx, pl, state, pod, nodes)
		if !status.IsSuccess() {
			return framework.AsStatus(fmt.Errorf("running PreScore plugin %q: %w", pl.Name(), status.AsError()))
		}
	}

	return nil
}
```

[RunScorePlugins](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L759:25&popover=pinned)
```go
// RunScorePlugins runs the set of configured scoring plugins. It returns a list that
// stores for each scoring plugin name the corresponding NodeScoreList(s).
// It also returns *Status, which is set to non-success if any of the plugins returns
// a non-success status.
func (f *frameworkImpl) RunScorePlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodes []*v1.Node) (ps framework.PluginToNodeScores, status *framework.Status) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(score, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()
	pluginToNodeScores := make(framework.PluginToNodeScores, len(f.scorePlugins))
	for _, pl := range f.scorePlugins {
		pluginToNodeScores[pl.Name()] = make(framework.NodeScoreList, len(nodes))
	}
	ctx, cancel := context.WithCancel(ctx)
	errCh := parallelize.NewErrorChannel()

	// Run Score method for each node in parallel.
	// 这里既可以按照plugin并行计算，也可以按照Node并行计算，由于实际场景中Node数量是远远大于plugin数量，所以这里以Node并行计算
	f.Parallelizer().Until(ctx, len(nodes), func(index int) {
		for _, pl := range f.scorePlugins {
			nodeName := nodes[index].Name
			s, status := f.runScorePlugin(ctx, pl, state, pod, nodeName)
			if !status.IsSuccess() {
				err := fmt.Errorf("plugin %q failed with: %w", pl.Name(), status.AsError())
				errCh.SendErrorWithCancel(err, cancel)
				return
			}
			pluginToNodeScores[pl.Name()][index] = framework.NodeScore{
				Name:  nodeName,
				Score: s,
			}
		}
	})
	if err := errCh.ReceiveError(); err != nil {
		return nil, framework.AsStatus(fmt.Errorf("running Score plugins: %w", err))
	}

	// Run NormalizeScore method for each ScorePlugin in parallel.
	// 这里执行归一化操作，由于归一化只能是同一plugin下的不同Node执行，所以这里只能以plugin进行并行计算
	f.Parallelizer().Until(ctx, len(f.scorePlugins), func(index int) {
		pl := f.scorePlugins[index]
		nodeScoreList := pluginToNodeScores[pl.Name()]
		if pl.ScoreExtensions() == nil {
			return
		}
		status := f.runScoreExtension(ctx, pl, state, pod, nodeScoreList)
		if !status.IsSuccess() {
			err := fmt.Errorf("plugin %q failed with: %w", pl.Name(), status.AsError())
			errCh.SendErrorWithCancel(err, cancel)
			return
		}
	})
	if err := errCh.ReceiveError(); err != nil {
		return nil, framework.AsStatus(fmt.Errorf("running Normalize on Score plugins: %w", err))
	}

	// Apply score defaultWeights for each ScorePlugin in parallel.
	// 对归一化的得分加上plugin的权重, 这里以plugin进行并行计算，其实可以基于Node进行并行计算,最新的k8s中已经改为基于Node的并行了
	f.Parallelizer().Until(ctx, len(f.scorePlugins), func(index int) {
		pl := f.scorePlugins[index]
		// Score plugins' weight has been checked when they are initialized.
		weight := f.pluginNameToWeightMap[pl.Name()]
		nodeScoreList := pluginToNodeScores[pl.Name()]

		for i, nodeScore := range nodeScoreList {
			// return error if score plugin returns invalid score.
			if nodeScore.Score > framework.MaxNodeScore || nodeScore.Score < framework.MinNodeScore {
				err := fmt.Errorf("plugin %q returns an invalid score %v, it should in the range of [%v, %v] after normalizing", pl.Name(), nodeScore.Score, framework.MinNodeScore, framework.MaxNodeScore)
				errCh.SendErrorWithCancel(err, cancel)
				return
			}
			nodeScoreList[i].Score = nodeScore.Score * int64(weight)
		}
	})
	if err := errCh.ReceiveError(); err != nil {
		return nil, framework.AsStatus(fmt.Errorf("applying score defaultWeights on Score plugins: %w", err))
	}

	return pluginToNodeScores, nil
}

```


[RunPreBindPlugins](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L856:25&popover=pinned)

```go
// RunPreBindPlugins runs the set of configured prebind plugins. It returns a
// failure (bool) if any of the plugins returns an error. It also returns an
// error containing the rejection message or the error occurred in the plugin.
// 只有有一个PreBindPlugin返回不是Success，就认定Pod失败，调度周期结束
func (f *frameworkImpl) RunPreBindPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (status *framework.Status) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(preBind, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()
	for _, pl := range f.preBindPlugins {
		status = f.runPreBindPlugin(ctx, pl, state, pod, nodeName)
		if !status.IsSuccess() {
			err := status.AsError()
			klog.ErrorS(err, "Failed running PreBind plugin", "plugin", pl.Name(), "pod", klog.KObj(pod))
			return framework.AsStatus(fmt.Errorf("running PreBind plugin %q: %w", pl.Name(), err))
		}
	}
	return nil
}
```


[RunBindPlugins](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L883:25&popover=pinned)

```go
// RunBindPlugins runs the set of configured bind plugins until one returns a non `Skip` status.
// 如果BindPlugin返回Skip则继续执行下一个BindPlugin，直到返回Sucees或非Skip的错误
func (f *frameworkImpl) RunBindPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (status *framework.Status) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(bind, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()
	if len(f.bindPlugins) == 0 {
		return framework.NewStatus(framework.Skip, "")
	}
	for _, bp := range f.bindPlugins {
		status = f.runBindPlugin(ctx, bp, state, pod, nodeName)
		if status != nil && status.Code() == framework.Skip {
			continue
		}
		if !status.IsSuccess() {
			err := status.AsError()
			klog.ErrorS(err, "Failed running Bind plugin", "plugin", bp.Name(), "pod", klog.KObj(pod))
			return framework.AsStatus(fmt.Errorf("running Bind plugin %q: %w", bp.Name(), err))
		}
		return status
	}
	return status
}
```
[RunPostBindPlugins](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L917:25&popover=pinned)
```go
// RunPostBindPlugins runs the set of configured postbind plugins.
// 执行所有的PostBindPlugin，不关心执行结果
func (f *frameworkImpl) RunPostBindPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(postBind, framework.Success.String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()
	for _, pl := range f.postBindPlugins {
		f.runPostBindPlugin(ctx, pl, state, pod, nodeName)
	}
}

```

[RunReservePluginsReserve](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L942:25&popover=pinned)
```go
// RunReservePluginsReserve runs the Reserve method in the set of configured
// reserve plugins. If any of these plugins returns an error, it does not
// continue running the remaining ones and returns the error. In such a case,
// the pod will not be scheduled and the caller will be expected to call
// RunReservePluginsUnreserve.
// 只要遇到任意一个ReservePlugin返回非Succes，就返回失败，pod不会被调度，上层调用者需要执行 RunReservePluginsUnreserve 进行回滚
func (f *frameworkImpl) RunReservePluginsReserve(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (status *framework.Status) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(reserve, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()
	for _, pl := range f.reservePlugins {
		status = f.runReservePluginReserve(ctx, pl, state, pod, nodeName)
		if !status.IsSuccess() {
			err := status.AsError()
			klog.ErrorS(err, "Failed running Reserve plugin", "plugin", pl.Name(), "pod", klog.KObj(pod))
			return framework.AsStatus(fmt.Errorf("running Reserve plugin %q: %w", pl.Name(), err))
		}
	}
	return nil
}
```

[RunReservePluginsUnreserve](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L970:25&popover=pinned)
```go
// RunReservePluginsUnreserve runs the Unreserve method in the set of
// configured reserve plugins.
// 在 RunReservePluginsReserve 失败后，执行RunReservePluginsUnreserve进行回滚操作，plugin执行顺序跟RunReservePluginsReserve中的执行顺序
// 相反，不会判断返回值
func (f *frameworkImpl) RunReservePluginsUnreserve(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(unreserve, framework.Success.String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()
	// Execute the Unreserve operation of each reserve plugin in the
	// *reverse* order in which the Reserve operation was executed.
	for i := len(f.reservePlugins) - 1; i >= 0; i-- {
		f.runReservePluginUnreserve(ctx, f.reservePlugins[i], state, pod, nodeName)
	}
}
```

[RunPermitPlugins](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L998:25&popover=pinned)
```go
// RunPermitPlugins runs the set of configured permit plugins. If any of these
// plugins returns a status other than "Success" or "Wait", it does not continue
// running the remaining plugins and returns an error. Otherwise, if any of the
// plugins returns "Wait", then this function will create and add waiting pod
// to a map of currently waiting pods and return status with "Wait" code.
// Pod will remain waiting pod for the minimum duration returned by the permit plugins.
// 如果一个 PermitPlugin 返回Success，则执行下一个PermitPlugin
// 如果-个 PermitPlugin 返回Wait，记录当前plugin名字和要等待的超时时间，然后执行下一个PermitPlugin
// 如果一个 PermitPlugin 返回既不是Success，也不是Wait，则直接退出返回失败
// 最终如果经过所有PermitPlugin决策后，这个Pod需要Wait，则将根据返回Wait的plugin列表，为这个Pod创建waitingPod 加入到waitingPods中
func (f *frameworkImpl) RunPermitPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (status *framework.Status) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(permit, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()
	pluginsWaitTime := make(map[string]time.Duration)
	statusCode := framework.Success
	for _, pl := range f.permitPlugins {
		status, timeout := f.runPermitPlugin(ctx, pl, state, pod, nodeName)
		if !status.IsSuccess() {
			if status.IsUnschedulable() {
				msg := fmt.Sprintf("rejected pod %q by permit plugin %q: %v", pod.Name, pl.Name(), status.Message())
				klog.V(4).Infof(msg)
				status.SetFailedPlugin(pl.Name())
				return status
			}
			if status.Code() == framework.Wait {
				// Not allowed to be greater than maxTimeout.
				if timeout > maxTimeout {
					timeout = maxTimeout
				}
				pluginsWaitTime[pl.Name()] = timeout
				statusCode = framework.Wait
			} else {
				err := status.AsError()
				klog.ErrorS(err, "Failed running Permit plugin", "plugin", pl.Name(), "pod", klog.KObj(pod))
				return framework.AsStatus(fmt.Errorf("running Permit plugin %q: %w", pl.Name(), err)).WithFailedPlugin(pl.Name())
			}
		}
	}
	if statusCode == framework.Wait {
		waitingPod := newWaitingPod(pod, pluginsWaitTime)
		f.waitingPods.add(waitingPod)
		msg := fmt.Sprintf("one or more plugins asked to wait and no plugin rejected pod %q", pod.Name)
		klog.V(4).Infof(msg)
		return framework.NewStatus(framework.Wait, msg)
	}
	return nil
}
```


[WaitOnPermit](https://sourcegraph.com/github.com/kubernetes/kubernetes@aea7bbadd2fc0cd689de94a54e5b7b758869d691/-/blob/pkg/scheduler/framework/runtime/framework.go?L1049:25&popover=pinned)
```go
// WaitOnPermit will block, if the pod is a waiting pod, until the waiting pod is rejected or allowed.
// 阻塞等Pod，直到Pod从Wait变为Success或非Success，如果变为Success后续调度器就可以对这个Pod进行PreBind，Bind
func (f *frameworkImpl) WaitOnPermit(ctx context.Context, pod *v1.Pod) *framework.Status {
	waitingPod := f.waitingPods.get(pod.UID)
	if waitingPod == nil {
		return nil
	}
	defer f.waitingPods.remove(pod.UID)
	klog.V(4).Infof("pod %q waiting on permit", pod.Name)

	startTime := time.Now()
	s := <-waitingPod.s
	metrics.PermitWaitDuration.WithLabelValues(s.Code().String()).Observe(metrics.SinceInSeconds(startTime))

	if !s.IsSuccess() {
		if s.IsUnschedulable() {
			msg := fmt.Sprintf("pod %q rejected while waiting on permit: %v", pod.Name, s.Message())
			klog.V(4).Infof(msg)
			s.SetFailedPlugin(s.FailedPlugin())
			return s
		}
		err := s.AsError()
		klog.ErrorS(err, "Failed waiting on permit for pod", "pod", klog.KObj(pod))
		return framework.AsStatus(fmt.Errorf("waiting on permit for pod: %w", err)).WithFailedPlugin(s.FailedPlugin())
	}
	return nil
}
```