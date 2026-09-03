<!-- SPDX-License-Identifier: Apache-2.0 -->

# 2. Design a context-scoped ops overlay

The upstream composition mechanism is simple; the design contract around it is
not. Start by specifying behavior and object ownership, then write the sparse
overlay.

## 2.1 Write a behavior contract first

For every operation considered for replacement, record:

| Question | Example answer format |
|---|---|
| Purpose | observe, validate, augment, reroute, or virtualize |
| Enabled scope | one context, device, process, or all contexts |
| Supported object kinds | classic RC QP created by the custom family |
| Supported attributes | explicit mask and value ranges |
| Native delegation | before custom work, after it, conditional, or never |
| Return contract | exact errno/return and partial-list behavior |
| Completion contract | opcode, status, ordering, byte length, opaque IDs |
| Concurrency | locks, atomics, ownership, and allowed simultaneous calls |
| Failure behavior | fail closed, error completion, or native fallback |
| Teardown | how new work stops and in-flight work drains |

If the table cannot be filled without “probably native” or “ignore this flag,”
the operation is not ready to advertise as supported.

## 2.2 Prefer a sparse immutable overlay

Because `verbs_set_ops()` keeps existing callbacks for null fields, a custom
table can declare only the operations it owns.

```c
/* PSEUDOCODE — NOT COMPILABLE */

static const operation_table tutorial_overlay = {
    .create_endpoint  = tutorial_create_endpoint,
    .submit_send      = tutorial_submit_send,
    .query_endpoint   = tutorial_query_endpoint,
    .destroy_endpoint = tutorial_destroy_endpoint,
};
```

This is intentionally schematic. It does not reproduce the full upstream
structure or a private implementation.

Keep the overlay constant after construction. Do not mutate a process-global
table in response to environment changes while other threads may dispatch
through it. Configuration belongs in context-local state; the callback code can
read that state after the immutable callback has been selected.

## 2.3 Install after normal provider initialization

The provider should complete the normal base and hardware-variant setup before
applying the custom policy.

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

Applying the custom overlay last is not universally correct, but it is easy to
reason about: custom callbacks win, and their saved underlay reflects the
selected hardware variant. If a provider performs more setup later, audit that
code for additional `verbs_set_ops()` calls.

## 2.4 Make configuration context-local

Read configuration once while the context is private, validate it, and store an
enum or immutable policy object in provider-owned context state.

Avoid:

- a process-global “enabled” integer shared by every device;
- rereading an environment variable on each send or poll;
- partially enabling callbacks after objects already exist;
- treating every non-empty string as true;
- allowing a context to change mode while callbacks are in flight.

A useful configuration has at least three explicit outcomes:

```text
NATIVE      normal provider behavior
CUSTOM      custom overlay and its documented object restrictions
INVALID     context creation fails with a diagnostic
```

If an internal worker, management daemon, or diagnostic process must use native
behavior, select `NATIVE` before its objects are created. Do not toggle a global
flag around individual calls.

## 2.5 Delegate without recursion

A wrapper must not call the same public verb it intercepts. The public call
looks up the context table again and re-enters the wrapper.

Use one of two provider-owned strategies:

1. call the known underlying provider implementation directly when it is
   correct for every supported variant; or
2. save the effective native callback in context-local state before installing
   the custom overlay.

There is no provider API that returns a complete copy of every effective
private callback. Do not reach into libibverbs' opaque private state. For public
classic fast-path slots, a provider can identify the callback installed in the
context after its normal overlays. For other operations, keep an explicit
provider-owned underlay selector or call the correct named provider
implementation. The choice must include all provider cleanup and bookkeeping,
not only the lowest-level kernel command.

```text
PSEUDOCODE — NOT COMPILABLE

tutorial_submit_send(public_handle, request, bad_request):
    state = state_owned_by(public_handle.context)
    validate_owner_and_request(state, public_handle, request)

    if public_handle is proven native and request should use native behavior:
        return state.native.submit_send(public_handle, request, bad_request)

    return state.custom_backend.submit(public_handle, request, bad_request)
```

Passing the public handle directly to a native callback is valid only when the
handle is a genuine native or explicitly native-backed provider object accepted
by that callback. A strict custom façade cannot be delegated this way; it must
be handled entirely by custom code or translated to a separate, genuine native
underlay object with its own lifetime.

Do not use `dlsym(RTLD_NEXT, ...)` to find the provider underlay. This is not a
stack of public ELF wrappers; it is a provider-internal call graph with
context-specific variants.

## 2.6 Decide how custom callbacks reach their backend

The callback might perform local policy or redirect to a separate runtime. Keep
that backend contract independent of provider-private object layout.

```text
PSEUDOCODE — NOT COMPILABLE

custom_backend.submit(command):
    validate(command.kind, lengths, identifiers, and ownership)
    reserve_bounded_capacity_or_fail()
    publish(command)
    return according_to_the_public_verb_contract
```

The guide deliberately does not prescribe sockets, shared-memory layouts,
queues, wire headers, or scheduling algorithms. Those are separate system
designs. Whatever transport is chosen must define:

- framing and version negotiation;
- request/response correlation under concurrency;
- buffer ownership and lifetime;
- bounded queues and backpressure;
- crash detection and cleanup;
- completion ordering and stale-generation rejection;
- a threat model for mutually trusted or untrusted processes.

## 2.7 Treat unsupported behavior as part of the API

An overlay is safer when it supports less and says so precisely. Reject an
unsupported constructor or attribute before creating a handle. Preserve the
provider/libibverbs error convention for the specific function: some return an
errno value, some return `-1` and set `errno`, and some return `NULL` and set
`errno`.

Remember that a null field in a sparse overlay means “keep the previously
installed callback,” not “disable this operation.” If an inherited constructor
would create an object that violates custom-context closure, install an explicit
rejecting callback with the correct signature and error convention.

The same audit applies to inherited consumers, destructors, CQ notification and
event paths, and object-local extended methods. A native callback must never
receive a custom façade merely because its field was left null in the sparse
overlay.

Do not silently:

- discard an attribute bit;
- truncate an opaque ID;
- convert an extended object to a classic object unless the conversion is
  defined and lossless;
- report success for a completion whose data was truncated;
- route an unknown object to a native callback.

## 2.8 Worked design: observe and delegate

The smallest useful lab adds per-context observation to one classic operation
while leaving every object native. It is deliberately less ambitious than
virtualizing a QP or CQ, which makes it a good first proof of dispatch.

### Contract

- Intercept classic `post_send` for contexts explicitly opened in tutorial mode.
- Count calls and WRs in context-local atomics.
- Never inspect or log payloads, addresses, or memory keys.
- Delegate the unmodified request to the effective native callback.
- Preserve its return value and `bad_wr` result exactly.
- Leave extended WR-builder APIs outside the claimed profile.

### Context setup

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

The wrapper calls the saved native callback, not `ibv_post_send()`. It does not
create a new QP kind, so native QP representation and lifecycle remain intact.
Its support statement is narrow: classic post-send observation only.

The context-local counters still need cleanup. Either extend the provider's
normal context cleanup to release the custom state before it frees the provider
context, or overlay `free_context` with a wrapper that closes custom resources
and then calls the saved complete native free callback. Do not allocate
context-local state without a symmetric close path.

### Evidence

Run four tests:

1. unmodified build: classic ping-pong succeeds;
2. modified build, mode off: ping-pong succeeds and counters remain zero;
3. modified build, observe mode: ping-pong succeeds and deterministic call/WR
   counts match the test;
4. extended send (`ibv_qp_ex`/the ping-pong `-N` path): either remains explicitly
   native and excluded from the claim, or is rejected by a separately designed
   guard; it must not be reported as observed by the classic callback.

Only after this profile is correct should the experiment replace constructors
or return custom objects. At that point, Chapter 3's object-family closure rules
become mandatory.

## 2.9 Initialization and failure checklist

- [ ] The custom state is allocated per context or per device as designed.
- [ ] Base and hardware overlays are installed in a documented order.
- [ ] Effective native callbacks are recorded before custom replacement.
- [ ] The custom overlay is immutable once the context is published.
- [ ] Invalid configuration aborts context initialization cleanly.
- [ ] Partial initialization has a reverse-order cleanup path.
- [ ] Context close stops new custom work, closes backend/registry resources,
  destroys custom synchronization state, and then completes native cleanup.
- [ ] Internal/native contexts cannot accidentally enable custom callbacks.
- [ ] Callback diagnostics identify mode and context without logging payloads,
  addresses, memory keys, or credentials.

Next: [Close object families and lifecycles](03-close-object-lifecycles.md).
