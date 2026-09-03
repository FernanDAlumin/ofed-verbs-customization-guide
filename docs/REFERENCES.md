<!-- SPDX-License-Identifier: Apache-2.0 -->

# Public references

Last verified: 2026-09-03.

This page records the public sources used to check the tutorial. Links to
rdma-core point to a fixed commit so that statements remain reviewable after
upstream changes. No referenced source is distributed by this repository.

## Reference snapshot

- Project: [`linux-rdma/rdma-core`](https://github.com/linux-rdma/rdma-core)
- Release: [`v64.0`](https://github.com/linux-rdma/rdma-core/releases/tag/v64.0)
- Peeled commit:
  [`f272237493ef309984036f7b85655e11104c61c8`](https://github.com/linux-rdma/rdma-core/tree/f272237493ef309984036f7b85655e11104c61c8)
- At that revision: `PACKAGE_VERSION=64.0`, `IBVERBS_PABI_VERSION=64`, and
  the mlx5 provider private-ABI loading name uses `libmlx5-rdmav64.so`.
- [Top-level package and provider-private ABI constants](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/CMakeLists.txt#L84-L92)

## libibverbs public and provider-private interfaces

- [Public `ibv_context_ops` and `ibv_context`](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/libibverbs/verbs.h#L2031-L2082)
- [Inline classic CQ polling dispatch](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/libibverbs/verbs.h#L2990-L3005)
- [Inline classic send and receive dispatch](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/libibverbs/verbs.h#L3498-L3517)
- [Provider-private `verbs_context_ops`, registration, and context helpers](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/libibverbs/driver.h#L355-L565)
- [`verbs_set_ops()` sparse overlay implementation](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/libibverbs/dummy_ops.c#L695-L828)
- [Dummy-op initialization during context setup](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/libibverbs/device.c#L230-L290)
- [Provider-private header build treatment](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/libibverbs/CMakeLists.txt#L1-L18)
- [Private symbol and provider ABI versioning](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/Documentation/versioning.md#private-symbols-in-libibverbs)

## Provider discovery and mlx5 running example

- [Device discovery and provider matching](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/libibverbs/init.c#L240-L465)
- [Dynamic provider configuration and loading](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/libibverbs/dynamic_driver.c#L116-L240)
- [`verbs_open_device()` and provider `alloc_context`](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/libibverbs/device.c#L323-L367)
- [mlx5 common operation table](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/providers/mlx5/mlx5.c#L98-L187)
- [mlx5 common and CQE-version overlay order](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/providers/mlx5/mlx5.c#L2568-L2586)
- [mlx5 context allocation, device ops, and provider registration](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/providers/mlx5/mlx5.c#L2618-L2792)
- [mlx5 provider build target](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/providers/mlx5/CMakeLists.txt)
- [Public mlx5 direct-verbs manual](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/providers/mlx5/man/mlx5dv.7)

## Extended operation surfaces

- [Extended QP WR-builder callbacks](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/libibverbs/verbs.h#L1337-L1516)
- [Extended CQ polling callbacks](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/libibverbs/verbs.h#L1567-L1638)

These links support the warning that classic `post_send` and `poll_cq`
interposition does not automatically cover object-local extended interfaces.

## Build and validation

- [Official rdma-core build instructions](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/README.md#building)
- [In-place build script](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/build.sh)
- [Shared-provider target, `.driver` file, and symlink generation](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/buildlib/rdma_functions.cmake#L136-L227)
- [rdma-core testing guide](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/Documentation/testing.md)
- [`ibv_rc_pingpong` options](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/libibverbs/examples/rc_pingpong.c#L809-L850)
- [rdma-core contribution and kernel-change coordination](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/Documentation/contributing.md)

## Kernel boundary

- [Linux kernel userspace verbs access](https://docs.kernel.org/infiniband/user_verbs.html)

The kernel documentation explains the uverbs device boundary and why control
operations and mapped userspace datapaths have different call behavior.

## NVIDIA OFED documentation

- [NVIDIA MLNX_OFED 24.07 programming guide](https://networking-docs.nvidia.com/mlnxofedswum/24070610/programming)
- [NVIDIA MLNX_OFED 24.07 introduction and package layout](https://networking-docs.nvidia.com/mlnxofedswum/24070610/introduction)
- [NVIDIA MLNX_OFED 24.07 installation documentation](https://networking-docs.nvidia.com/mlnxofedswum/24070610/installation)
- [NVIDIA MLNX_OFED 24.07 rdma-core migration note](https://networking-docs.nvidia.com/mlnxofedswum/24070610/changes-and-new-features)
- [NVIDIA networking legal information](https://www.nvidia.com/en-us/about-nvidia/legal-info.html)
- [OpenFabrics Alliance OFED page](https://www.openfabrics.org/ofed-for-linux/)

Vendor documentation is version-specific. A vendor source package may differ
from public upstream and must be reviewed under its own file-level licenses.

## Licenses and contribution provenance

- [rdma-core v64.0 license rules](https://github.com/linux-rdma/rdma-core/blob/f272237493ef309984036f7b85655e11104c61c8/COPYING.md)
- [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- [Developer Certificate of Origin 1.1](https://developercertificate.org/)

The rdma-core root defines a default dual-license rule and explicitly allows
file- or directory-level overrides. Always inspect the exact material being
redistributed.
