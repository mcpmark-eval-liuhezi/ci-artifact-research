# CI Artifact Publishing — Real-World Example

An internal reference on how a mature, actively maintained open-source project publishes
CI build artifacts using GitHub's official artifact upload action (`actions/upload-artifact`).

## Repository Selected

- **Repository:** [maurosoria/dirsearch](https://github.com/maurosoria/dirsearch) — "Web path scanner", a well-known, actively maintained open-source security tool.
- **Workflow file path:** `.github/workflows/docker-image.yml`
- **Workflow name:** "Docker Images"
- **Workflow file (source):** https://github.com/maurosoria/dirsearch/blob/master/.github/workflows/docker-image.yml

## Most Recent Completed Run with Live Artifacts

- **Run ID:** `39362269311` (workflow run number 979)
- **Workflow:** Docker Images (`.github/workflows/docker-image.yml`)
- **Trigger:** `pull_request` — PR #1761, "Accelerate native static wildcard filtering"
- **Head SHA:** `5b352432171eac64a68212eb6a61a5cd3a9946da`
- **Status / conclusion:** completed / success (2026-10-20)
- **Run URL:** https://github.com/maurosoria/dirsearch/actions/runs/39362269311

## Artifact Details (exactly as returned by the GitHub API)

- **Artifact name:** `dirsearch-v0.5.0-rc1-container-threaded`
- **Artifact ID:** `12237818154`
- **Size:** 221 bytes
- **Digest (sha256):** `7db88c46afd8ba293112e45db28cbf8c82ecfdb8ba36216099f419b92f61f3e4`
- **Created:** 2026-10-20T03:03:55Z — **Expires:** 2026-11-19T03:03:54Z (not expired, verified live and downloadable)
- **Download URL (as returned by the API, `archive_download_url` field):**

  ```
  https://api.github.com/repos/maurosoria/dirsearch/actions/artifacts/12237818154/zip
  ```

  Note: this is the canonical endpoint the API returns. When fetched, GitHub responds with a
  short-lived, pre-signed Azure Blob Storage redirect that expires within minutes, so the
  canonical `archive_download_url` above is the URL to record — the redirected blob URL is
  ephemeral and should not be persisted.

## How This Workflow Uploads Artifacts

The `docker-image.yml` workflow runs on three triggers: `push` to `master`, `pull_request`
against `master`, and manual/automated `workflow_dispatch`/`workflow_call` invocations that
accept a `release_tag` input. Its single `build` job uses a matrix of three build stacks
(`threaded`, `async`, `native-rust`), checks out the repo, and builds a Linux x64 Docker image
per stack using `docker compose` piped into `docker buildx bake`. The freshly built image is
then smoke-tested by running `--version`, `--help`, and a small real scan against
`https://example.com`. After that, a "Write image metadata" step simply echoes the fully
qualified image tag (e.g. `ghcr.io/maurosoria/dirsearch:v0.5.0-rc1-threaded`) into a small
text file, `container-image-<stack>.txt`, inside the runner workspace — this file is the
artifact's content, not the image itself (the image is optionally pushed to GHCR only when
the `push` input is true). Finally, the "Upload image metadata" step invokes GitHub's official
`actions/upload-artifact@v4` action, which zips that file and attaches it to the workflow run
under the name `dirsearch-<release-tag>-container-<stack>` with a 30-day retention period.
Because the job is matrixed, each run produces one artifact per stack (three in total), which
is why run 39362269311 has three live artifacts attached.
