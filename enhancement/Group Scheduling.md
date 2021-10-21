## 关于在调度层面完成pod分组分发：
使用scheduler with multiple profiles + scheduler extender  
multiple-profiles (https://kubernetes.io/docs/reference/scheduling/config/#multiple-profiles)  
scheduler extender (https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/scheduler_extender.md)  

1. 将分组pod进行分组调度:  
方案一：  
我们可以设计一个filter插件用于筛选分组pod和非分组pod  
使用multiple-profiles创建两个scheduler，一个原生scheduler和分组scheduler  
在这两个scheduler中加入分组筛选插件，原生scheduler只管非分组pod，分组scheduler管理分组pod  
方案二：  
使用pod.Spec.SchedulerName来指定调度器  

2. 使用Propogation Policy进行调度：  
启动一个新的Propogation Policy Controller，用于对Propogation Policy的期望状态和实际状态进行reconcile。  
分组scheduler设置scheduler extender与Propogation Policy Controller合作共同完成调度决定（能合作到什么地步有待考证）  
Propogation Policy Controller需要介入pod生命周期管理，对于当前集群中不符合Propogation Policy的pod，可能进行删除，然后重新走调度流程。  

劣势：
1. Scheduler Extender通过http与controller进行合作调度，序列化和通信延时可能会变高（可能可以通过Affinity和UDS来解决）
2. 不能共享informer缓存（这个问题应该不大）
3. Extender能产生影响的地方，在整个scheduler调度流程中比较少

优势：
1. 对于不分组的pod没有影响，走原生调度过程
2. 分组scheduler仍然可以使用原生的调度插件，按照（原生插件 AND Extender）获取最终调度结果

