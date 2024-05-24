# 高级调度计划任务

## 1. Job

###1.1 job介绍

Job会创建一个或者多个 Pod，并将继续重试 Pod 的执行，直到指定数量的 Pod 成功终止。 随着 Pod 成功结束，Job跟踪记录成功完成的 Pod 个数。 当数量达到指定的成功个数阈值时，任务（即 Job）结束。 删除 Job的操作会清除所创建的全部 Pod。 挂起 Job的操作会删除 Job的所有活跃 Pod，直到 Job被再次恢复执行。

### 1.2 job使用

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: echo-job
spec:
  #suspend: true
  backoffLimit: 3
  completions: 2
  parallelism: 2
  template:
    spec:
      containers:
      - name: echo
        image: 192.168.0.203:8089/mark/busybox
        command: ["sh"]
        args:
        - -c
        - echo "Hello world!" && sleep 3 && exit 1 #模拟失败
      restartPolicy: Never
      imagePullSecrets:
      - name: docker-harbor
```

参数详解：

* suspend: true 代表job挂起 1.21版本后支持。
* backoffLimit 代表任务执行失败，失败多少次后不再执行。
* completions 代表执行的任务的pod数量。
* parallelism 代表执行任务的并发数，和completions联动，如果completions大于parallelism，会分批次按并发数执行。
* ttlSecondsAfterFinished 代表自动清理已完成 Job（状态为 `Complete` 或 `Failed`），0表示结束后立即清理。

## 2. Cronjob

### 2.1 Cronjob介绍

CronJob用于执行排期操作，例如备份、生成报告等。 一个 CronJob对象就像 Unix 系统上的 **crontab**（cron table）文件中的一行。 它用 [Cron](https://zh.wikipedia.org/wiki/Cron) 格式进行编写， 并周期性地在给定的调度时间执行 Job。

### 2.2 CronJob使用

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  startingDeadlineSeconds: 200
  concurrencyPolicy: Allow
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: 192.168.0.203:8089/mark/busybox
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
          imagePullSecrets:
          - name: docker-harbor
```

参数详解:

* schedule: job的调度周期，按分时日月周来调度
* startingDeadlineSeconds：表示启动job的期限，当job因为某种原因过了调度时间，设置个截止时间，比如200秒，在200秒之内创建job，如果未在200s内创建job的话，则认为job失败，继续下一次的调度。
* concurrencyPolicy：配置并发规则
  - `Allow`（默认）：CronJob允许并发 Job执行。
  - `Forbid`：CronJob不允许并发执行；老Job未执行完成或者失败，新的Job不会创建。
  - `Replace`：如果新 Job的执行时间到了而老 Job没有执行完，CronJob会用新 Job替换当前正在运行的 Job。

* successfulJobsHistoryLimit: 正常完成的job保留数量。
* failedJobsHistoryLimit：失败的Job保留数量。待排查。。。。。。。
* suspend：是否可以暂停Job创建，true暂停。



## 3. 初始化容器

### 3.1 初始化容器优点

* Init容器可以安装过程中应用容器中不存在的实用工具或个性化代码

* Init容器可以运行root,执行一些高权限命令

* Init执行完后立即退出



### 3.2 初始化容器和PostStart区别

PostStart: 依赖主应用环境，且不一定先与command运行。

InitContainer: 不依赖主应用环境，可以有更高的权限和更多的工具，一定会在主应用启动之前完成。

**Init 容器与普通的容器非常像，除了如下两点：**

- 它们总是运行到完成。
- 每个都必须在下一个启动之前成功完成。

如果 Pod 的 Init 容器失败，kubelet 会不断地重启该 Init 容器直到该容器成功为止。 然而，如果 Pod 对应的 `restartPolicy` 值为 "Never"，并且 Pod 的 Init 容器失败， 则 Kubernetes 会将整个 Pod 状态设置为失败。

### 3.3 InitContainer配置

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app.kubernetes.io/name: MyApp
spec:
  volumes:
  - name: init-vo
    emptyDir: {}
  containers:
  - name: myapp-container
    image: 192.168.0.203:8089/mark/busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 36']
    volumeMounts:
    - mountPath: /tmp
      name: init-vo
  initContainers:
  - name: init-myservice
    image: 192.168.0.203:8089/mark/busybox
    command: ['sh', '-c', 'mkdir /tmp/filexx && sleep 2']
    volumeMounts:
    - mountPath: /tmp
      name: init-vo
  - name: init-mydb
    image: 192.168.0.203:8089/mark/busybox
    command: ['sh', '-c', 'echo init-mydb  is running! && sleep 36']
  imagePullSecrets:
      - name: docker-harbor
```

**排查init失败命令**

```shell
kubectl  describe po myapp-pod
kubectl  logs  myapp-pod -c init-myservice
```

**pod创建过程**

![image-20240415103352517](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240415103352517.png)



## 4. 临时容器

使用临时容器，执行root或者bash、sh等高危命令，可以使用临时容器使用debug等工具调试等。1.16+

当由于容器崩溃或容器镜像不包含调试工具而导致 `kubectl exec` 无用时， 临时容器对于交互式故障排查很有用。

### 4.1 开启临时容器

```shell
二进制：1.25版本后不需要开启
kube-apiserver
vim /usr/lib/systemd/system/kube-apiserver.service
--feature-gates=EphemeralContainers=true

kubelet
vim /etc/kubernetes/kubelet-conf.yml
featureGates:
  EphemeralContainers: true
  
systemctl daemon-reload
systemctl restart kube-apiserver
systemctl restart kubelet
```



### 4.2 临时容器使用

```shell
# kubectl debug coredns-867d46bfc6-tkwfs  -it --image=192.168.0.203:8089/tools/debug-tools  -nkube-system
 
 
 
# kubectl  debug node/k8s-master02 -it --image=192.168.0.203:8089/tools/debug-tools
```



# 污点和容忍度

**污点（Taint）** 则相反——它使节点能够排斥一类特定的 Pod。

**容忍度（Toleration）** 是应用于 Pod 上的。容忍度允许调度器调度带有对应污点的 Pod。 容忍度允许调度但并不保证调度：作为其功能的一部分， 调度器也会[评估其他参数](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/pod-priority-preemption/)。

污点和容忍度（Toleration）相互配合，可以用来避免 Pod 被分配到不合适的节点上。 每个节点上都可以应用一个或多个污点，这表示对于那些不能容忍这些污点的 Pod， 是不会被该节点接受的。

### 1. 污点Taint

```shell
kubectl taint nodes node1 key1=value1:NoSchedule
```

- `NoExecute`

  这会影响已在节点上运行的 Pod，具体影响如下：如果 Pod 不能容忍这类污点，会马上被驱逐。如果 Pod 能够容忍这类污点，但是在容忍度定义中没有指定 `tolerationSeconds`， 则 Pod 还会一直在这个节点上运行。如果 Pod 能够容忍这类污点，而且指定了 `tolerationSeconds`， 则 Pod 还能在这个节点上继续运行这个指定的时间长度。 这段时间过去后，节点生命周期控制器从节点驱除这些 Pod。

- `NoSchedule`

  除非具有匹配的容忍度规约，否则新的 Pod 不会被调度到带有污点的节点上。 当前正在节点上运行的 Pod **不会**被驱逐。

- `PreferNoSchedule`

  `PreferNoSchedule` 是“偏好”或“软性”的 `NoSchedule`。 控制平面将**尝试**避免将不能容忍污点的 Pod 调度到的节点上，但不能保证完全避免。

### 2. 容忍度toleration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: 192.168.0.203:8089/mark/nginx:latest
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
  imagePullSecrets:
  - name: docker-harbor
```

`operator` 的默认值是 `Equal`。

一个容忍度和一个污点相“匹配”是指它们有一样的键名和效果，并且：

- 如果 `operator` 是 `Exists`（此时容忍度不能指定 `value`），或者
- 如果 `operator` 是 `Equal`，则它们的值应该相等。

4种匹配模式

* 完全匹配

```yaml
  tolerations:
  - key: "example-key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

* 不完全匹配

```yaml
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
```

* 模糊匹配

```yaml
  tolerations:
  - key: "example-key"
    operator: "Exists"
```

* 匹配所有

```yaml
  tolerations:
  - operator: "Exists"
```

### 3. 内置污点

- `node.kubernetes.io/memory-pressure`:  节点内存存在压力
- `node.kubernetes.io/disk-pressure`:  节点磁盘存在压力
- `node.kubernetes.io/pid-pressure`: （1.14 或更高版本）
- `node.kubernetes.io/unschedulable`: （1.10 或更高版本）节点不可调度
- `node.kubernetes.io/network-unavailable`: （**只适合主机网络配置**）
- `node.kubernetes.io/not-ready`: 节点未准备好
- `node.kubernetes.io/unreachable`: Node Controller访问不到节点
- `node.kubernetes.ioout-of-disk`: 节点磁盘耗尽

#### 3.1 tolerationSeconds实现节点宕机应用秒级恢复

节点不可用时配合tolerationSeconds来实现漂移到其他可用节点。

 **PS: tolerationSeconds不要随意使用，会造成循环创建和删除资源的场景出现。 **

```yaml
  
  tolerations:
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 10 #认300s
```

节点定义不可用时间可以从/usr/lib/systemd/system/kube-controller-manager.service文件的 --node-monitor-grace-period=40s来看。

![image-20240415221659205](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240415221659205.png)

![image-20240415221711140](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240415221711140.png)

### 4. 污点常用命令

* 创建污点

```shell
kubectl taint nodes k8s-master02 ssd=true:NoSchedule
```

* 修改污点(key和effect相同)

```shell
kubectl taint nodes k8s-master02 ssd=false:NoSchedule
```

* 删除污点

```shell
kubectl taint nodes k8s-master02 ssd=false:NoSchedule- #精准删除
kubectl taint nodes k8s-master02 ssd- #匹配的范围较大
```

* 查看污点

```
kubectl describe node k8s-master02 |grep Tanits -A 10
```



# 亲和性Affinity

`亲和性主要作用：`

* pod调度到哪个节点
* pod和pod是否调度到不同节点

### 1. 亲和性类型

| 类型                                                      | 描述                                                         |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| requiredDuringSchedulingIgnoredDuringExecution（硬亲和）  | 调度器只有在规则被满足的时候才能执行调度。此功能类似于 `nodeSelector`， 但其语法表达能力更强。 |
| preferredDuringSchedulingIgnoredDuringExecution（软亲和） | 调度器会尝试寻找满足对应规则的节点。如果找不到匹配的节点，调度器仍然会调度该 Pod。 |

### 2. 节点亲和性

节点亲和性概念上类似于 `nodeSelector`， 它使你可以根据节点上的标签来约束 Pod 可以调度到哪些节点上。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-nodea
  name: nginx-nodea
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-nd
  template:
    metadata:
      labels:
        app: nginx-nd
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution: #必须满足
            nodeSelectorTerms: //多个matchExpressions，满足一个就行
            - matchExpressions: //需要同时满足
              - key: ssd
                operator: In
                values:
                - "true"
          preferredDuringSchedulingIgnoredDuringExecution: #尽量满足
          - weight: 100 #根据权重大小来设置优先级1-100
            preference:
              matchExpressions:
              - key: ssd
                operator: NotIn
                values:
                - 'true'
      containers:
      - image: 192.168.0.203:8089/mark/nginx:1.15.12-alpine
        name: nginx-web
      imagePullSecrets:
      - name: docker-harbor
```

`operator` 字段来为 Kubernetes 设置在解释规则时要使用的逻辑操作符。 你可以使用 `In`、`NotIn`、`Exists`、`DoesNotExist`、`Gt` 和 `Lt` 之一作为操作符。

### 3. Pod 间亲和性与反亲和性

Pod 间亲和性与反亲和性使你可以基于已经在节点上运行的`Pod的标签`来约束 Pod 可以调度到的节点，而不是基于节点上的标签。

与[节点亲和性](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity)类似，Pod 的亲和性与反亲和性也有两种类型：

- `requiredDuringSchedulingIgnoredDuringExecution`
- `preferredDuringSchedulingIgnoredDuringExecution`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-nodea
  name: nginx-nodea
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-nd
  template:
    metadata:
      labels:
        app: nginx-nd
    spec:
      affinity:
        podAffinity: #pod亲和性：pod标签为security=S1部署到同一节点且zone还得一样
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: security
                operator: In
                values:
                - S1
              topologyKey: topology.kubernetes.io/zone
        podAntiAffinity: #pod反亲和性，pod标签为security=S2node上且region不同
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: security
                  operator: In
                  values:
                  - S2
          topologyKey: region
      containers:
      - image: 192.168.0.203:8089/mark/nginx:1.15.12-alpine
        name: nginx-web
      imagePullSecrets:
      - name: docker-harbor
```

**拓扑域topology？**

节点上key=value值相同的属于同一拓扑域，可以用来做跨机房。跨机柜、跨区域等容灾的部署。

`podAffinity`：topologyKey：region 表示同一个region拓扑域的节点

`podAntiAffinity`：topologyKey：region 表示不同region拓扑域的节点

![image-20240416220748128](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240416220748128.png)

### 4. pod间亲和性和反亲和性案例

#### 4.1 要求pod多副本部署到不同节点且不同region的拓扑域

* 设置k8s-master01 k8s-master02 k8s-master013的拓扑域

  k8s-master01 k8s-master02 为同一拓扑域，region=chuzhou

  k8s-master03 region=hefei

![image-20240416213454418](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240416213454418.png)

* 定义deployment 副本为2并按要求配置反亲和性

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-pod
  name: nginx-pod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - 'nginx-pod'
            topologyKey: region
      containers:
      - image: 192.168.0.203:8089/mark/nginx:1.15.12-alpine
        name: nginx-web
      imagePullSecrets:
      - name: docker-harbor
```

![image-20240416215124976](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240416215124976.png)

#### 4.2 部署和app=2标签pod的同节点

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-pod
  name: nginx-pod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - '2'
            topologyKey: region
      containers:
      - image: 192.168.0.203:8089/mark/nginx:1.15.12-alpine
        name: nginx-web
      imagePullSecrets:
      - name: docker-harbor
```



# 高级调度准入控制

### 1. 资源配额

资源配额，通过 `ResourceQuota` 对象来定义，对每个命名空间的资源消耗总量提供限制。 它可以限制命名空间中某种类型的对象的总数目上限，也可以限制命名空间中的 Pod 可以使用的计算资源的总上限。

资源配额的工作方式如下：

- 不同的团队可以在不同的命名空间下工作。这可以通过 [RBAC](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/) 强制执行。
- 集群管理员可以为每个命名空间创建一个或多个 ResourceQuota 对象。
- 当用户在命名空间下创建资源（如 Pod、Service 等）时，Kubernetes 的配额系统会跟踪集群的资源使用情况， 以确保使用的资源用量不超过 ResourceQuota 中定义的硬性资源限额。
- 如果资源创建或者更新请求违反了配额约束，那么该请求会报错（HTTP 403 FORBIDDEN）， 并在消息中给出有可能违反的约束。
- 如果命名空间下的计算资源 （如 `cpu` 和 `memory`）的配额被启用， 则用户必须为这些资源设定请求值（request）和约束值（limit），否则配额系统将拒绝 Pod 的创建。 提示: 可使用 `LimitRanger` 准入控制器来为没有设置计算资源需求的 Pod 设置默认值。

#### 1.1 计算资源配额

配额机制所支持的资源类型：

| 资源名称                 | 描述                                                         |
| ------------------------ | ------------------------------------------------------------ |
| `limits.cpu`             | 所有非终止状态的 Pod，其 CPU 限额总量不能超过该值。          |
| `limits.memory`          | 所有非终止状态的 Pod，其内存限额总量不能超过该值。           |
| `requests.cpu`           | 所有非终止状态的 Pod，其 CPU 需求总量不能超过该值。          |
| `requests.memory`        | 所有非终止状态的 Pod，其内存需求总量不能超过该值。           |
| `hugepages-<size>`       | 对于所有非终止状态的 Pod，针对指定尺寸的巨页请求总数不能超过此值。 |
| `cpu`                    | 与 `requests.cpu` 相同。                                     |
| `memory`                 | 与 `requests.memory` 相同。                                  |
| `configmaps`             | 在该命名空间中允许存在的 ConfigMap 总数上限。                |
| `persistentvolumeclaims` | 在该命名空间中允许存在的 [PVC](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) 的总数上限。 |
| `pods`                   | 在该命名空间中允许存在的非终止状态的 Pod 总数上限。Pod 终止状态等价于 Pod 的 `.status.phase in (Failed, Succeeded)` 为真。 |
| `replicationcontrollers` | 在该命名空间中允许存在的 ReplicationController 总数上限。    |
| `resourcequotas`         | 在该命名空间中允许存在的 ResourceQuota 总数上限。            |
| `services`               | 在该命名空间中允许存在的 Service 总数上限。                  |
| `services.loadbalancers` | 在该命名空间中允许存在的 LoadBalancer 类型的 Service 总数上限。 |
| `services.nodeports`     | 在该命名空间中允许存在的 NodePort 类型的 Service 总数上限。  |
| `requests.storage`       | 所有 PVC，存储资源的需求总量不能超过该值。                   |



#### 1.2 ResourceQuota配置

如何根据k8s集群的总资源来划分每个ns的ResourceQuota配比

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pods-medium
  namespace: rq-test
spec:
  hard:
    cpu: "10"
    memory: 20Gi
    pods: "2"
    requests.storage: "4Gi"
```



[root@k8s-master01 prd]# kubectl get ResourceQuota  -n rq-test
NAME          AGE   REQUEST     LIMIT
pods-medium   31s   pods: 0/2

`deployment副本超过ResourceQuota限制的pods数量报错`

```shell
[root@k8s-master01 prd]# kubectl describe rs nginx-deploy-1-6bbdbb97c6 -n rq-test

  Warning  FailedCreate      7s (x5 over 46s)  replicaset-controller  (combined from similar events): Error creating: pods "nginx-deploy-1-6bbdbb97c6-zttcw" is forbidden: exceeded quota: pods-medium, requested: pods=1, used: pods=2, limited: pods=2
```

### 2. LimitRange(限制范围)

LimitRange 是限制`命名空间`内可为每个适用的对象类别 （例如 Pod 或 [PersistentVolumeClaim](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)） 指定的资源分配量（限制和请求）的策略对象。

一个 **LimitRange（限制范围）** 对象提供的限制能够做到：

- 在一个命名空间中实施对每个 Pod 或 Container 最小和最大的资源使用量的限制。
- 在一个命名空间中实施对每个 [PersistentVolumeClaim](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) 能申请的最小和最大的存储空间大小的限制。
- 在一个命名空间中实施对一种资源的申请值和限制值的比值的控制。
- 设置一个命名空间中对计算资源的默认申请/限制值，并且自动的在运行时注入到多个 Container 中。

#### 2.1 request和limit限制范围

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraint
  namespace: rq-test
spec:
  limits:
  - default: # 此处定义默认限制值limit
      cpu: 500m
    defaultRequest: # 此处定义默认请求值 requests
      cpu: 500m
    max: # max 和 min 定义限制范围
      cpu: "1"
    min:
      cpu: 100m
    type: Container
```

* default: 默认配置pod的limit

* defaultRequest: 默认配置pod的 requests
* max: 限制不能大于max设置的值
* min: 限制不能小于min设置的值

#### 2.2 限制存储空间

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: storagelimits
  namespace: rq-test
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 3Gi
    min:
      storage: 2Gi
```

`如果存储pvc申请容量超过limitrange最大size则报错，反之也会报错。`
Error from server (Forbidden): error when creating "rq-test-nfs-pvc.yaml": persistentvolumeclaims "rqtest-nfs-pvc" is forbidden: `maximum storage usage per PersistentVolumeClaim is 3Gi, but request is 4Gi`

-----
Error from server (Forbidden): error when creating "rq-test-nfs-pvc.yaml": persistentvolumeclaims "rqtest-nfs-pvc" is forbidden: `minimum storage usage per PersistentVolumeClaim is 2Gi, but request is 1Gi`

### 3. 服务质量Qos

配置 Pod 以让其归属于特定的 [服务质量类（Quality of Service class，QoS class）](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-qos/). Kubernetes 在 **Node 资源不足**时使用 QoS 类来就驱逐 Pod 作出决定。

- Guaranteed: `最高服务质量`pod的resources中limit和requests cpu和memory值一致
- Burstable： `中服务质量`Pod 不符合 `Guaranteed` QoS 类的标准。Pod 中至少一个 Container 具有内存或 CPU 的请求或限制。
- BestEffort： `最底服务质量`Pod 中的 Container 必须没有设置内存和 CPU 限制或请求。



# 细粒度权限控制(RBAC)

基于角色（Role）的访问控制（RBAC）是一种基于组织中用户的角色来调节控制对计算机或网络资源的访问的方法。

RBAC 鉴权机制使用 `rbac.authorization.k8s.io` [API 组](https://kubernetes.io/zh-cn/docs/concepts/overview/kubernetes-api/#api-groups-and-versioning)来驱动鉴权决定， 允许你通过 Kubernetes API 动态配置策略。

## 1. Role和ClusterRole区别

`Role`: 为某一命令空间下设置访问的权限

`ClusterRole`：整个k8s集群设置访问权限

角色绑定（Role Binding）是将角色中定义的权限赋予一个或者一组用户。 它包含若干**主体（Subject）**（用户、组或服务账户）的列表和对这些主体所获得的角色的引用。 RoleBinding 在指定的名字空间中执行授权，而 ClusterRoleBinding 在集群范围执行授权。

### 1.1 权限绑定逻辑图

<img src="C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240417220207543.png" alt="image-20240417220207543"  />

## 2. 权限及绑定

### 2.0 Role配置字段解析

**resources**: ["pods"] #所有的资源必须写复数

* `resources`: ["pods","pods/log", "pods/exec"] #要允许某主体读取 `pods` 同时访问这些 Pod 的 `log` 子资源

* `resourceNames`：按名称指定具体资源

**apiGroups**： 支持两类API Groups，可以` kubectl api-resources` 查看

| API Groups            | 资源                                                         |
| --------------------- | ------------------------------------------------------------ |
| Core Groups（核心组） | Container<br/>Pod<br/>ReplicationController<br/>Endpoint<br/>Service<br/>ConfigMap<br/>Secret<br/>Volume<br/>PersistentVolumeClaim<br/>Event<br/>LimitRange<br/>PodTemplate<br/>Binding<br/>ComponentStatus<br/>Namespace<br/>Node<br/> |
| 具有分组信息的 API    | `apps/v1`<br/>DaemonSet<br/>Deployment<br/>StatefulSet<br/>ReplicaSet <br/>`batch/v1`   <br/>Job<br/>`batch/v1beta`<br/>CronJob |

**verbs**: ["get", "list", "watch", "create", "update", "patch", "delete"]

### 2.1 Role

定义一个rq-test命名空间下的pod只读权限

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rq-test
  name: pod-readonly
rules:
- apiGroups: [""] # "" 标明 core API 组
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

#### 2.1 1 RoleBinding 

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定允许 "jane" 读取 "rq-test" 名字空间中的 Pod
# 你需要在该名字空间中有一个名为 “ pod-readonly” 的 Role
kind: RoleBinding
metadata:
  name: read-pods
  namespace: rq-test
subjects:
# 你可以指定不止一个“subject（主体）”
- kind: User
  name: jane # "name" 是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" 指定与某 Role 或 ClusterRole 的绑定关系
  kind: Role        # 此字段必须是 Role 或 ClusterRole
  name: pod-readonly  # 此字段必须与你要绑定的 Role 或 ClusterRole 的名称匹配
  apiGroup: rbac.authorization.k8s.io
```

```shell
kubectl create rolebinding read-pods --role=pod-readonly --user=kube-user:jane --namespace=rq-test
```

jane用户绑定了rq-test命名空间下的pod-readonly所具备的权限，该命名空间下pod的只读权限。

#### 2.1.2 RoleBinding绑定ClusterRole权限

```shell
kubectl create rolebinding myappnamespace-myapp-view-binding --clusterrole=view --serviceaccount=myappnamespace:myapp --namespace=acme
```



### 2.2 ClusterRole 

定义为k8s集群通用的pod只读权限

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole 
metadata:
  name: secret-reader
rules:
- apiGroups: [""] # "" 标明 core API 组
  resources: ["secret"]
  verbs: ["get", "watch", "list"]
```

#### 2.2.1 ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# 此集群角色绑定允许 “manager” 组中的任何人访问任何名字空间中的 Secret 资源
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager      # 'name' 是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

```shell
kubectl create clusterrolebinding myapp-view-binding --clusterrole=secret-reader --serviceaccount=kube-user:manager
```

manager组内用户绑定了secret-reader的clusterrole的权限

#### 2.2.2  ClusterRole聚合

将若干 ClusterRole **聚合（Aggregate）** 起来，形成一个复合的 ClusterRole。 作为集群控制面的一部分，控制器会监视带有 `aggregationRule` 的 ClusterRole 对象集合。`aggregationRule` 为控制器定义一个标签[选择算符](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/)供后者匹配应该组合到当前 ClusterRole 的 `roles` 字段中的 ClusterRole 对象。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: [] # 控制面自动填充这里的规则
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
# 当你创建 "monitoring-endpoints" ClusterRole 时，
# 下面的规则会被添加到 "monitoring" ClusterRole 中
rules:
- apiGroups: [""]
  resources: ["services", "endpointslices", "pods"]
  verbs: ["get", "list", "watch"
```



## 3. RBAC企业实践



![image-20240417223535089](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240417223535089.png)

### 3.1 配置通用ClusterRole

* NameSpace的只读权限

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-readonly-clusterrole
rules:
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "watch", "list"]
```



* pod非只读权限

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pods-edit-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "update", "patch", "delete"]
```



* pod只读

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name:pods-readonly-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```



* 工作负载资源的只读权限

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name:workload-readonly-clusterrole
rules:
- apiGroups: ["apps"]
  resources: ["deployments","statefulsets","daemonsets","cronjob"]
  verbs: ["get", "watch", "list"]
```



### 3.2 指定ns下创建ServiceAccout

* 创建ns

```shell
# kubectl  create namespace kube-user
```

* 创建mark的sa

```shell
# kubectl  create serviceaccount mark -nkube-user
serviceaccount/yueyue created
```

* 创建yueyue的sa

```shell
# kubectl  create serviceaccount yueyue -nkube-user
serviceaccount/yueyue created
```



## 3.3 RoleBinding绑定

* 将kube-user下的用户绑定全局namespace的只读权限

```shell
kubectl create clusterrolebinding read-namespace --clusterrole=namespace-readonly-clusterrole --group=system:serviceaccounts:kube-user
clusterrolebinding.rbac.authorization.k8s.io/read-namespace created
```

* mark用户绑定rq-test ns下的pod非只读权限

```shell
kubectl create rolebinding not-read-pod --clusterrole=pods-edit-clusterrole  --serviceaccount=kube-user:mark --namespace=rq-test
```



* mark用户绑定rq-test ns下的deployment只读权限

```shell
kubectl create rolebinding read-workload --clusterrole=workload-readonly-clusterrole  --serviceaccount=kube-user:mark --namespace=rq-test
```

根据企业需要灵活设置RBAC权限控制，坑点就是不同apiGroups分组类型的资源，要分别配置分组类型哦。



# 云原生存储及存储进阶

rook如何使用

如何对接stroage class 外部存储



![image-20240516212835272](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240516212835272.png)





![image-20240516213827316](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240516213827316.png)



![image-20240516214635660](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240516214635660.png)









![image-20240516215523148](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240516215523148.png)



![image-20240516220416094](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240516220416094.png)













nfs csi



























# 中间件容器化及Helm

## 1. 部署应用至K8S流程

* 部署应用的类型
  * 架构
  * 配置
  * 端口
  * 启动命令
* 镜像
  * 如何制作镜像，建议从官方下载，需要定制化的也可以参考官方的dockfile
* 部署的方式
  * 有无状态应用
  * 配置是否分离
  * 部署文件的来源
  * 如何部署
* 程序如何使用
  * 什么协议
  * 内部 or 外部访问

## 2. K8s管理集群的两种方式

![image-20240523091125534](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240523091125534.png)



## 3. Operator创建redis集群

### 3.1 部署

Operator仓库： https://operatorhub.io, 生产环境需要将镜像打到自己的镜像仓库中

* redis集群介绍
  1. 所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽。
  2. 节点的fail是通过集群中超过半数的节点检测失效时才生效。
  3. 客户端与redis节点直连,不需要中间proxy层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可。
  4. redis-cluster把所有的物理节点映射到[0-16383]slot上（不一定是平均分配）,cluster 负责维护node<->slot<->value。
  5. Redis集群预分好16384个桶，当需要在 Redis 集群中放置一个 key-value 时，根据 CRC16(key) mod 16384的值，决定将一个key放到哪个桶中。
  6. Redis Cluster要求至少需要3个master才能组成一个集群，同时每个master至少需要有一个slave节点

* 创建Operator和CRD

```
git clone https://gitee.com/dukuan/td-redis-operator.git
cd td-redis-operator
kubectl create -f deploy/deploy.yaml
```

customresourcedefinition.apiextensions.k8s.io/redisclusters.cache.tongdun.net created
customresourcedefinition.apiextensions.k8s.io/redisstandbies.cache.tongdun.net created
role.rbac.authorization.k8s.io/admin created
role.rbac.authorization.k8s.io/operator created
rolebinding.rbac.authorization.k8s.io/admin created
rolebinding.rbac.authorization.k8s.io/operator created
clusterrole.rbac.authorization.k8s.io/admin-cluster created
clusterrolebinding.rbac.authorization.k8s.io/admin-cluster created
serviceaccount/operator created
deployment.apps/operator created



* 创建redis集群

```
kubectl create -f cr/redis_cluster.yaml
```

解析

```
apiVersion: cache.tongdun.net/v1alpha1
kind: RedisCluster
metadata:
  name: redis-cluster-trump
  namespace: redis
spec:
  app: cluster-trump  #要与名称对应起来看
  capacity: 32768     #容量大小 M
  dc: hz              #机房
  env: demo           # demo，staging，prodction
  image: tongduncloud/redis-cluster:0.2 #redis镜像
  monitorimage: tongduncloud/redis-exporter:1.0 #exporter镜像
  netmode: ClusterIP
  proxyimage: tongduncloud/predixy:1.0 #代理镜像
  proxysecret: "123"    #代理密码
  realname: demo        #负责人
  secret: abc   #redis密码
  size: 1       #集群group数量
  storageclass: "nfs-storageclass" #pvc或者动态storageclass,生成不要用nfs
  vip: ""  #实际
```



* 查看集群状态

```
kubectl get rediscluster -n redis redis-cluster-trump -oyaml

status:
  capacity: 1024
  clusterIP: 10.101.7.232:6379
  externalip: :30804
  gmtCreate: "2024-05-23 02:56:57"
  phase: Ready
  size: 3
  slots:
    redis-cluster-trump-0:
    - 5461-10922
    redis-cluster-trump-1:
    - 0-5460
    redis-cluster-trump-2:
    - 10923-16383

```

* 部署图形化管理工具

```
kubectl create -f deploy/web-deploy.yaml
```

![image-20240523112202126](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240523112202126.png)

* 扩容集群

```
redis_cluster.yaml修改size值来扩容
kubectl apply -f redis_cluster.yaml
```

* 删除集群

```
kubectl delete rediscluster xx -n redis
```



### 3.2 验证redis集群的可用性

* 连接redis svc地址

```
[root@k8s-master01 cr]# kubectl exec -it redis-cluster-trump-0-0 -c redis-cluster-trump-0 -n redis -- bash
[root@redis-cluster-trump-0-0 /]# redis-cli -h redis-cluster-trump
redis-cluster-trump:6379> auth mark12
OK
redis-cluster-trump:6379> set a 1
(error) MOVED 15495 172.18.195.57:6379
redis-cluster-trump:6379>
[root@redis-cluster-trump-0-0 /]# redis-cli -h 172.18.195.57
172.18.195.57:6379> auth mark12
OK
172.18.195.57:6379> set a 1
OK
172.18.195.57:6379> get a
"1"
```

* 连接redis代理的svc地址

```
172.18.195.57:6379> [root@k8s-master01 cr]# kubectl exec -it redis-cluster-trump-0-0 -c redis-cluster-trump-0 -n redis -- bash
[root@redis-cluster-trump-0-0 /]# redis-cli -h predixy-redis-cluster-trump
predixy-redis-cluster-trump:6379> auth mark123
OK
predixy-redis-cluster-trump:6379> get a
"1"
predixy-redis-cluster-trump:6379> set b 2
OK
predixy-redis-cluster-trump:6379> get b
"2"
```

## 4. Helm

Helm：https://helm.sh/zh/docs/intro/install/

Helm Charts仓库：https://artifacthub.io/

### 4.1 安装Helm

```
wget https://get.helm.sh/helm-v3.15.0-linux-amd64.tar.gz
tar xf helm-v3.15.0-linux-amd64.tar.gz
cd linux-amd64 && mv helm /usr/local/bin/
```

### 4.2 安装zookeeper

* 添加bitnami仓库（**推荐**）

```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

* 查看bitnami repo内组件的版本

```
helm search repo bitnami/zookeeper -l
```

* 下载zookeeper chart包

```
helm pull bitnami/zookeeper --version 13.3.0
```

* 安装 需要修改vaues.yaml的配置：副本数、auth、持久化

```
 helm install -n lianxi zookeeper .
```

* 提供zookeeper连接地址给开发人员(zookeeper:2181)

```
[root@k8s-master01 zookeeper]# kubectl  get svc -n lianxi
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
zookeeper            ClusterIP   10.104.73.198   <none>        2181/TCP                     2m22s
zookeeper-headless   ClusterIP   None            <none>        2181/TCP,2888/TCP,3888/TCP   2m22s
```

### 4.3 安装kafka带zookeeper方式

```
helm install kafka bitnami/kafka \
--set controller.replicaCount=0 \
--set kraft.enabled=false \
--set zookeeper.enabled=false \
--set broker.replicaCount=3 \
--set externalZookeeper.servers=zookeeper \
--set broker.persistence.enabled=false \
--set listeners.client.protocol=PLAINTEXT \
--set listeners.controller.protocol=PLAINTEXT \
--set listeners.interbroker.protocol=PLAINTEXT \
--set listeners.external.protocol=PLAINTEXT -n lianxi
```



###4.4 安装kafka不带zookeeper方式

```
helm install kafka bitnami/kafka \
--set kraft.enabled=true \
--set zookeeper.enabled=false \
--set controller.replicaCount=3 \
--set controller.persistence.enabled=false \
--set broker.persistence.enabled=false \
--set listeners.client.protocol=PLAINTEXT \
--set listeners.controller.protocol=PLAINTEXT \
--set listeners.interbroker.protocol=PLAINTEXT \
--set listeners.external.protocol=PLAINTEXT -n lianxi
```

### 4.5 验证kafka集群

```
 kubectl run kafka-client --restart='Never' --image docker.io/bitnami/kafka:3.7.0-debian-12-r6 --namespace lianxi --command -- sleep infinity
    kubectl exec --tty -i kafka-client --namespace lianxi -- bash

    PRODUCER:
        kafka-console-producer.sh \
            --broker-list kafka-broker-0.kafka-broker-headless.lianxi.svc.cluster.local:9092,kafka-broker-1.kafka-broker-headless.lianxi.svc.cluster.local:9092,kafka-broker-2.kafka-broker-headless.lianxi.svc.cluster.local:9092 \
            --topic test
>>>xxx
>nihao

    CONSUMER:
        kafka-console-consumer.sh \
            --bootstrap-server kafka.lianxi.svc.cluster.local:9092 \
            --topic test \
            --from-beginning
xxx
nihao
```

### 4.6 Helm常用命令

#### 4.6.1 下载chart包

```
helm pull bitnami/zookeeper --version 13.3.0 
```



#### 4.6.2 搜索chart包

```
helm search repo bitnami/zookeeper -l
```



#### 4.6.3 创建包

```
helm create
```



#### 4.6.4 安装chart包

```
helm install 
```



#### 4.6.5 查看helm

```
helm list
```



#### 4.6.6 查看helm安装参数

```
helm get values
```



#### 4.6.7 更新

* 本地化更新，先修改values.yaml

```
helm upgrade -n lianxi zookeeper .
```

* 在线更新，需要把set的参数都带上 

```
helm upgrade kafka bitnami/kafka \
--set controller.replicaCount=0 \
....
--set listeners.interbroker.protocol=PLAINTEXT \
--set listeners.external.protocol=PLAINTEXT -n lianxi
```

#### 4.6.8 删除

```
helm delete
```

#### 4.6.9 回滚

```
#查看历史版本
helm history zookeeper  -n lianxi
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION
1               Thu May 23 21:36:53 2024        deployed        zookeeper-13.3.0        3.9.2           Install complete

#更加版本号回滚
helm rollback zookeeper 1
```

### 4.7 charts包目录解析

```
[root@k8s-master01 helm-test]# tree .
.
├── charts #依赖包，比如kafka依赖zookeeper,可以把zookeeper chart包存放此处
├── Chart.yaml #chart的基本信息
	apiVersion: v2  //chart的api版本，默认是2
    name: helm-test  //chart名称
    description: A Helm chart for Kubernete //chart描述信息
    type: application //图表类型
    version: 0.1.0  //chart版本
    appVersion: "1.16.0" //应用版本

├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl #自定义模版或函数
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt  //chart安装完毕后的提示信息，一般回简单介绍应用的用法
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests //测试文件
│       └── test-connection.yaml
└── values.yaml //全局的变量

```



### 4.8 常用变量

```
name: {{ .Release.Name }}  实例名称，helm install的名称
namespace: {{ .Release.Namespace }} 实例的命名空间
a: "{{ .Release.IsInstall }}"    返回布尔值，安装恢复true
b: "{{ .Release.IsUpgrade }}"   是否更新或回滚，返回布尔值
version : "{{ .Release.Revision}}"  当前实例的修订版本号，从1开始，每次升级或回滚就贵+1
desr: {{ .Chart.Description }} 从Chart.yaml中获取，

```

4.9 常用函数

https://masterminds.github.io/sprig/strings.html

| 函数       | 例子                                                     | 描述                                                         |
| ---------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| trim       | trim "   hello    "   hello                              | The `trim` function removes space from either side of a string: |
| trimAll    | trimAll "$" "$5.00" 5.00                                 | Remove given characters from the front or back of a string:  |
| trimSuffix | trimSuffix "-" "hello-" hello                            | Trim just the suffix from a string:                          |
| trimPrefix | trimPrefix "-" "-hello" hello                            | Trim just the prefix from a string:                          |
| upper      | upper "hello" HELLO                                      | Convert the entire string to uppercase:                      |
| lower      | lower "HELLO" hello                                      | Convert the entire string to lowercase:                      |
| title      | title "hello world" Hello World                          | Convert to title case:                                       |
| repeat     | repeat 3 "hello" hellohellohello                         | Repeat a string multiple times:                              |
| substr     | substr 0 5 "hello world" hello                           | Get a substring from a string. It takes three parameters:    |
| trunc      | trunc 5 "hello world" hello trunc -5 "hello world" world | Truncate a string (and add no suffix)                        |
| indent     | indent 4 $lots_of_text                                   | The `indent` function indents every line in a given string to the specified indent width. This is useful when aligning multi-line strings |
| nindent    | nindent 4 $lots_of_text                                  | The `nindent` function is the same as the indent function, but prepends a new line to the beginning of the string. |
| quote      |                                                          | hese functions wrap a string in double quotes (`quote`) or single quotes (`squote` |

### 4.9 逻辑控制

#### 4.9.1 If/Else

```
{{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
 {{- end }}

```

* eq 判断是否符合条件

```
{{- if eq "1" .Values.version.enabled }}
   xxx
 {{- else if eq 2 .Values.version.enabled }}  
   yy
 {{- else }}  
   zzz
 {{- end }}
```

#### 4.9.2 range 循环

* 循环列表

```
{{- range .Values.slice }}
 {{ . }} .取到列表中的值
{{- end}}
```

* 循环字典

```
{{- range $k , $v := .Values.slice }}
 {{ $K }}: {{ $v }} .取到列表中的值
{{- end}}
```

#### 4.9.3 with

相当于切换环境, 可以直接调用局部的值

toYaml 相当于把.Values.volumes中整块内容拿过来, 内容是顶格的所以需要nindent来缩进

```
{{- with .Values.volumes }}
volumes:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

#### 4.9.4 include

include调用_helpers.tpl文件的defind

```
{{- define "helm-test.labels" -}}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

deployment.yaml文件中调用
{{- include "helm-test.labels" . | nindent 4 }}
```



#### 4.9.4 default

如果.Values.OO 不存在， 就是xxx默认值

```
{{ default "xxx" .Values.OO }}
```







# K8S容器日志收集



# Prometheus监控



# Alertmanager告警



# 服务发布Ingress进阶



# CI/CD持续集成







故障处理流程

恢复业务-排查问题-复盘
