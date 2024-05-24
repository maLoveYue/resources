# k8s架构剖析



## 1. K8s概念

kubernetes是谷歌公司开发的一套容器管理系统，提供了资源调度、扩容缩容、服务发现、存储编排、自动部署和回滚，并具有高可用、负载均衡和故障自动恢复的能力。

## 2. K8s集群架构图

<img src="C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20231204145541553.png" alt="image-20231204145541553" style="zoom:80%;" />

# 控制节点组件介绍

<img src="C:\Users\Mark\Desktop\K8S集群组件.png" alt="K8S集群组件" style="zoom:200%;" />



## 1. 组件介绍

* ### kube-apiserver

  提供整体集群管理API接口：其他模块之间的数据交互通信

  集群内部模块之间的通信枢纽：各模块之前不会相互调用，都是通过api server来交互的

  集群安全控制：api server 提供验证和授权保证集群的安全

  数据中心枢纽：只有api server可以和etcd交互存放集群数据

  集群入口：提供REST API接口供外部调用

* ### etcd

  etcd在kubernetes集群是用来存放数据并通知变动的。

  Kubernetes中没有用到数据库，它把关键数据都存放在etcd中，这使kubernetes的整体结构变得非常简单。在kubernetes中，数据是随时发生变化的，比如说用户提交了新任务、增加了新的Node、Node宕机了、容器死掉了等等，都会触发状态数据的变更。状态数据变更之后呢，Master上的kube-scheduler和kube-controller-manager，就会重新安排工作，它们的工作安排结果也是数据。这些变化，都需要及时地通知给每一个组件。etcd有一个特别好用的特性，可以调用它的api监听其中的数据，一旦数据发生变化了，就会收到通知。有了这个特性之后，kubernetes中的每个组件只需要监听etcd中数据，就可以知道自己应该做什么。kube-scheduler和kube-controller-manager呢，也只需要把最新的工作安排写入到etcd中就可以了，不用自己费心去逐个通知了

* ### kube-scheduler

  Scheduler负责节点资源管理，接收来自kube-apiserver创建Pods的任务，收到任务后它会检索出所有符合该Pod要求的Node节点（通过预选策略和优选策略），开始执行Pod调度逻辑。调度成功后将Pod绑定到目标节点上。

* ### kube-controller-manager

   k8s 集群的管理控制中心： 负责集群内 Node、Namespace、Service、Token、Replication 等资源对象的管理，使集群内的资源对象维持在预期的工作状态。

  每一个 controller 通过 api-server 提供的 restful 接口实时监控集群内每个资源对象的状态，当发生故障，导致资源对象的工作状态发生变化，就进行干预，尝试将资源对象从当前状态恢复为预期的工作状态，常见的 controller 有 Namespace Controller、Node Controller、Service Controller、ServiceAccount Controller、Token Controller、ResourceQuote Controller、Replication Controller等。

  有许多不同类型的控制器。以下是一些例子：

  - 节点控制器（Node Controller）：负责在节点出现故障时进行通知和响应
  - 任务控制器（Job Controller）：监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成
  - 端点分片控制器（EndpointSlice controller）：填充端点分片（EndpointSlice）对象（以提供 Service 和 Pod 之间的链接）。
  - 服务账号控制器（ServiceAccount controller）：为新的命名空间创建默认的服务账号（ServiceAccount）。

* ### kubelet

  运行在每个节点上

  通过api server接口监听Scheduler产生的绑定pod事件，然后从etcd获取pod清单，下载镜像并启动容器

  同时监视分配给该Node节点的 pods，周期性获取容器状态，再通过api-server通知各个组件。

* ### kube-proxy

  kube-proxy 会作为 daemon（守护进程） 跑在每个节点上通过watch的方式监控着etcd中关于Pod的最新状态信息

  Pod资源被删除了或新建或ip变化了等一系列变动，它就立即将这些变动，反应在iptables 或 ipvs规则中，以便之后 再有请求发到service时，service可以通过ipvs最新的规则将请求的分发到pod上

  `kube-proxy和service的关系:`

  ​	Kube-proxy负责制定数据包的转发策略，并以守护进程的模式对各个节点的pod信息实时监控并更新转发规则，service收到请求后会根据kube-proxy制定好的策略来进行请求的转发，从而实现负载均衡

  节点上pod之间的通信和负责均衡，将流量分发到后端的正确的机器上。

* ### 容器运行时

  这个基础组件使 Kubernetes 能够有效运行容器。 它负责管理 Kubernetes 环境中容器的执行和生命周期。

  * 容器运行时

  kubernets集群内部service解析，可以让pod解析service名称，然后通过service ip连接到应用。

  * Calico 

  符合CNI标准的网络插件，负责为每个pod分配一个不重复的ip，每个节点相当于一个路由器，这样就可以让不同节点上的pod相互通信。

# Pod介绍

## 1. 什么是pod

k8s中最小的可部署单元，一组或一个多个容器组成，每个pod包含pause容器,pause容器是pod的父容器，主要作用是僵尸进程的回收，使得同一个pod共享存储、网络、pid、ipc等，比如同一个pod不同容器间可以通过localhost:port方式互访。 

## 2. 常见的pod状态及排查

| 状态                         | 解释                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| Pending （挂起）             | pod已被接受，但容器未创建，通过kubectl describe 查看         |
| Running （运行中）           | pod已经绑定到一个节点了，容器已经创建完成了，至少一个已经运行正常，或正在启动或重启，kubectl logs查看 |
| Succeeded （成功）           | 所有容器执行成功并不会重启，kubectl logs查看                 |
| Failed (失败)                | 所有容器已终止，至少有一个容器已失败方式终止，也就是容器以非零状态退出，kubectl logs和kubectl describe 查看 |
| Unknown (未知)               | 通常是通信问题导致无法获取pod状态                            |
| ImagePullBackOffErrImagePull | 镜像拉取失败，镜像不存在或网络不通或登录认证，kubectl describe 查看 |
| CrashLoopBackOff             | 容器启动失败，kubectl logs查看，一般是容器启动命令和健康检测不通过导致的 |
| OOMKilled                    | 容器内存溢出，一般是内存limit设置过小，或者程序本身内存溢出，通过logs |
| Terminating                  | Pod正在被删除                                                |
| Completed                    | 容器内主进程退出，一般是计划任务执行结束会显示改状态，可通过logs查看容器日志 |
| ContainerCreating            | pod正在创建，一般是正在下载镜像，或者有配置不当地方，kubectl describe 查看 |

## 3. pod镜像拉取策略

| 策略名称     | 描述                                                         |
| ------------ | ------------------------------------------------------------ |
| Always       | 总是拉取，当镜像tag为latest时，且imagePullPolicy未配置，默认为Always |
| Never        | 不管在不在都不会拉取                                         |
| IfNotPresent | 如果不存在就拉取，tag为非latest时，默认该策略                |

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - name: nginx
    image: 192.168.0.203:8089/mark/nginx:latest
    imagePullPolicy: IfNotPresent
    command:
    - sh
    - -c
    - sleep 5 ; nginx -g 'daemon off;'
    ports:
    - containerPort: 80
  imagePullSecrets:
  - name: docker-harbor
```

## 4. pod重启策略

| 策略名称  | 描述                                                 |
| --------- | ---------------------------------------------------- |
| Always    | 默认策略，容器失败时，自动重启该容器                 |
| OnFailure | 容器以非0状态码终止，自动重启该容器，echo $?的状态码 |
| Never     | 无论何种状态，都不重启                               |

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - name: nginx
    image: 192.168.0.203:8089/mark/nginx:latest
    imagePullPolicy: IfNotPresent
    command:
    - sh
    - -c
    - sleep 5 ; nginx -g 'daemon off;'
    ports:
    - containerPort: 80
  restartPolicy: Always
  imagePullSecrets:
  - name: docker-harbor
```



## 5. pod存活性探针

加入探针后需要等健康检查通过后，才会变更pod状态和引流，在健康检查延迟时间后会执行对应得检查，对于时间启动比较长的应用需要用**startupProbe** ，

| 探针                          | 描述                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| startupProbe（启动时）        | k8s 1.16新加的探测方式，判读容器应用是否已经启动，如果配置startupProbe,就会先禁用其他探测，直到成功为止，如果失败就会杀死容器，没配置默认成功。（用于容器启动阶段检测）只执行一次,对于容器启动时间过程长适用，不仅仅是重启容器 是从新拉取容器开始。 |
| livenessProbe （ 存活性探针） | 探测容器是否运行，失败的话，kubelet会kill 容器并更具重启策略进行处理，未指定探针的话，默认为Success。 |
| readinessProbe （就绪性探针） | 用于探测程序是否健康，即判断容器是否Ready,如果是处理该请求，反之endpoints controller将从所有的service的endpos 中删除此容器的所在的pod的IP，如果未指定，默认Success。该探针失败不会重启pod。 |

## 6. pod探针的实现方式

| 方式                      | 描述                                                         |
| ------------------------- | ------------------------------------------------------------ |
| ExecAction                | 在容器内执行一个指定的命令，根据命令反馈值（0健康）来判断是的健康。 |
| TCPSocketAction           | tcp连接检查容器指定端口，端口连接没问题，容器健康            |
| HTTPGetAction（建议生产） | 指导url进行get 请求，状态码在200-400之间，认为容器健康       |
| grpc | 1.24版本后启用，主要是服务是grpc开发的 . |

<img src="C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20231211213713248.png" alt="image-20231211213713248" style="zoom: 80%;" />

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - name: nginx
    image: 192.168.0.203:8089/mark/nginx:latest
    imagePullPolicy: IfNotPresent
    command:
    - sh
    - -c
    - sleep 30; nginx -g 'daemon off;'
    startupProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 10 #健康检查延迟时间
      timeoutSeconds: 2  #健康检查超时时间
      periodSeconds: 20 #健康检查每次间隔时间
      successThreshold: 1 #1次成功表示就绪
      failureThreshold: 2 #2次失败表示未就绪
    livenessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 10
      timeoutSeconds: 2
      periodSeconds: 5
      successThreshold: 1
      failureThreshold: 2
    readinessProbe:
      httpGet:
        path: /index.html
        port: 80
        scheme: HTTP
      initialDelaySeconds: 10
      timeoutSeconds: 2
      periodSeconds: 5
      successThreshold: 1
      failureThreshold: 2
    ports:
    - containerPort: 80
  restartPolicy: Always
  imagePullSecrets:
  - name: docker-harbor

```

# pod生命周期

##  1. 启动过程

<img src="https://cdn.nlark.com/yuque/0/2022/png/1393829/1657802847661-96e579e8-108d-41df-a488-efa061965cdb.png" alt="img" style="zoom:80%;" />

## 2. 退出过程

<img src="C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20231211211703159.png" alt="image-20231211211703159" style="zoom:80%;" />

## 3. 钩子函数

| 函数      | 描述                     |
| --------- | ------------------------ |
| postStart | 在容器创建后，执行该指令 |
| preStop   | 容器删除前，执行该指令   |

```YAML
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - name: nginx
    image: 192.168.0.203:8089/mark/nginx:latest
    imagePullPolicy: IfNotPresent
    lifecycle:
      postStart:
        exec:
          command:
          - sh
          - -c
          - mkdir /tmp/makun
      preStop:
        exec:
          command:
          - sh
          - -c
          - sleep 30
    command:
    - sh
    - -c
    - nginx -g 'daemon off;'
    ports:
    - containerPort: 80
  restartPolicy: Always
  imagePullSecrets:
  - name: docker-harbor

```

# 资源调度

##  5.1 无状态资源管理Deployment

### 5.1.1 创建deployment

用该命令可以生成个简易的deployment的yaml文件

```shell
# kubectl create deploy nginx --image=registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine --replicas=3 -oyaml --dry-run=client >nginx-deploy.yaml

```

查看deploymet资源

```shell
[root@k8s-master01 prd]# kubectl  get deploy
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx          3/3     3            3           18s

```

### 5.1.2 deployment滚动更新

一般更新的是deployment spec.template的内容，一般的更新流程是新创建rs来一次增加副本数来替换老的rs副本数

```shell
# kubectl edit deployment 【deployment名称】
```

### 5.1.3 deployment回滚

一般建议回滚到上一版本，回滚至指定版本添加--to-revision=2

* 查看deployment能被回滚的版本 

```shell
[root@k8s-master01 ~]# kubectl rollout history deploy nginx
deployment.apps/nginx
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
4         <none>
```

* --revision指定版本查看详细信息

```shell
[root@k8s-master01 ~]# kubectl rollout history deploy nginx --revision=3
deployment.apps/nginx with revision #3
Pod Template:
  Labels:       app=nginx
        pod-template-hash=6dcf4bd4b9
  Containers:
   nginx:
    Image:      registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.1-alpine
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
```

* 回滚到上一版本

```shell
[root@k8s-master01 ~]# kubectl rollout  undo deployment nginx
deployment.apps/nginx rolled back
```

* 回滚到指定版本

```shell
[root@k8s-master01 ~]# kubectl rollout  undo deployment nginx --to-revision=4
deployment.apps/nginx rolled back
```

### 5.1.4 deployment扩容/缩容

业务有预期的扩容，随着业务流量增加可扩容，缩容谨慎去做。

```shell
[root@k8s-master01 ~]# kubectl scale deploy nginx --replicas=5
deployment.apps/nginx scaled
 
 [root@k8s-master01 ~]# kubectl  get po
NAME                            READY   STATUS    RESTARTS   AGE
nginx-657dcdc747-5zvt8          1/1     Running   0          21m
nginx-657dcdc747-g8k87          1/1     Running   0          7s
nginx-657dcdc747-jq8s7          1/1     Running   0          7s
nginx-657dcdc747-v6dns          1/1     Running   0          21m
nginx-657dcdc747-zwmpm          1/1     Running   0          21m

```

### 5.1.5  deployment暂停和恢复

针对需要多次修改配置不直接更新的，需要先暂停pause,禁止更新。

* pause 暂停

```shell
# kubectl rollout pause deployment nginx
deployment.apps/nginx paused
```

* resume恢复

```shell
# kubectl rollout resume deployment nginx
deployment.apps/nginx resumed
```

### 5.1.6 deployment更新策略

* revisionHistoryLimit历史版本清理策略

  默认保留10个历史版本，其余的会被后台进行垃圾回收，当revisionHistoryLimit设置为0时，不保留历史记录。

* 更新策略

  .spec.strategy.type.Recreate：重建，先删除旧的pod再创建新的pod。

  .spec.strategy.type.RollingUpdate：滚动更新，指定 maxSurge和maxUnavailable来控制更新比例。

  **maxSurge:** 最大可以超过的pod期望数量，比如deploy replicas副本数为3, maxSurge为1的话，更新过程中最多会有3+1=4个pod。
  **maxUnavailable:**最大不可用pod

```yaml
 spec:
    replicas: 3
    revisionHistoryLimit: 10
    strategy:
      rollingUpdate:
        maxSurge: 25%
        maxUnavailable: 25%
      type: RollingUpdate
```

## 5.2 有状态资源管理StatefulSet

### 5.2.1 创建Statefulset

需要statefulset部署的条件

* 有序
* 稳定唯一网络标识
* 数据持久化

<img src="C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20231221204120971.png" alt="image-20231221204120971"  />

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
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
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # 必须匹配 .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # 默认值是 1
  minReadySeconds: 10 # 默认值是 0
  template:
    metadata:
      labels:
        app: nginx # 必须匹配 .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
        ports:
        - containerPort: 80
          name: web
```

```shell
# kubectl create -f sts-nginx.yaml
```



### 5.2.2 Headless Service

对于无头sevice，集群不会分配cluster ip，kube-proxy在主机自然不会创建iptables或者ipvs的路由条目，是通过CoreDns解析去实现转发。

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-sts-svc
  labels:
    app: nginx-sts-svc
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None #设置为None就不会分配ip
  selector:
    app: nginx-sts

```



Headless格式：ServiceName.namespace.svc.cluster.local

```
#nslookup nginx-sts-svc.default.svc.cluster.local 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   nginx-sts-svc.default.svc.cluster.local
Address: 172.25.92.88
Name:   nginx-sts-svc.default.svc.cluster.local
Address: 172.18.195.26
Name:   nginx-sts-svc.default.svc.cluster.local
Address: 172.25.244.232

```



### 5.2.3  statefulSet删除pod的过程

<img src="C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20231221204706367.png" alt="image-20231221204706367"  />

### 5.2.4 statefulSet扩容缩容

可以通过edit或者scale来控制副本数，进行扩容或缩容操作。

* **扩容**

```shell
[root@k8s-master01 prd]# kubectl  scale sts web --replicas=5
statefulset.apps/web scaled
```

可以通过动态查看

```shell
[root@k8s-master01 ~]# kubectl  get pod -w -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          35m
web-1   1/1     Running   0          35m
web-2   1/1     Running   0          35m
web-3   0/1     Pending   0          1s
web-3   0/1     Pending   0          1s
web-3   0/1     ContainerCreating   0          1s
web-3   1/1     Running             0          3s
web-4   0/1     Pending             0          0s
web-4   0/1     Pending             0          0s
web-4   0/1     ContainerCreating   0          0s
web-4   1/1     Running             0          3s

```

* **缩容**

```shell
[root@k8s-master01 prd]# kubectl  scale sts web --replicas=3
statefulset.apps/web scaled
```

可以通过动态查看

```shell
[root@k8s-master01 ~]# kubectl  get pod -w -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          35m
web-1   1/1     Running   0          35m
web-2   1/1     Running   0          35m
web-3   1/1     Running             0          3s
web-4   1/1     Running             0          3s
web-4   1/1     Terminating         0          2m49s
web-4   0/1     Terminating         0          2m51s
web-4   0/1     Terminating         0          2m59s
web-4   0/1     Terminating         0          2m59s
web-3   1/1     Terminating         0          3m2s
web-3   0/1     Terminating         0          3m5s
web-3   0/1     Terminating         0          3m8s
web-3   0/1     Terminating         0          3m8s
```

### 5.2.5 statefulSet更新策略

* OnDelete策略

需要先手动删除pod，才会更新。

```yaml
  updateStrategy:
    type: OnDelete
```

* RollingUpdate策略

pod会滚动更新

```yaml
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
```

* 分段更新

控制partition分区数值来灰度发布pod更新，如果为只能>=3的pod，只有RollingUpdate策略才有partition。

```YAML
  updateStrategy:
    rollingUpdate:
      partition: 3
    type: RollingUpdate
```

## 5.3 守护进程DaemonSet

### 5.3.1  创建DaemonSet 

DaemonSet 会在所有的节点上创建一个pod（特殊情况除外）

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
        name: nginx

```

* 选择带标签的节点进行部署,会立即匹配新的节点去创建，删除不合适的节点pod

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeSelector:
        disktype: ssd
      containers:
      - image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
        name: nginx
```

### 5.3.2 DaemonSet 更新和回滚

* 更新策略

  和statefulSet 一样分为OnDelete和RollingUpdate

  ```
  updateStrategy:
        rollingUpdate:
          maxUnavailable: 1
        type: RollingUpdate
  ```

* 回滚

  ```shell
  #查看ds历史版本
  # kubectl  rollout history ds nginx
  daemonset.apps/nginx
  REVISION  CHANGE-CAUSE
  1         <none>
  2         <none>
  3         <none>
  
  #指定版本查看详细信息
  [root@k8s-master01 prd]# kubectl  rollout history ds nginx --revision=3
  daemonset.apps/nginx with revision #3
  Pod Template:
    Labels:       app=nginx
    Containers:
     nginx:
      Image:      registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12
      Port:       <none>
      Host Port:  <none>
      Environment:        <none>
      Mounts:     <none>
    Volumes:      <none>
  
  #回滚至上一个版本
  [root@k8s-master01 prd]# kubectl rollout undo ds nginx
  daemonset.apps/nginx rolled back
  
  [root@k8s-master01 prd]# kubectl  rollout history ds nginx --revision=4
  daemonset.apps/nginx with revision #4
  Pod Template:
    Labels:       app=nginx
    Containers:
     nginx:
      Image:      registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
      Port:       <none>
      Host Port:  <none>
      Environment:        <none>
      Mounts:     <none>
    Volumes:      <none>
    
  #回滚到指定版本
  [root@k8s-master01 prd]# kubectl rollout undo ds nginx --to-revision=3
  daemonset.apps/nginx rolled back
  
  [root@k8s-master01 prd]# kubectl  rollout history ds nginx --revision=5
  daemonset.apps/nginx with revision #5
  Pod Template:
    Labels:       app=nginx
    Containers:
     nginx:
      Image:      registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12
      Port:       <none>
      Host Port:  <none>
      Environment:        <none>
      Mounts:     <none>
    Volumes:      <none>
  
  ```

  

## 5.4 自动扩缩容

转发：https://juejin.cn/post/7014432378546290725

### 5.4.1 HPA水平自动伸缩

HPA则要解决的是业务负载压力波动很大，需要人工根据监控报警来不断调整副本数的问题。

`HPA两个关键点：`

* 如何识别业务的忙闲程度
* 使用什么样的副本调整策略

`HPA接口类型：`

* v1为稳定版自动水平伸缩，只支持cpu指标

* v2为beta版本，分为v2beta1（支持cpu、内存、自定义指标）

* v2beta2（支持cpu、内存、自定义指标Custom和ExternalMetrics）

```shell
[root@k8s-master01 prd]# kubectl  get apiservices|grep autoscali
v1.autoscaling                         Local                        True        26d
v2beta1.autoscaling                    Local                        True        26d
v2beta2.autoscaling                    Local                        True        26d
```

* HPA自动扩容实践

```
#deployment或者sts资源，必须要配置requests指标，ds不支持hpa

#暴露端口
kubectl expose deploy nginx --port=80

#对deploy设置hpa,cpu指标的10%来判定
 kubectl autoscale deploy nginx --cpu-percent=10 --min=1 --max=10
 
 #测试cpu使用上升
 while true ; do wget -q -O- http://10.100.112.253 >/dev/null ; done

```

### 5.4.2 VPA垂直自动伸缩

VPA是解决资源配额（Pod的CPU、内存的limit/request）评估不准的问题。



# 6. k8s基础

## 6.1 服务发布入门

<img src="C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20231227205223477.png" alt="image-20231227205223477"  />

​																								服务发布流量图

### 6.1.1 认识Label和Selector

Label:标签，对k8s添加分组，以key=valune来命名。

Selector:标签选择器，可以通过根据资源的标签查询精确的对象信息。

* 查看所有节点所有标签

```shell
[root@k8s-master01 ~]# kubectl get node  --show-labels
NAME           STATUS   ROLES    AGE   VERSION    LABELS
k8s-master01   Ready    <none>   27d   v1.20.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master01,kubernetes.io/os=linux,node.kubernetes.io/node=
k8s-master02   Ready    <none>   27d   v1.20.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master02,kubernetes.io/os=linux,node.kubernetes.io/node=
k8s-master03   Ready    <none>   27d   v1.20.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master03,kubernetes.io/os=linux,node.kubernetes.io/node=
```

* 匹配标签

```shell
[root@k8s-master01 ~]# kubectl get node  -l disktype=ssd
NAME           STATUS   ROLES    AGE   VERSION
k8s-master03   Ready    <none>   27d   v1.20.10
You have new mail in /var/spool/mail/root
[root@k8s-master01 ~]# kubectl get node  -l disktype #只根据key来检索
NAME           STATUS   ROLES    AGE   VERSION
k8s-master03   Ready    <none>   27d   v1.20.10
```

* 创建标签

```shell
[root@k8s-master01 ~]# kubectl label node k8s-master01 subnet=7
node/k8s-master01 labeled
[root@k8s-master01 ~]# kubectl get node  -l subnet=7
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    <none>   27d   v1.20.10
```

* 手动指定pod至特定的标签

  **nodeSelector:** 指定标签为subnet: "7"的节点

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "3"
  creationTimestamp: "2023-12-26T07:03:56Z"
  generation: 11
  labels:
    app: nginx
  name: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
        imagePullPolicy: IfNotPresent
        name: nginx
      dnsPolicy: ClusterFirst
      nodeSelector:
        subnet: "7"
```

```shell
[root@k8s-master01 ~]# kubectl get node  -l subnet=7
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    <none>   28d   v1.20.10
[root@k8s-master01 ~]# kubectl  get po -owide
NAME                     READY   STATUS    RESTARTS   AGE     IP               NODE           NOMINATED NODE   READINESS GATES
nginx-577b86f65c-6z92x   1/1     Running   0          2m35s   172.25.244.225   k8s-master01   <none>           <none>
```

* 过滤标签

```shell
#且，过滤role=master和subnet=7的node
[root@k8s-master01 ~]# kubectl get node -l role=master,subnet=7 
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    <none>   28d   v1.20.10

#非，除了 subnet=7的node
[root@k8s-master01 ~]# kubectl get node -l subnet!=7
NAME           STATUS   ROLES    AGE   VERSION
k8s-master02   Ready    <none>   28d   v1.20.10
k8s-master03   Ready    <none>   28d   v1.20.10

#或，subnet为7或者120的
[root@k8s-master01 ~]# kubectl  get node -l 'subnet in (7,120)' --show-labels
NAME           STATUS   ROLES    AGE   VERSION    LABELS
k8s-master01   Ready    <none>   28d   v1.20.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master01,kubernetes.io/os=linux,node.kubernetes.io/node=,role=master,subnet=7
k8s-master02   Ready    <none>   28d   v1.20.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master02,kubernetes.io/os=linux,node.kubernetes.io/node=,role=master,subnet=120

```

* 修改标签

```shell
#单个节点修改标签
[root@k8s-master01 ~]# kubectl  label node k8s-master01 subnet=121 --overwrite
node/k8s-master01 labeled
[root@k8s-master01 ~]# kubectl  get node -l subnet=121 --show-labels
NAME           STATUS   ROLES    AGE   VERSION    LABELS
k8s-master01   Ready    <none>   28d   v1.20.10   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master01,kubernetes.io/os=linux,node.kubernetes.io/node=,role=master,subnet=121


#批量修改
[root@k8s-master01 ~]# kubectl label node -l subnet subnet=110 --overwrite
node/k8s-master01 labeled
[root@k8s-master01 ~]# kubectl  get node -l subnet=110 --show-labels
NAME           STATUS   ROLES    AGE   VERSION    LABELS
k8s-master01   Ready    <none>   28d   v1.20.10   role=master,subnet=110
k8s-master02   Ready    <none>   28d   v1.20.10   role=master,subnet=110
```

* 删除标签

```shell
#单个删除
[root@k8s-master01 ~]# kubectl label node k8s-master01 subnet-
node/k8s-master01 labeled

#批量删除
[root@k8s-master01 ~]# kubectl label node -l subnet subnet-
node/k8s-master02 labeled
```

### 6.1.2 认识Service

Service是一组pod的访问策略，通过标签选择器来匹配pod，之后通过service来通信。

* 创建Service

```yaml
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80 #自己的端口
    targetPort: 80 #容器的端口也可以为容器端口的名称
```

创建service #kubectl create -f svc.yaml

创建service后，会自动生成和service同名的 endpoint

![image-20240329092326683](C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240329092326683.png)

如何跨namespace访问？   service名称.namespace

### 6.1.3 Service类型

Service是一个抽象的概念。它通过一个虚拟的IP的形式(VIPs)，映射出来指定的端口，通过代理客户端发来的请求转发到后端一组Pods中的一台（也就是endpoint）

#### 1.ClusterIP

集群中内部使用，默认值。东西流量主要使用

```
 type: ClusterIP
```



#### 2.NodePort

在所有安装kube-proy映射个端口来访问后端的pod，访问方式NodeIP:NodePort, 代理端口是--service-node-port-range范围中随机的。

任意一个宿主机就可以访问改svc代理的pod服务

```shell
[root@k8s-master01 prd]# kubectl  get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        35d
nginx-svc    NodePort    10.103.83.251   <none>        80:30976/TCP   19m
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: NodePort
```

#### 3.LoadBalancer

使用云厂商的负载均衡器公开服务,通常而言 `Kubernetes` 集群都是部署在云上的，比如：谷歌云、阿里云等等，这些云服务商一般都可以自动提供负载均衡器。

以阿里云为例：

CCM（Cloud Controller Manager）是阿里云提供的一个用于Kubernetes与阿里云基础产品进行对接的组件，目前包括以下功能：

- 管理负载均衡

  当Service的类型设置为LoadBalancer时，CCM组件会为该Service创建并配置阿里云负载均衡CLB，包括CLB实例、监听、后端服务器组等资源。当Service对应的后端Endpoint或者集群节点发生变化时，CCM会自动更新CLB的后端服务器组。

- 实现跨节点通信

  当集群网络组件为Flannel时，CCM组件负责打通容器与节点间网络，将节点的Pod网段信息写入VPC的路由表中，从而实现容器的跨节点通信。该功能无需配置，安装即可使用。****

* 创建 `tomcat-svc-loadbalancer.yaml`

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: tomcat-loadbalancer
  spec:
    type: LoadBalancer # 注意：这里 type 设置为 LoadBalancer
    ports:
      - port: 80
        targetPort: 8080
    selector:
      app: tomcat
  ```

* 执行创建

```shell
$ kubectl create -f tomcat-svc-loadbalancer.yaml
service/tomcat-loadbalancer created
# PORT(S) 这里，80 为集群内部服务的端口，30544 为 nodePort 端口（由集群随机选择）
$ kubectl get svc
NAME                  TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes            ClusterIP      10.96.0.1      <none>        443/TCP        86d
tomcat-loadbalancer   LoadBalancer   10.109.126.5   <pending>     80:30544/TCP   8s
创建服务以后，云服务商需要一段时间才能创建负载均衡器并将其 IP 地址写入服务对象。如果成功创建阿里云的负载均衡器，比如获取服务信息的时候显示 EXTERNAL-IP 为 101.37.XX.XX，可以在通过本地浏览器直接访问 http://101.37.XX.XX
```



#### 4.ExternalName

通过返回定义的CNAME别名，访问集群外的服务，它通常用于将集群内的服务与集群外部的服务进行互联，比如连接到外部数据库、消息队列或者其他无法直接暴露在集群中的服务。

```shell
apiVersion: v1
kind: Service
metadata:
  name: en-svc
  namespace: default
spec:
  type: ExternalName # type 类型需要选择 ExternalName
  externalName: www.shiyanlou.com # externalName 中填写外部服务对应的域名 
```

```
  
[root@k8s-master01 prd]# kubectl  get svc
NAME             TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
kubernetes       ClusterIP      10.96.0.1        <none>          443/TCP        120d
my-service       NodePort       10.108.214.214   <none>          80:31184/TCP   82m
nginx-external   ExternalName   <none>           www.baidu.com   <none>         4s

容器内部测试：
/ # ping en-svc
PING en-svc (121.40.227.60): 56 data bytes
64 bytes from 121.40.227.60: seq=0 ttl=89 time=21.416 ms
64 bytes from 121.40.227.60: seq=1 ttl=89 time=20.747 ms

64 bytes from 121.40.227.60: seq=2 ttl=89 time=20.639 ms
```



####5. Service代理外部服务

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-external
  name: nginx-external
spec:
  ports:
  - name: http
    protocol: TCP
    port: 5553
    targetPort: 53
  type: ClusterIP
---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    app: baidu
  name: baidu
subsets:
- addresses:
  - ip: 8.8.8.8
  ports:
  - name: http
    port: 53
    protocol: TCP

```

#### 6. 多端口的Service



可以参考kube-dns service设置 多个端口映射至master1上的coreDns pod

```
[root@k8s-master01 ~]# kubectl get ep -n kube-system
NAME             ENDPOINTS                                                 AGE
kube-dns         172.25.244.247:53,172.25.244.247:53,172.25.244.247:9153   120d
metrics-server   172.25.244.248:4443                                       120d
You have new mail in /var/spool/mail/root
[root@k8s-master01 ~]# kubectl  get svc -n kube-system
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
kube-dns         ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP   120d
metrics-server   ClusterIP   10.103.119.92   <none>        443/TCP                  120d

```

```
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: CoreDNS
  name: kube-dns
  namespace: kube-system
spec:
  ports:
  - name: dns
    port: 53
    protocol: UDP
    targetPort: 53
  - name: dns-tcp
    port: 53
    protocol: TCP
    targetPort: 53
  - name: metrics
    port: 9153
    protocol: TCP
    targetPort: 9153
  selector:
    k8s-app: kube-dns
  type: ClusterIP
status:

```



### 6.1.4 Ingress服务发布

ingress为k8s集群服务提供入口，可以提供负责均衡服务、SSL终止、基于域名的虚拟主机、应用灰度发布。Ingress控制器有Nginx、Haproxy、Istio等。

<img src="C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240329112300922.png" alt="image-20240329112300922" style="zoom:50%;" />

#### 1. nginx与ingress关系

<img src="C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240329112957345.png" alt="image-20240329112957345" style="zoom:50%;" />



#### 2. 安装Ingress Controller

```shell
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.6.4/deploy/static/provider/cloud/deploy.yaml
mv deploy.yaml ingress-nginx-controller.yaml 
kubectl apply -f ingress-nginx-controller.yaml
```



注意，可能会应网络问题无法拉取相关镜像，可以修改ingress-nginx-controller.yaml文件里image字段的仓库地址，我已经把镜像传到我的Docker Hub仓库中：

- registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:v1.0.0
- registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen:v1.0

修改后：

```shell
# egrep "image" ingress-nginx-controller.yaml | grep -v imagePullPolicy
        image: tantianran/ingress-controller:v1.6.4
        image: tantianran/kube-webhook-certgen:v20220916-gd32f8c343
        image: tantianran/kube-webhook-certgen:v20220916-gd32f8c343
        
        
[root@k8s-master01 prd]# kubectl  get po -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-2pgpc        0/1     Completed   0          17s
ingress-nginx-admission-patch-w2hbr         0/1     Completed   1          17s
ingress-nginx-controller-765fc67d56-bg67t   1/1     Running     0          17s


```



#### 3.域名发布

配置多个匹配路径

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-goweb
spec:
  ingressClassName: nginx
  rules:
  - host: "test.noblameops.local"
    http:
      paths:
      - path: /login
        pathType: Prefix
        backend:
          service:
            name: test-goweb
            port:
              number: 80
      - path: /home
        pathType: Prefix
        backend:
          service:
            name: test-goweb
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-goweb
            port:
              number: 80
```





## 6.2 配置管理

#### 6.2.1 配置分离

配置分离：主要有ConfigMap和Secret，程序与配置分离，具备命名空间隔离性。



#### 6.2.2 ConfigMap

##### 1. 创建cm

Usage:

```shell
kubectl create configmap NAME [--from-file=[key=]source] [--from-literal=key1=value1] [--dry-run=server|client|none]

```



* 目录创建方式

```shell
# kubectl create cm gamecm --from-file=../conf/
configmap/gamecm created

[root@k8s-master01 conf]# ll
total 8
-rw-r--r-- 1 root root 42 Apr  9 10:36 game1.conf
-rw-r--r-- 1 root root 40 Apr  9 10:35 game.conf
[root@k8s-master01 conf]#

```

* 文件创建方式

```shell
# kubectl create cm game1cm --from-file=../conf/game1.conf
```

* 修改data key名称

```shell
kubectl create cm game2cm --from-file=game2-conf=../conf/game1.conf

[root@k8s-master01 conf]# kubectl  get cm game2cm -oyaml
apiVersion: v1
data:
  game2-conf: |
    username=mark1
    url=https://www.baidu1.com
kind: ConfigMap

```

* 批量创建环境变量方式

```shell
#kubectl  create cm game2-cm-env --from-env-file=./game2.conf
```

* 单独创建环境变量方式

```shell
#kubectl  create cm game3-cm-env --from-literal=username=mark  --from-literal=password=123 
```

* yaml文件创建方式

```shell
# kubectl  create cm game3-cm-env --from-literal=username=mark   --dry-run=client -oyaml
apiVersion: v1
data:
  username: mark
kind: ConfigMap
metadata:
  name: game3-cm-env

```

##### 2.  使用valueFrom定义环境变量

**Deployment资源中配置cm环境变量**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-cm-dp
  name: nginx-cm-dp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-cm
  template:
    metadata:
      labels:
        app: nginx-cm
    spec:
      containers:
      - image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
        name: nginx-web
        env:
        - name: USERNAME
          valueFrom:
            configMapKeyRef:
              name: game2-cm-env
              key: username
        - name: URL
          valueFrom:
            configMapKeyRef:
              name: game2-cm-env
              key: url
```

##### 3. envFrom批量设置环境变量

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-cm-dp
  name: nginx-cm-dp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-cm
  template:
    metadata:
      labels:
        app: nginx-cm
    spec:
      containers:
      - image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
        name: nginx-web
        envFrom:
        - configMapRef:  #可选多个cm配置环境变量
            name: game2-cm-env
          prefix: CM_  #给pod内的环境变量添加前缀为CM_username

```

##### 4. 挂载配置文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-cm-dp
  name: nginx-cm-dp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-cm
  template:
    metadata:
      labels:
        app: nginx-cm
    spec:
      containers:
      - image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
        name: nginx-web
        volumeMounts:
          - name: game1conf #volume的名称
            mountPath: /tmp/conf #挂载到pod的目录
      volumes:
      - name: game1conf
        configMap:
          name: game1cm
```

* 指定cm key值挂载，针对多个文件的cm

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-cm-dp
  name: nginx-cm-dp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-cm
  template:
    metadata:
      labels:
        app: nginx-cm
    spec:
      containers:
      - image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
        name: nginx-web
        volumeMounts:
          - name: gameconf
            mountPath: /tmp/conf
        envFrom:
        - configMapRef:
            name: game2-cm-env
          prefix: CM_
        - configMapRef:
            name: game3-cm-env
          prefix: CK_
      volumes:
      - name: gameconf
        configMap:
          name: gamecm
          items:
          - key: game.conf
            path: game.conf.20240409bak
            mode: 0644 #优先级更高
          defaultMode: 0777


```

#### 6.2.3 Secret

Secret 是一种包含少量敏感信息例如密码、令牌或密钥的对象。 这样的信息可能会被放在 [Pod](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/) 规约中或者镜像中。 使用 Secret 意味着你不需要在应用程序代码中包含机密数据。

##### 1. Secret常用类型

| 类型                                  | 用法                                     |
| ------------------------------------- | ---------------------------------------- |
| Opaque                                | 用户定义的任意数据                       |
| `kubernetes.io/service-account-token` | 服务账号令牌                             |
| `kubernetes.io/dockerconfigjson`      | `~/.docker/config.json` 文件的序列化形式 |
| `kubernetes.io/basic-auth`            | 用于基本身份认证的凭据                   |
| `kubernetes.io/ssh-auth`              | 用于 SSH 身份认证的凭据                  |
| `kubernetes.io/tls`                   | 用于 TLS 客户端或者服务器端的数据        |
| `bootstrap.kubernetes.io/token`       | 启动引导令牌数据                         |





##### 2. 创建Secret

* 创建Opaque类型的Secret

```shell
# echo 'admin'>username
# echo 'password123'>passwd


# kubectl create secret  generic userinfo-secret --from-file=./username --from-file=./passwd
secret/userinfo-secret created
或者
#kubectl create secret generic db-user-pass \
    --from-literal=username=admin \
    --from-literal=password='password123'


[root@k8s-master01 secret]# kubectl  get secret userinfo-secret -oyaml
apiVersion: v1
data:
  passwd: cGFzc3dvcmQxMjMK
  username: YWRtaW4K
kind: Secret
metadata:
  name: userinfo-secret
  namespace: default
  resourceVersion: "5106034"
  uid: 42837121-e021-4327-b44b-bfe624bc5131
type: Opaque
```

`使用Secret到deployment中`

```yaml
#valueFrom

apiVersion: v1
kind: Pod
metadata:
  name: env-single-secret
spec:
  containers:
  - name: envars-test-container
    image: nginx
    env:
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: backend-user
          key: backend-username
#envFrom
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-secret
spec:
  containers:
  - name: envars-test-container
    image: nginx
    envFrom:
    - secretRef:
        name: test-secret
        
        
#文件挂载
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-secret-dp
  name: nginx-secret
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
        name: nginx-web
        volumeMounts:
        - name: userinfo
          mountPath: /tmp/config
      volumes:
      - name: userinfo
        secret:
          secretName: userinfo-secret
```

* 创建容器仓库的Secret

```shell
kubectl create secret docker-registry docker-harbor \
  --docker-email=18906534060@1 \
  --docker-username=makun \
  --docker-password=Harbor12345 \
  --docker-server=192.168.0.203:8089
  
  
  或者
kubectl  create  secret docker-registry test-harbor --from-file=.dockerconfigjson=/root/.docker/config.json  
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - name: nginx
    image: 192.168.0.203:8089/mark/nginx:latest
  imagePullSecrets: #引用容器仓库的secret
  - name: docker-harbor
```

* 创建tls的secret

```shell
模拟证书
# openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=test.com"
 创建
#kubectl create secret tls my-tls-secret --cert=tls.crt --key=tls.key
```

公钥/私钥对必须事先存在，`--cert` 的公钥证书必须采用 .PEM 编码， 并且必须与 `--key` 的给定私钥匹配。

`ingress加上证书文件`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-goweb
spec:
  ingressClassName: nginx
  rules:
  - host: "test.noblameops.local"
    http:
      paths:
      - path: /login
        pathType: Prefix
        backend:
          service:
            name: test-goweb
            port:
              number: 80
  tls:
  - secretName: my-tls-secret
```

##### 3. subPath解决挂载覆盖

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-cm-dp
  name: nginx-cm-dp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-cm
  template:
    metadata:
      labels:
        app: nginx-cm
    spec:
      containers:
      - image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
        name: nginx-web
        volumeMounts:
          - name: game1conf
            mountPath: /etc/game1.conf
            subPath: game1.conf
      volumes:
      - name: game1conf
        configMap:
          name: game1cm
```



##### 4. ConfigMap&Secret热更新

```shell
#针对环境变量类型的cm
kubectl  edit cm gamecm

#针对挂载的配置文件类型的,先修改好配置文件再replace
kubectl create cm nginx-conf-cm -from-file=nginx.conf --dry-run=client -oyaml|kubectl replace -f -
```

##### 5. ConfigMap&Secret使用限制

* `valueFrom引用的key必须存在`
* `具备命令空间隔离性`
* `valueFrom、envFrom无法热更新环境变量`
* `envFrom如果key无效，会自动忽略`
* `subPath无法热更新`
* `ConfigMap&Secret size不要太大`

##### 6.  ConfigMap&Secret只读

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  ...
data:
  ...
immutable: true #cm不可更改
```

## 6.3 持久化存储入门

### 6.3.1 安装NFS

NFS(Network File System)意为网络文件系统，它最大的功能就是可以通过网络，让不同的机器不同的操作系统可以共享彼此的文件。简单的讲就是可以挂载远程主机的共享目录到本地，就像操作本地磁盘一样，非常方便的操作远程文件。

```shell
#server
yum -y install rpcbind nfs-utils

#配置文件
 vim /etc/exports
 /data/nfs/ 192.168.0.0/24(rw,no_root_squash,no_subtree_check,sync)
#刷新配置
exportfs -r
#启动服务，并设置开机启动
systemctl start rpcbind
systemctl start nfs
systemctl enable rpcbind
systemctl enable nfs


#client
yum -y install rpcbind nfs-utils
mount -t nfs 192.168.0.203:/data/nfs /mnt/nfs
```

### 6.3.2 volume支持类型

容器中的文件在磁盘上是临时存放的，这给在容器中运行较重要的应用带来一些问题。 当容器崩溃或停止时会出现一个问题。此时容器状态未保存， 因此在容器生命周期内创建或修改的所有文件都将丢失。 在崩溃期间，kubelet 会以干净的状态重新启动容器。 当多个容器在一个 Pod 中运行并且需要共享文件时，会出现另一个问题。 跨所有容器设置和访问共享文件系统具有一定的挑战性。

#### 1. emptyDir

同pod不同容器之前数据共享。

特性：

```yaml
emptyDir:
  sizeLimit: 500Mi #来限制 emptyDir 卷的存储容量
  medium： Memory #挂载 tmpfs（基于 RAM 的文件系统）,吃容器的内存消耗
```



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-volume
  name: nginx-volume
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-cm
  template:
    metadata:
      labels:
        app: nginx-cm
    spec:
      containers:
      - image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
        name: nginx-web
        volumeMounts:
          - name: my-dir
            mountPath: /tmp/config
      - image: 192.168.0.203:8089/java-demo/tomcat
        name: tomcat-web
        volumeMounts:
          - name: my-dir
            mountPath: /mnt/config
      volumes:
      - name: my-dir
        emptyDir: {}
      imagePullSecrets:
      - name: docker-harbor
```





#### 2. hostPath

HostPath 卷存在许多安全风险，最佳做法是尽可能避免使用 HostPath。 当必须使用 HostPath 卷时，它的范围应仅限于所需的文件或目录，并以只读方式挂载。

| 取值                | 行为                                                         |
| :------------------ | :----------------------------------------------------------- |
|                     | 空字符串（默认）用于向后兼容，这意味着在安装 hostPath 卷之前不会执行任何检查。 |
| `DirectoryOrCreate` | 如果在给定路径上什么都不存在，那么将根据需要创建空目录，权限设置为 0755，具有与 kubelet 相同的组和属主信息。 |
| `Directory`         | 在给定路径上必须存在的目录。                                 |
| `FileOrCreate`      | 如果在给定路径上什么都不存在，那么将在那里根据需要创建空文件，权限设置为 0644，具有与 kubelet 相同的组和所有权。 |
| `File`              | 在给定路径上必须存在的文件。                                 |
| `Socket`            | 在给定路径上必须存在的 UNIX 套接字。                         |
| `CharDevice`        | 在给定路径上必须存在的字符设备。                             |
| `BlockDevice`       | 在给定路径上必须存在的块设备。                               |



```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
      readOnly: true #设置挂载目录只读
  volumes:
  - name: test-volume
    hostPath:
      # 宿主机上目录位置
      path: /data
      # 此字段为可选
      type: Directory
```

#### 3. nfs

`nfs` 卷能将 NFS (网络文件系统) 挂载到你的 Pod 中。 不像 `emptyDir` 那样会在删除 Pod 的同时也会被删除，`nfs` 卷的内容在删除 Pod 时会被保存，卷只是被卸载。 这意味着 `nfs` 卷可以被预先填充数据，并且这些数据可以在 Pod 之间共享。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-volume
  name: nginx-volume
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-cm
  template:
    metadata:
      labels:
        app: nginx-cm
    spec:
      containers:
      - image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12-alpine
        name: nginx-web
        volumeMounts:
          - name: my-nfs
            mountPath: /tmp/config
      - image: 192.168.0.203:8089/java-demo/tomcat
        name: tomcat-web
        volumeMounts:
          - name: my-nfs
            mountPath: /mnt/conf
      volumes:
      - name: my-nfs
        nfs:
          server: 192.168.0.203
          path: /data/nfs
          readOnly: true
      imagePullSecrets:
      - name: docker-harbor
```

### 6.3.3 pv/pvc持久卷



#### 1. volume缺点

* 数据卷不被挂载了，数据回收的实现
* 只读挂载的实现
* 单节点挂载实现
* 存储空间使用限制实现



#### 2. 持久化存储方式

<img src="C:\Users\Mark\AppData\Roaming\Typora\typora-user-images\image-20240412100935362.png" alt="image-20240412100935362"  />



#### 3. PV回收策略

* `保留（Retain)`

回收策略 `Retain` 使得用户可以`手动回收资源`。当 PersistentVolumeClaim 对象被删除时，PersistentVolume 卷仍然存在，对应的数据卷被视为"已释放（released）"。 由于卷上仍然存在这前一申领人的数据，该卷还不能用于其他申领。 管理员可以通过下面的步骤来手动回收该卷：

1. 删除 PersistentVolume 对象。与之相关的、位于外部基础设施中的存储资产在 PV 删除之后仍然存在。

2. 根据情况，手动清除所关联的存储资产上的数据。

3. 手动删除所关联的存储资产。

   

* `删除（Delete）`

对于支持 `Delete` 回收策略的卷插件，删除动作会将 PersistentVolume 对象从 Kubernetes 中移除，同时也会从外部基础设施中移除所关联的存储资产。 动态制备的卷会继承[其 StorageClass 中设置的回收策略](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/#reclaim-policy)， 该策略默认为 `Delete`。管理员需要根据用户的期望来配置 StorageClass； 否则 PV 卷被创建之后必须要被编辑或者修补。 参阅[更改 PV 卷的回收策略](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/change-pv-reclaim-policy/)。



* `回收（Recycle）--废弃`



#### 4. 访问模式

* ReadWriteOnce

卷可以被一个节点以读写方式挂载。 ReadWriteOnce 访问模式仍然可以在同一节点上运行的多个 Pod 访问该卷。RWO

* ReadOnlyMany

卷可以被多个节点以只读方式挂载。ROX

* ReadWriteMany

卷可以被多个节点以读写方式挂载。RWX

https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/#access-modes

### 6.3.4 pv使用

#### 1.创建nfs类型pv

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 1Gi #存储容量
  volumeMode: Filesystem 
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-slow
  nfs:
    path: /data/nfs
    server: 192.168.0.203
```

`volumeMode` 属性设置为 `Filesystem` 的卷会被 Pod **挂载（Mount）** 到某个目录。 如果卷的存储来自某块设备而该设备目前为空，Kuberneretes 会在第一次挂载卷之前在设备上创建文件系统。



#### 2.创建hostpath类型PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-hostpath
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: my-hostpath
  hostPath:
    path: /mnt/data
```



#### 3. PV状态解析

* Available: 可用，没有被PVC绑定的空闲资源
* Bound ：已绑定，已经被PVC绑定
* Release：已释放，PVC被删除，但是资源还未被重新使用
* Failed：失败，自动回收失败



#### 3. 创建PVC并挂载

```yaml
#PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-nfs-pvc
spec:
  accessModes:
    - ReadWriteOnce #与pv保持一致
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi #必须小于等于pv的设置
  storageClassName: nfs-slow #pv的值相同


---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-pv
  name: nginx-pv
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-pv
  template:
    metadata:
      labels:
        app: nginx-pv
    spec:
      containers:
      - image: 192.168.0.203:8089/mark/nginx:1.15.12-alpine
        name: nginx-web
        volumeMounts:
          - name: my-nfs
            mountPath: /usr/share/nginx/html
      volumes:
      - name: my-nfs
        persistentVolumeClaim:
          claimName: my-nfs-pvc
      imagePullSecrets:
      - name: docker-harbor
```

而挂载的操作一般分2个阶段：

- Attach 阶段，这个阶段其实就是连接远程的云硬盘。
- 而第一阶段完成后，就会格式化云硬盘，然后 mount 到前面创建的 Volume 目录下面。这个阶段一般称作 Mount 阶段。

https://47th.subscription-from-chickenfarmer.com/api/v1/client/subscribe?token=b4ccb09718d7cdb6ce361ae91b63afdb

#### 4. PVC创建pending几种原因

PVC pending分析原因

* 存储大小大于PV的值
*  accessModes与PV不一致
* storageClassName值不存在



pod挂载PVC pending原因

* PVC没创建
* PVC和pod不在同一个namespace

###  6.3.5 动态存储



参考存储文档，专门讲解

































































# troubleshoot

## 1.创建pod时请求harbor仓库，无法拉取镜像

* 创建secret 

  ```
  kubectl create secret docker-registry docker-harbor  --docker-server=http://192.168.0.203:8089 --docker-username=makun     --docker-password=Harbor12345  --docker-email=18X060@163.com
  ```

* yaml中配置认证

  ```
  [root@k8s-master01 prd]# cat nginx.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    labels:
      run: nginx
    name: nginx
  spec:
    containers:
    - name: nginx
      image: 192.168.0.203:8089/mark/nginx:latest
      ports:
      - containerPort: 81
    imagePullSecrets:
    - name: docker-harbor
  
  ```

  如何查看haproxy 流量切到的
  
  

## 2. master节点NotReady排查

* tail -100f /var/log/messages

```
node k8s-master03 hasn't been updated for 1h44m16.262431991s. Last Ready is: &NodeCondition{Tdy,Status:Unknown,LastHeartbeatTime:2024-03-28 20:02:37 +0800 CST,LastTransitionTime:2024-03-28 20:03:56 +0800 CST,Reason:NodeStatusUnknown,Message:Kubelet stopped posting node status.,}

```

kubelet获取不了节点信息

* 节点上查看 journalctl -xeu kubelet

```
 Failed to make webhook authenticator request: Post "https://192.168.0.110:8443/apis/authentication.k8s.io/v1/tokenreview
```

访问apiserver失败，尝试访问每台节点的6443端口正常，判断8443代理端口失败或者vip的问题

三个节点灰度重启keepalive, systemctl start keepalived无效

**再灰度重启haproxy服务systemctl restart haproxy，恢复正常！！！**



