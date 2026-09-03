<!-- SPDX-License-Identifier: Apache-2.0 -->

# 5. Validate behavior and compatibility

[简体中文](zh-CN/05-validation-matrix.md)

A custom callback is not validated when it prints a message. Validation must
show that the intended callback ran, native-off behavior stayed unchanged,
object families remained type-safe, and unsupported paths failed as designed.

## 5.1 Define profiles instead of saying “verbs compatible”

Create a versioned support table for each release of the customization.

| Area | Operation/mode | Object representation | Callback policy | Required evidence |
|---|---|---|---|---|
| Context | native mode | Native | `NATIVE` | baseline comparison and loaded-object proof |
| QP | classic RC create/query/modify/destroy | Native or custom | `WRAP_NATIVE`, `CUSTOM`, or `REJECTED` | lifecycle tests |
| QP | extended create, `open_qp`, imported-context path | Native, custom, or none | `NATIVE`, `ADAPTED`, or `REJECTED` | mask-by-mask negative tests |
| Send | opcode, list length, SGE count, flags | QP-profile dependent | `WRAP_NATIVE`, `CUSTOM`, or `REJECTED` | result and `bad_wr` tests |
| Receive | classic/SRQ, buffer limits | QP/SRQ-profile dependent | explicit policy | data/error semantic tests |
| CQ | classic poll/destroy | Native or custom | `WRAP_NATIVE`, `CUSTOM`, or `REJECTED` | batch and lifetime tests |
| CQ | ex/event/notify/resize/modify | Native, custom, or none | explicit policy | creation and event tests |
| MR | classic/ex/dmabuf/import | Native, custom, or none | explicit policy | ownership and lifetime tests |
| Concurrency | threads/contexts/devices/fork | Profile-specific | `SUPPORTED` or `REJECTED` | synchronization and isolation tests |

Never use “not tested” as a synonym for “supported.” Use `UNTESTED` as its own
state and exclude it from compatibility claims.

For extended QP and CQ profiles, verify object-local WR-builder and polling
callbacks as well as the context operation table. A classic callback counter
remaining at zero during an extended test can be correct evidence that the path
bypassed the classic overlay, not evidence that no operation occurred.

Name the representation profile for each family:

| Profile | Object representation | Callback behavior |
|---|---|---|
| Native control | Native | Native callback |
| Native-object wrapper | Native | Custom wrapper delegates to saved native callback |
| Strict custom family | Custom | Custom callbacks or explicit rejection for the whole family |
| Registry mixed family | Native and custom | Registry classifies before native/custom dispatch |

## 5.2 Phase 1: source and build checks

- [ ] The source tag, commit, provider-private ABI, and compiler are recorded.
- [ ] The provider target builds from a clean directory.
- [ ] Warnings introduced by the customization are treated as failures.
- [ ] No unresolved symbol is hidden by loading a different system library.
- [ ] The custom overlay is constant after initialization.
- [ ] Every non-null custom field is listed in the support matrix.
- [ ] Sanitizer-enabled or equivalent debug builds are available where the
  provider and test environment support them.

Compile-only CI is useful but does not prove that a matching device selected the
provider.

## 5.3 Phase 2: provider-selection checks

For native and custom modes, record:

- executable path;
- loaded libibverbs path;
- loaded provider path and private-ABI suffix;
- provider build identity;
- selected context mode;
- device and port in a private lab record.

Fail the test if the expected custom callback count is zero. Also fail if a
custom callback appears in native mode.

## 5.4 Phase 3: native-off equivalence

With customization disabled:

1. enumerate and query devices;
2. create and destroy the target object families;
3. run the native smoke traffic supported by the lab;
4. exercise any hardware variants affected by overlay order;
5. compare return values, completions, and diagnostics with the unmodified
   pinned build.

The goal is semantic equivalence, not cycle-identical performance. Any
difference requires an explanation before custom mode is tested.

## 5.5 Phase 4: object-family closure

### QP family

- in native-object wrapper mode, native creators/destructors remain intact and
  the custom callback accepts only the native representation before delegating;
- in strict custom-family mode, classic create reaches the custom creator and
  query, modify, post, and destroy recognize the resulting object;
- in registry mixed mode, every reachable creator registers the correct kind
  before the object can reach a routed callback;
- every extended creator either follows its profile, adapts losslessly, or
  fails early;
- `open_qp`, provider-specific creation, and use on imported contexts follow
  the selected profile explicitly;
- a QP cannot attach an incompatible CQ, SRQ, PD, or context;
- failed create leaves no registered or partially live object;
- invalid double-destroy/use-after-destroy behavior is not claimed unless an
  explicit delayed-reclamation or tombstone contract exists.

### CQ family

- classic create/poll/destroy form a complete path;
- extended create is implemented or rejected;
- event channel, notify, resize, modify, and CQ event behavior are explicit;
- a custom CQ never reaches a native callback that expects a larger native
  object unless the CQ is genuinely a native object;
- poll-versus-destroy races are controlled.

### SRQ and MR families

- an unsupported SRQ is rejected before association with a custom QP;
- MR kind and owning PD/context are validated before use;
- imported and dmabuf memory follows an explicit policy;
- unregister/deregister cannot race with an in-flight operation without a
  defined lifetime mechanism.

## 5.6 Phase 5: public API semantics

For each supported operation test:

- success and every reachable error return convention;
- the first failing WR and exact `bad_wr` pointer;
- multi-WR partial success;
- zero-length and maximum supported lengths;
- supported and unsupported SGE counts;
- flags, signaling, inline behavior, and opcodes;
- full-width opaque `wr_id` round trips;
- polling with `ne = 0`, `1`, and a batch;
- completion status/opcode/length/identity;
- ordering within each QP and across shared resources where promised;
- receive-before-send and send-before-receive behavior;
- buffer-too-small behavior;
- flush and failure completions.

Document any semantic delta from native behavior next to the support matrix.

## 5.7 Phase 6: unknown and foreign handles; stale async work

Negative tests should pass handles that are:

- valid native objects produced by a reachable constructor in mixed mode;
- valid objects used in a cross-context relationship accepted by the public API;
- valid provider objects deliberately omitted from the registry by a controlled
  internal fault-injection hook.

Strict mode should make these paths unreachable through constructors, then
assert that invariant in debug builds. Mixed mode must look up a still-valid
handle before downcast and fail closed for an unclassified entry. Do not create
tests by casting one public object family to another or fabricating arbitrary
pointers; those violate the public API before provider classification begins.

Test stale *custom commands and completions* separately by attaching a
generation to those records, destroying/recreating the logical object, and
proving that old-generation work is rejected. Do not pass a freed C handle and
interpret a non-crash as provider safety; the public verb may dereference it
before the wrapper runs.

Run ASan/UBSan or equivalent instrumentation where practical. “It did not
crash” is not proof that an invalid container conversion was safe.

## 5.8 Phase 7: concurrency and failure

Exercise at least:

- simultaneous context creation with different modes;
- concurrent post/poll on supported objects;
- if concurrent destroy is explicitly supported, controlled
  create/post/poll/destroy interleavings under the delayed-lifetime design;
- otherwise, tests that enforce the documented caller-synchronization boundary
  without issuing a public verb on a freed handle;
- backend queue full and bounded backpressure;
- backend disconnect during a call;
- stale completion after object destruction;
- context close with live children;
- two devices or an explicit single-device rejection;
- fork, or an explicit unsupported result after fork.

Use deterministic barriers in tests rather than relying only on stress timing.
Then add longer stress tests to find issues outside the modeled interleavings.

## 5.9 Phase 8: hardware and performance

Run hardware tests only after build, loader, native-off, closure, and negative
tests pass. Begin with a documented upstream smoke test and the smallest custom
profile.

Performance claims require a separate methodology:

- identical hardware, firmware, kernel, provider build flags, CPU affinity, and
  link configuration;
- native-off and unmodified baselines;
- multiple runs and a stated aggregation method;
- setup and steady-state costs reported separately;
- CPU and memory costs as well as latency/throughput;
- raw data and scripts that can legally be published.

This tutorial makes no performance claim.

## 5.10 Release acceptance criteria

A customization is ready for a controlled lab release only when:

1. the source and loaded-provider identities match;
2. native mode matches the unmodified control for the tested profile;
3. every custom table field has closed object and error paths;
4. unsupported constructors and attributes fail explicitly;
5. no callback recursively calls its public verb;
6. valid but unclassified objects fail before downcast, and stale custom async
   records are rejected by generation;
7. teardown and context close have deterministic ownership and state their
   caller-synchronization requirements;
8. concurrency and backend-failure tests pass;
9. the support matrix matches the tests and makes no raw-pointer UAF guarantee;
10. rollback has been performed successfully at least once.

Next: [Handle versions and vendor forks](06-versioning-and-portability.md).
