<!-- SPDX-License-Identifier: Apache-2.0 -->

# 5. 验证行为与兼容性

[English](../05-validation-matrix.md)

自定义回调打印出一条消息，并不意味着它已经通过验证。验证必须证明：预期的回调确实
执行了；关闭自定义功能时的原生行为保持不变；对象族仍然类型安全；不支持的路径按照
设计失败。

## 5.1 定义 profile，而不是笼统声称“兼容 verbs”

为自定义实现的每个版本创建一张带版本号的支持表。

| 范围 | 操作/模式 | 对象表示 | 回调策略 | 所需证据 |
|---|---|---|---|---|
| Context | 原生模式 | 原生 | `NATIVE` | 基线比较和已加载对象证明 |
| QP | 经典 RC 创建/查询/修改/销毁 | 原生或自定义 | `WRAP_NATIVE`、`CUSTOM` 或 `REJECTED` | 生命周期测试 |
| QP | 扩展创建、`open_qp`、导入 context 路径 | 原生、自定义或无对象 | `NATIVE`、`ADAPTED` 或 `REJECTED` | 逐 mask 负面测试 |
| 发送 | opcode、链表长度、SGE 数量、flags | 取决于 QP profile | `WRAP_NATIVE`、`CUSTOM` 或 `REJECTED` | 返回结果和 `bad_wr` 测试 |
| 接收 | 经典路径/SRQ、缓冲区限制 | 取决于 QP/SRQ profile | 显式策略 | 数据和错误语义测试 |
| CQ | 经典 poll/destroy | 原生或自定义 | `WRAP_NATIVE`、`CUSTOM` 或 `REJECTED` | 批处理和生命周期测试 |
| CQ | ex/event/notify/resize/modify | 原生、自定义或无对象 | 显式策略 | 创建和事件测试 |
| MR | classic/ex/dmabuf/import | 原生、自定义或无对象 | 显式策略 | 所有权和生命周期测试 |
| 并发 | 线程/context/设备/fork | 特定于 profile | `SUPPORTED` 或 `REJECTED` | 同步和隔离测试 |

绝不能把“尚未测试”当作“已经支持”的同义词。应将 `UNTESTED` 作为独立状态，并从兼容性声明中排除。

对于扩展 QP 和 CQ profile，除了 context 操作表，还要验证对象本地的 WR-builder 和轮询回调。在扩展路径测试中，经典回调计数器保持为零，可能恰好证明该路径绕过了经典 overlay，而不是证明没有发生任何操作。

为每个对象族命名其表示 profile：

| Profile | 对象表示 | 回调行为 |
|---|---|---|
| 原生对照 | 原生 | 原生回调 |
| 原生对象包装 | 原生 | 自定义 wrapper 委托给保存的原生回调 |
| 严格自定义对象族 | 自定义 | 整个对象族使用自定义回调或显式拒绝 |
| 注册表混合对象族 | 原生和自定义 | 在原生/自定义分派前由注册表完成分类 |

## 5.2 阶段 1：源码与构建检查

- [ ] 已记录源码 tag、commit、provider 私有 ABI 和编译器。
- [ ] provider target 能从干净目录完成构建。
- [ ] 自定义改动引入的 warning 被视为失败。
- [ ] 不会因为意外加载另一个系统库而掩盖 unresolved symbol。
- [ ] 自定义 overlay 在初始化后保持不变。
- [ ] 支持矩阵列出了每一个非空的自定义字段。
- [ ] 在 provider 和测试环境允许时，提供启用 sanitizer 或等效机制的 debug 构建。

仅编译 CI 很有价值，但它不能证明匹配的设备实际选择了这个 provider。

## 5.3 阶段 2：provider 选择检查

对原生模式和自定义模式分别记录：

- 可执行文件路径；
- 实际加载的 libibverbs 路径；
- 实际加载的 provider 路径和私有 ABI suffix；
- provider build identity；
- 选中的 context 模式；
- 私有实验记录中的设备和端口。

如果预期的自定义回调计数为零，测试必须失败；如果在原生模式下出现自定义回调，测试同样必须失败。

## 5.4 阶段 3：关闭自定义功能时的等价性

关闭自定义功能后：

1. 枚举并查询设备；
2. 创建和销毁目标对象族；
3. 运行实验环境支持的原生 smoke traffic；
4. 覆盖受 overlay 顺序影响的所有硬件变体；
5. 将返回值、completion 和诊断信息与未修改的固定版本构建进行比较。

目标是语义等价，而不是逐 cycle 性能完全相同。在测试自定义模式之前，任何差异都必须得到解释。

## 5.5 阶段 4：对象族闭包

### QP 对象族

- 在原生对象包装模式下，原生构造器和析构器保持不变；自定义回调只接受原生对象表示，
  然后再执行委托；
- 在严格自定义对象族模式下，classic create 到达自定义构造器，且 query、modify、post
  和 destroy 都能识别由它创建的对象；
- 在注册表混合模式下，每一条可达的创建路径都必须在对象到达路由回调之前登记正确的对象类别；
- 每一个 extended 构造器都必须遵循其 profile、能够无损适配，或者尽早失败；
- `open_qp`、provider-specific 创建以及在 imported context 上的使用，都必须明确遵循所选 profile；
- QP 不能关联不兼容的 CQ、SRQ、PD 或 context；
- 创建失败后不能残留已登记或部分存活的对象；
- 除非存在显式的延迟回收或 tombstone 契约，否则不要声称无效的 double-destroy/use-after-destroy 行为受到支持。

### CQ 对象族

- 经典 create/poll/destroy 构成完整路径；
- 扩展 create 已实现或被拒绝；
- event channel、notify、resize、modify 和 CQ event 的行为均有明确定义；
- 除非一个自定义 CQ 确实是合法的原生 CQ，否则它绝不能到达要求更大原生对象布局的原生回调；
- poll 与 destroy 之间的竞争受到控制。

### SRQ 和 MR 对象族

- 不支持的 SRQ 在与自定义 QP 建立关联之前就被拒绝；
- 使用 MR 前验证其类别以及所属 PD/context；
- imported memory 和 dmabuf memory 遵循显式策略；
- 如果没有明确的生命周期机制，unregister/deregister 不能与进行中的操作竞争。

## 5.6 阶段 5：公开 API 语义

对每一种受支持的操作测试：

- 成功路径和每一种可达错误的返回约定；
- 第一个失败的 WR 和精确的 `bad_wr` 指针；
- 多 WR 链表的部分成功；
- 零长度和支持的最大长度；
- 支持和不支持的 SGE 数量；
- flags、signaling、inline 行为和 opcode；
- 完整位宽的不透明 `wr_id` 往返；
- 使用 `ne = 0`、`1` 和一个批次进行轮询；
- completion 的 status/opcode/length/identity；
- 每个 QP 内的顺序，以及在作出相应承诺时共享资源之间的顺序；
- receive-before-send 和 send-before-receive 行为；
- 缓冲区过小时的行为；
- flush completion 和 failure completion。

在支持矩阵旁记录与原生行为存在的每一项语义差异。

## 5.7 阶段 6：未知句柄、归属于其他 context 的句柄，以及陈旧的异步工作

负面测试应传入以下 handle：

- 在混合模式下由可达构造器创建的合法原生对象；
- 在公开 API 所接受的跨 context 关系中使用的合法对象；
- 通过受控内部 fault-injection hook 刻意不登记到注册表中的合法 provider 对象。

严格模式应通过构造器让这些路径不可达，并在 debug 构建中断言该不变量。混合模式必须
在 downcast 之前查询一个仍然有效的 handle，并对未分类条目 fail closed。不要通过把
一种公开对象强制转换成另一种对象族，或伪造任意指针来创建测试；在 provider 开始分类
之前，这些做法就已经违反了公开 API。

应单独测试陈旧的*自定义命令和 completion*：给这些记录附加 generation，销毁并重新创建逻辑对象，然后证明旧 generation 的工作会被拒绝。不要传入已经释放的 C handle，再把“没有崩溃”解释成 provider 是安全的；公开 verb 可能在 wrapper 运行前就解引用该 handle。

在实际可行时运行 ASan/UBSan 或等效 instrumentation。“没有崩溃”不能证明无效的 container conversion 是安全的。

## 5.8 阶段 7：并发与失败

至少覆盖：

- 使用不同模式并发创建 context；
- 在支持的对象上并发 post/poll；
- 如果明确支持并发 destroy，则在延迟生命周期设计下测试受控的
  create/post/poll/destroy 并发交错；
- 否则，测试必须执行已记录的调用方同步边界，且不得对已经释放的 handle 调用公开 verb；
- backend queue 满和有界 backpressure；
- 调用期间 backend 断开；
- 对象销毁后到达陈旧 completion；
- context 关闭时仍有存活 child；
- 两个设备，或者明确拒绝多设备；
- 支持 `fork()`，或者在 fork 后返回明确的不支持结果。

测试应使用确定性 barrier，而不是只依赖压力测试的时序。随后再加入持续时间更长的压力测试，以发现建模交错之外的问题。

## 5.9 阶段 8：硬件与性能

只有在构建、loader、native-off、闭包和负面测试均通过后，才能开始硬件测试。先运行有文档记录的 upstream smoke test，并从最小自定义 profile 开始。

性能声明需要独立的方法学：

- 完全相同的硬件、firmware、kernel、provider 构建 flags、CPU affinity 和链路配置；
- native-off 基线和未修改基线；
- 多次运行和明确的聚合方法；
- 分开报告 setup 成本和 steady-state 成本；
- 除 latency/throughput 外，同时报告 CPU 和内存成本；
- 可以合法公开的原始数据与脚本。

本教程不作任何性能声明。

## 5.10 发布验收标准

只有满足以下条件，自定义实现才可以用于受控实验室发布：

1. 源码 identity 与实际加载的 provider identity 一致；
2. 在已测试 profile 中，原生模式与未修改的对照实现一致；
3. 每一个自定义操作表字段都有闭合的对象路径和错误路径；
4. 不支持的构造器和属性会被明确拒绝；
5. 没有回调递归调用它对应的公开 verb；
6. 合法但未分类的对象在 downcast 之前失败，陈旧的自定义异步记录通过 generation 被拒绝；
7. teardown 和 context close 具有确定的所有权，并说明调用方同步要求；
8. 并发测试和 backend failure 测试通过；
9. 支持矩阵与测试一致，且不对原始指针作出 UAF 安全保证；
10. 至少成功执行过一次 rollback。

下一章：[处理版本与 vendor fork](06-versioning-and-portability.md)。
