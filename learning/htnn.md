# HTNN

## 简介

**HTNN**（Hyper Trust-Native Network）是蚂蚁集团开源的云原生跨层网络解决方案，基于 Envoy 和 Istio 构建，支持通过 Go runtime 进行原生扩展。目前开源的部分为其 L7 南北向接入网关，是 MOSN 社区在下一代网关方向上的核心产品。

HTNN 起源于蚂蚁内部的 MOSN 项目，经过几年的发展融合，沉淀出 **MoE（MOSN on Envoy）** 架构。该架构的核心思想是充分发挥 Envoy 的高性能网络底座与云原生生态优势，同时继承 MOSN 在 Golang 研发效能和开发者生态上的积累，实现"高性能 + 高研发效率"的双重目标。

核心特性：

- **Go 原生扩展**：在 Envoy 官方 Golang filter 之上，构造了一套类似 Kong/APISIX 的插件化 filter manager，开发者可用全功能 Go 编写插件并编译为共享库（`.so`）加载到 Envoy 中
- **多种扩展机制并存**：除 Go 外，还支持 Lua、ExtProc、Wasm 四种扩展方式，通过 CRD 灵活配置
- **全栈开源**：从数据面（Envoy + Go shared library）、控制面（Istio + HTNN controller）到产品层（Console、Dashboard），全部开源
- **云原生标准**：原生支持 Kubernetes Gateway API 和 Istio CRD，配置抽象遵循云原生规范
- **多集群管理**：内置多集群管理机制，可统一管理多套 Kubernetes 集群的网关配置
- **低门槛测试框架**：提供 mock 接口支持单元测试，以及集成测试框架支持在本地运行 Envoy 验证插件逻辑

## 定位

### 云原生网关与扩展框架

HTNN 的定位不是"又一个 Envoy 网关"，而是**围绕研发效率设计的云原生网关扩展框架**。在蚂蚁内部，HTNN 已推动落地并沉淀了企业级能力（插件平台、多集群、变更管控/审计、稳定性与可观测性增强），开源后社区可共享这些能力建设。

其独特价值在于：将 Envoy 底层复杂的 C++ 机制、CGO 桥接细节、xDS 配置顺序等问题封装起来，让插件开发者只需关注业务逻辑——就像 AIGW 的 `llmproxy` 插件一样，开发者实现 `DecodeHeaders`、`DecodeData`、`EncodeHeaders` 等标准方法即可，无需感知 Envoy 工作线程与 Go goroutine 的交互细节。

### 与 Envoy、Istio 的关系

| 组件 | 角色 | HTNN 的职责 |
| --- | --- | --- |
| **Envoy** | 数据面代理 | HTNN 将 Go 插件编译为 `.so` 共享库，由 Envoy 的 `envoy.filters.http.golang` 加载执行；同时封装底层 CGO 调用，提供安全的 Go API |
| **Istio** | 控制面 | HTNN 通过 patch 方式嵌入 Istio，增加 HTNN controller 组件，负责调和 HTNN CRD（如 `FilterPolicy`），并将配置翻译为 xDS 推送给 Envoy |
| **HTNN** | 扩展框架 + 产品层 | 提供插件生命周期管理、配置校验、多集群 Console、Dashboard 等上层能力 |

### 与 AIGW 的关系

AIGW 是基于 HTNN 框架开发的**具体业务插件**。在 AIGW 项目中：

- 使用 `mosn.io/htnn/api` 提供的 Go SDK 开发 `llmproxy` 插件
- 通过 `plugins.RegisterPlugin("llmproxy", ...)` 向 HTNN 的 FilterManager 注册
- 利用 HTNN 提供的 `ConsumerManager` 实现多租户消费组的请求隔离与限流
- 编译产物 `libgolang.so` 由 Envoy 加载，HTNN 的 FilterManager 负责按配置顺序串行执行过滤器链

可以如此理解：**HTNN 是插件运行时和框架，AIGW 是运行在该框架上的一个领域插件**。

## 核心架构

HTNN 采用全栈分层架构，各层标准开放，可按需取用：

```plaintext
┌──────────────────────────────────────────┐
│ Console / Dashboard (产品层)             │
│ 域名接入、API 管理、变更管控、多集群管理 │
├──────────────────────────────────────────┤
│ Control Plane (控制面)                   │
│ Istio + HTNN Controller                  │
│ 调和 Gateway API / Istio CRD / HTNN CRD  │
│ 翻译为 xDS 配置，通过 ADS 推送给数据面   │
├──────────────────────────────────────────┤
│ Data Plane (数据面)                      │
│ Envoy + libgolang.so (Go shared library) │
│ HTNN FilterManager → 插件链串行执行      │
└──────────────────────────────────────────┘
```

### 数据面：Envoy + Go Shared Library

HTNN 数据面由 Envoy 和 Go 共享库两部分组成。用户编写的 Go 插件代码（如 AIGW 的 `llmproxy`）通过 `CGO_ENABLED=1 go build -buildmode=c-shared` 编译为 `.so` 文件，Envoy 在启动时通过 `dlopen` 动态加载。

HTNN 在 Envoy 原生 Golang filter 之上增加了以下封装：

- **FilterManager**：管理插件生命周期，按配置顺序调用 `DecodeHeaders` → `DecodeData` → `EncodeHeaders` → `EncodeData` → `OnLog`
- **ConsumerManager**：管理消费组（Consumer），支持按消费者维度限流、鉴权、路由
- **Goroutine 安全包装**：默认将每个 Envoy → Go 的调用包装在 goroutine 中执行，避免阻塞 Envoy 工作线程；同时提供 `NonBlockingPhases` 机制，允许插件声明某些阶段无需 goroutine 包装，减少性能损耗
- **生命周期保护**：通过 `destroyed` 标记位防止请求结束后 Go 代码继续访问已释放的 Envoy C++ 对象

### 控制面：Istio + HTNN Controller

HTNN 没有采用传统的"写入 EnvoyFilter"或"成为 MCP Server"两种方式，而是通过 patch 直接嵌入 Istio 控制面：

- **HTNN CRD 调和**：监听 `FilterPolicy` 等自定义资源，校验配置合法性
- **Native Plugin 框架**：允许以插件方式修改发送给数据面的 xDS 配置
- **Service Registry**：允许对接外部服务发现系统，转换为数据面内的上游集群信息

这种嵌入方式避免了 EnvoyFilter 的状态管理难题，也避免了 MCP Server 需要同步路由的复杂性。

## 核心功能详解

### 插件机制

HTNN 的插件由两类对象组成：`config`（配置管理）和 `filter`（请求级逻辑执行）。开发者通过 Protobuf 描述配置字段，HTNN 在应用 CRD 时通过 webhook 校验用户输入，并将合法配置通过 xDS 下发给 Envoy。

插件注册采用全局注册表模式，AIGW 中的典型注册代码：

```go
func init() {
    plugins.RegisterPlugin("llmproxy", &filterFactory{})
}
```

FilterManager 在请求到达时按配置顺序创建插件实例，形成过滤器链。每个请求拥有独立的 filter 实例，保证请求级状态隔离。

### Consumer 管理

HTNN 提供 Consumer 抽象，用于标识请求的调用方（如租户、应用、用户）。AIGW 利用这一能力实现多租户 QoS：

- 在插件配置中定义 Consumer 的匹配规则（如按 Header、API Key）
- 为不同 Consumer 设置独立的限流阈值、优先级、路由策略
- 在 Access Log 中注入 Consumer 标识，实现按租户维度的可观测性

### 多版本兼容

由于 Envoy Golang filter API 仍在演进，几乎每个版本都会引入 breaking change。HTNN 提供了一套数据面 API 版本选择机制，通过 Go build tag 编译适配不同 Envoy 版本的共享库：

| Envoy 版本 | build tag |
| --- | --- |
| dev | `envoydev` |
| 1.32 | 最新版本，无需 tag |
| 1.31 | `envoy1.31` |
| 1.29 | `envoy1.29` |

当在旧版本 Envoy 上调用新版本接口时，HTNN 的兼容层会输出错误日志并返回空值，避免进程崩溃。

### 测试框架

HTNN 提供了一整套降低插件开发门槛的测试能力：

- **单元测试**：`mosn.io/htnn/api/plugins/tests/pkg/envoy` package 为所有 Envoy API 提供 mock 接口，支持 `go test -race` 提前发现 data race
- **集成测试**：提供运行所需的 Envoy 二进制和配置脚手架，开发者无需手动部署完整 HTNN 环境即可验证插件逻辑
- **覆盖率**：针对 Go 编译为 `.so` 时无法自动 flush 覆盖率数据的问题，HTNN 在集成测试框架中加入了测试结束时手动 flush 的能力

## 发展历程

HTNN 并非从零开始，而是 MOSN 社区多年技术演进的自然结果：

### 2017-2018：MOSN 诞生

蚂蚁集团为解决微服务架构中多语言 SDK 升级困难、服务治理能力弱等痛点，启动 Service Mesh 技术调研。2018 年 MOSN 诞生并开始在小规模环境落地，同年完成 618 大促核心链路覆盖。

### 2019-2020：MOSN 成熟与开源

MOSN 在 2019 年双 11 完成核心支付链路 100% 覆盖，峰值 QPS 达千万级。2020 年 7 月获得 Istio 官方推荐，成为 Istio 数据面的可选方案之一。期间 MOSN 开始探索商业化落地，并在年底完成首个外部案例。

### 2021：MoE 架构提出

MOSN 社区提出 **MoE（MOSN on Envoy）** 架构，将 MOSN 的服务治理能力通过 Golang filter 扩展机制下沉到 Envoy 中运行。蚂蚁网络基础设施团队向 Envoy 社区贡献了 Golang filter 提案，目标是让 Envoy 和 MOSN 优势互补——Envoy 提供高性能网络底座，MOSN/Go 提供高研发效率的上层扩展。同年，MOSN 完成了与 Envoy、Dapr、WASM 生态的对接。

### 2022：MOSN 1.0 与南北向网关规划

MOSN 1.0 正式发布，标志着项目进入成熟期。社区在 Roadmap 中明确提出南北向网关演进方向：数据面统一基于 MoE 架构，控制面走标准 xDS 协议与 Istio 对接。这一规划为 HTNN 的诞生奠定了架构基础。

### 2023-2024：HTNN 开源

HTNN 作为 MOSN 社区孵化的下一代网关产品正式开源，定位不再是单纯的数据面代理，而是基于 Istio + Envoy 的完整网关解决方案。开源内容包含数据面插件框架、控制面 controller、官方插件集以及 Helm 安装支持。AIGW 项目正是在这一时期基于 HTNN 数据面框架启动开发。

### 至今：持续迭代

HTNN 持续完善企业级能力，包括 Gateway API 支持、多集群管理、4/7 层联动防攻击（XDP）、官方插件 Hub 建设等。其数据面 API 跟随 Envoy 版本持续演进，保持对最新 Envoy 版本的兼容性。

## 竞品对比

| 维度 | HTNN | APISIX | Kong | Envoy Gateway |
| --- | --- | --- | --- | --- |
| 核心优势 | 研发效率、Go 原生扩展、Istio 集成 | 高性能、动态路由 | 生态丰富、插件市场 | 云原生标准、xDS 原生 |
| 扩展语言 | Go（完整运行时）、Lua、Wasm、ExtProc | Lua、Wasm、Go（外部） | Lua、Go（外部） | Wasm、EnvoyFilter（C++） |
| 控制面 | Istio + HTNN controller | 自研控制面 | 自研控制面 | Envoy Gateway CRD |
| 插件开发门槛 | 低（自带 mock + 集成测试） | 中 | 中 | 高（需理解 Envoy C++） |
| 服务网格集成 | 原生（与 Istio 一体） | 需额外适配 | 需额外适配 | 原生（同属 Envoy 生态） |
| 多集群管理 | 内置 | 支持 | 企业版支持 | 有限 |
| 适用场景 | 需要频繁定制插件的网关/Mesh | API 网关、流量入口 | API 网关、微服务管理 | 云原生入口、标准网关 |

### 与 AIGW 的关系（技术视角）

AIGW 选择 HTNN 而非直接使用 Envoy Golang filter 或 Wasm，原因如下：

1. **开发效率**：HTNN 封装了 CGO 线程安全、请求生命周期管理、配置解析等底层细节，AIGW 开发者只需专注 LLM 调度逻辑
2. **测试便利**：利用 HTNN 提供的 mock 接口，AIGW 可以在不启动 Envoy 的情况下完成大量单元测试
3. **多租户能力**：HTNN 的 ConsumerManager 为 AIGW 提供了现成的多租户 QoS 框架
4. **生态复用**：AIGW 可以直接使用 HTNN 数据面的服务发现、集群管理、指标采集等通用能力
5. **未来演进**：随着 HTNN 支持 Gateway API、多集群管理等能力，AIGW 可以平滑继承这些特性
