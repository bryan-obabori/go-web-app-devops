# Go Web App — CI/CD & GitOps on AWS EKS

A hands-on DevOps project that takes a simple Go web application from source code to an automated Kubernetes deployment on AWS EKS.

The project demonstrates containerization, Kubernetes, Helm, GitHub Actions CI, Docker Hub, Argo CD GitOps, ingress routing, and automated application rollouts.

## Architecture

```text
Developer
   |
   | git push
   v
GitHub Repository
   |
   v
GitHub Actions
   |
   |-- Build Go application
   |-- Run unit tests
   |-- Run golangci-lint
   |-- Build linux/amd64 Docker image
   |-- Push image to Docker Hub
   |-- Update Helm image tag
   |
   v
Git Repository
helm/go-web-app-chart/values.yaml
   |
   v
Argo CD
   |
   |-- Automated Sync
   |-- Prune
   |-- Self Heal
   v
AWS EKS
   |
   v
Kubernetes Deployment
   |
   v
Service
   |
   v
NGINX Ingress Controller
   |
   v
Application
```

## Tech Stack

- Go
- Docker
- Docker Hub
- Kubernetes
- AWS EKS
- eksctl
- Helm
- GitHub Actions
- Argo CD
- NGINX Ingress Controller
- AWS Network Load Balancer
- kubectl

## Repository Structure

```text
.
├── .github/
│   └── workflows/
│       └── ci.yaml
├── helm/
│   └── go-web-app-chart/
│       ├── templates/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── ingress.yaml
│       ├── Chart.yaml
│       └── values.yaml
├── k8s/
│   └── manifests/
├── static/
├── Dockerfile
├── go.mod
├── main.go
└── main_test.go
```

## CI Pipeline

Every application push to `main` triggers GitHub Actions.

The pipeline:

1. Checks out the repository.
2. Configures Go.
3. Builds the application.
4. Runs unit tests.
5. Runs static analysis with `golangci-lint`.
6. Builds a Docker image for `linux/amd64`.
7. Pushes the image to Docker Hub.
8. Tags each image using the GitHub Actions run ID.
9. Updates the Helm `values.yaml` file with the new image tag.
10. Commits the updated desired state back to Git.

Example Docker image:

```text
bryanobabori/go-web-app:run-32087572959
```

Example Helm value:

```yaml
image:
  tag: "run-32087572959"
```

Docker Hub authentication is supplied to GitHub Actions through repository secrets rather than stored in source control.

## GitOps Continuous Deployment

Argo CD runs inside the EKS cluster and monitors:

```text
helm/go-web-app-chart
```

When CI updates the image tag in Git, Argo CD detects the new desired state and deploys it automatically.

The Argo CD application is configured with:

- Automated synchronization
- Pruning
- Self healing

Git therefore acts as the source of truth for the Kubernetes application state.

## Kubernetes Architecture

The application uses three main Kubernetes resources.

### Deployment

Runs the Go application container and maintains the desired number of Pods.

### Service

Provides a stable internal endpoint and forwards traffic:

```text
Service port 80
      |
      v
Container port 8080
```

### Ingress

Uses host-based routing through the NGINX Ingress Controller.

```text
go-web-app.local
      |
      v
NGINX Ingress
      |
      v
Service
      |
      v
Pod
```

## Helm

The Kubernetes manifests are packaged into a Helm chart.

The Deployment references the image tag dynamically:

```yaml
image: bryanobabori/go-web-app:{{ .Values.image.tag }}
```

CI only needs to update:

```yaml
image:
  tag: "run-<github-run-id>"
```

instead of rewriting the Kubernetes Deployment manifest.

## Running Locally

Build the application:

```bash
go build -o main .
```

Run it:

```bash
./main
```

Open:

```text
http://localhost:8080/courses
```

## Docker

Build locally:

```bash
docker build -t go-web-app:v1 .
```

Run:

```bash
docker run -p 8080:8080 go-web-app:v1
```

The CI pipeline explicitly builds for:

```text
linux/amd64
```

to match the EKS worker-node architecture.

## Key Troubleshooting Lessons

### Container Architecture Mismatch

The application was initially built on an Apple Silicon Mac, producing an ARM64 image.

The EKS worker nodes were AMD64, causing Kubernetes to fail to pull a compatible image.

The pipeline was updated to explicitly build:

```yaml
platforms: linux/amd64
```

### Invalid Kubernetes Image Name

A GitHub Actions run ID was initially written into YAML as an unquoted numeric value:

```yaml
tag: 32085297528
```

YAML interpreted the value numerically and Helm rendered it using scientific notation:

```text
bryanobabori/go-web-app:3.2085297528e+10
```

Kubernetes rejected the image with:

```text
InvalidImageName
```

The solution was to make the Docker tag explicitly string-based:

```yaml
tag: "run-32085297528"
```

### Host-Based Ingress Routing

Accessing the AWS load balancer directly by IP returned an NGINX `404`.

The Ingress rule expected:

```text
Host: go-web-app.local
```

A local `/etc/hosts` mapping was used so requests contained the expected hostname and could be routed through the Ingress to the application.

## End-to-End GitOps Test

A visible application change was made and pushed to Git.

No manual Argo CD synchronization command was used.

```text
Code Change
    |
    v
GitHub Actions
    |
    v
Build + Test + Lint
    |
    v
New Docker Image
    |
    v
Helm Tag Updated
    |
    v
Argo CD Detects Change
    |
    v
EKS Rolls Out New Pod
    |
    v
Updated Website
```

The application ultimately reached:

```text
Sync Status: Synced
Health Status: Healthy
```

and the updated content was successfully served through the Kubernetes ingress.

## What This Project Demonstrates

- Building multi-stage Docker images
- Running containerized Go applications
- Deploying applications to Kubernetes
- Running workloads on AWS EKS
- Packaging Kubernetes applications with Helm
- Implementing CI with GitHub Actions
- Managing CI credentials with GitHub secrets
- Publishing versioned Docker images
- Implementing GitOps CD with Argo CD
- Configuring Kubernetes ingress
- Diagnosing container architecture problems
- Troubleshooting Kubernetes image failures
- Understanding desired state and automated reconciliation

## Application

The application is a lightweight Go HTTP service used as the workload for this DevOps implementation.

The primary focus of this repository is the infrastructure, CI/CD, Kubernetes, and GitOps workflow.