## 🏗️ Architecture

A production-style GitOps pipeline on AWS EKS — Terraform for infrastructure, GitHub Actions for CI, ArgoCD for CD, and a full observability stack (ECK + Prometheus/Grafana) with Slack alerting.

```mermaid
flowchart TB
    subgraph LOCAL["💻 Local / Developer"]
        DEV[Developer]
        TF[Terraform]
    end

    subgraph GH["GitHub"]
        REPO[(Git Repo)]
        GHA[GitHub Actions]
        GHCR[(GHCR - Container Registry)]
    end

    subgraph CI["🔁 Continuous Integration"]
        CHECKOUT[Checkout]
        BUILD[Build]
        TRIVY[Trivy Scan]
        PUSH[Push Image]
    end

    subgraph AWS["☁️ AWS Account"]
        S3[(S3 - TF Remote State)]
        ROUTE53[Route 53]
        ACM[ACM - TLS Certs]
        ALB[ALB / Load Balancer Controller]

        subgraph VPC["VPC"]
            subgraph PUBLIC["Public Subnet"]
                IGW[Internet Gateway]
                NAT[NAT Gateway]
                BASTION[Bastion Host]
            end

            subgraph PRIVATE["Private Subnet"]
                subgraph EKS["EKS Cluster (Public Access Disabled)"]
                    ARGOCD[ArgoCD + Image Updater]
                    APP[App Pods + DB]
                    LOG[Logging: ECK Stack<br/>Elasticsearch / Kibana / Filebeat]
                    MON[Monitoring: Prometheus<br/>Grafana / Alertmanager]
                    IRSA[IRSA / Pod Identity]
                end
            end
        end
    end

    SLACK[Slack Alerts]
    USER[End User]

    DEV -->|Push Code| REPO
    DEV -->|Create Infra| TF
    TF -->|Provision| AWS
    TF -.->|Remote Backend| S3

    REPO -->|Trigger| GHA
    GHA --> CHECKOUT --> BUILD --> TRIVY --> PUSH
    PUSH -->|Push Image| GHCR

    GHCR -.->|Pull Image| ARGOCD
    ARGOCD -->|Sync / Reconcile| APP

    BASTION -->|kubectl / Helm| EKS

    USER -->|HTTPS| ROUTE53 --> ACM --> ALB --> APP
    APP --> LOG
    APP --> MON
    MON -->|Alerts| SLACK
    ARGOCD -->|Deploy Notifications| SLACK

    style CI fill:#e8f4fd,stroke:#2b6cb0
    style EKS fill:#e6fffa,stroke:#0d9488
    style AWS fill:#fff5f0,stroke:#dd6b20
```

### Pipeline Flow

1. **Infra Provisioning** — Terraform stands up the VPC, EKS cluster, IAM roles, ALB, Route 53, and ACM certs, with remote state in S3.
2. **CI (GitHub Actions)** — On push, the pipeline checks out code, builds the image, scans it with **Trivy**, and pushes it to **GHCR**.
3. **CD (ArgoCD)** — ArgoCD + Image Updater watches the registry/manifests and reconciles the cluster state automatically (GitOps — no manual `kubectl apply`).
4. **Traffic** — Route 53 → ACM (TLS) → ALB / Gateway API → services → pods running in a private subnet (no public EKS API access).
5. **Observability** — ECK stack (Elasticsearch, Kibana, Filebeat) handles logs; Prometheus, Grafana, and Alertmanager handle metrics and alerting.
6. **Notifications** — Deployment and alert events are pushed to **Slack** in real time.

### Tech Stack

| Layer | Tools |
|---|---|
| IaC | Terraform (S3 remote backend) |
| CI | GitHub Actions, Trivy (vulnerability scanning), GHCR |
| CD | ArgoCD, ArgoCD Image Updater |
| Orchestration | AWS EKS (private cluster), Kubernetes Gateway API |
| Networking | VPC, ALB, NAT/Internet Gateway, Route 53, ACM |
| Logging | ECK Stack (Elasticsearch, Kibana, Filebeat, ECK Operator) |
| Monitoring | Prometheus, Grafana, Alertmanager |
| Security | IRSA / Pod Identity, private EKS API endpoint |
| Alerting | Slack |
