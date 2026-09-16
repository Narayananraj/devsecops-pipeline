# GitHub Actions Workflows

This directory contains the CI/CD automation for the DevSecOps pipeline project.

## Workflow File

- `ci-cd.yml` — main build, test, lint, vulnerability-scan, and container image workflow

## Trigger Conditions

The workflow runs on:

- Pushes to the `main` branch
- Pull requests targeting `main`

It ignores changes to `kubernetes/deployment.yaml` when triggered by push, which helps avoid redundant rebuilds for deployment manifest-only updates.

## Pipeline Overview

The workflow is organized into a sequential pipeline with dependency gates:

1. `test`
   - Runs unit tests with Vitest
   - Ensures the application logic passes after each change

2. `lint`
   - Runs ESLint static analysis
   - Validates code quality and catches common issues

3. `build`
   - Depends on `test` and `lint`
   - Installs dependencies and compiles the Vite production bundle
   - Uploads the built artifacts from `dist/`

4. `docker`
   - Depends on `build`
   - Downloads the build output
   - Configures Docker Buildx
   - Logs in to GitHub Container Registry (GHCR)
   - Builds the Docker image
   - Scans the image with Trivy
   - Pushes the image to GHCR when the scan passes

## Job Details

### Test Job

- Runner: `ubuntu-latest`
- Checkout code
- Set up Node.js 20 with npm cache
- Run `npm ci`
- Execute `npm test`

### Lint Job

- Runner: `ubuntu-latest`
- Checkout code
- Set up Node.js 20 with npm cache
- Run `npm ci`
- Execute `npm run lint`

### Build Job

- Runner: `ubuntu-latest`
- Waits for `test` and `lint` to finish successfully
- Builds the production bundle with `npm run build`
- Uploads the generated `dist/` directory as a workflow artifact

### Docker Job

- Runner: `ubuntu-latest`
- Downloads build artifacts
- Builds the image using the repository Dockerfile
- Uses Trivy to scan for `CRITICAL` and `HIGH` vulnerabilities
- Fails the workflow if vulnerabilities are found
- Pushes the final image to GHCR using image metadata tags

## Container Registry Setup

This workflow expects a GitHub Actions secret named `TOKEN` to authenticate to GHCR.

The workflow does the following:

- Uses `docker/login-action@v3`
- Authenticates as `github.actor`
- Publishes the image under the repository name in lowercase
- Creates tags based on commit SHA and branch name

## Security Controls

The Docker stage includes a vulnerability gate using Trivy:

- `vuln-type: 'os,library'`
- `severity: 'CRITICAL,HIGH'`
- `exit-code: '1'`
- `ignore-unfixed: true`

This means the workflow will fail if the built image contains high-impact vulnerabilities that are still unpatched.

## Artifacts

The build job stores the production bundle as a workflow artifact named:

- `build-artifacts`

This artifact is then used in the Docker job.

## Notes

- The workflow currently focuses on build validation and container security.
- Kubernetes deployment manifests are not automatically applied by this workflow.
- You may extend this workflow with deployment jobs for staging or production environments if needed.
- Repository-specific values such as image names, registry credentials, and deployment targets may require environment-specific configuration.

## Local Equivalent

The workflow mirrors the following local commands:

```bash
npm ci
npm test
npm run lint
npm run build
docker build -t devsecops-pipeline .
```

These commands provide the same checks the pipeline enforces in GitHub Actions.
