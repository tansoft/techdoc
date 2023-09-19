# 概念

* https://kubernetes.io/zh-cn/
* http://docs.kubernetes.org.cn/

## 核心组件

- 虽然可以在集群任意节点运行，但是一般会有专用节点。
- apiserver：提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
- etcd：保存了整个集群的状态，需要考虑备份计划。
- controller-manager：负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
- cloud-controller-manager：与底层云提供商的平台交互
- scheduler：负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
- Add-ons
  - kube-dns负责为整个集群提供DNS服务
  - kube-ui提供集群状态基础信息查看
  - Heapster提供资源监控
  - Ingress Controller为服务提供外网入口
  - Federation提供跨可用区的集群
  - Fluentd-elasticsearch提供集群日志采集、存储与查询

## 节点组件

- kubelet：负责维护容器的生命周期，下载Secrets，安装Volume（CVI）和网络（CNI），运行docker/experimentally/rkt，健康检查。
- kube-proxy：负责为Service提供cluster内部的服务发现和负载均衡；
- supervisord：轻量级监控系统，用于保障kubelet和docker运行
- fluentd：守护进程，可提供cluster-level logging。

## API设计原则

* 声明式
* 每个API对象都有3大类属性：元数据metadata、规范spec和状态status。
* 元数据是用来标识API对象的，每个对象都至少有3个元数据：namespace，name和uid；还有常用如 Labels 标签：env标识部署环境：env=dev、env=testing、env=production。
* Spec描述了对象所需的状态 - 希望Object具有的特性。
* Status描述了对象的实际状态，并由Kubernetes系统提供和更新。

## Node 节点

* 对应物理机。
* 节点驱逐策略：
  * 节点控制器同时检查区域中节点的不健康百分比（NodeReady条件为ConditionUnknown或ConditionFalse）。
  * 如果不健康节点的比例为 --unhealthy-zone-threshold（默认为0.55），那么驱逐速度就会降低（因为可能是可用区大规模故障）。
  * 如果集群很小（即 <= --large-cluster-size-threshold节点 - 默认值为50），则停止驱逐，否则， --secondary-node-eviction-rate（默认为0.01）每秒逐出。
  * 默认状态逐出速率：--node-eviction-rate（默认为0.1）/秒，这意味着最快每10秒从1个节点驱逐Pod。

## Pod

* 最小运行单元，可支持多个容器，容器之间共享网络和文件，可进程间通信。

### Pod的运行类型

#### Deployment：长期运行 Long-running

Deployment比ReplicaSet高级，而ReplicaSet代替ReplicationController

#### Job和CronJob：批处理任务Batch

运行完成即退出，根据 spec.completions 策略不同：

* 单Pod型任务有一个Pod成功就标志完成
* 定数成功型任务保证有N个任务全部成功
* 工作队列型任务根据应用确认的全局成功而标志成功

Job类型：

* 单一Job：只运行一个Pod，成功终止视为完成。
* 对预期不会终止的 Pod 使用 ReplicationController、ReplicaSet 和 Deployment ，例如 Web 服务器。
* 确定任务完成数的并行Job：
  * `.spec.completions` 设置任务数，默认值1。
  * 成功Pod 个数达到 `.spec.completions` 时，Job 视为完成。
  * 使用 `.spec.completionMode="Indexed"` 时，每个 Pod 都会获得一个不同的索引值0-N。
  * 也可以设置并行度 `.spec.parallelism`
  * 可以设置 `.spec.completionMode` 完成模式：
    - `NonIndexed`：默认值。
    - `Indexed`：Pod 完成时会获得对应的完成索引，0..N，获取方式：
      - Pod 注解 `batch.kubernetes.io/job-completion-index`。
      - Pod 主机名的一部分，遵循模式 `$(job-name)-$(index)`。 带索引的 Job（Indexed Job）与 Service，可通过 DNS 使用确切主机名访问。
      - 在容器环境变量 `JOB_COMPLETION_INDEX` 中。
* 带工作队列的并行Job：
  * 不设置 `spec.completions`，设置 `.spec.parallelism`，默认值1。
  * 一个 Pod 已成功终止则不再创建新 Pod。其他Pod处理完成后退出。
  * 一旦至少 1 个 Pod 成功完成，并且所有 Pod 都已终止，即可宣告 Job 成功完成。
* 通过 ```.spec.template.spec.restartPolicy``` 设置失败是否重试。
* 通过 ```.spec.backoffLimit``` 限制失败重试次数。
* 通过 ```.spec.activeDeadlineSeconds``` 限制总重试时间。
* 通过 ```.spec.ttlSecondsAfterFinished``` 设置自动清理已完成Job时间。

##### CronJob

* ```.spec.startingDeadlineSeconds```：如果任务因为某种原因错过了调度时间，判断是否达到这个最大Deadline，如果未到则进行调度。注意控制器检查间隔是10秒，这个值设置太小可能不生效。
* ```.spec.concurrencyPolicy```：设置调度任务的并发度：
  * Allow：默认值，允许并发执行。
  * Forbid：不允许并发。新任务要执行时而老任务没有执行完，CronJob 会忽略新任务执行。
  * Replace：新任务替换老任务。

#### DaemonSet：节点后台服务 Node-daemon

保证每个节点Node有且只有一个实例在运行，包括：存储、日志、监控等服务。

#### StatefulSet（PetSet）：有状态应用 Stateful App

* Pod有状态，名字唯一，多个副本从0开始命名。

* 挂载自己独立的存储，故障漂移后，会加载原来的存储，如：数据库，Zookeeper，etcd等。

* 网络名称唯一，多个副本从-0开始命名，如：web-0，web-1

* 可通过无头服务名称进行访问：$(ServiceName).$(NameSpace).svc.cluster.local，其中 cluster.local 是集群域。Pod 也有独立名字：$(pod 名称)-N.$(服务域名)，pod名称由 StatefulSet 的 `serviceName` 域来设定。

* 注意DNS可能会延后，可以通过API查询或减少DNS缓存时长（30秒）。

* 扩展时顺序0..N，缩容时N..0。

* StatefulSet 不应将 `pod.Spec.TerminationGracePeriodSeconds` 设置为 0。 这种做法是不安全的。缩容时为了优雅退出，不应该直接删除，而是缩容为0再删除。

* 更新策略：通过 StatefulSet 的 `.spec.updateStrategy` 字段指定：

  - `OnDelete`：控制器不会自动更新 Pod。 必须手动删除 Pod 以便让控制器创建新 Pod，以此来对 StatefulSet 的 `.spec.template` 的变动作出反应。
  - `RollingUpdate`：对 Pod 执行自动的滚动更新。这是默认的更新策略。耗时Pod可以设置 ```.spec.minReadySeconds```
  - 通过 ```.spec.updateStrategy.rollingUpdate.partition``` 可以指定分区滚动更新。
  - 通过 ```.spec.updateStrategy.rollingUpdate.maxUnavailable``` 指定滚动数量。
  - 更新出错是等待不是回滚。

* PersistentVolumeClaim 存储保留策略：

  * 在 StatefulSet 的生命周期中，可选字段 `.spec.persistentVolumeClaimRetentionPolicy` 控制是否删除以及如何删除 PVC。 需要启用 `StatefulSetAutoDeletePVC` 可以配置两个策略：

    - `whenDeleted`：配置删除时卷保留行为。
    - `whenScaled`：配置缩减时卷保留行为。

    对于你可以配置的每个策略，你可以将值设置为 `Delete` 或 `Retain`。

    - `Delete`：删除。

    - `Retain`（默认）：Pod 删除时保留。

## Controller

Controller可以创建和管理多个Pod，提供副本管理、滚动升级和集群级别的自愈能力。例如，如果一个Node故障，Controller就能自动将该节点上的Pod调度到其他健康的Node上。

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

* PV 持久存储卷 Persistent Volume：声明存储

* PVC 持久化存储声明 Persistent Volume Claim：表达使用者的需求

* PV 对应 Node，PVC 对应 Pod。
* PV 不属于任何namespace，PVC属于某个特定namespace
* 支持类型：

```bash
# 典型用法
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: gcr.io/google_containers/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume					# 指定volumes名字
      subPath: html								# 不指定默认是/，指定是源盘上的对应目录
      subPathExpr: $(POD_NAME)		# 可以使用变量进行设置
      readOnly: true
  volumes:
  - name: cache-volume
    emptyDir: {}									# 不同类型的存储

# node上建的临时目录，销毁时删除
# emptyDir.medium 设置为 Memory，可以挂载内存盘 tmpfs
emptyDir: {}

# 可以挂载 host 主机的目录
# 一般不建议，容易有安全性问题，如果AdmissionPolicy限制了还需要readOnly挂载
hostPath:
	path: /data
  type: Directory									# 常用设置还包括：DirectoryOrCreate，File，FileOrCreate，Socket 等，FileOrCreate不会创建父目录

# Local，和hostPath类似，但是Local可由k8s自动调度，因为支持对kube-reserved和system-reserved指定本地存储资源
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem		# 如果是 Block 则暴露原始块设备
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:							# 通过节点亲和性选择对应的机器
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node

# 获取git仓库
gitRepo:
  repository: "git@somewhere:me/my-git-repository.git"
  revision: "22f1d8406d464b0c0874075539c1f2e96c253775"

#pvc
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 10Gi

persistentVolumeClaim:
	claimName: block-pvc

# aws 已淘汰方式
aws ec2 create-volume --availability-zone eu-west-1a --size 10 --volume-type gp3
awsElasticBlockStore:
	volumeID: <volume-id>
	fsType: ext4

# gcp
gcloud compute disks create --size=500GB --zone=us-central1-a my-data-disk
gcePersistentDisk:
	pdName: my-data-disk
	fsType: ext4

# NFS
volumes:
- name: test-volume
	nfs:
		server: my-nfs-server.example.com
		path: /my-nfs-volume
		readonly: true

# 加密信息
secret:
  secretName: mysecret
  defaultMode: 0400
  items:
  - key: username
		path: my-group/my-username
  optional: false # default setting; "mysecret" must exist

# 配置信息
configMap:
	name: log-config
	items:
	- key: log_level
		path: log_level

# projected
# 多个源volume映射到单个目录
projected:
	sources:
	- xxx:

```

### StorageClass 动态卷制备

不需要预先声明PV，在需要时直接新建。

添加注解 `storageclass.kubernetes.io/is-default-class` 来指定默认的 `StorageClass` 

```yaml
# aws ebs
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "10"		# 只适用于 io1 卷。每 GiB 每秒 I/O 操作。
  fsType: ext4
  encrypted: false
  kmsKeyId: xxx

# gcp
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard

# 使用时
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 30Gi
```

### CSI 容器存储接口

各种形式的存储都通过CSI来实现。常用配置字段：

* ```driver```：驱动程序的标准名称。
* ```volumeHandle```：唯一标识。CSI中的volume_id。
* ```readOnly```：是否只读加载。
* ```fsType```：如果VolumeMode为Filesystem，这里指定文件系统格式化类型。
* ```volumeAttributes```：对应属性定义。

#### CSI卷克隆

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: clone-of-pvc-1
    namespace: myns
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: cloning
  resources:
    requests:
      storage: 5Gi
  dataSource:									# 指定克隆来源
    kind: PersistentVolumeClaim
    name: pvc-1
```

### 挂载卷的传播

卷可以共享到同一Pod中其他容器，或同节点的其他Pod。由 `Container.volumeMounts` 中的 `mountPropagation` 字段控制。

* ```None```：默认模式。不感知挂载变化。
* ```HostRoContainer```：rslave模式。只有挂载到主机的卷或子目录，会让容器看到。
* ```Bidirectional```：rshared模式。特权容器才可以使用。容器的挂载会传播给整个主机和容器。
* 某些部署环境还需要在Docker的systemd服务文件中设置MountFlags=shared

### 卷的备份

* VolumeSnapshotContent 对应 PV，VolumeSnapshot 对应PVC。VolumeSnapshotClass 指定 VolumeSnapshot 的不同属性。

## Secret 密钥对象

用来保存和传递密码、密钥、认证凭证这些敏感信息的对象。

### 更新 secret，如更新证书：
```bash
kubectl create secret tls hello \
  --namespace test \
  --key ./privkey.pem \
  --cert ./fullchain.pem \
  --dry-run \
  -o yaml \
  | \
  kubectl apply -f -
```

## Labels 标签

可用于快速查找过滤，筛选符合条件的资源。如：

* environment=production,tier!=frontend
* environment in (production, qa)
* tier notin (frontend, backend)
* partition
* !partition

### 推荐标签

| 键                             | 描述                                             | 示例           | 类型   |
| ------------------------------ | ------------------------------------------------ | -------------- | ------ |
| `app.kubernetes.io/name`       | 应用程序的名称                                   | `mysql`        | 字符串 |
| `app.kubernetes.io/instance`   | 用于唯一确定应用实例的名称                       | `mysql-abcxzy` | 字符串 |
| `app.kubernetes.io/version`    | 应用程序的当前版本（例如语义版本、修订版哈希等） | `5.7.21`       | 字符串 |
| `app.kubernetes.io/component`  | 架构中的组件                                     | `database`     | 字符串 |
| `app.kubernetes.io/part-of`    | 此级别的更高级别应用程序的名称                   | `wordpress`    | 字符串 |
| `app.kubernetes.io/managed-by` | 用于管理应用程序的工具                           | `helm`         | 字符串 |

## Annotations 注解 

类似标签：

* 构建、发布的镜像信息，如时间戳，发行ID，git分支，PR编号，镜像hashes和注Registry地址。
* 一些日志记录、监视、分析或audit repositories。
* 一些工具信息：例如，名称、版本和构建信息，联系方式。
* 用户或工具/系统来源信息，例如来自其他生态系统组件对象的URL。

## User Account 用户帐户和 Service Account 服务帐户

* 用户帐户对应人，与服务 namespace 无关
* 服务帐户是运行程序的身份，属于特定 namespace

## Namespace 名字空间

* 默认空间 default
* 系统名字空间 kube-system
* 低级别资源（如Node和persistentVolumes）不属于Namespace
* Events是一个例外：它们可能有也可能没有Namespace，具体取决于Events的对象。

## ConfigMap 配置

* ConfigMap的修改会自动更新，但是环境变量不会更新

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # 定义环境变量
        - name: PLAYER_INITIAL_LIVES # 请注意这里和 ConfigMap 中的键名是不一样的
          valueFrom:
            configMapKeyRef:
              name: game-demo           # 这个值来自 ConfigMap
              key: player_initial_lives # 需要取值的键
      volumeMounts:											# Pod中有/config/game.properties等两个文件
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:															# 声明两个配置文件 *.properties
    - name: config
      configMap:
        name: game-demo
        items:													# 如果不指定items，那会按configmap项生成所有配置文件
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"
```



## 访问授权

* RBAC 基于角色的访问控制 Role-based Access Control：引入角色（Role）和角色绑定（RoleBinding）概念。

* ABAC 基于属性的访问控制 Attribute-based Access Control：ABAC 直接跟用户关联。

## 调度和优先级

### scope 作用域

| 作用域                      | 描述                                                |
| --------------------------- | --------------------------------------------------- |
| `Terminating`               | 匹配 `spec.activeDeadlineSeconds` 不小于 0 的 Pod。 |
| `NotTerminating`            | 匹配 `spec.activeDeadlineSeconds` 是 nil 的 Pod。   |
| `BestEffort`                | 匹配 Qos 是 BestEffort 的 Pod。                     |
| `NotBestEffort`             | 匹配 Qos 不是 BestEffort 的 Pod。                   |
| `PriorityClass`             | 匹配所有引用了所指定优先级类的 Pods。               |
| `CrossNamespacePodAffinity` | 匹配设置了跨名字空间（反）亲和性条件的 Pod。        |

### 匹配逻辑

```operator``` 可以支持 ```In```，```NotIn```，```Exists```，```DoesNotExist``` 匹配。

* 其中 In 是必须有values的，exists不用。

```yaml
  scopeSelector:
    matchExpressions:
      - scopeName: PriorityClass
        operator: In
        values:
          - middle
```

### 基于优先级调度

* 定义高中低三个优先级

```yaml
apiVersion: v1
kind: List
items:
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-high
  spec:
    hard:
      cpu: "1000"
      memory: 200Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["high"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-medium
  spec:
    hard:
      cpu: "10"
      memory: 20Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["medium"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-low
  spec:
    hard:
      cpu: "5"
      memory: 10Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["low"]
```

* 使用高优先级启动Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority
spec:
  containers:
  - name: high-priority
    image: ubuntu
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
    resources:
      requests:
        memory: "10Gi"
        cpu: "500m"
      limits:
        memory: "10Gi"
        cpu: "500m"
  priorityClassName: high
```

### 调度逻辑

#### nodeSelector 节点选择器

#### 亲和性和反亲和性

* 使用  `.spec.affinity.nodeAffinity` 字段设置节点亲和性。 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        # 节点必须在某个区域
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In	# NotIn，Exists，DoesNotExist，Gt，Lt
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      # 权重 1-100，符合的Pod会增加评分，最后判断权重总和最高的节点放置。
      - weight: 1
        preference:
        	# 节点最好有 another-node-label-value in another-node-label-key
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
      - weight: 50
        preference:
          matchExpressions:
          - key: label-2
            operator: In
            values:
            - key-2
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0
```



* 可以标明规则是“偏好”，在无法匹配时可以调度。
  * `requiredDuringSchedulingIgnoredDuringExecution`： 调度器只有在规则被满足的时候才能执行调度。此功能类似于 `nodeSelector`， 但其语法表达能力更强。
  * `preferredDuringSchedulingIgnoredDuringExecution`： 调度器会尝试寻找满足对应规则的节点。如果找不到匹配的节点，调度器仍然会调度该 Pod。
* 可以定义规则允许哪些Pod放置在一起。
* `NotIn` 和 `DoesNotExist` 可用来实现节点反亲和性行为。也可以使用节点污点把 Pod 从特定节点驱逐。

##### Pod间亲和性

如果 X 上已运行一或多个满足规则 Y 的 Pod， 则这个 Pod 应运行在 X 上。

- 通过 `topologyKey` 来表达拓扑域（X）的概念。
- Pod 间亲和性，使用 `.affinity.podAffinity` 字段。
- Pod 间反亲和性，使用 `.affinity.podAntiAffinity` 字段。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
  	# 同区域已有运行security=S1的Pod，调度到该Pod使用的节点上。
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
    # 同区域已有运行security=S2的Pod，不调度到该Pod使用的节点上。
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: registry.k8s.io/pause:2.0
```

###### 实际案例

- Nginx+Redis场景

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```

##### 调度方案中设置节点亲和性

```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta3
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
  - schedulerName: foo-scheduler
    pluginConfig:
      - name: NodeAffinity
        args:
          addedAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: scheduler-profile
                  operator: In
                  values:
                  - foo
```

#### nodeName 字段

#### Pod 拓扑分布约束



# Pod

## Pod的状态 phase

* 挂起（Pending）：Pod 已被 Kubernetes 系统接受，但有一个或者多个容器镜像尚未创建。包括调度 Pod 的时间和通过网络下载镜像的时间。
* 运行中（Running）：该 Pod 已经绑定到了一个节点上，Pod 中所有的容器都已被创建。至少有一个容器正在运行，或者正处于启动或重启状态。
* 成功（Succeeded）：Pod 中的所有容器都被成功终止，并且不会再重启。
* 失败（Failed）：Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止，容器以非0状态退出或者被系统终止。如果某个Node挂了或失联，对应Pod也会设置成Failed。
* 未知（Unknown）：因为某些原因无法取得 Pod 的状态，通常是因为与 Pod 所在主机通信失败。
* 当一个 Pod 被删除时，会看到 Pod 状态为 `Terminating`（终止）。 这个 `Terminating` 状态并不是 Pod 阶段之一。 Pod 被赋予一个可以优雅终止的期限，默认为 30 秒。 你可以使用 `--force` 参数来[强制终止 Pod](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced)。

## Pod的安全策略

http://docs.kubernetes.org.cn/690.html

```yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  name: permissive
spec:
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  hostPorts:
  - min: 8000
    max: 8080
  volumes:
  - '*'
```

| 常用配置名称             | 用途                     |
| ------------------------ | ------------------------ |
| privileged               | 授权容器的运行           |
| defaultAddCapabilities   | 添加默认的一组能力       |
| requiredDropCapabilities | 去掉某些能力             |
| allowedCapabilities      | 容器能够请求添加某些能力 |
| volumes                  | 控制卷类型的使用         |
| hostNetwork              | 主机网络的使用           |
| hostPorts                | 主机端口的使用           |

## Pod 探针（健康检查）

* ExecAction：容器内执行命令，命令退出时返回码为 0 则认为诊断成功。
* TCPSocketAction：对容器 IP 地址指定端口进行 TCP 检查。如果端口打开，则认为成功。
* HTTPGetAction：对容器url执行 HTTP Get 请求。状态码大于等于200 且小于 400，则认为成功。
* gRPCAction：使用gRPC执行一个远程调用。目标应该实现 [gRPC健康检查](https://grpc.io/grpc/core/md_doc_health-checking.html)。 如果响应的状态是 "SERVING"，则认为诊断成功。 gRPC 探针是一个 Alpha 特性，只有在你启用了 "GRPCContainerProbe" [特性门控](https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/feature-gates/)时才能使用。

每次探测都将获得以下三种结果之一：

* 成功：容器通过诊断。
* 失败：容器未通过诊断。
* 未知：诊断失败，不会采取任何行动。

探针指示策略：

- livenessProbe：指示容器是否正在运行。如果存活探测失败，则杀死容器，并根据重启策略处理。如果容器不提供存活探针，则默认为 Success。
- readinessProbe：指示容器是否准备好提供服务。如果就绪探测失败，端点控制器将从与 Pod 匹配的所有 Service 的端点中删除该 Pod 的 IP 地址。初始延迟之前的就绪状态默认为 Failure。如果容器不提供就绪探针，则默认为 Success。
- startupProbe：指示容器中的应用是否已经启动。如果提供了启动探针，则所有其他探针都会被禁用，直到此探针成功为止。如果启动探测失败，kubelet 将杀死容器， 而容器依其重启策略进行重启。 如果容器没有提供启动探测，则默认状态为 Success。
- 如果容器中进程在遇到问题或不健康的情况下自行崩溃，则不一定需要存活探针，kubelet 将根据 Pod 的restartPolicy 自动执行操作。
- 如果您希望容器在探测失败时被杀死并重新启动，请指定存活探针，并指定restartPolicy 为 Always 或 OnFailure。
- 如果要仅在探测成功时才开始向 Pod 发送流量，请指定就绪探针。Pod 在没有流量情况下启动，并在探针探测成功后才开始接收流量。
- 如果希望容器能够自行维护，可以单独指定就绪探针。
- 如果只想在 Pod 被删除时能够排除请求，则不一定需要就绪探针，删除 Pod 时，会自动将自身置于未完成状态，无论就绪探针是否存在。当等待 Pod 中容器停止时，Pod 仍处于未完成状态。
- 对于所包含的容器需要较长时间才能启动就绪的 Pod 而言，启动探针是有用的，允许使用远远超出存活态时间间隔所允许的时长（initialDelaySeconds + failureThreshold × periodSeconds）。periodSeconds 的默认值是 10 秒。

## Pod 的调度策略

Pod只会调度一次，分派到某个节点后会一直运行到终止。UID是唯一确定一个Pod的。

* 调度 (scheduling)：Pod 匹配到合适的节点。
* 抢占 (Preemption)：终止低优先级的 Pod 以便高优先级的 Pod 可以调度运行。
* 驱逐 (Eviction)：资源匮乏节点上，主动让一个或多个 Pod 失效。

## 镜像拉取策略

imagePullPolicy 字段描述镜像拉取方式

* IfNotPresent：只有当镜像在本地不存在时才会拉取。
* Always：每当 kubelet 启动一个容器时，会判断容器镜像仓库对应摘要，拉取最新镜像。
* Never：Kubelet 不会尝试获取镜像，镜像应已以某种方式存在本地。

* 如果你省略 imagePullPolicy 字段，并且容器镜像的标签是 :latest 或没有指定， imagePullPolicy 会自动设置为 Always，否则为IfNotPresent。
* 多架构镜像通过带上 ```-$(ARCH)``` 标签拉取

## Pod 容器的重启策略

PodSpec 中有一个 restartPolicy 字段。可能值为 Always、OnFailure 和 Never。

* Always：默认值。 只要容器不在运行状态就自动重启。

* OnFailure：只在容器异常时才重启，包含多个容器的Pod，所有容器都异常后，Pod才进入Failure状态。
* Never：从不重启。
* 以五分钟为上限的指数退避延迟（10秒，20秒，40秒…）重新启动，并在成功执行十分钟后重置。

## Pod 的更新策略（Deployment）

通过```.spec.strategy```指定更新策略，`.spec.strategy.type` 可以是 “Recreate” 或 “RollingUpdate”

* Recreate：所有旧Pod先终止，再启动新Pod。
* RollingUpdate比例缩放：默认行为，指定maxUnavailable 和 maxSurge控制滚动过程。
  * maxUnavailable：指定 更新过程中不可用的 Pod 的个数上限。可以是绝对数字或百分比，默认值25%。
  * maxSurge：可选字段，指定可以创建Pod超出数量。可以是绝对数字或百分比，默认值25%。
  * 先缩容maxUnavailable个实例，启动maxUnavailable+maxSurge新实例，再滚动完成所有升级。
  * 优雅升级：
    * 给老Pod发送 SIGTERM 信号，并等待 terminationGracePeriodSeconds(默认为 30 秒)，强杀。
    * 还可以通过prehook，进行终止通知(在发送SIGTERM前调用)，支持http get和execute

```json
"lifecycle": {
    "preStop": {
    	"httpGet": {
            "path": "/shutdown",
            "port": 3000,
            "scheme": "HTTP"
        }
	#or
	"exec": {
            "command": ["/bin/sh", "-c", "sleep 30"]
        }
    }
}
```

.spec.progressDeadlineSeconds：可选字段，允许Deployment更新所用时间，会引发自动回滚。默认值600秒。需要大于 .spec.minReadySeconds。

.spec.minReadySeconds：可选字段，最小等待新Pod启动时间，默认值0。达到时间后，会再进行容器探针。

## Pod 的扩缩容策略（ReplicaSet）

通过 ```.spec.replicas```指定扩缩策略，选择缩容Pod算法：

* 先剔除Pending或不可调度的Pod
* 判断 ```controller.kubernetes.io/pod-deletion-cost``` 注解，默认值0，可以负数，较小的先裁减
* node上副本个数较多的Pod优先裁减。
* 最近创建的Pod优先裁减。
* 随机选择。

## Init 容器

* Init 容器是一种特殊容器，在Pod内的应用容器启动之前运行。
* Init 容器可以包括一些应用镜像中不存在的实用工具和安装脚本。
* Init 容器总是运行到成功完成为止。
* 每个 Init 容器都必须在下一个 Init 容器启动之前成功完成，即串行执行的。
* 如果 Pod 的 Init 容器失败，Kubernetes 会不断地重启该 Pod，直到 Init 容器成功为止。
* 然而，如果 Pod 对应的 restartPolicy 为 Never，它不会重新启动。
* 用途：
    * 包含定制化安装。例如，创建镜像没必要 FROM 另一个镜像，只需要在安装过程中使用类似 sed、 awk、 python 或 dig 这样的工具。
    * 分离创建和部署，准备代码或填入配置文件POD_IP。
    * 能够访问 Secret，而应用程序容器则不能。
    * 必须在应用程序容器启动之前运行完成且阻塞执行，因此能简单实现阻塞。应用程序容器是并行运行的。

```Dockerfile
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

## 临时容器

在Pod启动后，如果需要调试观察，通过临时容器进入。使用api中ephemeralcontainers创建。

## Pod 的网络隔离

通过  ```policyTypes``` 设置，默认是出入都允许的。如果设置包含 ```Egress``` 则根据 NetworkPolicy 的 ```egress``` 列表中的设置来判断。```Ingress``` 同理。

## Pod 的 DNS 策略

* Pod 可以通过 $(ip).$(NameSpace).pod.cluster.local 访问，如：172-17-0-3.default.pod.cluster.local
* 如果通过Service暴露，还可以通过 $(ip).$(ServiceName).$(NameSpace).svc.cluster.local 访问，如：172-17-0-3.test.default.svc.cluster.local
* Pod的 hostname 由  ```metadata.name``` 指定，还可以指定subdomain，则域名为：$(hostname).$(subdomain).$(NameSpace).svc.cluster.local。

```yaml
apiVersion: v1
kind: Pod
spec:
  hostname: busybox-1
  subdomain: default-subdomain
```

上游 DNS 解释使用 ```dnsPolicy``` 进行设置。

* ```Default```：继承节点的名称解析配置。

* ```ClusterFirst```：默认值。与集群域后缀不匹配的查询转发到从节点继承的上游服务器。可配置额外的存根域和上游 DNS 服务器。

* ```ClusterFirstWithHostNet```：对于以 hostNetwork 方式运行的 Pod，应显式设置其 DNS 策略。

  ```yaml
  spec:
    hostNetwork: true
    dnsPolicy: ClusterFirstWithHostNet
  ```

* ```None```：允许 Pod 忽略 Kubernetes 环境中的 DNS 设置。Pod 会使用其 dnsConfig 字段 所提供的 DNS 设置。

  ```yaml
  spec:
  	dnsPolicy: "None"
    dnsConfig:
      nameservers:
        - 1.2.3.4
      searches:
        - ns1.svc.cluster-domain.example
        - my.dns.search.suffix
      options:
        - name: ndots
          value: "2"
        - name: edns0
  # 容器中resolv.conf将会生成如下内容：
  kubectl exec -it dns-example -- cat /etc/resolv.conf
  nameserver 1.2.3.4
  search ns1.svc.cluster-domain.example my.dns.search.suffix
  options ndots:2 edns0
  ```

# Service

## 代理模式

* iptables：只支持轮询
* ipvs：内核空间工作，性能更好，优先使用。支持算法：
  * `rr`：轮替（Round-Robin）
  * `lc`：最少链接数（Least Connection）
  * `dh`：目标地址哈希（Destination Hashing）
  * `sh`：源地址哈希（Source Hashing）
  * `sed`：最短预期延迟（Shortest Expected Delay）
  * `nq`：从不排队（Never Queue）

## 流量策略

流量是只转发到本机的端点还是可以跨node转发。设置 ```spec.externalTrafficPolicy``` 控制外部流量策略，设置 ```spec.internalTrafficPolicy``` 设置内部流量策略。

* ```Cluster```：路由到所有端点，包括其他node的端点。
* ```Local```：本机端点。如果本机异常则不转发。
  * 如果启用了 ```ProxyTerminatingEndpoints``` 特性门控，在 Local Terminating 时可以对外转发，实现优雅下线。

## 环境变量

一个名称为 ```redis-primary``` 的服务，对应变量：注意需要先创建Service，再创建Pod。

```bash
REDIS_PRIMARY_SERVICE_HOST=10.0.0.11
REDIS_PRIMARY_SERVICE_PORT=6379
REDIS_PRIMARY_PORT=tcp://10.0.0.11:6379
REDIS_PRIMARY_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_PRIMARY_PORT_6379_TCP_PROTO=tcp
REDIS_PRIMARY_PORT_6379_TCP_PORT=6379
REDIS_PRIMARY_PORT_6379_TCP_ADDR=10.0.0.11
```

## DNS

* 默认产生 service.namespace 的域名解释
* 支持 DNS SRV 记录查询：如名为http的tcp端口 SRV 查询 ```_http._tcp.my-service.my-ns``` 返回 端口号、http和ip地址。

## 无头服务 Headless Services

不需要负载均衡和单独Service IP的服务。通过 ```.spec.clusterIP``` 指定为 None 实现。

* 带选择符的无头服务：只增加对应Pod的A记录。
* 无选择符的无头服务：
  * 对于 ExternalName 类型服务，查找 CNAME 记录。
  * 对其他服务，查找与 Service 名称相同的 Endpoints 记录。

## 服务类型

通过 ```ServiceTypes``` 指定所需 Service 类型，默认是 ClusterIP。

- `ClusterIP`：通过集群的内部 IP 暴露服务，只能在集群内访问。
- ```NodePort```：通过Node的 IP 和端口暴露服务。NodePort 会路由到对应的 ClusterIP 服务。可以集群外访问。
- ```LoadBalancer```：使用云提供商的负载均衡器向外部暴露服务。 外部负载均衡器可以将流量路由到自动创建的 NodePort 服务和 ClusterIP 服务上。
- ```ExternalName```：通过返回 CNAME 和对应值，可以将服务映射到 externalName 字段的内容（例如：foo.bar.example.com）。 无需创建任何类型代理。

## Ingress

Ingress 不是服务类型，但可以充当集群入口点。可以同IP地址公开多个服务（url路由功能）。

## 客户端亲和性

使用 ```service.spec.sessionAffinity``` 设置：

* ```ClientIP```：根据客户端IP选择对应后端。超时时间 ```service.spec.sessionAffinityConfig.clientIP.timeoutSeconds``` 默认值10800秒
* ```None```：默认值。

# Yaml

## Deployment

```bash
apiVersion: apps/v1					# 使用api
kind: Deployment						# 声明类型
metadata:
  name: nginx-deployment		# 名字
  labels:
  	app: nginx
spec:
	replicas: 2								# 运行Pod数量，ReplicaSet
  selector:									# 选择器
==============================================================
		matchLabels:						# 匹配标签
      app: nginx
——————————————————————————————————————————————————————————————
	  matchExpressions:				# 匹配表达式
  	  - {key: tier, operator: In, values: [cache]}
    	- {key: environment, operator: NotIn, values: [dev]}
==============================================================
  template:									# 部署模版
    metadata:
      labels:								# 标签
        app: nginx
    spec:
      containers:						# 镜像信息
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80	# 服务端口
```

## Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
  annotations:
    imageregistry: "https://hub.docker.com/"
spec:
  containers:
  - name: cuda-test
  	image: "registry.k8s.io/cuda-vector-add:v0.1"
  	command: ['sh', '-c', 'echo xxx && sleep 3600']
	imagePullPolicy: IfNotPresent
  	ports:
  	- containerPort: 80
	args:
        - -param1   #镜像启动需要传入的参数
        - value1
        - -param2
        - value2
  	resources:
  		requests:							# 最小请求的资源数，可以多用
  			cpu: 500m						# 500m = 0.5
  			memory: 500M				# 10进制：G/M/k，2的幂数：Gi/Mi/Ki，注意大小写，500m = 0.5 字节
  		limits:								# 最多占用资源数，不可以超出使用
  			nvidia.com/gpu: 1		# 必须含有 gpu 资源
  			cpu: 2.0
  			ephemeral-storage: 100M # 临时目录限制资源
  			storage: 500M				# 限制存储
	env:
        - name: MY_NODE_NAME_ENV
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName  # 其他可用的值：metadata.name, metadata.namespace, status.podIP, spec.serviceAccountName
        - name: MY_CPU_LIMIT_ENV
          valueFrom:
            resourceFieldRef:
              containerName: <containername>
              resource: limits.cpu # requests.cpu, requests.memory, limits.memory
  initContainers:						# init 容器，等待myservice服务可用再启动
  - name: init-service
  	image: busybox:1.28
	# 也可以直接 nslookup myservice
  	command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  priorityClassName: <priorityClassName>
  nodeSelector:
    accelerator: nvidia-tesla-p100
```

## StatefulSet

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx							# 名为 nginx 的 Headless Service 用来控制网络域名
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web								# 名为 web 的 StatefulSet 有一个 Spec，3 Pod 副本中启动 nginx 容器
spec:
  selector:
    matchLabels:
      app: nginx  				# 必须有，必须匹配 .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 						# 默认值是 1，如果配置HorizontalPodAutoscaler则不要指定
  minReadySeconds: 10 		# 默认值是 0
  template:
    metadata:
      labels:
        app: nginx 				# 必须匹配 .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:		# 通过 PersistentVolume 制备程序准备的 PersistentVolumes 来提供稳定存储
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Delete
```

## DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:      # 不在控制平面节点上部署，如果需要可以删除容忍度设置
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

## Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
	ttlSecondsAfterFinished: 200	# Job完成200秒后，自动清理已完成Job，设置为0完成后立即删除
  backoffLimit: 5								# 失败重试次数
  activeDeadlineSeconds: 100		# 100秒内必须完成，优先级高于 backoffLimit
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
——————————————————————————————————————————————————————————————
				command: ["bash"]				# 触发Failure Job示范
				args:
				- -c
				- echo "Hello world!" && sleep 5 && exit 42
      restartPolicy: Never
  podFailurePolicy:							# 使用该策略，需要设置 .spec.restartPolicy 为 Never
    rules:											# 如果 action 为 Count，代表按默认方式处理，.spec.backoffLimit 计数增加
    - action: FailJob						# 返回 42 判断为整个 Job 失败，返回其他非0值 判断为容器失败
      onExitCodes:
				containerName: pi				# 可选
        operator: In						# In 和 NotIn 二选一
        values: [42]
    - action: Ignore						# 对于状态为 DisruptionTarget 失败的 Pod 采取 Ingore 操作
      onPodConditions:
      - type: DisruptionTarget	# 表示 Pod 失效
```

## CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "* * * * *"					# 同Crontab，分钟 小时 天 月 周，按本地时区
  timeZone: "Etc/UTC"						# 需要启用 CronJobTimeZone 特性门控
  startingDeadlineSeconds: 3600
  concurrencyPolicy: Allow
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.28
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

## Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
	selector:
    app.kubernetes.io/name: MyApp		# 定位到 MyApp Pod
  ports:
    - name: http
    	protocol: TCP
      port: 80
      targetPort: 9376							# MyApp Pod 的 9376 端口，可以是名字，对应Pod 的 .spec.containers.ports.name
    - name: https
    	protocol: TCP
    	port: 443
    	targetPort: 9377
	externalIPs:											# 可以声明外部可以访问到的ip
    - 80.11.12.10
```

### 集群模式

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: clusterIP
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
```

### NodePort模式

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - port: 80
      targetPort: 9376
      nodePort: 30007
```

### 云厂商负载均衡模式

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```

### 云商内部负载均衡

```yaml
[...]
metadata:
    name: my-service
    annotations:
    		<options>        
[...]

<options>
# AWS
service.beta.kubernetes.io/aws-load-balancer-internal: "true"
# GCP
cloud.google.com/load-balancer-type: "Internal"
# Azure
service.beta.kubernetes.io/azure-load-balancer-internal: "true"
# Aliyun
service.beta.kubernetes.io/alibaba-cloud-loadbalancer-address-type: "intranet"
# tencent cloud
service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-xxxxx
```

### AWS

```yaml
metadata:
  name: my-service
  annotations:
  	<something>

# TLS
		service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
		service.beta.kubernetes.io/aws-load-balancer-backend-protocol: (https|http|ssl|tcp)
		service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443,8443"
		service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: "ELBSecurityPolicy-TLS-1-2-2017-01"

# Proxy
		service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"

# log
		service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
    # 发布访问日志的时间间隔。5分钟或60分钟
    service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval: "60"
    service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: "my-bucket"
    service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix: "my-bucket-prefix/prod"

# 连接排空
		service.beta.kubernetes.io/aws-load-balancer-connection-draining-enabled: "true"
		# 最大时间，60秒
    service.beta.kubernetes.io/aws-load-balancer-connection-draining-timeout: "60"

# NLB
		service.beta.kubernetes.io/aws-load-balancer-type: "nlb"

# 属性设置
		service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "60"
		# 跨区负载均衡
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    # 标签
    service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: "environment=prod,owner=devops"
    # 健康次数阈值，默认为 2，必须介于 2 和 10 之间
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: ""
    # 不健康次数阈值，默认为 6，必须介于 2 和 10 之间
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "3"
    # 健康检查间隔秒数，默认为 10，必须介于 5 和 300 之间
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "20"
    # 健康检查超时，默认值为 5，必须介于 2 和 60 之间
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
    # 由已有的安全组所构成的列表，可以配置到所创建的 ELB 之上。
    # 与注解 service.beta.kubernetes.io/aws-load-balancer-extra-security-groups 不同，
    # 重置ELB的安全组。如果多个ELB复用同一个，删除ELB时会删除安全组导致异常
    service.beta.kubernetes.io/aws-load-balancer-security-groups: "sg-53fae93f"
    # 额外附加的安全组，可多个ELB复用
    service.beta.kubernetes.io/aws-load-balancer-extra-security-groups: "sg-53fae93f,sg-42efd82e"
    # 用于选择目标节点
    service.beta.kubernetes.io/aws-load-balancer-target-node-labels: "ingress-gw,gw-name=public-api"
```

### 引用外部服务为Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
---
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service					# 这里的 name 要与 Service 的名字相同
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```

### Service with TLS

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 80
    protocol: TCP
    name: http
  - port: 443
    protocol: TCP
    name: https
  selector:
    run: my-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: nginxsecret
      - name: configmap-volume
        configMap:
          name: nginxconfigmap
      containers:
      - name: nginxhttps
        image: bprashanth/nginxhttps:1.0
        ports:
        - containerPort: 443
        - containerPort: 80
        volumeMounts:
        - mountPath: /etc/nginx/ssl
          name: secret-volume
        - mountPath: /etc/nginx/conf.d
          name: configmap-volume
```

## Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  defaultBackend:
  	service:
  		name: test
  		port:
  			number: 80
  rules:
  - host: "foo.bar.com"
  	http:
      paths:
      - path: /testpath
        pathType: Prefix		# 前缀匹配，区分大小写。如果使用 Exact 是精确匹配
        						# 注意 /foo/bar 匹配 /foo/bar/baz, 但不匹配 /foo/barbaz；url中最后的/可指定可不指定，都能相互匹配。
        backend:
          service:
            name: test
            port:
              number: 80
      - path: /testpath2
      	...
  - host: "*.bar2.com"
  	http:
  		...
  - http:										# 判断前两个host都不是，然后匹配没有host的也可以
  		...
```

### Ingress with static resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-backend
spec:
  defaultBackend:
    resource:											# resource 和 service 是互斥的
      apiGroup: k8s.example.com
      kind: StorageBucket
      name: static-assets
  rules:
    - http:
        paths:
          - path: /icons
            pathType: ImplementationSpecific
            backend:
              resource:
                apiGroup: k8s.example.com
                kind: StorageBucket
                name: icon-assets
```

### Ingress with TLS

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 编码的证书
  tls.key: base64 编码的私钥
type: kubernetes.io/tls
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
      - https-example.foo.com
    secretName: testsecret-tls
  rules:
  - host: https-example.foo.com			# host和证书有关，规则等都要完全匹配
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80
```

## HPA

等价：kubectl autoscale rs frontend --max=10 --min=3 --cpu-percent=50

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    kind: ReplicaSet
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

## NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - ipBlock:
            cidr: 172.17.0.0/16
            except:
              - 172.17.1.0/24
        - namespaceSelector:
            matchLabels:
              project: myproject
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 6379
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
      ports:
        - protocol: TCP
          port: 5978
```

### 允许/拒绝所有出入站流量

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-network-policy
spec:
  podSelector: {}
  policyTypes:
  - Ingress					# 默认拒绝入站流量
  - Egress					# 默认拒绝出站流量
	# 增加下面这句则允许所有入站流量
	ingress:
  - {}
  # 增加下面这句则允许所有出站流量
  egress:
  - {}
```

### 针对端口范围设置

允许名字空间 `default` 中所有带有标签 `role=db` 的 Pod 使用 TCP 协议与 `10.0.0.0/24` 范围内的 IP 通信，目标端口介于 32000 和 32768 之间

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: multi-port-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 32000
      endPort: 32768
```

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:													# 键值对
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

	game.properties: |					# 类文件键
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
immutable: true								# configmap不允许更新
```

## Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: { ... }
  creationTimestamp: 2020-01-22T18:41:56Z
  name: mysecret
  namespace: default
  resourceVersion: "164619"
  uid: cfee02d6-c137-11e5-8d73-42010af00002
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
type: Opaque
```

### Secret 文件引用

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    env:
    	- name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
            optional: false				# mysecret必须存在且包含username键
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      optional: false							# 默认设置，意味着 "mysecret" 必须已经存在
      defaultMode: 0400
      items:											# 如果不指定items，则整个直接映射到目录/etc/foo里，按key生成username和password文件
      - key: username							# 只引用加密项username
      	path: dir/txt							# 生成文件 /etc/foo/dir/txt
```

## LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraint
spec:
  limits:
  - default: # 此处定义默认限制值
      cpu: 500m
    defaultRequest: # 此处定义默认请求值
      cpu: 500m
    max: # max 和 min 定义限制范围
      cpu: "1"
    min:
      cpu: 100m
```



# 常用命令

## Docker命令

### 

### 远程管理

命令行其实是跟 docker daemon 通讯，默认是本地的unix socket，可以使用tcp进行远程管理

* 开启tcp协议

```bash
vi /lib/systemd/system/docker.service
# 找到行 ExecStart=/usr/bin/dockerd 添加参数 -H tcp://0.0.0.0:3272

systemctl daemon-reload
systemctl restart docker
```

* 客户端访问

```bash
docker -H 192.168.1.1:3272 ps
或
export DOCKER_HOST="tcp://192.168.1.1:3272"
docker ps
```

#### 带ssh的pod

```Dockerfile
FROM alpine

RUN apk update && \
    apk add --no-cache openssh tzdata && \ 
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    sed -i "s/#PermitRootLogin.*/PermitRootLogin yes/g" /etc/ssh/sshd_config && \
       ssh-keygen -t dsa -P "" -f /etc/ssh/ssh_host_dsa_key && \
    ssh-keygen -t rsa -P "" -f /etc/ssh/ssh_host_rsa_key && \
    ssh-keygen -t ecdsa -P "" -f /etc/ssh/ssh_host_ecdsa_key && \
    ssh-keygen -t ed25519 -P "" -f /etc/ssh/ssh_host_ed25519_key && \
    echo "root:password" | chpasswd

# 开放22端口
EXPOSE 22
 
# 执行ssh启动命令
CMD ["/usr/sbin/sshd", "-D"]
```

### Docker内容还原


#### undocker

```bash
pip install git+https://github.com/larsks/undocker/ 
docker save IMAGE_NAME | undocker -i -o IMAGE_NAME
```

#### dockerfile-from-image

```bash
https://github.com/CenturyLinkLabs/dockerfile-from-image
docker run -v /var/run/docker.sock:/var/run/docker.sock centurylink/dockerfile-from-image <IMAGE_TAG_OR_ID>
```

#### bash shell

dockerfile2.sh

```bash
#!/bin/bash
#########################################################################
# File Name: dockerfile.sh
# Author: www.linuxea.com
# Version: 1
# Created Time: Thu 14 Feb 2019 10:52:01 AM CST
#########################################################################
case "$OSTYPE" in
 linux*)
 docker history --no-trunc --format "{{.CreatedBy}}" $1 | # extract information from layers
 tac | # reverse the file
 sed 's,^\(|3.*\)\?/bin/\(ba\)\?sh -c,RUN,' | # change /bin/(ba)?sh calls to RUN
 sed 's,^RUN #(nop) *,,' | # remove RUN #(nop) calls for ENV,LABEL...
 sed 's, *&& *, \\\n \&\& ,g' # pretty print multi command lines following Docker best practices
 ;;
 darwin*)
 docker history --no-trunc --format "{{.CreatedBy}}" $1 | # extract information from layers
 tail -r | # reverse the file
 sed -E 's,^(\|3.*)?/bin/(ba)?sh -c,RUN,' | # change /bin/(ba)?sh calls to RUN
 sed 's,^RUN #(nop) *,,' | # remove RUN #(nop) calls for ENV,LABEL...
 sed $'s, *&& *, \\\ \\\n \&\& ,g' # pretty print multi command lines following Docker best practices
 ;;
 *)
 echo "unknown OSTYPE: $OSTYPE"
 ;;
esac
```

```bash
bash dockerfile2.sh marksugar/redis:5.0.0
```

#### docker history

```bash
docker history <image>
```

## 缩放

```bash
# 针对 Dependment 缩放
kubectl scale deployment/nginx-deployment --replicas=10
kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80

# 针对 ReplicaSet 缩放 HPA
kubectl autoscale rs frontend --max=10 --min=3 --cpu-percent=50

# 针对 Statefulset 缩放
kubectl scale statefulset statefulset --replicas=3
```

## Deployment

```bash
# 创建
kubectl create -f nginx-deployment.yaml --record
kubectl create deployment nginx --image nginx

# 应用
kubectl apply -f https://k8s.io/examples/application/deployment.yaml

# 比较差异
kubectl diff -f configs/

# 递归
kubectl diff -R -f configs/
kubectl apply -R -f configs/

# 查看定义
kubectl get deployment nginx-deployment -o yaml

# 查看Deployment
kubectl describe deployments
kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           18s
# 查看对应的ReplicaSet（在更新版本时，可以看到前后两个版本）
kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-75675f5897   3         3         3       18s
# 查看具体部署
kubectl get rs -w
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   2         2         2         2m
nginx-3926361531   2         2         0         6s
nginx-3926361531   2         2         1         18s
nginx-2142116321   1         2         2         2m
nginx-2142116321   1         2         2         2m
# 查看对应每个Pod
kubectl get pods --show-labels
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-75675f5897-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453
nginx-deployment-75675f5897-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=3123191453

# 替换
kubectl replace -f nginx.yaml

# 更新deployment使用的镜像：
kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
# 交互式操作，并将 .spec.template.spec.containers[0].image 从 nginx:1.14.2 更改至 nginx:1.16.1
kubectl edit deployment/nginx-deployment

# 暂停上线
kubectl rollout pause deployment/nginx-deployment
# 可以继续修改，不会立即运行
# 恢复上线
kubectl rollout resume deployment/nginx-deployment
# 查看部署状态，如果上线完成返回值为0
kubectl rollout status deployment/nginx-deployment
echo $?

# 回滚
# 查看历史
kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl apply --filename=https://k8s.io/examples/controllers/nginx-deployment.yaml
2           kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.161
# 为部署添加注释
kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="image updated to 1.16.1
# 查看指定的修订历史
kubectl rollout history deployment/nginx-deployment --revision=2
# 回滚版本
kubectl rollout undo deployment/nginx-deployment
# 回滚到指定版本
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# 删除
kubectl delete -f nginx.yaml -f redis.yaml
```

## Namespace

```bash
# 命令行直接创建
kubectl create namespace new-namespace

# 描述文件创建
cat my-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: new-namespace

kubectl create -f ./my-namespace.yaml

# 运行
kubectl run nginx --image=nginx --namespace=xxx

# 查看
kubectl get namespaces
NAME          STATUS    AGE
default       Active    1d
kube-system   Active    1d

# 操作namespace
kubectl --namespace=xxx run nginx --image=nginx
kubectl --namespace=xxx get pods

# 设置默认操作的namespace
kubectl config set-context $(kubectl config current-context) --namespace=xxx
kubectl config set-context --current --namespace=xxx
# 验证
kubectl config view --minify | grep namespace:

# 查看位于名字空间中的资源
kubectl api-resources --namespaced=true

# 查看不在名字空间中的资源
kubectl api-resources --namespaced=false

# 删除
kubectl delete namespaces new-namespace
```

## Pod

```bash
# 查看
kubectl get pods -l environment=production,tier=frontend
kubectl get pods -l 'environment in (production,qa),tier in (frontend),environment notin (frontend)'
kubectl get pods --field-selector status.phase=Running,metadata.namespace!=default,spec.restartPolicy=Always
kubectl get -f myapp.yaml
kubectl describe -f myapp.yaml

# 查看指定容器日志
kubectl logs myapp-pod -c init-myservice
```

### Pod 安全策略

```bash
psp.yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  name: permissive
spec:
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  hostPorts:
  - min: 8000
    max: 8080
  volumes:
  - '*'

#创建
kubectl create -f ./psp.yaml

#查看
kubectl get psp
NAME        PRIV   CAPS  SELINUX   RUNASUSER         FSGROUP   SUPGROUP  READONLYROOTFS  VOLUMES
permissive  false  []    RunAsAny  RunAsAny          RunAsAny  RunAsAny  false           [*]
privileged  true   []    RunAsAny  RunAsAny          RunAsAny  RunAsAny  false           [*]
restricted  false  []    RunAsAny  MustRunAsNonRoot  RunAsAny  RunAsAny  false           [emptyDir secret downwardAPI configMap persistentVolumeClaim projected]

#修改
kubectl edit psp permissive

#删除
kubectl delete psp permissive
```

## Node

```bash
# 查看
kubectl describe node node-name

# 逐出节点
# 将 node 标记为不可调度，防止新pod调度到该节点，但不会影响任何现有pod，用在节点需要维护重启等。
kubectl cordon node-name
```

## Service

```bash
# 直接给 Deployment 创建 Service
# 注意后建service，pod中的环境变量中将没有service对应变量
kubectl expose deployment/my-nginx
# 等价于
kubectl create -f xxx.yaml
----------------
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
```

## Job

```bash
# 查看job
kubectl describe jobs/pi

# 列出对应的Pod
pods=$(kubectl get pods --selector=job-name=pi --output=jsonpath='{.items[*].metadata.name}')
echo $pods

# 查看Log
kubectl logs podname

# 挂起Job和恢复
kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":true}}'
kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":false}}'
```

## 监控 Heapster

```
#启动proxy
kubectl proxy
#查看heapster运行状态
kubectl get services --namespace=kube-system

#获取信息
#内存使用
curl http://localhost:8001/api/v1/proxy/namespaces/kube-system/services/heapster/api/v1/model/namespaces/mem-example/pods/memory-demo/metrics/memory/usage
#cpu使用
curl http://localhost:8001/api/v1/proxy/namespaces/kube-system/services/heapster/api/v1/model/namespaces/cpu-example/pods/cpu-demo/metrics/cpu/usage_rate

```

## REST Api

```bash
# 删除 ReplicaSet 和它的所有 Pod，可以使用 kubectl delete 或使用API：
kubectl proxy --port=8080
curl -X DELETE  'localhost:8080/apis/apps/v1/namespaces/default/replicasets/frontend' \
  -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
  -H "Content-Type: application/json"
# 只删除 ReplicaSet，可以使用 kubectl delete --cascade=orphan 或API：
kubectl proxy --port=8080
curl -X DELETE  'localhost:8080/apis/apps/v1/namespaces/default/replicasets/frontend' \
  -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
  -H "Content-Type: application/json"

```

## 其他

```bash
# 查看 DNS 信息
kubectl get services kube-dns --namespace=kube-system

# 调试
kubectl run curl --image=radial/busyboxplus:curl -i --tty
nslookup my-nginx
kubectl exec curl-deployment-1515033274-1410r -- curl https://my-nginx --cacert /etc/nginx/ssl/tls.crt

# 代理访问（把名字空间prometheus里的deploy/prometheus-server的8080端口映射到localhost:9090）
kubectl port-forward -n prometheus deploy/prometheus-server 8080:9090

# 创建证书密钥
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/nginx.key -out /tmp/nginx.crt -subj "/CN=my-nginx/O=my-nginx"
kubectl create secret tls nginxsecret --key /tmp/nginx.key --cert /tmp/nginx.crt
# 查看密钥
kubectl get secrets
# 等价于
cat /tmp/nginx.crt | base64
cat /tmp/nginx.key | base64
kubectl apply -f nginxsecrets.yaml
---
apiVersion: "v1"
kind: "Secret"
metadata:
  name: "nginxsecret"
  namespace: "default"
type: kubernetes.io/tls  
data:
  tls.crt: "LS0tLS1CR..."
  tls.key: "LS0tLS1CR..."

# 创建 configmap
kubectl create configmap nginxconfigmap --from-file=default.conf
# 查看 configmap
kubectl get configmaps

# 查看授权信息上下文
kubectl config use-context
kubectl config view
# 配置文件在 $HOME/.kube/config
# 可以配置 proxy-url: https://proxy.host:3128 来使用代理请求
```

# helm

* k8s包管理工具，类似apt、yum，简化k8s应用部署。每个包称为一个Chart，一个Chart是一个目录。
* Tiller 是 Helm 的服务端，部署在 Kubernetes 集群中。Tiller 用于接收 Helm 的请求，并根据 Chart 生成 Kubernetes 的部署文件（ Helm 称为 Release ），然后提交给 Kubernetes 创建应用。Tiller 还提供 Release 的升级、删除、回滚等一系列功能。Release 可以理解为 Helm 使用 Chart 包部署的一个应用实例，可以多次部署不同的Release（实例）。

## 常用命令

```bash
# 查看chart实例列表
helm list
# 查看全部的chart实例(包括删除的)
helm list -a
# 列出已经删除的Release
helm ls --deleted
# 查看指定的chart实例信息
helm status RELEASE_NAME [flags]

# 创建chart实例部署到k8s
helm install chart包 [flags]
# 常用可选参数：
    --name str                # 指定helm实例名称
    --namespace str            # 指定安装的namespace。不指定则默认命名空间
    --version str            # 指定charts版本
    -f values.yaml            # 从文件读取自定义属性集合
    --set "key=value"        # 通过命令方式注入自定义参数，用于覆盖values.yaml定义的值
                              # 对象类型数据可以用 . (点)分割属性名。示例: --set apiApp.requests.cpu=1
                              # 可以使用多个 --set 指定多个参数
helm install --name test --namespace test ./helm-chart

# 删除chart实例（注意：若不加purge参数，则并没有彻底删除）
helm delete RELEASE_NAME [flags]
# 选项：
    --purge        # 彻底删除实例，并释放端口

# 更新chart实例。默认情况下，如果chart实例名不存在，则upgrade会失败
helm upgrade [选项] 实例名称 [chart包目录] [flags]
# 选项：
    -i                        # 实例名存在则更新，不存在时则安装（合并了install和uprade命令功能）
    --set "key=value"        # 通过命令方式注入自定义参数
# 示例：根据chart包更新chart实例
helm upgrade myapp ./myapp
# 不存在则安装，存在则更新
helm upgrade -i myapp ./myapp
# 示例：使用–set参数更新chart实例
helm upgrade --set replicas=2 --set host=www.xxxx.com myapp

# 查看chart实例的版本信息
helm history HELM_NAME [flags]
# 回滚chart实例的指定版本
helm rollback HELM_NAME [REVISION] [flags]

# 在Artifact Hub或自己的hub实例中搜索Helm charts
helm search hub [KEYWORD] [flags]
# 搜索系统上配置的所有仓库，并查找匹配，keyword 接受关键字字符串或者带引号的查询字符串
helm search repo [keyword] [flags]
# 显示指定chart包(目录、文件或URL)中的Charts.yaml文件内容
helm show chart [CHART] [flags]
# 显示指定chart(目录、文件或URL)中的README文内容
helm show readme [CHART] [flags]
# 显示指定chart(目录、文件或URL)包中的values.yaml文件内容
helm show values [CHART] [flags]
# 显示指定chart(目录、文件或URL)中的所有的内容（values.yaml, Charts.yaml, README）
helm show all [CHART] [flags]

# 从包仓库中检索包并下载到本地
helm pull [chart URL | repo/chartname] [...] [flags]
# 下载charts到本地
helm fetch redis

# 自定义 Helm 包
# 创建charts包
helm create NAME [flags]
# 选项：
    -p, --starter string     # 名称或绝对路径。
                              # 如果给定目录路径不存在，Helm会自动创建。
                              # 如果给定目录存在且非空，冲突文件会被覆盖，其他文件会被保留。
# 安装自定义包
helm install --set replicas=2 ./NAME
# 检查chart语法正确性
helm lint myapp
# 打包成一个chart版本包文件。
helm package [CHART_PATH] [...] [flags]
# 查看生成的yaml文件
helm template myapp-1.tgz

#仓库管理
helm repo COMMAND [flags]
从父命令继承的可选参数：
        --debug                       启用详细输出
          --kube-apiserver string       Kubernetes-API服务器的地址和端口
          --kube-as-group stringArray   Kubernetes操作的组，可以重复此标志以指定多个组。
          --kube-as-user string         Kubernetes操作的用户名
          --kube-ca-file string         Kubernetes-API服务器连接的认证文件
          --kube-context string         kubeconfig context的名称
          --kube-token string           用于身份验证的字符串令牌
          --kubeconfig string           kubeconfig文件的路径
  -n,     --namespace string            该请求的作用域的命名空间名称
          --registry-config string      注册表配置文件的路径 (default "~/.config/helm/registry.json")
          --repository-cache string     包含缓存库索引的文件的路径 (default "~/.cache/helm/repository")
          --repository-config string    包含仓库名称和url的文件的路径 (default "~/.config/helm/repositories.yaml")
# 查看chart仓库列表
helm repo list [flags]
# 选项：
    -o, --output format ：打印指定格式的输出。支持的格式：table, json, yaml (default table)
# 从chart仓库中更新本地可用chart的信息
helm repo update [flags]
# 读取当前目录，并根据找到的chart为chart仓库生成索引文件(index.yaml)。
helm repo index
# 选项
    --merge string   # 合并生成的索引到已经存在的索引文件
    --url string     # chart仓库的绝对URL
# 添加chart仓库
helm repo add [NAME] [URL] [flags]
helm repo add myharbor https://harbor.qing.cn/chartrepo/charts --username admin --password password
# 删除一个或多个仓库
helm repo remove [REPO1 [REPO2 ...]] [flags]

# 列举所有的chart中声明的依赖
helm dependency list
# 列举指定chart的依赖
helm dependency list CHART [flags]
# 查看helm版本
helm version
# 打印所有Helm使用的客户端环境信息
helm env [flags]

```

## 自定义 Helm 包例子

需要包含 deployment、service、ingress，运行 helm create 时自动生成：

* deployment.yaml

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: {{ .Release.Name }}  #deployment应用名
  labels:
    app: {{ .Release.Name }}     #deployment应用标签定义
spec:
  replicas: {{ .Values.replicas}}    #pod副本数
  selector:
    matchLabels:
      app: {{ .Release.Name }}       #pod选择器标签
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}     #pod标签定义
    spec:
      containers:
        - name: {{ .Release.Name }}           #容器名
          image: {{ .Values.image }}:{{ .Values.imageTag }}    #镜像地址
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
```

* service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-svc     #服务名
spec:
  selector:     #pod选择器定义
    app: {{ .Release.Name }}
  ports:
  - protocol: TCP 
    port: 80
    targetPort: 80
```

* ingress.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress     #ingress应用名
spec:
  rules:
    - host: {{ .Values.host }}      #域名
      http:
        paths: 
          - path: /  
            backend: 
              serviceName: {{ .Release.Name }}-svc     #服务名
              servicePort: 80
```

* values.yaml：变量定义，通过 helm install 时通过 --set 传入，或通过 -f(--values) yaml文件 传入

```yaml
#域名
host: www.XXX.com

#镜像参数
image: XXXXXXXXXXXXXXXXXX
imageTag: 1.7.9
 
#pod 副本数
replicas:1
```

## 常用 Helm

* prometheus

```bash
kubectl create namespace prometheus
# add prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2"
# webserver listen at:
http://prometheus-server.prometheus.svc.cluster.local/
kubectl port-forward -n prometheus deploy/prometheus-server 8080:9090
```

* grafana

```bash
cat << EoF > ./grafana.yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.prometheus.svc.cluster.local
      access: proxy
      isDefault: true
EoF

kubectl create namespace grafana

# add grafana Helm repo
helm repo add grafana https://grafana.github.io/helm-charts

helm install grafana grafana/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set persistence.enabled=true \
    --set adminPassword='mypassword' \
    --values ./grafana.yaml \
    --set service.type=LoadBalancer

# access web
export ELB=$(kubectl get svc -n grafana grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "http://$ELB"

# password
kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

```

# 疑难杂症

* DNS解释超时问题：https://monkeywie.cn/2019/12/10/k8s-dns-lookup-timeout/

