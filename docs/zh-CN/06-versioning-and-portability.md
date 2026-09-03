<!-- SPDX-License-Identifier: Apache-2.0 -->

# 6. 处理版本与 vendor fork

[English](../06-versioning-and-portability.md)

Provider interposition 位于私有边界上。版本管理是设计的一部分，而不是代码正常运行后
再补上的发布文书。

## 6.1 公开 API 稳定不等于 provider 私有 ABI 稳定

应用程序包含公开 verbs header，并链接到 libibverbs 导出的接口。Provider 还依赖内部结构与 symbol，包括 `verbs_context_ops`。rdma-core 会单独对这个 provider ABI 进行版本管理；v64.0 设置一个 `IBVERBS_PABI_VERSION`，并以匹配的 `-rdmav<version>.so` suffix 构建 provider 文件名。

这种机制可以防止部分意外混用，但不会让 provider 内部接口获得可移植性。一个源码兼容的字段仍可能改变语义、overlay 顺序、对象所有权或硬件前置条件，而自定义 fork 不会因此自动得到保护。

## 6.2 记录可复现的版本 manifest

为每次构建保留一份私有 manifest：

```yaml
# 模板——请用实验室的实际数据替换所有值
source:
  distribution: upstream-rdma-core-or-vendor-ofed
  release: exact-release-string
  commit: exact-source-commit-if-available
  source_archive_sha256: exact-sha256-if-applicable
provider:
  name: target-provider
  private_abi: exact-value
build:
  compiler: exact-version
  build_type: Debug-or-Release
  options: exact-cmake-options
runtime:
  distribution: exact-os-version
  kernel: exact-kernel-version
  firmware: exact-firmware-version
  device: exact-device-model
artifacts:
  libibverbs_sha256: exact-sha256
  provider_sha256: exact-sha256
```

不要把已填充的私有 manifest 复制到公开教程中。它可能泄露 hostname、内部路径、设备标识符或非公开 commit。

## 6.3 对每个 fork 重新执行源码定位

MLNX_OFED 可能包含由 vendor 维护的 rdma-core 衍生版本，其 package version 和文件布局可能不同于公开 upstream。针对确切的源码树，重新执行第 1 章中的搜索，并回答：

- `verbs_context_ops` 的结构是否相同？
- `verbs_set_ops()` 是否仍使用稀疏、非空字段覆盖语义？
- provider 私有 ABI 版本和文件名 suffix 是什么？
- 哪些 base overlay 和 variant overlay 会运行，顺序如何？
- QP/CQ/SRQ/MR 是否增加了新的构造器或消费者？
- provider 是否把实现拆分到了不同文件？
- provider context/object 布局是否发生变化？
- build target 名称或 loader 配置是否变化？
- 是否存在使用不同 license header 的 vendor-only 文件？

NVIDIA 当前文档还把较新的开发工作放在 DOCA-OFED/DOCA-Host 下，同时保留独立的 MLNX_OFED LTS 材料。应记录实际的 distribution family 和 release string，不要把“OFED”视为一棵连续版本化的源码树。

绝不能仅凭行号移植自定义 overlay。

## 6.4 升级流程

对每一个新的 upstream 或 vendor release：

1. 创建全新的源码 checkout；
2. diff provider-facing operation-table 定义；
3. 枚举新增、删除和签名变化的回调；
4. diff provider context 初始化及每一次 overlay 调用；
5. diff 被拦截操作的公开 inline dispatch 路径；
6. 重新生成对象族闭包矩阵；
7. 复核 provider context 和对象布局假设；
8. 把自定义改动作为经过审查的源码变更重新应用，而不是盲目套用补丁；
9. 一起构建 libibverbs 和 provider；
10. 从 native-off 行为开始，运行完整验证矩阵。

被拦截对象族中的任何新构造器或消费者，在完成分类之前都属于不支持范围。应把编译
成功视为移植工作的起点，而不是完成标志。

## 6.5 硬件变体与 overlay 顺序

固定版本的 mlx5 示例会在 common ops table 之后应用一个 CQE-version-specific overlay。其他版本可能增加更多 variant overlay。如果自定义回调包装某个 variant operation，就要为该 context 保存或选择有效的原生实现。

至少测试每一种受支持变体的一个代表。如果实验环境无法覆盖某个变体，将其标记为 `UNTESTED` 或直接拒绝。不要根据相似的结构体布局推断支持情况。

## 6.6 Import、fork 与静态链接

不同路径可能通过正常 open、import 或 static provider registration 创建 context。逐一审计：

- 是否初始化相同的自定义状态？
- 模式选择是否完全一致？
- 是否在正确的 overlay 之后捕获原生回调？
- imported object 能否安全进入自定义对象族？
- 跨越 `fork()` 时会发生什么？
- static linking 是否在支持范围内；若是，provider 的注册方式是否不同？

如果一条路径尚未完成设计和测试，就应拒绝该路径，或明确记录为不支持。

## 6.7 分发策略

最安全的开发产物是针对确切源码包的源码分支或便于审查的小型补丁，并由操作者在隔离
环境中完成构建。单独的 provider 二进制难以审计，也很容易与错误的 libibverbs、kernel
或硬件组合使用。

本教程既不分发 source patch，也不分发 binary。如果另一个项目需要分发这些内容，则必须：

- 保留所有适用的 upstream/vendor license 和 copyright notice；
- 显著标识修改；
- 按照所选许可证路径的要求，提供相应源码和构建信息；
- 声明确切的 libibverbs 私有 ABI 和支持的 stack matrix；
- 在没有明确 deployment 和 rollback 方案时，避免替换系统 package。

## 6.8 可移植性声明措辞

推荐：

> 已在硬件/firmware/kernel 矩阵 Z 上，针对源码版本 Y 中的 provider X 完成测试。其他版本需要重新进行源码审查和闭包审查。

避免：

> 适用于 OFED、所有 mlx5 设备或任何 libibverbs 应用程序。

可移植性是一张由证据支撑的矩阵，而不是从公开 verbs API 自动继承的属性。

返回：[简体中文 README](../../README.zh-CN.md) · [公开资料索引](../REFERENCES.md)。
