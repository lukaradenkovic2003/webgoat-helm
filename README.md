# webgoat-helm

Helm chart and GitOps configuration for deploying the WebGoat application to Amazon EKS via **ArgoCD**. This repo is the single source of truth for what's currently running on the cluster — any change here (manual or automated via the `WebGoat` app pipeline) results in an automatic sync to EKS.

Third of the three repositories in the project:

| Repo | Role |
|---|---|
| [webgoat-infra](https://github.com/lukaradenkovic2003/webgoat-infra) | AWS infrastructure (Terraform) |
| [WebGoat](https://github.com/lukaradenkovic2003/WebGoat) | Application code + DevSecOps CI pipeline |
| **webgoat-helm** (this repo) | Helm chart + ArgoCD GitOps + DAST |

## Structure

```
.
├── Chart.yaml                # Helm chart metadata
├── values.yaml                 # Configuration (image tag, resources, ingress host...)
├── argocd-app.yaml               # ArgoCD Application resource (self-referencing GitOps)
├── templates/
│   ├── deployment.yaml               # WebGoat Deployment (env vars, probes, resources)
│   ├── service.yaml                     # ClusterIP service
│   ├── ingress.yaml                        # AWS ALB Ingress (internet-facing)
│   └── webgoat-zap-scan-job.yaml              # Post-deploy DAST job (OWASP ZAP)
└── .helmignore
```

## GitOps flow (ArgoCD)

`argocd-app.yaml` defines an ArgoCD `Application` resource that:

- Watches this repo (`repoURL: webgoat-helm`, `targetRevision: HEAD`, `path: .`)
- Deploys into the `webgoat` namespace on the EKS cluster (auto-creates the namespace: `CreateNamespace=true`)
- Has `automated` sync enabled with `prune: true` and `selfHeal: true` — every change in Git is automatically applied to the cluster, and any manual change made directly on the cluster (drift) is automatically reverted back to the Git state
- Passes `ingress.cloudflareSecret` as a Helm parameter (overriding the value in `values.yaml`)

**Flow from code to production:**
1. A developer pushes a change to the `WebGoat` repo
2. The CI pipeline in the `WebGoat` repo runs SAST/SCA/image scans, pushes a new image to ECR, and updates `tag:` in this repo's `values.yaml`
3. ArgoCD (which continuously watches this repo) detects the change and automatically syncs the new version to EKS
4. A post-deploy hook runs a DAST scan against the live endpoint

## Deployment (`templates/deployment.yaml`)

Key points:
- `env`: `WEBGOAT_PORT=8080`, `WEBWOLF_PORT=9090` — **mandatory**, since the WebGoat Spring Boot application reads its port from these variables (`${webgoat.port}`); without them the application fails to start (`APPLICATION FAILED TO START`)
- `readinessProbe` / `livenessProbe` on `/WebGoat/login` (port 8080) — ensure Kubernetes (and, through it, the ALB Target Group) only treats a pod as "Ready" once the application actually responds, not just once the container is running
- Resources (`requests`/`limits`) configurable via `values.yaml`

## Ingress (`templates/ingress.yaml`)

Uses the **AWS Load Balancer Controller** (ALB Ingress, `alb.ingress.kubernetes.io/*` annotations):
- `scheme: internet-facing`, `target-type: ip` (traffic goes directly to pod IPs, not through NodePort)
- HTTP → HTTPS redirect (`ssl-redirect`)
- Health check on `/WebGoat/login`, success codes `200-399`
- Host-based routing on `{{ .Values.ingress.host }}` (`webgoat-devsecops.xyz`)

## DAST — OWASP ZAP (`templates/webgoat-zap-scan-job.yaml`)

A Kubernetes `Job` with Helm hook annotations `post-install,post-upgrade` — automatically runs **after every ArgoCD sync** (new install or release upgrade):
- Uses the `ghcr.io/zaproxy/zaproxy:stable` image
- Waits 15s for the service to become available, then runs `zap-baseline.py` against `http://<service>/WebGoat` (baseline scan, passive runtime vulnerability analysis)
- `hook-delete-policy: before-hook-creation` — the previous job is deleted before each new run
- `ttlSecondsAfterFinished: 3600` — the job is automatically cleaned up an hour after finishing

You currently view scan results via `kubectl logs job/webgoat-zap-scan -n webgoat`. To persist reports (e.g. an HTML report as an artifact), extend the job to upload the result to S3 or similar — the current version only prints to stdout/pod logs.

## Edge Security (Cloudflare + AWS)

Although the Cloudflare and security-group configuration itself lives in the `webgoat-infra` repo (`cloudflare.tf`), it directly affects this chart:

- **Cloudflare proxy** in front of the ALB — the DNS CNAME record (`webgoat-devsecops.xyz`) is proxied (orange cloud), so clients never see the real ALB IP address
- **Origin hardening** — the AWS Security Group on the ALB should accept traffic only from official Cloudflare IP ranges; this prevents bypassing Cloudflare and reaching the ALB directly via its DNS name or IP
- **Geo-blocking** is defined through Cloudflare WAF rules (`ip.geoip.country`) in `webgoat-infra`, not in this repo

`values.yaml`'s `ingress.cloudflareSecret` field is used as a shared value (e.g. a custom header the ALB checks) to confirm that a request actually came through Cloudflare, rather than hitting the origin directly.

## `values.yaml` — key parameters

| Parameter | Description |
|---|---|
| `image.repository` / `image.tag` | ECR image path; `tag` is updated automatically by the `WebGoat` CI pipeline |
| `replicaCount` | Number of replicas (currently 1) |
| `service.port` / `service.targetPort` | 80 → 8080 (service-to-containerPort mapping) |
| `resources` | CPU/memory requests and limits |
| `ingress.host` | Public domain (`webgoat-devsecops.xyz`) |
| `ingress.cloudflareSecret` | Value used to verify traffic came through Cloudflare |

## Installing ArgoCD on EKS (one-time)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f argocd-app.yaml
```

After this, ArgoCD takes over managing the application automatically, according to the `syncPolicy.automated` settings.

## Manual sync / troubleshooting

```bash
kubectl get pods -n webgoat
kubectl get ingress -n webgoat
kubectl logs -n webgoat <pod-name>
kubectl logs -n webgoat job/webgoat-zap-scan
```

In the ArgoCD UI: **Refresh** pulls the latest state from Git, **Sync** applies it if auto-sync is lagging.
