# CI Artifact Publishing — Real-World Live Example

Internal guide reference: how a mature, actively maintained open-source project publishes
its CI build artifacts using GitHub's official artifact upload action (`actions/upload-artifact`).

## Repository Selected

- **Repository:** [astral-sh/uv](https://github.com/astral-sh/uv) — "An extremely fast Python package and project manager, written in Rust" (~90k stars, pushed continuously; very actively maintained, not archived, not a fork)
- **How it was picked:** from a live GitHub search for public workflows that use `actions/upload-artifact`, then verifying that the project's own CI genuinely calls the action and that its most recent completed runs still carry live, downloadable artifacts.
- **Workflow file that does the uploading (path inside the repo):** `.github/workflows/build-dev-binaries.yml`
  - This is a reusable `workflow_call` workflow. It is invoked by the top-level CI entry point `.github/workflows/ci.yml` (the `build-dev-binaries` job), which is the file the run itself is attributed to.

## Most Recent Completed Run with Live Artifacts

- **Run ID:** `39417317936` (CI run number 51936, workflow "CI", `.github/workflows/ci.yml`)
- **Trigger:** `pull_request` — PR #22758 "Recheck compatibility before reusing an installed wheel variant" (head branch `konsti/recheck-installed-wheel-variants`, head SHA `8f7234378201d250cd5880b6a0cccf75e647397b`)
- **Status / conclusion:** completed / success (2026-10-20)
- **Run URL:** https://github.com/astral-sh/uv/actions/runs/39417317936
- **Artifacts attached:** 15 total, all with `expired: false` (platform binaries for linux libc/musl/aarch64/armv7, macOS and Windows, plus junit test results and benchmark profiles)

## Artifact Details (exactly as returned by the GitHub API)

- **Artifact name:** `uv-linux-libc-1a506b35e192e5d8687c92320ac82afeec832710`
- **Artifact ID:** `12262405126`
- **Size:** 41,747,500 bytes (~41.7 MB)
- **Digest (sha256):** `f1c330467245448e142680c06ba4fc7c0e1b9492791f5fbe7e053d5e813fad00`
- **Created:** 2026-10-20T11:48:39Z — **Expires:** 2026-10-21T11:48:33Z (the workflow sets `retention-days: 1`; verified live and downloadable at time of research)
- **Download URL (the API's `archive_download_url` field, verbatim):**

  ```
  https://api.github.com/repos/astral-sh/uv/actions/artifacts/12262405126/zip
  ```

  Verified live: fetching this canonical endpoint returns a short-lived, pre-signed Azure
  Blob Storage URL that serves the artifact ZIP, so the canonical API URL above is the one to
  record — the signed blob URL is ephemeral and expires within minutes.

## How This Workflow Uploads Artifacts

The top-level CI workflow `.github/workflows/ci.yml` is triggered by `push` to `main`, by
every `pull_request`, and by manual `workflow_dispatch`. A `plan` job first inspects what
changed, and when code changes are detected the `build-dev-binaries` job calls the reusable
`.github/workflows/build-dev-binaries.yml`, which contains one build job per target platform
(`build-binary-linux-libc`, `build-binary-linux-aarch64`, `build-binary-linux-musl`, `build-binary-linux-armv7-gnueabihf`, and macOS/Windows variants). Each job checks out the
source, installs the Rust toolchain (with `Swatinem/rust-cache` for incremental caching), and
compiles the binaries with `cargo build --locked --profile no-debug-nightly -Z checksum-freshness`.
The artifact comes straight out of that build: the final "Upload binary" step invokes GitHub's
official `actions/upload-artifact@v7.0.1` (pinned to full commit SHA
`043fb46d1a93c77aae656e7c1c64a875d1fc6a0a`) with `name: uv-linux-libc-${{ github.sha }}`,
which zips the freshly built `./target/no-debug-nightly/uv` and `./target/no-debug-nightly/uvx`
binaries and attaches them to the workflow run with `retention-days: 1`. Embedding `github.sha`
in the name (on this pull-request run it is the merge commit `1a506b35…`, which is why the
artifact suffix differs from the PR head SHA) lets the downstream smoke, integration, system and
ecosystem test jobs re-download, via `actions/download-artifact`, exactly the binaries produced
for the commit under test — a textbook pattern of artifacts as the hand-off between build and
test stages.
