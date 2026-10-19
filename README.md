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
    # reproducible runs. The outputs below first shipped in v1.0.0;
    # an older SHA validates, but leaves every output empty. The
    # schemas ship with the action from v1.1.0; an earlier SHA
    # still fetches them from lfit/releng-global-jjb at run time.
    uses: lfreleng-actions/verify-release-schema-action@<sha>  # vX.Y.Z
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
| distribution-type  | True     |                                           | Release content type. Accepted values (lowercase, case-sensitive): `artifact`, `container`, `maven`, `npm`, `packagecloud`, `pypi`                                                |
| release-file       | True     |                                           | Path to the release file describing the release contents                                                                                                                          |
| lftools-uv-version | False    | _(empty — latest from PyPI)_              | Optional exact `lftools-uv` version to run via `uvx` (interpolated into `lftools-uv==<value>`, not a full pip specifier). Leave unset to use the latest version published on PyPI |
| schema-ref         | False    | _(empty)_                                 | **Deprecated and ignored.** The action validates against its bundled schemas; setting this emits a warning and nothing else                                                       |

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

The action emits string fields verbatim, and renders the boolean
`tag_release` as lowercase `true` or `false`. `git_tag` does **not**
fall back to `version`: `global-jjb`'s `release-job.sh` applies that
fallback, but reporting a value the file does not contain would
mislead, and a caller wanting it can write the fallback directly:

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
> happens when one of the six string fields above (every field but
> `tag_release`) holds an unquoted value that YAML loads as something other
> than a string, such as `ref: on`, or when `tag_release` isn't a boolean.
> The error names the field: quote a string field's value, and write
> `tag_release` as an unquoted `true` or `false`. Of 180 real ONAP release
> files measured, none tripped the check.

## Behavior

- Supplying an unsupported `distribution-type` causes the action to
  fail fast with a non-zero exit code.
- The action validates against the schemas bundled in its `schemas/`
  directory, so pinning the action's commit SHA also pins the schema
  revision. The schema files are no longer fetched from any other
  repository; the action still installs `uv` and `lftools-uv` (see
  Requirements).
- Deprecated: `schema-ref`, which used to select the
  `lfit/releng-global-jjb` revision to download. The action now
  ignores it, with a warning, so existing callers keep working.
- Schema validation delegates to `lftools-uv schema verify` (run via
  `uvx` in an isolated, cached environment); a non-zero exit code from
  `lftools-uv` propagates to the action step.

## Requirements/Dependencies

The action needs a bash/Linux-like shell environment. It installs
[uv](https://docs.astral.sh/uv/) via
[astral-sh/setup-uv](https://github.com/astral-sh/setup-uv) and
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
  - `npm` — `release-npm-schema.yaml`
  - `packagecloud` — `release-packagecloud-schema.yaml`
  - `pypi` — `release-pypi-schema.yaml`
- The imported schemas came unedited from
  [lfit/releng-global-jjb](https://github.com/lfit/releng-global-jjb),
  where they're licensed EPL-1.0; they keep that licence.
  [`schemas/README.md`](schemas/README.md) records the source commit
  and how to change a schema, and the test workflow checks each
  imported file against its recorded digest.
- The imported container schema checks that `containers` is an
  array, but not any entry inside it. It places `properties` and
  `additionalProperties: false` directly on `containers`, and JSON
  Schema Draft 4 ignores those keywords on any instance that is not
  an object, so they never reach the entries. The action accepts an
  entry such as `name: 'Bad Name!'`, an entry with extra keys, or a
  bare `42`.
  The imported file stays unedited, per
  [`schemas/README.md`](schemas/README.md).
  [`lfreleng-actions/docker-release-detect-action`](https://github.com/lfreleng-actions/docker-release-detect-action)
  applies Docker's name and tag grammar to each entry; its section
  [The release container schema](https://github.com/lfreleng-actions/docker-release-detect-action#the-release-container-schema)
  compares the two validators field by field.
- The `npm` schema originates in this repository (Apache-2.0). Its one
  required field is `version`, a full semantic version such as `1.1.2`
  or `2.0.0-rc.1`, so a file holding a `project` and a `version` is
  valid. `distribution_type`, when present, must be `npm`; `git_tag`,
  `tag_release` and `ref` are optional.
- The `schema verify` sub-command in `lftools-uv` is a direct port of
  the original `lftools schema verify` implementation (same
  `jsonschema.Draft4Validator` semantics), so validation results match
  the legacy tool while avoiding its heavier, older dependency tree.
