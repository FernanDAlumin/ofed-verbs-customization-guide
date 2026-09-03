<!-- SPDX-License-Identifier: Apache-2.0 -->

# 6. Handle versions and vendor forks

Provider interposition lives on a private boundary. Version management is part
of the design, not release paperwork added after the code works.

## 6.1 Public API stability is not provider-private ABI stability

Applications include public verbs headers and link to exported libibverbs
interfaces. Providers additionally depend on internal structures and symbols,
including `verbs_context_ops`. rdma-core versions this provider ABI separately;
v64.0 sets an `IBVERBS_PABI_VERSION` and builds provider filenames with a
matching `-rdmav<version>.so` suffix.

This protects against some accidental mixing, but it does not make provider
internals portable. A source-compatible field may change semantics, overlay
order, object ownership, or hardware preconditions without helping a custom
fork.

## 6.2 Record a reproducible version manifest

Keep a private manifest for every build:

```yaml
# TEMPLATE — replace every value with data from the lab
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

Do not copy a populated private manifest into a public tutorial. It may reveal
hostnames, internal paths, device identifiers, or non-public commits.

## 6.3 Re-run source navigation for every fork

MLNX_OFED can contain a vendor-maintained rdma-core derivative whose package
version and file layout differ from public upstream. For the exact source tree,
repeat the searches from Chapter 1 and answer:

- Is `verbs_context_ops` the same shape?
- Does `verbs_set_ops()` still use sparse non-null overlay semantics?
- What is the provider-private ABI version and filename suffix?
- Which base and variant overlays run, and in what order?
- Did QP/CQ/SRQ/MR gain new constructors or consumers?
- Did the provider split implementations across different files?
- Are provider context/object layouts different?
- Did build target names or loader configuration change?
- Are there vendor-only files with different license headers?

NVIDIA's current documentation also places newer development under DOCA-OFED /
DOCA-Host while retaining standalone MLNX_OFED LTS material. Record the actual
distribution family and release string; do not treat “OFED” as one continuously
versioned source tree.

Never port a custom overlay by line number alone.

## 6.4 Upgrade procedure

For each new upstream or vendor release:

1. create a fresh source checkout;
2. diff the provider-facing operation-table definition;
3. enumerate added, removed, and signature-changed callbacks;
4. diff provider context initialization and every overlay call;
5. diff public inline dispatch paths for intercepted operations;
6. regenerate the object-family closure matrix;
7. review provider context and object layout assumptions;
8. reapply the customization as a reviewed source change, not a blind patch;
9. build libibverbs and provider together;
10. run the full validation matrix, beginning with native-off behavior.

Any new creator or consumer in an intercepted family is unsupported until
classified. Treat compiler success as the start of the port, not completion.

## 6.5 Hardware variants and overlay ordering

The pinned mlx5 example applies a CQE-version-specific overlay after its common
ops table. Other versions may add more variant overlays. If a custom callback
wraps a variant operation, save or select the effective native implementation
for that context.

Test at least one representative of every supported variant. If the lab cannot
exercise a variant, mark it `UNTESTED` or reject it. Do not infer support from
similar structure layouts.

## 6.6 Import, fork, and static linking

Separate paths may create a context through normal open, import, or static
provider registration. Audit each path:

- Does it initialize the same custom state?
- Is the mode selection identical?
- Are native callbacks captured after the correct overlays?
- Can imported objects enter a custom family safely?
- What happens across `fork()`?
- Is static linking in scope, and if so, are providers registered differently?

If a path is not designed and tested, reject or document it as unsupported.

## 6.7 Distribution policy

The safest development artifact is a source branch or a small, reviewable patch
against an exact source package, built by the operator in an isolated
environment. A provider binary alone is difficult to audit and easy to combine
with the wrong libibverbs, kernel, or hardware.

This tutorial distributes neither source patches nor binaries. If a separate
project does so, it must:

- preserve every applicable upstream/vendor license and copyright notice;
- identify modifications prominently;
- ship the corresponding source and build information required by its chosen
  license path;
- declare the exact libibverbs private ABI and supported stack matrix;
- avoid replacing system packages without an explicit deployment and rollback
  plan.

## 6.8 Portability claim language

Prefer:

> Tested against provider X from source release Y on hardware/firmware/kernel
> matrix Z. Other versions require a new source and closure review.

Avoid:

> Works with OFED, all mlx5 devices, or any libibverbs application.

Portability is an evidence-backed matrix, not a property inherited from the
public verbs API.

Next: [Respect licensing and provenance](07-licensing-and-provenance.md).
