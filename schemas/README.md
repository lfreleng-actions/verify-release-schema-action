<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 The Linux Foundation
-->

# Release schemas

The action validates release files against the schemas in this
directory. They ship with the action, so the action's pinned version
fixes the schema revision, and the action fetches no schema file from
outside the `lfreleng-actions` organisation. The action's tooling
(`uv` and `lftools-uv`) still installs from its usual sources.

## Provenance

These files come byte-for-byte from
[`lfit/releng-global-jjb`](https://github.com/lfit/releng-global-jjb),
directory `schema/`, at commit
`09659ce65d005b15595a44bbef6304fd3cc83ac1`. That's the revision this
action downloaded at run time before it bundled them.

<!-- markdownlint-disable MD013 -->

| File                               | `distribution-type` |
| ---------------------------------- | ------------------- |
| `release-artifact-schema.yaml`     | `artifact`          |
| `release-container-schema.yaml`    | `container`         |
| `release-schema.yaml`              | `maven`             |
| `release-packagecloud-schema.yaml` | `packagecloud`      |
| `release-pypi-schema.yaml`         | `pypi`              |

<!-- markdownlint-enable MD013 -->

They keep their original EPL-1.0 licence and copyright headers
(`LICENSES/EPL-1.0.txt`). Leave them unedited, so they stay
comparable with their source. `global-jjb.sha256` records their
digests, and the test workflow fails if a file no longer matches.

## Changing a schema

Changes are deliberate releases of this action, never a side effect
of another repository moving:

- **Updating an imported schema** to a newer upstream revision: copy
  the file unedited, update the commit above and the digest in
  `global-jjb.sha256`, and add fixtures for whatever changed.
- **A new distribution type, or a change this organisation alone
  needs:** add a schema of its own here under the repository's
  Apache-2.0 licence, rather than editing an imported file.
