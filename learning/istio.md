# Istio 技术解构

> 本文档面向非云原生背景的开发者，从"为什么需要 Istio"开始，逐步深入到"Istio 是怎么工作的"。前半部分侧重概念与使用视角，后半部分进入架构与实现细节。

## 定位：Istio、Envoy 与 AIGW 的关系

Istio 是一套**云原生服务治理系统**，Envoy 是其数据面代理——负责执行流量路由、安全加密、可观测性等全部策略。两者分工明确：istiod（控制面）翻译用户意图为 xDS 配置，Envoy（数据面）执行配置。

AIGW 是基于 Envoy 的 Golang HTTP Filter 扩展，通过 Istio 的 EnvoyFilter 机制注入到 Envoy 过滤器链中，为 Envoy（而非 Istio 控制面）增加了推理感知的智能调度能力。两者在 Envoy 过滤器链中协作——Istio 提供基础设施（服务发现、mTLS、基础路由），AIGW 提供领域智能（KV Cache 感知、推理负载均衡、TTFT 预测）——使整个栈能够治理推理服务流量。

关键区分：**AIGW 扩展的是 Envoy 的数据面能力，不是 Istio 的控制面能力。** 它借 Istio 的壳（EnvoyFilter CRD + xDS 部署模式），在 Envoy 里做推理专用的智能调度。istiod 不理解 AIGW 的配置（模型映射、LB 权重），只是通过 EnvoyFilter CRD 原样传递给 Envoy。

- [一、为什么需要服务网格](#一为什么需要服务网格)
  - [1.1 从单体到微服务的网络挑战](#11-从单体到微服务的网络挑战)
  - [1.2 服务网格是什么](#12-服务网格是什么)
  - [1.3 Istio 的定位](#13-istio-的定位)
- [二、整体架构](#二整体架构)
  - [2.1 控制面：istiod](#21-控制面istiod)
  - [2.2 数据面：Envoy Sidecar](#22-数据面envoy-sidecar)
  - [2.3 Sidecar 注入机制](#23-sidecar-注入机制)
- [三、xDS 动态配置协议](#三xds-动态配置协议)
  - [3.1 什么是 xDS](#31-什么是-xds)
  - [3.2 五个核心 xDS API](#32-五个核心-xds-api)
  - [3.3 ADS：聚合发现服务](#33-ads聚合发现服务)
  - [3.4 Delta xDS：增量推送](#34-delta-xds增量推送)
  - [3.5 xDS 在 Istio 中的工作流程](#35-xds-在-istio-中的工作流程)
- [四、Istio CRD 详解](#四istio-crd-详解)
  - [4.1 Gateway](#41-gateway)
  - [4.2 VirtualService](#42-virtualservice)
  - [4.3 DestinationRule](#43-destinationrule)
  - [4.4 ServiceEntry](#44-serviceentry)
  - [4.5 EnvoyFilter](#45-envoyfilter)
  - [4.6 PeerAuthentication](#46-peerauthentication)
  - [4.7 AuthorizationPolicy](#47-authorizationpolicy)
  - [4.8 WorkloadEntry 与 WorkloadGroup](#48-workloadentry-与-workloadgroup)
- [五、流量管理](#五流量管理)
  - [5.1 请求路由](#51-请求路由)
  - [5.2 故障注入](#52-故障注入)
  - [5.3 流量迁移（金丝雀发布）](#53-流量迁移金丝雀发布)
  - [5.4 流量镜像](#54-流量镜像)
  - [5.5 重试与超时](#55-重试与超时)
  - [5.6 熔断](#56-熔断)
- [六、Sidecar 模式与 Ambient 模式](#六sidecar-模式与-ambient-模式)
  - [6.1 Sidecar 模式](#61-sidecar-模式)
  - [6.2 Ambient 模式](#62-ambient-模式)
  - [6.3 两种模式对比](#63-两种模式对比)
- [七、安全：零信任网络](#七安全零信任网络)
  - [7.1 零信任的三个支柱](#71-零信任的三个支柱)
  - [7.2 自动 mTLS](#72-自动-mtls)
  - [7.3 SPIFFE 身份与证书生命周期](#73-spiffe-身份与证书生命周期)
  - [7.4 授权策略](#74-授权策略)
- [八、Envoy 集成深度解析](#八envoy-集成深度解析)
- [九、服务发现](#九服务发现)
- [十、可观测性](#十可观测性)
- [十一、Istio Gateway 与 Kubernetes Ingress](#十一istio-gateway-与-kubernetes-ingress)
- [十二、部署与安装](#十二部署与安装)
- [十三、排障指南](#十三排障指南)

## 一、为什么需要服务网格

### 1.1 从单体到微服务的网络挑战

在单体应用中，所有模块运行在同一个进程里，模块间调用就是一次函数调用——快速、可靠、无需考虑网络。但当应用拆分为微服务后，模块间调用变成了网络请求，随之而来的是一系列新问题：

- **服务发现**：调用方如何知道目标服务的地址？服务实例随时可能扩缩容、重启、迁移。
- **安全通信**：服务间流量如何加密？如何确认对方身份？
- **流量控制**：如何将 10% 的流量导向新版本做金丝雀测试？某个服务过载时如何熔断？
- **可观测性**：一次请求跨越 5 个服务，其中某个变慢了，怎么定位？
- **弹性**：网络抖动时自动重试、超时控制、限流——每个服务都要实现吗？

传统做法是在每个服务的代码中集成这些能力（SDK 模式）。但这意味着每种语言都要维护一套 SDK，升级时所有服务都要改代码重新部署。服务网格的核心思想是：**把这些能力从应用代码中剥离出来，下沉到基础设施层**。

### 1.2 服务网格是什么

服务网格（Service Mesh）是一个**专门处理服务间通信的基础设施层**。它的核心实现方式是：在每个服务实例旁部署一个代理（Proxy），所有进出服务的网络流量都经过这个代理。代理负责服务发现、负载均衡、安全加密、流量控制、可观测性——应用代码完全不需要知道代理的存在。

```plaintext
传统模式：
  服务 A ──(直接调用)──► 服务 B

服务网格模式：
  服务 A ──► 代理 A ──► 代理 B ──► 服务 B
         │                      │
         └─── 控制面统一管理 ───┘
```

服务网格分为两个平面：

- **数据面**：由所有代理组成，负责实际的流量转发、安全加密、指标采集。代理是"干活的人"。
- **控制面**：负责管理和配置所有代理——下发路由规则、分发证书、收集遥测数据。控制面是"指挥官"。

### 1.3 Istio 的定位

Istio 是目前最广泛使用的开源服务网格实现，CNCF 毕业项目。它的数据面代理使用 Envoy，控制面是一个名为 istiod 的单进程组件。Istio 的核心价值主张是：**以无侵入的方式为微服务提供安全、流量控制和可观测性**——应用代码零修改。

| 维度 | 说明 |
| --- | --- |
| **数据面代理** | Envoy（C++ 实现，CNCF 毕业项目） |
| **控制面** | istiod（Go 实现，1.5 版本起合并了 Pilot/Citadel/Galley） |
| **部署模式** | Sidecar（每 Pod 一个 Envoy）或 Ambient（每节点一个 ztunnel） |
| **协议支持** | HTTP/1.1、HTTP/2、gRPC、TCP、WebSocket |
| **安全模型** | 自动 mTLS + SPIFFE 身份 + L7 授权策略 |
| **最新稳定版** | 1.26.x（截至 2026 年 4 月） |

## 二、整体架构

### 2.1 控制面：istiod

在 Istio 1.5（2020 年）之前，控制面由三个独立进程组成：Pilot（流量管理）、Citadel（证书管理）、Galley（配置校验）。1.5 版本将它们合并为单进程 **istiod**，简化了部署、降低了资源消耗、消除了组件间通信开销。虽然合并为一个二进制，三个逻辑功能仍然独立存在。

#### Pilot（流量管理）

Pilot 是控制面的"编译器"——它将人类可读的高级路由规则翻译成 Envoy 可执行的低级代理配置。具体职责：

- 监听 Kubernetes API Server，获取 Service、Endpoint、Pod 资源
- 读取 Istio CRD（VirtualService、DestinationRule、Gateway 等）
- 将高级规则翻译为 Envoy xDS 配置（Listener、Route、Cluster、Endpoint）
- 通过 xDS 协议（gRPC 流）将配置推送到每个 Envoy 代理

#### Citadel（安全 / 证书管理）

Citadel 是控制面的"CA 机构"——它为网格中的每个工作负载颁发 X.509 证书，用于 mTLS 通信。具体职责：

- 作为集群内证书颁发机构（CA）
- 为每个工作负载签发 X.509 证书，证书中嵌入 SPIFFE 身份标识（`spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>`）
- 管理证书轮换——证书有效期通常为 24 小时，到期前自动续签
- 处理 Envoy 代理发起的 CSR（Certificate Signing Request）：Envoy 生成密钥对和 CSR，发送给 istiod 签名，获得签名后的证书

#### Galley（配置校验）

Galley 是控制面的"门卫"——它在用户提交的 Istio CRD 生效前进行校验。合并进 istiod 后，Galley 的校验功能由 istiod 的 Validating Admission Webhook 承载。当你应用一个 VirtualService 或 DestinationRule 时，istiod 的 webhook 会在配置到达数据面之前拒绝格式错误或语义冲突的配置。

#### 三个功能的协作流程

1. Galley 校验用户提交的 CRD
2. Pilot 读取已校验的 CRD 和 K8s 服务/端点数据，计算期望的代理配置，通过 xDS 推送给所有 Envoy
3. Citadel 为 Envoy 签发证书，使它们能建立 mTLS 连接
4. 三者共享一个进程、共享内存，无组件间网络开销

### 2.2 数据面：Envoy Sidecar

数据面由部署在每个应用 Pod 旁的 **Envoy 代理**组成。每个 Pod 都有自己的 Envoy 实例。

Envoy 在运行时做了什么：

- **拦截所有进出流量**：通过 init 容器（`istio-init`）设置的 iptables 规则，将所有入站/出站流量重定向到 Envoy
- **出站处理**：应用调用 `http://reviews:9080` → iptables 重定向到本地 Envoy（端口 15001）→ Envoy 查找 xDS 路由表 → 转发到正确的上游
- **入站处理**：流量到达 Pod → iptables 重定向到本地 Envoy（端口 15006）→ 应用安全策略 → 转发给应用容器
- **上报遥测**：将指标、访问日志、追踪 span 上报给 istiod 或外部遥测后端

### 2.3 Sidecar 注入机制

Sidecar 注入是将 Envoy 代理添加到 Pod 中的过程。Istio 支持两种方式：

**自动注入（最常用）：** Istio 在 Kubernetes API Server 注册一个 Mutating Admission Webhook。当 Pod 在标记了 `istio-injection=enabled` 的命名空间中创建时，webhook 修改 Pod 规范，注入 `istio-init` init 容器（设置 iptables 规则）和 `istio-proxy` sidecar 容器（运行 Envoy）。

**手动注入：** `istioctl kube-inject -f deployment.yaml | kubectl apply -f -`，在提交给 K8s 之前预处理 YAML。适用于需要精细控制或无法使用 webhook 的场景。

**限制：** `hostNetwork: true` 的 Pod 不会被注入；注入只在 Pod 创建时发生，已有 Pod 需重启才能获得 sidecar。

## 三、xDS 动态配置协议

### 3.1 什么是 xDS

xDS（Extension Discovery Service）是 Envoy 与管理服务器之间的**配置协议**。在 Istio 中，管理服务器就是 istiod。可以把 xDS 理解为控制面和数据面之间的"神经系统"——控制面通过它告诉每个代理该如何行为。

没有 xDS，Envoy 只能使用静态配置文件，每次变更都需要重启。有了 xDS，Envoy 通过长效 gRPC 流**在运行时动态接收配置**——当新服务部署或路由规则变更时，istiod 只需在现有流上推送一个更新，秒级生效。

### 3.2 五个核心 xDS API

每个 xDS API 对应 Envoy 配置的一个特定层次：

| API | 全称 | 发现什么 | 类比 |
| --- | --- | --- | --- |
| **LDS** | Listener Discovery Service | Listener——Envoy 监听的套接字（IP:端口 + 过滤器链） | 大楼的正门 |
| **RDS** | Route Discovery Service | 路由——URL 路径匹配、Header 匹配、流量拆分规则 | 大厅里的楼层索引牌 |
| **CDS** | Cluster Discovery Service | Cluster——上游后端的逻辑分组 | 大楼里的一个部门 |
| **EDS** | Endpoint Discovery Service | Endpoint——Cluster 内的单个 IP:端口实例 | 部门里的一个工位 |
| **ADS** | Aggregated Discovery Service | 以上全部，在**单一 gRPC 流**上保证推送顺序 | 一个接待员统一处理所有信息 |

**LDS（Listener Discovery Service）：** Listener 定义 Envoy 如何接收入站连接——绑定的地址和端口、过滤器链、TLS 上下文。当 istiod 推送 LDS 更新时，它告诉 Envoy："开始监听这个新端口"或"停止监听那个端口"。

**RDS（Route Discovery Service）：** Route 定义 Envoy 如何转发 HTTP 请求——Virtual Host 域名匹配、路由条目（路径/Header 匹配、权重路由）、重定向/重试/超时/故障注入。RDS 被 LDS 引用：一个 HTTP Listener 通过名称引用路由配置，RDS 提供实际的路由内容。

**CDS（Cluster Discovery Service）：** Cluster 定义上游后端分组——发现类型（EDS/静态/DNS）、负载均衡策略、熔断器设置、异常点检测、TLS 设置。VirtualService 中的 `destination.host` 映射到 CDS 中的一个 Cluster。

**EDS（Endpoint Discovery Service）：** Endpoint 定义 Cluster 内的单个后端实例（IP:端口对，即 K8s 中的 Pod IP + 容器端口）。EDS 是更新最频繁的 xDS 资源，因为 Pod 不断创建和销毁。在 Istio 中，EDS 是服务发现的主要机制：Envoy 不通过 DNS，而是直接从 istiod 通过 EDS 获取 Pod IP。

### 3.3 ADS：聚合发现服务

ADS 是**生产环境 Istio 最重要的 xDS 变体**。它将所有资源类型（LDS、RDS、CDS、EDS）复用到**单一 gRPC 双向流**上，解决了**配置顺序一致性**的关键问题。

没有 ADS 时，如果 LDS 和 RDS 在不同的流上，可能出现竞态条件：Envoy 收到了一条路由，引用了一个 CDS 还没提供的 Cluster，导致临时 404 或连接中断。ADS 确保 istiod 按正确顺序推送更新：CDS 在 EDS 之前、LDS 在 RDS 之前等。

**ADS 在 Istio 中的工作流程：**

1. 每个 Envoy 打开一条 gRPC 双向流到 `istiod:15012`
2. Envoy 发送 `DiscoveryRequest`，指定所需的资源类型
3. istiod 在同一条流上推送 `DiscoveryResponse`，按全局一致的顺序
4. 流永久保持打开，istiod 在配置变更时即时推送更新

### 3.4 Delta xDS：增量推送

Delta xDS 是为了减少全量推送的带宽和 CPU 开销而引入的优化，在 Istio 1.22（2024 年）成为默认模式。

**State-of-the-World（SotW）xDS**（原始协议）：每次推送发送该类型的完整资源集合。如果有 1000 个 Cluster 且只变了 1 个，istiod 仍然推送全部 1000 个 Cluster 给每个受影响的代理。

**Delta xDS**：只发送变更的资源。协议新增：请求中的 `subscribe`/`unsubscribe` 字段（Envoy 告诉 istiod 它关心哪些资源，实现懒加载）；响应中的 `removed_resources` 字段（istiod 告诉 Envoy 哪些资源已被删除）；每次推送只包含新增、修改或删除的资源。

四种 xDS 协议变体总结：

| 变体 | 传输 | 范围 | 顺序保证 |
| --- | --- | --- | --- |
| SotW，按类型 | 每种类型独立 gRPC 流 | 每次推送全量 | 无 |
| SotW，ADS | 单一聚合 gRPC 流 | 每次推送全量 | 有 |
| Delta，按类型 | 每种类型独立 gRPC 流 | 仅变更 | 无 |
| Delta，ADS | 单一聚合 gRPC 流 | 仅变更 | 有 |

Istio 默认使用 **Delta ADS**，兼具顺序保证和增量效率。

### 3.5 xDS 在 Istio 中的工作流程

一个请求从发出到到达目标服务的完整路径：

```plaintext
应用 A 调用 reviews:9080
    │
    ▼
iptables 拦截，重定向到本地 Envoy 出站 Listener（端口 15001）
    │
    ▼
Envoy 查找 LDS → 找到端口 9080 对应的 Listener
    │
    ▼
Listener 委托给 HTTP 连接管理器 → 查找 RDS 路由表
    │
    ▼
RDS 根据 VirtualService 规则匹配（host、path、header 等）
    │
    ▼
匹配的路由指向一个 Cluster（来自 CDS）
    │
    ▼
EDS 提供该 Cluster 的健康 Pod IP 列表
    │
    ▼
Envoy 应用 DestinationRule 中的负载均衡策略，选择一个 Pod
    │
    ▼
流量到达目标 Pod → iptables 重定向到入站 Envoy（端口 15006）
    │
    ▼
入站 Envoy 应用安全策略 → 转发给应用容器
```

Envoy 中几乎一切皆可通过 xDS 动态配置：

| 资源 | xDS API | 动态内容 |
| --- | --- | --- |
| Listener | LDS | 服务入口点、端口、过滤器链 |
| Route | RDS | 流量路由规则、金丝雀权重、Header 路由 |
| Cluster | CDS | 后端服务分组、负载均衡策略 |
| Endpoint | EDS | Pod IP、健康状态 |
| Secret | SDS | TLS 证书和私钥，轮换无需重启 |
| Runtime | RTDS | 功能开关、百分比灰度 |

唯一真正静态的配置是引导配置（bootstrap config），它告诉 Envoy 去哪里找 istiod，加载自 `/etc/istio/proxy/envoy-rev0.json`。

## 四、Istio CRD 详解

Istio 通过 Custom Resource Definition（CRD）扩展 Kubernetes，提供声明式的高级 API 来配置网格。istiod 监听这些 CRD 并将它们翻译为 Envoy xDS 配置。

### 4.1 Gateway

**作用：** 定义流量如何进入网格（ingress）或离开网格（egress）。Gateway 配置边缘的负载均衡器——开放哪些端口、支持哪些协议、使用哪个 TLS 证书。

**类比：** 大楼的正门。它不控制楼内发生什么，只控制谁可以进入以及如何进入。

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
meta
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "bookinfo.example.com"
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: bookinfo-cert
    hosts:
    - "bookinfo.example.com"
```

要点：Gateway 只配置 ingress gateway（运行在边缘的特殊 Envoy），不配置 sidecar；它开放端口和协议但不定义路由逻辑（那是 VirtualService 的工作）；`selector` 匹配 ingress gateway Pod 的标签。

### 4.2 VirtualService

**作用：** 定义流量路由规则——到特定 host 的请求应该如何路由、拆分、重定向或重试。这是 Istio 流量管理的核心 CRD。

**类比：** 十字路口的交通指挥，根据目的地、车牌号或车流量将车辆导向不同道路。

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
meta
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        x-user-type:
          exact: premium
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
```

要点：`hosts` 定义此 VirtualService 作用于哪个服务；`match` 支持 URL 路径、Header、查询参数、方法、端口匹配；`route` 指定目标，支持权重拆分；对于外部流量，VirtualService 必须通过 `gateways` 字段绑定到 Gateway。

### 4.3 DestinationRule

**作用：** 定义路由之后的流量策略——负载均衡、连接池设置、熔断、出站连接的 TLS 模式。它还定义 subset（Pod 的命名分组），供 VirtualService 路由引用。

**类比：** 目的地的服务质量规则。交通指挥（VirtualService）决定走哪条路，DestinationRule 决定走哪条车道、限速多少。

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      connectionPool:
        http:
          http1MaxPendingRequests: 200
```

要点：`subsets` 按 Label 分组 Pod，VirtualService 通过名称引用；`trafficPolicy` 可设在顶层或每个 subset 内；熔断通过 `outlierDetection` 和 `connectionPool` 配置。

### 4.4 ServiceEntry

**作用：** 将网格外部服务纳入 Istio 的服务注册表。没有 ServiceEntry，Envoy 无法路由到外部 host。

**类比：** 给外部访客发放访客证，使其能被大楼安保系统识别。

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
meta
  name: external-api
spec:
  hosts:
  - api.external-service.com
  location: MESH_EXTERNAL
  ports:
  - number: 443
    name: https
    protocol: TLS
  resolution: DNS
```

要点：`location: MESH_EXTERNAL` 表示服务在网格外，`MESH_INTERNAL` 用于在网格内但不通过 K8s 发现的服务；`resolution` 可以是 `DNS`、`STATIC`、或 `NONE`；可以像内部服务一样对 ServiceEntry 的 host 应用 VirtualService 和 DestinationRule。

### 4.5 EnvoyFilter

**作用：** 直接修补或扩展 istiod 生成的 Envoy 配置。当 Istio 的高级 API 不够用时，这是"逃生舱"。

**类比：** 对代理进行热接线——直接访问 Istio 通常抽象掉的机器级配置。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
meta
  name: custom-lua-filter
  namespace: istio-system
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_OUTBOUND
      listener:
        filterChain:
          filter:
            name: envoy.filters.http.connection_manager
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.lua
        typed_config:
          '@type': type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
          inline_code: |
            function envoy_on_request(request_handle)
              request_handle:headers():add("x-custom-header", "injected-by-mesh")
            end
  workloadSelector:
    labels:
      app: reviews
```

要点：`applyTo` 指定修补 Envoy 配置的哪个部分（HTTP_FILTER、LISTENER、CLUSTER、ROUTE 等）；`patch.operation` 支持 INSERT_BEFORE/INSERT_AFTER/INSERT_FIRST/MERGE/ADD/REMOVE/REPLACE；`workloadSelector` 限制哪些代理接收此补丁；这是强大但危险的工具，误用可能破坏网格且不保证跨 Istio 版本稳定；AIGW 正是通过 EnvoyFilter CRD 将 `llmproxy` 插件配置注入到指定路由。

### 4.6 PeerAuthentication

**作用：** 定义入站连接的 mTLS 要求。

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
meta
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

**模式：**

| 模式 | 行为 |
| --- | --- |
| `UNSET` | 继承父级（命名空间级或网格级） |
| `DISABLE` | 不使用 mTLS，仅明文 |
| `PERMISSIVE` | 同时接受 mTLS 和明文（迁移期使用） |
| `STRICT` | 要求 mTLS，拒绝明文 |

也可以针对特定工作负载和端口设置。在 Ambient 模式中，PeerAuthentication 由 ztunnel 在 L4 层执行。

### 4.7 AuthorizationPolicy

**作用：** 定义工作负载的访问控制（授权）——哪些主体可以在什么条件下访问哪些服务。这是 Istio 零信任网络在 L7 层的核心。

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
meta
  name: reviews-viewer
  namespace: default
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/productpage"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/api/reviews*"]
    when:
    - key: request.headers[x-token]
      values: ["valid-token"]
```

**动作类型：** ALLOW（仅匹配的请求被允许，默认拒绝）、DENY（匹配的请求被拒绝，优先级高于 ALLOW）、CUSTOM（委托给外部授权服务器如 OPA）、AUDIT（记录匹配的请求日志，1.18+）。

**评估顺序：**

1. 如果任何 DENY 策略匹配 → 拒绝（DENY 优先级最高）
2. 如果任何 CUSTOM 策略匹配 → 外部授权器决定
3. 如果任何 ALLOW 策略匹配 → 允许
4. 如果没有 ALLOW 策略存在 → 允许（无策略 = 开放）
5. 如果至少有一个 ALLOW 策略但没有匹配 → 拒绝（隐式拒绝所有）

### 4.8 WorkloadEntry 与 WorkloadGroup

**WorkloadEntry** 描述一个非 Kubernetes 工作负载（VM、裸金属服务器），使其能参与网格。**WorkloadGroup** 描述一组 WorkloadEntry——类似于 K8s Deployment 但面向 VM。VM 加入网格时，使用 `istioctl workload entry configure` 命令基于 WorkloadGroup 模板生成引导文件，VM 运行独立 Envoy 连接 istiod。

## 五、流量管理

### 5.1 请求路由

请求路由使用 VirtualService 的 `match` 条件，根据 URI 路径/前缀/正则、HTTP 方法、Header、查询参数、来源命名空间/标签、端口等维度将流量导向不同目标。规则从上到下评估，第一个匹配的规则生效。

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
meta
  name: api-routing
spec:
  hosts:
  - api.example.com
  gateways:
  - api-gateway
  http:
  - match:
    - uri:
        prefix: "/v2"
      method:
        exact: POST
    route:
    - destination:
        host: api-server
        subset: v2
  - match:
    - uri:
        prefix: "/v1"
    route:
    - destination:
        host: api-server
        subset: v1
  - route:
    - destination:
        host: api-server
        subset: v1
```

### 5.2 故障注入

故障注入主动引入错误以测试系统弹性。Istio 支持延迟注入（模拟网络延迟，`fixedDelay: 7s`，可设置 `percentage` 控制影响比例）和中止注入（模拟服务故障，`httpStatus: 500`）。可以在同一规则中组合两者。这是混沌工程的核心工具：验证重试逻辑、熔断器和降级机制是否正确工作，而不需要真的破坏生产服务。

### 5.3 流量迁移（金丝雀发布）

流量迁移使用权重路由逐步将流量从旧版本迁移到新版本。只需更新 VirtualService 权重，秒级生效，不需要代码修改或重新部署。

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
meta
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
```

渐进迁移：从 90/10 开始 → 观察指标 → 移到 80/20 → 继续推进直到 0/100。

### 5.4 流量镜像

镜像将实时流量的副本发送到次要目标，不影响主响应。镜像流量是"发后即忘"的——原始调用者不会看到镜像结果。

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
meta
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 100
    mirror:
      host: reviews
      subset: v2
    mirrorPercentage:
      value: 50
```

典型场景：用真实生产流量测试新版本不影响用户；为新的分析管线捕获真实请求载荷；金丝雀发布前的影子测试。

### 5.5 重试与超时

**重试**自动重新尝试失败的请求，无需修改应用代码。通过 VirtualService 的 `retries` 字段配置：`attempts`（最大重试次数，含首次）、`perTryTimeout`（每次尝试超时）、`retryOn`（触发重试的条件，如 `5xx,reset,connect-failure`）。注意：重试在服务中断期间可能放大负载，务必设置 `perTryTimeout` 并与熔断器配合。

**超时**设置请求耗时的上限，通过 VirtualService 的 `timeout` 字段配置（如 `5s`）。代理强制执行超时，无论应用做什么。默认超时是禁用的——你应该始终设置显式超时。

### 5.6 熔断

熔断通过停止向不健康后端发送流量来防止级联故障。在 DestinationRule 中配置，包含两部分：

**连接池**（`connectionPool`）：限制并发连接数和请求数。如 `maxConnections: 100`（最大 TCP 连接）、`http1MaxPendingRequests: 200`（等待连接的最大请求数）、`http2MaxRequests: 200`（最大并发 HTTP/2 请求数）。

**异常点检测**（`outlierDetection`）：基于 5xx 错误率自动驱逐不健康后端。如 `consecutive5xxErrors: 5`（连续 5 次 5xx 后驱逐）、`interval: 30s`（检查间隔）、`baseEjectionTime: 30s`（驱逐持续时间）、`maxEjectionPercent: 50`（最大驱逐比例）、`minHealthPercent: 25`（低于此健康比例时停止驱逐）。

工作原理：Envoy 追踪每个后端的 5xx 错误率 → 连续 N 次 5xx 后驱逐该后端 → 驱逐时间到期后允许回来 → 如果再次失败，驱逐时间翻倍 → 至少保持一定比例的健康后端，避免全面故障。

## 六、Sidecar 模式与 Ambient 模式

### 6.1 Sidecar 模式

Sidecar 模式是 Istio 自诞生以来的架构。每个 Pod 注入一个 Envoy 代理作为 sidecar 容器。特征：每个 Pod 一个 Envoy（与应用容器共存）；iptables 将所有流量重定向到 sidecar；完整的 L4 和 L7 功能；每 Pod 资源开销约 50-100MB 内存 + 约 50m CPU；服务间调用经过两跳代理（源 sidecar + 目标 sidecar）；Pod 生命周期与 sidecar 耦合，升级 Istio 需重启 Pod。

### 6.2 Ambient 模式

Ambient 模式在 Istio 1.18（2023 年）作为 alpha 引入，1.22（2024 年）进入 beta，1.24（2025 年）达到稳定。它移除了 sidecar 要求，将数据面分为两层。

**ztunnel（L4——零信任隧道）：** 每节点 DaemonSet，为该节点上所有 Pod 提供 L4（TCP）网络能力。通过 `istio-cni`（不是 Pod 内的 iptables）重定向流量到节点本地 ztunnel。ztunnel 处理 mTLS（使用 istiod 颁发的工作负载证书）、L4 路由、SPIFFE 身份认证、基础 L4 指标。ztunnel 用 Rust 编写，极轻量（约 10-20MB 内存），**不解析 HTTP 内容**——不能做基于路径的路由、重试或熔断。

**Waypoint Proxy（L7）：** 可选的每命名空间或每服务 Envoy 部署，在需要 L7 功能时提供。ztunnel 在需要 L7 处理时将流量导向 waypoint。Waypoint 提供 HTTP 路由、重试、超时、熔断、L7 授权策略、流量迁移、故障注入、丰富的可观测性。不需要 L7 功能时，流量完全绕过 waypoint。

```plaintext
需要 L7：
  源 Pod ──► ztunnel (L4, mTLS) ──► Waypoint (L7, 路由, 策略) ──► ztunnel (L4, mTLS) ──► 目标 Pod

仅需 L4：
  源 Pod ──► ztunnel (L4, mTLS) ──► ztunnel (L4, mTLS) ──► 目标 Pod
```

### 6.3 两种模式对比

| 维度 | Sidecar 模式 | Ambient 模式 |
| --- | --- | --- |
| 代理位置 | 每 Pod（sidecar 容器） | 每节点（ztunnel）+ 每命名空间（waypoint） |
| L4 功能 | 通过 Envoy sidecar | 通过 ztunnel（节点本地） |
| L7 功能 | 通过 Envoy sidecar | 通过 waypoint proxy（可选） |
| 每 Pod 资源开销 | 约 50-100MB 内存 + 约 50m CPU | L4 零开销；waypoint 共享 |
| 服务间流量跳数 | 2-3 | 1-3 |
| Istio 升级是否需重启 Pod | 是（sidecar 在 Pod 内） | 否（ztunnel 是 DaemonSet，独立升级） |
| 应用耦合度 | sidecar 共享 Pod 生命周期 | 解耦；Pod 不受代理变更影响 |
| 成熟度 | 自 2018 年生产验证 | 1.24（2025 年）稳定，快速成熟中 |
| 功能对等 | 完整功能集 | 1.25-1.26 接近完全对等 |

长期方向：Ambient 模式是未来，但 Sidecar 模式将继续被支持。

## 七、安全：零信任网络

### 7.1 零信任的三个支柱

**零信任**意味着：永不信任，始终验证。在传统网络中，边界内的服务默认被信任。在零信任模型中，每个服务间调用都必须经过认证和授权，无论网络位置。Istio 通过三个机制实现零信任：**身份**（每个工作负载拥有加密身份，SPIFFE ID）、**认证**（所有流量通过 mTLS 加密，通过证书证明身份）、**授权**（细粒度策略控制谁可以访问什么）。

### 7.2 自动 mTLS

Istio 默认启用自动双向 TLS。当两个 sidecar 代理通信时：源 Envoy 发起 TLS 握手 → 双方出示 istiod CA 签发的 X.509 证书 → 各自验证对方证书 → 建立加密隧道。应用代码从不接触证书或 TLS 库。

**PERMISSIVE 模式**（默认）：服务端同时接受 mTLS 和明文，允许渐进迁移。**STRICT 模式**（生产推荐）：服务端拒绝任何明文连接。

Istio 方法的精妙之处在于 mTLS 对应用透明——代码发起普通 HTTP 调用，sidecar 或 ztunnel 自动将其升级为 mTLS。

### 7.3 SPIFFE 身份与证书生命周期

Istio 使用 SPIFFE 标准为工作负载分配身份。SPIFFE ID 是一个 URI，唯一标识一个工作负载：`spiffe://cluster.local/ns/<namespace>/sa/<service-account>`。这个身份嵌入在 X.509 证书的 URI SAN 字段中，当两个工作负载通过 mTLS 通信时，各方都能看到对方的 SPIFFE ID 并据此做出授权决策。

证书由 istiod 自动管理：Envoy 启动时在本地生成密钥对，发送 CSR 给 istiod 签名获得证书；证书有效期通常 24 小时，到期前 50% 时自动续签，零停机；Istio 不支持证书撤销（CRL/OCSP），依赖短期证书代替，被入侵时撤销服务账号。

### 7.4 授权策略

AuthorizationPolicy 实现细粒度、L7 感知的访问控制，工作在三个层级：网格级（`istio-system` 命名空间，无 selector，空 spec = 隐式拒绝所有）、命名空间级（特定命名空间，无 selector）、工作负载级（带 selector）。条件字段包括 `source.principals`、`source.namespaces`、`source.ip`、`operation.methods`、`operation.paths`、`when` 条件等。

## 八、Envoy 集成深度解析

### 8.1 Envoy 过滤器链

Envoy 的请求处理基于三层过滤器链，每层负责不同网络层级的处理：

```plaintext
连接进入
  |
  v
[Listener] ---- Listener Filters (TLS Inspector, Original Src, Proxy Protocol)
  |
  v
[Filter Chain Match] (按目的地址/SNI/协议匹配)
  |
  v
[Network Filters]
  |- envoy.filters.network.tcp_proxy        (L4 TCP 代理)
  |- envoy.filters.network.http_connection_manager (L7 HTTP 管理器)
  |     |
  |     v
  |   [HTTP Filters]                        (HTTP 层过滤器链)
  |     |- envoy.filters.http.health_check   (健康检查)
  |     |- envoy.filters.http.cors          (跨域)
  |     |- envoy.filters.http.fault         (故障注入)
  |     |- envoy.filters.http.ratelimit     (限流)
  |     |- envoy.filters.http.ext_authz     (外部鉴权)
  |     |- envoy.filters.http.golang        (Go 扩展 -- AIGW 使用)
  |     |- envoy.filters.http.router        (路由 -- 必须，终态过滤器)
  |
  v
[Upstream Cluster]
```

各层过滤器的作用：

- **Listener Filters**：在连接建立阶段运行，不处理数据流量本身，主要用于提取连接元数据（如 TLS SNI、Proxy Protocol 头、原始源地址）。典型过滤器包括 `tls_inspector`（检测 ClientHello 是否包含 SNI）、`original_dst`（获取 iptables REDIRECT 的原始目标地址）
- **Network Filters**：处理 L3/L4 层的原始字节流，可分为 ReadFilter、WriteFilter 和 ReadWriteFilter。最重要的 Network Filter 是 `http_connection_manager`（HTTP 连接管理器），它将原始字节流解析为 HTTP 消息并传递给 HTTP Filter 链。另一个常用的是 `tcp_proxy`，直接转发 TCP 流量
- **HTTP Filters**：在 HTTP 连接管理器内部运行，处理解码后的 HTTP 请求和响应。每个 HTTP Filter 可以拦截、修改或终止请求。HTTP Filter 链的最后一个必须是 `router` 过滤器，负责将请求路由到上游集群

### 8.2 xDS 配置与过滤器注入

Istio 控制面（istiod）将用户定义的 VirtualService、DestinationRule、PeerAuthentication 等资源翻译为 Envoy 的 xDS 配置，通过 gRPC 流式推送到每个 Envoy Sidecar。

xDS 协议族：

| xDS 类型 | 全称 | 对应 Envoy 配置 | Istio 资源来源 |
| --- | --- | --- | --- |
| **LDS** | Listener Discovery Service | Listener + Filter Chain | Gateway、Sidecar 自动配置 |
| **RDS** | Route Discovery Service | Route Configuration | VirtualService |
| **CDS** | Cluster Discovery Service | Cluster | DestinationRule、ServiceEntry |
| **EDS** | Endpoint Discovery Service | ClusterLoadAssignment | K8s Endpoints/EndpointSlices |
| **SDS** | Secret Discovery Service | TLS 证书 | istiod 自动签发/轮换 |

xDS 推送模式：

- **State of the World (SotW)**：每次推送完整配置快照，是 Istio 默认且最成熟的方式。Envoy 建立一条到 istiod:15012 的 gRPC 双向流，istiod 在配置变更时推送全量资源
- **Delta xDS (Incremental)**：仅推送变更的部分配置，减少传输量和处理开销。Istio 从 1.12 开始支持 Delta xDS，默认仍使用 SotW，可通过 `PILOT_ENABLE_DELTA_XDS=true` 启用
- **On Demand Discovery**：懒加载模式，只有在 Envoy 实际收到请求时才拉取对应的路由和集群配置，减少初始配置推送量。通过 `PILOT_ENABLE_EDS_DEBOUNCE` 控制

Istio 为每个 Sidecar 生成的典型 HTTP Filter 链（按顺序）：

1. `istio.authn` -- PeerAuthentication 策略执行，mTLS 证书验证
2. `istio.stats` -- Telemetry V2 指标采集（Prometheus 统计）
3. `istio.alloc_connect` -- 连接池分配跟踪
4. `envoy.filters.http.cors` -- 跨域策略（如配置）
5. `envoy.filters.http.fault` -- 故障注入（如 VirtualService 中配置）
6. `envoy.filters.http.ext_authz` -- 外部授权（如 AuthorizationPolicy 配置 CUSTOM action）
7. `envoy.filters.http.ratelimit` -- 全局限流（如配置 RateLimit）
8. 其他自定义过滤器（通过 EnvoyFilter 或 WasmPlugin 注入）
9. `envoy.filters.http.router` -- 终态路由过滤器，必须位于链末

### 8.3 EnvoyFilter CRD

EnvoyFilter 是 Istio 提供的 CRD，允许用户直接修补（patch）istiod 生成的 Envoy 配置，实现自定义过滤器的注入或已有配置的修改。

EnvoyFilter 核心字段：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
meta
  name: my-golang-filter
  namespace: istio-system        # 作用域：istio-system = 全局
spec:
  workloadSelector:              # 可选，限定生效的工作负载
    labels:
      app: my-service
  configPatches:
    - applyTo: HTTP_FILTER       # 修补目标类型
      match:                     # 匹配条件
        context: SIDECAR_INBOUND # GATEWAY / SIDECAR_INBOUND / SIDECAR_OUTBOUND / ANY
        proxy:
          proxyVersion: "^1\\.2.*"  # 可选，匹配代理版本
        listener:
          portNumber: 8080          # 可选，匹配监听端口
      patch:
        operation: INSERT_BEFORE   # 插入操作类型
        value:                     # 要注入的 Envoy 配置片段
          name: envoy.filters.http.golang
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.golang.v3alpha.Config
            library_path: /etc/envoy/libgolang.so
            library_name: llmproxy
```

Patch 操作类型：

| 操作 | 说明 |
| --- | --- |
| **INSERT_BEFORE** | 在匹配到的过滤器之前插入 |
| **INSERT_AFTER** | 在匹配到的过滤器之后插入 |
| **INSERT_FIRST** | 在过滤器链最前面插入 |
| **MERGE** | 合并配置到匹配的过滤器（用于修改已有过滤器的参数） |
| **REMOVE** | 删除匹配的过滤器 |
| **REPLACE** | 替换匹配的过滤器 |

**注意**：使用 INSERT_BEFORE/INSERT_AFTER 等相对操作时，必须设置 `priority` 字段来确保 EnvoyFilter 的应用顺序。未设置 priority 且使用相对操作的 EnvoyFilter 会被 `istioctl analyze` 标记为 `IST0151` 警告。Priority 值越小越先应用，默认为 0。

EnvoyFilter 的作用域层级：

- `namespace: istio-system` + 无 workloadSelector = 全局生效，应用于所有 Envoy 代理
- `namespace: istio-system` + 有 workloadSelector = 应用于匹配的工作负载
- `namespace: <app-namespace>` + 无 workloadSelector = 应用于该命名空间所有工作负载
- `namespace: <app-namespace>` + 有 workloadSelector = 精确匹配特定工作负载

### 8.4 WasmPlugin 扩展

除 EnvoyFilter 外，Istio 还提供 WasmPlugin CRD，通过 WebAssembly 沙箱方式扩展 Envoy。WasmPlugin 的优势是运行时隔离和动态加载，但性能开销高于 Golang Filter（需要 Wasm 虚拟机的解释/JIT 开销）。AIGW 项目选用 Golang Filter 而非 Wasm，是为获得接近原生的性能和完整的 Go 运行时支持。

### 8.5 流量劫持与 Sidecar 配置

Sidecar 注入通过 Mutating Webhook 自动完成：在标记了 `istio-injection=enabled` 的命名空间中创建 Pod 时，webhook 注入 `istio-init`（设置 iptables 规则后退出）和 `istio-proxy`（运行 Envoy）两个容器。也可通过 `istioctl kube-inject` 手动注入。

iptables 规则将 Pod 的所有出入流量透明劫持到 Envoy：出站流量重定向到 `127.0.0.1:15001`（Envoy 出站 Listener），入站流量重定向到 `127.0.0.1:15006`（Envoy 入站 Listener）。Envoy 自身发出的流量通过 `ISTIO_OUTPUT` 链排除，避免回环。默认使用 REDIRECT 模式（修改目标地址），也可配置 TPROXY 模式（保留原始目标地址，需 `CAP_NET_ADMIN` 权限）。

Envoy 在 15001 端口设置通配出站 Listener，通过 `original_dst` 将流量分发到对应服务端口的虚拟出站 Listener；在 15006 端口设置通配入站 Listener，执行 mTLS 认证和授权策略后转发给应用容器。

`Sidecar` CRD 可精细控制代理配置：`egress` 限制出站允许访问的服务（减少 Envoy 配置量），`ingress` 限制入站监听端口，`outboundTrafficPolicy` 设置出站默认策略（`ALLOW_ANY` 放行或 `REGISTRY_ONLY` 仅允许已注册服务）。

## 九、服务发现

### 9.1 两级服务注册

Istio 的服务注册采用两级结构：**服务定义**（Service / ServiceEntry）描述逻辑服务（主机名、端口、协议），**端点发现**（Endpoints / EndpointSlices）提供实际的 Pod IP 列表。

服务定义来源：Kubernetes Service（Istiod 通过 Informer 机制 Watch，自动转换为内部 Service 实例）、ServiceEntry（注册网格外部服务或非 K8s 服务）、WorkloadEntry（描述非 K8s 工作负载如 VM）。

端点发现来源：Kubernetes Endpoints/EndpointSlices（Pod 扩缩容时实时更新）、DNS 解析（对 DNS 类型的 ServiceEntry）、STATIC（直接指定 IP 列表）。

### 9.2 Istiod 到 Envoy 的配置翻译

| Istio 概念 | Envoy xDS 资源 | 说明 |
| --- | --- | --- |
| Service + Port | Listener + Route + Cluster | 每个 Service 端口生成对应的虚拟监听器、路由规则和集群 |
| DestinationRule | Cluster 配置 | 负载均衡策略、连接池、异常检测、subset 映射 |
| Endpoints | ClusterLoadAssignment (EDS) | 每个 Cluster 的端点列表（IP:Port、权重、健康状态） |
| VirtualService | Route Configuration (RDS) | 路由规则、权重分配、重试、超时、故障注入 |

大型集群可通过 `discoverySelectors` 限定 istiod 只关注特定命名空间，减少资源消耗和 xDS 推送量。

## 十、可观测性

Istio 的核心优势之一是**无需修改应用代码即可获得全链路可观测性**——Envoy Sidecar 在代理层完成指标采集、追踪注入和访问日志记录，应用进程完全无感知。

### 10.1 指标

Envoy 内置的 `istio.stats` 过滤器自动采集 Prometheus 指标：`istio_requests_total`（请求计数）、`istio_request_duration_milliseconds`（延迟分布）、`istio_request_bytes` / `istio_response_bytes`（请求/响应体大小）、TCP 连接指标。所有指标携带丰富的维度标签（source_app、destination_app、response_code 等）。通过 `Telemetry` CRD 可自定义指标维度。

### 10.2 分布式追踪

Istio 自动为每个请求生成 Span 并传播追踪上下文，但**需要应用配合转发追踪头**才能形成完整链路。Envoy 在收到请求时检查是否已有追踪头（`x-b3-traceid` 等），没有则自动生成根 Span。支持 Zipkin、Jaeger、OpenTelemetry 等后端，默认采样率 1%。

### 10.3 访问日志

通过 `Telemetry` CRD 配置 Envoy 输出访问日志，支持 `expression` 过滤（如仅记录错误请求）。默认输出到 stdout，也可通过 OpenTelemetry 发送到远端日志服务。

### 10.4 可观测性工具栈

| 工具 | 功能 |
| --- | --- |
| **Kiali** | 服务拓扑图、流量可视化、配置验证 |
| **Prometheus** | 指标采集和存储（生产建议独立部署） |
| **Grafana** | 指标仪表盘，Istio 提供官方 Dashboard 模板 |
| **Jaeger / Zipkin** | 分布式追踪 UI、服务依赖图、延迟分析 |
| **Envoy Admin** (port 15000) | 原始 Envoy 统计、配置 Dump |

## 十一、Istio Gateway 与 Kubernetes Ingress

### 11.1 Kubernetes Ingress 的局限性

Kubernetes Ingress API（`networking.k8s.io/v1`）是集群入口流量的标准抽象，但存在以下局限：

- **功能有限**：仅支持基本的 HTTP Host/Path 路由和 TLS 终止，不支持流量镜像、故障注入、重试策略、请求头操作等高级流量治理
- **协议限制**：主要面向 HTTP/HTTPS，对 gRPC、WebSocket、TCP、UDP 的支持不完善
- **扩展性差**：每个 Ingress 资源只能关联一个 IngressClass（即一个 Ingress Controller），无法实现多团队多网关的隔离管理
- **注解爆炸**：高级功能依赖特定 Ingress Controller 的注解实现（如 NGINX 的 `nginx.ingress.kubernetes.io/rewrite-target`），不同 Controller 的注解语法不兼容，导致供应商锁定
- **缺乏权限模型**：Ingress 没有角色划分，集群运维和应用开发者操作同一资源，无法实现多租户隔离
- **L4 穿透困难**：TCP/UDP 流量需要通过特殊注解配置，无法像 L7 一样声明式管理

### 11.2 Istio Gateway 的优势

Istio Gateway（`networking.istio.io/v1beta1`）将网关配置与路由配置分离，提供更强的表达力和更灵活的控制：

```yaml
# Gateway：描述入口点（端口、协议、TLS 证书）
apiVersion: networking.istio.io/v1beta1
kind: Gateway
meta
  name: my-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: my-tls-secret
      hosts:
        - myapp.example.com

---
# VirtualService：描述路由规则，绑定到 Gateway
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
meta
  name: my-app
spec:
  hosts:
    - myapp.example.com
  gateways:
    - my-gateway
  http:
    - match:
        - uri:
            prefix: /api
      route:
        - destination:
            host: api-service
            port:
              number: 8080
      retry:
        attempts: 3
        perTryTimeout: 2s
    - match:
        - uri:
            prefix: /v2
      route:
        - destination:
            host: api-service
            subset: v2
      mirror:
        host: api-service-v2-canary
      mirrorPercentage:
        value: 10
```

核心对比：

| 维度 | Kubernetes Ingress | Istio Gateway + VirtualService |
| --- | --- | --- |
| 路由能力 | Host + Path 前缀/精确匹配 | Host + Path + Header + Query + Method + 权重路由 |
| TLS 管理 | 引用 K8s Secret | 引用 K8s Secret + 自动签发/轮换 + mTLS |
| 流量治理 | 无（依赖注解） | 重试、超时、故障注入、流量镜像、限流 |
| 协议支持 | HTTP/HTTPS（主要） | HTTP/HTTPS/gRPC/WebSocket/TCP/TLS/UDP |
| 多网关 | 通过 IngressClass（单一） | 通过 selector 绑定不同 Gateway Deployment |
| 角色划分 | 无 | Gateway（运维）+ VirtualService（开发）分离 |
| 服务网格集成 | 无 | VirtualService 可同时绑定 mesh 内 Gateway 和 Sidecar |

### 11.3 为什么选择 Istio Gateway

1. **统一数据面**：Gateway 和 Sidecar 都是 Envoy，共享相同的配置语法和可观测性工具
2. **端到端流量管理**：从入口网关到服务间调用的全链路流量控制，使用同一套 API（VirtualService、DestinationRule）
3. **零信任安全**：Gateway 到 Sidecar 的全链路 mTLS，AuthorizationPolicy 统一管理
4. **多团队协作**：运维管理 Gateway 资源（端口、证书），开发管理 VirtualService 资源（路由规则），互不干扰
5. **可扩展性**：通过 EnvoyFilter 和 WasmPlugin 扩展网关功能，与 AIGW 的 Golang Filter 集成天然兼容

### 11.4 与 Kubernetes Gateway API 的关系

Kubernetes Gateway API（`gateway.networking.k8s.io`）是 Kubernetes 社区推进的新一代入口 API 标准，旨在替代 Ingress。Istio 从 1.18 开始完整支持 Gateway API，提供了三种 GatewayClass 实现：

- `istio`：基于 Istio Gateway 的实现（推荐）
- `istio-waypoint`：Ambient 模式的 Waypoint 代理

Gateway API 引入了三个角色：Cluster Admin（GatewayClass）、Cluster Operator（Gateway）、Application Developer（HTTPRoute），与 Istio Gateway + VirtualService 的设计理念一致。未来趋势是 Istio 的流量管理 API 逐步对齐 Gateway API，但 VirtualService 和 DestinationRule 由于功能更丰富（故障注入、镜像等）短期内不会被取代。

## 十二、部署与安装

安装方式：**istioctl**（官方推荐，内置升级预检和配置验证）和 **Helm**（与 GitOps 兼容，声明式管理）。IstioOperator 已在 1.15 中弃用。推荐使用 Helm 部署资源，istioctl 执行管理任务（`istioctl precheck`、`istioctl proxy-status`、`istioctl analyze`）。

Profile 是预定义的 Helm chart 值集合，控制安装哪些组件：

| Profile | 组件 | 适用场景 |
| --- | --- | --- |
| **default** | istiod + Ingress Gateway | 生产环境基线 |
| **demo** | 全部组件 + 可观测性栈 | 演示体验（临时存储，不可用于生产） |
| **minimal** | 仅 istiod | 纯 Sidecar 场景，入口由外部 LB 处理 |
| **preview** | default + 实验性功能（如 Ambient CNI） | 预览实验性功能 |

可通过 `istioctl profile dump <profile>` 查看详情，通过 `--set` 或 Helm value 文件自定义。

## 十三、排障指南

### 13.1 诊断工具

- **`istioctl proxy-status`**：检查所有代理的 xDS 配置同步状态（SYNCED / STALE / NOT SENT）
- **`istioctl proxy-config`**：查看 Envoy 的 Listener、Route、Cluster、Endpoint 配置（`listener`、`route`、`cluster`、`endpoint` 子命令）
- **`istioctl analyze`**：静态分析网格配置，检测常见问题（如 VirtualService 引用不存在的 subset `IST0001`、AuthorizationPolicy 冲突 `IST0107`、EnvoyFilter 缺少 priority `IST0151`）
- **`istioctl dashboard`**：一键打开 Kiali / Jaeger / Grafana / Envoy Admin 管理界面
- **Envoy Admin 接口**（port 15000）：通过 `kubectl exec` 访问 `config_dump`、`stats`、`connections`、`runtime` 等端点
- **istiod 调试端点**（port 15014）：查看 xDS 配置（`/debug/config_dump`）、推送事件（`/debug/push_status`）、服务注册表（`/debug/registryz`）

### 13.2 常见问题

| 问题 | 症状 | 排查方法 |
| --- | --- | --- |
| **Sidecar 未注入** | Pod 中无 istio-proxy 容器 | 检查命名空间标签或 Pod 注解；检查 Webhook 状态 |
| **配置未同步（STALE）** | `proxy-status` 显示 STALE | 检查 istiod 日志和 xDS 连接 |
| **503 UH** | 无健康上游端点 | `proxy-config endpoint` 检查端点；检查 DestinationRule subset label |
| **mTLS 失败** | 连接被拒绝或重置 | `proxy-config listener` 检查 mTLS 配置；`istioctl analyze` 检查策略冲突 |
| **路由未生效** | VirtualService 规则未按预期执行 | `proxy-config route` 检查路由是否下发；检查 Host 匹配 |
| **Envoy 内存过高** | Sidecar 内存占用异常 | 用 Sidecar CRD 限制 egress 范围；用 discoverySelectors 减少推送量 |
| **AuthorizationPolicy 拒绝** | 请求返回 403 | `proxy-config listener` 检查 RBAC 规则；查看访问日志 |
