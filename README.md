# explosion/gha-cibuildwheel

Reusable GitHub Actions workflows for building and publishing Python wheels across Explosion AI projects.

## Overview

This repository provides two reusable workflows:
- **`cibuildwheel.yml`**: Builds Python wheels and source distributions, creates GitHub releases
- **`publish_pypi.yml`**: Publishes releases to PyPI

The workflows support both:
- **C-extension packages**: Multi-platform wheel building using cibuildwheel
- **Pure Python packages**: Universal wheel building without platform-specific compilation

## Usage

### Basic Setup for C-extension Packages

For packages with C extensions (like spaCy, thinc, cymem, etc.):

#### `.github/workflows/cibuildwheel.yml`
```yaml
name: Build

on:
  push:
    tags:
      - 'release-v[0-9]+.[0-9]+.[0-9]+**'
      - 'prerelease-v[0-9]+.[0-9]+.[0-9]+**'

jobs:
  build:
    uses: explosion/gha-cibuildwheel/cibuildwheel.yml@main
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

#### `.github/workflows/publish_pypi.yml`
```yaml
name: Publish to PyPI

on:
  release:
    types:
      - published

jobs:
  publish:
    uses: explosion/gha-cibuildwheel/publish_pypi.yml@main
    with:
      pypi-package-name: 'your-package-name'
```

### Setup for Pure Python Packages

For pure Python packages (like confection, wasabi, catalogue):

```yaml
name: Build

on:
  push:
    tags:
      - 'release-v[0-9]+.[0-9]+.[0-9]+**'
      - 'prerelease-v[0-9]+.[0-9]+.[0-9]+**'

jobs:
  build:
    uses: explosion/gha-cibuildwheel/cibuildwheel.yml@main
    with:
      pure-python: true
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Workflow: cibuildwheel.yml

Handles wheel building and GitHub release creation.

### Inputs

| Input | Description | Default | Required |
|-------|-------------|---------|----------|
| `pure-python` | Whether this is a pure Python package | `false` | No |
| `package-dir` | Directory containing the package to build | `.` | No |
| `output-dir` | Output directory for built wheels | `wheelhouse` | No |
| `config-file` | Path to cibuildwheel config file | `{package}/pyproject.toml` | No |
| `cibw-archs-linux` | Architectures to build on Linux | `auto` | No |
| `cibw-env-vars` | JSON object of environment variables for cibuildwheel | `{}` | No |
| `build-sdist` | Whether to build source distribution | `true` | No |
| `create-release` | Whether to create a GitHub release | `true` | No |
| `release-name-prefix` | Prefix to remove from tag for release name | `release-` | No |
| `prerelease-name-prefix` | Prefix for prerelease tags | `prerelease-` | No |
| `os-matrix` | JSON array of OS runners to build on | See below | No |
| `python-version` | Python version for sdist build | `3.11` | No |
| `cibuildwheel-version` | Version of cibuildwheel to use | `v2.21.3` | No |

**Default OS Matrix:**
```json
["ubuntu-latest", "windows-latest", "macos-13", "macos-14"]
```

### Secrets

| Secret | Description | Required |
|--------|-------------|----------|
| `GITHUB_TOKEN` | GitHub token for creating releases | No (uses default if not provided) |

### Advanced Examples

#### Custom OS Matrix
```yaml
jobs:
  build:
    uses: explosion/gha-cibuildwheel/cibuildwheel.yml@main
    with:
      os-matrix: '["ubuntu-latest", "windows-latest", "macos-13", "macos-14", "ubuntu-24.04-arm"]'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

#### Custom Environment Variables
```yaml
jobs:
  build:
    uses: explosion/gha-cibuildwheel/cibuildwheel.yml@main
    with:
      cibw-env-vars: '{"CIBW_SOME_OPTION": "value", "CIBW_BUILD": "cp38-*"}'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

#### Skip Release Creation
```yaml
jobs:
  build:
    uses: explosion/gha-cibuildwheel/cibuildwheel.yml@main
    with:
      create-release: false
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Workflow: publish_pypi.yml

Handles PyPI publishing when a GitHub release is published.

### Inputs

| Input | Description | Default | Required |
|-------|-------------|---------|----------|
| `pypi-package-name` | PyPI project name (for environment URL) | - | **Yes** |

### Example

```yaml
name: Publish to PyPI

on:
  release:
    types:
      - published

jobs:
  publish:
    uses: explosion/gha-cibuildwheel/publish_pypi.yml@main
    with:
      pypi-package-name: 'spacy'
```

## How It Works

### C-extension Packages
1. **Push tag** matching pattern (e.g., `release-v3.7.0`)
2. **cibuildwheel.yml** triggers:
   - Builds wheels for all platforms using `cibuildwheel`
   - Builds source distribution
   - Creates draft GitHub release with artifacts
3. **Review and publish** the GitHub release
4. **publish_pypi.yml** triggers:
   - Downloads artifacts from release
   - Publishes to PyPI using trusted publishing (OIDC)

### Pure Python Packages
1. **Push tag** matching pattern
2. **cibuildwheel.yml** triggers:
   - Builds single universal wheel on Ubuntu
   - Builds source distribution
   - Creates draft GitHub release with artifacts
3. **Review and publish** the GitHub release
4. **publish_pypi.yml** triggers (same as above)

## Requirements

### Repository Setup
- **pyproject.toml** with build configuration
- **Trusted publishing** configured on PyPI (recommended)
- **GitHub environment** named "pypi" with deployment protection rules (optional)

### For C-extension Packages
- **cibuildwheel configuration** in pyproject.toml:
  ```toml
  [tool.cibuildwheel]
  build = "cp38-* cp39-* cp310-* cp311-* cp312-*"
  skip = "*-musllinux_*"
  ```

## Migration Guide

### From Individual Workflows

1. Replace your `.github/workflows/cibuildwheel.yml` with the minimal version above
2. Replace your `.github/workflows/publish_pypi.yml` with the minimal version above
3. Add `pypi-package-name` parameter with your actual PyPI package name
4. For pure Python packages, add `pure-python: true`

### Version Pinning

For production use, pin to a specific version instead of using `@main`:

```yaml
uses: explosion/gha-cibuildwheel/cibuildwheel.yml@v1.0.0
```

## Supported Projects

Currently used by:
- **spaCy** - Industrial-strength NLP (C-extension)
- **thinc** - Functional deep learning (C-extension)
- **cymem** - Memory management helpers (C-extension)
- **murmurhash** - Cython bindings for MurmurHash (C-extension)
- **preshed** - Cython hash tables (C-extension)
- **srsly** - Serialization utilities (C-extension)
- **confection** - Configuration system (Pure Python)

## Troubleshooting

### Wheels not building for certain platforms
Check your `os-matrix` input includes all required platforms.

### Environment variables not being set
Ensure `cibw-env-vars` is valid JSON. Test with:
```bash
echo '{"CIBW_SOME_OPTION": "value"}' | jq .
```

### Release not being created
Verify:
- Tag matches the pattern (e.g., `release-v1.0.0`)
- `GITHUB_TOKEN` has appropriate permissions
- `create-release` is not set to `false`

### PyPI upload fails
Check:
- Trusted publishing is configured correctly
- `pypi-package-name` matches your actual PyPI project
- GitHub environment "pypi" exists (if using environment protection)

## License

MIT