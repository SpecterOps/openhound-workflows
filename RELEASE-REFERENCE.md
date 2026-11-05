# Release workflow reference

Reusable workflows for collector releases: build Python distributions, upload to
PyPI from the collector, then test and publish CLI and scheduler images to GHCR.
Only a published GitHub Release starts the collector workflow. Setup and local
validation do not publish anything.

`build-python.yml` checks out the **caller** at `refs/tags/<tag>` with full history,
runs `uv build`, verifies wheel and sdist metadata with PEP 440 version comparison,
and uploads `python-distributions`. Inputs: required `tag`, optional
`python-version` (default `3.13`). Permission: `contents: read`.

`publish-container.yml` checks out the caller's tag, builds and smoke-tests both
`cli` and `enterprise`, then publishes AMD64/ARM64 images with OCI source/version
labels and SBOMs. Inputs: required `tag`, optional `prerelease` (default `false`)
and `version-build-arg` (default `COLLECTOR_VERSION`). Permissions:
`contents: read`, `packages: write`. `GITHUB_TOKEN` is available automatically;
`secrets: inherit` is unnecessary.

| Target | Version tag | Stable alias |
| --- | --- | --- |
| enterprise (scheduler) | `ghcr.io/<lowercase-owner>/<lowercase-repo>:<version>` | `latest` |
| cli | `ghcr.io/<lowercase-owner>/<lowercase-repo>:<version>-cli` | `latest-cli` |

Release tags use `[v]MAJOR.MINOR.PATCH`, optionally followed by `aN`, `bN`, `rcN`,
`-alpha.N`, `-beta.N`, or `-rc.N`. Examples: `v1.2.3`, `1.2.3rc1`,
`v1.2.3-rc.1`. A suffix requires the GitHub Release prerelease flag. Python
normalizes `1.2.3-rc.1` to `1.2.3rc1`; image tags preserve the spelling after
removing the leading `v`. Development, local and post versions are unsupported.

Rolling aliases update only after **both** targets succeed, only for stable
releases, and only when the current release has the highest published stable
semantic version (not publication date). Container publication is serialized per
collector, including reruns. `queue: max` retains up to 100 pending runs (see
[GitHub concurrency](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency)).
Keep published stable release records and tags;
deleting them defeats the historical version comparison. Do not run a competing
workflow that writes the same aliases. The two alias writes are not atomic;
rerunning repairs a partial promotion. Prereleases get version tags only.

## Optional PyPI publication

In your collector's `release.yml`, add this job after `build-python`:

```yaml
  publish-pypi:
    needs: build-python
    runs-on: ubuntu-latest
    environment: pypi
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: python-distributions
          path: dist/
      - uses: astral-sh/setup-uv@v7
      - run: uv publish --trusted-publishing always --check-url https://pypi.org/simple/ dist/*
```

Change `publish-container` to `needs: publish-pypi` so images publish after the
package upload succeeds. Configure a GitHub environment named `pypi` in the
collector repository and a PyPI Trusted Publisher for that repository,
`release.yml`, and the `pypi` environment.

PyPI documents a [reusable-workflow Trusted Publisher limitation](https://docs.pypi.org/trusted-publishers/troubleshooting/#reusable-workflows-on-github),
so upload runs directly in the collector's workflow. The check URL lets uv skip
identical files already published. After a partial failure, prefer **Re-run failed
jobs**; rebuilding distributions on a full rerun can produce different bytes.

## Onboard another collector

Read that repository's agent instructions. Start with the release workflow in
[README.md](README.md) and adapt its Python version and version build argument
only as needed. Keep its source,
Dockerfile, lockfile, dependencies, and tests in that collector. Delete or adapt
old publishers and push/main guards so only Release `published` events publish.
Commit all required files before creating the release tag; the release checkout
must contain its smoke script and compatible Dockerfile.

Verify this contract first:

| Area | Required compatibility | Okta pilot |
| --- | --- | --- |
| Build | `uv build` on the requested Python; matching wheel/sdist release version | Python 3.13; hatch-vcs reads Git tags |
| Version source | Check actual backend; static versions must be committed to match the tag, or implement backend-specific injection in the collector before adoption | No Python build injection needed |
| Container | Targets `cli` and `enterprise`; release version passed through the named build argument | `COLLECTOR_VERSION` feeds `SETUPTOOLS_SCM_PRETEND_VERSION`; force collector reinstall to avoid a stale cached wheel |
| Plugin | Installed `openhound.sources` entry point loads the collector | `okta` loads `openhound_okta.main:app` |
| Metadata | Required extension metadata is included in the wheel and runtime | `openhound_okta/extension.yaml` is force-included |
| Runtime | Non-root UID and GID; CLI entrypoint accepts `--help` | UID/GID 65532; distroless Python 3.13 |
| Scheduler | Initialization tested with a mocked service, no real credentials or collection | `/app/src/scheduler.py`, mocked `Service.start` |
| Architectures | Resolve/build dependencies on both AMD64 and ARM64 | `uv.lock` and base images must work on both |
| Context | Exclude runtime credentials, local environments and collection data | `.dockerignore` allowlists build metadata and `src/`; excludes key/certificate files |

Do not place runtime credentials under an allowlisted source directory. Inject
credentials only at runtime. Do not assume all collectors use VCS versioning or
Okta's imports, plugin name, scheduler configuration, or UID.

Each collector supplies `.github/scripts/container-smoke.py`. The shared
workflow feeds it to `python -` **inside each locally built image**, with network
disabled, working directory from the Dockerfile, and environment variables
`EXPECTED_VERSION` (release tag without `v`) and `CONTAINER_TARGET` (`cli` or
`enterprise`). Exit nonzero on failure. Do not require the script in the image or
widen `.dockerignore` for it. The script must import the collector, compare its
installed distribution version using Python normalization, load its entry point,
check packaged extension metadata and expected UID/GID, and initialize its
scheduler with a mocked service. The shared workflow separately checks nonzero
UID/GID and CLI `--help`. The pilot script uses `packaging`, already a runtime
dependency through OpenHound; verify this when adapting it.

## External configuration

Before enabling a caller, an administrator must configure:

- Shared repository visibility: public for public callers, or compatible private
  visibility with Actions reusable-workflow access explicitly granted to callers.
- Caller/organization Actions policies that allow this workflow repository and
  all referenced actions. See [GitHub reusable workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows).
- GHCR package access: allow the collector's Actions token to write the package;
  link existing packages to the collector if necessary and choose package
  visibility explicitly. Source labels identify the collector, not this repo.
- Collector-local GitHub environment named `pypi`, including any protection rules.
- PyPI Trusted Publisher (or pending publisher for a new project) for the
  **collector** owner/repository, workflow filename `release.yml`, environment
  `pypi`, and correct distribution name (`openhound-okta` for the pilot).
- Caller job permissions matching the callee. Reusable workflows cannot elevate
  the caller's token. Grant `id-token: write` only to the direct PyPI upload job.

No remote settings, releases, tags, packages or images are created by setup.
Local validation does not verify hosted publication or remote permissions.
