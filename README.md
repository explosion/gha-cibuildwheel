# explosion/gha-cibuildwheel

A reusable GitHub Actions workflow that builds wheels and a source
distribution for Explosion's Python packages and attaches them to a draft
GitHub release.

It supports:

- **Compiled packages** (spaCy, thinc, cymem, preshed, ...): wheels are built
  with [cibuildwheel](https://cibuildwheel.pypa.io/) on Linux (x86_64 and
  aarch64), Windows (x64 and ARM64) and macOS (Intel and Apple silicon).
- **Pure Python packages** (confection, weasel, ...): a single universal
  wheel is built with `uv build`.

Publishing to PyPI is not part of this workflow. Releases are published
from the GitHub release by
[explosion/wheelmonger](https://github.com/explosion/wheelmonger).

## Usage

Add `.github/workflows/cibuildwheel.yml` to the package repository. Pin the
workflow to a commit SHA:

```yaml
name: Build

on:
  push:
    tags:
      # ** matches 'zero or more of any character'
      - 'release-v[0-9]+.[0-9]+.[0-9]+**'
      - 'prerelease-v[0-9]+.[0-9]+.[0-9]+**'

permissions: {}

jobs:
  build_wheels:
    uses: explosion/gha-cibuildwheel/.github/workflows/cibuildwheel.yml@<sha> # main
    permissions:
      contents: write
      actions: read
    with:
      wheel-name-pattern: "preshed-*.whl"
      pure-python: false
    secrets:
      gh-token: ${{ secrets.GITHUB_TOKEN }}
```

For a compiled package, configure cibuildwheel under `[tool.cibuildwheel]`
in `pyproject.toml` (which Python versions to `skip`, the manylinux images,
repair commands and so on).

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `wheel-name-pattern` | Pattern matching the built wheels, e.g. `"preshed-*.whl"`. **Required.** | |
| `pure-python` | Build one universal wheel instead of running cibuildwheel | `false` |
| `package-dir` | Path to the package directory | `.` |
| `output-dir` | Directory the wheels are written to | `wheelhouse` |
| `config-file` | cibuildwheel config file | `{package}/pyproject.toml` |
| `python-version` | Python version for pure Python builds | `3.11` |
| `tag-pattern-release` | Tag pattern for release builds | `release-v[0-9]+.[0-9]+.[0-9]+**` |
| `tag-pattern-prerelease` | Tag pattern for prerelease builds | `prerelease-v[0-9]+.[0-9]+.[0-9]+**` |
| `create-release` | Create a draft GitHub release with the artifacts | `true` |
| `env` | Extra environment variables for the builds, one `KEY: value` per line | `''` |
| `submodules` | Check out submodules: `"true"`, `"false"` or `"recursive"` | `'false'` |
| `os-matrix` | JSON list of runners to build compiled wheels on | see below |

The default `os-matrix` is:

```json
["ubuntu-latest", "ubuntu-24.04-arm", "windows-latest", "windows-11-arm", "macos-15-intel", "macos-latest"]
```

A macOS-only package can pass:

```yaml
    with:
      os-matrix: '["macos-15-intel", "macos-latest"]'
```

## Secrets

| Secret | Description |
|--------|-------------|
| `gh-token` | Token used to check for existing releases and create the draft release. Defaults to `github.token`. |

## How it works

1. Pushing a tag such as `release-v3.0.15` triggers the workflow.
2. `check_release` fails if a non-draft release already exists for that
   version.
3. `build_wheels` builds the wheels, and `build_sdist` builds the source
   distribution with `uv build --sdist`.
4. `create_release` creates a draft GitHub release named after the version
   (`v3.0.15`) with all the artifacts. Tags starting with `prerelease-` are
   marked as prereleases.
5. Publish the release, then run wheelmonger to upload the artifacts to PyPI.

Compiled wheels use cibuildwheel's `build[uv]` frontend. The build
environment created by uv doesn't include pip, so `before-build` commands
must not call `pip`.
