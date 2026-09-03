<!-- SPDX-License-Identifier: Apache-2.0 -->

# 0. 范围与术语

[English](../00-scope-and-terminology.md)

本章精确定义我们所定制的对象。这一区分可以避免把一个小型用户态实验误称为新
verb、内核驱动或可移植的 libibverbs 扩展——它实际上并不属于这些类别。

## 0.1 五个层次

RDMA 应用通常会与多个层次交互，而这些层次各自具有不同的稳定性和审查规则。

| 层次 | 典型内容 | 与本教程相关的稳定性 |
|---|---|---|
| 应用程序 API（application API） | `ibv_*`、`rdma_*` 以及 provider-specific 公共 API | 公开源码/API；通常期望保持兼容性 |
| libibverbs 核心层（libibverbs core） | 设备发现、provider 加载、公共对象处理、命令编组 | 内部实现以及导出的 ABI |
| 用户态 provider（userspace provider） | 设备匹配、上下文（context）初始化、对象实现、数据路径回调（callback） | provider 私有源码；与版本紧密绑定 |
| 内核 RDMA 用户态 API（kernel RDMA uAPI） | uverbs ioctl/write 命令以及内核驱动行为 | 需要用户态与内核协调维护的契约 |
| NIC 与固件（firmware） | 队列、doorbell、完成格式、设备命令 | 依赖具体硬件与固件 |

本教程修改的是 **用户态 provider（userspace provider）**，不会新增内核 uAPI 或
固件命令。把结果描述成“没有修改驱动”会产生歧义：内核驱动或许没有改变，但
用户态 provider 确实已经被定制。

## 0.2 三类常被称为“custom verbs”的项目

编辑源码之前，先对预期结果进行分类。

### A. 覆盖或增强现有 provider 操作

例如，校验现有 QP 请求、收集 provider 本地遥测信息、把受支持的操作路由到自定义
用户态 runtime，或为某个 context 选择不同实现。

这是本教程讨论的主题。现有公共 verb 仍进入已有的 provider operation slot；这并不
意味着会新增应用程序符号或内核命令。

### B. 添加 provider-specific 用户态 API

如果应用必须显式请求新行为，那么有文档说明的 provider-specific API 可能比透明
替换 callback 更清晰。在 mlx5 生态中，`mlx5dv_*` API 就属于这一设计空间。一个真正
的公共 API 还需要处理头文件、符号/版本映射、文档、兼容性承诺以及上游审查。

本教程会提及这一替代方案，但不会设计这样的 API。

### C. 添加新的内核或硬件 verb

如果功能需要新的 uverbs 命令、内核对象、设备命令或固件行为，仅修改 provider 并
不够。内核与 rdma-core 的变更必须一起设计、一起审查。上游 rdma-core 明确记录了
这一协同要求。

这条路径不在本教程范围内。

## 0.3 本教程使用的术语

**基础操作表（base ops）**
: 在应用本教程的定制之前，provider 为某个 context 提供的正常 operation callback。

**变体覆盖层（variant overlay）**
: provider 针对某种硬件或 ABI 变体应用的稀疏操作表（sparse operation table）。上游 mlx5 会采用这一模式，为
  特定 CQE 格式安装轮询 callback。

**自定义覆盖层（custom overlay）**
: 只包含本次定制有意替换的 callback，且保持不可变的稀疏操作表。

**原生 callback / 底层实现（native callback / underlay）**
: 如果没有 custom overlay，本应处理该操作的有效 provider callback。这里强调“有效”，
  是因为它可能已经包含某个硬件特定变体。

**公共句柄（public handle）**
: 应用可见的对象指针，例如 `ibv_qp *` 或 `ibv_cq *`。

**Provider 对象（provider object）**
: 包含或支撑某个 public handle、由 provider 拥有的内存分配对象。

**对象族闭包（object-family closure）**
: 对一个公共对象而言，所有可达的构造器、消费者、查询、修改、事件路径和析构器，
  都对该对象的具体类型与语义达成一致这一性质。

**原生对象包装模式（native-object wrapper mode）**
: 自定义 callback 在普通原生 provider 对象上观察、校验或增强某项操作，然后委托给
  有效的原生 callback。该模式不会引入自定义对象表示。

**严格模式（strict mode）**
: 对每个被拦截的对象族，自定义 context 只允许自定义对象。对于不受支持的构造器，
  应直接失败，而不是创建之后可能进入自定义 callback 的原生对象。

**混合模式（mixed mode）**
: 同一个 context 中可以同时存在原生对象和自定义对象。注册表（registry）必须在任何向下转型
  之前识别句柄，再把操作路由到保存的原生 callback 或自定义 callback。

**失败关闭（fail closed）**
: 当一个仍然有效、但尚未分类或来自其他 context 的对象合法到达 callback，或者操作/
  属性不受支持时，显式返回错误。对于携带 generation 的自定义异步记录，应按
  generation 拒绝陈旧记录。这并不能让已经释放的公共 C 指针变得安全。

## 0.4 这一设计可以和不可以作出的声明

Callback overlay 可以如实声称：在一个自定义 provider 构建中，选定操作已被重定向。
仅凭 callback overlay 本身，不能声称：

- 它是在任意已安装 provider 上进行叠加的稳定插件接口；
- 它完整兼容所有 verbs；
- 它对 inline、extended 和 provider-specific API 都保持应用透明；
- 它可以与缺少所需自定义 runtime 的对端保持 wire compatibility；
- wrapper 没有实际实现和测试过的零拷贝、隔离、顺序或恢复能力；
- 它可以跨 rdma-core、MLNX_OFED、固件或硬件版本移植。

应把以上每一项都当作独立的工程声明，并分别提供能力矩阵和测试。

## 0.5 前置知识

读者应熟悉：

- C 函数指针与 container 风格的对象布局；
- QP、CQ、PD、MR、WR、WC 以及基本 RC SEND/RECV 术语；
- Linux 上的 CMake/Ninja 或 Make；
- 动态库选择问题的调试；
- 使用一次性机器或专用实验节点开展 RDMA 实验。

本教程不需要任何私有实现。教程所使用的全部源码位置均来自
[公开参考资料](../REFERENCES.md)。

## 0.6 选择能够满足需求的最小机制

选择 provider interposition 之前，按以下顺序判断：

1. 这一行为能否完全放在应用程序中？如果可以，优先采用应用程序库。
2. 应用是否需要一个显式的 provider 功能？考虑使用有文档说明的 provider-specific
   API。
3. 需求是否是在一个受控 provider 构建中改变现有对象行为？context-scoped ops overlay
   可能适用。
4. 是否需要新的内核/硬件原语？停止 provider-only 方案，转而设计相互配套的内核和
   rdma-core 变更。

Provider interposition 很强大，因为它位于公共调用点之下；同时，它也继承了 provider
私有 ABI、生命周期和硬件假设。应当因为这些权衡与部署需求匹配而采用它，而不是仅仅
因为替换函数指针看起来很方便。

下一章：[追踪分派（dispatch）路径](01-trace-the-dispatch-path.md)。
