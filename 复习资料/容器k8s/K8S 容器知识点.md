## 1、为什么后端面试会问 K8S / 容器

现在很多后端服务都运行在容器和 Kubernetes（K8S）上，面试官通常不是想考你“会不会背 YAML”，而是想看你是否理解：

- 应用是怎么被打包、运行、隔离、发布的
- 服务是怎么在集群里被调度、发现、扩缩容、恢复的
- 你是否具备线上排障、部署、运维协作的基础认知

如果你是后端开发，至少要能把下面这条链路讲顺：

`代码 -> 镜像 -> 容器 -> Pod -> Deployment -> Service -> Ingress`

---

## 2、容器基础

### （1）什么是容器

容器本质上不是虚拟机，它更像是“受隔离限制的进程”。

容器通过 Linux 内核能力实现隔离：

- `Namespace`：做资源视图隔离，比如进程、网络、挂载点、主机名
- `Cgroups`：做资源限制，比如 CPU、内存、IO
- `UnionFS`：做分层文件系统，支持镜像复用

所以容器和虚拟机的区别是：

- 虚拟机隔离的是整套 OS
- 容器隔离的是进程运行环境，共享宿主机内核

容器更轻量，启动更快，资源利用率更高。

### （2）容器和虚拟机区别

| 对比项 | 容器 | 虚拟机 |
|---|---|---|
| 隔离粒度 | 进程级 | 硬件/OS 级 |
| 启动速度 | 秒级甚至毫秒级 | 分钟级 |
| 资源开销 | 小 | 大 |
| 是否共享宿主机内核 | 是 | 否 |
| 适合场景 | 微服务、弹性部署、CI/CD | 强隔离、多 OS 环境 |

### （3）镜像和容器的关系

- `镜像（Image）`：只读模板，包含应用代码、运行时、依赖、启动命令
- `容器（Container）`：镜像运行起来后的实例

可以理解为：

- 类似 `Java class` 和 `Java object`
- 镜像是“安装包/模板”
- 容器是“运行中的进程实例”

### （4）Docker 常见架构

Docker 常见组件：

- `Docker Client`：执行 `docker build`、`docker run` 的客户端
- `Docker Daemon`：真正负责构建、拉取镜像、启动容器
- `Image Registry`：镜像仓库，比如 Docker Hub、Harbor

常见流程：

1. 写 `Dockerfile`
2. 执行 `docker build` 构建镜像
3. 执行 `docker push` 推送到镜像仓库
4. 机器或 K8S 节点执行 `docker pull`
5. 基于镜像启动容器

### （5）Dockerfile 常见指令

- `FROM`：基础镜像
- `WORKDIR`：工作目录
- `COPY` / `ADD`：复制文件
- `RUN`：构建时执行命令
- `ENV`：环境变量
- `EXPOSE`：声明端口
- `CMD`：默认启动命令
- `ENTRYPOINT`：固定入口命令

面试常问：

- `RUN` 是构建镜像时执行
- `CMD` 是容器启动时执行
- `ENTRYPOINT` 一般用于固定主命令，`CMD` 用于补充默认参数

### （6）容器为什么适合微服务

- 环境一致，解决“我本地能跑”
- 启动快，便于弹性扩缩容
- 镜像可复用，方便持续交付
- 隔离性比普通进程强
- 天然适合 K8S 调度和编排

---

## 3、Kubernetes 是什么

Kubernetes 是一个容器编排平台，用来管理大量容器化应用。

它解决的核心问题：

- 容器运行在哪台机器
- 容器挂了怎么自动重启
- 某个服务怎么扩容到多个副本
- 多个副本怎么做服务发现和负载均衡
- 配置、密钥、存储怎么挂进去
- 怎么滚动发布、回滚、灰度

一句话总结：

`Docker 解决“怎么跑一个容器”，K8S 解决“怎么管理一群容器”。`

---

## 4、K8S 整体架构

### （1）集群角色

K8S 集群通常分为两类节点：

- `Master / Control Plane`：控制平面，负责管理集群
- `Worker Node`：工作节点，负责真正运行 Pod

### （2）Control Plane 核心组件

#### 1）`kube-apiserver`

集群统一入口，所有操作都通过 API Server。

- `kubectl` 调用它
- Controller 调用它
- Scheduler 调用它

它可以理解为 K8S 的“前门”。

#### 2）`etcd`

分布式 KV 存储，保存整个集群状态。

比如：

- 有哪些 Node
- 有哪些 Pod
- Deployment 期望副本数是多少
- Service 如何映射

#### 3）`kube-scheduler`

调度器，决定一个 Pod 应该被调度到哪个 Node 上。

它会考虑：

- 节点资源是否足够
- 污点/容忍度
- 亲和性/反亲和性
- 节点标签

#### 4）`kube-controller-manager`

控制器管理器，负责不断把“实际状态”向“期望状态”靠拢。

例如：

- Deployment 希望有 3 个 Pod
- 实际只有 2 个
- Controller 就会补 1 个

这就是 K8S 最核心的理念：`声明式 + 自动调谐（Reconcile）`

### （3）Node 核心组件

#### 1）`kubelet`

每个节点上的代理，负责：

- 接收 Pod 规范
- 调用容器运行时启动容器
- 上报 Pod 状态

#### 2）`kube-proxy`

负责 Service 的网络转发和负载均衡能力，常基于 iptables 或 IPVS 实现。

#### 3）`Container Runtime`

容器运行时，比如：

- containerd
- CRI-O

过去 Docker 常见，但现在 K8S 更常直接对接符合 `CRI` 的运行时。

---

## 5、K8S 最核心的对象

### （1）Pod

Pod 是 K8S 中最小部署单元。

一个 Pod 里可以有一个或多个容器，这些容器：

- 共享网络命名空间
- 共享存储卷
- 可以通过 `localhost` 通信

通常一个 Pod 放一个主业务容器，必要时加 sidecar 容器。

面试高频点：

- Pod 不是容器，Pod 是容器的封装
- 一个 Pod 内多个容器共享 IP 和端口空间
- Pod 是临时的，随时可能被销毁重建

### （2）ReplicaSet

负责维持指定数量的 Pod 副本。

比如希望 3 个副本，如果死了 1 个，它会补回来。

一般开发不直接操作 ReplicaSet，而是通过 Deployment 管理。

### （3）Deployment

Deployment 用来管理无状态应用，是最常见的部署对象。

它的能力包括：

- 管理 ReplicaSet 和 Pod
- 滚动更新
- 回滚版本
- 声明副本数

你可以这样理解：

- Pod：运行实例
- ReplicaSet：保证副本数
- Deployment：管理副本和发布策略

#### Deployment 常见字段

- `replicas`：副本数
- `selector`：选择哪些 Pod
- `template`：Pod 模板
- `strategy`：发布策略

#### 常见面试问法

问：Deployment 是怎么滚动更新的？

答：它会创建新的 ReplicaSet，逐步拉起新 Pod，同时逐步删除旧 Pod，过程中通过 `maxSurge` 和 `maxUnavailable` 控制发布节奏，确保服务可用。

### （4）Service

Service 用来给一组 Pod 提供稳定访问入口。

因为 Pod IP 不稳定，Pod 重建后 IP 会变，所以不能直接依赖 Pod IP。

Service 提供：

- 稳定虚拟 IP
- 服务发现
- 负载均衡

#### Service 常见类型

#### 1）`ClusterIP`

默认类型，只能集群内部访问。

#### 2）`NodePort`

在每个 Node 暴露一个端口，外部可以通过 `NodeIP:NodePort` 访问。

#### 3）`LoadBalancer`

云环境常用，自动创建云厂商负载均衡器，对外暴露服务。

#### 4）`ExternalName`

把 Service 映射到外部域名。

#### Service 和 Pod 的关系

Service 通过 `label selector` 选中一组 Pod 作为后端。

所以标签系统非常重要。

### （5）Ingress

Ingress 用来做七层流量入口管理，常用于 HTTP/HTTPS 路由。

它能实现：

- 域名路由
- 路径转发
- HTTPS 证书终止
- 部分灰度能力

注意：

- Ingress 只是规则对象
- 真正执行流量转发的是 `Ingress Controller`，比如 Nginx Ingress Controller

### （6）ConfigMap 和 Secret

#### `ConfigMap`

用于存放配置，比如：

- 配置文件
- 环境变量
- 启动参数

#### `Secret`

用于存放敏感信息，比如：

- 数据库密码
- Token
- TLS 证书

注意：

- Secret 默认只是 base64 编码，不等于加密
- 真正安全还要结合 RBAC、etcd 加密、外部密钥系统

### （7）Volume / PV / PVC / StorageClass

容器默认是无状态的，Pod 删除后容器内文件通常会丢失，所以需要存储抽象。

#### `Volume`

Pod 内部挂载卷的抽象，生命周期通常跟 Pod 相关。

#### `PV`

PersistentVolume，集群层面的持久化存储资源。

#### `PVC`

PersistentVolumeClaim，应用对存储的申请。

#### `StorageClass`

定义动态创建存储卷的方式。

面试表达建议：

`PVC 是“我要多大、多快、什么模式的存储”，PV 是“集群里真实存在的存储资源”。`

### （8）Namespace

Namespace 用于做资源隔离和逻辑分组。

常见用途：

- 区分 dev/test/prod
- 区分不同业务线
- 配合 RBAC 做权限控制

---

## 6、其他常见 Workload

### （1）StatefulSet

用于有状态应用，比如：

- MySQL
- Redis
- Kafka

特点：

- Pod 名字稳定
- 网络标识稳定
- 存储绑定稳定
- 支持有序启动和停止

和 Deployment 的区别：

- Deployment 适合无状态
- StatefulSet 适合有状态

### （2）DaemonSet

确保每个 Node 上都运行一个 Pod。

常见用途：

- 日志采集
- 节点监控
- CNI / CSI 组件

### （3）Job

一次性任务，跑完即结束。

比如：

- 数据修复
- 批处理
- 离线脚本

### （4）CronJob

定时任务版 Job。

比如：

- 每天凌晨备份
- 定时清理日志

---

## 7、K8S 的声明式思想

K8S 非常强调“期望状态”。

你提交的 YAML 不是“执行脚本”，而是在声明：

- 我希望有 3 个副本
- 我希望镜像版本是 v2
- 我希望容器监听 8080

控制器会不断对比：

- 期望状态（spec）
- 实际状态（status）

如果不一致，就自动修正。

这套模式叫：

- `Declarative API`：声明式 API
- `Reconcile Loop`：调谐循环

---

## 8、CRD 和 Operator

### （1）CRD 是什么

CRD 全称 `CustomResourceDefinition`，即自定义资源定义。

K8S 默认内置很多资源：

- Pod
- Deployment
- Service

但如果你想扩展 K8S，让它认识新的资源类型，比如：

- `RedisCluster`
- `MysqlCluster`
- `Elasticsearch`

就可以定义一个 CRD。

简单理解：

`CRD = 让 K8S 学会一种新的资源类型`

### （2）Operator 是什么

Operator 本质上是“懂某类业务逻辑的 Controller”。

它通常做两件事：

1. 定义 CRD
2. 监听自定义资源变化，并自动执行运维动作

例如 Redis Operator：

- 你提交一个 `RedisCluster` 资源
- Operator 看到后自动创建 StatefulSet、Service、PVC
- 节点故障时自动做修复、主从切换、扩缩容

### （3）CRD 和 Operator 的关系

- `CRD` 负责扩展资源类型
- `Operator` 负责实现对应控制逻辑

面试一句话：

`CRD 解决“定义新对象”，Operator 解决“让这个对象具备自动化运维能力”。`

---

## 9、K8S 网络基础

### （1）每个 Pod 都有独立 IP

K8S 的典型网络模型要求：

- 每个 Pod 有自己的 IP
- Pod 之间可以直接通信
- 不需要 NAT 才能互通

这通常依赖 CNI 插件实现，比如：

- Calico
- Flannel
- Cilium

### （2）Service 为什么存在

因为 Pod 会漂移，IP 不固定。

Service 提供一个稳定入口，把流量转发到后端一组 Pod。

### （3）容器端口、Pod 端口、Service 端口

这块容易混：

- `containerPort`：容器监听端口
- `targetPort`：Service 转发到的目标端口
- `port`：Service 自己暴露的端口
- `nodePort`：Node 对外暴露端口

---

## 10、K8S 调度基础

Scheduler 选节点时常见考虑因素：

- 资源请求 `requests`
- 资源限制 `limits`
- 节点标签 `nodeSelector`
- 亲和性 `affinity`
- 反亲和性 `anti-affinity`
- 污点 `taints`
- 容忍度 `tolerations`

### （1）requests 和 limits

- `requests`：调度时保底资源，决定能不能被调度进去
- `limits`：容器最多能用多少资源

CPU 超限通常会被限流；
内存超限通常可能被 OOM Kill。

### （2）为什么要设置 requests / limits

- 避免资源抢占失控
- 提高调度准确性
- 防止某个容器把节点打爆

---

## 11、探针机制

K8S 常见 3 种探针：

### （1）`livenessProbe`

存活探针。

如果失败，说明容器“活着但坏了”，K8S 会重启容器。

### （2）`readinessProbe`

就绪探针。

如果失败，说明容器暂时不能接流量，Service 会把它摘掉，但不一定重启。

### （3）`startupProbe`

启动探针。

适合启动慢的应用，避免应用还没启动好就被 livenessProbe 杀掉。

面试常问：

问：readiness 和 liveness 的区别？

答：readiness 决定是否接流量，liveness 决定是否该重启。

---

## 12、常见发布方式

### （1）滚动发布

最常见，逐步替换旧 Pod。

优点：

- 平滑
- 对用户影响小

### （2）蓝绿发布

同时保留旧版本和新版本，切流量时整体切换。

优点：

- 回滚快
- 版本隔离清晰

缺点：

- 资源成本高

### （3）金丝雀发布

先让少量流量进入新版本，观察没问题再逐步放量。

优点：

- 风险更低
- 适合线上灰度

---

## 13、常见基础命令

### （1）查看资源

```bash
kubectl get pods
kubectl get deploy
kubectl get svc
kubectl get ingress
kubectl get nodes
kubectl get all -n dev
```

### （2）查看详情

```bash
kubectl describe pod pod-name
kubectl describe deploy deploy-name
kubectl describe node node-name
```

### （3）查看日志

```bash
kubectl logs pod-name
kubectl logs -f pod-name
kubectl logs pod-name -c container-name
```

### （4）进入容器

```bash
kubectl exec -it pod-name -- /bin/sh
kubectl exec -it pod-name -c container-name -- /bin/sh
```

### （5）应用和删除 YAML

```bash
kubectl apply -f app.yaml
kubectl delete -f app.yaml
```

### （6）查看 YAML

```bash
kubectl get deploy app -o yaml
kubectl get pod pod-name -o wide
```

### （7）副本伸缩

```bash
kubectl scale deploy app --replicas=3
```

### （8）滚动更新镜像

```bash
kubectl set image deploy/app app=repo/app:v2
kubectl rollout status deploy/app
kubectl rollout history deploy/app
kubectl rollout undo deploy/app
```

### （9）端口转发

```bash
kubectl port-forward svc/app 8080:80
kubectl port-forward pod/pod-name 8080:8080
```

### （10）查看事件

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## 14、YAML 里最常见的字段

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: demo-app
          image: demo/demo-app:v1
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: dev
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"
```

重点理解：

- `apiVersion`：资源版本
- `kind`：资源类型
- `metadata`：名字、标签、命名空间
- `spec`：你声明的期望状态

---

## 15、常规使用场景

### （1）部署一个 Spring Boot 服务

典型流程：

1. 打包 Jar
2. 写 Dockerfile 构建镜像
3. 推送镜像到仓库
4. 编写 Deployment YAML
5. 编写 Service YAML
6. 如果需要对外访问，再配 Ingress

### （2）注入配置

常见做法：

- 配置放 ConfigMap
- 密码放 Secret
- 通过环境变量或挂载文件注入容器

### （3）应用扩容

常见方式：

- 手动改 replicas
- 使用 HPA 自动扩容

### （4）应用升级

常见方式：

- 更新镜像 tag
- 触发 Deployment 滚动更新
- 观察 Pod、日志、探针、Service 是否正常

---

## 16、HPA 是什么

HPA 全称 Horizontal Pod Autoscaler，水平自动扩缩容。

它会根据指标自动调整 Pod 副本数，比如：

- CPU 使用率
- 内存使用率
- 自定义业务指标

例如：

- 当前 3 个 Pod
- CPU 使用率持续过高
- HPA 自动扩成 5 个 Pod

面试注意：

- HPA 扩的是 Pod 数量，不是单 Pod 规格
- VPA 才更偏向调整单 Pod 资源

---

## 17、K8S 常见排障思路

### （1）Pod 起不来

先看：

```bash
kubectl get pods
kubectl describe pod pod-name
kubectl logs pod-name
```

常见原因：

- 镜像拉取失败
- 启动命令错误
- 环境变量缺失
- 探针配置错误
- 资源不足

### （2）服务访问不通

排查链路：

1. Pod 是否 Running
2. readinessProbe 是否通过
3. Service selector 是否选中了正确 Pod
4. Endpoints 是否存在
5. 端口映射是否正确
6. Ingress / LB / 防火墙 是否有问题

### （3）Pod 被频繁重启

常见原因：

- OOM
- livenessProbe 失败
- 业务进程异常退出
- 依赖服务不可用导致应用启动失败

### （4）发布后接口报错

重点看：

- 新旧版本兼容性
- 数据库变更是否向后兼容
- 配置是否同步
- readiness 是否过早放流量

---

## 18、后端面试高频问答

### （1）Deployment 和 StatefulSet 的区别

答：

- Deployment 管理无状态应用，Pod 名字和存储都不稳定
- StatefulSet 管理有状态应用，Pod 标识、网络、存储都稳定

### （2）Service 为什么不能省

答：

Pod IP 会变化，而 Service 提供稳定地址和负载均衡能力，客户端不需要感知 Pod 变化。

### （3）为什么 Pod 里一般只放一个主容器

答：

单一职责更清晰，扩缩容粒度更合理，问题定位更简单。多容器通常用于 sidecar，如日志采集、代理、服务网格等。

### （4）readinessProbe 和 livenessProbe 分别解决什么问题

答：

- readiness 解决“服务现在能不能接流量”
- liveness 解决“进程是不是已经卡死，需要重启”

### （5）CRD / Operator 的价值是什么

答：

它们把复杂中间件的运维经验平台化。开发或运维只需要声明一个高层资源，比如 `RedisCluster`，Operator 自动完成部署、扩缩容、故障恢复等动作。

### （6）K8S 的核心思想是什么

答：

核心是声明式管理和控制器调谐。用户描述期望状态，系统持续把实际状态修正到期望状态。

### （7）容器和虚拟机最大的区别是什么

答：

容器共享宿主机内核，隔离的是进程级运行环境；虚拟机隔离的是完整 OS，所以容器更轻量。

---

## 19、面试回答模板

如果面试官问“你理解的 K8S 是什么”，可以按这个结构回答：

`K8S 是一个容器编排平台，主要解决容器化应用的部署、调度、扩缩容、服务发现和故障恢复问题。对开发来说，最核心的对象是 Pod、Deployment、Service 和 Ingress。Pod 是最小部署单元，Deployment 负责无状态应用的副本管理和滚动发布，Service 提供稳定访问入口，Ingress 负责七层流量路由。K8S 本质上是声明式系统，开发提交 YAML 描述期望状态，控制器不断调谐实际状态。对于更复杂的场景，还可以通过 CRD 和 Operator 扩展 K8S 的资源类型和自动化运维能力。`

如果面试官问“你线上怎么排查 K8S 问题”，可以这样回答：

`我一般按资源链路排查：先看 Pod 是否正常 Running，再看 describe 和日志；如果是服务不通，就继续看 readiness、Service selector、Endpoints、端口映射、Ingress 规则；如果有重启，再重点排查探针、OOM、配置缺失和依赖服务异常。`

---

## 20、速记版

### （1）最小知识闭环

- 容器 = 受隔离的进程
- 镜像 = 容器运行模板
- K8S = 容器编排平台
- Pod = 最小部署单元
- Deployment = 管无状态发布
- Service = 稳定访问入口
- Ingress = HTTP/HTTPS 流量入口
- ConfigMap/Secret = 配置与密钥
- PV/PVC = 持久化存储
- CRD/Operator = 平台扩展和自动化运维

### （2）最重要的 3 个思想

- 声明式
- 控制器调谐
- 资源对象化

### （3）最常考的 5 个区别

- 容器 vs 虚拟机
- Pod vs 容器
- Deployment vs StatefulSet
- Service vs Ingress
- readinessProbe vs livenessProbe

---

## 21、建议你的复习顺序

如果你是短时间恶补，建议按下面顺序背：

1. 容器是什么，和虚拟机区别
2. K8S 架构，至少知道 apiserver、scheduler、controller-manager、kubelet
3. Pod、Deployment、Service、Ingress
4. ConfigMap、Secret、PV/PVC
5. StatefulSet、DaemonSet、Job、CronJob
6. 探针、滚动更新、HPA
7. CRD、Operator
8. 常用 kubectl 命令
9. 常见排障思路

这样基本能覆盖大部分后端面试场景。
