# Scheduling

> Introduction to scheduling mechanism

The scheduling mechanism is mainly by summarizing the collected cluster real-time status information (including the heartbeat of the region and the heartbeat of the store), and then judging whether scheduling needs to be generated according to different strategies, and then generate scheduling `Operator` which send to TiKV to achieve scheduling.

Each scheduling `Operator` is only used to operate a region migration, which including add some peers, transfer the raft group leader and remove some peers. an `Operator` will record the ID of the operator region, the relative strategy name, operator priority, and specific scheduling steps, etc. The `Operator` in PD may be generated from two operate, one is the `checker` and the other is the `scheduler`. The generated schedule will be stored in a map, and then it will be returned to the corresponding TiKV through the response when the region heartbeat comes. Let us first look at how checker and scheduler work.

## Checker

After PD server started, there is a [background worker](https://github.com/tikv/pd/blob/release-5.2/server/cluster/coordinator.go#L90-L158) will polling all regions and then check the health status of each region:

```Go
    func (c *coordinator) patrolRegions() {
        ...
        timer := time.NewTimer(c.cluster.GetPatrolRegionInterval())
        ...
        for {
            select {
            case <-timer.C:
                timer.Reset(c.cluster.GetPatrolRegionInterval())
            case <-c.ctx.Done():
                log.Info("patrol regions has been stopped")
                return
            }

            regions := c.cluster.ScanRegions(key, nil, patrolScanRegionLimit)
            ...
            for _, region := range regions {
                ...
                if c.checkRegion(region) {
                    break
                }
            }
            ...
        }
    }

```

In this function, the `checkRegion` will be executed to determine whether the region needs to be scheduled. If a schedule `Operator` is generated, it will be send to TiKV through the heartbeat of this region. The initialization of the checker can be found in the file `coordinator.go`, which mainly contains [four checkers](https://github.com/tikv/pd/blob/release-5.2/server/schedule/checker_controller.go#L64-L107): `RuleChecker`, `MergeChecker`, `JointStateChecker`, `SplitChecker`.

`RuleChecker` is the most critical checker. It will check whether a region has down peers or offline peers. It will also check whether the number of replicas of the current region is the same as the number of replicas specified in the [`Placement Rules`](https://docs.pingcap.com/tidb/stable/configure-placement-rules#placement-rules). If the conditions are met, it will trigger the logic of the corresponding supplementary replicas or delete the redundant replicas. Logic. In addition, `RuleChecker` will also check whether the current copy of this region is placed in the most reasonable place, and if it is not, it will be placed in a more reasonable place.

`MergeChecker` It will check whether the current region meets the merge conditions, such as whether the size of the region is less than max-merge-region-size, whether the key of the region is less than `max-merge-region-keys`, and whether there has been no split operation in the recent period, etc. If these conditions are met, an adjacent region will be selected to try to merge the two regions.

## Scheduler

Let's first take a look at the code about the scheduler running process. schedulers are running concurrency. each scheduler has a [scheduler controller](https://github.com/tikv/pd/blob/release-5.2/server/cluster/coordinator.go#L762-L789), which running in a background worker:

```Go
    func (c *coordinator) runScheduler(s *scheduleController) {
        defer logutil.LogPanic()
        defer c.wg.Done()
        defer s.Cleanup(c.cluster)

        timer := time.NewTimer(s.GetInterval())
        defer timer.Stop()

        for {
            select {
            case <-timer.C:
                timer.Reset(s.GetInterval())
                if !s.AllowSchedule() {
                    continue
                }
                if op := s.Schedule(); len(op) > 0 {
                    added := c.opController.AddWaitingOperator(op...)
                    log.Debug("add operator", zap.Int("added", added), zap.Int("total", len(op)), zap.String("scheduler", s.GetName()))
                }

            case <-s.Ctx().Done():
                log.Info("scheduler has been stopped",
                    zap.String("scheduler-name", s.GetName()),
                    errs.ZapError(s.Ctx().Err()))
                return
            }
        }
    }
```

Similar to the checker, when the PD is started, the specified scheduler will be added according to the configuration. Each scheduler runs in a goroutine by executing `go runScheduler`, and then executes the `Schedule()` function at a dynamically adjusted time interval. There are two things that a function has to do. The first is to execute the scheduling logic of the corresponding scheduler to determine whether to generate an `Operator`, and the other is to determine the time interval for the next execution of `Schedule()`.
PD contains many schedulers. For details, you can check the schedulers package, which contains the implementation of all schedulers. The schedulers that PD will run by default include `balance-region-scheduler`, `balance-leader-scheduler`, `balance-hot-region-scheduler`. Let's take a look at the specific functions of these schedulers:

- The [`balance-region-scheduler`](https://github.com/tikv/pd/blob/release-5.2/server/schedulers/balance_region.go#L136-L144) calculates a score based on the size of the region size on a store and the usage of available space. Then, according to the calculated score, the region is evenly distributed to each store through the `Operator` that generates the balance-region. The reason why the available space is considered here is that the actual situation may have differences in storage capacity of different stores.
- The [`balance-leader-scheduler`](https://github.com/tikv/pd/blob/release-5.2/server/schedulers/balance_leader.go#L137-L153) is similar to the balance-region-scheduler. It calculates a score based on the region count, and then uses the `Operator` that generates the balance-leader to distribute the leaders as evenly as possible across the stores.
- The [`balance-hot-region-scheduler`](https://github.com/tikv/pd/blob/release-5.2/server/schedulers/hot_region.go#L152-L155) implements the related logic of hotspot scheduling. For TiDB, if there are hotspots and only a few stores have hotspots, then the overall resource utilization of the system will be lowered, and it is easy to form a system bottleneck. Therefore, PD also needs to count the hotspots in response to this situation. Come out, and by generating the corresponding schedule, the hot spots are scattered to all stores as much as possible. So that all stores can share the pressure of reading and writing.

There are some other schedulers to choose from. Each scheduler of PD implements an interface called [Scheduler](https://github.com/tikv/pd/blob/release-5.2/server/schedule/scheduler.go#L33-L45):

```GO
    // Scheduler is an interface to schedule resources.
    type Scheduler interface {
        http.Handler
        GetName() string
        // GetType should in accordance with the name passing to schedule.RegisterScheduler()
        GetType() string
        EncodeConfig() ([]byte, error)
        GetMinInterval() time.Duration
        GetNextInterval(interval time.Duration) time.Duration
        Prepare(cluster opt.Cluster) error
        Cleanup(cluster opt.Cluster)
        Schedule(cluster opt.Cluster) []*operator.Operator
        IsScheduleAllowed(cluster opt.Cluster) bool
    }
```

The most important thing is the `Schedule()` interface function, which is used to implement the specific scheduling-related logic of each scheduler. In addition, the interface function `IsScheduleAllowed()` is used to determine whether the scheduler is allowed to execute the corresponding scheduling logic. Before executing the scheduling logic, each scheduler will first call this function to determine whether the scheduling rate is exceeded. Specifically, in the code, this function is called `AllowSchedule()`, and then the `IsScheduleAllowed()` method implemented by different schedulers is called.

PD can control the speed at which the scheduler generates operators by setting the limit, but the limit here is just one that maintains a window size, and different operator types have their own window sizes. For example, `balance-region` and `balance-hot-region` two schedulers will generate operators related to migrate region, and the type of this operator is `OpRegion`. We can control the operator of this type of `OpRegion` by adjusting the `region-schedule-limit` parameter. Concurrency. The specific operator type definition can be found in the file [`operator.go`](https://github.com/tikv/pd/blob/release-5.2/server/schedule/operator/operator.go). An operator may contain multiple types. For example, the operator generated by `balance-hot-region` may belong to both `OpRegion` and `OpHotRegion`.

## More

This section mainly introduces the main operation process of PD scheduling. For more details, you can continue to refer to the corresponding code study. and welcome to contribute some [first-friendy issues](https://github.com/tikv/pd/issues?q=is%3Aissue+is%3Aopen+label%3A%22help+wanted%22).
