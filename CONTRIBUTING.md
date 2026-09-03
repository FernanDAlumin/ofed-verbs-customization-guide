<!-- SPDX-License-Identifier: Apache-2.0 -->

# Contributing

Thank you for improving the guide. Contributions should make provider
customization easier to reason about without turning this repository into a
provider fork or a collection of source excerpts.

## Ground rules

1. Contributions are accepted under Apache License 2.0, as described by section
   5 of the license.
2. Use a `Signed-off-by` line to certify the Developer Certificate of Origin
   1.1. Run `git commit -s` or add the line manually.
3. Submit only material that you have the right to contribute. Obtain any
   necessary permission from an employer, university, client, or funding body.
4. Do not submit private or project-specific implementation source, patches,
   ABI layouts, logs, traces, benchmarks, credentials, topology information, or
   binaries.
5. Prefer an immutable upstream link over copied source. If copying is truly
   necessary, open a provenance issue first and identify the exact file,
   revision, copyright holder, license, and intended repository path.
6. Mark every C-like teaching fragment as non-compilable pseudocode unless it
   is intentionally introduced as a separately reviewed example program.

The DCO 1.1 text is available from the
[Linux Foundation](https://developercertificate.org/).

The DCO policy applies to contributions made after this policy was introduced;
the repository's GitHub-generated license-only initial commit predates it. A
sign-off becomes part of the permanent public commit record, including the name
and email chosen by the contributor.

## Documentation style

- The English documentation is canonical. Keep `README.zh-CN.md` aligned when
  changing project scope or the learning path.
- Use `custom_*`, `tutorial_*`, or other neutral names in pseudocode. Do not use
  names from a private implementation.
- Distinguish the public verbs API, libibverbs core, userspace provider internals,
  kernel uAPI, and hardware in every architectural claim.
- State whether an operation is supported, rejected, or outside scope. Never
  turn an unknown behavior into an implied compatibility claim.
- Pin source citations to a tag or commit rather than a floating branch.
- Add a source to `docs/REFERENCES.md` when a factual statement depends on it.

## Pull-request checklist

- [ ] The change contains no private or source-derived implementation material.
- [ ] New claims are supported by a stable public reference.
- [ ] Pseudocode is independently written and labeled non-compilable.
- [ ] Object creation, use, error, and destruction paths remain consistent.
- [ ] Unsupported behavior fails closed in the tutorial design.
- [ ] No system-provider replacement or privileged installation step was added.
- [ ] Paths, hostnames, addresses, keys, logs, and binary artifacts are absent.
- [ ] License, attribution, and trademark statements remain accurate.
- [ ] The commit includes `Signed-off-by`.
