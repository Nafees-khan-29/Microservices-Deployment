# Production-Grade GitOps Pipeline on AWS EKS

A self-built GitOps platform that takes a microservices application from `git push` to a running, monitored, TLS-secured production environment on AWS EKS — with zero manual `kubectl apply`.

Built to learn and demonstrate real-world DevOps tooling: Terraform, GitHub Actions, Trivy, ArgoCD, the Kubernetes Gateway API, and a full observability stack, wired together the way they'd actually be used on a team.

> 📦 The application deployed on top of this platform is Google's open-source [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) microservices demo (Apache 2.0). **The application code is not mine** — the infrastructure, CI/CD pipeline, GitOps workflow, security setup, and observability stack around it are what this project is about.

---

## What This Project Demonstrates

- Provisioning cloud infrastructure as code with **Terraform** (VPC, EKS, IAM, bastion host)
- A CI pipeline that only builds and scans what changed, using **GitHub Actions matrix builds** + **Trivy**
- **GitOps delivery** with ArgoCD — the cluster reconciles itself from Git, nobody runs `kubectl apply` by hand
- Automated image promotion via **ArgoCD Image Updater**
- Securing a Kubernetes control plane (private EKS API endpoint + bastion access)
- Routing and TLS termination with the **Gateway API**, ALB, Route 53, and ACM
- Full observability: logs (**ECK stack**), metrics/dashboards (**Prometheus + Grafana**), and alerting (**Alertmanager → Slack**)
- Autoscaling with a **Horizontal Pod Autoscaler**

---

## Architecture

![Architecture Diagram](./docs/images/arh.png)

### Pipeline Flow

1. **Infra Provisioning** — Terraform stands up the VPC (3 AZs), a private EKS cluster, IAM roles, and a bastion host, with remote state in S3.
2. **CI (GitHub Actions)** — A trigger workflow detects *which* microservices changed on push (via `git diff` on `src/**`) and fans out a matrix build — only changed services get rebuilt, not all eleven.
3. **Security scan** — Each image is scanned with **Trivy** for HIGH/CRITICAL CVEs before being pushed to **GHCR**.
4. **CD (ArgoCD)** — ArgoCD watches the repo and image tags, and reconciles the cluster automatically — `selfHeal` and `prune` are on, so drift gets corrected without manual intervention.
5. **Traffic** — Route 53 → ACM (TLS) → ALB via the Kubernetes **Gateway API** → services → pods, all inside a private subnet with no public EKS API access.
6. **Observability** — the ECK stack ships logs; Prometheus/Grafana/Alertmanager handle metrics, dashboards, and alerting.
7. **Notifications** — deployment and alert events are pushed to **Slack**.

---

## Design Decisions

- **Private EKS API endpoint + bastion host** — the control plane isn't reachable from the internet. All `kubectl`/`helm` access goes through a bastion in the public subnet, which is a pattern I wanted to practice rather than leaving the default public endpoint on.
- **ArgoCD over `kubectl apply` in CI** — pushing manifests directly from a pipeline means the pipeline needs cluster credentials, and there's no single source of truth for "what's actually running." GitOps flips that: the cluster pulls from Git, so Git is always the source of truth and CI never touches the cluster directly.
- **EKS over self-managed Kubernetes** — I wanted to focus on the CI/CD and GitOps layer rather than on managing `etcd` and control-plane upgrades myself.
- **Matrix builds keyed off changed paths** — with 11 microservices, rebuilding all of them on every push wastes CI minutes and time. Detecting the diff and only building what changed was a deliberate optimization.
- **Gateway API instead of classic Ingress** — used it to get hands-on with the newer, more expressive Kubernetes routing standard rather than the older Ingress resource.

---

## Tech Stack

| Layer | Tools |
|---|---|
| IaC | Terraform (`terraform-aws-modules` for VPC/EKS/EC2), S3 remote backend |
| CI | GitHub Actions (path-based matrix builds), Docker Buildx, Trivy, GHCR |
| CD | ArgoCD, ArgoCD Image Updater |
| Orchestration | AWS EKS 1.34 (private API endpoint), Kubernetes Gateway API |
| Networking | VPC (3 AZs), ALB, NAT/Internet Gateway, Route 53, ACM, external-dns |
| Logging | ECK Stack (Elasticsearch, Kibana, Filebeat) |
| Monitoring | Prometheus, Grafana, Alertmanager |
| Scaling | Horizontal Pod Autoscaler |
| Security | Bastion host, EKS Pod Identity, private control-plane endpoint |
| Alerting | Slack |
| Application | [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) — 11-service polyglot microservices demo (not written by me) |

---

## Repository Structure

```
.
├── terraform/              # VPC, EKS, bastion host, IAM
├── .github/workflows/      # CI: change detection + matrix build/scan/push
├── argocd/                 # ArgoCD Application + Image Updater config
├── gateway-api-manifests/  # Gateway API routing + ALB config
├── external-dns/           # external-dns IAM policy + Helm values
├── observability/          # Prometheus/Grafana/ECK Helm values + routes
├── scaling/                # HPA definitions
├── kubernetes-manifests/   # Per-service K8s manifests (Online Boutique)
├── helm-chart/             # Helm chart for the application
├── kustomize/              # Kustomize base + components (Online Boutique)
└── src/                    # Microservice source code (Online Boutique, not mine)
```

---

## Screenshots

| | |
|---|---|
| ![Online Boutique running live](./docs/images/online-boutique-live.png) | **App running live**<br>Traffic reaching the app through Route 53 → ACM → ALB, confirming DNS, TLS, and routing all work end to end. |
| ![Grafana dashboards list](./docs/images/grafana-dashboards-list.png) | **Grafana dashboards populated**<br>Kubernetes mixin dashboards auto-discovered and receiving live data — confirms Prometheus is actually scraping the cluster. |
| ![Grafana pod CPU metrics](./docs/images/grafana-pod-cpu-metrics.png) | **Per-pod resource metrics**<br>CPU usage, throttling, and quota for a running service pod, drilled down from the cluster-wide view. |
| ![Prometheus targets all UP](./docs/images/prometheus-targets-up.png) | **Prometheus scrape targets healthy**<br>Alertmanager, the API server, and other ServiceMonitors all reporting UP — the monitoring stack is wired correctly, not just installed. |
| ![Kibana Discover showing shipped logs](./docs/images/kibana-discover-logs.png) | **Centralized logging via ECK**<br>Filebeat shipping container logs from every microservice into Elasticsearch, searchable in Kibana Discover in real time. |

---

## What I Learned / Challenges

> *(A few notes from building this — worth expanding with your own specifics before publishing.)*

- Getting ArgoCD Image Updater to pick up new image tags reliably required tightening the `allowTags` regex to match the CI's `sha-<commit>` tagging scheme — without it, ArgoCD would either miss new builds or pick up unrelated tags.
- Locking down the EKS API to private-only meant every `kubectl`/`helm` operation had to go through the bastion host — a good forcing function for understanding SSH tunneling and security group scoping between the bastion and the cluster.
- Path-based matrix builds in GitHub Actions needed a `detect-changes` job that diffs `src/**` and feeds a dynamic JSON list into a reusable workflow — this was the trickiest part of the CI to get right.

---

## Status

This is a learning/portfolio project — built to get hands-on with production-style DevOps tooling, not a live production service. The infrastructure is not currently running (to avoid ongoing AWS costs); see the [Terraform](./terraform) and [ArgoCD](./argocd) configs to see exactly how it's provisioned and deployed. Screenshots of the running system (ArgoCD sync, Grafana dashboards, Slack alerts) are in [`/docs/images`](./docs/images).

---

## Future Improvements

- Multi-environment promotion (dev → staging → prod) instead of a single ArgoCD Application
- Move the demo's in-cluster databases to managed services (RDS) for real persistence
- Add Kubernetes NetworkPolicies / OPA-Gatekeeper for pod-to-pod security
- Secrets management via AWS Secrets Manager or Sealed Secrets instead of plain K8s secrets
- Cluster Autoscaler / Karpenter for node-level autoscaling alongside the existing HPA

---

## Credits

- Application code: [GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo) (Apache License 2.0)
- Everything else (infra, CI/CD, GitOps, observability, security setup): built by me as a learning project

## License

Apache License 2.0 — see [LICENSE](./LICENSE)