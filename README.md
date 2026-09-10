# DevOps Pipeline Demo

An end-to-end CI/CD pipeline with GitOps delivery. A single push to this repository moves a Spring Boot application through build, testing, code quality analysis, containerization, and deployment to a Kubernetes cluster — with no manual deployment step.

## How it works

```
GitHub  ──►  Jenkins  ──►  SonarQube quality gate  ──►  Docker Hub
                 │
                 └──►  updates image tag in helm/values.yaml  ──►  commits to Git
                                                                        │
                                                          ArgoCD  ◄─────┘
                                                             │
                                                             ▼
                                                     Kubernetes cluster
```

**CI (Jenkins)**

1. Checkout from GitHub
2. `mvn clean package` — compile and package the application
3. `mvn test` — run unit tests
4. SonarQube scan, with a quality gate that fails the build if it doesn't pass
5. Build a Docker image tagged with the Jenkins build number
6. Push the image to Docker Hub

**CD (ArgoCD)**

7. Jenkins updates `image.tag` in `helm/values.yaml` and pushes the commit
8. ArgoCD, running inside the cluster, detects the change in Git
9. ArgoCD renders the Helm chart and applies it to Kubernetes
10. Kubernetes performs a rolling update to the new image

Jenkins never talks to Kubernetes directly. It only writes to Git and Docker Hub. ArgoCD is the only component holding cluster credentials, and Git is the single source of truth for what is deployed — so a rollback is just a revert.

## Stack

| Tool | Role |
|---|---|
| Java 17 / Spring Boot | Sample application |
| Maven | Build and dependency management |
| Jenkins | CI orchestration (declarative pipeline, Groovy) |
| SonarQube | Static analysis and quality gate |
| Docker | Containerization |
| Docker Hub | Image registry |
| Helm | Kubernetes manifest templating |
| ArgoCD | GitOps continuous delivery |
| Kubernetes | Runtime (three-node cluster) |

## Repository layout

```
.
├── src/                    Spring Boot application
├── Dockerfile              Multi-stage build for the application image
├── Jenkinsfile             Pipeline definition
├── pom.xml                 Maven build, includes the Sonar plugin
├── argocd-app.yaml         ArgoCD Application manifest
├── helm/
│   ├── Chart.yaml
│   ├── values.yaml         Image repository and tag (updated by Jenkins)
│   └── templates/
│       ├── deployment.yaml
│       └── service.yaml
└── infra/
    ├── docker-compose.yml  Local Jenkins and SonarQube
    └── Dockerfile.jenkins  Jenkins image with the Docker CLI installed
```

## Running it locally

**Prerequisites:** Docker Desktop with Kubernetes enabled, `kubectl`, `helm`.

**1. Start Jenkins and SonarQube**

```bash
cd infra
docker compose up -d --build
```

Jenkins runs on `http://localhost:8080`, SonarQube on `http://localhost:9000` (default login `admin` / `admin`).

**2. Configure Jenkins**

- Install the SonarQube Scanner and Docker Pipeline plugins
- Add a SonarQube server named `SonarQube` pointing at `http://sonarqube:9000` with an authentication token
- Add credentials: `dockerhub-creds` (Docker Hub username + access token) and `github-creds` (GitHub username + personal access token with `repo` scope)
- Set the Jenkins URL to `http://jenkins:8080/` so SonarQube can reach it
- Create a pipeline job pointing at this repository

**3. Add the SonarQube webhook**

In SonarQube, go to Administration → Configuration → Webhooks and add `http://jenkins:8080/sonarqube-webhook/`. Without this the quality gate stage waits indefinitely, because `waitForQualityGate` is notified by callback rather than polling.

**4. Deploy ArgoCD**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f argocd-app.yaml
```

Retrieve the admin password and open the UI:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

**5. Verify**

```bash
kubectl get pods
kubectl get deployment devops-pipeline-demo -o jsonpath="{.spec.template.spec.containers[0].image}"
kubectl port-forward svc/devops-pipeline-demo 8090:8080
curl http://localhost:8090/actuator/health
```

## Notes from building this

A few problems that took real debugging, kept here because they're the kind of thing that isn't obvious from documentation:

**The quality gate hung with no error.** `waitForQualityGate` doesn't poll SonarQube — it waits for SonarQube to call Jenkins back over a webhook. With no webhook configured, the stage sits on `PENDING` until it times out. Adding a `timeout` block around the scan stage turned a silent hang into a clear failure, which made the cause findable.

**`docker: not found` inside Jenkins.** The official Jenkins image ships without the Docker CLI, and installing the CLI alone isn't enough — a client needs a daemon to talk to. The fix is Docker-outside-of-Docker: install `docker.io` in a custom Jenkins image and mount the host's Docker socket, so Jenkins borrows the host daemon rather than running its own.

**Git refused to operate on the workspace.** Adding `user: root` to the Jenkins service changed the process owner, so Git saw a checkout owned by a different user and refused with `fatal: not in a git directory`. Clearing the workspace and adding a `safe.directory` entry resolved it.

**Pushing from a detached HEAD.** Jenkins checks out a specific commit rather than a branch, so the GitOps stage pushes with `HEAD:main` instead of `main`.

**NodePort isn't reachable on the host with kind.** The cluster nodes are themselves containers, so a NodePort is exposed inside them rather than on the host. `kubectl port-forward` is the way in during local development.
