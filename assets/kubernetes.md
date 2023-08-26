官方中文文档： https://kubernetes.io/zh-cn/docs/concepts/overview/

Kubernetes 是一个可移植、可扩展的开源平台，用于管理容器化的工作负载和服务，可促进声明式配置和自动化。 Kubernetes 拥有一个庞大且快速增长的生态，其服务、支持和工具的使用范围相当广泛。

Kubernetes 为你提供了一个可弹性运行分布式系统的框架。 Kubernetes 会满足你的扩展要求、故障转移你的应用、提供部署模式等。 

Kubernetes 为你提供：

- **服务发现和负载均衡**
    
    Kubernetes 可以使用 DNS 名称或自己的 IP 地址来暴露容器。 如果进入容器的流量很大， Kubernetes 可以负载均衡并分配网络流量，从而使部署稳定。
    

- **存储编排**
    
    Kubernetes 允许你自动挂载你选择的存储系统，例如本地存储、公共云提供商等。
    

- **自动部署和回滚**
    
    你可以使用 Kubernetes 描述已部署容器的所需状态， 它可以以受控的速率将实际状态更改为期望状态。 例如，你可以自动化 Kubernetes 来为你的部署创建新容器， 删除现有容器并将它们的所有资源用于新容器。
    

- **自动完成装箱计算**
    
    你为 Kubernetes 提供许多节点组成的集群，在这个集群上运行容器化的任务。 你告诉 Kubernetes 每个容器需要多少 CPU 和内存 (RAM)。 Kubernetes 可以将这些容器按实际情况调度到你的节点上，以最佳方式利用你的资源。
    

- **自我修复**
    
    Kubernetes 将重新启动失败的容器、替换容器、杀死不响应用户定义的运行状况检查的容器， 并且在准备好服务之前不将其通告给客户端。
    

- **密钥与配置管理**
    
    Kubernetes 允许你存储和管理敏感信息，例如密码、OAuth 令牌和 SSH 密钥。 你可以在不重建容器镜像的情况下部署和更新密钥和应用程序配置，也无需在堆栈配置中暴露密钥。


## Kubernetes对象

 https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/
 在 Kubernetes 系统中，**Kubernetes 对象**是持久化的实体。 Kubernetes 使用这些实体去表示整个集群的状态。 具体而言，它们描述了如下信息：
 - 哪些容器化应用正在运行（以及在哪些节点上运行）
- 可以被应用使用的资源
- 关于应用运行时行为的策略，比如重启策略、升级策略以及容错策略
 一旦创建该对象， Kubernetes 系统将不断工作以确保该对象存在。通过创建对象，你本质上是在告知 Kubernetes 系统，你想要的集群工作负载状态看起来应是什么样子的， 这就是 Kubernetes 集群所谓的**期望状态（Desired State）**。
 几乎每个 Kubernetes 对象包含两个嵌套的对象字段，它们负责管理对象的配置： 对象 **`spec`（规约）** 和对象 **`status`（状态）**。 对于具有 `spec` 的对象，你必须在创建对象时设置其内容，描述你希望对象所具有的特征： **期望状态（Desired State）**。

在想要创建的 Kubernetes 对象所对应的 `.yaml` 文件中，需要配置的字段如下：

- `apiVersion` - 创建该对象所使用的 Kubernetes API 的版本
- `kind` - 想要创建的对象的类别
- `metadata` - 帮助唯一标识对象的一些数据，包括一个 `name` 字符串、`UID` 和可选的 `namespace`
- `spec` - 你所期望的该对象的状态

常用对象的类型有：
- pod
- service
- deployment
## kubernetes组件

- Node:一组工作机器，称为[节点](https://kubernetes.io/zh-cn/docs/concepts/architecture/nodes/)， 会运行容器化应用程序。每个集群至少有一个工作节点。
- Pod:Pod 就是作为应用负载的组件
- Control Plane:管理集群中的工作节点和 Pod.在生产环境中，控制平面通常跨多台计算机运行， 一个集群通常运行多个节点，提供容错性和高可用性。通常会在同一个计算机上启动所有控制平面组件， 并且不会在此计算机上运行用户容器。
	- 包含的其他组件...
- 插件:插件使用 Kubernetes 资源（[DaemonSet](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/daemonset/)、 [Deployment](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/) 等）实现集群功能。dns dashboard...

## kubernetes架构
![[Pasted image 20230826163645.png]]
- 节点到控制面的连接：Kubernetes 采用的是中心辐射型（Hub-and-Spoke）API 模式。 所有从节点（或运行于其上的 Pod）发出的 API 调用都终止于 API 服务器。 其它控制面组件都没有被设计为可暴露远程服务。连接是安全的。
- 控制面到节点的连接：
	- API 服务器到 kubelet
	- API 服务器到节点、Pod 和服务