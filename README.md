# OpenHound release workflows

Publish a GitHub Release for your collector, and GitHub builds and publishes its
Docker images for you. Publishing the Python package to PyPI is optional.

There are two repositories involved:

- **Your collector repository** (for example, `openhound-azure`) keeps your code,
  Dockerfile, and a small release workflow.
- **This repository** (`openhound-workflows`) provides the shared build steps.
  Those steps run against **your collector's code** at the release tag.

## 1. Make the shared workflows available

Commit and push this repository to GitHub. You will need the full commit SHA
(the 40-character commit ID) containing these workflows for the next step.
Your collector must have permission to use this repository's workflows. If this
repository is private, grant that access in its GitHub Actions settings.

## 2. Prepare your collector

Your collector needs:

- `pyproject.toml` and `uv.lock`, with a working `uv build`. The built package
  version must match the release tag.
- A Dockerfile with targets named `cli` and `enterprise` (the scheduler), using
  `COLLECTOR_VERSION` to set the installed collector version. Both targets must
  support AMD64 and ARM64 and run as a non-root user.
- A `.dockerignore` that keeps credentials and local data out of the build.

The container requirements and Docker version-cache advice are in the
[technical reference](RELEASE-REFERENCE.md#onboard-another-collector).

## 3. Add a release workflow to your collector

Save this as `.github/workflows/release.yml` **in your collector repository**.
Replace both `FULL_COMMIT_SHA` values with the shared workflow commit from step 1.
If the shared repository uses a different owner or name, replace those too.

```yaml
name: Publish release

on:
  release:
    types: [published]

permissions:
  contents: read

jobs:
  build-python:
    uses: SpecterOps/openhound-workflows/.github/workflows/build-python.yml@FULL_COMMIT_SHA
    with:
      tag: ${{ github.event.release.tag_name }}
      python-version: '3.13'

  publish-container:
    needs: build-python
    permissions:
      contents: read
      packages: write
    uses: SpecterOps/openhound-workflows/.github/workflows/publish-container.yml@FULL_COMMIT_SHA
    with:
      tag: ${{ github.event.release.tag_name }}
      prerelease: ${{ github.event.release.prerelease }}
```

This example **does not publish to PyPI**. It checks the Python package and then
publishes containers. To also publish the package, follow the
[optional PyPI setup](RELEASE-REFERENCE.md#optional-pypi-publication).

In GitHub settings, allow your collector's Actions to use these workflows and
write to its GHCR package. Choose whether the package should be public or private.
Disable any old workflows that also publish on merges or write the same image tags.

## 4. Publish a release

Commit the setup above before creating your release tag. In your collector's
GitHub repository, create and **publish a Release**, using a tag such as `v1.2.0`.
Pushing a tag alone does not start this workflow.

For `SpecterOps/openhound-azure`, that release produces:

```text
ghcr.io/specterops/openhound-azure:1.2.0
ghcr.io/specterops/openhound-azure:1.2.0-cli
```

The newest stable version also gets `latest` and `latest-cli`. For a prerelease,
use a tag such as `v1.3.0rc1` and mark the GitHub Release as a prerelease; it will
get versioned images without changing `latest`. Keep published stable releases
and tags so older release reruns cannot move `latest` backward.

To adopt future shared workflow changes, update both commit SHAs in your
collector's release workflow. They stay on the version you chose until you do.
