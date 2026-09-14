# DevSecOps Pipeline

A Vite + React + TypeScript tic-tac-toe web application with Docker and Kubernetes deployment artifacts. The project demonstrates a small DevSecOps-friendly application workflow: build a front-end UI, run automated tests, package it in a container, and prepare it for deployment in a containerized environment.

## Project Overview

This repository contains a classic Tic-Tac-Toe game with the following UI and game features:

- Interactive 3x3 board for two players
- Win detection and draw detection
- Live score tracking for X, O, and draw outcomes
- Game history with completed board snapshots and timestamps
- Reset controls for current game and full score history
- Responsive styling using React, Vite, TypeScript, and Tailwind CSS

## Tech Stack

- React 18
- Vite
- TypeScript
- Vitest
- ESLint
- Tailwind CSS
- Docker
- Kubernetes manifests

## Project Structure

```text
.
├── Dockerfile
├── package.json
├── src/
│   ├── App.tsx
│   ├── components/
│   ├── utils/
│   └── __tests__/
├── kubernetes/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── vite.config.ts
```

## Getting Started

### Prerequisites

Install the following before running the project:

- Node.js 20+
- npm
- Docker (optional, for container builds)
- kubectl (optional, for Kubernetes deployment)

### Install dependencies

```bash
npm install
```

### Run locally

```bash
npm run dev
```

The development server starts with Vite and serves the application locally.

## Available Scripts

```bash
npm run dev      # start the Vite development server
npm run build    # create a production build
npm run lint     # run ESLint checks
npm run test     # run Vitest test suite
npm run preview  # preview the production build locally
```

## Testing

The project includes unit tests for the game logic in the test suite under the source test directory.

```bash
npm run test
```

## Docker

A multi-stage Dockerfile is included to build a production asset bundle and serve it using an unprivileged Nginx container.

```bash
docker build -t devsecops-pipeline .
docker run -p 8080:8080 devsecops-pipeline
```

The container is configured to expose port 8080.

## Kubernetes

The workspace includes Kubernetes manifest placeholders for deployment, service, and ingress resources:

```text
kubernetes/
├── deployment.yaml
├── service.yaml
├── ingress.yaml
```

Update those manifests with your environment-specific namespace, image repository, and ingress host settings before deploying.

## License

This project is provided for learning and local development purposes. Add a license file if you intend to distribute or reuse the project in a production environment.


