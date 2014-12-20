首先看下fleet的整体模块图

![clipboard.png][1]

### server 数据结构
```
type Server struct {
    agent *agent.Agent    //  fleet维护的etcd客户端
    aReconciler *agent.AgentReconciler  // agent状态协调器
    usPub *agent.UnitStatePublisher // 状态分发器
    usGen *unit.UnitStateGenerator  //状态产生器
    engine *engine.Engine  // 集群管理， 选主， 协调
    mach *machine.CoreOSMachine // coreos机器状态管理
    hrt heart.Heart  // 触发心跳，注册机器存活状态到etcd
    mon *heart.Monitor // 心跳监控 
    api *api.Server  //  监听systemd的activation，处理fleetctl的请求

    engineReconcileInterval time.Duration // 协调间隔

    stop chan bool  //server停止标识
}
```

### 主流程

main
userset ： cfgPath，printVersion  
cfgset：  解析 [fleet.conf](https://github.com/coreos/fleet/blob/master/fleet.conf.sample)
> 生成config.Config

server.New
server.Run
> s.hrt.Beat   向etcd注册心跳 ，告诉大家我来了。
> go s.Monitor()  监控并心跳正常发送，如果异常，最多重试4次，在etcd保存设置机器存活的超时ttl。

-- m.check(hrt) 尝试发送心跳，成功则获得最近的更新索引（Node.ModifiedIndex）
              定期的注册server所在的机器的状态，汇报通过PeriodicRefresh生成的dynamicState ， 做服务发现。 

> go s.api.Available(s.stop)   将api的http状态设置为可用，即可以接受fleetctl请求

> go s.mach.PeriodicRefresh(machineStateRefreshInterval, s.stop)   定期更新dynamicState （相对于静态，存放的是机器相关ç配置信息）

-- m.Refresh()  定期的获取MachineState{ID，PublicIP，Metadata}, 更新dynamicState，最后往etcd汇报m.State的时候就会优先返回dynamicState
            
>  go s.agent.Heartbeat(s.stop)   

--  a.registry.UnitHeartbeat(j, machID, ttl) 定期的更新job-state到etcd。 

> go s.aReconciler.Run(s.agent, s.stop)   启动协调器，维护agent的状态不断的向etcd中state的状态同步

-- NewPeriodicReconciler(reconcileInterval, reconcile, ar.rStream)  每隔reconcileInterval执行一次Reconcile，

-- Run

--- 在eStream或者ticker到期 定时触发agent跟registry状态的同步

--- ar.Reconcile(a)    将agent的状态转移到期望状态

---- desiredAgentState（a,reg）从registry 获得agent的units ， jobs以及agent所在机器的状态

----- reg.Units() 获得所有的unit

----- reg.Schedule() 调度unit，按照unit名字排序

------ 获得所有 /etcd-key-prefix/job下面的job 以及状态（inactive | loaded | launched）

------ 获得/etcd-key-prefix/states/的 units 以及状态 (LOAD | ACTIVE  | SUB )

------ determineJobState(heartbeats[name], su.TargetMachineID, us) 根绝job最近的state 和其unit的的状态， 
生成满足期望的job。

---- calculateTaskChainsForUnits （dAgentState, cAgentState）  计算出agent需要满足期望的任务链，

---- ar.launchTaskChain(tc, a) 执行任务
                            
> go s.engine.Run(s.engineReconcileInterval, s.stop)    维护coreos集群的状态： 机器和ScheduledUnits，ScheduledUnits包括全局global units（所有的agent都执行的units）和jobs。

-- reconcile

--- renewLeadership 如果自己是leader，更新存活的租期为leaseTTL，如果不是则尝试acquireLeadership竞争leader，竞争的过程就是都去etcd创建一个/etcd-key-prefix/lease/engine-leader的key，首先创建的成为leader。 成为leader之后，首先执行

---- engine.Reconciler.Reconcile()

----- calculateClusterTasks  首先调度jobs，判断job调度到的目标机器是否可以被执行job，如果无法执行就将job设置为未调度。判断如下：

> * 如果job指定了target machineId，调度之
> * target machine 必须满足job需要的metadata（配置文件里面的MachineMetadata， 例如metadata="region=us-west,az=us-west-1"），
> * 跟machineof指定的job在同一个机器
> * 跟conflict指定的job不在同一个机器

----- 然后调用engine.leastLoadedScheduler.Decide, 选择一个负载最小（sortableAgentStates.Less）的机器，也就是执行scheduled units的job数最少的机器，将job 调度（clust.schedule(j.Name, dec.machineID)）到选出来的机器

> beatchan := make(chan *unit.UnitStateHeartbeat)    
> go s.usGen.Run(beatchan, s.stop)      心跳包生成，每1s心跳一次

-- Generate() 生成心跳，返回一个管道

---  lastSubscribed 填充

---- GetUnitStates （subscribed） 获得订阅的unit的状态，获得机器的report信息
         
> go s.usPub.Run(beatchan, s.stop)     分发心跳包， 跟sub通过beatchan来传递心跳

-- 生成一个每ttl/2s分发一次的goroutine

--- queueForPublish

--  生成numPublishers个分发toPublish（调用queueForPublish）的任务

--- toPublish是一个队列，存放的是unit的名字， 从这里看出来，fleet报告的状态是最终状态，而不是历史连续的一个状态。

---  生成一个分发beatchan的goroutine

---- queueForPublish

  [1]: http://www.serfdom.cn/usr/uploads/2014/12/3607585694.png

