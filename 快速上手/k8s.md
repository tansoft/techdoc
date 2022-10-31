# 概念

## 核心组件

- etcd保存了整个集群的状态；
- apiserver提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
- controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
- scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
- kubelet负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；
- Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）；
- kube-proxy负责为Service提供cluster内部的服务发现和负载均衡；

## 推荐Add-ons

- kube-dns负责为整个集群提供DNS服务
- Ingress Controller为服务提供外网入口
- Heapster提供资源监控
- Dashboard提供GUI
- Federation提供跨可用区的集群
- Fluentd-elasticsearch提供集群日志采集、存储与查询

## API设计原则

* 声明式
* 每个API对象都有3大类属性：元数据metadata、规范spec和状态status。
* 元数据是用来标识API对象的，每个对象都至少有3个元数据：namespace，name和uid；例如标签env标识部署环境：env=dev、env=testing、env=production。
* Status描述了系统实际当前达到的状态。

## Node 节点

* 对应物理机。

## Pod

* 最小运行单元，可支持多个容器，容器之间共享网络和文件，可进程间通信。
* Pod的运行类型：

### Deployment：长期运行 Long-running

### Job：批处理任务Batch

运行完成即退出，根据 spec.completions 策略不同：

* 单Pod型任务有一个Pod成功就标志完成
* 定数成功型任务保证有N个任务全部成功
* 工作队列型任务根据应用确认的全局成功而标志成功

### DaemonSet：节点后台服务 Node-daemon

保证每个节点Node有且只有一个实例在运行，包括：存储、日志、监控等服务。

### PetSet：有状态应用 Stateful App

Pod有状态，名字唯一。挂载自己独立的存储，故障漂移后，会加载原来的存储，如：数据库，Zookeeper，etcd等。

## Deployment 部署

* 部署，升级，变更服务

### RC 复制控制器 Replication Controller

* 监控Pod按指定数目运行，只支持Long-running，较早概念，可被RS替代。

### RS 副本集 Replica Set

* 支持多种部署模式，作为Deployment的理想状态参数使用。

## Service 服务

提供Pod的服务发现和负载均衡。Service有对应内部虚拟IP，负载均衡是由Kube-proxy实现，每个节点都有部署。

## Federation 集群联邦

多个k8s统一管理。每个Federation有自己的分布式存储、API Server和Controller Manager。

* 创建、更改会在所有子K8s集群中执行。
* 提供业务请求服务时，会先在各个子K8s集群之间做负载均衡。
* 对于发送到具体K8s集群的业务请求和普通k8s一致。
* 集群之间负载均衡通过域名实现。

## Volume 存储卷

* PVC 对应 Pod，PV 对应 Node。

* PVC 持久化存储声明 Persistent Volume Claim

* PV 持久存储卷 Persistent Volume

## Secret 密钥对象

用来保存和传递密码、密钥、认证凭证这些敏感信息的对象。

## 用户帐户（User Account）和服务帐户（Service Account）
* 用户帐户对应人，与服务 namespace 无关
* 服务帐户是运行程序的身份，属于特定 namespace

## 名字空间 Namespace

* 默认空间 default
* 系统名字空间 kube-system

## 访问授权

* RBAC 基于角色的访问控制 Role-based Access Control：引入角色（Role）和角色绑定（RoleBinding）概念。

* ABAC 基于属性的访问控制 Attribute-based Access Control：ABAC 直接跟用户关联。