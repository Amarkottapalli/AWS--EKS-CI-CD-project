# 2048 Game — CI/CD Deployment on AWS EKS
## Overview
This project implements a complete CI/CD pipeline that takes source code for a containerized 2048 game application and deploys it to a production-style environment on **AWS EKS (Fargate)**. The pipeline automates the full path from code commit to a live, monitored application: build → static code analysis → quality gate → containerization → GitOps-based deployment → observability.

The goal was to replicate a real-world DevOps workflow — not just deploy an app, but build the automated, quality-gated pipeline that a team would actually use to ship it.

## Architecture

```
Developer Push → Jenkins (EC2) → Maven Build → SonarQube Analysis → Quality Gate
     → Docker Build & Push (DockerHub) → ArgoCD (GitOps Sync) → AWS EKS (Fargate)
     → ALB Ingress → Live App
                                      ↓
                        Prometheus + Grafana (Observability)
```

- **CI (Jenkins):** Pulls code, builds with Maven, runs SonarQube analysis, enforces a quality gate, then builds and pushes a Docker image.
- **CD (ArgoCD):** Watches the Kubernetes manifests repo and auto-syncs any change to the EKS cluster — a GitOps pattern, so the cluster state always matches what's in Git rather than being deployed manually.
- **Runtime (EKS Fargate):** Pods run serverless on Fargate (no EC2 worker nodes to manage), exposed externally via an ALB Ingress.
- **Observability (Prometheus/Grafana):** Cluster and app metrics scraped and visualized in real time.

## Tools Used
| Category | Tool |
|---|---|
| CI/CD Orchestration | Jenkins |
| Build | Maven |
| Code Quality | SonarQube |
| Containerization | Docker, DockerHub |
| GitOps CD | ArgoCD |
| Container Orchestration | AWS EKS (Fargate) |
| Ingress | ALB Ingress Controller |
| IAM | IRSA / OIDC |
| Monitoring | Prometheus, Grafana |

## Pipeline Walkthrough

### 1. Jenkins Pipeline — End-to-End Build
Pipeline #27 completing all 7 stages successfully in 6 min 18 sec: Checkout Code → Build (Maven) → SonarQube Analysis → Quality Gate → Build & Push Docker Image → Deploy (ArgoCD).
<img width="748" height="359" alt="jenkins pipeline" src="https://github.com/user-attachments/assets/9699883e-b027-4ddc-a027-028f2f90061b" />


### 2. SonarQube — Code Quality Gate
Static analysis on the 2048 Game project (1.7k lines of Java) passed the quality gate cleanly: 0 bugs, 0 vulnerabilities, 0 code smells, 88.4% test coverage, 1.6% duplication.
<img width="869" height="365" alt="sonarqube" src="https://github.com/user-attachments/assets/7de1ba0e-9155-4b30-94c5-fa35f2e449f8" />


### 3. Docker Image — Built and Pushed to DockerHub
Image pushed to `AMAR/2048-game-project:latest` (145.31 MB) as part of the automated pipeline stage.
<img width="741" height="276" alt="docker repo" src="https://github.com/user-attachments/assets/de43634c-423e-4ab3-b56d-50f1caecb4f6" />


### 4. ArgoCD — GitOps Sync to EKS
ArgoCD tracking the `2048-game-project` application from its Git source, showing **Healthy** and **Synced** status against the `k8s` manifest path.
<img width="869" height="279" alt="argo cd" src="https://github.com/user-attachments/assets/eb1b83ee-823b-4c9e-bf26-f8c42b3a4a8e" />


### 5. Verifying the Deployment on EKS
Confirming pods, service, and ingress are correctly provisioned on the Fargate-backed cluster:
```
kubectl get pods
kubectl get svc
kubectl get ingress
```
Both replicas `Running` with 0 restarts, ClusterIP service exposed on port 80, and ALB ingress address resolved.
<img width="518" height="309" alt="kub ctl cmd" src="https://github.com/user-attachments/assets/c5c58ee9-93be-461b-a464-8a67f4eb50a2" />


### 6. Application Live via ALB
The app accessible externally through the AWS Application Load Balancer DNS endpoint, confirming the full path from Git commit to a publicly reachable service.
<img width="515" height="307" alt="project deploymentdeployment " src="https://github.com/user-attachments/assets/377eb4ef-8e8b-44c9-9a7f-0cc787e9caeb" />


### 7. Observability — Prometheus & Grafana
Real-time dashboard tracking CPU usage, memory usage, request rate (RPS), HTTP response time, pod restart count, and error rate — confirming the app is not just running, but observable in production style.
<img width="578" height="311" alt="grafana dashboard " src="https://github.com/user-attachments/assets/a7f25a93-42fd-437f-bd43-2b6801dfd9f7" />


## Challenges & Fixes

- **ALB Ingress not provisioning / stuck pending:** Usually caused by the ALB Ingress Controller's IAM role missing the correct IRSA/OIDC-linked policy, or the subnets not being tagged correctly (`kubernetes.io/role/elb`) for AWS to discover them.
- **Fargate pods stuck in `Pending`:** Typically means no Fargate profile matches the pod's namespace/labels — fixed by creating/adjusting the Fargate profile selector.
- **ArgoCD showing `OutOfSync` instead of `Synced`:** Usually a mismatch between the manifests repo and what's applied — resolved with a manual sync or by checking the target revision/path config.
- **SonarQube Quality Gate failing on first run:** Common on first analysis if coverage thresholds aren't met — resolved by adding/adjusting test coverage until the gate passed.
- **Jenkins pipeline failing at the Docker push stage:** Usually DockerHub credentials not configured correctly in Jenkins credentials store, or missing `docker login` step.

## What I'd Do Differently
- Automate Fargate profile and IAM role creation via Terraform instead of manual `eksctl`/console steps, for full infrastructure-as-code reproducibility.
- Add Slack/email notifications on pipeline failure for faster feedback.
- Set up Prometheus alerting rules (not just dashboards) for proactive incident response.
- ## Attribution
Base project structure and configs adapted from [Abhishek Veeramalla's aws-devops-zero-to-hero](https://github.com/iam-veeramalla/aws-devops-zero-to-hero/tree/main/day-22) (Apache-2.0 License — see `LICENSE`). Deployed, configured, and validated independently on my own AWS account. All screenshots below are from my own pipeline runs and live cluster.


## License
The base project structure is licensed under Apache-2.0 (see `LICENSE`). This repository includes my own execution, configuration, and documentation built on top of that base.
