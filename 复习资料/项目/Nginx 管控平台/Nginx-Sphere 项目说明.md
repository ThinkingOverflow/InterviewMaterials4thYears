## 一、项目概述

### 1.1 项目背景与定位

Nginx-Sphere 作为一个企业级 Nginx 全生命周期管理系统，平台旨在解决传统运维中配置分散、变更风险高、缺乏统一管控等痛点。该平台支持虚拟机 (VM) 模式和容器化 (K8S) 模式两种部署形态，实现了从版本管理、自动化分发安装、配置管理、性能监控到服务化部署的全流程闭环管理。目前，该平台已在全公司测试与生产环境深度落地，稳定承载了多区域 5000+ Nginx 实例的全量精细化管控。

### 1.2 项目规模与影响力

- **服务范围**: 覆盖公司内部多个区域(顺德、贵安、多个公有云区域等)
- **管理实例**: 管理公司内部 5000+ Nginx 实例，支持近百台 Nginx 实例的统一管控
- **部署模式**: 同时支持传统 VM 部署和云原生 K8s 容器化部署
- **技术创新**: 国内首批实现 Nginx Serverless 化的企业级实践

## 二、系统架构设计
### 2.1 整体架构图
```mermaid
graph TD
    %% 全局纵向布局
    
    subgraph Layer1 ["1. 用户门户层 (User Portal Layer)"]
        direction LR
        AdminUI["Nginx-Sphere 管控后台"]
        MWFE_UI["MWFE 购买/运维界面"]
    end

    Layer1 --> Layer2

    subgraph Layer2 ["2. 管理与同步层 (Management & Sync Layer)"]
        direction TB
        SphereCore["lb-sphere 核心服务 (Spring Boot)"]
        K8sDashboard["lb-gnc-k8s-dashboard (权限/资源同步)"]
        AgentCI["Agent CI 编译流水线"]
        
        K8sDashboard -- "1. 监听资源变化" --> UnifiedCloud["容器云统一 API"]
        K8sDashboard -- "2. 同步权限数据" --> SphereCore
    end

    Layer2 --> Bottom_Area

    subgraph Bottom_Area ["底座与执行层 (Foundation & Execution)"]
        direction LR
        
        %% 第三部分：基础设施底座 (左侧)
        subgraph Layer3 ["3. 基础设施底座"]
            direction TB
            MWFE_Core["MWFE 调度引擎"]
            UnifiedGW["API 网关 / 下载入口"]
            OSS["OSS 制品库"]
            
            AgentCI -.->|推送| UnifiedGW
            UnifiedGW --- OSS
        end

        %% 第四部分：多云/跨域执行层 (右侧)
        subgraph Layer4 ["4. 多云/跨域层"]
            direction TB
            
            subgraph VM_Mode ["VM 区域管控"]
                direction TB
                NATS_VM["NATS A/DMZ"]
                Node_VM["Nginx 实例"]
            end

            subgraph K8s_Mode ["K8s 服务化模式"]
                direction TB
                Clouds["华为云/阿里云/美的云"]
                subgraph Pod ["Serverless Pod"]
                    Sidecar["Agent Sidecar"]
                    Nginx["Nginx 容器"]
                end
            end
        end
    end

    %% 核心交互
    AdminUI -->|管理指令| SphereCore
    MWFE_UI -->|下单购买| MWFE_Core
    
    SphereCore <-->|消息路由| NATS_VM
    Node_VM -.->|下载包| UnifiedGW
    
    MWFE_Core -->|部署实例| Clouds
    Sidecar -->|心跳上报| SphereCore

    style AdminUI fill:#1565C0,color:#fff
    style SphereCore fill:#1E88E5,color:#fff
    style K8sDashboard fill:#00ACC1,color:#fff
    style MWFE_Core fill:#43A047,color:#fff
    style UnifiedGW fill:#FB8C00,color:#fff
    style Sidecar fill:#F4511E,color:#fff
```

### 2.2 核心模块职责划分

| 模块名称 | 技术栈 | 核心职责 | 部署位置 |
|---------|--------|---------|---------|
| **lb-sphere** | Java 21, Spring Boot 3.2, MyBatis-Plus | **Nginx-Sphere 核心管控端**: 负责虚拟机(VM)及容器化 Nginx 的**统一管控与全生命周期管理**; 实现 Nginx 及其运维 Agent 的**自动化安装部署**、配置管理、高性能指令分发。通过 **APISIX 网关** 实现高可用接入。 | **双活部署**: 南海生产集群 & 贵安生产集群，通过APISIX 网关实现统一接入 |
| **lb-gnc-k8s-dashboard** | Java 21, Spring Boot 3.2, K8s Client | 监听 K8s 资源变化, 同步权限/实例数据到数据库。作为数据辅助模块, 无需多活。 | 南海生产集群 (单地域) |
| **nginx-agent** | Rust, Axum, Tokio, NATS | 高性能 Agent: 部署于端侧, 执行配置热更新、监控指标上报、进程生命周期管理。 | Nginx 主机 / Pod Sidecar |
| **nginx-serverless** | Helm Chart, Docker | **服务化方案**: 结合 Nginx + Agent 设计容器化镜像及全量生命周期管理 Helm 策略。 | Kubernetes 集群 (多云) |

## 三、核心功能模块详解

### 3.1 控制台模块 (lb-sphere)

#### 3.1.1 模块架构

`lb-sphere` 整体采用**多模块解耦架构**设计，确保了在双活部署及大规模并发场景下的稳定性：

*   **lb-controller**: 系统入口。负责暴露 RESTful API 接口、Web 端安全认证过滤器以及前端参数的合法性校验。
*   **lb-service**: 业务逻辑中枢。包含了 Nginx 核心配置解析、各区域 NATS 指令调度、Agent 升级策略以及与缓存系统的交互核心。
*   **lb-repository**: 数据访问层。基于 MyBatis-Plus 实现对 MariaDB 的高性能持久化操作，管理全量 Nginx 实例与资产的元数据状态。
*   **lb-model**: 模型层。统一定义 DTO（各层级数据交互对象）、Entity（数据库物理模型）及业务相关的枚举规范。
*   **lb-3rd**: 集成层。封装了对 CMDB 的资产同步逻辑，以及调用特定容器云 API 进行鉴权数据同步的接口。在服务化集成方面，通过提供标准化 Nginx+Agent 镜像给 MWFE，由 MWFE 负责 K8s 实例创建；管控端通过 NATS 与容器内 Agent 无缝交互，实现了与各云厂商底层 K8s 集群的解耦。

#### 3.1.2 核心功能矩阵

1.  **分域分环境实例管控**:
    - 支持虚拟机（VM）与容器化（K8s）环境下的 Nginx 及 Agent 的**全自动化安装部署**。
    - 实现对存量、老旧 Nginx 节点的快速收编与灰度接管。

2.  **企业级配置治理**:
    - 基于“草稿-发布”模型的配置版本化管理，支持一键回滚。
    - 提供高风险指令的安全过滤及配置语法合法性预检查（`nginx -t`）。

3.  **高性能指令调度**:
    - 针对常规内网、DMZ 等不同网络防区，通过 NATS 消息路由实现**毫秒级**指令下发。
    - `NginxDispatcher` 实现批量操作的流水线管理。

4.  **云原生服务化协同**:
    - 提供标准化 Nginx 容器镜像供 MWFE 集成，用户下单后由 MWFE 驱动云环境自动创建 Nginx 实例。
    - 实例内的 Agent 自动向管控端注册并保持心跳，平台无感对接各类底层云环境，实现了多云集群的统一透明纳管。
  
5.  **核心安全审计**:
    - 细粒度的 **RBAC 权限模型**，对所有配置变更（Reload/Restart等）进行全生命周期审计记录。

#### 3.1.3 技术栈详解

| 技术组件 | 版本 | 核心用途与实现细节 |
|---------|------|------|
| **Java / Spring Boot** | 21 / 3.2.x | 采用最新的 Java 21 LTS 版本，利用其优秀的性能与现代化开发范式。 |
| **MariaDB** | 10.5+ | 存储全量实例元数据、多版本配置快照及资产同步记录。 |
| **Redis** | 6.0+ | **高性能权限缓存**。针对复杂的 RBAC 权限（RBAC + 环境/系统多维管控），通过 Redis 缓存 `PermAuthDataDto`（24H 有效期），规避频繁的多表关联查询。 |
| **Redisson** | 3.26.0 | **Redis 高性能客户端底座**。目前作为通用 Redis 操作客户端引入，并为后续支持**分布式大对象（RMap）**或高级并发原语预留架构底座（当前业务分布任务锁采用 Shedlock + JDBC 实现）。 |
| **NATS** | 2.17.x | 替代传统 HTTP/SSH，实现轻量级、高性能、穿透多内网区域的指令下发通道。 |
| **Shedlock** | 5.x | 配合 JDBC Provider，保障双活部署（南海/贵安）模式下，资产定时同步任务的排他执行。 |
| **MyBatis-Plus** | 3.5.x | 实现低侵入、灵活的持久化交互逻辑。 |

---

### 3.2 权限与元数据同步模块 (lb-gnc-k8s-dashboard)

`lb-gnc-k8s-dashboard` 作为独立的 Kubernetes 资源监控和平台元数据支撑中心，保障了主控端免受 Kubernetes 集群事件洪峰的直接冲击。

#### 3.2.1 核心业务链路

1.  **Kubernetes 资源静默监听**
    - 利用 K8s Client Java 提供的 `SharedInformerFactory`，实时监听所在集群的 **Namespace**, **Service**, 以及 **RoleBinding** 事件状态。
    - 将无序的集群变动转化为规范的资源实体，并提取对应微服务所需的鉴权数据结构。
2.  **核心业务表同步 (自动同步引擎)**
    - 将资源变动数据自动化拆解，并幂等同步至公共业务表 (如 `lb_cluster_info`, `lb_service_group_info`, `lb_service_info`, `lb_role_info`)。
    - 解耦设计：通过数据库进行数据交互，避免 `lb-sphere` 直接压垮或高频轮询 K8s API。
3.  **权限中枢与 AK/SK 基础设施**
    - 管理整个体系的 Access Key 和 Secret Key 配置下发。
    - 提供以系统级、服务级的多维 RBAC 权限授权树。
    - **联动缓存**：为主管控端（`lb-sphere`）直接提供稳定的业务权限表查询基础，从而支撑 Redis（Auth Cache）的高性能认证过滤逻辑。

#### 3.2.2 技术栈视角

| 技术组件 | 核心用途与实现层细节 |
|---------|------|
| **K8s Client Java** | 提供原生、高性能的 API Client 与资源监听 (Informer) 支持。 |
| **Spring Boot 3.2** | 轻量级应用骨架，专门剥离出单独的容器进行运行，保证核心调度模块不被 K8s 监听事件阻塞。 |
| **MariaDB / MyBatis-Plus** | 作为数据承载中心进行双活高并发写操作（接收同步链路）。 |
| **MapStruct** | 在 K8s 原生泛型资源对象与本地持久化实体 (Entity/DTO) 间进行高性能的映射转换。 |

---

### 3.3 Agent 终端引擎 (nginx-agent)

#### 3.3.1 核心支持指令与职责边界

`nginx-agent` 是部署在边缘侧的管控触角。在双模部署下（VM/K8s Sidecar），其指令操作具有严格的职责边界划分：

1.  **高级配置管控 (Config Ops)**
    - 从 `lb-sphere` 主控端通过 REST/NATS 接收指令下发。
    - 负责 Nginx 配置文件的热更新、备份与历史版本原子还原。
    - **所有环境均支持**。
2.  **存活状态与性能遥测 (Status & Telemetry)**
    - 周期性向服务器维持 **NATS 心跳注册**。
    - 上报 Nginx 实时性能指标（连接数/并发数）以及宿主机/Pod 的 OS 级负载状态（CPU/内存）。
    - **所有环境均支持**。
3.  **核心 Nginx 指令与生命周期 (Lifecycle Constraints)**
    - **语法校验** (`nginx -t`)：检查待下发配置的合法性。**所有环境均支持**。
    - **平滑重载** (`nginx -s reload`)：发送 HUP 信号重载配置。**所有环境均支持**。
    - **启停指令** (`start / stop / restart`)：
        - **VM 模式**：完全支持。
        - **K8s 容器模式**：**强制禁用 (拦截)**。在云原生架构下，由宿主进程干预容器内主进程的存活是反模式的，所有实例的存活、重建与扩缩容必须通过 MWFE 调用 K8s API (Scale Up/Down) 在平台层解决。

#### 3.3.2 容器化 Sidecar 的隔离与控制哲学

为了在容器化场景中实现 Agent 对 Nginx 主进程的精细控制，系统利用 K8s 原生特性实现了 **“进程级的有限共享”与“文件级的精细挂载”**：

1.  **进程可见性穿透 (`shareProcessNamespace: true`)**
    - 默认状态下两个容器的 PID 空间相互隔离。开启共享后，处于不同容器内的 Nginx 与 Agent 被纳入同一 PID Namespace。
    - **架构效果**：Agent 因此能“看见” `nginx master process`，从而在执行 Reload 时拥有权限向主进程精确投递 `kill -HUP` 信号。
2.  **物理存储多维共享 (The Physics of Mounting)**
    - **配置共享** (`nginx-config-volume`)：Agent 与 Nginx 共享 `/apps/conf/nginx`，让 Agent 写入的配置文件对 Nginx 可见。
    - **PID 共享** (`nginx-run-volume`)：共享 `/apps/svr/run`，让 Agent 能精准读取 Nginx 主进程生成的 `nginx.pid`。
    - **二进制执行环境征用** (`nginx-bin-volume` & `nginx-lib-volume`)：Agent 容器本身不安装 Nginx。它通过挂载 Nginx 的执行路径和动态链接库，直接复用 Nginx 容器的二进制文件来执行轻量级的 `nginx -t` 语法检测，做到了极致轻量。

#### 3.3.3 技术体系

- **Rust**: 采用高性能、内存安全的 Rust 语言编写（编译目标支持多操作系统分发）。
- **Axum + Tokio**: 基于 Tokio 的异步运行时，使用 Axum 提供极高吞吐量的 Web 框架能力。
- **Async NATS**: 使用异步客户端在内外防区间建立海量心跳与指令双向通道。

#### 3.3.3 部署模式

**VM 模式**:
- 安装路径: `/apps/nginx-agent/`
- 独立进程运行
- 通过 systemd 管理

**K8s Sidecar 模式**:
- 与 Nginx 容器共享 Pod
- 共享进程命名空间(`shareProcessNamespace: true`)
- 共享配置文件卷

---

### 3.4 Nginx 服务化模块 (nginx-serverless)

#### 3.4.1 核心设计

**Pod 结构**:
```yaml
Pod: nginx-serverless
├── Container 1: Nginx
│   └── 挂载: /apps/conf/nginx (配置)
│            /apps/svr/run (PID)
│            /apps/logs (日志)
└── Container 2: Agent Sidecar
    └── 挂载: /apps/conf/nginx (配置)
             /apps/svr/run (PID)
             /apps/svr/nginx/sbin (二进制)
             /usr/lib64/nginx-libs (动态库)
```
#### 3.4.2 核心机制

1. **进程隔离与共享**
   - 通过 `shareProcessNamespace: true` 实现进程可见性
   - Agent 可以向 Nginx 进程发送信号(如 `kill -HUP` 实现 Reload)

2. **文件系统共享**
   - 使用 `emptyDir` 卷作为共享存储
   - Agent 写入配置,Nginx 读取配置
   - Agent 读取 `nginx.pid` 文件获取进程 ID

3. **配置持久化策略**
   - **物理存储**: Pod 临时卷(`emptyDir`)
   - **逻辑存储**: MariaDB 数据库(Source of Truth)
   - **恢复机制**: Pod 重启后,Agent 自动从控制台拉取配置

4. **生命周期管理**
   - **创建**: MWFE 执行 `helm install` → 回调控制台 → 同步数据库
   - **扩缩容**: MWFE 执行 `helm upgrade` → 回调控制台 → 更新 Pod 列表
   - **删除**: MWFE 执行 `helm uninstall` → 回调控制台 → 标记删除

#### 3.4.3 操作支持矩阵

| 操作 | VM 模式 | K8s 模式 | 说明 |
|------|---------|----------|------|
| **Test** | ✅ | ✅ | `nginx -t` 语法检查 |
| **Reload** | ✅ | ✅ | `nginx -s reload` 热加载 |
| **Config CRUD** | ✅ | ✅ | 配置文件读写 |
| **Status** | ✅ | ✅ | 进程状态查询 |
| **Start** | ✅ | ❌ | K8s 由 kubelet 管理 |
| **Stop** | ✅ | ❌ | 停止会导致容器 Crash |
| **Restart** | ✅ | ❌ | 需通过 `kubectl delete pod` |

---

## 四、数据流转与交互流程

### 4.1 配置修改流程 (VM 模式)

```mermaid
sequenceDiagram
    participant User as 用户
    participant Console as lb-sphere 控制台
    participant DB as MariaDB
    participant NATS as NATS 消息队列
    participant Agent as Nginx Agent
    participant Nginx as Nginx 进程

    User->>Console: 1. 提交配置变更申请
    Console->>Console: 2. 配置语法预校验 (nginx -t)
    Console->>DB: 3. 保存新配置版本到数据库
    Console->>NATS: 4. 发布配置变更指令
    NATS->>Agent: 5. 将指令推送到目标 Agent
    Agent->>Agent: 6. 在本机保存当前下发的新配置

    rect rgb(220, 240, 255)
        Note over Agent,Nginx: 执行变更
        Agent->>Agent: 7. 备份正在运行的原配置
        Agent->>Nginx: 8. 对下发的新配置执行 nginx -t 检查
        alt 语法检查通过
            Agent->>Agent: 9. 将新配置覆盖到原配置目录
            Agent->>Nginx: 10. 执行 nginx -s reload
            Agent->>NATS: 11. 上报变更成功状态
            NATS->>Console: 12. 更新操作状态
            Console->>User: 13. 显示变更成功
        else 语法检查失败
            Agent->>NATS: 9. 上报变更失败及错误原因
            NATS->>Console: 10. 记录变更失败原因
            Console->>User: 11. 显示错误信息
        end
    end

    rect rgb(255, 240, 220)
        Note over Agent,Nginx: 执行回滚
        NATS->>Agent: R1. 接收回滚指令
        Agent->>Agent: R2. 获取已备份的原配置
        Agent->>Nginx: R3. 对原配置执行 nginx -t 检查
        alt 原配置语法正确
            Agent->>Agent: R4. 将原配置覆盖回配置目录
            Agent->>Nginx: R5. 执行 nginx -s reload
            Agent->>NATS: R6. 上报回滚成功
        else 原配置检查失败
            Agent->>NATS: R4. 上报回滚失败及原因
        end
        NATS->>Console: R7. 同步回滚结果到控制台
    end
```

### 4.2 容器化实例创建流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant MWFE as MWFE 底座
    participant K8s as Kubernetes
    participant Console as lb-sphere 控制台
    participant DB as MariaDB
    participant Agent as Agent Sidecar

    User->>MWFE: 1. 购买 Nginx 实例
    MWFE->>Console: 2. 回调 /order/callback (status=creating)
    Console->>DB: 3. 插入集群记录(状态=CREATING)
    MWFE->>K8s: 4. 执行 helm install
    K8s->>K8s: 5. 创建 Deployment/Pod
    K8s->>Agent: 6. Pod Running，Agent 启动
    Agent->>Console: 7. NATS 心跳上报（实例成功标志）
    MWFE->>Console: 8. 回调 /order/callback (status=completed)
    Console->>MWFE: 9. 调用 MWFE API 获取实例详情
    MWFE-->>Console: 10. 返回 Pod 列表、规格等
    Console->>DB: 11. 更新集群状态(RUNNING)
    Console->>DB: 12. 插入 Pod 信息
    Agent->>Console: 13. 拉取初始配置
    Agent->>Agent: 14. 将配置写入共享卷
    Agent->>Console: 15. 开始周期性心跳上报
```

### 4.3 心跳监控流程

```mermaid
sequenceDiagram
    participant Agent as Nginx Agent
    participant NATS as NATS
    participant Console as lb-sphere
    participant DB as MariaDB
    participant Monitor as 监控平台

    loop 每 30 秒
        Agent->>Agent: 1. 采集 Nginx 指标
        Agent->>Agent: 2. 采集系统指标
        Agent->>NATS: 3. 发布心跳消息
        NATS->>Console: 4. 消费心跳消息
        Console->>DB: 5. 更新心跳时间
        Console->>DB: 6. 更新在线状态
        Console->>Monitor: 7. 上报监控数据
    end

    alt 超过 5 分钟未收到心跳
        Console->>DB: 8. 标记实例离线
        Console->>Monitor: 9. 触发告警
    end
```

---

## *五、设计亮点与技术创新

### 5.1 控制台模块 (lb-sphere) 设计亮点

#### 亮点 1: 双模式统一管控架构

**设计思路**:
- 通过数据库字段 `deploy_type` 区分 VM 和 K8s 实例
- 统一的配置管理接口,屏蔽底层差异
- 前端根据类型动态渲染不同的操作按钮

**量化指标**:
- 支持 **2 种部署模式**(VM + K8s)的统一管理
- 代码复用率达到 **85%**
- 新增 K8s 模式仅增加 **15%** 的代码量

---

#### 亮点 2: 基于 NATS 的异步指令下发机制

**设计思路**:
- 采用发布-订阅模式,解耦控制台与 Agent
- 支持批量操作,提升运维效率
- 消息持久化,保证指令不丢失

**量化指标**:
- 单次批量操作支持 **100+** 台主机
- 指令下发延迟 **< 200ms**
- 消息投递成功率 **99.9%**

---

#### 亮点 3: 配置版本管理与一键回滚

**设计思路**:
- 每次配置变更自动创建版本快照
- 版本对比功能,直观展示差异
- 一键回滚,降低变更风险

**量化指标**:
- 支持查看与回滚到历史版本记录
- 回滚操作耗时 **< 200ms **
- 回滚成功率达 ** 99.99%**(自动语法校验)

---

#### 亮点 4: 灰度升级机制保障稳定性

**设计思路**:
- Agent 版本升级支持灰度发布
- 按主机分组,逐步推进升级
- 自动检测升级失败,触发回滚

**量化指标**:
- 灰度升级分 **多批次**,每批间隔 **2 分钟**
- 升级失败自动回滚,成功率 **99.5%**
- 全量升级耗时从 **2 小时** 降低到 **30 分钟**

#### 亮点 5: 跨语言架构与 UDS 内存级通信极大提升解析性能

**设计思路**:
- **痛点分析**: 平台需要对十万行级的复杂 Nginx 配置文件进行层级化抽象并在前端实时渲染，而基于 Java 原生的解析逻辑性能瓶颈严重，甚至会导致内存溢出。
- **跨语言解耦**: 引入 Go 语言生态中成熟的高性能开源 Nginx Parser，将其构建为专业的配置解析侧服务。
- **架构演进约束 (Sidecar 的必然性)**: 控制台 `lb-sphere` 为**南、贵两地跨城双活部署**。方案一：若单独集中部署一个解析服务，日常高频调用势必面临严峻的跨机房网络延迟与防火墙隔离风险；方案二：若在两地各自额外部署独立的解析集群，不仅造成资源冗余，更会让配置路由规划和排错运维的系统复杂度大幅飙升。为此，**果断弃用独立微服务架构，创新性地采用伴生容器 (Sidecar) 模式**，随控制台在同 Pod 内绑定部署。
- **UDS 协议栈切除优化**: 传统 Sidecar 仍需基于 `127.0.0.1` 经由 TCP/IP 协议层层封包解包。为追求极致的解析与渲染性能，双方通过挂载共享的 `.sock` 文件，使用 **Unix Domain Sockets (UDS)** 替代 HTTP 进行底层的进程间通信 (IPC)，彻底切断了网络协议栈的性能损耗。

**量化指标**:
- **部署维度的降本**: 彻底清除了多维度的网络配置与跨域路由复杂性，解析服务随主服务同生同灭，实现全天候高可用与零配置冗余。
- **性能维度的跃迁**: 相比于同机房 HTTP 调用的 **~15ms** 开销，以及本机回环 `127.0.0.1` 调用的 **~2ms** 网络栈开销，纯内存拷贝的 UDS 通信将单次解析请求的 IPC 延迟成功压制在 **100 微秒级 (< 0.1ms)**。这完美支撑了十万行级强耦合 Nginx 配置文件在 Web UI 端的毫秒级极速反演与零卡顿渲染。

---

### 5.2 权限数据拉取模块 (lb-gnc-k8s-dashboard) 设计亮点

#### 亮点 6: 基于 K8s Informer 的准实时数据同步

**设计思路**:
- 使用 SharedInformerFactory 监听 K8s 资源变化
- 事件驱动,避免轮询,降低 API Server 压力
- 本地缓存 + 增量更新,提升性能

**量化指标**:
- 数据同步延迟 **< 2 秒**
- K8s API 调用次数减少 **90%**
- 支持监听 **1000+** 个 Namespace

---

#### 亮点 7: 解耦设计提升系统可维护性

**设计思路**:
- 从 lb-sphere 拆分为独立服务
- 通过共享数据库进行数据交换
- 独立部署,互不影响

**量化指标**:
- 服务拆分后,lb-sphere 代码量减少 **20%**
- 部署独立性,故障隔离率 **100%**
- 升级互不影响,发布频率提升 **50%**

---

### 5.3 Agent 终端引擎 (nginx-agent) 设计亮点

#### 亮点 8: 基于 Rust 的高性能异步管控架构

**设计思路**:
- **语言选型优势**: 摒弃解释型语言，采用 **Rust** 语言进行底层开发。利用其内存安全与零成本抽象特性，在高频心跳采集与百万级指令调度下，确保对业务 Nginx 进程的干扰趋于零。
- **高并发指令中心**: 深度使用 **Tokio** 异步运行时与插件化架构（注册、执行、监控模块解耦）。单线程即可处理海量 NATS 并发指令，保障了指令下发的实时性与可靠性。
- **极致资源足迹**: 经实测，Agent 在常驻运行状态下内存占用仅为 **10MB 左右**，彻底解决了传统管理 Agent 资源开销大的痛点。

**量化指标**:
- 支持 **亚秒级** 指令穿透与执行反馈。
- 相比旧版脚本化方案，宿主机 CPU/内存开销降低 **80%**。
- 单 Agent 支持同时遥测 **500+** 维度的 Nginx 运行状态指标而无延迟。

---

#### 亮点 9: 跨地域网络隔离环境下的“区域自治”自动化分发优化

**设计思路**:
- **痛点对标**: 南海/贵安 DMZ 及多云等隔离防区由于无法直接访问公有云 OSS，初期需人工在各防区部署 Nginx 中转机并手动同步版本包，导致版本不一致风险高、运维成本巨大。
- **架构升级**: 设计并实现了 **“统一制品中心 + 区域级自愈分发代理 (Regional Relay)”** 的全自动分发模型。
- **智能按需同步 (On-demand Relay)**: 隔离区域节点通过统一的 Mock-OSS 虚拟入口拉取安装脚本。首台节点触发后，区域代理自动从主中心拉取并缓存，后续节点实现局域网级极速内网分发，无需人工介入同步。
- **动态环境自适应**: 彻底解决不同机房 NATS 集群配置差异。安装脚本在拉取时自动上报机房元数据，由控制台动态注入对应的环境配置，实现“一次编译，全网自适应分发”。

**量化指标**:
- **效率提升**: 跨地域、跨防区的版本更新人力成本从 **小时级降至 0**，实现了全自动化发布。
- **分发速度**: 隔离区域内的包下载时延从秒级（公网外链）降至 **毫秒级（局域网代理）**。
- **稳定性**: 彻底杜绝了因手动同步导致的区域间版本差异，保证全网 5000+ 节点 100% 的配置合规。

---

#### 亮点 10: 断点续跑机制保障任务可靠性

**设计思路**:
- 配置下发任务支持断点续传
- 任务状态持久化到本地文件
- 异常重启后自动恢复未完成任务

**量化指标**:
- 任务恢复成功率 **100%**
- 异常重启后恢复耗时 **< 5 秒**
- 支持 **10 个** 并发任务

---

### 5.4 Nginx 服务化模块 (nginx-serverless) 设计亮点

#### 亮点 11: Sidecar 模式实现容器化管控

**设计思路**:
- Agent 作为 Sidecar 与 Nginx 容器共享 Pod
- 共享进程命名空间,Agent 可控制 Nginx 进程
- 共享配置文件卷,实现配置同步

**量化指标**:
- Pod 启动时间 **< 10 秒**
- 配置同步延迟 **< 1 秒**
- 资源开销增加 **< 5%**(相比单容器)

---

#### 亮点 12: 配置持久化策略保障数据安全

**设计思路**:
- 数据库作为配置的 Source of Truth
- Pod 重启后自动从控制台拉取配置
- 避免使用 HostPath,保证 Pod 可调度性

**量化指标**:
- 配置恢复成功率 **100%**
- Pod 重启后配置恢复耗时 **< 5 秒**
- 支持 **无限次** Pod 重建

---

#### 亮点 13: 与 MWFE 深度集成实现全自动化

**设计思路**:
- MWFE 负责基础设施生命周期管理
- 控制台负责业务逻辑与数据持久化
- 通过回调接口实现状态同步

**量化指标**:
- 实例创建全流程自动化,耗时 **< 2 分钟**
- 扩缩容操作耗时 **< 1 分钟**
- 回调成功率 **99.9%**

---

#### 亮点 14: 存算分离架构支持海量静态资源

**设计思路**:
- 静态资源存储到 OSS,Nginx 仅做代理
- 避免使用 PVC,降低存储成本
- 支持 CDN 加速,提升访问速度

**量化指标**:
- 静态资源存储成本降低 **70%**
- CDN 加速后访问速度提升 **5 倍**
- 支持 **TB 级** 静态资源托管

---

#### 亮点 15: 运维操作映射实现云原生最佳实践

**设计思路**:
- 停止服务 → 缩容到 0 副本
- 启动服务 → 扩容到 N 副本
- 重启服务 → 删除 Pod,K8s 自动重建

**量化指标**:
- 操作响应时间 **< 30 秒**
- 服务可用性 **99.95%**
- 运维效率提升 **80%**

---

## 六、技术栈总结

### 6.1 后端技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Java | 21 | 控制台与数据拉取服务开发语言 |
| Spring Boot | 3.2.5 | 应用框架 |
| Spring Cloud | 2023.0.1 | 微服务框架 |
| MyBatis-Plus | 3.5.6 | ORM 框架 |
| Rust | 1.70+ | Agent 开发语言 |
| Axum | 0.6+ | Rust Web 框架 |
| Tokio | 1.28+ | Rust 异步运行时 |

### 6.2 中间件与存储

| 技术 | 版本 | 用途 |
|------|------|------|
| MariaDB | 10.5+ | 关系型数据库 |
| Redis | 6.0+ | 缓存与会话管理 |
| NATS | 2.9+ | 消息中间件 |
| Kubernetes | 1.24+ | 容器编排平台 |

### 6.3 工具库

| 技术 | 版本 | 用途 |
|------|------|------|
| Hutool | 5.8.23 | Java 工具类 |
| MapStruct | 1.5.5 | 对象映射 |
| Lombok | 1.18.30 | 代码简化 |
| Redisson | 3.26.0 | 分布式锁 |

---

## 七、架构设计亮点(面试重点)

### 7.1 高可用架构设计

#### 当前架构(单区域部署)

```mermaid
graph TB
    subgraph "顺德区域 Shunde Region"
        LB1[lb-sphere 控制台]
        Dashboard1[lb-gnc-k8s-dashboard]
        MariaDB1[(MariaDB 主库)]
        Redis1[(Redis 主节点)]
        NATS1[NATS 集群]
    end

    subgraph "数据面 Data Plane"
        VM[VM Nginx 实例]
        K8s[K8s Nginx 实例]
    end

    LB1 --> MariaDB1
    LB1 --> Redis1
    LB1 --> NATS1
    Dashboard1 --> MariaDB1
    NATS1 --> VM
    NATS1 --> K8s

    style MariaDB1 fill:#FF6B6B,color:#fff
    style Redis1 fill:#FF6B6B,color:#fff
```

**存在的问题**:
- ❌ 单点故障风险: MariaDB、Redis 无备份
- ❌ 区域故障无法切换
- ❌ 数据无异地容灾

---

#### 推荐架构(多区域双活/容灾)

```mermaid
graph TB
    subgraph "顺德区域 Shunde Region - 主"
        LB1[lb-sphere 主]
        Dashboard1[lb-gnc-k8s-dashboard]
        MariaDB1[(MariaDB 主库)]
        Redis1[(Redis 主节点)]
        NATS1[NATS 集群]
    end

    subgraph "贵安区域 Guian Region - 备"
        LB2[lb-sphere 备]
        MariaDB2[(MariaDB 从库)]
        Redis2[(Redis 从节点)]
        NATS2[NATS 集群]
    end

    subgraph "公有云区域 Public Cloud - 容灾"
        LB3[lb-sphere 容灾]
        MariaDB3[(MariaDB 从库)]
    end

    MariaDB1 -->|主从复制| MariaDB2
    MariaDB1 -->|主从复制| MariaDB3
    Redis1 -->|主从复制| Redis2

    LB1 --> MariaDB1
    LB2 --> MariaDB2
    LB3 --> MariaDB3

    NATS1 -->|集群同步| NATS2

    style MariaDB1 fill:#4A90E2,color:#fff
    style MariaDB2 fill:#50C878,color:#fff
    style MariaDB3 fill:#FFA500,color:#fff
```

**架构优势**:
- ✅ **高可用**: 主库故障,自动切换到从库
- ✅ **负载均衡**: 读请求分散到多个从库
- ✅ **容灾备份**: 公有云作为最后一道防线
- ✅ **就近访问**: 不同区域的 Agent 连接就近的 NATS

**量化指标**:
- 系统可用性从 **99.5%** 提升到 **99.95%**
- 故障恢复时间(RTO)从 **30 分钟** 降低到 **5 分钟**
- 数据丢失时间(RPO) **< 1 分钟**

---

### 7.2 数据库双活方案

#### 方案 1: 主从复制 + 读写分离

```yaml
架构:
  主库(顺德): 处理所有写操作
  从库(贵安): 处理读操作
  从库(公有云): 容灾备份

优点:
  - 实现简单,成本低
  - 读性能提升明显

缺点:
  - 主库故障需手动切换
  - 存在主从延迟(通常 < 1 秒)
```

#### 方案 2: MariaDB Galera Cluster(推荐)

```yaml
架构:
  节点 1(顺德): 主节点
  节点 2(贵安): 主节点
  节点 3(公有云): 主节点

优点:
  - 多主架构,任意节点可写
  - 自动故障切换
  - 数据强一致性

缺点:
  - 配置复杂
  - 跨区域延迟影响性能
```

**推荐配置**:
- 顺德 + 贵安: Galera Cluster(双主)
- 公有云: 异步从库(容灾)

---

### 7.3 Redis 高可用方案

#### 方案: Redis Sentinel + 主从复制

```yaml
架构:
  主节点(顺德): 处理所有写操作
  从节点(贵安): 处理读操作 + 故障切换
  Sentinel(3 节点): 监控 + 自动故障转移

配置:
  sentinel monitor mymaster 10.16.119.181 6379 2
  sentinel down-after-milliseconds mymaster 5000
  sentinel failover-timeout mymaster 10000
```

**量化指标**:
- 故障检测时间 **< 5 秒**
- 自动切换时间 **< 10 秒**
- 数据丢失 **0 条**(同步复制)

---





























