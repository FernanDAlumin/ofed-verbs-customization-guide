<!-- SPDX-License-Identifier: Apache-2.0 -->

# Provenance policy

This repository is an independently written tutorial about a general provider
design pattern. It contains no source code, source-derived patch, private ABI,
log, trace, benchmark result, or binary from any private research or production
implementation.

This is a project provenance representation, not a formal legal “clean-room”
certification. The repository owner must review and confirm it before each
public release.

## What the tutorial is based on

- Public `linux-rdma/rdma-core` source, pinned to release `v64.0` at commit
  `f272237493ef309984036f7b85655e11104c61c8` for line-stable references.
- Public NVIDIA MLNX_OFED documentation, used to explain how vendor source
  packages relate to upstream `rdma-core`.
- Independently authored diagrams, prose, checklists, and pseudocode.

The exact public references are listed in [docs/REFERENCES.md](docs/REFERENCES.md).

## What is deliberately excluded

- Complete upstream source files, header definitions, patches, and screenshots.
- Project-specific names, callbacks, data structures, wire formats, IPC layouts,
  constants, environment variables, algorithms, and performance parameters.
- Private repository URLs, commit identifiers, machine names, network
  addresses, filesystem paths, credentials, logs, and document metadata.
- Instructions that replace a system provider in place or distribute a custom
  provider binary.

## Contribution rule

Contributors must write from public sources or their own original knowledge.
Do not submit renamed, translated, mechanically transformed, or generated
versions of non-public source. Public upstream source should normally be linked,
not copied. See [CONTRIBUTING.md](CONTRIBUTING.md).
