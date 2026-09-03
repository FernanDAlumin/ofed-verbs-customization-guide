<!-- SPDX-License-Identifier: Apache-2.0 -->

# Operation-closure template

Copy this template into the private design record for a concrete provider
customization. Add every operation present in the pinned source version.

Record both the object representation and callback policy.

- Object representation: `NATIVE`, `CUSTOM`, or `NONE`.
- Callback policy: `NATIVE`, `WRAP_NATIVE`, `CUSTOM`, `ADAPTED`, `REJECTED`,
  `NOT REACHABLE`, or `UNTESTED`.

`WRAP_NATIVE` is a custom callback over a native provider object that delegates
to the effective native callback. `CUSTOM` is reserved for callbacks that
understand a custom object representation.

This matrix classifies objects that still have a valid public base when dispatch
begins. It does not make freed public pointers safe. Record stale asynchronous
commands/completions by logical ID and generation in the relevant notes.

## Context

| Path | Representation | Callback policy | Callback/guard | Native delegate | Test ID | Notes |
|---|---|---|---|---|---|---|
| normal open |  |  |  |  |  |  |
| imported context |  |  |  |  |  |  |
| static provider |  |  |  |  |  |  |
| close with no children |  |  |  |  |  |  |
| close with live children |  |  |  |  |  |  |
| post-fork use |  |  |  |  |  |  |

## QP

| Path | Representation | Callback policy | Accepted masks/types | Object proof | Test ID | Notes |
|---|---|---|---|---|---|---|
| classic create |  |  |  |  |  |  |
| extended create |  |  |  |  |  |  |
| provider-specific create |  |  |  |  |  |  |
| `open_qp` / use on imported context |  |  |  |  |  |  |
| query |  |  |  |  |  |  |
| modify/state transitions |  |  |  |  |  |  |
| classic post send |  |  |  |  |  |  |
| extended WR builder |  |  |  |  |  |  |
| post receive |  |  |  |  |  |  |
| ECE/rate/counter helpers |  |  |  |  |  |  |
| destroy |  |  |  |  |  |  |
| async error/flush |  |  |  |  |  |  |

## CQ

| Path | Representation | Callback policy | Accepted modes | Object proof | Test ID | Notes |
|---|---|---|---|---|---|---|
| classic create |  |  |  |  |  |  |
| extended create |  |  |  |  |  |  |
| classic poll |  |  |  |  |  |  |
| extended polling |  |  |  |  |  |  |
| request notification |  |  |  |  |  |  |
| resize/modify |  |  |  |  |  |  |
| event/channel path |  |  |  |  |  |  |
| destroy |  |  |  |  |  |  |

## SRQ, MR, PD, and cross-object relationships

| Path | Representation | Callback policy | Ownership rule | Lifetime rule | Test ID | Notes |
|---|---|---|---|---|---|---|
| SRQ create/post/query/modify/destroy |  |  |  |  |  |  |
| MR classic/extended/dmabuf/import |  |  |  |  |  |  |
| MR deregister/unimport |  |  |  |  |  |  |
| PD allocate/deallocate/import |  |  |  |  |  |  |
| QP ↔ CQ association |  |  |  |  |  |  |
| QP ↔ SRQ association |  |  |  |  |  |  |
| WR ↔ MR association |  |  |  |  |  |  |
| object ↔ context ownership |  |  |  |  |  |  |

## Sign-off

- Source tag/commit:
- Provider private ABI:
- Reviewer:
- Test report:
- Unsupported profile published:
- Rollback tested:
