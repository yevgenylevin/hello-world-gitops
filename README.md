# Hello World CI/CD and GitOps Project

## Project Overview

This project demonstrates a complete DevOps workflow for a containerized Flask application.

The application exposes:

* `/` — returns `Hello World`
* `/health` — returns a JSON health response

The project includes:

* GitHub source control
* Docker containerization
* Jenkins CI pipeline
* Docker Hub image publishing
* Trivy vulnerability scanning
* Kubernetes deployment
* Helm chart deployment
* Argo CD GitOps deployment
* Dev, stage, and prod environments
* Argo CD App of Apps
* Argo Rollouts canary deployment
* Horizontal Pod Autoscaler

## Architecture

```text
GitHub
  ↓
Jenkins CI
  ↓
Docker Build
  ↓
Trivy Security Scan
  ↓
Docker Hub
  ↓
GitHub Helm and GitOps configuration
  ↓
Argo CD
  ↓
Kubernetes
  ↓
Argo Rollouts Canary Deployment
```

## Repository Structure

```text
.
├── app/
│   └── app.py
├── Dockerfile
├── Jenkinsfile
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── helm/
│   └── hello-world/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── environments/
│   ├── dev/
│   │   └── values.yaml
│   ├── stage/
│   │   └── values.yaml
│   └── prod/
│       └── values.yaml
├── argocd/
│   ├── app-of-apps.yaml
│   └── apps/
├── rollouts/
│   └── canary/
│       ├── rollout.yaml
│       └── service.yaml
└── README.md
```

## Run the Application Locally

Install Python dependencies:

```cmd
py -m pip install -r requirements.txt
```

Start the application:

```cmd
py app\app.py
```

Open:

```text
http://localhost:5000
```

Health endpoint:

```text
http://localhost:5000/health
```

## Docker

Build the image:

```cmd
docker build -t leyevge/hello-world:v2 .
```

Run the container:

```cmd
docker run --rm -p 5000:5000 leyevge/hello-world:v2
```

Push the image:

```cmd
docker push leyevge/hello-world:v2
```

## Kubernetes Deployment

Deploy the mandatory Kubernetes manifests:

```cmd
kubectl apply -f k8s\deployment.yaml
kubectl apply -f k8s\service.yaml
```

Verify:

```cmd
kubectl get deployments
kubectl get pods
kubectl get services
```

The application is available through:

```text
http://localhost:30080
```

The deployment uses:

* 2 replicas
* CPU requests: `100m`
* CPU limit: `250m`
* Memory request: `128Mi`
* Memory limit: `256Mi`
* NodePort service on port `30080`

## Jenkins CI Pipeline

The Jenkins pipeline is defined in `Jenkinsfile`.

Pipeline stages:

1. Checkout source code from GitHub
2. Build Docker image
3. Scan Docker image with Trivy
4. Fail on CRITICAL vulnerabilities
5. Push the image to Docker Hub

Docker Hub credentials are stored in Jenkins Credentials.

Required Jenkins credential:

```text
ID: dockerhub-credentials
Kind: Username with password
Username: Docker Hub username
Password: Docker Hub access token
```

The credential ID is referenced in the Jenkinsfile:

```groovy
DOCKERHUB_CREDENTIALS = "dockerhub-credentials"
```

## Trivy Security Scan

The pipeline runs Trivy before pushing the Docker image:

```text
trivy image --severity CRITICAL --exit-code 1 --no-progress <image>
```

The pipeline fails when Trivy detects a CRITICAL vulnerability.

The Docker image was changed from a Debian-based Python image to:

```dockerfile
FROM python:3.12-alpine
```

This removed the CRITICAL vulnerabilities detected in the earlier Debian-based image.

## Helm Deployment

Create the Helm release:

```cmd
helm install hello-world helm\hello-world
```

Verify:

```cmd
helm list
kubectl get pods
kubectl get services
```

Upgrade after changes:

```cmd
helm upgrade hello-world helm\hello-world
```

## Multi-Environment Deployment

The project contains separate Helm values files for three environments.

| Environment | Replicas | Image Tag | CPU Limit | Memory Limit |
| ----------- | -------: | --------- | --------- | ------------ |
| Dev         |        1 | v2        | 100m      | 128Mi        |
| Stage       |        2 | v2        | 250m      | 256Mi        |
| Prod        |        3 | v2        | 500m      | 512Mi        |

Render an environment locally:

```cmd
helm template hello-world-dev helm\hello-world -f environments\dev\values.yaml
```

## Argo CD GitOps

Argo CD deploys the Helm chart automatically from GitHub.

Child applications:

* `hello-world-dev`
* `hello-world-stage`
* `hello-world-prod`

The applications use automated synchronization:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

Check application status:

```cmd
kubectl get applications -n argocd
```

Expected status:

```text
Synced
Healthy
```

## Argo CD App of Apps

The parent application is defined in:

```text
argocd/app-of-apps.yaml
```

Apply it:

```cmd
kubectl apply -f argocd\app-of-apps.yaml
```

It manages the dev, stage, and prod Argo CD Applications from Git.

## Argo Rollouts Canary Deployment

The canary rollout is defined in:

```text
rollouts/canary/rollout.yaml
```

The rollout uses these gradual deployment steps:

```text
20%
50%
100%
```

Apply the initial rollout:

```cmd
kubectl create namespace hello-world-canary
kubectl apply -f rollouts\canary
```

Check rollout status:

```cmd
kubectl get rollout -n hello-world-canary
kubectl get pods -n hello-world-canary
```

Promote a paused rollout:

```cmd
kubectl-argo-rollouts promote hello-world-canary -n hello-world-canary
```

Rollback to the previous revision:

```cmd
kubectl-argo-rollouts undo hello-world-canary -n hello-world-canary
```

## Horizontal Pod Autoscaler

HPA is enabled for stage and prod.

Stage configuration:

```text
Minimum replicas: 2
Maximum replicas: 4
Target CPU utilization: 50%
```

Prod configuration:

```text
Minimum replicas: 3
Maximum replicas: 6
Target CPU utilization: 50%
```

Check HPA status:

```cmd
kubectl get hpa -A
kubectl describe hpa hello-world-stage -n hello-world-stage
```

Metrics Server is required for CPU-based autoscaling:

```cmd
kubectl top nodes
kubectl top pods -n hello-world-stage
```

## Troubleshooting

### Dockerfile not found

If Docker reports that it cannot find `Dockerfile`, check that Windows did not save it as `Dockerfile.txt`.

```cmd
dir
```

Rename if needed:

```cmd
ren Dockerfile.txt Dockerfile
```

### Jenkins Docker Hub login fails

Check that Jenkins Credentials contains a valid Docker Hub access token.

The credential ID must match:

```text
dockerhub-credentials
```

### Kubernetes pods are not running

Check pod details and logs:

```cmd
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Argo CD application is not syncing

Check application details:

```cmd
kubectl describe application hello-world-stage -n argocd
```

Force a Git refresh:

```cmd
kubectl annotate application hello-world-stage -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

### HPA shows unknown metrics

Check that Metrics Server is running:

```cmd
kubectl get pods -n kube-system | findstr metrics-server
kubectl top nodes
```

## Evidence

Before submission, include at least one Jenkins screenshot showing a successful pipeline run with:

* Docker image build
* Trivy scan
* Docker Hub push
* `Finished: SUCCESS`

Useful verification commands:

```cmd
kubectl get applications -n argocd
kubectl get hpa -A
kubectl get rollout -n hello-world-canary
kubectl get pods -n hello-world-canary
```
