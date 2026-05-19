# Upstream Build Workflow

This document explains how the automated Docker image build pipeline works for
`paludi/ha-plugins-sip-gateway`.

## Overview

The workflow defined in [`.github/workflows/build-upstream.yml`](../.github/workflows/build-upstream.yml)
monitors the public upstream repository [`arnonym/ha-plugins`](https://github.com/arnonym/ha-plugins),
builds the `ha-sip` Docker image from it, and publishes the result to the
owner's private GitHub Container Registry (GHCR) namespace.

```
arnonym/ha-plugins (default branch)
        │
        │  poll for new commits
        ▼
  paludi/ha-plugins-sip-gateway  (this repo)
  ├── .github/workflows/build-upstream.yml  ← CI pipeline
  ├── .github/build-state.json              ← persisted state
  └── docs/upstream-build.md               ← this file
        │
        │  docker build + push
        ▼
  ghcr.io/paludi/ha-sip:<version>
```

---

## Triggers

| Trigger | When |
|---------|------|
| **Schedule (daily)** | Every day at **06:00 UTC** — polls for upstream changes. |
| **Schedule (weekly)** | Every **Sunday at 02:00 UTC** — explicit weekly check. |
| **`workflow_dispatch`** | Manually from the *Actions* tab, with an optional *force build* flag. |

Because GitHub Actions cannot subscribe to push events in another user's
repository, the "trigger on upstream commit" requirement is satisfied by the
daily polling schedule. The workflow resolves the current upstream default
branch at runtime, compares its latest SHA with the SHA stored in
`.github/build-state.json`, and only builds when they differ (or when forced).

---

## Build logic

```
┌─────────────────────────────────────────────────┐
│ 1. Checkout paludi/ha-plugins-sip-gateway        │
│ 2. Resolve upstream HEAD branch + latest SHA     │
│ 3. Compare with .github/build-state.json         │
│                                                  │
│    SHA unchanged?  ──► skip (exit 0)             │
│    SHA changed?    ──► continue                  │
│                                                  │
│ 4. Checkout arnonym/ha-plugins @ default branch  │
│ 5. docker build (local, no push)                 │
│ 6. Run unit tests inside built container         │
│                                                  │
│    Tests fail?  ──► fail workflow (no push)      │
│    Tests pass?  ──► continue                     │
│                                                  │
│ 7. docker push to ghcr.io/paludi/ha-sip:<ver>   │
│               and ghcr.io/paludi/ha-sip:latest  │
│ 8. Commit updated build-state.json               │
└─────────────────────────────────────────────────┘
```

### Why tests run inside the container

The upstream unit tests (in `ha-sip/src/tests/`) include tests that import
`pjsua2`, which is the Python binding for the [PJSIP](https://www.pjsip.org/)
library. PJSIP must be compiled from source — this is done during the Docker
build. Running the tests *inside* the already-built container means every test
can execute in the same environment that will be shipped, and there is no need
to separately compile PJSIP in CI.

---

## Versioning

Image versions follow `major.minor.patch` (e.g. `1.0.1`).

- The initial state in `.github/build-state.json` stores version `1.0.0`.
- Each successful publish **increments the patch component by 1**.
- The very first publish therefore produces version **`1.0.1`**.

The current version is always readable from `.github/build-state.json`:

```json
{
  "last_built_sha": "<upstream git SHA>",
  "version": "1.0.3",
  "last_build_date": "2025-06-01T06:00:00Z"
}
```

To bump the minor or major version manually, edit `build-state.json`, set the
desired version (e.g. `"version": "1.1.0"`), and commit the change. The next
automated build will then produce `1.1.1`.

---

## Image location

```
ghcr.io/paludi/ha-sip:<version>   # e.g. ghcr.io/paludi/ha-sip:1.0.1
ghcr.io/paludi/ha-sip:latest
```

The image is pushed to the **private** GHCR namespace of the `paludi` account.
Visibility can be changed in the GitHub Package settings if needed.

---

## Required permissions

The workflow uses only `GITHUB_TOKEN` (automatically provided by GitHub
Actions) — no additional secrets are needed.

| Permission | Reason |
|------------|--------|
| `contents: write` | Commit the updated `build-state.json` back to the repo. |
| `packages: write` | Push the Docker image to GHCR. |

> **Note:** Make sure the repository's *Actions* settings allow workflows to
> create commits (*Settings → Actions → General → Workflow permissions →
> Read and write permissions*).

---

## Manual / forced build

To rebuild and republish even if the upstream SHA has not changed:

1. Open the repository on GitHub.
2. Go to **Actions → Build and Publish ha-sip Docker Image**.
3. Click **Run workflow**.
4. Enable the **Force build** checkbox.
5. Click **Run workflow** to start.

---

## Source repository reference

| Item | Value |
|------|-------|
| Upstream repo | `arnonym/ha-plugins` |
| Branch | Upstream default branch (resolved at runtime) |
| Docker context | `./ha-sip` |
| Dockerfile | `ha-sip/Dockerfile` |
| Build arg | `BUILD_FROM=debian:bookworm` |
