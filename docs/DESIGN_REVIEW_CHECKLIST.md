<!-- SPDX-License-Identifier: Apache-2.0 -->

# Provider customization design review

Use this as the final gate before implementing or enabling a custom provider
overlay. A checked box should point to evidence, not intuition.

## Scope

- [ ] The project states whether it overrides an existing operation, adds a
  provider-specific public API, or requires a new kernel uAPI.
- [ ] Userspace provider changes are not described as `LD_PRELOAD` interception.
- [ ] The supported provider, source release, private ABI, hardware, firmware,
  kernel, and operation profile are explicit.
- [ ] “Kernel driver unchanged” is distinguished from “userspace provider
  customized.”
- [ ] Full verbs, transparency, portability, zero-copy, performance, and
  isolation claims are absent unless separately proved.

## Source and dispatch

- [ ] The exact source tag/commit or vendor archive hash is recorded.
- [ ] Provider registration and every context open/import path are located.
- [ ] Base and hardware-variant `verbs_set_ops()` calls are enumerated.
- [ ] The custom overlay order is intentional.
- [ ] Effective native callbacks are saved or selected per context before
  replacement.
- [ ] `verbs_set_ops()` is used so private, public fast-path, and compatibility
  slots stay coherent.
- [ ] Callback tables are immutable after context publication.
- [ ] Classic context callbacks and extended object-local callbacks are reviewed
  separately.

## Configuration and reentrancy

- [ ] Configuration is parsed and validated once per context.
- [ ] Invalid configuration fails context initialization.
- [ ] Different contexts/devices cannot accidentally share mode or state.
- [ ] Internal/native workers receive a separate native context or equivalent
  construction-time bypass.
- [ ] No wrapper calls the same public verb and recurses into itself.
- [ ] Native delegation invokes the complete effective provider callback, not a
  lower-level helper that skips cleanup or bookkeeping.

## Object-family closure

- [ ] [OPERATION_CLOSURE_TEMPLATE.md](OPERATION_CLOSURE_TEMPLATE.md) is complete
  for the pinned `verbs_context_ops` definition.
- [ ] Every intercepted family covers classic, extended, `open_qp`,
  imported-context, and provider-specific paths or rejects them early.
- [ ] QP, CQ, SRQ, MR, PD, context, and cross-object relationships are
  classified.
- [ ] Strict mode prevents mixed native/custom objects in intercepted families;
  or mixed mode uses a registry before downcast.
- [ ] Every in-scope classic, extended, `open_qp`, direct-verbs, and
  imported-context creator registers a mixed-mode object before returning it;
  insertion failure invokes the matching destructor.
- [ ] No code performs `container_of` before proving that a still-valid public
  handle belongs to the custom allocation family.
- [ ] Valid but unclassified objects from reachable constructors fail closed
  before downcast; invalid casts and fabricated pointers are outside the claim.
- [ ] Raw-pointer use-after-free is not claimed to be repaired by a registry;
  generation is used only for custom records that actually carry it.
- [ ] Custom CQs do not reach native notify/resize/event/destroy functions unless
  they are valid native CQs.
- [ ] Unsupported SRQ and event-CQ associations fail during construction.

## Semantics

- [ ] Return and `errno` conventions match each public operation.
- [ ] Partial WR-list success and `bad_wr` are correct.
- [ ] Full-width opaque identifiers round-trip without hidden bit packing.
- [ ] Completion status, opcode, length, identity, signaling, and ordering are
  defined.
- [ ] Buffer-too-small, queue-full, backend-disconnect, and flush behavior are
  explicit.
- [ ] Any difference from native receive/backpressure semantics is documented.
- [ ] Memory ownership, access, registration, import/export, and lifetime are
  not inferred from raw pointers or keys.

## Lifetime and concurrency

- [ ] Partial create failures unwind in reverse order.
- [ ] Exactly one caller owns the `LIVE → DYING → DEAD` transition.
- [ ] Destroy either relies on a documented caller-synchronization precondition,
  or a supported concurrent-destroy design blocks new operations and owns
  in-flight references safely.
- [ ] Generation tracking prevents stale custom commands/completions from being
  accepted by a recycled logical identity.
- [ ] Context close with live child objects has a defined result.
- [ ] Context close transitions to `CLOSING`, stops custom work, releases custom
  backend/registry/synchronization state, and completes native cleanup last.
- [ ] Shared control channels provide framing and request correlation.
- [ ] Queue capacity and backpressure are bounded.
- [ ] Fork behavior is supported and tested or explicitly rejected.
- [ ] Diagnostics omit payloads, pointers, memory keys, credentials, and private
  topology.

## Build, loading, and tests

- [ ] libibverbs, internal headers, provider, configuration, and tools come from
  the same build and private ABI.
- [ ] The build is in-place or side-by-side; no file under `/usr` is replaced.
- [ ] Loader diagnostics prove the expected provider was selected.
- [ ] Native-off behavior matches the unmodified pinned control.
- [ ] Every `CUSTOM`, `ADAPTED`, and `REJECTED` cell has a test.
- [ ] Extended QP/CQ, SRQ, event, imported-context, and unsupported paths have
  negative tests.
- [ ] Valid unclassified/cross-context objects and stale async records are
  exercised; sanitizers are used for accidental UAF where practical.
- [ ] Multi-context, multi-thread, failure, teardown, and rollback tests pass.
- [ ] Rollback uses a pristine control artifact and re-proves the loaded library
  paths.
- [ ] Hardware testing begins only after source, loader, closure, and negative
  tests pass.

## Distribution and rights

- [ ] Every distributed file has known provenance and license.
- [ ] Original upstream/vendor notices and modification markers are preserved.
- [ ] Employer, university, client, coauthor, funder, and patent authority are
  resolved before publication.
- [ ] Trademarks are used only for factual identification without endorsement.
- [ ] Release archives contain no private code, manifests, logs, paths,
  addresses, keys, credentials, or binaries outside the reviewed scope.

## Decision

- [ ] **Proceed** — every required item has evidence.
- [ ] **Proceed with reduced profile** — unsupported paths are rejected and the
  published matrix is narrowed.
- [ ] **Stop** — an object, ABI, rights, or rollback invariant is unresolved.
