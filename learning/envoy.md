# Envoy

## 简介

Envoy 是一个由 Lyft 开发并于 2016 年开源的高性能云原生代理，采用 C++ 编写，2017 年加入 CNCF，现为 CNCF 毕业项目。它被设计为现代分布式系统的"流量中枢"，核心目标是让网络对应用程序透明。

核心特性：

- **L3/L4/L7 全栈代理**：支持 TCP/UDP、HTTP/1.1、HTTP/2、gRPC 乃至 Redis、MongoDB 等协议
- **进程外架构**：以独立进程运行，通过 Sidecar 模式与业务应用并行部署
- **动态配置**：通过 xDS API 从控制面实时拉取配置，无需重启即可生效
- **顶级可观测性**：内置丰富的指标、分布式追踪和访问日志
- **高度可扩展**：支持 Lua、WebAssembly（Wasm）以及 C++ 原生过滤器扩展

## 定位

### 云原生网络基础设施

Envoy 的定位不是单纯的"反向代理"，而是**可编程的云原生网络基础设施**。它将网络治理逻辑（负载均衡、熔断、限流、TLS、可观测性等）从业务代码中完全剥离，使开发者专注于业务，运维人员专注于网络管控。

### 三种核心部署模式

| 模式 | 说明 | 典型场景 |
| --- | --- | --- |
| **Edge Proxy** | 部署在集群边界，处理南北向流量 | API 网关、流量入口、TLS 卸载 |
| **Sidecar** | 与每个服务实例并行部署，拦截所有出入流量 | 服务网格（Istio、Consul Connect） |
| **Ingress Gateway** | 作为 Kubernetes Ingress 或 Gateway API 实现 | 多租户网关、跨集群流量管理 |

### 生态中的位置

Envoy 是 **Istio 服务网格的事实标准数据面**。Istio 的控制面（Pilot）通过 xDS协议向 Envoy 下发路由、集群、监听器等配置，Envoy 负责实际执行流量治理策略。在 AIGW 项目中，Envoy 作为运行时容器，通过 Golang Filter 扩展机制加载自定义的业务逻辑（`libgolang.so`），实现 LLM 推理请求的 intelligent routing。

## Golang 扩展机制

### 为什么 Envoy 可以加载 Go 共享库

Envoy 的核心架构是**过滤器链（Filter Chain）**。从监听器（Listener）到网络过滤器（Network Filter）再到 HTTP 过滤器（HTTP Filter），所有功能都以可插拔的 Filter形式存在。Envoy 官方在 contrib 扩展中引入了 `envoy.filters.http.golang`，使得 HTTP Filter 可以用 Go 语言实现。

技术原理：

1. **编译为 C 共享库**：Go 代码通过 `CGO_ENABLED=1 go build -buildmode=c-shared` 编译为 `libgolang.so`。这个 `.so` 文件本质上是一个标准的 ELF 动态链接库，包含 C 兼容的导出符号。
2. **Envoy 动态加载**：Envoy 在启动或配置更新时，通过 `dlopen` 系统调用将 `libgolang.so` 加载到进程地址空间。配置中指定 `library_path` 即可指向该文件。
3. **CGO 桥接**：Go 侧通过 CGO 暴露 C 兼容的接口，Envoy C++ 侧调用这些接口。HTNN（MOSN）框架在此基础上封装了更友好的 Go API，开发者只需实现 `DecodeHeaders`、`DecodeData`、`EncodeHeaders` 等标准方法，无需直接操作 CGO。
4. **完整的 Go 运行时**：与 WebAssembly 扩展不同，Golang Filter 运行在完整的 Go 运行时中，支持 Goroutine、GC、channel 等全部 Go 语言特性。

### 与 Wasm 扩展的对比

| 维度 | Golang Filter | Wasm Filter |
| --- | --- | --- |
| 开发语言 | Go | Rust/C++/Go（TinyGo） |
| 编译产物 | `libgolang.so` | `.wasm` 字节码 |
| 运行时 | 完整 Go 运行时（含 GC、Goroutine） | 沙箱虚拟机（V8/Wasmtime） |
| 性能 | 接近原生（CGO 少量开销） | 有沙箱解释/JIT 开销 |
| 动态加载 | `dlopen` 加载 `.so` | 运行时加载 Wasm 模块 |
| 生态限制 | 可使用全部 Go 标准库和第三方库 | TinyGo 仅支持部分 Go 特性 |

### 对 AIGW 的意义

AIGW 利用这一机制，将复杂的 LLM 推理调度逻辑用 Go 实现并编译为 `libgolang.so`，由 Envoy 在 HTTP 请求处理链中调用。这种方式的优势在于：

- **零侵入**：不需要修改 Envoy 源码，也不需要重新编译 Envoy
- **热更新**：更新业务逻辑只需替换 `.so` 文件并触发配置重载
- **开发效率**：Go 的编译速度和调试体验远优于 C++
- **生态复用**：可以直接使用 Go 生态中的 HTTP 客户端、JSON 解析库、Prometheus SDK 等

## 竞品调研

### 核心对比

| 特性 | Envoy | Nginx | HAProxy | Traefik | Pingora |
| --- | --- | --- | --- | --- | --- |
| 核心优势 | 动态配置、服务网格 | 静态内容、Web 服务器 | 极致性能、低延迟 | 云原生、K8s | Rust、内存安全 |
| 配置方式 | xDS API | 静态配置 | 静态配置 | Providers | Rust API |
| K8s 集成 | Istio/Mesh | Ingress Ctrl | Ingress Ctrl | Gateway API | 有限 |
| 动态发现 | xDS/EDS/CDS | 有限 | 有限 | 自动发现 | 代码自定义 |
| 可观测性 | OTel/Prometheus | 中等 | 中等 | Dashboard | Prometheus/OTel |
| 适用场景 | 微服务、Mesh | 传统 Web | 金融、高并发 | 容器化 | CDN、边缘 |
| 学习曲线 | 中高 | 中等 | 中等 | 低 | 高（Rust） |

### 与 AIGW 的关系

AIGW 选择 Envoy 作为运行时有以下原因：

1. **扩展机制**：Envoy 的 Golang Filter API 允许用 Go 语言编写自定义 HTTP 过滤器，编译为共享库（`.so`）后由 Envoy 动态加载，无需修改 Envoy 源码或重启进程
2. **动态路由**：通过 `cluster_header` 和 `SetUpstreamOverrideHost` 机制，AIGW 可以在运行时动态决定请求的目标后端，绕过 Envoy 的静态集群负载均衡
3. **云原生集成**：作为 Istio 的数据面，AIGW 可以直接部署在 Istio 服务网格中，利用 Istio 的 xDS 配置下发和服务发现能力
4. **可观测性**：Envoy 的 Access Log 和 Dynamic Metadata 机制为 AIGW 提供了全链路的监控和日志注入能力
