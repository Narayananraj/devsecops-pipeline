# DevSecOps Pipeline

A React + TypeScript tic-tac-toe web application used as a working example of a complete DevSecOps delivery pipeline — from source code to a running, GitOps-managed Kubernetes deployment. The application itself is intentionally simple; the focus of this project is the surrounding engineering: automated quality gates, container security scanning, hardened container design, and GitOps-based deployment.

## What This Project Demonstrates

- A CI/CD pipeline with sequential quality gates — testing and linting must pass before a build is produced, and a build must pass before a container image is created
- Container vulnerability scanning integrated directly into the build pipeline, with the pipeline blocking on critical or high-severity findings
- A build-once, scan-in-place, push-if-clean pattern — the image that is scanned is the exact image that is pushed, with no rebuild in between
- A hardened container image running as a non-root user, based on a minimal Alpine footprint
- A GitOps deployment model, where the pipeline updates a Kubernetes manifest in Git and a cluster-side controller (ArgoCD) reconciles the cluster to match
- Cost-conscious infrastructure choices throughout, favoring lightweight images, artifact reuse, and ephemeral test infrastructure over always-on resources

## Application Overview

A classic Tic-Tac-Toe game with:

- Interactive 3x3 board for two players
- Win detection and draw detection
- Live score tracking for X, O, and draw outcomes
- Game history with completed board snapshots and timestamps
- Reset controls for the current game and full score history
- Responsive styling using React, Vite, TypeScript, and Tailwind CSS

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript, Vite |
| Styling | Tailwind CSS |
| Testing | Vitest |
| Linting | ESLint |
| Containerization | Docker (multi-stage build, non-root Nginx) |
| CI/CD | GitHub Actions |
| Security scanning | Trivy |
| Container registry | GitHub Container Registry (GHCR) |
| Orchestration | Kubernetes |
| Deployment model | GitOps via ArgoCD |

## Project Structure

```text
.
├── .github/
│   └── workflows/
│       ├── ci-cd.yml        # CI/CD pipeline definition
│       └── README.md        # Pipeline-specific documentation
├── kubernetes/
│   ├── deployment.yaml      # Application Deployment (image tag managed by CI)
│   ├── service.yaml         # ClusterIP Service
│   └── ingress.yaml         # Ingress (optional, for domain-based access)
├── src/
│   ├── App.tsx
│   ├── components/
│   ├── utils/
│   └── __tests__/
├── Dockerfile                # Multi-stage build, non-root Nginx runtime
├── nginx.conf                 # Hardened Nginx config with security headers
├── .dockerignore
├── .nvmrc                     # Pinned Node.js version for local/CI parity
├── vite.config.ts
├── tailwind.config.js
├── eslint.config.js
└── package.json
```

## Getting Started

### Prerequisites

- Node.js (version pinned in `.nvmrc`)
- npm
- Docker (for container builds)
- kubectl (for Kubernetes deployment)

### Install dependencies

```bash
npm install
```

### Run locally

```bash
npm run dev
```

## Available Scripts

```bash
npm run dev      # start the Vite development server
npm run build    # create a production build
npm run lint     # run ESLint checks
npm run test     # run the Vitest test suite
npm run preview  # preview the production build locally
```

## Testing

```bash
npm run test
```

Unit tests cover the core game logic and are run automatically in CI before any build proceeds.

## Docker

The Dockerfile uses a multi-stage build: a Node build stage compiles the production bundle, and a minimal, non-root Nginx image (`nginxinc/nginx-unprivileged`) serves the static output. The container runs as an unprivileged user and listens on port 8080 rather than 80, avoiding the need for root privileges at runtime.

```bash
docker build -t devsecops-pipeline .
docker run -p 8080:8080 devsecops-pipeline
```

Visit `http://localhost:8080` to view the running application.

## CI/CD Pipeline

The full pipeline is defined in `.github/workflows/ci-cd.yml` and documented in detail in `.github/workflows/README.md`. In summary, it runs five stages in dependency order:

```
test ──┐
       ├──> build ──> docker (build, scan, push) ──> update-k8s (GitOps commit)
lint ──┘
```

- **test / lint** run in parallel and gate the pipeline
- **build** compiles the production bundle and hands it forward as an artifact
- **docker** builds the container image once, scans it with Trivy, and pushes it to GHCR only if the scan is clean
- **update-k8s** updates the image tag in `kubernetes/deployment.yaml` and commits the change back to the repository, which ArgoCD then syncs to the cluster

No stage in this pipeline can be silently skipped on failure — every quality and security gate either passes or stops the pipeline.

## Kubernetes & GitOps

The `kubernetes/` directory holds the desired state of the application in the cluster:

```text
kubernetes/
├── deployment.yaml   # image tag updated automatically by CI
├── service.yaml
└── ingress.yaml       # optional, requires a real domain and ingress controller
```

Deployment is not performed by running `kubectl apply` manually or from CI. Instead, an ArgoCD Application watches the `kubernetes/` path of this repository and automatically syncs any change to the cluster, with self-healing enabled to correct manual drift. This means the Git history of `kubernetes/deployment.yaml` is a complete audit trail of every image that has been deployed.

## Security Controls Summary

| Control | Detail |
|---|---|
| Dependency vulnerabilities | Checked via `npm audit`; Dependabot recommended for ongoing monitoring |
| Container vulnerabilities | Trivy scan, gated on `CRITICAL` and `HIGH` severity |
| Container privilege | Runs as non-root (`nginx-unprivileged`) |
| Image traceability | Immutable, commit-SHA-based tags — no mutable `latest` tag |
| Registry authentication | Scoped GitHub token, least-privilege where possible |
| Deployment auditability | GitOps — every deployed change is a Git commit |

## License

This project is provided for learning and portfolio purposes. Add a license file if you intend to distribute or reuse it in a production environment.