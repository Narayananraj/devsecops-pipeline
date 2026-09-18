# CI/CD Pipeline — DevSecOps Workflow

This directory contains the GitHub Actions automation that drives the complete build-to-deploy lifecycle for this project, implementing DevSecOps practices at every stage: automated quality gates, container vulnerability scanning, and GitOps-based deployment.

## Workflow File

- `ci-cd.yml` — five-stage pipeline covering testing, static analysis, build, container security scanning, and Kubernetes manifest updates

## Trigger Conditions

The workflow runs on:

- Pushes to the `main` branch
- Pull requests targeting `main`

Pushes that **only** modify `kubernetes/deployment.yaml` are ignored (`paths-ignore`). This exists because the pipeline itself commits to that file in its final stage — without this exclusion, every automated deployment update would re-trigger the entire pipeline, creating an infinite loop.

## Pipeline Architecture

The workflow enforces a sequential quality-gate model — each stage must pass before the next begins, and no stage can be silently bypassed on failure.

```
┌──────┐   ┌──────┐
│ test │   │ lint │   (parallel — no dependency between them)
└───┬──┘   └───┬──┘
    └─────┬─────┘
      ┌───▼───┐
      │ build │
      └───┬───┘
      ┌───▼────┐
      │ docker │   (build → scan → push, single build reused throughout)
      └───┬────┘
   ┌──────▼───────┐
   │  update-k8s  │   (GitOps commit — triggers ArgoCD sync)
   └──────────────┘
```

## Stage Breakdown

### 1. Test

- **Runner:** `ubuntu-latest`
- Checks out the repository
- Installs Node.js with dependency caching enabled
- Installs dependencies via `npm ci` (reproducible, lockfile-exact install)
- Runs the unit test suite

### 2. Lint

- **Runner:** `ubuntu-latest`, runs in parallel with `test`
- Same checkout and dependency setup
- Runs static code analysis to catch quality and style issues before they reach a build

### 3. Build

- **Depends on:** `test`, `lint` (both must succeed)
- Compiles the production bundle
- Uploads the build output as a workflow artifact, so downstream stages consume the exact same build rather than recompiling

### 4. Docker — Build, Scan, and Push

- **Depends on:** `build`
- Downloads the build artifact (no rebuild)
- Sets up Docker Buildx and authenticates to GitHub Container Registry (GHCR)
- Normalizes the repository name to lowercase, since OCI image references must be lowercase-only while GitHub repository names may contain mixed case
- Generates immutable, SHA-based image tags (no mutable `latest` tag, to keep every deployed image traceable to an exact commit)
- **Builds the image once**, loading it into the local Docker daemon without pushing
- Scans that exact image with Trivy for `CRITICAL` and `HIGH` severity vulnerabilities across OS packages and application libraries
- The workflow fails immediately if vulnerabilities are found — the push step never executes on a failed scan
- Only after a clean scan does the same, already-scanned image get pushed to GHCR

This build-once-scan-then-push sequence ensures the image that gets scanned is exactly the image that gets deployed — there is no separate rebuild between security validation and registry push.

### 5. Update Kubernetes Deployment (GitOps)

- **Depends on:** `docker`, and only runs on a direct push to `main` (not on pull requests)
- Checks out the repository with write access
- Recomputes the lowercase image name and constructs the new image reference using the commit SHA
- Updates the image tag in `kubernetes/deployment.yaml` in place
- Commits and pushes the change back to the repository, with `[skip ci]` in the commit message so this automated commit does not re-trigger the pipeline

This stage is the bridge into GitOps: it does not deploy anything directly. Instead, it updates the desired state in Git, which a GitOps controller (ArgoCD) watches and syncs to the cluster automatically. This gives a complete, auditable trail — every production image change is a Git commit, not a manual `kubectl apply`.

## Container Registry Setup

- Registry: `ghcr.io` (GitHub Container Registry)
- Authenticates as `github.actor` using a GitHub Actions secret named `TOKEN`
- Images are published under the lowercase repository name
- Tagged by full commit SHA and branch reference

## Security Controls

| Control | Configuration |
|---|---|
| Vulnerability scanner | Trivy |
| Scan scope | `os`, `library` |
| Severity gate | `CRITICAL`, `HIGH` |
| Failure behavior | `exit-code: 1` — blocks the pipeline, no silent pass-through |
| Noise reduction | `ignore-unfixed: true` — skips vulnerabilities with no available patch |

No step in this pipeline uses failure-suppression patterns (`continue-on-error`, `|| true`, or equivalent) on a quality or security check. Every gate either passes cleanly or stops the pipeline.

## Artifacts

| Artifact | Produced by | Consumed by |
|---|---|---|
| `build-artifacts` | `build` | `docker` |

## GitOps Integration

This workflow's final responsibility ends at updating the deployment manifest in Git. Cluster-side reconciliation is handled by ArgoCD, configured separately to watch the `kubernetes/` path of this repository and auto-sync on change, with self-healing enabled to correct any manual drift in the cluster back to the Git-defined state.

## Notes

- Repository-specific values (registry credentials, image names) are resolved dynamically within the workflow and require no manual editing between environments.
- Kubernetes manifests are not applied directly by this workflow — deployment is fully GitOps-driven via ArgoCD.
- Ingress and TLS configuration are intentionally excluded from the current manifest set and can be layered in for environments with a real domain.

## Local Equivalent

The following commands approximate what the pipeline validates, for local verification before pushing:

```bash
npm ci
npm test
npm run lint
npm run build
docker build -t <image-name> .
```