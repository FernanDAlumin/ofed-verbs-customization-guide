<!-- SPDX-License-Identifier: Apache-2.0 -->

# 2. 设计上下文级（context-scoped）ops overlay

[English](../02-design-an-ops-overlay.md)

上游提供的组合机制很简单，但围绕它的设计契约并不简单。应先明确行为与对象所有权，
再编写稀疏 overlay。

## 2.1 先编写行为契约

对于每一个考虑替换的操作，记录以下内容：

| 问题 | 答案格式示例 |
|---|---|
| 目的 | 观察、校验、增强、重路由或虚拟化 |
| 启用范围 | 一个 context、设备、进程或全部 context |
| 支持的对象类型 | 由自定义对象族创建的 classic RC QP |
| 支持的属性 | 明确的 mask 与取值范围 |
| 原生委托 | 在自定义工作之前、之后、按条件执行，或永不委托 |
| 返回契约 | 精确的 errno/返回值以及请求链表部分成功行为 |
| 完成契约 | opcode、status、ordering、byte length、opaque ID |
| 并发 | 锁、原子操作、所有权和允许同时发生的调用 |
| 失败行为 | fail closed、错误完成或原生回退 |
| 清理 | 如何停止新工作并排空进行中的工作 |

如果填写这张表时仍需使用“可能是原生对象”或“忽略这个 flag”之类的表述，就不应把
该操作宣传为已支持。

## 2.2 优先采用稀疏且不可变的 overlay

由于 `verbs_set_ops()` 会为 null 字段保留已有 callback，因此 custom table 只需声明
自己负责的操作。

```c
/* PSEUDOCODE — NOT COMPILABLE */

static const operation_table tutorial_overlay = {
    .create_endpoint  = tutorial_create_endpoint,
    .submit_send      = tutorial_submit_send,
    .query_endpoint   = tutorial_query_endpoint,
    .destroy_endpoint = tutorial_destroy_endpoint,
};
```

这段代码有意保持为示意形式，不会复现完整上游结构或任何私有实现。

overlay 构造完成后应保持不变。当其他线程可能正在通过它进行分派时，不要因环境配置
变化而修改进程全局表。配置应放入 context-local 状态；选择了不可变 callback 之后，
callback 代码可以读取该状态。

## 2.3 在正常 provider 初始化之后安装

应用自定义策略之前，provider 应当先完成正常的基础设置和硬件变体设置。

```text
PSEUDOCODE — NOT COMPILABLE

finish_context(context):
    install(context, provider_base_ops)
    install(context, selected_hardware_variant_ops)

    state = initialize_custom_state_for_this_context()
    state.native = record_effective_callbacks_needed_for_delegation()

    if state.configuration is CUSTOM:
        require(closed_supported_object_families)
        install(context, tutorial_overlay)
```

最后安装 custom overlay 并不总是正确，但这样容易推理：自定义 callback 获胜，同时它
保存的 underlay 反映所选硬件变体。如果 provider 此后还会执行更多设置，就要审查后续
代码中是否还存在其他 `verbs_set_ops()` 调用。

## 2.4 让配置保持 context-local

在 context 尚未公开时读取一次配置，完成校验，并把 enum 或不可变 policy object 保存
到 provider 所有的 context 状态中。

应避免：

- 所有设备共享一个进程全局的“enabled”整数；
- 每次 send 或 poll 都重新读取环境变量；
- 对象已经存在之后才部分启用 callback；
- 把任何非空字符串都解释为 true；
- callback 仍在执行时允许 context 切换模式。

一个实用配置至少应有三种明确结果：

```text
NATIVE      正常 provider 行为
CUSTOM      custom overlay 及其已记录的对象限制
INVALID     context 创建失败并给出诊断信息
```

如果内部 worker、管理 daemon 或诊断进程必须使用原生行为，应在创建其对象之前选择
`NATIVE`。不要在单次调用前后切换全局 flag。

## 2.5 委托时避免递归

wrapper 不能调用自己所拦截的同一个公共 verb。该公共调用会再次查询 context table，
从而重新进入 wrapper。

应采用以下两种由 provider 掌控的策略之一：

1. 如果对每个受支持变体都正确，直接调用已知的 provider 底层实现；或
2. 安装 custom overlay 之前，把有效原生 callback 保存到 context-local 状态中。

不存在一个 provider API 可以返回所有有效私有 callback 的完整副本。不要访问
libibverbs 的不透明私有状态。对于公共 classic 快路径 slot，provider 可以在完成正常
overlay 后识别 context 中已经安装的 callback。对于其他操作，应维护显式的、由
provider 管理的 underlay selector，或者调用正确的具名 provider 实现。选择的入口必须
包含 provider 的全部清理和记账逻辑，而不能只是最底层的内核命令。

```text
PSEUDOCODE — NOT COMPILABLE

tutorial_submit_send(public_handle, request, bad_request):
    state = state_owned_by(public_handle.context)
    validate_owner_and_request(state, public_handle, request)

    if public_handle is proven native and request should use native behavior:
        return state.native.submit_send(public_handle, request, bad_request)

    return state.custom_backend.submit(public_handle, request, bad_request)
```

只有当 public handle 是真正的原生 provider 对象，或是该 callback 明确接受、且确由
原生对象支撑的 provider 对象时，才可以把它直接传给原生 callback。strict custom
façade 不能采用这种委托方式；它必须完全由自定义代码处理，或者转换到另一个具有独立
生命周期的真正原生 underlay 对象。

不要使用 `dlsym(RTLD_NEXT, ...)` 查找 provider underlay。这里不是一组公共 ELF wrapper
组成的调用栈，而是带有 context-specific 变体的 provider 内部调用图。

## 2.6 确定自定义 callback 如何到达其 backend

callback 可以执行本地策略，也可以把操作重定向到独立运行时（runtime）。后端
（backend）契约应与
provider 私有对象布局相互独立。

```text
PSEUDOCODE — NOT COMPILABLE

custom_backend.submit(command):
    validate(command.kind, lengths, identifiers, and ownership)
    reserve_bounded_capacity_or_fail()
    publish(command)
    return according_to_the_public_verb_contract
```

本教程刻意不指定 socket、共享内存布局、队列、wire header 或调度算法。这些都属于
独立的系统设计。无论选择哪种传输方式，都必须定义：

- framing 与版本协商；
- 并发下的请求/响应关联；
- buffer 所有权和生命周期；
- 有界队列与背压；
- 崩溃检测与清理；
- 完成顺序与陈旧 generation 拒绝；
- 针对相互信任或不信任进程的威胁模型（threat model）。

## 2.7 把不受支持的行为也视为 API 的一部分

支持范围越小且描述越精确，overlay 通常越安全。在创建 handle 之前拒绝不受支持的
构造器或属性。对于具体函数，要保留 provider/libibverbs 的错误约定：有些函数返回
errno 值，有些返回 `-1` 并设置 `errno`，还有些返回 `NULL` 并设置 `errno`。

需要记住，稀疏 overlay 中的 null 字段表示“保留先前安装的 callback”，而不是“禁用
这项操作”。如果继承的构造器会创建违反 custom-context 闭包的对象，就应安装一个签名
和错误约定都正确的显式拒绝 callback。

同样的审查也适用于继承的消费者、析构器、CQ 通知与事件路径，以及对象本地的
extended method。不能仅仅因为稀疏 overlay 中的对应字段为 null，就让原生 callback
接收 custom façade。

不要静默执行以下行为：

- 丢弃 attribute bit；
- 截断 opaque ID；
- 把 extended 对象转换为 classic 对象，除非转换已经定义且完全无损；
- 数据被截断后仍为 completion 报告成功；
- 把未知对象路由到原生 callback。

## 2.8 完整示例：观察并委托

最小的实用实验是在一个 classic operation 上添加 per-context 观察，同时让所有对象
保持原生表示。它刻意比 QP/CQ 虚拟化更简单，因此适合作为 dispatch 的第一个概念验证。

### 契约

- 仅对以 tutorial mode 显式打开的 context 拦截 classic `post_send`；
- 使用 context-local 原子变量统计调用次数和 WR 数量；
- 绝不检查或记录 payload、地址或 memory key；
- 把未修改的请求委托给有效原生 callback；
- 完整保留其返回值和 `bad_wr` 结果；
- 把 extended WR builder API 排除在声明的支持范围之外。

### Context 设置

```text
PSEUDOCODE — NOT COMPILABLE

after_provider_and_hardware_overlays(context):
    state = context.custom_state
    state.native_post_send = effective_classic_post_send(context)
    initialize_atomic_counters(state)

    if validated_context_mode is TUTORIAL_OBSERVE:
        install(context, overlay_containing_only_tutorial_post_send)
```

### Wrapper

```text
PSEUDOCODE — NOT COMPILABLE

tutorial_post_send(qp, first_wr, bad_wr):
    state = state_owned_by(qp.context)
    if state is absent or state.mode is not TUTORIAL_OBSERVE:
        fail_closed_because_dispatch_and_state_disagree()

    atomically_increment_call_count(state)
    atomically_add_bounded_wr_list_length(state, first_wr)

    return state.native_post_send(qp, first_wr, bad_wr)
```

wrapper 调用保存的原生 callback，而不是 `ibv_post_send()`。它不会创建新的 QP 类型，
因此 QP 的原生表示和生命周期保持不变。其支持声明很窄：只观察 classic post-send。

context-local counter 同样需要清理。可以扩展 provider 的正常 context cleanup，让它在
释放 provider context 之前释放自定义状态；也可以用 wrapper 覆盖 `free_context`，由
wrapper 先关闭自定义资源，再调用保存的、完整的原生 free callback。不要分配缺少对称
close 路径的 context-local 状态。

### 证据

运行四项测试：

1. 未修改构建：classic ping-pong 成功；
2. 修改后的构建，mode off：ping-pong 成功且 counter 保持为零；
3. 修改后的构建，observe mode：ping-pong 成功，确定性的调用/WR 计数与测试一致；
4. extended send（`ibv_qp_ex`/ping-pong 的 `-N` 路径）：要么明确保持原生并排除在支持
   声明之外，要么由单独设计的 guard 拒绝；不得把它误报为已经由 classic callback
   观察。

只有这个 profile 正确之后，实验才应继续替换构造器或返回自定义对象。此时，第 3 章的
对象族闭包规则就成为强制要求。

## 2.9 初始化与失败检查清单

- [ ] 自定义状态按设计分配给每个 context 或设备。
- [ ] 基础 overlay 和硬件 overlay 按有文档说明的顺序安装。
- [ ] 替换自定义 callback 之前，已经记录有效原生 callback。
- [ ] context 发布之后，custom overlay 保持不可变。
- [ ] 非法配置能够干净地中止 context 初始化。
- [ ] 部分初始化失败时，有一条按相反顺序清理资源的路径。
- [ ] context close 会停止新的自定义工作、关闭 backend/注册表资源、销毁自定义同步
  状态，然后完成原生清理。
- [ ] 内部/原生 context 不会意外启用自定义 callback。
- [ ] callback 诊断能够标识 mode 和 context，但不会记录 payload、地址、memory key 或
  凭据。

下一章：[对象族闭包与生命周期](03-close-object-lifecycles.md)。
