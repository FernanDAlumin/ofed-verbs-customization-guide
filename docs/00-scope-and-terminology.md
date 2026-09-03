<!-- SPDX-License-Identifier: Apache-2.0 -->

# 0. Scope and terminology

This chapter defines exactly what is being customized. That distinction keeps a
small userspace experiment from being described as a new verb, a kernel driver,
or a portable libibverbs extension when it is none of those things.

## 0.1 The five layers

An RDMA application usually interacts with several layers that have different
stability and review rules.

| Layer | Typical contents | Stability relevant to this guide |
|---|---|---|
| Application API | `ibv_*`, `rdma_*`, and provider-specific public APIs | Public source/API; compatibility is expected |
| libibverbs core | device discovery, provider loading, common object handling, command marshalling | Internal implementation plus exported ABI |
| Userspace provider | device matching, context setup, object implementations, datapath callbacks | Provider-private source; tightly versioned |
| Kernel RDMA uAPI | uverbs ioctls/write commands and kernel driver behavior | Coordinated userspace/kernel contract |
| NIC and firmware | queues, doorbells, completion formats, device commands | Hardware- and firmware-specific |

This tutorial changes the **userspace provider**. It does not add a kernel uAPI
or firmware command. Calling the result “no driver changes” is ambiguous: the
kernel driver may be unchanged, but the userspace provider has definitely been
customized.

## 0.2 Three different projects often called “custom verbs”

Before editing source, classify the intended result.

### A. Override or augment an existing provider operation

Examples include validating an existing QP request, collecting provider-local
telemetry, routing a supported operation to a custom userspace runtime, or
choosing a different implementation for a context.

This is the subject of the guide. The existing public verb still enters an
existing provider operation slot. No new application symbol or kernel command
is implied.

### B. Add a provider-specific userspace API

If applications must explicitly request new behavior, a documented
provider-specific API may be clearer than transparent callback replacement. In
the mlx5 ecosystem this is the design space occupied by `mlx5dv_*` APIs. A real
public API also requires headers, symbol/version-map work, documentation,
compatibility commitments, and upstream review.

This guide mentions that alternative but does not design such an API.

### C. Add a new kernel or hardware verb

If the feature needs a new uverbs command, kernel object, device command, or
firmware behavior, provider-only work is insufficient. Kernel and rdma-core
changes must be designed and reviewed together. Upstream rdma-core explicitly
documents this coordination requirement.

This path is outside scope.

## 0.3 Terminology used by the tutorial

**Base ops**
: The provider's normal operation callbacks for a context before the tutorial's
  customization is applied.

**Variant overlay**
: A sparse operation table applied by the provider for a hardware or ABI
  variant. Upstream mlx5 uses this pattern for a CQE-format-specific polling
  callback.

**Custom overlay**
: A sparse, immutable table containing only the callbacks intentionally
  replaced by the customization.

**Native callback / underlay**
: The effective provider callback that would have handled the operation without
  the custom overlay. “Effective” matters because it may already include a
  hardware-specific variant.

**Public handle**
: An application-visible object pointer such as `ibv_qp *` or `ibv_cq *`.

**Provider object**
: The provider-owned allocation that contains or backs a public handle.

**Object-family closure**
: The property that every creator, consumer, query, modifier, event path, and
  destructor reachable for a public object agrees on the object's concrete kind
  and semantics.

**Native-object wrapper mode**
: A custom callback observes, validates, or augments an operation on an ordinary
  native provider object, then delegates to the effective native callback. No
  custom object representation is introduced.

**Strict mode**
: A custom context admits only custom objects for every intercepted family.
  Unsupported constructors fail instead of creating native objects that could
  later reach custom callbacks.

**Mixed mode**
: Native and custom objects may coexist in one context. A registry classifies a
  handle before any downcast and routes the operation to a saved native or
  custom callback.

**Fail closed**
: A still-valid but unclassified/foreign object that legally reaches a callback,
  or an unsupported operation/attribute, produces an explicit error. Stale
  custom async records are rejected by a carried generation. This does not make
  a freed public C pointer safe.

## 0.4 What the design can and cannot claim

A callback overlay can honestly claim that selected operations are redirected
inside a custom provider build. It cannot, by itself, claim:

- a stable plugin interface for stacking on an arbitrary installed provider;
- full verbs compatibility;
- application transparency across inline, extended, and provider-specific APIs;
- wire compatibility with a peer that lacks any required custom runtime;
- zero-copy behavior, isolation, ordering, or recovery that the wrappers do not
  actually implement and test;
- portability across rdma-core, MLNX_OFED, firmware, or hardware versions.

Treat each of those as a separate engineering claim with a capability matrix
and tests.

## 0.5 Prerequisites

The reader should be comfortable with:

- C function pointers and container-style object layouts;
- QP, CQ, PD, MR, WR, WC, and basic RC send/receive terminology;
- CMake/Ninja or Make on Linux;
- debugging dynamic-library selection;
- using a disposable machine or dedicated lab node for RDMA experiments.

No private implementation is needed. All source locations used in the guide are
from the public references listed in [REFERENCES.md](REFERENCES.md).

## 0.6 Choose the smallest viable mechanism

Use this decision sequence before choosing provider interposition:

1. Can the behavior live entirely in the application? Prefer an application
   library.
2. Does the application need an explicit provider feature? Consider a
   documented provider-specific API.
3. Is the requirement to alter existing object behavior for a controlled
   provider build? A context-scoped ops overlay may fit.
4. Is a new kernel/hardware primitive required? Stop and design coordinated
   kernel and rdma-core changes.

Provider interposition is powerful because it sits below public call sites. It
also inherits provider-private ABI, lifecycle, and hardware assumptions. Use it
because those trade-offs match the deployment, not merely because replacing a
function pointer looks convenient.

Next: [Trace the dispatch path](01-trace-the-dispatch-path.md).
