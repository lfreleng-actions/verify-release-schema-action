<!--
# SPDX-License-Identifier: Apache-2.0
# SPDX-FileCopyrightText: 2025 The Linux Foundation
-->

# ✅ Verify Release Schema Action

Verifies a release file's contents against an approved schema. Supported
distribution types: artifact, container, Maven, PackageCloud, and PyPI.

## verify-release-schema-action

## Usage Example

Validates a release file against the schema for the specified distribution
type.

<!-- markdownlint-disable MD013 -->

```yaml
steps:
  - name: "Verify Release Schema"
    id: verify-schema
    uses: lfreleng-actions/verify-release-schema-action@main
    with:
      distribution-type: "maven"
      release-file: "releases/release.yaml"
```

<!-- markdownlint-enable MD013 -->

## Inputs

<!-- markdownlint-disable MD013 -->

| Name              | Required | Default | Description                              |
| ----------------- | -------- | ------- | ---------------------------------------- |
| distribution-type | True     |         | The release content type                 |
| release-file      | True     |         | File that describes the release contents |

<!-- markdownlint-enable MD013 -->

## Outputs

This action does not produce any outputs.

## Requirements/Dependencies

Python and pip must be available on the runner. The action installs
[lftools](https://pypi.org/project/lftools/) to perform schema validation.

## Notes

- Supported distribution types and their schemas:
  - `artifact` — `release-artifact-schema.yaml`
  - `container` — `release-container-schema.yaml`
  - `maven` — `release-schema.yaml`
  - `packagecloud` — `release-packagecloud-schema.yaml`
  - `pypi` — `release-pypi-schema.yaml`
- The action fetches schemas from the
  [lfit/releng-global-jjb](https://github.com/lfit/releng-global-jjb)
  repository at runtime.
- Unsupported distribution types will cause the action to report an error
  and skip schema validation.
