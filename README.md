<!--
# SPDX-License-Identifier: Apache-2.0
# SPDX-FileCopyrightText: 2025 The Linux Foundation
-->

# ✅ Verify Release Schema Action

Verifies a release file's contents against an approved schema.

Supported distribution types: `artifact`, `container`, `maven`,
`packagecloud`, and `pypi` (values are case-sensitive and MUST be
lowercase).

## verify-release-schema-action

## Usage Example

Validates a release file against the schema for the specified distribution
type.

<!-- markdownlint-disable MD013 -->

```yaml
steps:
  - name: "Verify Release Schema"
    id: verify-schema
    # Pin to a commit SHA for supply-chain hardening and
    # reproducible runs. Replace with a newer SHA (or a release
    # tag, once published) when upgrading.
    uses: lfreleng-actions/verify-release-schema-action@456dc3b587011f5b20ddcfde30228aa6d9132d27
    with:
      distribution-type: "maven"
      release-file: "releases/release.yaml"
```

<!-- markdownlint-enable MD013 -->

## Inputs

<!-- markdownlint-disable MD013 -->

| Name               | Required | Default                                   | Description                                                                                                                                                                       |
| ------------------ | -------- | ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| distribution-type  | True     |                                           | Release content type. Accepted values (lowercase, case-sensitive): `artifact`, `container`, `maven`, `packagecloud`, `pypi`                                                       |
| release-file       | True     |                                           | Path to the release file describing the release contents                                                                                                                          |
| lftools-uv-version | False    | _(empty — latest from PyPI)_              | Optional exact `lftools-uv` version to run via `uvx` (interpolated into `lftools-uv==<value>`, not a full pip specifier). Leave unset to use the latest version published on PyPI |
| schema-ref         | False    | pinned commit of `lfit/releng-global-jjb` | Git ref (branch, tag or commit SHA) of `lfit/releng-global-jjb` to fetch schemas                                                                                                  |

<!-- markdownlint-enable MD013 -->

## Outputs

This action does not produce any outputs.

## Behavior

- Supplying an unsupported `distribution-type` causes the action to
  fail fast with a non-zero exit code.
- The action downloads the schema from `lfit/releng-global-jjb` at the
  ref specified by `schema-ref`. Pin this input to a commit SHA for
  fully deterministic validation.
- Schema validation delegates to `lftools-uv schema verify` (run via
  `uvx` in an isolated, cached environment); a non-zero exit code from
  `lftools-uv` propagates to the action step.

## Requirements/Dependencies

`curl` must be available on the runner, along with a bash/Linux-like
shell environment. The action installs [uv](https://docs.astral.sh/uv/)
via [astral-sh/setup-uv](https://github.com/astral-sh/setup-uv) and
then invokes
[lftools-uv](https://pypi.org/project/lftools-uv/)
(the modernised successor to `lftools`) via `uvx` to perform schema
validation. By default the action runs the latest version of
`lftools-uv` published on PyPI; set the optional `lftools-uv-version`
input to pin to a specific version for reproducible runs. The action
does not require a pre-existing Python or `pip` installation — `uv`
provides its own managed Python toolchain.

## Notes

- Supported distribution types and their schemas:
  - `artifact` — `release-artifact-schema.yaml`
  - `container` — `release-container-schema.yaml`
  - `maven` — `release-schema.yaml`
  - `packagecloud` — `release-packagecloud-schema.yaml`
  - `pypi` — `release-pypi-schema.yaml`
- The action retrieves schema sources from the
  [lfit/releng-global-jjb](https://github.com/lfit/releng-global-jjb)
  repository at the ref specified by `schema-ref`.
- The `schema verify` sub-command in `lftools-uv` is a direct port of
  the original `lftools schema verify` implementation (same
  `jsonschema.Draft4Validator` semantics), so validation results match
  the legacy tool while avoiding its heavier, older dependency tree.
