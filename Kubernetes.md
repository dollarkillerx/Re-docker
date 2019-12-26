# Kubernetes

### Kubernetes 架构与组件
- master     # 管理
    - api server
    - kubelet
    - kube-proxy
- node pool  # 节点机器
    - pod        #  单个服务        
    - kubelt
    - kube-proxy

#### Master 
- 集群所有的控制命令都在master执行
- 每个kubernetes至少一个master  (单点故障避免集群master)
- Api server:
    - k8s集群控制restful api核心组建
    - 集群各组建之间数据交互和通信中枢
    - 提供集群控制的安全机制(身份任务,授权 以及admission control)
- Scheduler:
    - 通过API Server的Watch接口监听新建Pod副本信息，并通过调度算法为该Pod选择一个最合适的Node
    - 支持自定义调度算法provider
    - 默认调度算法内置预选策略和优选策略，决策考量资源需求，服务质量，软硬件约束，
- ControllerManager
    - 集群内各种资源controller的核心管理者
    - 针对每一种具体的资源，都有相应的controller
    - 保证旗下管理的每一个controller对应的资源始终处于 期望状态
```
每一Controller逻辑:
for {
    获取资源期望状态
    获取资源当前状态
    改变: 当前状态->期望状态
}
```
#### Node组件
- Pod
- kubelet
- Kube-proxy
Kubernetes集群工作负载节点
    - kubernetes集群由多个node共同承担工作负载，Pod被分配到某一个具体的Node上执行
    - kubernetes通过node controller对node资源进行管理。支持动态在集群中添加或删除Node
    - m 






