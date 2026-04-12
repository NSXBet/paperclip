# Paperclip Deployment (NSXBet Fork)

## Prerequisites

### AWS IAM Role

Create an OIDC-based IAM role for GitHub Actions with ECR push permissions:

- **Role name**: e.g. `GithubPaperclip`
- **Trust policy**: GitHub OIDC provider for `NSXBet/paperclip`
- **Permissions**: `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:CreateRepository`, `ecr:DescribeRepositories`

Set the role ARN as a repository variable:
```bash
gh variable set AWS_ROLE_ARN --repo NSXBet/paperclip --body "arn:aws:iam::492684252576:role/GithubPaperclip"
```

### ECR Repositories

Created automatically by CI on first push:
- `paperclip` (Docker image)
- `charts/paperclip` (Helm chart OCI)

### Secrets (External Secrets)

Store the following keys in your secret manager (referenced by `ClusterSecretStore: gcp-backend`):
- `DATABASE_URL` - PostgreSQL connection string
- `BETTER_AUTH_SECRET` - Auth secret for Better Auth

## CI Pipelines

| Workflow | Trigger | What it does |
|---|---|---|
| `docker-build.yml` | Push to `master` (app code) | Build + push Docker image to ECR |
| `helm-push.yml` | Push to `master` (`charts/`) | Package + push Helm chart to ECR OCI |
| `sync-upstream.yml` | Weekly (Monday 09:00 UTC) or manual | Sync fork with upstream, rebase custom commits on top |

## Helm Install

```bash
# Add ECR credentials for Helm
aws ecr get-login-password --region us-east-1 | \
  helm registry login --username AWS --password-stdin 492684252576.dkr.ecr.us-east-1.amazonaws.com

# Install
helm install paperclip oci://492684252576.dkr.ecr.us-east-1.amazonaws.com/charts/paperclip \
  --version 0.1.0 \
  --namespace paperclip \
  --create-namespace
```

## Values Override Examples

### Production (external RDS, ExternalSecret for secrets)
```yaml
image:
  tag: "<git-sha>"

replicaCount: 2

externalSecret:
  enabled: true
  key: "production/paperclip"

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

ingress:
  enabled: true
  host: paperclip.nsxbet.com
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
```

### Dev/Staging (bundled PostgreSQL)
```yaml
postgresql:
  enabled: true

externalSecret:
  enabled: false

persistence:
  size: 5Gi
```
