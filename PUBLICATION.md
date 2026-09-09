# Publication policy

This repository is public. Before publishing changes, review every tracked
file and any new repository history against the boundaries below.

## Allowed material

- Independently written summaries of controlled experiments.
- High-level architecture diagrams and interoperability conclusions.
- Measurements with a defined workload, boundary, firmware, and oracle.
- Links to public standards, open-source projects, and separately licensed
  model sources.
- Small original examples that require no proprietary platform material.
- Failure descriptions needed to explain a claim boundary, without publishing
  proprietary implementation details.

## Prohibited material

- Platform SDKs, libraries, headers, documentation, system modules, firmware,
  or files extracted from proprietary software.
- Decompiled code, copied pseudocode, symbol/offset databases, raw register or
  packet captures, and close reconstructions of proprietary implementation.
- Signing keys, certificates, account data, credentials, private URLs, IP
  addresses, personal filesystem paths, or unredacted console logs.
- Exploit chains, privilege-escalation steps, access-control bypasses, update
  manipulation, or instructions for unauthorized access.
- Proprietary title assets, captured shaders, copyrighted test media, model
  weights, or tokenizers without explicit redistribution permission.
- Claims of Khronos conformance, manufacturer approval, or firmware-wide
  compatibility without the required independent basis.

## Contributor checklist

Before committing:

1. Confirm the text is independently written and needed for a research claim.
2. Replace raw implementation details with the smallest high-level conclusion.
3. Remove local paths, hostnames, addresses, title identifiers, credentials,
   and raw logs.
4. State firmware, workload, oracle, and evidence grade.
5. Separate hardware proof from source-derived or inferred behavior.
6. Confirm third-party names, links, and measurements are attributed.
7. Keep generated binaries and evidence outside Git.
8. Run a repository-wide secret and prohibited-file review before publication.

This policy is a documentation boundary, not legal advice. Maintainers remain
responsible for obtaining appropriate legal review before public distribution.
