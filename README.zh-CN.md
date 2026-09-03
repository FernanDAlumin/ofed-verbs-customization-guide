<!-- SPDX-License-Identifier: Apache-2.0 -->

# 在 OFED 中定制 RDMA Verbs

## Userspace provider 侧设计教程

[English](README.md) · [中文教程](docs/zh-CN/00-scope-and-terminology.md) ·
[公开资料索引](docs/REFERENCES.md)

英文教程是规范版本；`docs/zh-CN/` 提供逐章完整中文版，本页是中文总入口。

本教程解释一个 `libibverbs` 调用如何到达 userspace provider，以及如何在
已有 OFED/rdma-core provider 中，为单个 context 安装一层自定义操作回调。
教程以公开的 `mlx5` provider 为贯穿示例，但不会复制完整上游源码，也不包含
任何研究或生产项目的实际实现。

核心方法可以概括为：

> 先完成 provider 的正常 context 初始化，再为边界明确且生命周期闭合的对象族
> 安装一张稀疏的自定义 operation-table overlay。

真正困难的部分不是替换函数指针，而是保证对象归属、操作闭包、错误语义、并发、
原生回退、ABI 版本和回滚路径始终一致。

教程区分三种设计：只包装并委托原生对象的 callback、在 strict context 中使用
自定义对象表示的完整对象族，以及允许原生/自定义对象共存的注册表混合模式。三者的
安全条件不同，不能混为一谈。

## 项目边界

这是独立撰写的工程教程，不是 provider fork、可安装补丁或论文复现 artifact。
仓库不包含任何私有实现的源码、改写补丁、ABI 布局、日志、trace、性能结果或
二进制。所有 C 风格片段都是为本教程独立编写的、不可编译伪代码。

公开事实使用固定版本链接。主要参考基线是
[`linux-rdma/rdma-core` v64.0](https://github.com/linux-rdma/rdma-core/tree/f272237493ef309984036f7b85655e11104c61c8)。
MLNX_OFED 等 vendor fork 可能移动或修改相应代码，因此实际操作时必须对目标
源码包重新完成版本记录和定位，不能照搬行号。

## 为什么不是 `LD_PRELOAD`

常见数据面接口并不都经过可由动态链接器替换的导出符号。例如，上游
`ibv_post_send()` 和 `ibv_post_recv()` 是通过 `qp->context->ops` 直接分派的
header-inline 路径。因此，本教程讨论的是在 userspace provider 内改变单个
context 的回调表，而不是依赖 `LD_PRELOAD` 抢占公共符号。

provider 本身仍由 libibverbs 动态加载；“provider 被加载”与“应用调用如何在
provider 内分派”是两个不同层次。

## 你将学到

- 区分公开 verbs API、libibverbs core、userspace provider、kernel uAPI 和 NIC；
- 在固定源码版本中找到 provider 注册、context 初始化、基础 ops 和硬件变体；
- 用稀疏 overlay 替换选定回调，而不是运行时修改共享表；
- 保存或识别原生 underlay，并避免 wrapper 再调用同一个公共 verb 导致递归；
- 让 QP、CQ、SRQ、MR 的创建、使用、错误和销毁路径形成对象族闭包；
- 在 native-object wrapper、strict custom family 和 registry mixed 三种模式中
  明确选择；
- 对不支持的 extended verbs 明确返回错误；
- 在不覆盖系统 provider 的前提下完成构建、加载验证和回滚；
- 建立兼容矩阵并测试关闭模式、并发、失败和 ABI 不匹配。

## 本教程不做什么

- 不设计新的 kernel verb、ioctl、固件命令或 wire protocol；
- 不提供生产 provider，也不宣称完整或透明的 verbs 兼容；
- 不把 `mlx5` provider 内部接口描述成稳定公开 ABI；
- 不指导覆盖 `/usr` 下的系统 `libibverbs` 或 `libmlx5`；
- 不公开自定义 runtime、传输协议、调度算法或性能结果。

如果需要新增 kernel uAPI，应同时准备 kernel 与 rdma-core 变更；如果只是新增
应用可见的 provider-specific 能力，应优先评估文档化的 DV 风格 API。本教程的
ops interposition 适用于你明确维护并验证自定义 provider 构建的场景。

## 学习路径

1. [范围与术语](docs/zh-CN/00-scope-and-terminology.md)
2. [追踪 verbs dispatch 路径](docs/zh-CN/01-trace-the-dispatch-path.md)
3. [设计 context 级 ops overlay](docs/zh-CN/02-design-an-ops-overlay.md)
4. [对象族闭包与生命周期](docs/zh-CN/03-close-object-lifecycles.md)
5. [安全构建、加载与回滚](docs/zh-CN/04-build-load-and-rollback.md)
6. [验证行为和兼容性](docs/zh-CN/05-validation-matrix.md)
7. [处理版本和 vendor fork](docs/zh-CN/06-versioning-and-portability.md)

## 最重要的不变量

一旦 context 安装了自定义 callback，任何能够到达这些 callback 的公开对象，都
必须具有已知的类型和 owner。

绝不能先对来源未知的 public handle 做 `container_of`，再读取 wrapper 尾部的
tag 判断它是不是自定义对象。如果它由原生路径创建，这次读取本身就已经越界。
正确做法是：创建阶段强制对象族闭包，或者在 downcast 之前查询线程安全注册表。

## 安全实验原则

只在一次性开发节点、VM、具备合适设备权限的容器或专用实验机中测试。使用
in-place 构建和 side-by-side provider，不覆盖系统文件。启用自定义 dispatch
之前，应记录目标进程实际加载的 libibverbs/provider 路径，并准备独立的停止和
回滚通道。

## 许可证

仓库原创内容使用 [Apache License 2.0](LICENSE)。公开上游只作链接引用，不属于
本项目分发内容。详情见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)、
[PROVENANCE.md](PROVENANCE.md) 和 [TRADEMARKS.md](TRADEMARKS.md)。
