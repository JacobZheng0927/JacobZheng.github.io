# K8S 从入门到放弃

[toc]

## 1.简介

Kubernetes是一个开源的，用于管理云平台中多个主机上的容器化的应用，Kubernetes的目标是让部署容器化的应用简单并且高效（powerful）,Kubernetes提供了应用部署，规划，更新，维护的一种机制。

## 2.架构

### 2.1 整体架构

![architecture](https://markdown-image-upload.oss-cn-beijing.aliyuncs.com/img/architecture.png)

- **Kubernetes 主控组件（Master）** 

  ![Kubernetes master架构示意图](https://markdown-image-upload.oss-cn-beijing.aliyuncs.com/img/Kubernetes%20master%E6%9E%B6%E6%9E%84%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

  包含三个进程，都运行在集群中的某个节上，主控组件通常这个节点被称为 master 节点。

  - [ApiServer](https://kubernetes.io/docs/admin/kube-apiserver/)

    提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；

  - [Controller-manager](https://kubernetes.io/docs/admin/kube-controller-manager/) 

    负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；

  - [Scheduler](https://kubernetes.io/docs/admin/kube-scheduler/)

    负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；

  - etcd

    保存了整个集群的状态

  Kubernetes master 节点负责维护集群的目标状态。当你要与 Kubernetes 通信时，使用如 `kubectl` 的命令行工具，就可以直接与 Kubernetes master 节点进行通信。

  > “master” 是指管理集群状态的一组进程的集合。通常这些进程都跑在集群中一个单独的节点上，并且这个节点被称为 master 节点。master 节点也可以扩展副本数，来获取更好的可用性及冗余。

- Kubernetes Node 节点

  ![kubernetes node架构示意图](https://markdown-image-upload.oss-cn-beijing.aliyuncs.com/img/kubernetes%20node%E6%9E%B6%E6%9E%84%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

  集群中的 node 节点（虚拟机、物理机等等）都是用来运行你的应用和云工作流的机器。Kubernetes master 节点控制所有 node 节点；你很少需要和 node 节点进行直接通信。每个node都运行两个进程：

  - **[kubelet](https://kubernetes.io/docs/admin/kubelet/)**
  
    和 master 节点进行通信,负责维护容器的生命周期，同时也负责Volume（CSI）和网络（CNI）的管理；
  
  - **[kube-proxy](https://kubernetes.io/docs/admin/kube-proxy/)**
  
    一种网络代理，将 Kubernetes 的网络服务代理到每个节点上，负责为Service提供cluster内部的服务发现和负载均衡；



---

Kubernetes的架构设计以及组件之间的通信协议

![NeatReader-1590555572397](https://markdown-image-upload.oss-cn-beijing.aliyuncs.com/img/NeatReader-1590555572397.png)

### 2.2 分层架构

Kubernetes设计理念和功能其实就是一个类似Linux的分层架构

![Kubernetes分层架构示意图](https://markdown-image-upload.oss-cn-beijing.aliyuncs.com/img/Kubernetes%E5%88%86%E5%B1%82%E6%9E%B6%E6%9E%84%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

自顶向下分别是：

- 云原生生态：在接口层之上的庞大容器集群管理调度的生态系统，可以划分为两个范畴
  - Kubernetes外部：日志、监控、配置管理、CI/CD、Workflow、FaaS、OTS应用、ChatOps、GitOps、SecOps等
  - Kubernetes内部：CRI、CNI、CSI、镜像仓库、Cloud Provider、集群自身的配置和管理
- 接口层：kubectl命令行工具、客户端SDK以及集群联邦
- 管理层：系统度量（如基础设施、容器和网络的度量），自动化（如自动扩展、动态Provision等）以及策略管理（RBAC、Quota、PSP、NetworkPolicy等）
- 应用层：部署（无状态应用、有状态应用、批处理任务、集群应用等）和路由（服务发现、DNS解析等）
- 核心层：Kubernetes最核心的功能，对外提供API构建高层的应用，对内提供插件式应用执行环境

## 3.Kubernetes网络

> Kubernetes本身并不提供网络功能，只是把网络接口开放出来，通过插件的形式实现。

在本地单台机器上运行docker容器的话会注意到所有容器都会处在`docker0`网桥自动分配的一个网络IP段内（172.17.0.1/16）。该值可以通过docker启动参数`--bip`来设置。这样所有本地的所有的容器都拥有了一个IP地址，而且还是在一个网段内彼此就可以互相通信了。

Kubernetes管理的是集群，Kubernetes中的网络要解决的核心问题就是**每台主机的IP地址网段划分，以及单个容器的IP地址分配**。

- 保证每个Pod拥有一个集群内唯一的IP地址
- 保证不同节点的IP地址划分不会重复
- 保证跨节点的Pod可以互相通信
- 保证不同节点的Pod可以与跨节点的主机互相通信



## 3.Kubernetes 对象

### **3.1 理解kubernetes中的对象**

在 Kubernetes 系统中，*Kubernetes 对象* 是持久化的条目。Kubernetes 使用这些条目去表示整个集群的状态。特别地，它们描述了如下信息：

- 什么容器化应用在运行（以及在哪个 Node 上）
- 可以被应用使用的资源
- 关于应用如何表现的策略，比如重启策略、升级策略，以及容错策略

> 用大白话讲，就是kubernetes描述一个对象=让kubernetes创建并维护这个对象，这个对象就是系统的预期结果，kubernets需要持续工作去维护

后续需要使用kubernetes API与Kubernetes对象交互，例如使用`kubectl`命令时，CLI会调用必要的Kubernetes API。

---

### **3.2 描述Kubernetes对象**

对象都有3大类属性：`元数据metadata`、`规范spec`和`状态status`。

- 元数据

  每个API对象的元数据至少包括以下三个内容

  - namespace
  - name
  - uid

- 规范Spec

  Spec 描述了对象的期望状态——希望对象具有的特征

- 状态Status

  Status描述了系统实际当前达到的状态

Kubernetes需要一直工作保持Status和Spec匹配

> 当使用 Kubernetes API 创建对象时（或者直接创建，或者基于`kubectl`），API 请求必须在请求体中包含 JSON 格式的信息。**更常用的是，需要在 .yaml 文件中为 kubectl 提供这些信息**。 `kubectl` 在执行 API 请求时，将这些信息转换成 JSON 格式。

以下为一个Kubernetes Deployment的必须字段和对象Spec

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

```bash
kubectl create -f nginx-deployment.yaml --record

#输出如下
deployment "nginx-deployment" created
```



---

### **3.3 详解Kubernetes对象**

将 Kubernetes 对象简单的分类为以下几种资源对象，这些对象都可以在 yaml 文件中作为一种 API 类型来配置。

| 类别     | 名称                                                         |
| :------- | ------------------------------------------------------------ |
| 资源对象 | Pod、ReplicaSet、ReplicationController、Deployment、StatefulSet、DaemonSet、Job、CronJob、HorizontalPodAutoscaling、Node、Namespace、Service、Ingress、Label、CustomResourceDefinition |
| 存储对象 | Volume、PersistentVolume、Secret、ConfigMap                  |
| 策略对象 | SecurityContext、ResourceQuota、LimitRange                   |
| 身份对象 | ServiceAccount、Role、ClusterRole                            |

---

#### 基本资源管理

##### Pod 

> `Pod` 是最小可部署的 Kubernetes 对象模型。
>
> Kubernetes集群中所有业务类型的基础，可以看作运行在K8集群中的小机器人

Pod 是 Kubernetes 应用程序的基本执行单元，即它是 Kubernetes 对象模型中创建或部署的最小和最简单的单元。Pod 表示在 集群 上运行的进程。

Pod 封装了应用程序容器（或者在某些情况下封装多个容器）、存储资源、唯一网络 IP 以及控制容器应该如何运行的选项。 Pod 表示部署单元：Kubernetes 中应用程序的单个实例，它可能由单个 容器 或少量紧密耦合并共享资源的容器组成。

> Docker 是 Kubernetes Pod 中最常用的容器运行时，但 Pod 也能支持其他的容器运行时。

Kubernetes 集群中的 Pod 可被用于以下两个主要用途：

- **运行单个容器的 Pod**。“每个Pod中一个容器”的模式是最常见的用法；在这种使用方式中，你可以把Pod想象成是单个容器的封装，kuberentes管理的是Pod而不是直接管理容器。
- **运行多个协同工作的容器的 Pod**。 一个Pod中也可以同时封装几个需要紧密耦合互相协作的容器，它们之间共享资源。这些在同一个Pod中的容器可以互相协作成为一个service单位——一个容器共享文件，另一个“sidecar”容器来更新这些文件。Pod将这些容器的存储资源作为一个实体来管理。

如果要平行拓展应用，应运行多个pod，被称为**replication**

**为什么不直接在一个容器中运行多个应用程序呢？**

1. 透明。让Pod中的容器对基础设施可见，以便基础设施能够为这些容器提供服务，例如进程管理和资源监控。这可以为用户带来极大的便利。
2. 解耦软件依赖。每个容器都可以进行版本管理，独立的编译和发布。未来kubernetes甚至可能支持单个容器的在线升级。
3. 使用方便。用户不必运行自己的进程管理器，还要担心错误信号传播等。
4. 效率。因为由基础架构提供更多的职责，所以容器可以变得更加轻量级。



Pod 为其组成容器提供了两种共享资源：*网络* 和 *存储*。

**网络**

每个 Pod 分配一个唯一的 IP 地址。 Pod 中的每个容器共享网络命名空间，包括 IP 地址和网络端口。 *Pod 内的容器* 可以使用 `localhost` 互相通信。 当 Pod 中的容器与 *Pod 之外* 的实体通信时，它们必须协调如何使用共享的网络资源（例如端口）。

**存储**

一个 Pod 可以指定一组共享存储卷。 Pod 中的所有容器都可以访问共享卷，允许这些容器共享数据。 卷还允许 Pod 中的持久数据保留下来，以防容器重启后文件丢失。



**Pod的持久性**

Pod在设计支持就不是作为持久化实体的。在调度失败、节点故障、缺少资源或者节点维护的状态下都会死掉会被驱逐。

当Pod被创建后（不论是由你直接创建还是被其他Controller），都会被Kuberentes调度到集群的Node上。直到Pod的进程终止、被删掉、因为缺少资源而被驱逐、或者Node故障之前这个Pod都会一直保持在那个Node上。

Pod不会自愈。如果Pod运行的Node故障，或者是调度器本身故障，这个Pod就会被删除。同样的，如果Pod所在Node缺少资源或者Pod处于维护状态，Pod也会被驱逐。Kubernetes使用更高级的称为Controller的抽象层，来管理Pod实例。虽然可以直接使用Pod，但是在Kubernetes中通常是使用Controller来管理Pod的。



**Pod的终止**

因为Pod作为在集群的节点上运行的进程，所以在不再需要的时候能够优雅的终止掉是十分必要的（比起使用发送KILL信号这种暴力的方式）。用户需要能够放松删除请求，并且知道它们何时会被终止，是否被正确的删除。用户想终止程序时发送删除pod的请求，在pod可以被强制删除前会有一个宽限期，会发送一个TERM请求到每个容器的主进程。一旦超时，将向主进程发送KILL信号并从API server中删除。如果kubelet或者container manager在等待进程终止的过程中重启，在重启后仍然会重试完整的宽限期。

示例流程如下：

1. 用户发送删除pod的命令，默认宽限期是30秒；
2. 在Pod超过该宽限期后API server就会更新Pod的状态为“dead”；
3. 在客户端命令行上显示的Pod状态为“terminating”；
4. 跟第三步同时，当kubelet发现pod被标记为“terminating”状态时，开始停止pod进程：
   1. 如果在pod中定义了preStop hook，在停止pod前会被调用。如果在宽限期过后，preStop hook依然在运行，第二步会再增加2秒的宽限期；
   2. 向Pod中的进程发送TERM信号；
5. 跟第三步同时，该Pod将从该service的端点列表中删除，不再是replication controller的一部分。关闭的慢的pod将继续处理load balancer转发的流量；
6. 过了宽限期后，将向Pod中依然运行的进程发送SIGKILL信号而杀掉进程。
7. Kublete会在API server中完成Pod的的删除，通过将优雅周期设置为0（立即删除）。Pod在API中消失，并且在客户端也不可见。

删除宽限期默认是30秒。 `kubectl delete`命令支持 `—grace-period=` 选项，允许用户设置自己的宽限期。如果设置为0将强制删除pod。在kubectl>=1.5版本的命令中，你必须同时使用 `--force` 和 `--grace-period=0` 来强制删除pod。

**强制删除Pod**

Pod的强制删除是通过在集群和etcd中将其定义为删除状态。当执行强制删除命令时，API server不会等待该pod所运行在节点上的kubelet确认，就会立即将该pod从API server中移除，这时就可以创建跟原pod同名的pod了。这时，在节点上的pod会被立即设置为terminating状态，不过在被强制删除之前依然有一小段优雅删除周期。

强制删除对于某些pod具有潜在危险性，请谨慎使用。使用StatefulSet pod的情况下，请参考删除StatefulSet中的pod文章。



**init容器**

> 这是一种专用的容器，在应用程序容器启动之前运行，用来包含一些应用镜像中不存在的实用工具或安装脚本。

- Init 容器总是运行到成功完成为止。
- 每个 Init 容器都必须在下一个 Init 容器启动之前成功完成。

- 如果 Pod 的 Init 容器失败，Kubernetes 会不断地重启该 Pod，直到 Init 容器成功为止。然而，如果 Pod 对应的 `restartPolicy` 为 Never，它不会重新启动。

-  Init 容器不支持 Readiness Probe，因为它们必须在 Pod 就绪之前运行完成。

[查看init容器yaml示例](#使用init容器)



**pause容器**

。。。

pod部分过于冗长了，理解了概念后续详细内容可以实际使用中再探讨。

![kubernetes-pod-cheatsheet](https://markdown-image-upload.oss-cn-beijing.aliyuncs.com/img/kubernetes-pod-cheatsheet.png)



---

#### 集群资源管理

##### Node

> Node是kubernetes集群的工作节点，可以是物理机也可以是虚拟机。

Node包括如下状态信息：

- Address
  - HostName：可以被kubelet中的`--hostname-override`参数替代。
  - ExternalIP：可以被集群外部路由到的IP地址。
  - InternalIP：集群内部使用的IP，集群外部无法访问。
- Condition
  - OutOfDisk：磁盘空间不足时为`True`
  - Ready：Node controller 40秒内没有收到node的状态报告为`Unknown`，健康为`True`，否则为`False`。
  - MemoryPressure：当node没有内存压力时为`True`，否则为`False`。
  - DiskPressure：当node没有磁盘压力时为`True`，否则为`False`。
- Capacity
  - CPU
  - 内存
  - 可运行的最大Pod个数
- Info：节点的一些版本信息，如OS、kubernetes、docker等

node相关命令详见[node命令](#Node管理)



---

##### Namespace



**哪些情况下适合使用多个namespace**

因为namespace可以提供独立的命名空间，因此可以实现部分的环境隔离。当你的项目和人员众多的时候可以考虑根据项目属性，例如生产、测试、开发划分不同的namespace。

命名空间为名称提供了一个范围。资源的名称需要在命名空间内是唯一的，但不能跨命名空间。命名空间不能相互嵌套，每个 Kubernetes 资源只能在一个命名空间中。



**获取集群中有哪些namespace**

```bash
kubectl get ns
```

集群中默认会有`default`和`kube-system`这两个namespace。

`default`用于默认部署用户的普通应用

`kube-system`用于部署为整个集群提供服务的应用，例如`kubedns`、`heapseter`、`EFK`等

在执行`kubectl`命令时可以使用`-n`指定操作的namespace。



---

##### Label

> Label能够将组织架构映射到系统架构上

Label key的组成：

- 不得超过63个字符
- 可以使用前缀，使用/分隔，前缀必须是DNS子域，不得超过253个字符，系统中的自动化组件创建的label必须指定前缀，`kubernetes.io/`由kubernetes保留
- 起始必须是字母（大小写都可以）或数字，中间可以有连字符、下划线和点

Label value的组成：

- 不得超过63个字符
- 起始必须是字母（大小写都可以）或数字，中间可以有连字符、下划线和点



打完标签后可以使用Label selector对object进行筛选

Label selector有两种类型：

- *equality-based* ：可以使用`=`、`==`、`!=`操作符，可以使用逗号分隔多个表达式
- *set-based* ：可以使用`in`、`notin`、`!`操作符，另外还可以没有操作符，直接写出某个label的key，表示过滤有某个key的object而不管该key的value是何值，`!`表示没有该label的object

**示例**

```bash
$ kubectl get pods -l environment=production,tier=frontend
$ kubectl get pods -l 'environment in (production),tier in (frontend)'
$ kubectl get pods -l 'environment in (production, qa)'
$ kubectl get pods -l 'environment,environment notin (frontend)'
```

[Label selector示例](#在API object中设置label selector)



---

##### Annotation

Annotation，顾名思义，就是注解。Annotation可以将Kubernetes资源对象关联到任意的非标识性元数据。使用客户端（如工具和库）可以检索到这些元数据。













- 

- 

- Service

  Kubernetes `Service` 定义了这样一种抽象：逻辑上的一组 `Pod`，一种可以访问它们的策略 —— 通常称为微服务。 这一组 `Pod` 能够被 `Service` 访问到，通常是通过`selector`实现的。

  举个例子，考虑一个图片处理 backend，它运行了3个副本。这些副本是可互换的 —— frontend 不需要关心它们调用了哪个 backend 副本。 然而组成这一组 backend 程序的 `Pod` 实际上可能会发生变化，frontend 客户端不应该也没必要知道，而且也不需要跟踪这一组 backend 的状态。 `Service` 定义的抽象能够解耦这种关联。

- Volume

  Kubernetes 卷具有明确的生命周期——与包裹它的 Pod 相同。 因此，卷比 Pod 中运行的任何容器的存活期都长，在容器重新启动时数据也会得到保留。 当然，当一个 Pod 不再存在时，卷也将不再存在。也许更重要的是，Kubernetes 可以支持许多类型的卷，Pod 也能同时使用任意数量的卷。

  卷的核心是包含一些数据的目录，Pod 中的容器可以访问该目录。 特定的卷类型可以决定这个目录如何形成的，并能决定它支持何种介质，以及目录中存放什么内容。

- API：Kubernetes集群中的管理操作单元

  每个API对象都有3大类属性：`元数据metadata`、`规范spec`和`状态status`。

  - 元数据

    每个API对象的元数据至少包括以下三个内容

    - namespace
    - name
    - uid

  - 规范

    各种标签来标识和匹配不同的对象

  - 状态

    系统实际当前达到的状态

Kubernetes 也包含大量的被称作 Controller 的高级抽象。控制器基于基本对象构建并提供额外的功能和方便使用的特性。具体包括：

- Deployment
- DaemonSet
- StatefulSet
- ReplicaSet
- Job





## 4.单机k8s安装

1.

## 8.命令速查

### Node管理

获取目前node列表

```
kubectl get node
```

禁止pod调度到该节点上

```bash
kubectl cordon <node>
```

驱逐该节点上的所有pod

```bash
kubectl drain <node>
```

### Pod管理

查看所有的namespace下的容器(包括system的)

```
kubectl get po --all-namespace
```

查看6小时内的日志

```
kubectl logs --since 6h xxx
```



## 9.yaml示例

### 使用init容器

```yaml
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

---

### 在API object中设置label selector

在`service`、`replicationcontroller`等object中有对pod的label selector，使用方法只能使用等于操作，例如：

```yaml
selector:
    component: redis
```

在`Job`、`Deployment`、`ReplicaSet`和`DaemonSet`这些object中，支持*set-based*的过滤，例如：

```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```

另外在node affinity和pod affinity中的label selector的语法又有些许不同，示例如下：

```yaml
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
```