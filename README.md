# ML Platform IDP

An internal developer platform pipeline for deploying AI/ML workloads to Kubernetes. Built to make model deployment invisible to data scientists — they push code, the platform handles everything else.

## What This Does

Provides a complete CI/CD chain for containerized ML workloads:

- Builds and pushes Docker images to AWS ECR
- Scans for vulnerabilities with Trivy (warn in dev, hard fail in prod)
- Signs images with Cosign (Sigstore) for cryptographic integrity
- Writes immutable audit records to S3 for compliance traceability
- Triggers downstream deployment via Argo Events pipeline

## Architecture
```
Git push → GitHub Actions CI Pipeline
    ├── Build image → push to ECR
    ├── Trivy scan (CVE threshold: warn dev / hard fail prod)
    ├── Cosign image signing (Sigstore)
    ├── Audit record → S3 (immutable, object-locked)
    └── Repository dispatch → argo-event-pipeline
            └── Argo Events → Argo Workflow → Deploy to cluster
```
## Environment Strategy

| Environment | Trivy Threshold | Replicas | Debug |
|-------------|----------------|----------|-------|
| dev         | warn only      | 1        | true  |
| test        | warn only      | 2        | false |
| prod        | hard fail 7+   | 3        | false |

Environments are managed via Helmfile with separate values files per environment.

## Stack

- **CI/CD:** GitHub Actions
- **Registry:** AWS ECR (immutable tags, scan on push)
- **Image signing:** Cosign (Sigstore)
- **Deployment:** Helm + Helmfile
- **Event-driven deployment:** Argo Events + Argo Workflows
- **Audit trail:** S3 with structured JSON records
- **Authentication:** OIDC (no long-lived credentials)
- **Application:** Django + Gunicorn

## Pipeline Flow

**1. Build and Push**
Every merge to main triggers a Docker build. The image is tagged with the Git SHA for immutability and pushed to ECR.

**2. Trivy Scan**
Trivy scans the pushed image for HIGH and CRITICAL CVEs. In dev and test environments this is a warning only. In production this is a hard fail that blocks deployment.

**3. Cosign Signing**
Every image is signed with Cosign after passing the scan. The signature is stored alongside the image in ECR. Kubernetes admission controllers can verify signatures before allowing pods to run.

**4. Audit Log**
A structured JSON record is written to S3 for every pipeline run regardless of outcome. Records include timestamp, actor, image digest, and status of each job. S3 object lock ensures tamper-evidence for GxP compliance.

**5. Argo Events Trigger**
On successful signing, a repository dispatch event fires to the argo-event-pipeline repo. The image tag travels with the payload so the downstream pipeline deploys the exact signed artifact.

## Helmfile Environments

Deploy to a specific environment:

```bash
helmfile -e dev apply
helmfile -e test apply
```

## Local Development

Requirements:
- Docker
- kind
- kubectl
- Helm
- Helmfile

```bash
kind create cluster --name ml-platform
docker build -t ml-platform-idp:latest .
kind load docker-image ml-platform-idp:latest --name ml-platform
helmfile -e dev apply
kubectl get pods -n ml-platform-dev
```

Health check:
```bash
kubectl port-forward svc/ml-platform 8000:8000 -n ml-platform-dev
curl http://localhost:8000/health/
```

## Design Decisions

**Why GitHub Actions over Argo Events for CI?**
Argo Events handles event-driven deployment inside the cluster. GitHub Actions handles the build and scan layer outside the cluster where source code lives. Keeping CI outside the cluster and CD inside is the correct separation of concerns. The two connect via repository dispatch.

**Why ECR over Docker Hub?**
ECR is private by default, integrates natively with AWS IAM and OIDC, supports immutable tags, and scans on push automatically. For a workload running on EKS, ECR removes the cross-cloud dependency entirely.

**Why immutable image tags?**
Every image is tagged with the Git SHA. Once pushed, that tag cannot be overwritten. In a GxP context this is non-negotiable — you need to prove that the exact artifact deployed to a validated environment is the one that was tested. Mutable tags like `latest` make that impossible.

**Why Cosign over manual signing?**
Cosign integrates directly into the GitHub Actions pipeline with no key management overhead when using keyless signing via Sigstore's public infrastructure. The signature is stored in ECR alongside the image. Any admission controller can verify it without additional tooling.

**Why warn on Trivy in dev but hard fail in prod?**
Dev is a learning environment. Blocking every CVE in dev creates friction that slows down data scientists without improving security. The hard fail belongs at the prod gate where validated artifacts are promoted to customer-facing environments.

**Why S3 for audit records over a database?**
S3 with object lock is WORM storage — write once, read many. Once a record is written it cannot be modified or deleted until the retention period expires. That's the tamper-evidence property required for GxP audit trails. A database can be modified. S3 object lock cannot.

**Why Helmfile over raw Helm?**
Helm manages individual chart releases. Helmfile manages multiple releases across multiple environments with environment-specific values files. Without Helmfile, deploying to dev versus test requires manual values overrides and is error-prone. Helmfile makes environment promotion declarative and repeatable.

**Why OIDC over long-lived AWS credentials?**
Long-lived credentials stored in GitHub secrets are a security liability — they never expire, can be leaked in logs, and are difficult to rotate. OIDC federation issues short-lived tokens scoped to the specific workflow run. No credentials to rotate, no credentials to leak.

---

## Troubleshooting

**Cosign sign fails with 401**
ECR requires a fresh auth token for Cosign. Make sure the Login to ECR step runs before the sign step and that your OIDC role has `ecr:GetAuthorizationToken` permission.

**Trivy scan fails with image not found**
The scan job pulls the image from ECR. If the build job failed or the image wasn't pushed, the scan has nothing to pull. Check the build job logs first.

**Repository dispatch not triggering argo-event-pipeline**
Verify `GH_PAT` secret is set in `ml-platform-idp` repo and has `repo` scope. Check the audit job logs for the curl response code. A 204 means success. A 401 means the token is invalid or missing scope.

**Helmfile apply fails with diff plugin error**
Helmfile requires the helm-diff plugin. Install it:
```bash
helm plugin install https://github.com/databus23/helm-diff
```

**Pod stuck in CrashLoopBackOff after deploy**
Check logs first:
```bash
kubectl logs <pod-name> -n ml-platform-dev
```
Common causes: ALLOWED_HOSTS not set for Kubernetes probe IPs, Gunicorn can't find the Django module (check WORKDIR in Dockerfile), missing environment variables.

**Health probe returning 400**
Django is rejecting the probe request due to ALLOWED_HOSTS. Set `ALLOWED_HOSTS = ['*']` in settings for dev. In production scope this to the cluster's internal IP range.

**kind cluster not found after restart**
kind clusters don't persist across Docker Desktop restarts. Recreate:
```bash
kind create cluster --name ml-platform
kind load docker-image ml-platform-idp:latest --name ml-platform
helmfile -e dev apply
```

**Cosign keyless signing fails in air-gapped environment**
Keyless signing requires access to Sigstore's public Fulcio and Rekor infrastructure. In air-gapped or private environments, deploy a private Sigstore instance or switch to key-based signing with a key stored in AWS KMS.

**S3 audit write fails with access denied**
Your OIDC role needs `s3:PutObject` on the audit bucket path. Verify the IAM policy includes:
```json
{
  "Effect": "Allow",
  "Action": ["s3:PutObject"],
  "Resource": "arn:aws:s3:::gitops-infra-pipeline-tfstate/ml-platform-audit/*"
}
```

## GxP Compliance Notes

This pipeline is designed with GxP-regulated environments in mind:

- Immutable image tags prevent overwriting validated artifacts
- Cosign signatures provide cryptographic proof of image integrity from build to deployment
- S3 audit records with object lock satisfy tamper-evidence requirements
- OIDC authentication eliminates long-lived credentials from the audit surface
- Trivy thresholds are environment-specific — dev is permissive, prod is strict

In a fully validated environment, Trivy scan reports and Cosign signatures would be retained alongside deployment records as part of the IQ/OQ evidence package.

## Related Projects

- [argo-event-pipeline](https://github.com/SmartBrisco/argo-event-pipeline) — Event-driven deployment pipeline
- [gitops-infra-pipeline](https://github.com/SmartBrisco/gitops-infra-pipeline) — Infrastructure provisioning via Terraform
- [platform-observability](https://github.com/SmartBrisco/platform-observability) — Unified observability stack
- [namespace-provisioner](https://github.com/SmartBrisco/namespace-provisioner) — Kubernetes operator for namespace governance
- [Internal-Developer-Platform](https://github.com/SmartBrisco/Platform) — Bootstrap all projects with a single command
