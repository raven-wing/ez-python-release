
# EZ Python Release

Automated semantic versioning and PyPI publishing for Python projects using Poetry. This GitHub Action handles the complete release workflow: determining version bumps, creating releases, building packages, and optionally publishing to PyPI.

## Features

- **Automated Semantic Versioning**: Uses [python-semantic-release](https://python-semantic-release.readthedocs.io/) to automatically determine version bumps based on commit messages
- **Poetry Integration**: Built-in support for Poetry dependency management and package building
- **GitHub Releases**: Automatically creates GitHub releases with changelog
- **PyPI Publishing**: Optional OIDC-based publishing to PyPI (no API tokens needed)
- **Configurable**: Flexible inputs for customizing Python version, Poetry version, build commands, and more
- **Artifact Upload**: Automatically uploads built distributions as GitHub artifacts

## Prerequisites

### GitHub App Setup

This action requires a GitHub App for authentication. This provides:
- Elevated permissions to bypass branch protection rules
- Better security than Personal Access Tokens
- Proper attribution for automated commits

**To create a GitHub App:**

1. Go to your GitHub Settings → Developer settings → GitHub Apps → New GitHub App
2. Configure the app:
   - **Name**: Choose a name (e.g., "My Release Bot")
   - **Homepage URL**: Your repository URL
   - **Webhook**: Uncheck "Active"
   - **Permissions**:
     - Repository permissions:
       - Contents: Read and write
       - Metadata: Read-only
3. Create the app and note the **App ID**
4. Generate a private key and download it
5. Install the app on your repository (Settings → GitHub Apps → Install)
6. Add secrets to your repository:
   - `RELEASER_APP_ID`: Your App ID
   - `RELEASER_PRIVATE_KEY`: Contents of the downloaded private key file

**Branch Protection Bypass:**

To allow the app to push to protected branches:
1. Go to repository Settings → Branches → Branch protection rules
2. Edit your main branch rule
3. Under "Allow specified actors to bypass required pull requests"
4. Add your GitHub App to the bypass list

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
    permissions:
      contents: write
    steps:
      - uses: raven-wing/ez-python-release@v1
        with:
          github_app_id: ${{ secrets.RELEASER_APP_ID }}
          github_app_private_key: ${{ secrets.RELEASER_PRIVATE_KEY }}
```

### With PyPI Publishing

To publish to PyPI, use a separate job with OIDC trusted publishing. This is more secure than including PyPI publishing in the same job as building.

**Setup Steps:**
1. Configure [Trusted Publishing](https://docs.pypi.org/trusted-publishers/) on PyPI
2. Add a `publish-to-pypi` job that runs after the release job

```yaml
name: Release and Publish

on:
  push:
    branches:
      - main

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
    permissions:
      contents: write
    steps:
      - uses: raven-wing/ez-python-release@v1
        with:
          github_app_id: ${{ secrets.RELEASER_APP_ID }}
          github_app_private_key: ${{ secrets.RELEASER_PRIVATE_KEY }}

  publish-to-pypi:
    name: Publish to PyPI
    needs: release
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # OIDC for PyPI trusted publishing
    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: python-package-distributions
          path: dist

      - name: Publish to PyPI
        uses: pypa/gh-action-pypi-publish@release/v1
```

### Advanced Configuration

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: raven-wing/ez-python-release@v1
        with:
          github_app_id: ${{ secrets.RELEASER_APP_ID }}
          github_app_private_key: ${{ secrets.RELEASER_PRIVATE_KEY }}
          python-version: '3.11'
          poetry-version: '1.7.1'
          git_committer_name: 'Release Bot'
          git_committer_email: 'bot@example.com'
          build_command: 'poetry build --format wheel'
          packages_dir: 'dist'
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `github_app_id` | GitHub App ID for generating authentication token | Yes | - |
| `github_app_private_key` | GitHub App private key for authentication | Yes | - |
| `python-version` | Python version to use for building the package | No | `3.10` |
| `poetry-version` | Poetry version to install (empty = latest) | No | `''` |
| `git_committer_name` | Name for git commits made by semantic-release | No | `semantic-release` |
| `git_committer_email` | Email for git commits made by semantic-release | No | `semantic-release@users.noreply.github.com` |
| `build_command` | Command to build the package | No | `poetry build` |
| `packages_dir` | Directory containing distribution packages | No | `dist` |
| `root_options` | Additional root options for python-semantic-release | No | `''` |

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
    permissions:
      contents: write
    outputs:
      released: ${{ steps.release.outputs.released }}
      version: ${{ steps.release.outputs.version }}
    steps:
      - uses: raven-wing/ez-python-release@v1
        id: release
        with:
          github_app_id: ${{ secrets.RELEASER_APP_ID }}
          github_app_private_key: ${{ secrets.RELEASER_PRIVATE_KEY }}

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

The release job requires the following permission:

```yaml
permissions:
  contents: write    # For creating releases and pushing version bumps
```

If you're publishing to PyPI in a separate job, that job needs:

```yaml
permissions:
  id-token: write    # For PyPI OIDC trusted publishing
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
    permissions:
      contents: write
    concurrency:
      group: release-${{ github.ref }}
      cancel-in-progress: false
    steps:
      - uses: raven-wing/ez-python-release@v1
        with:
          github_app_id: ${{ secrets.RELEASER_APP_ID }}
          github_app_private_key: ${{ secrets.RELEASER_PRIVATE_KEY }}
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
