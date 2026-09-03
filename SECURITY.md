<!-- SPDX-License-Identifier: Apache-2.0 -->

# Security policy

## Supported versions

Security and safety corrections are applied to the latest version on the
default branch. This repository publishes documentation and non-compilable
pseudocode; it does not publish an installable provider.

| Version | Supported |
|---|---|
| Latest default branch | Yes |
| Older commits and forks | No |

## Reporting a problem

Before publication, the maintainer must enable and test GitHub Private
Vulnerability Reporting. Use that channel for reports that could expose
sensitive information or cause unsafe modification of an RDMA system. If the
Security tab does not offer a private report form, do not put sensitive details
in a public issue; contact the repository owner through a private channel they
advertise first. Include the affected page, the unsafe outcome, and a minimal
reproduction that contains no credentials, real payload, memory key, or private
topology data.

The maintainer will acknowledge a report when practical, confirm the affected
scope, coordinate a correction, and agree on disclosure timing with the
reporter. This is a community project and does not promise a fixed response SLA.

Documentation issues that belong here include:

- a command that can unintentionally overwrite or load a system provider;
- a missing version check that can mix incompatible libibverbs/provider ABIs;
- pseudocode that permits an unsafe downcast, use-after-free, recursion, or
  fail-open path;
- accidental inclusion of private implementation material, logs, addresses,
  memory keys, credentials, or binaries;
- a dependency or archive instruction that retrieves unpinned content.

Issues in real `rdma-core`, MLNX_OFED, a kernel RDMA driver, firmware, or
hardware should be reported to the relevant upstream or vendor security process,
not to this tutorial.

## Safe-use boundary

Perform provider experiments only on a disposable development host, VM, or
dedicated lab node. Build and load the provider side by side with the system
stack. Do not replace files under `/usr` as part of this tutorial. Preserve an
independent shell or console path for rollback before testing an RDMA provider.
Test only systems and devices you own or are explicitly authorized to use.
