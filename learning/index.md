# AIGW 项目拆解文档

> 本文档面向希望了解 AIGW 的同学，从"它能做什么"开始，逐步深入到"它是怎么做的"。前半部分侧重功能、行为与使用视角，后半部分进入架构与代码实现。

- [一、项目概述](#一项目概述)
  - [1.1 项目定位](#11-项目定位)
  - [1.2 解决的核心问题](#12-解决的核心问题)
- [二、核心功能全景](#二核心功能全景)
  - [2.1 功能矩阵](#21-功能矩阵)
  - [2.2 关键设计目标](#22-关键设计目标)
- [三、端到端请求生命周期](#三端到端请求生命周期)
  - [3.1 整体流程概览](#31-整体流程概览)
  - [3.2 模型映射](#32-模型映射)
  - [3.3 元数据中心](#33-元数据中心)
  - [3.4 输入序列长度的生命周期管理](#34-输入序列长度的生命周期管理)
  - [3.5 其他行为设计说明](#35-其他行为设计说明)
- [四、配置体系与部署模式](#四配置体系与部署模式)
  - [4.1 配置层次](#41-配置层次)
  - [4.2 插件配置结构](#42-插件配置结构)
  - [4.3 部署模式](#43-部署模式)
  - [4.4 环境变量](#44-环境变量)
- [五、高可用设计](#五高可用设计)
  - [5.1 级联降级链](#51-级联降级链)
  - [5.2 熔断器](#52-熔断器)
  - [5.3 故障转移](#53-故障转移)
  - [5.4 冷启动保护](#54-冷启动保护)
  - [5.5 异步写与零阻塞主路径](#55-异步写与零阻塞主路径)
  - [5.6 输入序列长度的四重释放保障](#56-输入序列长度的四重释放保障)
  - [5.7 超时控制](#57-超时控制)
  - [5.8 并发安全](#58-并发安全)
  - [5.9 已知局限](#59-已知局限)
- [六、扩展性设计](#六扩展性设计)
  - [6.1 为什么需要扩展性](#61-为什么需要扩展性)
  - [6.2 扩展点一览](#62-扩展点一览)
  - [6.3 扩展机制总结](#63-扩展机制总结)
- [七、架构分层与依赖关系](#七架构分层与依赖关系)
  - [7.1 整体架构图](#71-整体架构图)
  - [7.2 分层职责](#72-分层职责)
  - [7.3 核心依赖关系](#73-核心依赖关系)
- [八、目录结构详解](#八目录结构详解)
- [九、关键模块深度拆解](#九关键模块深度拆解)
  - [9.1 插件系统（plugins/llmproxy）](#91-插件系统pluginsllmproxy)
  - [9.2 协议转换层（transcoder）](#92-协议转换层transcoder)
  - [9.3 负载均衡与调度（aigateway）](#93-负载均衡与调度aigateway)
  - [9.4 元数据中心客户端（metadata_center）](#94-元数据中心客户端metadata_center)
  - [9.5 预测与统计（prediction / metrics_stats）](#95-预测与统计prediction--metrics_stats)
  - [9.6 异步基础设施](#96-异步基础设施)
  - [9.7 熔断器（circuitbreaker）](#97-熔断器circuitbreaker)
- [十、构建与测试](#十构建与测试)
  - [10.1 常用 Makefile 目标](#101-常用-makefile-目标)
  - [10.2 测试框架](#102-测试框架)
- [十一、总结](#十一总结)

## 一、项目概述

**AIGW**（AI Gateway）是一个面向大规模 LLM（大语言模型）推理服务的**智能推理调度网关**。它运行在 [**Envoy**](envoy.md) 代理内部，以扩展模块的形式存在，负责将用户的 AI 请求智能地分发到后端的推理服务器集群。

### 1.1 项目定位

| 维度 | 说明 |
| --- | --- |
| **产品形态** | Envoy 扩展共享库（`libgolang.so`），被 Envoy 动态加载 |
| **技术框架** | HTNN（基于 Envoy Golang Filter 的扩展框架） |
| **核心能力** | 智能路由、负载均衡、过载保护、延迟预测 |
| **协议支持** | OpenAI HTTP / SSE、Triton gRPC |
| **运行时** | Envoy（Standalone 或 [Istio](istio.md) xDS 模式） |
| **开发语言** | Go 1.22 |

### 1.2 解决的核心问题

在大规模 LLM 推理集群中，传统的负载均衡（如轮询、最少连接）无法充分利用推理服务的特性：

- **KV Cache 前缀命中**：相同的输入序列前缀在不同请求间可以复用已计算的 KV Cache，优先路由到已缓存该前缀的节点能显著降低延迟。
- **预填充 / 解码阶段差异**：请求在预填充（Prefill）阶段占用的显存和算力远大于解码（Decode）阶段，负载指标不能简单用"连接数"衡量。
- **LoRA 适配器**：不同请求可能需要不同的 LoRA 权重，路由需要考虑节点上是否已加载目标 LoRA。
- **TTFT / TPOT 预测**：首 token 时间和每 token 时间对用户体验至关重要，需要基于输入序列长度、缓存命中长度等因子进行预测。

AIGW 通过 **多维感知调度算法** 和 **在线自适应预测模型**，解决了上述问题。

## 二、核心功能全景

### 2.1 功能矩阵

| 功能模块 | 核心能力 | 解决什么问题 |
| --- | --- | --- |
| **智能路由** | 模型名映射、Header 灰度、LoRA 感知 | 请求模型名与后端不一致；按 header 灰度/蓝绿 |
| **负载均衡** | `inference_lb` 多因子评分调度 | 传统轮询不考虑缓存命中、队列深度、预填充负载，导致节点负载不均、用户体验差 |
| **过载保护** | 基于实时队列深度的调度避让 | 避免将请求打到已积压大量请求的节点，防止级联过载 |
| **KV Cache 感知** | 输入序列前缀哈希匹配，优先命中缓存节点 | 复用 KV Cache，降低 TTFT（首 token 延迟） |
| **TTFT 预测** | 6 参数多项式 RLS 在线学习 | 根据输入序列长度和缓存命中长度预测首 token 时间，用于超时控制和负载评估 |
| **TPOT 预测** | 分段 RLS（按 batch size 阈值分段） | 预测每输出一个 token 的耗时，辅助调度决策 |
| **协议转换** | OpenAI ↔ Triton gRPC 双向转换 | 前端 OpenAI 协议，后端 Triton gRPC，需要协议适配 |
| **流式处理** | SSE 逐帧转换、Reasoning Content 拆分 | 支持流式输出；`<think>` 标签内容分离 |
| **多租户 QoS** | ConsumerManager 消费组管理（HTNN 框架提供） | 不同租户/消费者的请求隔离与限流 |
| **可观测性** | Prometheus 指标、Envoy Access Log 注入 | 监控 LB 决策、缓存命中率、TTFT、元数据中心延迟 |
| **熔断与容错** | 元数据中心节点级熔断、故障转移 | 元数据中心故障时不影响主请求，自动降级为随机选择 |
| **异步上报** | 负载/缓存索引的异步无阻塞上报 | 避免上报操作阻塞主请求路径 |

### 2.2 关键设计目标

1. **零阻塞主请求路径**：所有上报操作（负载、缓存索引）均为异步；查询操作带严格超时，超时后降级。
2. **在线自适应**：TTFT/TPOT 预测模型不需要离线训练，根据实际请求延迟持续自我修正。
3. **与云原生基础设施无缝融合**：作为 Envoy 扩展运行，可直接部署于 Istio 服务网格中。
4. **高度可扩展**：新增负载均衡算法、协议转换器、集群发现方式均可通过注册机制插件化扩展。

## 三、端到端请求生命周期

本章节从**行为视角**介绍一个请求从进入 AIGW 到返回给用户的完整旅程，不涉及代码实现细节。

### 3.1 整体流程概览

```plaintext
用户请求 (OpenAI ChatCompletion)
    │
    ▼
[1. 请求接收与解析]
    │  ── 等待完整请求体到达
    │  ── 解析 OpenAI JSON，提取模型名、messages、stream 标志
    │
    ▼
[2. 模型映射与路由决策]
    │  ── 将用户请求的模型名映射为内部场景名 + 目标集群 + 后端协议
    │  ── 支持按请求 Header 匹配灰度规则（如不同版本、不同租户走不同后端）
    │  ── 提取 LoRA 适配器 ID（如有）
    │
    ▼
[3. 输入序列哈希计算]
    │  ── 对请求中的输入序列内容分块计算哈希值
    │  ── 用于后续的 KV Cache 前缀匹配
    │
    ▼
[4. 负载均衡选点]
    │  ── 根据集群名和调度算法选择后端推理节点
    │  ── 若开启负载感知：查询各节点的实时队列长度、预填充负载
    │  ── 若开启缓存感知：查询各节点的 KV Cache 前缀命中情况
    │  ── 综合评分排序，从最优候选集中随机选择一个节点
    │  ── 将选中的节点地址告知 Envoy，后续流量直接发往该节点
    │
    ▼
[5. 请求编码与转发]
    │  ── HTTP1 后端（sglang / vLLM / TensorRT）：直接透传原始 OpenAI JSON
    │  ── gRPC 后端（Triton）：将 JSON 转换为 gRPC protobuf 格式
    │  ── 向元数据中心异步上报：该节点增加一个请求、记录输入序列长度
    │
    ▼
后端推理服务器处理请求
    │
    ▼
[6. 响应头处理]
    │  ── 请求成功（HTTP 200）：将本次输入序列的哈希值异步保存到元数据中心
    │  ── 请求失败（>=400）：准备读取错误信息
    │  ── 流式请求：设置 SSE 响应头，准备逐帧推送
    │  ── 非流式请求：等待完整响应体
    │
    ▼
[7. 响应体转换]
    │  流式响应：
    │    ── 逐帧解析 SSE 数据流
    │    ── 首帧到达时：通知元数据中心"预填充已完成"，释放输入序列长度占用
    │    ── 若启用 reasoning 分离：提取 <think> 标签内容到 reasoning_content 字段
    │    ── 若首帧即报错：将 HTTP 状态码改为 400，丢弃后续所有数据帧
    │
    │  非流式响应：
    │    ── 一次性转换完整响应体
    │    ── 替换模型名、分离 reasoning content
    │    ── 若请求超时未结束：通过定时器自动释放输入序列长度占用
    │
    ▼
[8. 请求收尾与日志记录]
    │  ── 向元数据中心异步上报：该节点请求计数减 1
    │  ── 计算 TTFT（发送完成到首包到达的时间）
    │  ── 记录首包时间、末包时间
    │  ── 将路由决策、缓存命中率、延迟等指标写入访问日志
```

> 以上流程中，步骤 2 的映射规则详见 [3.2 模型映射](#32-模型映射)；步骤 5 涉及的元数据中心交互详见 [3.3 元数据中心](#33-元数据中心)；步骤 5 和步骤 7 中输入序列长度的上报和释放详见 [3.4 输入序列长度的生命周期管理](#34-输入序列长度的生命周期管理)。

### 3.2 模型映射

用户请求中携带的 `model` 字段（如 `"qwen3"`）并不直接对应后端推理服务使用的模型名，AIGW 会通过**模型映射规则**（`model_mapping_rule`）将它转换成三个决策结果：

- 内部场景名（`scene_name`）

  用户请求的模型名是"外部名"，AIGW 内部用 `scene_name` 来标识一个路由场景。例如用户请求 `model: "qwen3"`，映射后 `scene_name` 可能是 `"qwen3-prod"`。这个内部名会：

  - 被设置到访问日志的 `target_model_name` 字段，用于可观测性
  - 如果请求带 LoRA，`scene_name` 会被 LoRA ID 替代后传给后端

  `scene_name` 在 Protobuf 配置中定义（`plugins/llmproxy/config/config.proto`），属于 `Rule` 消息的 `scene_name` 字段。同一个 `model` 下的多条 Rule 必须属于同一个 `cluster` 和 `backend`（由 `validateRules()` 校验），但可以有不同的 `scene_name`，用于区分灰度环境（如 canary vs prod）。未指定 `backend` 时默认为 `triton`。

- 目标集群（`cluster`）

  同一个模型可能部署在多个集群（如 `cluster-a`、`cluster-b`），映射规则决定了请求路由到哪个集群。集群名会：

  - 传给负载均衡器，用于查询该集群下的后端节点列表
  - 设置到请求 Header `Cluster-Name` 中
  - 用于查询元数据中心的负载指标和 KV Cache 信息

  集群之间的共享与隔离：

  - **共享**：TTFT/TPOT 预测模型按 `modelName`（用户请求的原始模型名）全局共享，不区分集群；全局 LB 算法和 LB 配置（`lb_mapping_rule`）也是全局的
  - **隔离**：每个集群有独立的后端节点列表（`ClusterInfo.Endpoints`）、独立的负载指标（队列深度、输入序列长度，通过元数据中心按集群名查询）、独立的 KV Cache 索引（按集群名查询和保存）、独立的负载均衡器实例
  - **约束**：同一 `model` 下的所有 Rule 必须指向同一个 `cluster`，即一个用户模型名只能映射到一个集群

- 后端协议（`backend`）

  后端推理引擎有不同的 API 协议，`backend` 字段决定 AIGW 用什么协议与后端通信。目前支持：

  | 协议值 | 推理引擎 | 通信方式 |
  |--------|----------|----------|
  | `triton` | Triton Inference Server | gRPC |
  | `sglang` | SGLang | HTTP（OpenAI 兼容） |
  | `vllm` | vLLM | HTTP（OpenAI 兼容） |
  | `tensorrt-llm` | TensorRT-LLM | HTTP（OpenAI 兼容） |

  HTTP 类后端（sglang/vllm/tensorrt-llm）AIGW 直接透传请求体；Triton 后端则需要将 OpenAI 格式转码为 gRPC 请求。

#### 灰度路由

映射规则支持**基于请求 Header 的灰度匹配**。同一个用户模型名下可以配多条 Rule，按 Header 数量降序排列优先匹配。例如：

```yaml
model: "qwen3"
rules:
  - scene_name: "qwen3-canary"
    cluster: "cluster-b"
    backend: "vllm"
    headers:
      - key: "x-tenant"
        value: "premium"    # 优先匹配
  - scene_name: "qwen3-prod"
    cluster: "cluster-a"
    backend: "sglang"       # 默认兜底（无 headers）
```

这样 `x-tenant: premium` 的请求走 canary 集群，其余走 prod。

### 3.3 元数据中心

元数据中心（Metadata Center）是 AIGW 的**外部依赖服务**（独立项目 `aigw-project/metadata-center`，与 AIGW 主项目分开部署），为 AIGW 提供两个关键能力：

- **实时负载指标**：每个推理节点的当前排队请求数（`QueuedReqNum`）和输入序列长度（`PromptLength`），供 `inference_lb` 评分调度使用。
- **KV Cache 前缀索引**：记录每个节点已缓存的 prompt 哈希值，供 AIGW 查询哪些节点命中了当前请求的 KV Cache 前缀。

没有元数据中心，AIGW 只能退化为随机选择节点——负载感知和缓存感知都依赖它。

#### 读写分离

AIGW 对元数据中心的操作分为两类，采用不同的通信策略：

- **写操作**（异步）：`AddRequest`（请求进入时 +1 / +promptLength，路径 `/v1/load/stats`）、`DeleteRequest`（请求结束时 -1，路径 `/v1/load/stats`）、`DeletePromptLength`（预填充完成时释放输入序列长度，路径 `/v1/load/prompt`）、`SaveKVCache`（请求成功后保存 prompt 哈希，路径 `/v1/cache/save`）。写操作通过异步工作池提交，不阻塞主请求路径；队列满时静默丢弃。
- **读操作**（同步）：`QueryLoad`（查询集群内各节点负载，路径 `/v1/load/stats`）、`QueryKVCache`（查询 prompt 哈希的缓存命中节点，路径 `/v1/cache/query`）。读操作在请求处理的关键路径上，带 100ms 超时；超时或失败时 AIGW 降级为随机选择，不影响请求正常转发。

#### 在请求生命周期中的角色

回顾 3.1 的流程图，元数据中心参与了以下步骤：

1. **步骤 4（负载均衡选点）**：同步查询 `QueryLoad` 和 `QueryKVCache`，获取各节点的队列深度和缓存命中率，用于多因子评分。
2. **步骤 5（请求编码与转发）**：异步上报 `AddRequest`，告知元数据中心"该节点增加一个请求、输入序列长度为 N"。
3. **步骤 6（响应头处理）**：请求成功时异步上报 `SaveKVCache`，将本次 prompt 哈希保存到对应节点。
4. **步骤 7（响应体转换）**：首 token 到达时异步上报 `DeletePromptLength`，释放输入序列长度占用；非流式请求通过 TTFT 预测定时器兜底释放。
5. **步骤 8（请求收尾）**：异步上报 `DeleteRequest`，将该节点请求计数减 1。

### 3.4 输入序列长度的生命周期管理

AIGW 中的 prompt 指的是发送给模型的完整输入序列（所有 messages 序列化后的内容），可能长达数万 token。

在 LLM 推理中，请求到达推理引擎后首先经历 **预填充（Prefill）** 阶段——引擎需要处理完整的输入序列，计算并缓存 KV（Key-Value）张量。这一阶段消耗大量 GPU 算力和显存带宽，且耗时与输入序列长度成正比。预填充完成后才进入 **解码（Decode）** 阶段，此时每步只需处理一个 token，资源消耗远低于预填充。

因此，**一个节点上正在预填充的输入序列总长度**是衡量该节点当前预填充负载的核心指标。AIGW 将其上报到元数据中心后供 `inference_lb` 计算评分时使用——`PrefillLoad = 节点 PromptLength / 集群最大 PromptLength`，预填充负载越高的节点评分越低，越不容易被选中。

输入序列长度的关键特征是**时效性极短**：它只在预填充阶段有意义，一旦预填充完成就应立即释放。如果请求结束后仍占用着输入序列长度，元数据中心会误认为该节点仍然处于高负载状态，导致调度器错误地避开它，造成负载不均。因此 AIGW 必须精确管理输入序列长度的**生命周期**——在正确的时机增加、在正确的时机释放，且保证恰好释放一次。

具体来说，输入序列长度的生命周期如下：

- **增加**：请求转发到后端节点后，AIGW 调用 `AddRequest` 将该节点的请求计数 +1，并将输入序列长度（按 512 字节分块计数）上报到元数据中心。
- **释放**：释放时机取决于请求类型：
  - **流式请求**：首 token 返回时，预填充阶段即结束，此时立即调用 `DeletePromptLength` 释放输入序列长度占用。
  - **非流式请求**：无法精确知道预填充何时结束，因此采用"预测 TTFT × 1.2"作为超时时间，到期自动释放；若请求提前结束，也立即释放。
- **清除**：请求结束时（无论成功或失败），AIGW 在 `OnLog` 阶段调用 `DeleteRequest` 将该节点的请求计数 -1。
- **幂等保护**：无论走哪条释放路径，`DeletePromptLength` 只会执行一次（通过 `isPromptLengthDeleted` 标志位保证），防止重复扣减导致负载指标失真。

### 3.5 其他行为设计说明

#### 流式首帧错误处理

在 SSE 流式场景中，HTTP 200 的响应头先到达，但首帧数据可能就是错误信息（如模型不存在）。AIGW 会在收到首帧后：

- 识别出这是一个错误响应
- 将 HTTP 状态码从 200 修改为 400
- 丢弃后续所有数据帧
- 向用户返回标准 OpenAI 格式的错误 JSON

这样用户无需关心后端是否使用流式协议，始终收到符合预期的错误响应。

#### 负载均衡的"评分 + 随机"策略

`inference_lb` 不是简单地选择评分最高的节点，而是：

1. 对所有候选节点按综合评分降序排序
2. 取前 N%（默认 5%，至少 1 个）作为最优候选集
3. 在候选集中**随机选择**一个节点

这样做的好处是：既保证了大概率选择优质节点，又避免了所有请求都打到同一个"最优"节点导致其迅速过载。

## 四、配置体系与部署模式

本章节从**使用视角**介绍如何配置和部署 AIGW，不涉及代码实现细节。

### 4.1 配置层次

AIGW 的配置分为三个层次：

| 层次 | 作用 | 典型内容 |
| --- | --- | --- |
| **Envoy 引导配置** | Envoy 本身的启动配置 | Listener 地址、Filter 链、路由表、集群定义 |
| **插件配置** | AIGW 核心业务逻辑的配置 | 模型映射规则、负载均衡算法、LB 权重、协议类型 |
| **环境变量** | 运行时行为开关 | 元数据中心地址、功能开关、超时参数 |

### 4.2 插件配置结构

插件配置是 AIGW 最核心的业务配置，定义在 Envoy 的 `typed_per_filter_config` 中：

```yaml
plugins:
  - name: llmproxy
    config:
      protocol: openai                    # 请求协议：目前仅支持 openai
      algorithm: inference_lb             # 负载均衡算法：inference_lb 或 RoundRobin
      model_mapping_rule:                 # 模型映射规则
        qwen3:                            # 用户请求的模型名
          rules:
            - scene_name: qwen3-32b       # 内部场景名（后端实际模型名）
              backend: sglang             # 后端服务类型：sglang / vllm / tensorrt / triton
              cluster: qwen3.service      # 目标集群名
              route_name: qwen3_default   # 路由标识名（用于日志）
              headers:                    # Header 匹配条件（灰度规则）
                - key: x-canary
                  value: "true"
              subset:                     # 子集选择（如 LoRA 标签）
                - key: lora_id
                  value: "lora-001"
        llama3:
          rules:
            - backend: vllm
              cluster: llama3.service
      lb_mapping_rule:                    # 负载均衡算法配置（按模型名覆盖）
        qwen3:
          load_aware_enable: true         # 是否启用负载感知
          cache_aware_enable: true        # 是否启用缓存感知
          candidate_percent: 5            # 候选集比例（%）
          cache_radio_weight: 2           # 缓存命中率权重（字段名为 radio 是 ratio 的历史拼写错误，已固化在 protobuf 中）
          request_load_weight: 1          # 请求队列负载权重
          prefill_load_weight: 3          # 预填充负载权重
      log:                                # 日志配置
        enable: true
        path: /var/log/aigw/llm.log
```

#### 模型映射规则设计

- **同一模型可配置多条规则**：规则按 Header 匹配条件排序，Header 数量多的优先匹配；无 Header 的规则作为兜底。
- **规则一致性校验**：同一模型的所有规则必须指向相同的 `cluster` 和 `backend`，确保后端一致性。
- **LoRA 支持**：规则中的 `subset` 可用于选择已加载特定 LoRA 的节点；`lora_id` 会在转发时替换请求中的模型名。

#### 负载均衡配置设计

- **按模型名覆盖**：全局默认配置可通过 `lb_mapping_rule` 按模型名精细化覆盖。
- **动态权重调整**：`request_load_weight` 会根据集群内队列长度差异动态放大（差异越大，权重越高），避免节点间负载差距拉大。

### 4.3 部署模式

#### 模式一：本地独立模式（Standalone）

适用于本地开发、测试或不使用 Istio 的生产环境。

- **配置方式**：Envoy 使用静态配置 `envoy-local.yaml`，所有 Listener、Route、Cluster 都在配置文件中写死。
- **服务发现**：从本地 `clusters.json` 文件加载后端节点列表。
- **AIGW 配置**：在 Envoy 路由的 `typed_per_filter_config` 中直接嵌入 `llmproxy` 插件配置。
- **暴露端口**：
  - `10000`：AIGW 服务入口（用户请求发往此处）
  - `10001`：Mock 推理后端（本地测试用）
  - `15000`：Envoy Admin 管理接口

```plaintext
用户请求 ──► Envoy Listener:10000 ──► Golang Filter (AIGW) ──► 后端推理服务器
                                          │
                                          └── 从 clusters.json 获取后端列表
```

#### 模式二：Istio xDS 模式

适用于生产环境，利用 Istio 的控制平面实现动态配置下发和服务发现。

- **配置方式**：Envoy 通过 ADS（Aggregate Discovery Service）从 Istio Pilot 动态获取 Listener、Route、Cluster 配置。
- **服务发现**：后端节点通过 Istio `ServiceEntry` 注册，由 Istio 自动发现。
- **AIGW 配置**：通过 Istio `EnvoyFilter` CRD 将 `llmproxy` 插件配置注入到指定路由。
- **关键 CRD**：
  - `Gateway`：定义 AIGW 入口网关，监听 10000 端口
  - `VirtualService`：定义路由规则（如前缀匹配 `/v1/chat/completions`）
  - `ServiceEntry`：定义后端推理服务的静态端点（或自动发现）
  - `EnvoyFilter-golang-httpfilter`：在 Envoy HTTP 过滤器链中插入 Golang Filter，加载 `libgolang.so`
  - `EnvoyFilter-golang-routeconfig`：将 `llmproxy` 的模型映射、LB 配置注入到指定路由
  - `EnvoyFilter-cluster-header`：设置 `cluster_header: "Cluster-Name"`，使路由目标由 AIGW 动态决定

```plaintext
Istio Pilot (控制平面)
    │  通过 xDS 协议推送配置
    ▼
Envoy Sidecar / Gateway ──► Golang Filter (AIGW)
    │                           │
    │                           └── 通过 ServiceEntry 发现后端
    └── 接收动态 LDS/RDS/CDS
```

### 4.4 环境变量

环境变量用于控制运行时行为，无需重新编译：

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `AIGW_META_DATA_CENTER_HOST` | —（空） | 元数据中心主机名；未设置则元数据中心不启动 |
| `AIGW_META_DATA_CENTER_PORT` | `80` | 元数据中心端口 |
| `AIGW_PROMETHEUS_ADDRESS` | `:6061` | Prometheus 指标端点 |
| `AIGW_PPROF_ADDRESS` | — | pprof 调试端点（未设置则不启动） |
| `AIGW_USE_MOVING_AVERAGE` | — | 启用指数移动平均替代 RLS 预测 |
| `AIGW_AI_PROXY_SPLIT_REASONING` | — | 启用 `<think>` 标签内容分离 |
| `AIGW_LLM_LOGSIZE` | `100MB` | 日志轮转大小 |
| `AIGW_LLM_LOGBACKUPS` | `100` | 日志备份数量 |
| `AIGW_METADATA_CENTER_QUEUE_SIZE` | `1000` | 异步上报队列大小 |
| `AIGW_METADATA_CENTER_WORKER_COUNT` | `100` | 异步 worker 数 |
| `AIGW_METADATA_CENTER_MAX_RETRY` | `0` | 异步上报最大重试次数 |
| `AIGW_METADATA_CENTER_UPDATE_STATS_TIMEOUT` | `100ms` | 元数据中心查询超时 |

## 五、高可用设计

AIGW 的核心设计原则是**请求完成优先于最优路由**——每一层故障都降级到更简单但仍能工作的行为，绝不因辅助系统故障而丢弃用户请求。

### 5.1 级联降级链

当元数据中心不可用时，负载均衡的调度策略逐级降级：

```plaintext
缓存感知 + 负载感知（最优）
  → 缓存查询失败 → 仅负载感知（Score = -W2·RequestLoad - W3·PrefillLoad）
    → 负载查询失败 → 随机选择（返回全部 host 列表）
      → 元数据中心完全禁用 → 随机选择
```

每一级都保证请求仍能被转发，只是路由质量下降。代码实现在 `inference_lb.go`：`QueryKVCache` 失败时 `caches` 为 nil，评分公式中 `CacheHitRate=0`（line 394-397）；`QueryLoad` 失败时 `GetCandidateByStats` 直接返回全部 host（line 389-392）。

### 5.2 熔断器

AIGW 为元数据中心的**每个节点**独立维护一个三态熔断器（`pkg/circuitbreaker/`），术语源自电气断路器——closed 表示电路闭合（请求通过），open 表示电路断开（请求被拒绝）：

```plaintext
        失败次数 ≥ MaxFailures（默认 10）
     ┌───────────────────────────────────┐
     │                                   ▼
  closed（闭合/放行）────────────────► open（断开/拒绝）
     ▲                                   │
     │  探测成功次数 ≥ HalfOpenRequests  │ 探测失败
     │                （默认 3）         │
     └── half-open（半开/试探）──────────┘
```

- **closed（闭合）**：正常状态，请求正常通过。失败次数累计达到阈值后转为 open
- **open（断开）**：该节点从候选列表中移除，请求自动路由到其他健康节点。冷却期（默认 5s）过后转为 half-open
- **half-open（半开）**：允许少量探测请求通过；探测成功则恢复为 closed，失败则立即重回 open
- 熔断器由 `sync.Mutex` 保护，保证并发安全

### 5.3 故障转移

元数据中心客户端使用**一致性哈希 + 多候选**策略实现故障转移（`metadata_center.go:357-381`）：

1. 以集群名为 key 做 FNV-1a 哈希，确定主节点索引。
2. 从该索引开始，返回 `failoverRetry + 1` 个候选节点（默认 `AIGW_META_MAX_FAILOVER_RETRY=1`，即 2 个候选）。
3. 主节点请求失败 → 调用 `ReportFailure`（喂给熔断器）→ 试下一个候选。
4. 全部失败才返回错误，触发上层降级。

### 5.4 冷启动保护

元数据中心通过 DNS 轮询发现后端节点（`pkg/metadata_center/servicediscovery/`）。新发现的节点在 `ColdStartDelay`（默认 10 分钟）内标记为 cold，`GetAvailableHosts()` 优先返回 warm 节点，仅当所有 warm 节点熔断时才使用 cold 节点。静态 IP 模式跳过冷启动（假设已就绪）。

DNS 轮询 goroutine 内置 panic recovery（`service_discovery.go:110-116`），防止解析异常导致监控线程崩溃。

### 5.5 异步写与零阻塞主路径

AIGW 对元数据中心的所有写操作（`AddRequest`、`DeleteRequest`、`DeletePromptLength`、`SaveKVCache`）均通过异步工作池提交，**绝不阻塞请求处理**：

- 队列满时静默丢弃（`select/default`，`async_request.go:95-97`），返回 "queue is full" 错误但不影响请求转发。
- 重试默认关闭（`AIGW_METADATA_CENTER_MAX_RETRY=0`），退避间隔为线性 `attempt × 10ms`。

**代价**：元数据中心长时间不可用时，异步上报持续丢失，计数器会漂移。部分可自愈：输入序列长度靠定时器兜底释放（见 5.6），请求计数靠 `OnLog` 的 `DecreaseMetaDataCenter` 兜底。

### 5.6 输入序列长度的四重释放保障

输入序列长度若不释放，元数据中心会误判节点高负载，导致调度器持续避开该节点。AIGW 设计了四条互补的释放路径：

| 释放路径 | 触发条件 | 代码位置 |
|----------|----------|----------|
| 首帧到达 | 流式请求首 token 返回 | `filter.go:398-405` |
| 响应完成 | 非流式请求完整返回 | `filter.go:275` |
| 定时器到期 | 非流式请求在 TTFT×1.2 后自动释放 | `metadata_center.go:57` |
| OnLog 清理 | 请求结束（无论成功/失败） | `filter.go:363-365` |

`DeletePromptLength()` 通过 `isPromptLengthDeleted` 标志位保证恰好执行一次，防止重复扣减导致计数器失真。TTFT 无历史数据时默认 500ms，预测下限 100ms。

### 5.7 超时控制

所有对元数据中心的同步查询均带严格超时，超时即降级：

| 超时参数 | 默认值 | 用途 |
|----------|--------|------|
| `AIGW_METADATA_CENTER_FETCH_METRIC_TIMEOUT` | 100ms | `QueryLoad` 查询超时 |
| `AIGW_META_DATA_CACHE_FETCH_TIMEOUT` | 100ms | `QueryKVCache` 查询超时 |
| `AIGW_METADATA_CENTER_CLIENT_TIMEOUT` | 100ms | HTTP 客户端拨号超时 |
| `AIGW_METADATA_CENTER_UPDATE_STATS_TIMEOUT` | 100ms | 异步任务默认超时 |

### 5.8 并发安全

| 数据结构 | 保护机制 | 用途 |
|----------|----------|------|
| 熔断器状态 | `sync.Mutex` | 保护 failureCount、state 等字段 |
| 服务发现节点表 | `sync.RWMutex` | DNS 更新写锁，查询读锁 |
| 集群管理器 | `sync.Map` + `sync.RWMutex`（双检锁） | 读多写少，创建时防重复 |
| 预测模型表 | `sync.Map` | 模型名→预测器，无锁读写 |
| 访问日志字段 | `sync.Mutex` | 请求级日志字段并发写入 |

集群管理器的双检锁模式：先 `sync.Map.Load` 无锁快速路径，未命中再加锁二次确认，避免重复创建。

### 5.9 已知局限

| 问题 | 说明 |
|------|------|
| 异步任务默认不重试 | `MaxRetries=0`，元数据中心不可用时计数器漂移无修正；`revisePromptLength` 将负值钳位到 0 说明漂移确实会发生 |
| 无健康检查端点 | 外部编排器无法通过 HTTP 探针判断 AIGW 存活 |
| 未知节点零负载偏差 | 元数据中心无记录的节点默认 `TotalReqs=0`，看起来最轻载反而吸引流量（代码注释承认此问题） |
| UpdateCluster 数据竞争 | `cluster/cluster.go` 中 `cl.hosts` 和 `cl.lb` 更新无锁保护，依赖 Envoy 控制面单线程假设 |

## 六、扩展性设计

本章节介绍 AIGW 预留的扩展点及其设计原理，不涉及代码实现细节。

### 6.1 为什么需要扩展性

AIGW 在多个关键位置设计了**注册-发现**式的扩展机制，而非硬编码，使得新增实现无需修改现有代码，只需新增文件并在入口注册。

### 6.2 扩展点一览

#### 扩展点 1：负载均衡算法

AIGW 的负载均衡分为两层：全局层负责按 `cluster_name` 选择集群，集群内层负责按具体算法选择节点。集群内层的算法通过"工厂注册表"管理：每种算法实现 `LoadBalancer` 接口（`types/lb.go:40`），在初始化时通过 `RegisterLbType` 向注册表登记名称和构造函数（`manager/manager.go:27`）。调度时，AIGW 根据配置中的 `algorithm` 字段从注册表中查找对应的算法实例；找不到时回退到 `RoundRobin`（但当前未注册 RoundRobin 的工厂函数，回退会返回 `nil`）。

**当前注册的实现**：

- `inference_lb`：推理专用多因子评分算法（`inferencelb/inference_lb.go:45`）

#### 扩展点 2：协议转换器

协议转换器采用"工厂注册表"模式：每种协议实现 `Transcoder` 接口（`transcoder/transcoder.go:51-66`），在初始化时通过 `RegisterTranscoderFactory` 注册到工厂（`transcoder/transcoder.go:72`）。AIGW 根据配置中的 `protocol` 字段获取对应的转换器实例。

转换器的职责包括：

- 将前端请求解析为内部统一表示（模型名、messages、stream 标志等）
- 将内部表示编码为后端协议格式
- 将后端响应解码并转换为前端协议格式

**当前注册的实现**：

- `openai`：OpenAI Chat Completions API 转换器（`transcoder/openai/openai.go:78`）。该转换器同时处理 HTTP 后端（sglang/vllm/tensorrt-llm）和 gRPC 后端（Triton），根据 `backend` 字段在内部选择编解码路径。Triton 不是一个独立的转换器，而是 `openai` 转换器内部的一个后端协议分支。

#### 扩展点 3：集群信息提供者

AIGW 将"集群信息从何而来"抽象为 `ClusterInfoProvider` 接口（`types/clusterinfoprovider.go:17-20`），通过依赖注入传入 `NewClusterManager`。提供者负责获取集群节点信息并通过 `ClusterInfoNotifier` 回调推送变更。

**当前实现**：

- `StaticClusterProvider`（`discovery/staticdemo/provider.go:74`）：从 `/etc/aigw/static_clusters.json` 加载集群配置，并在 `init()` 中硬编码注入（`clustermanager/init.go:24-29`）。同时内嵌 CDS 服务器，实现 Envoy Delta xDS 协议向 Envoy 推送集群配置。

#### 扩展点 4：元数据中心后端

AIGW 将元数据中心的能力抽象为 `MetadataCenter` 接口（`types/metadatacenter.go:84-87`），组合了 `InferenceLoadStats`（读写负载指标）和 `KVCacheIndexer`（读写缓存索引）两个子接口。通过 `RegisterMetadataCenter` 函数（`init.go:27`）注册全局单例。

**当前实现**：

- HTTP 客户端（`metadata_center.go:167`）：通过 RESTful API 与独立的元数据中心服务通信，包含 DNS 服务发现、节点级熔断和故障转移。

#### 扩展点 5：插件

AIGW 基于 HTNN 框架，HTNN 支持在一个共享库中注册多个插件。每个插件实现 HTNN 的插件生命周期接口，通过 `plugins.RegisterPlugin` 注册（`plugins/llmproxy/config.go:29`），HTNN 的 FilterManager 负责按配置顺序串行执行。

**当前注册的插件**：

- `llmproxy`：唯一的业务插件，处理所有 LLM 请求

### 6.3 扩展机制总结

所有扩展点遵循统一的设计模式：

1. **接口隔离**：将可变部分抽象为小而明确的接口（`LoadBalancer`、`Transcoder`、`ClusterInfoProvider`、`MetadataCenter`）
2. **注册-发现**：通过全局注册表（`RegisterLbType`、`RegisterTranscoderFactory`、`RegisterMetadataCenter`）或依赖注入（`NewClusterManager`）在初始化时登记实现，运行时按名称或配置查找
3. **零侵入**：新增实现无需修改现有代码，只需新增文件并在入口注册

## 七、架构分层与依赖关系

从本章开始进入实现视角，介绍 AIGW 的代码架构。

### 7.1 整体架构图

```plaintext
┌─────────────────────────────────────────────────────────────┐
│                      Envoy / Istio                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           HTNN Filter Manager ("fm")                  │  │
│  │  ┌──────────────────────────────────────────────────┐ │  │
│  │  │            LLMProxy Plugin                       │ │  │
│  │  │ ┌───────────┐ ┌──────────────┐ ┌───────────────┐ │ │  │
│  │  │ │ Transcoder│ │ LoadBalancer │ │ MetadataCenter│ │ │  │
│  │  │ │ (openai)  │ │(inference_lb)│ │   (client)    │ │ │  │
│  │  │ └───────────┘ └──────────────┘ └───────────────┘ │ │  │
│  │  └──────────────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐    ┌─────────────────┐    ┌───────────────┐
│   Prometheus  │    │ Metadata Center │    │ LLM Backends  │
│   (metrics)   │    │ (独立服务:8080) │    │ (sglang/vLLM/ │
│               │    │                 │    │  Triton/...)  │
└───────────────┘    └─────────────────┘    └───────────────┘
```

### 7.2 分层职责

| 层级 | 包路径 | 职责 |
| --- | --- | --- |
| **接入层** | `cmd/libgolang/` | 入口，注册 Filter 与 CM，启动 pprof/Prometheus |
| **插件层** | `plugins/llmproxy/` | Envoy Filter 生命周期（Decode/Encode/OnLog），请求级状态 |
| **协议层** | `plugins/llmproxy/transcoder/` | 请求/响应的协议解析与转换（OpenAI、Triton gRPC） |
| **调度层** | `pkg/aigateway/loadbalancer/` | 全局负载均衡入口、LB 工厂、`inference_lb` 核心算法 |
| **集群层** | `pkg/aigateway/clustermanager/` | 集群生命周期管理、服务发现、Host 列表动态更新 |
| **数据层** | `pkg/metadata_center/` | 元数据中心客户端：负载查询（同步）、请求上报（异步）、KV Cache 索引 |
| **预测层** | `pkg/prediction/` + `pkg/metrics_stats/` | RLS 在线学习，TTFT/TPOT 预测训练 |
| **基础设施** | `pkg/async_log/`、`pkg/async_request/`、`pkg/circuitbreaker/` | 异步日志、队列、熔断器 |
| **工具层** | `pkg/common/`、`pkg/errcode/`、`pkg/request/`、`pkg/trace/` | 环境变量读取、错误码、请求工具、TraceID |

### 7.3 核心依赖关系

```plaintext
plugins/llmproxy/filter.go
    ├── transcoder/openai          # 协议解析与转换
    ├── aigateway/loadbalancer     # 负载均衡选点
    │   └── clustermanager         # 集群管理与服务发现
    ├── metadata_center            # 元数据中心交互
    │   ├── async_request          # 异步上报队列
    │   ├── servicediscovery       # 节点发现与熔断
    │   └── circuitbreaker         # 节点级熔断器
    ├── metrics_stats              # TTFT 预测与统计
    │   └── prediction/rls         # RLS 在线学习
    ├── request                    # Access Log 字段注入
    ├── trace                      # TraceID
    └── errcode + aigateway/utils  # 错误响应构造
```

## 八、目录结构详解

```plaintext
.
├── cmd/libgolang/              # 共享库入口
│   ├── main.go                 # 注册 "fm" (FilterManager) 和 "cm" (ConsumerManager)
│   └── init.go                 # 启动 pprof 和 Prometheus HTTP 服务
│
├── pkg/                        # 核心业务逻辑
│   ├── aigateway/              # 集群管理、服务发现、负载均衡、OpenAI 结构体
│   │   ├── clustermanager/     # 全局集群管理器（懒加载 + 缓存 + 动态更新）
│   │   ├── discovery/          # 服务发现（静态配置 / xDS Delta CDS）
│   │   │   ├── common/         # Envoy Cluster protobuf 生成工具
│   │   │   └── staticdemo/     # 静态集群配置提供者 + 内嵌 CDS gRPC 服务器
│   │   ├── loadbalancer/       # 负载均衡框架
│   │   │   ├── manager/        # LB 算法工厂注册中心
│   │   │   ├── inferencelb/    # 推理专用多因子评分 LB（核心算法）
│   │   │   ├── lboptions/      # LB 选项与 Header 注入
│   │   │   ├── types/          # Host / LoadBalancer 接口定义
│   │   │   └── host/           # Host 接口实现
│   │   ├── openai/             # OpenAI API 兼容结构体
│   │   └── utils.go            # 网关统一错误响应构造
│   │
│   ├── async_log/              # 异步轮转日志（lumberjack + 有界队列）
│   ├── async_request/          # 异步工作池队列（指数退避重试）
│   ├── circuitbreaker/         # 三态计数熔断器（closed/open/half-open）
│   ├── common/                 # 环境变量解析、泛型 Context 安全取值
│   ├── errcode/                # 标准化错误码（400/401/429/503/500）
│   ├── metadata_center/        # 元数据中心客户端
│   │   ├── types/              # EndpointStats / MetadataCenter 接口
│   │   ├── servicediscovery/   # DNS 轮询 + 冷启动保护 + 一致性哈希选主
│   │   ├── metadata_center.go  # 主客户端：异步写 + 同步读
│   │   ├── init.go             # 全局单例注册
│   │   └── utils.go            # 功能开关（环境变量）
│   │
│   ├── metrics_stats/          # TTFT 匹配/记录；EMA 回退
│   │   ├── prediction.go       # 模型级 RLS 预测器缓存
│   │   ├── ttft.go             # 指数移动平均（EMA）实现
│   │   └── stats.go            # 统一对外接口（RLS / EMA 切换）
│   │
│   ├── prediction/             # RLS 预测算法实现
│   │   ├── prediction.go       # TTFT 预测器（6 参数多项式 RLS）
│   │   ├── tpot_prediction.go  # TPOT 预测器（分段 RLS）
│   │   └── rls/                # 底层 3 参数 RLS（循环展开优化）
│   │
│   ├── prom/                   # Prometheus 指标定义（全局变量 + promauto）
│   ├── request/                # Header/Path 修改、线程安全 Access Log 字段
│   ├── simplejson/             # 安全 JSON 编码（防 panic）
│   └── trace/                  # TraceID 从 PluginState 提取
│
├── plugins/                    # Envoy 插件
│   ├── api/v1/                 # 共享 protobuf 定义（HeaderValue）
│   ├── llmproxy/               # 主 LLM 代理插件
│   │   ├── config.go           # 插件工厂与 HTNN 注册
│   │   ├── filter.go           # 核心 Filter 生命周期（Decode/Encode/OnLog）
│   │   ├── cache_hash.go       # Prompt Murmur3 分块哈希
│   │   ├── metadata_center.go  # 请求级元数据中心交互逻辑
│   │   ├── config/             # Protobuf + Go 运行时配置解析
│   │   ├── log/                # LLM 专用日志项构造器
│   │   └── transcoder/         # 协议转换器
│   │       ├── transcoder.go   # Transcoder 接口
│   │       ├── common.go       # Reasoning 分离开关（AIGW_AI_PROXY_SPLIT_REASONING）
│   │       └── openai/         # OpenAI 协议实现（含 gRPC/Triton 编解码）
│   │
│   └── plugins.go              # 插件注册入口（副作用 import）
│
├── etc/                        # 配置文件
│   ├── config_crds/            # Istio CRD（EnvoyFilter / Gateway / ServiceEntry / VirtualService）
│   ├── clusters.json           # 静态集群端点（本地开发）
│   ├── envoy-istio.yaml        # Istio xDS 模式 Envoy 引导配置
│   ├── envoy-local.yaml        # 独立 Envoy 引导配置（静态 LDS/RDS/CDS）
│   └── istio.yaml              # Istio MeshConfig（空占位）
│
├── docs/                       # 开发者文档（en / zh / 架构图）
├── .github/workflows/          # GitHub Actions（test / lint）
├── Dockerfile                  # 多阶段构建（builder → envoyproxy/envoy）
├── Makefile                    # 构建/测试/开发的主入口
├── go.mod / go.sum             # Go 模块定义
└── .golangci.yml               # Lint 配置
```

## 九、关键模块深度拆解

### 9.1 插件系统（plugins/llmproxy）

#### 设计模式

- **工厂模式**：`filterFactory` 为每个请求创建独立的 `filter` 实例，保证请求级状态隔离。
- **注册表模式**：`plugins.RegisterPlugin("llmproxy", ...)` 向 HTNN 框架注册。

#### filter 结构体状态

```go
type filter struct {
    api.PassThroughFilter
    callbacks       api.FilterCallbackHandler   // Envoy Filter 回调
    config          *cfg.LLMProxyConfig         // 插件配置（含模型映射、LB 配置）
    isStream        bool                        // 是否流式请求
    transcoder      transcoder.Transcoder       // 协议转换器
    serverIp        string                      // 选中的后端 IP
    backendProtocol string                      // 后端协议（sglang/vllm/triton/...）
    modelName       string                      // 原始请求模型名
    traceId         string                      // Trace ID
    dropRespData    bool                        // 流式首帧错误时丢弃后续数据
    // ... 时间戳、元数据中心交互状态、prompt hash 等
}
```

### 9.2 协议转换层（transcoder）

#### Transcoder 接口

```go
type Transcoder interface {
    GetRequestData(headers api.RequestHeaderMap, data []byte) (*RequestData, error)
    EncodeRequest(modelName, backendProtocol string, headers api.RequestHeaderMap, buffer api.BufferInstance) (*RequestContext, error)
    DecodeHeaders(headers api.ResponseHeaderMap) error
    GetResponseData(data []byte) ([]byte, error)
    GetLLMLogItems() *log.LLMLogItems
}
```

#### OpenAI 实现的关键特性

| 特性 | 说明 |
| --- | --- |
| **模型映射** | 请求模型名 → 内部 scene_name + cluster + backend，支持多规则按 Header 灰度 |
| **LoRA 透传** | 从规则中提取 `lora_id`，转发时替换请求中的模型名为 `lora_id` |
| **Reasoning 拆分** | 环境变量控制 `<think>` 提取到 reasoning_content，流式维护跨 chunk 状态机 |
| **首帧错误处理** | SSE 首 chunk 即为错误时，改 HTTP status 为 400 并丢弃后续数据 |
| **gRPC 帧处理** | 5-byte frame header（compressed + length）+ protobuf 编解码 |

### 9.3 负载均衡与调度（aigateway）

#### 9.3.1 两层调度架构

```plaintext
GlobalLoadBalancer (clustermanager.Manager)
    │
    ├──► Cluster 1
    │       └── inferenceLoadBalancer.ChooseHost()
    │
    └──► Cluster 2
            └── RoundRobin.ChooseHost()
```

- **第一层（全局）**：`clustermanager.Manager` 实现 `GlobalLoadBalancer`，按 `cluster_name` 路由到对应集群。
- **第二层（集群内）**：每个集群按 `LoadBalancerType` 缓存不同的 LB 实例（如 `inference_lb`、`RoundRobin`）。

#### 9.3.2 inference_lb 核心算法

**选点流程**：

1. **标签过滤**：通过 `selector` 过滤 host（如匹配 `lora_id` 标签）。
2. **负载感知判断**：若未开启 `load_aware_enable`，直接纯随机选择。
3. **获取候选比例**：`candidate_percent`（默认 5%，至少 1 个节点）。
4. **数据查询**（同步）：
   - `QueryLoad(cluster)`：获取各 endpoint 的 `TotalReqs`、`PromptLength`。
   - `QueryKVCache(cluster, promptHash, topK)`：获取前缀缓存匹配结果。
5. **多因子评分**：

   ```plaintext
   Score = W1 * CacheHitRate - W2 * RequestLoad - W3 * PrefillLoad
   ```

   - `CacheHitRate`：来自元数据中心的 prefix cache 匹配长度比例。
   - `RequestLoad`：`(节点队列长度 - 最小队列长度) / 队列长度差值`（归一化）。
   - `PrefillLoad`：`节点 PromptLength / 最大 PromptLength`（归一化）。
   - `W1`（cache_ratio_weight）= 2（可配置）
   - `W2`（request_load_weight）= 1 * ceil(delta/5)（动态调整，队列差异大时权重增加）
   - `W3`（prefill_load_weight）= 3（可配置）
6. **排序选优 + 随机**：按 Score 降序排序，取前 `candNum` 个，最后在其中**随机选择**，平衡最优解与负载分散。

#### 9.3.3 集群管理器设计

- **懒加载 + 双检锁**：`Manager.clusters` 使用 `sync.Map`，先无锁读取，未命中再加锁创建。
- **动态更新**：`WatchCluster()` 监听集群变更，收到推送后全量更新所有 LB 实例的 host 列表。
- **服务发现**：当前为 `staticdemo.StaticClusterProvider`（从 `clusters.json` 加载），可替换为 Istio xDS 实现。

### 9.4 元数据中心客户端（metadata_center）

#### 架构定位

元数据中心是 AIGW 的**外部依赖服务**（独立项目 `aigw-project/metadata-center`），提供近实时负载指标和 KV Cache 索引。

#### 读写分离设计

| 操作 | 方式 | 时延要求 | 失败策略 |
| --- | --- | --- | --- |
| `Add` / `Delete` / `SaveKVCache` | **异步**（`async_request`） | 不阻塞请求 | 队列满丢弃 |
| `QueryLoad` / `QueryKVCache` | **同步** HTTP 调用 | 100ms 超时 | 降级为随机选择 |

#### 服务发现与健康检查

- **DNS 轮询**：周期性 `net.LookupIP` 发现元数据中心后端节点。
- **冷启动保护**：新节点在 `ColdStartDelay`（默认 10 分钟）内标记为 cold，优先使用 warm 节点。
- **节点熔断**：每个节点绑定 `circuitbreaker.CircuitBreaker`，失败次数达阈值后隔离。
- **一致性哈希选主**：`GetHosts(key, num)` 使用 FNV-1a 哈希选择节点，支持故障转移（返回多个候选）。

#### 输入序列长度生命周期管理

```plaintext
DecodeRequest (AddRequest: +promptLength)
    │
    ├──► Stream: 首帧 EncodeData ──► DeletePromptLength()（预填充完成）
    │
    └──► Non-Stream: 启动 timer (TTFT * 1.2)
             │
             ├──► timer 到期 ──► DeletePromptLength()
             └──► OnLog ──► StopTimer + DecreaseMetaDataCenter()
```

### 9.5 预测与统计（prediction / metrics_stats）

#### 9.5.1 TTFT 预测（6 参数多项式 RLS）

- **模型公式**：`y = a·input² + b·cached² + c·input·cached + d·input + e·cached + f`
  - `input`：请求输入序列长度
  - `cached`：cache hit 长度
- **在线训练**：每次请求结束后根据实际 TTFT 调用 `Train()` 更新参数。
- **默认模型**：内置 38 组预置训练数据，新模型首次出现时 `Clone()` 默认模型。

#### 9.5.2 TPOT 预测（分段 RLS）

- **分段策略**：按 `batch_size` 阈值分段（如 `[10, 50]` 分为 3 段）。
- **每段独立 RLS**：`y = a·batchsize + b·totalTokenNum + c`
- **用途**：预测每输出 token 的耗时，辅助调度决策。

#### 9.5.3 EMA 回退

- 环境变量 `AIGW_USE_MOVING_AVERAGE=enable` 时，切换到指数移动平均（`alpha=0.9`）。
- 按输入序列长度每 1KB 分桶，匹配时从最近的非空桶向前查找。

### 9.6 异步基础设施

#### async_request（异步任务队列）

- **工作者线程池**：启动 N 个 worker goroutine（默认 100），从有界 channel 消费任务。
- **指数退避重试**：间隔 `attempt * 10ms`。
- **配置**：队列大小（默认 1000）、worker 数、最大重试次数、超时（默认 100ms）。

#### async_log（异步日志）

- **单例模式**：`sync.Once` 保证每个进程一个实例。
- **有界队列**：队列满时丢弃日志并记录 Envoy 日志，防止阻塞。
- **底层**：`lumberjack.Logger` 实现按大小切割、备份数量限制、压缩、过期删除。

### 9.7 熔断器（circuitbreaker）

**三态状态机**：

```plaintext
          失败次数达阈值
   ┌────────────────────────────┐
   │                            ▼
closed ──► open (拒绝请求，进入冷却期)
   ▲                            │
   │    冷却期过后              │ 探测失败
   └── half-open (允许少量请求) ┘
          探测成功
```

- **线程安全**：`sync.Mutex` 保护所有状态变更。
- **应用位置**：元数据中心节点级熔断，实现故障隔离。

## 十、构建与测试

### 10.1 常用 Makefile 目标

```bash
# 构建共享库（推荐 Docker 构建，匹配 CI 环境）
make build-so

# 本地构建（需要 CGO + Envoy 开发头文件）
make build-so-local

# 单元测试（带 race 检测和覆盖率）
make unit-test

# 代码检查
make lint-go          # golangci-lint
make lint-license     # Apache-2.0 协议头检查
make fix-license      # 自动修复缺失的协议头

# 本地运行
make start-aigw-local   # 静态配置模式
curl 'localhost:10000/v1/chat/completions' \
  -H 'Content-Type: application/json' \
  --data '{"model":"qwen3","messages":[{"role":"user","content":"hello"}]}'
make stop-aigw

# 生成 Protobuf
cd plugins/llmproxy/config && buf generate
```

### 10.2 测试框架

- **单元测试**：Go 标准 `testing` + `stretchr/testify`。
- **Mock**：部分测试使用 `mosn.io/htnn/api/plugins/tests/pkg/envoy` 模拟 Envoy CAPI 上下文。
- **集成测试**：脚手架已搭好（`tests/integration/cmd/libgolang`），但**暂无实际测试代码**。

## 十一、总结

AIGW 是一个**架构清晰、职责分明、高度可扩展**的 Envoy Golang 扩展项目，其核心竞争力体现在：

1. **智能调度算法（inference_lb）**：综合 KV Cache 命中率、队列负载、预填充负载三因子评分排序，实现推理集群的高效利用。
2. **在线预测模型（RLS）**：无需离线训练，根据实际请求延迟在线更新 TTFT/TPOT 模型参数，适应集群负载变化。
3. **读写分离的元数据中心交互**：写操作异步化保证主请求零阻塞，读操作同步化 + 超时降级保证调度决策的实时性。
4. **完整的请求生命周期管理**：从模型映射、负载均衡、协议转换、流式处理到日志记录的端到端闭环。
5. **Envoy / Istio 原生集成**：作为共享库加载，无需额外进程，与现有云原生基础设施无缝融合。

对于希望深入贡献的开发者，建议按以下顺序阅读代码：

1. `plugins/llmproxy/filter.go` → 理解请求生命周期
2. `plugins/llmproxy/transcoder/openai/openai.go` → 理解协议转换与模型映射
3. `pkg/aigateway/loadbalancer/inferencelb/` → 理解核心调度算法
4. `pkg/metadata_center/metadata_center.go` → 理解元数据中心交互设计
5. `pkg/prediction/prediction.go` → 理解 RLS 预测模型
