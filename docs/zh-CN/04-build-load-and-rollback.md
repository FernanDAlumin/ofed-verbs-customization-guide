<!-- SPDX-License-Identifier: Apache-2.0 -->

# 4. 安全构建、加载与回滚

[English](../04-build-load-and-rollback.md)

如果错误地替换了系统库，provider 实验可能导致一台主机上的所有 RDMA 应用失败。
因此，本教程使用 in-place、side-by-side 构建，绝不覆盖安装在 `/usr` 下的文件。

## 4.1 选择合适的实验环境

使用以下环境之一：

- 专用 RDMA 开发节点；
- 配有合适虚拟设备或直通设备的 VM；
- 仅作为构建环境的容器，并在硬件测试时有意授予设备访问权限；
- 用 upstream software-RDMA 环境进行通用 libibverbs 测试，同时接受 mlx5-specific
  路径仍然需要 mlx5 硬件这一事实。

不要从管理、存储或集群控制路径依赖正在修改的 RDMA stack 的主机开始实验。
硬件测试之前，应保留第二条登录路径或 console 通道。

## 4.2 选择源码之前记录已安装的 stack

在目标 Linux 主机上，以只读方式收集事实：

```sh
ofed_info -s                  # when MLNX_OFED supplies this command
ibv_devices
ibv_devinfo
ldconfig -p | grep -E 'libibverbs|libmlx5'
uname -a
```

还要记录发行版 package version、NIC 型号、firmware、kernel driver，以及用于验证的
确切应用二进制。Debian 衍生发行版与 RPM 衍生发行版的 package 命令不同；应在实验记录
中保存所用命令和未经改写的版本字符串。

如果主机使用 MLNX_OFED，应从 vendor 获取与之匹配的公开 source package，而不能假定
某个 upstream tag 具有相同布局。NVIDIA 的公开 package 文档标出了源代码内容和目录布局。
不要以本教程的 Apache license 发布 vendor source archive。

## 4.3 使用固定的公开 baseline 学习

本指南中的示例已基于公开 rdma-core v64.0 检查：

```sh
git clone https://github.com/linux-rdma/rdma-core.git
cd rdma-core
git checkout --detach f272237493ef309984036f7b85655e11104c61c8
test "$(git rev-parse HEAD)" = f272237493ef309984036f7b85655e11104c61c8
git switch -c tutorial/provider-overlay
```

进行硬件实验时，应在同一 commit 上另建一份 pristine worktree/build。这样，native
control 和 rollback test 就拥有独立 artifact，而不是只依赖修改后 provider 内部的
某个 runtime mode。

预期的 peeled commit 记录在 [REFERENCES.md](../REFERENCES.md) 中。如果结果不匹配，
应停止操作并更新版本记录，再依赖相应源码位置。

按照固定 release 自己的 README 安装构建依赖。依赖包名称会变化；在本教程中重复列出
package 清单会很快过时。

## 4.4 添加文件之前识别集成点

在选定的 provider 中定位：

1. provider 的 CMake target 和 source list；
2. provider registration descriptor；
3. context allocation/import 函数；
4. 每一处 base 和 variant `verbs_set_ops()` 调用；
5. provider context structure 及其 cleanup path；
6. 用作 native underlay 的实现函数；
7. 覆盖目标 operation family 的测试或示例。

对于 in-tree 实验，应让变更保持局部化。典型源码布局会增加一个 provider-private
实现文件和一个 private declaration 文件，再把实现文件加入 provider 的 CMake target。
本指南有意不提供可直接应用的 diff。

编码之前，创建一份包含以下内容的设计记录：

- source tag 和 commit；
- 目标 provider 和 private ABI version；
- 要替换的 callback field；
- base/variant/custom overlay 的顺序；
- 每个对象族的表示/callback profile：native-object wrapper、strict custom family，
  或 registry mixed family；
- 不支持的构造器和 attribute；
- rollback procedure。

## 4.5 In-place 构建

upstream convenience script 会配置 in-place build：

```sh
./build.sh
```

如需显式的 debug build：

```sh
cmake -S . -B build -GNinja \
  -DIN_PLACE=1 \
  -DCMAKE_BUILD_TYPE=Debug \
  -DMLX5_DEBUG=ON \
  -DNO_MAN_PAGES=1 \
  -DNO_PYVERBS=1 \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=1

cmake --build build --target ibverbs mlx5 ibv_devices ibv_devinfo ibv_rc_pingpong
```

vendor fork 中的 target 名称可能不同。应检查其 CMake 定义，而不是不断重命名 target，
直到命令碰巧能够运行。

rdma-core 的 in-place build 会在 build tree 下生成 provider 配置，以及带有 private ABI
version 后缀的 provider library。应始终将该 build 的 libibverbs、header、provider、
配置和工具配套使用。

## 4.6 启用 customization 之前验证 baseline

在没有启用 custom mode 的情况下运行 in-place 工具：

```sh
build/bin/ibv_devices
build/bin/ibv_devinfo
```

需要时启用 libibverbs 诊断：

```sh
env VERBS_LOG_LEVEL=4 build/bin/ibv_devinfo
```

在合适的双端点或 loopback 实验环境中，修改 callback 之前先运行 upstream RC ping-pong
示例。保存命令、exit status、device/port 和简短的结果摘要。这将成为 native control。

```sh
# Server
build/bin/ibv_rc_pingpong -d <device> -i <port> -n 1000 -c

# Client
build/bin/ibv_rc_pingpong -d <device> -i <port> -n 1000 -c <server-host>
```

classic 路径通过后，使用与所声明 profile 相关的 upstream option 重复测试：`-e` 覆盖
CQ event，`-N` 使用新的/extended send WR API，`-t` 请求带 timestamp 的 extended CQ
路径。当 custom overlay 有意只支持 classic post/poll 时，这些是很有价值的负面测试。

## 4.7 证明实际加载了哪个 provider

“程序运行成功”并不能证明它使用了 custom build。至少使用一种 loader-level 方法验证
shared-object 选择：

```sh
env LD_DEBUG=libs build/bin/ibv_devinfo 2>/tmp/verbs-loader.log
strace -f -e trace=openat build/bin/ibv_devinfo 2>/tmp/verbs-openat.log
```

只检查 library path 和 loader error；不要发布包含机器特定路径或 device identifier 的日志。

当程序不以 setuid 方式运行时，libibverbs 还识别 `RDMAV_DRIVERS` 和旧的
`IBV_DRIVERS`。在固定版本中，绝对路径值被视为 provider path prefix，并在其后附加
private-ABI suffix。这可用于受控实验调用：

```sh
env RDMAV_DRIVERS="$PWD/build/lib/libmlx5" \
    VERBS_LOG_LEVEL=4 \
    "$PWD/build/bin/ibv_devinfo"
```

只能把这种方式与匹配的 in-place libibverbs 配合使用。它不是把针对某个 private ABI
构建的 provider 加载进任意 system library 的受支持方法。通常，使用生成的 in-place
配置更简单。

## 4.8 只有在 control 通过后才启用 custom mode

provider 应当在构造 context 时解析其 opt-in 配置，并输出一条简洁的诊断，其中包括：

- source/build identity；
- provider-private ABI version；
- 不包含敏感信息的 device/context identity；
- 所选 mode（`NATIVE`、`CUSTOM` 或 failure）；
- 支持的 operation profile。

不要记录 payload、virtual address、lkey/rkey、GID、credential 或 application secret。
不要在 datapath 上反复读取配置。

每次只测试一个 operation family。先完成 create/query/destroy 闭包，再加入 post/poll
traffic。在负面测试通过之前，customization 应保持默认关闭。

## 4.9 回滚

因为没有在系统范围内安装任何内容，回滚是一个操作流程：

1. 停止 in-place test process；
2. 在测试 shell 中 unset custom provider 和 mode variable；
3. 从干净 shell 运行同一 commit 的 pristine control build，或者发行版提供的
   `ibv_devinfo`；
4. 再次验证实际加载的 libibverbs 和 provider path；
5. 私下保留失败 build 和诊断信息，以便分析。

绝不能把 `sudo make install`、替换 package 或把 shared object 复制到 `/usr` 作为正常
教程步骤。打包和 fleet deployment 需要单独的运维与许可证审查。

下一章：[验证行为和兼容性](05-validation-matrix.md)。
