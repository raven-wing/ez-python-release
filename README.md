
# EZ Python Release

Automated semantic versioning and PyPI publishing for Python projects using Poetry. This GitHub Action handles the complete release workflow: determining version bumps, creating releases, building packages, and optionally publishing to PyPI.

## Features

- **Automated Semantic Versioning**: Uses [python-semantic-release](https://python-semantic-release.readthedocs.io/) to automatically determine version bumps based on commit messages
- **Poetry Integration**: Built-in support for Poetry dependency management and package building
- **GitHub Releases**: Automatically creates GitHub releases with changelog
- **PyPI Publishing**: Optional OIDC-based publishing to PyPI (no API tokens needed)
- **Configurable**: Flexible inputs for customizing Python version, Poetry version, build commands, and more
- **Artifact Upload**: Automatically uploads built distributions as GitHub artifacts

## Usage

### Basic Example

```yaml
name: Release

on:
  push:
    branches:
      - main

permissions:
  contents: write
  id-token: write

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.10'
      - name: Install dependencies
        run: |
          pip install poetry
          poetry install
      - name: Run tests
        run: poetry run pytest

  release:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: raven-wing/ez-python-release@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

### With PyPI Publishing

To publish to PyPI, you need to:
1. Configure [Trusted Publishing](https://docs.pypi.org/trusted-publishers/) on PyPI
2. Set `publish_to_pypi: 'true'` in the action
3. Ensure `id-token: write` permission is set

```yaml
name: Release and Publish

on:
  push:
    branches:
      - main

permissions:
  contents: write
  id-token: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: raven-wing/ez-python-release@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_to_pypi: 'true'
```

### With GitHub App Token

For better security and to avoid rate limits, use a GitHub App token:

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Generate GitHub App token
        id: generate_token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      - uses: raven-wing/ez-python-release@v1
        with:
          github_token: ${{ steps.generate_token.outputs.token }}
```

### Advanced Configuration

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: raven-wing/ez-python-release@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          python-version: '3.11'
          poetry-version: '1.7.1'
          git_committer_name: 'Release Bot'
          git_committer_email: 'bot@example.com'
          build_command: 'poetry build --format wheel'
          publish_to_pypi: 'true'
          pypi_packages_dir: 'dist'
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `github_token` | GitHub token for creating releases and pushing changes | Yes | - |
| `python-version` | Python version to use for building the package | No | `3.10` |
| `poetry-version` | Poetry version to install (empty = latest) | No | `''` |
| `git_committer_name` | Name for git commits made by semantic-release | No | `semantic-release` |
| `git_committer_email` | Email for git commits made by semantic-release | No | `semantic-release@users.noreply.github.com` |
| `build_command` | Command to build the package | No | `poetry build` |
| `root_options` | Additional root options for python-semantic-release | No | `''` |
| `publish_to_pypi` | Whether to publish to PyPI (requires OIDC) | No | `false` |
| `pypi_packages_dir` | Directory containing distribution packages | No | `dist` |

## Outputs

| Output | Description |
|--------|-------------|
| `released` | Whether a release was created (`true`/`false`) |
| `tag` | The git tag of the release |
| `version` | The version that was released |

### Using Outputs

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    outputs:
      released: ${{ steps.release.outputs.released }}
      version: ${{ steps.release.outputs.version }}
    steps:
      - uses: raven-wing/ez-python-release@v1
        id: release
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}

  notify:
    runs-on: ubuntu-latest
    needs: release
    if: needs.release.outputs.released == 'true'
    steps:
      - name: Send notification
        run: echo "Released version ${{ needs.release.outputs.version }}"
```

## Requirements

### Project Requirements

Your Python project must:
1. Use [Poetry](https://python-poetry.org/) for dependency management
2. Have a `pyproject.toml` file configured for python-semantic-release
3. Follow [Conventional Commits](https://www.conventionalcommits.org/) for commit messages

### Minimal pyproject.toml Configuration

```toml
[tool.poetry]
name = "your-package"
version = "0.1.0"
description = "Your package description"

[tool.semantic_release]
version_toml = ["pyproject.toml:tool.poetry.version"]
build_command = "pip install poetry && poetry build"
```

### Commit Message Format

This action uses semantic versioning based on commit messages:

- `fix:` commits trigger a patch release (0.0.X)
- `feat:` commits trigger a minor release (0.X.0)
- `BREAKING CHANGE:` in commit body triggers a major release (X.0.0)

Example:
```
feat: add new authentication method

This adds OAuth2 support for user authentication.
```

## Permissions

The action requires specific GitHub permissions:

```yaml
permissions:
  contents: write    # For creating releases and pushing version bumps
  id-token: write    # For PyPI OIDC publishing (if enabled)
```

## PyPI Trusted Publishing Setup

To use PyPI publishing with OIDC (no API tokens needed):

1. Go to your PyPI project settings
2. Navigate to "Publishing" → "Add a new publisher"
3. Configure the publisher:
   - **PyPI Project Name**: Your package name
   - **Owner**: Your GitHub username/organization
   - **Repository name**: Your repository name
   - **Workflow name**: Your workflow filename (e.g., `release.yml`)
   - **Environment name**: Leave empty (or set if using GitHub environments)

## Concurrency Control

To prevent multiple releases from running simultaneously:

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    concurrency:
      group: release-${{ github.ref }}
      cancel-in-progress: false
    steps:
      - uses: raven-wing/ez-python-release@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

## Troubleshooting

### No release is created

- Ensure your commits follow [Conventional Commits](https://www.conventionalcommits.org/) format
- Check that you're pushing to the branch configured in python-semantic-release
- Verify that `pyproject.toml` has proper semantic-release configuration

### PyPI publishing fails

- Verify that Trusted Publishing is configured correctly on PyPI
- Ensure `id-token: write` permission is set
- Check that the workflow name and repository match the PyPI publisher configuration

### Poetry installation fails

- Try pinning a specific Poetry version using `poetry-version` input
- Check that your Python version is compatible with Poetry

## Examples

See the [examples directory](examples/) for complete workflow examples.

## License

[MIT License](LICENSE)

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

## Related Projects

- [python-semantic-release](https://github.com/python-semantic-release/python-semantic-release)
- [Poetry](https://python-poetry.org/)
- [PyPI Trusted Publishing](https://docs.pypi.org/trusted-publishers/)
