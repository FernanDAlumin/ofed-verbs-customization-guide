<!-- SPDX-License-Identifier: Apache-2.0 -->

# 7. Licensing and provenance

This chapter explains the repository's own license and the different obligations
that arise when a reader modifies an OFED/rdma-core provider. It is an
engineering checklist, not legal advice for a particular organization or
jurisdiction.

## 7.1 License of this tutorial

The original prose, diagrams, checklists, and independently authored pseudocode
in this repository are licensed under Apache License 2.0. The complete text is
in the root [LICENSE](../LICENSE) file.

Apache-2.0 grants copyright permissions and an express patent license limited to
claims a contributor can license that are necessarily infringed by the
contribution as defined by the license. It also requires preservation of
applicable notices and prominent identification of modified files.

The license does not turn external links into included material, grant rights in
third-party projects, or grant third-party trademarks.

## 7.2 Publicly visible is not the same as included or relicensed

This tutorial links to public rdma-core and NVIDIA documentation. It does not
copy or distribute those materials. Their authors and licenses remain separate.

Do not copy an upstream source file into an Apache-only project and replace its
header. If an excerpt or file must be distributed, determine the governing
license from:

1. the exact file header;
2. any license file in that directory or its ancestors;
3. the root license rules for that exact release;
4. vendor additions or packaging terms;
5. NOTICE and modification-marking requirements.

The public rdma-core v64.0 root describes a default choice between GPLv2 and an
OpenIB.org BSD variant, while allowing individual files and directories to
override the default. Therefore “rdma-core is BSD” is not an adequate
file-by-file license conclusion.

## 7.3 Rules for a provider customization project

If you publish an actual provider patch or modified source tree:

- preserve original copyright and license notices;
- add an accurate copyright notice for original contributions only when the
  rights holder is known;
- mark modified files and changes as required;
- include the exact upstream/vendor license texts that apply;
- keep a provenance record mapping every file to its origin and revision;
- audit generated files, copied headers, build scripts, examples, and tests;
- do not assume the tutorial's Apache-2.0 license can cover upstream files;
- confirm rights from employers, universities, clients, coauthors, and funders;
- resolve patent strategy before public disclosure.

When targeting upstream rdma-core, follow its contribution and DCO process. A
local vendor fork may have a different submission process.

## 7.4 Linking is the default documentation policy

For educational explanation, prefer an immutable link plus your own description:

```text
At rdma-core v64.0, verbs_set_ops() overlays non-null fields and leaves the
remaining callbacks unchanged. See the pinned upstream implementation.
```

Avoid pasting the complete function, full operation table, provider object
definition, or source screenshot. Linking:

- preserves the upstream context and attribution;
- makes version drift visible;
- prevents an accidental mixed-license source file;
- helps keep tutorial pseudocode independent from production implementation.

## 7.5 Pseudocode rules

Every teaching fragment in this repository must be:

- independently written from the described design principle;
- visibly labeled `PSEUDOCODE — NOT COMPILABLE`;
- incomplete with respect to concrete ABI layouts and deployment details;
- expressed with neutral `tutorial_*` or `custom_*` names;
- free of private callback names, message layouts, constants, paths, and
  performance parameters.

Renaming identifiers or translating comments from a private or upstream source
does not create independent pseudocode.

## 7.6 Contributions

Contributions are accepted under Apache-2.0 section 5 and the Developer
Certificate of Origin 1.1 process described in
[CONTRIBUTING.md](../CONTRIBUTING.md). A sign-off certifies provenance; it does
not erase third-party license obligations.

Do not submit:

- private provider source or a source-derived rewrite;
- non-public design documents or generated output based on private source;
- real logs, traces, memory addresses/keys, hostnames, or network topology;
- a complete upstream file without prior license and provenance review;
- third-party logos, screenshots, figures, fonts, or specification PDFs merely
  because they are accessible online.

## 7.7 Patents, publication, and confidentiality

Publishing a tutorial can disclose an enabling technical method even when no
production code is released. Copyright and patent are separate. If the method
may be patentable or subject to an invention-assignment agreement, obtain advice
and authorization before publication. Public disclosure can affect patent
novelty, and Apache-2.0 includes a contributor patent grant within its terms.

Likewise, independently phrased text can still disclose a trade secret or
NDA-protected design. “No code copied” is not a substitute for confidentiality
clearance.

## 7.8 Trademarks and non-affiliation

Names such as NVIDIA, Mellanox, ConnectX, mlx5, MLNX_OFED, Linux, and rdma-core
are used for factual identification. Do not use vendor logos or imply that a
custom provider is official, certified, supported, or endorsed. See
[TRADEMARKS.md](../TRADEMARKS.md).

## 7.9 Release checklist

- [ ] Every repository file is original or has an approved provenance record.
- [ ] External source links are pinned to a tag/commit where possible.
- [ ] `LICENSE`, `THIRD_PARTY_NOTICES.md`, any applicable `NOTICE`, and the
  actual contents agree.
- [ ] No private names, code, ABI, logs, paths, addresses, keys, or binaries are
  present in any Git ref or release archive.
- [ ] Copyright and patent authority is confirmed.
- [ ] Trademarks are descriptive and non-endorsing.
- [ ] DCO sign-offs exist for accepted contributions.
- [ ] A clean clone contains only the intended tutorial source.

Next: [Run the final design review](DESIGN_REVIEW_CHECKLIST.md).
