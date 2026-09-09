# DevOps Pipeline Demo

A sample Java (Spring Boot) app used to build a full CI/CD + GitOps pipeline:

GitHub → Jenkins (build, test, SonarQube scan) → Docker Hub → Helm → ArgoCD → Kubernetes (dev/qa/stage/prod)

This is Phase 1: the app itself, containerized and Jenkins-ready. Later phases add Helm,
ArgoCD, and Kubernetes.

## Prerequisites

- Java 17 (`java -version`)
- Maven 3.9+ (`mvn -version`)
- Docker Desktop (or Docker Engine) running
- Git + a GitHub account

## Run it locally (no Docker)

```bash
mvn spring-boot:run
```

Visit `http://localhost:8080/api/hello` — you should get a JSON response.
Health check lives at `http://localhost:8080/actuator/health`.

## Run it in Docker

```bash
docker build -t devops-pipeline-demo:local .
docker run -p 8080:8080 devops-pipeline-demo:local
```

## Push this to GitHub

```bash
git init
git add .
git commit -m "Initial commit: sample app + Dockerfile + Jenkinsfile"
git branch -M main
git remote add origin https://github.com/<your-username>/devops-pipeline-demo.git
git push -u origin main
```

## Spin up Jenkins + SonarQube locally (Phase 2)

Since you don't have Kubernetes or cloud infra yet, start here — this gets your CI tools
running on your own machine in one command:

```bash
cd infra
docker compose up -d
```

- Jenkins: `http://localhost:8080` (get the initial admin password with
  `docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword`)
- SonarQube: `http://localhost:9000` (default login `admin` / `admin`)

In Jenkins you'll then:
1. Install the Docker Pipeline, Maven Integration, and SonarQube Scanner plugins.
2. Configure a Maven and JDK 17 tool under **Manage Jenkins > Tools**.
3. Add a SonarQube server under **Manage Jenkins > System** (name it `SonarQube` to match the Jenkinsfile).
4. Add a Docker Hub credential named `dockerhub-creds`.
5. Create a Pipeline job pointing at your GitHub repo — Jenkins will pick up the `Jenkinsfile` automatically.

> Note: Jenkins running on your laptop can't receive a GitHub webhook directly (no public
> URL). For now, use **Poll SCM** in the job config, or tunnel with `ngrok http 8080` if you
> want real webhooks.

## Roadmap (what comes next)

| Phase | What you'll add |
|---|---|
| 1 (done) | Java app, Dockerfile, Jenkinsfile skeleton |
| 2 | Local Jenkins + SonarQube running, first green pipeline run |
| 3 | Push built image to Docker Hub; add Nexus or JFrog for jar artifact storage |
| 4 | Add a Helm chart (`helm/` folder: deployment, service, configmap, secret templates) |
| 5 | Install Minikube, install ArgoCD, point it at your Helm chart repo |
| 6 | Wire Jenkins to bump `image.tag` in `values.yaml` and push — ArgoCD auto-syncs to the cluster |
| 7 | Add dev/qa/stage/prod as separate Helm value overlays or ArgoCD apps |

## Installing Minikube (when you're ready for Phase 5)

```bash
# macOS
brew install minikube

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

minikube start --cpus=4 --memory=8192
kubectl get nodes
```
