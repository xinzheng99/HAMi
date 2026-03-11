# HAMi 项目代码深度分析报告

**报告生成日期**: 2026-03-09
**分析版本**: master 分支 (最新)
**分析范围**: 完整源代码树
**更新日期**: 2026-03-09

---

## 目录

1. [项目概览](#1-项目概览)
2. [目录结构详解](#2-目录结构详解)
3. [核心组件分析](#3-核心组件分析)
4. [设备支持模块](#4-设备支持模块)
5. [调度器实现](#5-调度器实现)
6. [Device Plugin 实现](#6-device-plugin 实现)
7. [Webhook 机制](#7-webhook 机制)
8. [配置系统](#8-配置系统)
9. [监控与指标](#9-监控与指标)
10. [测试与质量](#10-测试与质量)
11. [安全分析](#11-安全分析)
12. [性能优化建议](#12-性能优化建议)

---

## 1. 项目概览

### 1.1 基本信息

| 项目 | 详情 |
|------|------|
| **项目名称** | HAMi (Heterogeneous AI Computing Virtualization Middleware) |
| **曾用名** | k8s-vGPU-scheduler |
| **许可证** | Apache License 2.0 |
| **CNCF 状态** | Sandbox 项目 & CNAI Landscape 项目 |
| **Go 版本** | go 1.25.5 |
| **Kubernetes 版本** | >= 1.18 |

### 1.2 代码统计

| 指标 | 数值 |
|------|------|
| **Go 源文件数量** | ~148 个 |
| **总代码行数** | ~50,700 行 |
| **测试文件数量** | ~50 个 |
| **测试代码行数** | ~10,000+ 行 |
| **文档文件数量** | 60+ 个 Markdown |
| **依赖模块数量** | 369 个 |

### 1.3 架构层次

```
┌─────────────────────────────────────────────────────────────┐
│                    应用层 (Application Layer)                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │
│  │   Webhook   │  │  Scheduler  │  │   Device Plugin     │   │
│  │  (admission)│  │  (extender) │  │  (gRPC server)      │   │
│  └─────────────┘  └─────────────┘  └─────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                    核心层 (Core Layer)                        │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              Device Abstraction Interface               │ │
│  │  (Devices interface with vendor implementations)        │ │
│  └─────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│                    基础设施层 (Infrastructure Layer)          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │
│  │  Util pkg   │  │  Metrics    │  │  Monitor (vGPU)     │   │
│  │  (client,   │  │  (prometheus)│  │  (in-container)     │   │
│  │   nodelock) │  │             │  │                     │   │
│  └─────────────┘  └─────────────┘  └─────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 目录结构详解

### 2.1 完整目录树

```
HAMi/
├── cmd/                              # 可执行程序入口
│   ├── device-plugin/nvidia/         # NVIDIA Device Plugin
│   │   ├── main.go                   # 入口函数
│   │   ├── root.go                   # Cobra 根命令
│   │   ├── plugin-manager.go         # 插件管理器
│   │   ├── vgpucfg.go                # vGPU 配置
│   │   └── watchers.go               # K8s 资源监听
│   ├── scheduler/                    # 调度器
│   │   ├── main.go                   # 入口 & HTTP 服务器
│   │   ├── metrics.go                # Prometheus 指标
│   │   └── (复用 pkg/scheduler)
│   └── vGPUmonitor/                  # vGPU 监控
│       ├── main.go                   # 入口函数
│       ├── feedback.go               # 资源反馈
│       ├── metrics.go                # 监控指标
│       └── validation.go             # 验证逻辑
│
├── pkg/                              # 核心 Go 包
│   ├── device/                       # 设备管理抽象层
│   │   ├── devices.go                # 设备接口定义 & 公共方法
│   │   ├── pods.go                   # Pod 设备管理
│   │   ├── quota.go                  # 配额管理
│   │   ├── common/                   # 公共常量 & 错误码
│   │   ├── nvidia/                   # NVIDIA GPU 实现
│   │   ├── ascend/                   # 华为昇腾 NPU 实现
│   │   ├── cambricon/                # 寒武纪 MLU 实现
│   │   ├── hygon/                    # 海光 DCU 实现
│   │   ├── iluvatar/                 # 天数智芯 GPU 实现
│   │   ├── mthreads/                 # 摩尔线程 GPU 实现
│   │   ├── metax/                    # 燧原科技 GPU 实现
│   │   ├── amd/                      # AMD GPU 实现
│   │   ├── kunlun/                   # 昆仑芯 XPU 实现
│   │   ├── enflame/                  # 浪潮 GCU 实现
│   │   └── awsneuron/                # AWS Neuron 实现
│   │
│   ├── device-plugin/                # Device Plugin 实现
│   │   └── nvidiadevice/nvinternal/  # 基于 NVIDIA 官方 DP 定制
│   │       ├── cdi/                  # CDI (Container Device Interface)
│   │       ├── imex/                 # IMEX 通道管理
│   │       ├── mig/                  # MIG (Multi-Instance GPU)
│   │       ├── plugin/               # Plugin 核心逻辑
│   │       │   ├── server.go         # gRPC 服务器 (Allocate, ListAndWatch)
│   │       │   ├── register.go       # Kubelet 注册
│   │       │   ├── lock.go           # 节点锁管理
│   │       │   ├── mps.go            # MPS (Multi-Process Service)
│   │       │   ├── util.go           # 工具函数
│   │       │   └── factory.go        # 工厂模式创建
│   │       └── rm/                   # Resource Manager
│   │           ├── devices.go        # 设备管理
│   │           ├── allocate.go       # 资源分配
│   │           ├── health.go         # 健康检查
│   │           └── nvml_devices.go   # NVML 设备操作
│   │
│   ├── scheduler/                    # 调度器核心
│   │   ├── scheduler.go              # 主调度逻辑 (Filter, Bind)
│   │   ├── score.go                  # 评分计算
│   │   ├── webhook.go                # Admission Webhook
│   │   ├── event.go                  # 事件记录
│   │   ├── nodes.go                  # 节点管理
│   │   ├── policy/                   # 调度策略
│   │   │   ├── gpu_policy.go         # GPU 调度策略
│   │   │   └── node_policy.go        # 节点调度策略
│   │   ├── routes/                   # HTTP 路由
│   │   │   ├── predicate.go          # Filter 路由
│   │   │   ├── bind.go               # Bind 路由
│   │   │   └── webhook.go            # Webhook 路由
│   │   └── config/                   # 配置管理
│   │
│   ├── metrics/                      # 指标收集
│   │   └── metrics.go                # Prometheus 指标定义
│   │
│   ├── monitor/                      # 监控组件
│   │   └── nvidia/                   # NVIDIA 监控实现
│   │       ├── v0/                   # v0 版本 API
│   │       └── v1/                   # v1 版本 API
│   │
│   └── util/                         # 工具包
│       ├── util.go                   # 通用工具函数
│       ├── types.go                  # 类型定义
│       ├── client/                   # K8s 客户端
│       ├── nodelock/                 # 节点锁实现
│       ├── leaderelection/           # 领导选举
│       └── flag/                     # 命令行参数
│
├── charts/hami/                      # Helm Chart
│   ├── Chart.yaml                    # Chart 元数据
│   ├── values.yaml                   # 默认配置 (480+ 行)
│   └── templates/                    # K8s 资源模板
│       ├── scheduler/                # 调度器资源
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── configmap.yaml
│       │   └── rbac.yaml
│       └── device-plugin/            # Device Plugin 资源
│           ├── daemonset.yaml
│           ├── service.yaml
│           └── configmap.yaml
│
├── examples/                         # 使用示例
│   ├── nvidia/                       # NVIDIA 示例
│   ├── ascend/                       # 昇腾示例
│   ├── cambricon/                    # 寒武纪示例
│   ├── hygon/                        # 海光示例
│   ├── iluvatar/                     # 天数智芯示例
│   ├── mthreads/                     # 摩尔线程示例
│   ├── metax/                        # 燧原示例
│   ├── kunlun/                       # 昆仑芯示例
│   ├── enflame/                      # 浪潮示例
│   └── awsneuron/                    # AWS Neuron 示例
│
├── docker/                           # Docker 镜像
│   ├── Dockerfile                    # 主镜像
│   ├── Dockerfile.hamicore           # HAMi-Core 镜像
│   ├── Dockerfile.hamimaster         # Master 镜像
│   └── entrypoint.sh                 # 入口脚本
│
├── libvgpu/                          # vGPU 核心库 (C/C++)
│   └── (外部子模块，需单独分析)
│
├── test/                             # 测试代码
│   ├── e2e/                          # E2E 测试
│   └── utils/                        # 测试工具
│
├── docs/                             # 文档
│   ├── develop/                      # 开发文档
│   ├── proposals/                    # 功能提案
│   └── CHANGELOG/                    # 变更日志
│
└── hack/                             # 构建脚本
    ├── tools/                        # 构建工具
    └── boilerplate/                  # 许可证头模板
```

---

## 3. 核心组件分析

### 3.1 设备抽象接口 (`pkg/device/devices.go`)

HAMi 的核心设计是设备抽象接口，所有设备厂商都实现同一套接口：

```go
type Devices interface {
    // 基本信息
    CommonWord() string

    // Pod 准入控制
    MutateAdmission(ctr *corev1.Container, pod *corev1.Pod) (bool, error)

    // 健康检查
    CheckHealth(devType string, n *corev1.Node) (bool, bool)

    // 节点清理
    NodeCleanUp(nn string) error

    // 资源名称
    GetResourceNames() ResourceNames

    // 节点设备获取
    GetNodeDevices(n corev1.Node) ([]*DeviceInfo, error)

    // 节点锁管理
    LockNode(n *corev1.Node, p *corev1.Pod) error
    ReleaseNodeLock(n *corev1.Node, p *corev1.Pod) error

    // 资源请求
    GenerateResourceRequests(ctr *corev1.Container) ContainerDeviceRequest

    // Pod 注解
    PatchAnnotations(pod *corev1.Pod, annoinput *map[string]string, pd PodDevices) map[string]string

    // 节点评分
    ScoreNode(node *corev1.Node, podDevices PodSingleDevice, previous []*DeviceUsage, policy string) float32

    // 资源使用
    AddResourceUsage(pod *corev1.Pod, n *DeviceUsage, ctr *ContainerDevice) error

    // 设备适配
    Fit(devices []*DeviceUsage, request ContainerDeviceRequest, pod *corev1.Pod, nodeInfo *NodeInfo, allocated *PodDevices) (bool, map[string]ContainerDevices, string)
}
```

**关键数据结构**:

```go
// 设备使用情况
type DeviceUsage struct {
    ID          string           // 设备 UUID
    Index       uint             // 设备索引
    Used        int32            // 已使用数量 (时间片)
    Count       int32            // 总数量
    Usedmem     int32            // 已使用显存
    Totalmem    int32            // 总显存
    Usedcores   int32            // 已使用核心数
    Totalcore   int32            // 总核心数
    Mode        string           // 运行模式 (hami-core/mig)
    MigTemplate []Geometry       // MIG 模板
    MigUsage    MigInUse         // MIG 使用情况
    Numa        int              // NUMA 节点
    Type        string           // 设备类型
    Health      bool             // 健康状态
    PodInfos    []*PodInfo       // 绑定的 Pod 信息
    CustomInfo  map[string]any   // 厂商自定义信息
}

// 设备信息（节点注册时使用）
type DeviceInfo struct {
    ID              string
    Index           uint
    Count           int32
    Devmem          int32
    Devcore         int32
    Type            string
    Numa            int
    Mode            string
    MIGTemplate     []Geometry
    Health          bool
    DeviceVendor    string         // 设备厂商
    CustomInfo      map[string]any // 自定义信息
    DevicePairScore DevicePairScore // 设备对拓扑评分
}
```

### 3.2 全局注册表

```go
var (
    // 正在请求的设备资源
    InRequestDevices   map[string]string  // key: 设备类型，value: 注解键名

    // 已支持的设备
    SupportDevices     map[string]string  // key: 设备类型，value: 注解键名

    // 设备实现实例
    DevicesMap         map[string]Devices // key: 设备类型，value: 实现实例

    // 待处理的设备类型列表
    DevicesToHandle    []string
)
```

---

## 4. 设备支持模块

### 4.1 NVIDIA GPU (`pkg/device/nvidia/`)

**文件列表**:
- `device.go` (964 行) - 主要实现
- `calculate_score.go` - 评分计算
- `links.go` - NVLink 拓扑支持

**核心特性**:

1. **资源配置**:
```go
const (
    ResourceCountName  = "nvidia.com/gpu"           // GPU 数量
    ResourceMemoryName = "nvidia.com/gpumem"        // 显存大小 (MB)
    ResourceMemPerc    = "nvidia.com/gpumem-percentage" // 显存百分比
    ResourceCoresName  = "nvidia.com/gpucores"      // 核心百分比
    ResourcePriority   = "nvidia.com/priority"      // 优先级
)
```

2. **注解控制**:
```go
const (
    GPUInUse     = "nvidia.com/use-gputype"      // 指定使用的 GPU 类型
    GPUNoUse     = "nvidia.com/nouse-gputype"    // 指定不使用的 GPU 类型
    NumaBind     = "nvidia.com/numa-bind"        // NUMA 绑定
    GPUUseUUID   = "nvidia.com/use-gpuuuid"      // 指定 UUID
    GPUNoUseUUID = "nvidia.com/nouse-gpuuuid"    // 排除 UUID
    AllocateMode = "nvidia.com/vgpu-mode"        // 分配模式
)
```

3. **Fit 算法流程**:
```
1. 遍历可用设备列表（从后往前，高分在前）
2. 检查设备健康状态
3. 检查 GPU 类型匹配（use/nouse 注解）
4. 检查 UUID 白名单/黑名单
5. 检查时间片是否用完 (Count <= Used)
6. 检查显存是否足够
7. 检查核心是否足够
8. 检查配额是否满足
9. 检查 MIG 模式特殊规则
10. 如需拓扑感知，进行拓扑优化
```

4. **MIG 支持**:
- 支持 `none` 和 `mixed` 模式
- 自动检测 MIG 配置并生成模板
- 动态分配 MIG 实例

### 4.2 华为昇腾 Ascend (`pkg/device/ascend/`)

**文件列表**:
- `device.go` (594 行) - 主要实现
- `vnpu.go` - vNPU 实现

**核心特性**:

1. **资源配置**:
```go
// 支持多种 Ascend 设备类型
ResourceName         = "huawei.com/Ascend910A"
ResourceMemoryName   = "huawei.com/Ascend910A-memory"
```

2. **Ascend910C 特殊处理**:
```go
// Ascend910C 最小分配单位为 2 NPU (1 个物理模块)
if reqNum == 1 {
    reqNum = 2  // 自动向上取整
} else if reqNum%2 != 0 {
    return error  // 拒绝奇数请求
}
```

3. **拓扑感知调度**:
- 基于 NetworkID 进行 HCCS 拓扑优化
- 优先选择同一 Network 内的 NPU
- 910C 优先分配完整模块（2 NPU）

4. **内存模板**:
```go
type VNPUConfig struct {
    MemoryAllocatable int64  // 可分配内存
    MemoryCapacity    int64  // 总内存
    Templates         []struct {
        Name   string
        Memory int64
    }
}
```

### 4.3 其他设备实现对比

| 设备类型 | 包路径 | 资源名称 | 特色功能 |
|----------|--------|----------|----------|
| **Cambricon MLU** | `pkg/device/cambricon/` | `cambricon.com/vmlu` | 支持 vMLU 切片 |
| **HYGON DCU** | `pkg/device/hygon/` | `hygon.com/dcunum` | 兼容 ROCm 生态 |
| **Iluvatar** | `pkg/device/iluvatar/` | `iluvatar.ai/BI-V100-vgpu` | 多卡型支持 |
| **Moore Threads** | `pkg/device/mthreads/` | `mthreads.com/vgpu` | 苏堤核心架构 |
| **MetaX** | `pkg/device/metax/` | `metax-tech.com/sgpu` | QoS 支持 |
| **Kunlun** | `pkg/device/kunlun/` | `kunlunxin.com/xpu` | 拓扑感知 |
| **Enflame** | `pkg/device/enflame/` | `enflame.com/vgcu` | VGCU 百分比分配 |
| **AMD** | `pkg/device/amd/` | `amd.com/gpu` | MIOpen 支持 |
| **AWS Neuron** | `pkg/device/awsneuron/` | `aws.amazon.com/neuron` | Inferentia/Trainium |

---

## 5. 调度器实现

### 5.1 架构概述

```
┌─────────────────────────────────────────────────────────────┐
│                    K8s API Server                           │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                  MutatingWebhook                            │
│  - 识别设备资源请求                                          │
│  - 设置 schedulerName                                        │
│  - 注入环境变量                                              │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              Scheduler Extender                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Filter (Predicate)                                  │    │
│  │  - getNodesUsage() 获取节点使用情况                   │    │
│  │  - calcScore() 计算节点评分                          │    │
│  │  - fitInDevices() 设备适配检查                       │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Score (Bind)                                        │    │
│  │  - 选择最优节点                                      │    │
│  │  - 绑定 Pod 到节点                                    │    │
│  │  - 更新注解和设备状态                                │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 调度流程 (`pkg/scheduler/scheduler.go`)

#### 5.2.1 Filter 阶段

```go
func (s *Scheduler) Filter(args extenderv1.ExtenderArgs) (*extenderv1.ExtenderFilterResult, error) {
    // 1. 获取 Pod 资源请求
    resourceReqs := device.Resourcereqs(args.Pod)

    // 2. 获取节点使用情况
    nodeUsage, failedNodes, err := s.getNodesUsage(args.NodeNames, args.Pod)

    // 3. 计算节点评分
    nodeScores, err := s.calcScore(nodeUsage, resourceReqs, args.Pod, failedNodes)

    // 4. 选择最优节点
    sort.Sort(nodeScores)
    bestNode := nodeScores.NodeList[len(nodeScores.NodeList)-1]

    // 5. 设置注解
    annotations[util.AssignedNodeAnnotations] = bestNode.NodeID
    util.PatchPodAnnotations(args.Pod, annotations)
}
```

#### 5.2.2 节点评分算法

**节点评分结构**:
```go
type NodeScore struct {
    NodeID  string
    Node    *corev1.Node
    Devices device.PodDevices
    Score   float32  // 综合评分
}
```

**评分计算**:
```go
// 默认评分 (基于资源使用率)
func (ns *NodeScore) ComputeDefaultScore(devices DeviceUsageList) {
    used, usedCore, usedMem := 统计已使用资源
    total, totalCore, totalMem := 统计总资源

    useScore := used / total
    coreScore := usedCore / totalCore
    memScore := usedMem / totalMem

    ns.Score = Weight * (useScore + coreScore + memScore)
}

// 策略覆盖评分
func (ns *NodeScore) OverrideScore(previous []*DeviceUsage, policy string) {
    // 根据设备类型的自定义评分累加
    for idx, val := range ns.Devices {
        devScore += device.GetDevices()[idx].ScoreNode(...)
    }
    ns.Score += devScore
}
```

**调度策略**:
| 策略 | Binpack | Spread |
|------|---------|--------|
| **Node 级别** | 优先填满节点 | 分散部署 |
| **GPU 级别** | 优先使用已用 GPU | 优先使用空闲 GPU |
| **实现** | Score 小的在前 | Score 大的在前 |

#### 5.2.3 Bind 阶段

```go
func (s *Scheduler) Bind(args extenderv1.ExtenderBindingArgs) (*extenderv1.ExtenderBindingResult, error) {
    // 1. 锁定节点
    for _, val := range device.GetDevices() {
        val.LockNode(node, current)
    }

    // 2. 更新 Pod 注解为 "allocating"
    util.PatchPodAnnotations(current, tmppatch)

    // 3. 调用 K8s API 绑定 Pod
    s.kubeClient.CoreV1().Pods().Bind(...)

    // 4. 释放节点锁
    for _, val := range device.GetDevices() {
        val.ReleaseNodeLock(node, current)
    }
}
```

### 5.3 设备适配算法 (`fitInDevices`)

```go
func fitInDevices(node *NodeUsage, requests ContainerDeviceRequests, ...) (bool, string) {
    // 1. 对节点内设备进行评分排序
    for index := range node.Devices.DeviceLists {
        node.Devices.DeviceLists[index].ComputeScore(requests)
    }
    sort.Sort(node.Devices)

    // 2. 遍历请求，尝试适配
    for _, k := range requests {
        fit, tmpDevs, reason := device.GetDevices()[k.Type].Fit(...)
        if fit {
            // 3. 更新设备使用状态
            for idx, val := range tmpDevs {
                device.GetDevices()[k.Type].AddResourceUsage(...)
            }
        }
    }
}
```

### 5.4 拓扑感知调度

**NVIDIA GPU 拓扑优化**:
```go
// 最佳组合选择
func computeBestCombination(nodeInfo *NodeInfo, combinations []ContainerDevices) ContainerDevices {
    // 遍历所有组合，选择 NVLink 分数最高的
    for _, partition := range combinations {
        totalScore := 计算组合内设备间 NVLink 总分
        if totalScore > bestScore {
            bestCombination = partition
        }
    }
    return bestCombination
}
```

**Ascend 910C 模块优化**:
```go
func computeBestCombination910C(nodeInfo *NodeInfo, reqNum int, devices ContainerDevices) ContainerDevices {
    // 1. 按物理模块分组 (每卡 2 NPU)
    cardTopology := make(map[int][]int)  // cardId -> [npuIdx1, npuIdx2]

    // 2. 优先选择完整模块
    for _, card := range sortedCards {
        if len(card) == 2 {  // 完整模块
            selectedIndices = append(selectedIndices, card...)
        }
    }
    return result
}
```

---

## 6. Device Plugin 实现

### 6.1 架构概述

HAMi Device Plugin 基于 NVIDIA 官方 `k8s-device-plugin` v0.18.2 定制开发：

```
┌─────────────────────────────────────────────────────────────┐
│                  gRPC Server                                │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Allocate()                                          │    │
│  │  - 从 Scheduler 注解获取分配决策                      │    │
│  │  - 生成容器环境变量                                  │    │
│  │  - 配置 Mount 路径 (libvgpu.so)                       │    │
│  │  - 返回 AllocateResponse                             │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  ListAndWatch()                                      │    │
│  │  - 上报设备列表给 Kubelet                             │    │
│  │  - 健康检查更新                                      │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Register()                                          │    │
│  │  - 向 Kubelet 注册 Plugin                            │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 核心流程 (`pkg/device-plugin/nvidiadevice/nvinternal/plugin/server.go`)

#### 6.2.1 Allocate 实现

```go
func (plugin *NvidiaDevicePlugin) Allocate(ctx context.Context, reqs *AllocateRequest) (*AllocateResponse, error) {
    // 1. 获取当前待分配 Pod
    current, err := util.GetPendingPod(ctx, nodename)

    // 2. 解析 Scheduler 写入的注解
    currentCtr, devreq, err := GetNextDeviceRequest(NvidiaGPUDevice, *current)

    // 3. 生成响应
    response, err := plugin.getAllocateResponse(deviceIDs)

    // 4. 注入环境变量（仅 HAMi-Core 模式）
    if plugin.operatingMode != "mig" {
        response.Envs["CUDA_DEVICE_MEMORY_LIMIT_0"] = fmt.Sprintf("%vm", devreq[0].Usedmem)
        response.Envs["CUDA_DEVICE_SM_LIMIT"] = fmt.Sprint(devreq[0].Usedcores)
        response.Envs["CUDA_DEVICE_MEMORY_SHARED_CACHE"] = cachePath
    }

    // 5. 配置 Mount（注入 libvgpu.so）
    response.Mounts = append(response.Mounts,
        &Mount{
            ContainerPath: fmt.Sprintf("%s/vgpu/libvgpu.so", hostHookPath),
            HostPath:      GetLibPath(),
            ReadOnly:      true,
        },
        &Mount{
            ContainerPath: fmt.Sprintf("%s/vgpu", hostHookPath),
            HostPath:      cacheFileHostDirectory,
        },
    )

    // 6. 更新 Pod 状态
    PodAllocationTrySuccess(nodename, NvidiaGPUDevice, NodeLockNvidia, current)
    return &responses, nil
}
```

#### 6.2.2 设备列表策略

```go
type DeviceListStrategies []DeviceListStrategy

// 支持的策略
const (
    DeviceListStrategyEnvVar        // 环境变量 (NVIDIA_VISIBLE_DEVICES)
    DeviceListStrategyVolumeMounts  // Volume Mounts
    DeviceListStrategyCDICRI        // CDI CRI
    DeviceListStrategyCDIAnnotations // CDI Annotations
)
```

#### 6.2.3 MIG 支持

```go
// 启动时检测 MIG 配置
if deviceSupportMig {
    // 1. 导出当前 MIG 配置
    cmd := exec.Command("nvidia-mig-parted", "export")
    yaml.Unmarshal(outStr, &plugin.migCurrent)

    // 2. 根据模式处理
    if plugin.operatingMode == "mig" {
        HamiInitMigConfig = plugin.processMigConfigs(...)
        plugin.migCurrent.MigConfigs["current"] = HamiInitMigConfig
    } else {
        // 关闭 MIG
        plugin.migCurrent.MigConfigs["current"] = allDisabled
    }

    // 3. 应用 MIG 配置
    plugin.ApplyMigTemplate()
}
```

### 6.3 节点锁机制

```go
// 锁目录
const NodeLockPath = "/run/hami/lock"

// 锁定节点
func LockNode(nodeName, lockType string, pod *corev1.Pod) error {
    lockFile := path.Join(NodeLockPath, lockType)
    // 使用 flock 实现互斥锁
    // 超时时间由配置控制 (默认 5 分钟)
}

// 释放节点锁
func ReleaseNodeLock(nodeName, lockType string, pod *corev1.Pod, force bool) error {
    // 删除锁文件
}
```

---

## 7. Webhook 机制

### 7.1 Admission Webhook (`pkg/scheduler/webhook.go`)

**处理流程**:

```go
func (h *webhook) Handle(_ context.Context, req admission.Request) admission.Response {
    // 1. 解码 Pod
    pod := &corev1.Pod{}
    h.decoder.Decode(req, pod)

    // 2. 检查是否已有其他调度器
    if pod.Spec.SchedulerName != "" && pod.Spec.SchedulerName != DefaultSchedulerName {
        return admission.Allowed("different scheduler")
    }

    // 3. 遍历所有设备，调用 MutateAdmission
    hasResource := false
    for _, val := range device.GetDevices() {
        found, err := val.MutateAdmission(c, pod)
        hasResource = hasResource || found
    }

    // 4. 设置调度器名称
    if hasResource && config.SchedulerName != "" {
        pod.Spec.SchedulerName = config.SchedulerName
    }

    // 5. 检查资源配额
    if !fitResourceQuota(pod) {
        return admission.Denied("exceeding resource quota")
    }

    // 6. 返回 Patch 响应
    return admission.PatchResponseFromRaw(req.Object.Raw, marshaledPod)
}
```

### 7.2 NVIDIA 特定处理

```go
func (dev *NvidiaGPUDevices) MutateAdmission(ctr *corev1.Container, p *corev1.Pod) (bool, error) {
    // 1. 注入优先级环境变量
    if priority, ok := ctr.Resources.Limits[ResourcePriority]; ok {
        ctr.Env = append(ctr.Env, corev1.EnvVar{
            Name:  "TASK_PRIORITY",
            Value: fmt.Sprint(priority.Value()),
        })
    }

    // 2. 注入核心策略环境变量
    if dev.config.GPUCorePolicy != DefaultCorePolicy {
        ctr.Env = append(ctr.Env, corev1.EnvVar{
            Name:  "CUDA_CORE_LIMIT_SWITCH",
            Value: string(dev.config.GPUCorePolicy),
        })
    }

    // 3. 自动填充 GPU 数量
    if hasMemoryRequest && dev.config.DefaultGPUNum > 0 {
        ctr.Resources.Limits[ResourceCountName] = dev.config.DefaultGPUNum
    }

    // 4. 设置 RuntimeClass
    if dev.config.RuntimeClassName != "" {
        p.Spec.RuntimeClassName = &dev.config.RuntimeClassName
    }

    return hasResource, nil
}
```

---

## 8. 配置系统

### 8.1 配置层次

```
┌─────────────────────────────────────────────────────────────┐
│  1. Helm values.yaml (charts/hami/values.yaml)              │
│     - 全局配置                                               │
│     - 镜像配置                                               │
│     - 资源限制                                               │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  2. ConfigMap (scheduler/device-plugin)                     │
│     - 调度策略配置                                           │
│     - Device Plugin 配置                                     │
│     - 节点特定配置                                           │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  3. 命令行参数 (cobra flags)                                │
│     - --node-scheduler-policy                                │
│     - --gpu-scheduler-policy                                 │
│     - --default-mem                                          │
│     - --metrics-bind-address                                 │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 核心配置项

**调度器配置 (`pkg/scheduler/config/config.go`)**:
```go
var (
    SchedulerName         string   // 调度器名称
    DefaultMem            int32    // 默认显存 (MB)
    DefaultCores          int32    // 默认核心百分比
    NodeSchedulerPolicy   string   // 节点调度策略 (binpack/spread)
    GPUSchedulerPolicy    string   // GPU 调度策略 (binpack/spread)
    MetricsBindAddress    string   // Metrics 监听地址
    NodeLabelSelector     map[string]string // 节点标签选择器
    LeaderElect           bool     // 是否启用领导选举
    NodeLockTimeout       time.Duration // 节点锁超时
)
```

**Device Plugin 配置 (`pkg/device/nvidia/device.go`)**:
```go
type NvidiaConfig struct {
    NodeDefaultConfig            // 节点默认配置
    ResourceCountName            string  // 资源计数名称
    ResourceMemoryName           string  // 资源内存名称
    ResourceCoreName             string  // 资源核心名称
    OverwriteEnv                 bool    // 是否覆盖环境变量
    DefaultMemory                int32   // 默认内存
    DefaultCores                 int32   // 默认核心
    DefaultGPUNum                int32   // 默认 GPU 数量
    MemoryFactor                 int32   // 内存因子
    GPUCorePolicy                string  // GPU 核心策略
    RuntimeClassName             string  // Runtime 类名称
    MigGeometriesList            []AllowedMigGeometries // MIG 几何
}

type NodeDefaultConfig struct {
    DeviceSplitCount            *uint    // 设备分割数 (默认 10)
    DeviceMemoryScaling         *float64 // 内存缩放
    DeviceCoreScaling           *float64 // 核心缩放
    PreConfiguredDeviceMemory   *int64   // 预配置设备内存
    LogLevel                    *string  // 日志级别
}
```

### 8.3 节点级配置

支持通过 ConfigMap 为不同节点配置不同参数：

```json
{
  "nodeconfig": [
    {
      "name": "node-1",
      "operatingmode": "hami-core",
      "devicememoryscaling": 1.5,
      "devicesplitcount": 20,
      "migstrategy": "none",
      "filterdevices": {
        "uuid": ["GPU-xxx"],
        "index": [0, 1]
      }
    }
  ]
}
```

---

## 9. 监控与指标

### 9.1 Prometheus 指标 (`pkg/metrics/metrics.go`)

**指标类型**:

| 指标名称 | 类型 | 说明 |
|----------|------|------|
| `hami_gpu_device_count` | Gauge | 节点 GPU 总数 |
| `hami_gpu_device_used` | Gauge | 节点 GPU 已使用数 |
| `hami_gpu_memory_total` | Gauge | GPU 总显存 (MB) |
| `hami_gpu_memory_used` | Gauge | GPU 已使用显存 (MB) |
| `hami_gpu_core_total` | Gauge | GPU 总核心数 |
| `hami_gpu_core_used` | Gauge | GPU 已使用核心数 |
| `hami_pod_scheduled_total` | Counter | 调度 Pod 总数 |
| `hami_pod_schedule_duration_seconds` | Histogram | 调度耗时 |
| `hami_node_lock_wait_duration_seconds` | Histogram | 节点锁等待耗时 |

**指标端点**:
```
http://{scheduler-ip}:9395/metrics
http://{device-plugin-ip}:31992/metrics
```

### 9.2 vGPU Monitor (`cmd/vGPUmonitor/`)

**功能**:
- 实时监控容器内 GPU 使用情况
- 收集 SM 利用率和显存使用率
- 通过反馈机制更新调度器状态

**实现**:
```go
// 监控周期
const ResyncInterval = 5 * time.Minute

// 收集指标
func collectMetrics() {
    // 1. 读取容器内 /sys 文件系统
    // 2. 解析 NVML 数据
    // 3. 上报到 Scheduler
}
```

---

## 10. 测试与质量

### 10.1 测试覆盖

| 包路径 | 测试文件 | 覆盖率估算 |
|--------|----------|-----------|
| `pkg/device/nvidia/` | `device_test.go` | ~70% |
| `pkg/device/amd/` | `device_test.go` | ~60% |
| `pkg/scheduler/` | `scheduler_test.go` | ~65% |
| `pkg/scheduler/` | `score_test.go` | ~75% |
| `pkg/device-plugin/` | `server_test.go` | ~50% |
| `pkg/util/` | `util_test.go` | ~80% |

### 10.2 测试类型

**单元测试**:
```go
// 示例：NVIDIA 设备测试
func TestNvidiaGPUDevices_Fit(t *testing.T) {
    // 构造测试设备
    devices := []*device.DeviceUsage{
        {ID: "GPU-1", Totalmem: 16384, Count: 10},
    }

    // 构造请求
    request := device.ContainerDeviceRequest{
        Nums: 1, Memreq: 4096,
    }

    // 执行适配
    fit, result, _ := nvdev.Fit(devices, request, pod, nodeInfo, nil)

    // 验证结果
    assert.True(t, fit)
    assert.Len(t, result[NvidiaGPUDevice], 1)
}
```

**E2E 测试** (`test/e2e/`):
- Pod 调度测试
- 设备共享测试
- MIG 功能测试
- 故障恢复测试

### 10.3 CI/CD 流程

```yaml
# .github/workflows/ci.yaml
jobs:
  verify:
    - make verify  # 代码格式检查

  test:
    - go test ./... -race -cover  # 单元测试

  build:
    - docker build  # 镜像构建

  scan:
    - trivy image  # 安全扫描
```

---

## 11. 安全分析

### 11.1 安全机制

1. **节点锁机制**:
   - 防止并发分配冲突
   - 超时自动释放（5 分钟）
   - 基于文件锁 (flock)

2. **注解验证**:
   - Webhook 验证资源配额
   - 拒绝特权容器
   - 检查 Scheduler 名称

3. **设备隔离**:
   - 基于 libvgpu.so 的硬限制
   - 显存隔离
   - 核心数限制

### 11.2 潜在风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **注解伪造** | 用户手动修改 Pod 注解 | Webhook 验证 + 准入控制 |
| **节点锁超时** | 5 分钟内无法调度大任务 | 可配置超时时间 |
| **Device Plugin 单点** | 节点级故障影响所有 Pod | 多副本 + 健康检查 |
| **libvgpu 注入** | 恶意替换 libvgpu.so | 只读 Mount + 权限控制 |

### 11.3 安全建议

1. 启用 RBAC 限制 Pod 注解修改权限
2. 配置合理的节点锁超时时间
3. 定期更新 Device Plugin 镜像
4. 监控异常设备分配事件

---

## 12. 性能优化建议

### 12.1 调度性能

**当前瓶颈**:
1. `getNodesUsage()` 全量扫描所有节点
2. `calcScore()` 并发计算但单 Pod 串行
3. 设备列表排序 O(n log n)

**优化建议**:

1. **缓存优化**:
```go
// 建议：使用增量更新替代全量扫描
type NodeUsageCache struct {
    mu sync.RWMutex
    cache map[string]*NodeUsage
    dirty map[string]bool  // 标记需要更新的节点
}
```

2. **并发优化**:
```go
// 建议：多 Pod 并发调度
func (s *Scheduler) FilterMultiple(pods []*corev1.Pod) {
    wg := sync.WaitGroup{}
    sem := make(chan struct{}, maxConcurrency)

    for _, pod := range pods {
        sem <- struct{}{}
        wg.Add(1)
        go func(p *corev1.Pod) {
            defer wg.Done()
            defer func() { <-sem }()
            s.Filter(p)
        }(pod)
    }
    wg.Wait()
}
```

3. **设备列表预排序**:
```go
// 建议：定期后台排序，Filter 时直接使用
go func() {
    ticker := time.NewTicker(10 * time.Second)
    for range ticker.C {
        for _, node := range nodes {
            node.Devices.Sort()  // 后台排序
        }
    }
}()
```

### 12.2 Device Plugin 性能

**当前瓶颈**:
1. gRPC 单线程处理 Allocate
2. NVML 调用频繁

**优化建议**:

1. **连接池**:
```go
// 建议使用连接池复用 NVML 连接
type NVMLPool struct {
    connections chan *nvml.Device
}
```

2. **批量操作**:
```go
// 建议：批量处理设备健康检查
func CheckHealthBatch(devices []Device) []HealthStatus {
    // 一次性获取所有设备状态
}
```

### 12.3 内存优化

**当前问题**:
- `PodDevices` 结构深层嵌套
- 设备列表频繁复制

**优化建议**:
```go
// 使用指针替代值传递
type NodeUsage struct {
    Devices *DeviceUsageList  // 指针替代值
}

// 使用 sync.Pool 复用对象
var deviceUsagePool = sync.Pool{
    New: func() interface{} {
        return &DeviceUsage{}
    },
}
```

---

## 附录

### A. 关键文件索引

| 文件 | 行数 | 说明 |
|------|------|------|
| `pkg/device/devices.go` | 534 | 设备接口定义 |
| `pkg/device/nvidia/device.go` | 964 | NVIDIA 实现 |
| `pkg/device/ascend/device.go` | 594 | 昇腾实现 |
| `pkg/scheduler/scheduler.go` | 730 | 调度器核心 |
| `pkg/scheduler/score.go` | 207 | 评分计算 |
| `pkg/scheduler/webhook.go` | 159 | Webhook 实现 |
| `pkg/device-plugin/.../server.go` | 876 | Device Plugin |

### B. 配置项速查

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `deviceSplitCount` | 10 | 设备分割数 |
| `defaultMem` | 0 | 默认显存 |
| `default-cores` | 0 | 默认核心百分比 |
| `node-scheduler-policy` | binpack | 节点策略 |
| `gpu-scheduler-policy` | spread | GPU 策略 |
| `metrics-bind-address` | :9395 | Metrics 端口 |
| `node-lock-timeout` | 5m | 锁超时 |

### C. 注解速查

| 注解 | 说明 |
|------|------|
| `hami.io/vgpu-devices-allocated` | 已分配设备 |
| `hami.io/vgpu-devices-to-allocate` | 待分配设备 |
| `hami.io/node-handshake` | 节点握手 |
| `nvidia.com/use-gputype` | 使用指定 GPU 类型 |
| `nvidia.com/nouse-gpuuuid` | 排除指定 UUID |

---

**报告结束**

本报告基于 HAMi master 分支代码分析生成，如需了解最新版本，请参考 GitHub 仓库：https://github.com/Project-HAMi/HAMi

---

## 补充：关键代码文件详解

### 核心入口文件分析

#### `cmd/scheduler/main.go` - 调度器入口

这是调度器的启动入口，主要职责：

1. **初始化 Kubernetes 客户端**: 建立与 API Server 的连接
2. **加载设备配置**: 调用 `config.InitDevices()` 初始化所有支持的设备类型
3. **创建调度器实例**: 通过 `scheduler.NewScheduler()` 创建调度器
4. **启动 HTTP 服务**: 提供 `/filter`, `/bind`, `/webhook` 三个核心接口

```go
// 关键启动流程
func start() error {
    // 初始化全局 Kubernetes 客户端
    client.InitGlobalClient(
        client.WithBurst(config.Burst),
        client.WithQPS(config.QPS),
        client.WithTimeout(config.Timeout),
    )

    // 初始化所有设备类型
    config.InitDevices()

    // 创建调度器实例
    sher = scheduler.NewScheduler()

    // 启动节点注解监听（后台 goroutine）
    go sher.RegisterFromNodeAnnotations()

    // 启动调度器主循环
    err = sher.Start()

    // 启动 HTTP 路由
    router.POST("/filter", routes.PredicateRoute(sher))
    router.POST("/bind", routes.Bind(sher))
    router.POST("/webhook", routes.WebHookRoute())
}
```

#### `cmd/device-plugin/nvidia/main.go` - Device Plugin 入口

NVIDIA 设备插件的启动入口，主要职责：

1. **加载配置**: 从命令行参数或配置文件加载配置
2. **初始化 NVML**: 与 NVIDIA 驱动建立连接
3. **启动 gRPC 服务器**: 提供 Device Plugin API
4. **注册到 Kubelet**: 向 Kubelet 注册设备

```go
// 关键启动流程
func start(c *cli.Context, o *options) error {
    // 初始化 Kubernetes 客户端
    client.InitGlobalClient()

    // 创建 kubelet socket 监听
    watcher, err := watch.Files(kubeletSocketDir)

    // 加载配置
    config, err := loadConfig(c, o.flags)

    // 初始化 NVML 库
    nvmllib := nvml.New(
        nvml.WithLibraryPath(driverRoot.tryResolveLibrary("libnvidia-ml.so.1")),
    )

    // 获取插件列表并启动
    plugins, err := GetPlugins(c.Context, infolib, nvmllib, devicelib, &devConfig)
    for _, p := range plugins {
        p.Start(o.kubeletSocket)
    }
}
```

### 设备抽象层深度解析

#### `pkg/device/devices.go` - 设备接口核心

这是整个项目最核心的设计，定义了 `Devices` 接口：

**接口方法详解**:

| 方法 | 调用时机 | 主要职责 |
|------|----------|----------|
| `CommonWord()` | 初始化时 | 返回设备类型的唯一标识符 |
| `MutateAdmission()` | Webhook 阶段 | 修改 Pod 的环境变量、资源请求等 |
| `CheckHealth()` | 定时巡检 | 检查节点设备是否健康 |
| `NodeCleanUp()` | 节点删除时 | 清理节点相关的设备信息 |
| `GetResourceNames()` | 资源解析时 | 返回该设备使用的 K8s 资源名称 |
| `GetNodeDevices()` | 节点注册时 | 从节点注解读取设备信息 |
| `LockNode()` | Bind 阶段 | 锁定节点防止并发调度 |
| `ReleaseNodeLock()` | Bind 完成/失败 | 释放节点锁 |
| `GenerateResourceRequests()` | Filter 阶段 | 解析容器的资源请求 |
| `PatchAnnotations()` | Filter 阶段 | 将分配结果写入 Pod 注解 |
| `ScoreNode()` | Filter 阶段 | 计算节点得分 |
| `AddResourceUsage()` | Filter 阶段 | 更新设备使用情况 |
| `Fit()` | Filter 阶段 | 检查设备是否满足请求 |

**核心数据流转**:

```
1. Device Plugin 发现设备
   └── 写入节点注解 (GetNodeDevices 读取)

2. Scheduler 轮询节点注解
   └── 存入内存缓存 (nodeManager.nodes)

3. Pod 提交到 Webhook
   └── MutateAdmission 注入配置

4. Scheduler Filter
   └── GenerateResourceRequests 解析请求
   └── Fit 检查资源是否满足
   └── AddResourceUsage 更新使用情况

5. Scheduler Bind
   └── LockNode 锁定节点
   └── PatchAnnotations 写入分配结果

6. Device Plugin Allocate
   └── 读取 Pod 注解，分配设备
```

#### `pkg/device/nvidia/device.go` - NVIDIA 实现详解

NVIDIA GPU 设备的具体实现，是最复杂也是最核心的实现：

**配置结构**:

```go
type NvidiaConfig struct {
    ResourceCountName            string  // "nvidia.com/gpu"
    ResourceMemoryName           string  // "nvidia.com/gpumem"
    ResourceCoreName             string  // "nvidia.com/gpucores"
    ResourceMemoryPercentageName string  // "nvidia.com/gpumem-percentage"
    DefaultMemory                int32   // 默认显存分配
    DefaultCores                 int32   // 默认算力分配
    DefaultGPUNum                int32   // 默认 GPU 数量
    GPUCorePolicy                GPUCoreUtilizationPolicy // 算力隔离策略
    RuntimeClassName             string  // 容器运行时类
    MigGeometriesList            []AllowedMigGeometries  // MIG 配置
}
```

**Fit 算法详解**:

```go
func (nv *NvidiaGPUDevices) Fit(devices []*device.DeviceUsage, request ContainerDeviceRequest, ...) (bool, map[string]ContainerDevices, string) {
    // 1. 遍历设备（从高分到低分）
    for i := len(devices) - 1; i >= 0; i-- {
        dev := devices[i]

        // 2. 健康检查
        if !dev.Health { continue }

        // 3. GPU 类型匹配（支持注解过滤）
        found, numa := nv.checkType(pod.GetAnnotations(), *dev, request)
        if !found { continue }

        // 4. UUID 过滤
        if !device.CheckUUID(pod.GetAnnotations(), dev.ID, GPUUseUUID, GPUNoUseUUID, ...) { continue }

        // 5. 时间片检查
        if dev.Count <= dev.Used { continue }

        // 6. 显存检查
        if dev.Totalmem - dev.Usedmem < memreq { continue }

        // 7. 算力检查
        if dev.Totalcore - dev.Usedcores < request.Coresreq { continue }

        // 8. 独占检查（算力 100% 时独占）
        if dev.Totalcore == 100 && request.Coresreq == 100 && dev.Used > 0 { continue }

        // 9. MIG 特殊处理
        if !nv.CustomFilterRule(allocated, request, tmpDevs, dev) { continue }

        // 10. 分配设备
        tmpDevs[k.Type] = append(tmpDevs[k.Type], device.ContainerDevice{...})
    }

    // 11. 拓扑感知调度（如需要）
    if needTopology {
        // 选择最优 GPU 组合
        combination := computeBestCombination(nodeInfo, combinations)
    }
}
```

### 调度器核心解析

#### `pkg/scheduler/scheduler.go` - 调度主逻辑

**核心结构**:

```go
type Scheduler struct {
    *nodeManager                        // 节点管理
    podManager    *device.PodManager    // Pod 管理
    quotaManager  *device.QuotaManager  // 配额管理
    leaderManager leaderelection.LeaderManager  // 选主

    kubeClient  kubernetes.Interface
    podLister   listerscorev1.PodLister
    nodeLister  listerscorev1.NodeLister

    cachedstatus map[string]*NodeUsage    // 缓存的节点状态
    overviewstatus map[string]*NodeUsage  // 全量节点状态
}
```

**Filter 流程详解**:

```go
func (s *Scheduler) Filter(args extenderv1.ExtenderArgs) (*extenderv1.ExtenderFilterResult, error) {
    // 1. 解析 Pod 资源请求
    resourceReqs := device.Resourcereqs(args.Pod)

    // 2. 获取所有节点的设备使用情况
    nodeUsage, failedNodes, err := s.getNodesUsage(args.NodeNames, args.Pod)

    // 3. 计算节点评分
    nodeScores, err := s.calcScore(nodeUsage, resourceReqs, args.Pod, failedNodes)

    // 4. 排序选择最优节点
    sort.Sort(nodeScores)
    bestNode := nodeScores.NodeList[len(nodeScores.NodeList)-1]

    // 5. 更新 Pod 注解
    annotations[util.AssignedNodeAnnotations] = bestNode.NodeID

    // 6. 各设备写入分配结果
    for _, val := range device.GetDevices() {
        val.PatchAnnotations(args.Pod, &annotations, bestNode.Devices)
    }

    // 7. 更新内存缓存
    s.podManager.AddPod(args.Pod, bestNode.NodeID, bestNode.Devices)
    s.quotaManager.AddUsage(args.Pod, bestNode.Devices)

    // 8. 持久化到 K8s
    util.PatchPodAnnotations(args.Pod, annotations)

    // 9. 返回唯一节点
    return &extenderv1.ExtenderFilterResult{NodeNames: &[]string{bestNode.NodeID}}, nil
}
```

**节点评分算法**:

```go
func (s *Scheduler) calcScore(nodeUsage *map[string]*NodeUsage, requests device.PodDeviceRequests, ...) (*policy.NodeScoreList, error) {
    for nodeID, node := range *nodeUsage {
        nodeScore := &policy.NodeScore{NodeID: nodeID, Node: node.Node}

        // 对节点内设备进行评分
        for _, k := range requests {
            // 调用设备特定的 Fit 方法
            fit, tmpDevs, reason := device.GetDevices()[k.Type].Fit(
                node.Devices.GetDevices(),
                k,
                pod,
                nodeInfo,
                allocated,
            )

            if fit {
                // 计算得分
                nodeScore.Devices[k.Type] = tmpDevs
            }
        }

        // 应用调度策略
        switch policy {
        case "binpack":
            // 优先使用已有负载的节点/设备
            nodeScore.Score = 计算资源使用率得分
        case "spread":
            // 优先使用空闲的节点/设备
            nodeScore.Score = 计算资源空闲率得分
        }
    }
}
```

### Webhook 机制详解

#### `pkg/scheduler/webhook.go`

**处理流程**:

```go
func (h *webhook) Handle(_ context.Context, req admission.Request) admission.Response {
    pod := &corev1.Pod{}
    h.decoder.Decode(req, pod)

    // 1. 检查调度器名称
    if pod.Spec.SchedulerName != "" &&
       pod.Spec.SchedulerName != corev1.DefaultSchedulerName &&
       pod.Spec.SchedulerName != config.SchedulerName {
        return admission.Allowed("different scheduler")
    }

    // 2. 遍历容器，调用设备特定的 MutateAdmission
    hasResource := false
    for idx, ctr := range pod.Spec.Containers {
        c := &pod.Spec.Containers[idx]

        // 跳过特权容器
        if ctr.SecurityContext != nil && *ctr.SecurityContext.Privileged {
            continue
        }

        for _, dev := range device.GetDevices() {
            found, err := dev.MutateAdmission(c, pod)
            hasResource = hasResource || found
        }
    }

    // 3. 设置调度器名称
    if hasResource && config.SchedulerName != "" {
        pod.Spec.SchedulerName = config.SchedulerName
    }

    // 4. 检查资源配额
    if !fitResourceQuota(pod) {
        return admission.Denied("exceeding resource quota")
    }

    // 5. 返回修改后的 Pod
    return admission.PatchResponseFromRaw(req.Object.Raw, marshaledPod)
}
```

### 配置加载流程

#### `pkg/scheduler/config/config.go`

**初始化流程**:

```go
func InitDevices() {
    // 1. 从配置文件加载
    config, err := LoadConfig(configFile)

    // 2. 初始化各设备
    InitDevicesWithConfig(config)
}

func InitDevicesWithConfig(config *Config) error {
    device.DevicesMap = make(map[string]device.Devices)

    // 注册 NVIDIA 设备
    device.DevicesMap[nvidia.NvidiaGPUDevice] = nvidia.InitNvidiaDevice(config.NvidiaConfig)

    // 注册其他设备
    device.DevicesMap[cambricon.CambriconMLUDevice] = cambricon.InitMLUDevice(config.CambriconConfig)
    device.DevicesMap[hygon.HygonDCUDevice] = hygon.InitDCUDevice(config.HygonConfig)
    // ... 更多设备

    return nil
}
```

### 资源请求解析

#### `pkg/device/devices.go` - Resourcereqs

```go
func Resourcereqs(pod *corev1.Pod) (counts PodDeviceRequests) {
    counts = make(PodDeviceRequests, len(pod.Spec.Containers))

    for i := range pod.Spec.Containers {
        devices := GetDevices()
        counts[i] = make(ContainerDeviceRequests)

        for idx, val := range devices {
            // 调用各设备的 GenerateResourceRequests
            request := val.GenerateResourceRequests(&pod.Spec.Containers[i])
            if request.Nums > 0 {
                counts[i][idx] = request
            }
        }
    }
    return counts
}
```

### 节点锁机制

#### `pkg/util/nodelock/`

节点锁用于防止同一节点被多个 Pod 同时调度导致资源竞争：

```go
const NodeLockPath = "/run/hami/lock"
var NodeLockTimeout = 5 * time.Minute

func LockNode(nodeName, lockType string, pod *corev1.Pod) error {
    // 通过节点注解实现分布式锁
    // 格式: "namespace/podname"
    lockValue := fmt.Sprintf("%s/%s", pod.Namespace, pod.Name)

    // 更新节点注解
    return util.PatchNodeAnnotations(node, map[string]string{
        NodeLockKey: lockValue,
    })
}

func ReleaseNodeLock(nodeName, lockType string, pod *corev1.Pod, force bool) error {
    // 删除锁注解
    return util.PatchNodeAnnotations(node, map[string]string{
        NodeLockKey: "",
    })
}
```

---

## 快速理解项目

### 一句话总结

**HAMi 是一个 Kubernetes 调度器扩展器，通过统一的设备抽象接口，实现多种异构设备（GPU/NPU/MLU等）的虚拟化和共享调度。**

### 核心概念

| 概念 | 解释 |
|------|------|
| **vGPU** | 虚拟 GPU，将物理 GPU 切分成多个虚拟实例 |
| **时间片** | GPU 使用的时间份额，决定共享程度 |
| **显存隔离** | 通过 libvgpu.so 限制容器可见的显存大小 |
| **算力隔离** | 限制容器可使用的 SM (Streaming Multiprocessor) 比例 |
| **MIG** | NVIDIA Multi-Instance GPU，硬件级 GPU 实例化 |

### 如何使用

```yaml
# 示例：申请 1 个 vGPU，3GB 显存，30% 算力
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: gpu-container
      resources:
        limits:
          nvidia.com/gpu: 1          # 申请 1 个 vGPU
          nvidia.com/gpumem: 3000    # 申请 3GB 显存
          nvidia.com/gpucores: 30    # 申请 30% 算力
```

### 代码阅读路径

**推荐阅读顺序**:

1. **入口点**: `cmd/scheduler/main.go` → 了解启动流程
2. **设备接口**: `pkg/device/devices.go` → 理解抽象设计
3. **NVIDIA 实现**: `pkg/device/nvidia/device.go` → 学习具体实现
4. **调度逻辑**: `pkg/scheduler/scheduler.go` → 掌握调度流程
5. **Webhook**: `pkg/scheduler/webhook.go` → 理解准入控制
6. **配置管理**: `pkg/scheduler/config/config.go` → 了解配置加载

---

**报告结束**

本报告基于 HAMi master 分支代码分析生成，如需了解最新版本，请参考 GitHub 仓库：https://github.com/Project-HAMi/HAMi
