# Project 1 Build Journal — Go Web App DevOps Lab

> **Project:** Go web application → Docker → Kubernetes → AWS EKS → NGINX Ingress → Helm → GitHub Actions CI → Docker Hub → Argo CD GitOps CD  
> **Repository:** `bryan-obabori/go-web-app-devops`  
> **Docker Hub image:** `bryanobabori/go-web-app`  
> **AWS region:** `us-east-1`  
> **EKS cluster:** `go-web-app-cluster`
>
> This journal records the major commands, design decisions, failures, fixes, and verification steps used while building Project 1.
>
> **Public-repository note:** Local home/document paths, personal or ephemeral IP addresses, secrets, tokens, passwords, AWS account identifiers, ARNs, and ephemeral load-balancer addresses are intentionally omitted or replaced with placeholders.

---

# 1. Goal

The goal was to take a Go web application through a complete DevOps delivery path:

```text
Go application
    ↓
Docker
    ↓
Kubernetes
    ↓
AWS EKS
    ↓
NGINX Ingress
    ↓
Helm
    ↓
GitHub Actions CI
    ↓
Docker Hub
    ↓
Argo CD GitOps
```

The project began with an existing Go application so the learning focus could remain on containerization, Kubernetes, CI/CD, EKS, Helm, and GitOps.

---

# 2. Run the Go Application Locally

The starter source was cloned and entered:

```bash
git clone <STARTER_REPOSITORY_URL>
cd go-web-app
```

Build the application:

```bash
go build -o main .
```

Run it:

```bash
./main
```

The application listened on port `8080` and was verified locally at:

```text
http://localhost:8080/courses
```

This established a working application baseline before introducing Docker or Kubernetes.

---

# 3. Containerize the Application

A multi-stage Dockerfile was used:

```dockerfile
FROM golang:1.22.5 AS base

WORKDIR /app
COPY go.mod .
RUN go mod download
COPY . .
RUN go build -o main .

FROM gcr.io/distroless/base-debian12 AS final

COPY --from=base /app/main .
COPY --from=base /app/static ./static

EXPOSE 8080
CMD ["./main"]
```

The builder stage contains the Go toolchain. The final stage contains only the compiled application and required static files, giving the runtime image a much smaller footprint.

Build locally:

```bash
docker build -t go-web-app:v1 .
```

Run locally:

```bash
docker run -p 8080:8080 go-web-app:v1
```

The containerized application was again verified through port `8080`.

The Docker Hub repository used throughout the lab was:

```text
bryanobabori/go-web-app
```

---

# 4. Create Kubernetes Resources

The original Kubernetes manifests were organized under:

```text
k8s/manifests/
```

The application required three core Kubernetes objects:

```text
Deployment
Service
Ingress
```

## Deployment

The Deployment kept one application Pod running with container port `8080`.

Representative configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go-web-app
  template:
    metadata:
      labels:
        app: go-web-app
    spec:
      containers:
        - name: go-web-app
          image: bryanobabori/go-web-app:v1
          ports:
            - containerPort: 8080
```

## Service

The Service remained internal to Kubernetes:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: go-web-app
spec:
  selector:
    app: go-web-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

Traffic flow:

```text
Service :80
    ↓
Pod :8080
```

## Ingress

The application used host-based ingress routing:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: go-web-app
spec:
  ingressClassName: nginx
  rules:
    - host: go-web-app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: go-web-app
                port:
                  number: 80
```

An Ingress object only defines routing rules. An ingress controller is required to implement them.

---

# 5. Create the EKS Cluster with `eksctl`

AWS CLI authentication was inspected with:

```bash
aws configure list
```

Sensitive AWS account and credential values are not recorded in this journal.

The EKS cluster was created with:

```bash
eksctl create cluster \
  --name go-web-app-cluster \
  --region us-east-1 \
  --nodegroup-name go-web-app-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --managed
```

This created:

```text
EKS control plane
+ managed node group
+ 2 × t3.medium worker nodes
```

Verify:

```bash
kubectl get nodes
```

The lab reached two `Ready` nodes.

---

# 6. Deploy the Kubernetes Workload

The manifests were applied with:

```bash
kubectl apply -f k8s/manifests/deployment.yaml
kubectl apply -f k8s/manifests/service.yaml
kubectl apply -f k8s/manifests/ingress.yaml
```

One early apply returned:

```text
no objects passed to apply
```

The cause was simple: the YAML file had not yet been saved in the editor. After saving the file, the apply succeeded.

Useful checks included:

```bash
kubectl get pods
kubectl get pods -l app=go-web-app
kubectl get pods -l app=go-web-app -o wide
```

---

# 7. Docker Architecture Failure: Apple Silicon vs EKS AMD64

The development machine used Apple Silicon (`arm64`), while the EKS `t3.medium` workers were `amd64`.

The first Docker image did not contain a compatible AMD64 image manifest. Kubernetes reported errors including:

```text
ErrImagePull
ImagePullBackOff
no match for platform in manifest
```

The fix was to explicitly build the container for Linux AMD64:

```bash
docker buildx build \
  --platform linux/amd64 \
  -t bryanobabori/go-web-app:v1 \
  --push .
```

After the corrected image was available, the failing Pod was replaced and the workload could start successfully.

Key lesson:

```text
Developer CPU architecture
        ≠
Target Kubernetes node architecture
```

Container builds must target the architecture where they will actually run.

---

# 8. Install NGINX Ingress Controller

The AWS deployment manifest for ingress-nginx was applied:

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/aws/deploy.yaml
```

The ingress controller Service was inspected with:

```bash
kubectl get svc -n ingress-nginx
```

It exposed a Kubernetes Service of type:

```text
LoadBalancer
```

AWS then provisioned a Network Load Balancer.

The actual NLB hostname and resolved IP addresses are intentionally omitted from this public journal.

Representative checks were:

```bash
nslookup <AWS-NLB-HOSTNAME>
```

---

# 9. Local Host-Based Routing

The Kubernetes Ingress expected the hostname:

```text
go-web-app.local
```

For the lab, the local hosts file mapped that hostname to an AWS NLB address using a placeholder equivalent to:

```text
<AWS-NLB-IP> go-web-app.local
```

The system hosts file was edited with:

```bash
sudo vi /etc/hosts
```

A useful troubleshooting lesson was that local hosts-file resolution and DNS-server resolution are different things:

```text
/etc/hosts resolution ≠ DNS server resolution
```

The definitive test was HTTP rather than ICMP:

```bash
curl -v http://go-web-app.local/courses
```

The successful traffic path became:

```text
local host mapping
      ↓
AWS Network Load Balancer
      ↓
NGINX Ingress Controller
      ↓
Ingress rule
      ↓
ClusterIP Service :80
      ↓
Pod :8080
      ↓
Go application
```

---

# 10. Package the Workload with Helm

The Helm chart was created under:

```text
helm/go-web-app-chart
```

The image tag became configurable through `values.yaml`:

```yaml
image:
  tag: v1
```

The Deployment template referenced the value:

```yaml
image: bryanobabori/go-web-app:{{ .Values.image.tag }}
```

Render locally:

```bash
helm template go-web-app ./go-web-app-chart
```

Install:

```bash
helm install go-web-app ./go-web-app-chart
```

Verify:

```bash
helm list
```

The release reached:

```text
STATUS: deployed
```

This moved the Kubernetes application configuration from three manually maintained manifests toward one reusable Helm package.

---

# 11. Create the Independent GitHub Project

The original Git remote was removed and the project was converted into an independent repository.

Inspect remotes:

```bash
git remote -v
```

Remove the old remote:

```bash
git remote remove origin
```

The Git history was reset for the new project:

```bash
rm -rf .git
git init -b main
```

Before the first commit, accidental local-only files were removed and the locally compiled Go binary was ignored:

```bash
echo "/main" >> .gitignore
```

Initial commit:

```bash
git add .
git commit -m "Initial DevOps implementation"
```

The public repository was created as:

```text
bryan-obabori/go-web-app-devops
```

using:

```bash
gh repo create bryan-obabori/go-web-app-devops \
  --public \
  --source=. \
  --remote=origin

git push -u origin main
```

---

# 12. Build the GitHub Actions CI Pipeline

The workflow file lives at:

```text
.github/workflows/ci.yaml
```

The pipeline evolved in stages.

## Build and test

```yaml
build-and-test:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v7
    - uses: actions/setup-go@v7
      with:
        go-version-file: go.mod
    - run: go build -o main .
    - run: go test ./...
```

This proved the project could build and test on a clean GitHub-hosted runner rather than only on the development machine.

## Static analysis

A lint job was added with `golangci-lint`:

```yaml
lint:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v7
    - uses: actions/setup-go@v7
      with:
        go-version-file: go.mod
    - uses: golangci/golangci-lint-action@v9
      with:
        version: v2.12
```

## Docker publishing

Docker publishing depends on both quality gates:

```yaml
docker:
  needs:
    - build-and-test
    - lint
```

Docker Hub credentials were stored as GitHub repository configuration rather than source code:

```text
Variable: DOCKERHUB_USERNAME
Secret:   DOCKERHUB_TOKEN
```

The pipeline used Buildx and explicitly targeted AMD64:

```yaml
- uses: docker/setup-buildx-action@v4

- uses: docker/login-action@v4
  with:
    username: ${{ vars.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

- uses: docker/build-push-action@v7
  with:
    context: .
    push: true
    platforms: linux/amd64
    tags: bryanobabori/go-web-app:run-${{ github.run_id }}
```

The earlier architecture failure directly informed this CI design.

---

# 13. CI Updates the Helm Image Tag

The CI workflow was given repository write permission:

```yaml
permissions:
  contents: write
```

After building a new image, CI updates the Helm image tag:

```yaml
- name: Update Helm image tag
  run: |
    sed -i 's/tag: .*/tag: "run-${{ github.run_id }}"/' helm/go-web-app-chart/values.yaml
```

Then the GitHub Actions bot commits that GitOps state back to `main`:

```yaml
- name: Commit updated Helm values
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
    git add helm/go-web-app-chart/values.yaml
    git commit -m "Update image tag to ${{ github.run_id }}"
    git push
```

This creates the bridge between CI and GitOps:

```text
CI builds immutable image
        ↓
CI updates desired image tag in Git
        ↓
Argo CD observes Git
        ↓
Argo deploys desired version
```

---

# 14. Install Argo CD

Create the namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD with server-side apply:

```bash
kubectl apply -n argocd \
  --server-side \
  --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Server-side apply was used because Argo CD includes large CRDs that can hit client-side annotation limits.

Verify:

```bash
kubectl get pods -n argocd
```

Core Argo CD components reached `Running`.

The UI was accessed temporarily through a local port forward:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

The initial admin password was retrieved dynamically from Kubernetes rather than stored in source control:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

---

# 15. Create the Argo CD Application

The application was registered with Argo CD:

```bash
argocd app create go-web-app \
  --repo https://github.com/bryan-obabori/go-web-app-devops.git \
  --revision main \
  --path helm/go-web-app-chart \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --project default
```

This configured Argo CD to watch:

```text
GitHub repository
  → main branch
  → helm/go-web-app-chart
  → default Kubernetes namespace
```

The application initially used manual synchronization.

---

# 16. Transfer Deployment Ownership to Argo CD

The application had already been installed manually with Helm.

To avoid two deployment systems trying to control the same resources, the manual Helm release was removed:

```bash
helm list
helm uninstall go-web-app
```

Then the first Argo CD synchronization was performed:

```bash
argocd app sync go-web-app
```

The important principle was:

```text
one resource
→ one clear configuration owner
```

After this point, Argo CD became the application deployment manager.

---

# 17. Diagnose `InvalidImageName`

After Argo CD began deploying from Git, the Pod entered:

```text
0/1 InvalidImageName
```

Inspection showed an image reference rendered in scientific notation, conceptually:

```text
bryanobabori/go-web-app:3.xxxxxe+10
```

The intended value was a large numeric GitHub Actions run ID.

The problem was YAML typing: an unquoted large numeric image tag was interpreted as a number and later serialized in scientific notation. The `+` character made the resulting Docker tag invalid.

The fix was to make the tag unambiguously a string.

Docker image:

```yaml
tags: bryanobabori/go-web-app:run-${{ github.run_id }}
```

Helm value update:

```yaml
sed -i 's/tag: .*/tag: "run-${{ github.run_id }}"/' helm/go-web-app-chart/values.yaml
```

Result:

```yaml
image:
  tag: "run-<GITHUB-RUN-ID>"
```

and:

```text
bryanobabori/go-web-app:run-<GITHUB-RUN-ID>
```

This resolved the invalid image-name problem.

---

# 18. Handle Git Divergence Caused by CI Bot Commits

Because GitHub Actions updates `values.yaml` and pushes to `main`, the remote branch can advance after a developer's last pull.

A local push therefore encountered:

```text
non-fast-forward
```

The safe recovery was:

```bash
git pull --rebase origin main
git push
```

When an unwanted local `values.yaml` edit was also present, it was discarded first:

```bash
git restore helm/go-web-app-chart/values.yaml
git pull --rebase origin main
git push
```

The important lesson was not to blindly force-push over CI-generated history.

---

# 19. Enable Fully Automated GitOps

Argo CD was switched from manual synchronization to automated reconciliation:

```bash
argocd app set go-web-app --sync-policy automated
argocd app set go-web-app --auto-prune
argocd app set go-web-app --self-heal
```

Meaning:

```text
Automated sync
Git changes → deployment

Auto-prune
resource removed from Git → resource removed from cluster

Self-heal
manual drift → restored to Git-defined state
```

Verification:

```bash
argocd app get go-web-app
```

Target state:

```text
Sync Policy: Automated (Prune)
Sync Status: Synced
Health Status: Healthy
```

---

# 20. End-to-End GitOps Test

The final test was designed to prove this complete path:

```text
application source change
        ↓
git push
        ↓
GitHub Actions build/test/lint
        ↓
Docker image pushed
        ↓
Helm image tag committed
        ↓
Argo CD detects Git change
        ↓
EKS Deployment rolls out
        ↓
website changes
```

A visible application text change was committed and pushed.

During validation, this command:

```bash
argocd app wait go-web-app --sync --health --timeout 180
```

returned too early because Argo CD's currently known revision was already healthy. `app wait` waits for the current target state; it does not inherently wait for a future Git revision to appear.

A hard refresh solved that timing issue:

```bash
argocd app get go-web-app --hard-refresh
```

The final combined verification was:

```bash
argocd app get go-web-app --hard-refresh && \
argocd app wait go-web-app --sync --health --timeout 180 && \
argocd app get go-web-app && \
kubectl get pods -l app=go-web-app && \
curl -s http://go-web-app.local/courses | grep "GitOps Deployed"
```

The test succeeded.

This proved that a source-code change could reach the running application without manually executing an Argo CD sync.

---

# 21. Final CI Workflow

The effective GitHub Actions workflow reached this form:

```yaml
name: CI

on:
  push:
    branches:
      - main

permissions:
  contents: write

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod
      - run: go build -o main .
      - run: go test ./...

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with:
          go-version-file: go.mod
      - uses: golangci/golangci-lint-action@v9
        with:
          version: v2.12

  docker:
    needs:
      - build-and-test
      - lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: docker/setup-buildx-action@v4

      - uses: docker/login-action@v4
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          platforms: linux/amd64
          tags: bryanobabori/go-web-app:run-${{ github.run_id }}

      - name: Update Helm image tag
        run: |
          sed -i 's/tag: .*/tag: "run-${{ github.run_id }}"/' helm/go-web-app-chart/values.yaml

      - name: Commit updated Helm values
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add helm/go-web-app-chart/values.yaml
          git commit -m "Update image tag to ${{ github.run_id }}"
          git push
```

---

# 22. Frequently Used Verification Commands

## Git

```bash
git status
git remote -v
git pull
git push
```

## GitHub Actions

```bash
gh run list --workflow ci.yaml --limit 3
gh run watch
gh run view --log-failed
```

## Kubernetes

```bash
kubectl get nodes
kubectl get pods
kubectl get pods -l app=go-web-app
kubectl get pods -l app=go-web-app -o wide
```

Inspect the deployed image:

```bash
kubectl get deployment go-web-app \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

## Helm

```bash
helm list
helm template go-web-app ./go-web-app-chart
```

## Argo CD

```bash
argocd app get go-web-app
argocd app wait go-web-app --sync --health --timeout 180
```

## Application

```bash
curl -v http://go-web-app.local/courses
```

---

# 23. Security Practices

No secret values belong in source control.

The project used repository-level GitHub configuration for Docker Hub credentials:

```text
Variable:
DOCKERHUB_USERNAME

Secret:
DOCKERHUB_TOKEN
```

Argo CD's bootstrap password was retrieved from Kubernetes at runtime rather than copied into a repository file.

This public journal intentionally excludes:

```text
Docker Hub PAT
Argo CD admin password
AWS access/secret keys
AWS account ID
AWS root/user ARNs
temporary AWS credentials
personal or ephemeral IP addresses
AWS NLB IP addresses
local home/document paths
```

---

# 24. Project Outcome

The final working system was:

```text
Developer
   │
   │ git push
   ▼
GitHub
   │
   ├── Build
   ├── Test
   ├── Lint
   │
   ▼
Docker Buildx
linux/amd64
   │
   ▼
Docker Hub
bryanobabori/go-web-app:run-<RUN-ID>
   │
   ▼
GitHub Actions updates Helm values.yaml
   │
   ▼
Git commit
   │
   ▼
Argo CD
Automated + Prune + Self Heal
   │
   ▼
AWS EKS
   │
   ▼
Deployment → Pod
   │
   ▼
ClusterIP Service
   │
   ▼
NGINX Ingress Controller
   │
   ▼
AWS Network Load Balancer
   │
   ▼
go-web-app.local
```

The most useful lessons from Project 1 were not just the happy-path commands. They included:

- understanding container architecture compatibility
- distinguishing `ImagePullBackOff` from `InvalidImageName`
- understanding that YAML can reinterpret an unquoted numeric-looking value
- separating Kubernetes Ingress rules from the ingress controller implementing them
- learning how Helm templates turn configuration values into Kubernetes objects
- understanding why a CI pipeline should gate image publication behind build/test/lint success
- storing credentials in GitHub secrets rather than repository files
- recognizing that CI bot commits can create legitimate Git divergence
- understanding Argo CD refresh, sync, prune, and self-heal behavior
- proving an end-to-end GitOps deployment rather than only demonstrating individual tools

Project 1 established the application delivery and GitOps foundation later reused in Project 2, where the AWS/EKS infrastructure itself was rebuilt explicitly with Terraform.
