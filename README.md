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
    # reproducible runs. The outputs below first shipped in v1.0.0:
    # REPLACE the placeholder with the commit SHA of v1.0.0 or later.
    # An older SHA validates, but leaves every output empty.
    uses: lfreleng-actions/verify-release-schema-action@0000000000000000000000000000000000000000  # REPLACE: v1.0.0 or later
    with:
      distribution-type: "maven"
      release-file: "releases/release.yaml"

  - name: "Use the release coordinates"
    shell: bash
    env:
      VERSION: "${{ steps.verify-schema.outputs.version }}"
      REF: "${{ steps.verify-schema.outputs.ref }}"
    run: |
      # A contributor authors the release file, so a value may carry
      # line breaks. Collapse CR as well as LF: the runner reads
      # stdout with ReadLine(), which treats a bare CR as a line
      # boundary too, so stripping LF alone would still let a value
      # like $'ok\r::stop-commands::x' emit a workflow command.
      printf 'Releasing %s from %s\n' \
        "${VERSION//[$'\r\n']/ }" "${REF//[$'\r\n']/ }"
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

Scalar fields read from the release file, published once validation
passes. Absent fields yield an empty string rather than failing,
because the schemas do not require every field.

<!-- markdownlint-disable MD013 -->

| Name                | Description                                                                  |
| ------------------- | ---------------------------------------------------------------------------- |
| `distribution_type` | Release content type declared in the file                                    |
| `project`           | Project the release file names                                               |
| `version`           | Version this release publishes                                               |
| `git_tag`           | Git tag named by the file. Empty when absent — see note below                |
| `log_dir`           | Jenkins log directory locating the staging build. Empty when absent          |
| `ref`               | Commit the release builds from. Empty when absent                            |
| `tag_release`       | `'true'` or `'false'`. Empty when absent; `global-jjb` treats absent as true |

<!-- markdownlint-enable MD013 -->

The action emits values verbatim. `git_tag` does **not** fall back to
`version`: `global-jjb`'s `release-job.sh` applies that fallback, but
reporting a value the file does not contain would mislead, and a
caller wanting it can write the fallback directly:

<!-- markdownlint-disable MD013 -->

```yaml
tag: ${{ steps.verify-schema.outputs.git_tag || steps.verify-schema.outputs.version }}
```

<!-- markdownlint-enable MD013 -->

Type-specific scalars (`package_name`, `pypi_project`,
`container_release_tag`) and list-valued fields (`artifacts`,
`containers`, `packages`) stay out of scope. GitHub requires static
output declarations, so adding them is a per-consumer decision rather
than something to do preemptively.

### Quoting requirements

Except for `tag_release`, which the schemas type as a boolean, quote
these fields in the release file. YAML 1.1 rewrites unquoted values
before the action sees them, and nothing recovers the authored text
afterwards:

<!-- markdownlint-disable MD013 -->

| Written         | Loaded as | Would publish | Result                        |
| --------------- | --------- | ------------- | ----------------------------- |
| `ref: on`       | `True`    | `true`        | ref rewritten                 |
| `ref: 0123`     | `83`      | `83`          | read as octal                 |
| `ref: [abc]`    | list      | `['abc']`     | Python syntax in an output    |
| `ref: null`     | `None`    | `''`          | indistinguishable from absent |

<!-- markdownlint-enable MD013 -->

Those all use `ref`, which the Maven schema neither declares nor forbids, so
each one **validates** and reaches extraction. The schema cannot catch them;
the action must.

The schema catches a _declared_ field earlier. An unquoted `version: 1.10`
would load as the float `1.1` and lose its trailing zero, but the schema types
`version` as a string, so validation rejects it before extraction sees it. The
hazard is identical; the layer that catches it differs.

Rather than publish a rewritten coordinate, the action fails with a message
naming the field and the type it found. Quoting preserves the value.

> **Upgrading from v0.5.x:** this check runs even when a caller ignores the
> outputs, so a release file that validated before can now fail. That
> happens when one of the seven fields above holds an unquoted value that
> YAML loads as something other than a string, such as `ref: on`. The error
> names the field; quote its value to fix it. Of 180 real ONAP release
> files measured, none tripped the check.

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
