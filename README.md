# Create Semantic Release

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Create%20Semantic%20Release-blue.svg?colorA=24292e&colorB=0366d6&style=flat&longCache=true&logo=github)](https://github.com/marketplace/actions/create-semantic-release)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

A GitHub Action to automatically create semantic versioned releases with auto-generated release notes.

## Features

- **Semantic Versioning**: Automatic version bumping following semver principles
- **Auto-Generated Release Notes**: GitHub automatically generates release notes from commits and PRs
- **Flexible Prefixing**: Support for custom tag prefixes (v, app-, service-, etc.)
- **Multiple Bump Types**: Support for major, minor, patch, and pre-release versions
- **Zero Configuration**: Works out of the box with sensible defaults
- **Release Outputs**: Returns tag, version, and release URL for downstream jobs

## How It Works

This action uses semantic versioning to automatically determine the next version number based on your current tags and the specified bump type:

- **major**: Increments the major version (1.0.0 → 2.0.0) - Breaking changes
- **minor**: Increments the minor version (1.0.0 → 1.1.0) - New features
- **patch**: Increments the patch version (1.0.0 → 1.0.1) - Bug fixes
- **premajor/preminor/prepatch**: Creates pre-release versions (1.0.0 → 2.0.0-0)
- **prerelease**: Increments pre-release version (1.0.0-0 → 1.0.0-1)

## Prerequisites

### GitHub Token Permissions

Your workflow must have these permissions:

```yaml
permissions:
  contents: write  # Required to create releases and tags
```

### Git Configuration

This action requires the full git history with all tags:

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Important: fetch all history and tags
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `bump-type` | Version bump type: `major`, `minor`, `patch`, `premajor`, `preminor`, `prepatch`, `prerelease` | Yes | - |
| `prefix` | Tag prefix (e.g., `v`, `app-v`, `service-`) | Yes | - |
| `github-token` | GitHub token (use `secrets.GITHUB_TOKEN`) | Yes | - |

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `release-tag` | Created release tag (with prefix) | `v1.2.3` |
| `release-version` | Version number without prefix | `1.2.3` |
| `release-url` | URL of the created release | `https://github.com/owner/repo/releases/tag/v1.2.3` |

## Usage

### Basic Example

```yaml
name: Create Release

on:
  workflow_dispatch:
    inputs:
      bump:
        description: 'Version bump type'
        required: true
        type: choice
        options:
          - patch
          - minor
          - major

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Important: fetch all tags

      - name: Create release
        uses: kubescript/create-release@v1
        with:
          bump-type: ${{ inputs.bump }}
          prefix: v
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Automatic Release on PR Merge

```yaml
name: Auto Release

on:
  pull_request:
    types: [closed]
    branches: [main]

permissions:
  contents: write

jobs:
  release:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Determine bump type from labels
        id: bump
        run: |
          if [[ "${{ contains(github.event.pull_request.labels.*.name, 'major') }}" == "true" ]]; then
            echo "type=major" >> $GITHUB_OUTPUT
          elif [[ "${{ contains(github.event.pull_request.labels.*.name, 'minor') }}" == "true" ]]; then
            echo "type=minor" >> $GITHUB_OUTPUT
          else
            echo "type=patch" >> $GITHUB_OUTPUT
          fi

      - name: Create release
        uses: kubescript/create-release@v1
        with:
          bump-type: ${{ steps.bump.outputs.type }}
          prefix: v
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Monorepo with Multiple Services

```yaml
name: Release Service

on:
  workflow_dispatch:
    inputs:
      service:
        description: 'Service to release'
        required: true
        type: choice
        options:
          - api
          - web
          - worker
      bump:
        description: 'Version bump type'
        required: true
        type: choice
        options:
          - patch
          - minor
          - major

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Create release
        uses: kubescript/create-release@v1
        with:
          bump-type: ${{ inputs.bump }}
          prefix: ${{ inputs.service }}-v
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Using Outputs for Downstream Jobs

```yaml
name: Release and Deploy

on:
  workflow_dispatch:
    inputs:
      bump:
        type: choice
        options: [patch, minor, major]
        required: true

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.create-release.outputs.release-tag }}
      version: ${{ steps.create-release.outputs.release-version }}
      url: ${{ steps.create-release.outputs.release-url }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Create release
        id: create-release
        uses: kubescript/create-release@v1
        with:
          bump-type: ${{ inputs.bump }}
          prefix: v
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Print release info
        run: |
          echo "Created release: ${{ steps.create-release.outputs.release-tag }}"
          echo "Version: ${{ steps.create-release.outputs.release-version }}"
          echo "URL: ${{ steps.create-release.outputs.release-url }}"

  deploy:
    needs: release
    runs-on: ubuntu-latest
    steps:
      - name: Deploy version
        run: |
          echo "Deploying version: ${{ needs.release.outputs.version }}"
          echo "From release: ${{ needs.release.outputs.url }}"
```

### Pre-release Versions

```yaml
name: Create Pre-release

on:
  push:
    branches: [develop]

permissions:
  contents: write

jobs:
  prerelease:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - name: Create pre-release
        uses: kubescript/create-release@v1
        with:
          bump-type: prerelease
          prefix: v
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

## Understanding Semantic Versioning

Semantic versioning follows the format: **MAJOR.MINOR.PATCH**

### Version Components

- **MAJOR**: Incremented for breaking changes (incompatible API changes)
- **MINOR**: Incremented for new features (backward-compatible)
- **PATCH**: Incremented for bug fixes (backward-compatible)

### Examples

Starting from version `v1.2.3`:

| Bump Type | New Version | Use Case |
|-----------|-------------|----------|
| `patch` | `v1.2.4` | Bug fixes, documentation updates |
| `minor` | `v1.3.0` | New features, enhancements |
| `major` | `v2.0.0` | Breaking changes, API redesign |
| `prepatch` | `v1.2.4-0` | Pre-release for patch version |
| `preminor` | `v1.3.0-0` | Pre-release for minor version |
| `premajor` | `v2.0.0-0` | Pre-release for major version |
| `prerelease` | `v1.2.4-1` | Increment existing pre-release |

## Troubleshooting

### Error: "Resource not accessible by integration"

**Problem**: GitHub token lacks required permissions.

**Solution**: Add `contents: write` permission to your workflow:
```yaml
permissions:
  contents: write
```

### Error: "No tags found" or incorrect version

**Problem**: Git history doesn't include tags.

**Solution**: Ensure you fetch all history and tags:
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Fetch all history and tags
```

### Release notes are empty or incomplete

**Problem**: Not enough commit history or no merged PRs.

**Solution**:
- Ensure `fetch-depth: 0` is set
- GitHub generates notes from PRs and commits since the last release
- For first release, notes will be based on all commits

### Duplicate tag error

**Problem**: Tag already exists.

**Solution**: This action creates a new version. If you want to re-create a release for an existing tag, delete the tag first or use a different bump type.

### Wrong version prefix

**Problem**: Tags have inconsistent prefixes.

**Solution**: Ensure all your tags use the same prefix. The action filters tags by the specified prefix to determine the next version.

## Best Practices

1. **Use PR Labels**: Label PRs with `major`, `minor`, or `patch` to automate bump type selection
2. **Conventional Commits**: Use conventional commit messages for better release notes
3. **Protected Branches**: Configure branch protection to require PR reviews before merging
4. **Automated Releases**: Trigger releases automatically on PR merge to main branch
5. **Changelog**: GitHub auto-generates release notes, but consider maintaining a CHANGELOG.md file

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

## Support

- Report issues: [GitHub Issues](https://github.com/kubescript/create-release/issues)
- Documentation: [GitHub Marketplace](https://github.com/marketplace/actions/create-semantic-release)

---

Made with ❤️ by [KubeScript](https://github.com/kubescript)
