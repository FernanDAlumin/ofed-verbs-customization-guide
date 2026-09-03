<!-- SPDX-License-Identifier: Apache-2.0 -->

# Version-record template

Use this template privately for each concrete build. Remove secrets, internal
paths, hostnames, addresses, and device identifiers before publishing a record.

```yaml
tutorial_revision: exact-guide-commit

source:
  distribution: upstream-rdma-core-or-vendor-ofed
  release: exact-release-string
  commit: exact-commit-if-available
  source_archive: exact-filename-if-applicable
  source_archive_sha256: exact-sha256-if-applicable

provider:
  name: provider-name
  private_abi: exact-value
  expected_dso: expected-private-abi-provider-name
  base_ops_location: file-and-symbol
  variant_overlays:
    - file-symbol-and-selection-condition
  custom_overlay_order: exact-order

build:
  compiler: exact-version
  generator: Ninja-or-Make
  build_type: Debug-or-Release
  cmake_options:
    - exact-option
  libibverbs_sha256: exact-sha256
  provider_sha256: exact-sha256

runtime:
  os: exact-distribution-and-version
  kernel: exact-kernel-version
  kernel_modules:
    - module-and-version
  device: model-and-revision
  firmware: exact-version

profiles:
  native_control: test-report-id
  custom_supported: test-report-id
  expected_rejections: test-report-id
  concurrency_failure: test-report-id
  rollback: test-report-id

review:
  operation_closure_matrix: path-or-report-id
  reviewer: reviewer-identity
  date: YYYY-MM-DD
```
