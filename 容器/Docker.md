
# Docker内容还原方法

## undocker

```bash
pip install git+https://github.com/larsks/undocker/ 
docker save IMAGE_NAME | undocker -i -o IMAGE_NAME
```

## dockerfile-from-image

```bash
https://github.com/CenturyLinkLabs/dockerfile-from-image
docker run -v /var/run/docker.sock:/var/run/docker.sock centurylink/dockerfile-from-image <IMAGE_TAG_OR_ID>
```

## shell

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

## docker history

```bash
docker history <image>
```

## Yaml

### Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <podname>
  labels:
    app: myapp
spec:
  containers:
  - name: <containername>
    image: <imageurl>
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
    resources:
      limits:
        cpu: "1"
        memory: "200Mi" #限制Pod最多使用的内存
      requests:
        cpu: "500m" #m表示千分之一，0.5 = 500m
        memory: "100Mi" #申请Pod启动成功最少有多少内存
    ports:
    - containerPort: 80
    args:
    - -param1   #镜像启动需要传入的参数
    - value1
    - -param2
    - value2
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
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  priorityClassName: <priorityClassName>
```

### ReplicaSet

```yaml
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 80
```

### Horizontal Pod Autoscaler(HPA)

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

### PriorityClass

1.8之后才有优先级和抢占，需要开启

```yaml
apiVersion: v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```

## kubectl

```bash
#创建命名空间
kubectl create namespace <nsname>
#根据文件创建资源
kubectl create -f myapp.yaml [--namespace=<nsname>]

#获取资源情况
kubectl get -f myapp.yaml
kubectl get services [--namespace=<nsname>]
kubectl get pod <podname> [--namespace=<nsname>] [--output=yaml]

#查询部署状态
kubectl describe -f myapp.yaml
#查询nodes
kubectl describe nodes

#查看日志
kubectl logs <podname> -c <containername>

#删除
kubectl delete pod <podname> [--namespace=<nsname>]
kubectl delete namespace <nsname>

#shell
kubectl exec -it <podname> -- sh

#停止节点调度，不接受新pod，正在运行的pod不影响
kubectl cordon <nodename>
```

## pod

* 使用 Job 运行预期会终止的 Pod，例如批量计算。
* 对预期不会终止的 Pod 使用 ReplicationController、ReplicaSet 和 Deployment ，例如 Web 服务器。
* 提供特定于机器的系统服务，使用 DaemonSet 为每台机器运行一个Pod。

### Init容器

* Init 容器总是运行到成功完成为止。
* 每个 Init 容器都必须在下一个 Init 容器启动之前成功完成。
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

### 获取pod状态

* 使用Heapster

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

### 服务等级

* 通过kubectl get pod可以获取到pod的服务等级qosClass
* Guaranteed：cpu和内存requests和limits一样
* Burstable：cpu和内存requests小于limits
* BestEffort：pod里所有容器cpu和内存都没有要求

## ReplicaSet

* ReplicaSet能确保运行指定数量的pod。ReplicaSet（RS）是Replication Controller（RC）的升级版本。唯一区别是选择器支持，RS支持labels user guide中set-based选择器，而RC仅支持equality-based选择器。
* 虽然ReplicaSets可以独立使用，但它主要被 Deployments 使用
* RS唯一不支持RC的rolling-update命令，请使用Deployments实现。
* ReplicaSet可以由HPA自动伸缩。