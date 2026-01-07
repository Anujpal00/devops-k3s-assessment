# DevOps K3s Assessment - Open WebUI Deployment

## Table of Contents

- [VM & Cluster Setup](#vm--cluster-setup)
- [Docker Installation](#docker-installation)
- [k3s Installation](#k3s-installation)
- [Helm Deployment](#helm-deployment)
- [Verification & Debugging](#verification--debugging)
- [OIDC Root Cause Analysis](#oidc-root-cause-analysis)
- [Final Outputs](#final-outputs)
- [Ownership Questions](#ownership-questions)
- [Extra Credit: Custom CA Trust](#extra-credit-custom-ca-trust)
- [References](#references)

## VM & Cluster Setup

Single VM used for deployment: Ubuntu 24 LTS

Lightweight single-node Kubernetes cluster via k3s

Docker installed for container runtime

### Why this setup is suitable

For an early-stage startup, this setup is ideal because:

- Lightweight and fast to deploy
- Minimal resource usage
- Easy to test and prototype applications
- Allows single developer to manage cluster without overhead

## Docker Installation

Installed Docker Engine:

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```

Verified installation:

```bash
docker version
docker ps
```

User added to Docker group for non-root usage:

```bash
sudo usermod -aG docker $USER
```

Logout/login or newgrp docker to apply group changes

## k3s Installation

Installed single-node k3s:

```bash
curl -sfL https://get.k3s.io | sh -
```

Verified cluster status:

```bash
sudo kubectl get nodes
kubectl get pods -A
```

## Helm Deployment

Added Helm repo and updated:

```bash
helm repo add open-webui https://open-webui.github.io/helm-charts
helm repo update
```

Initial deployment with OIDC enabled:

```bash
helm upgrade webui open-webui/open-webui \
  --namespace openwebui \
  --values values-oidc.yaml
```

Disabled OIDC temporarily (due to missing domain):

```bash
helm upgrade webui open-webui/open-webui \
  --namespace openwebui \
  --set oidc.enabled=false
```

## Verification & Debugging

Observed pod status:

```bash
kubectl get pods -n openwebui
```

```
NAME                                   READY   STATUS    RESTARTS   AGE
open-webui-0                           1/1     Running   0          15m
open-webui-ollama-xxxxx               1/1     Running   0          15m
open-webui-pipelines-xxxxx             1/1     Running   0          15m
open-webui-redis-xxxxx                 1/1     Running   0          15m
```

Logs inspection:

```bash
kubectl logs -n openwebui statefulset/open-webui
```

Observation:

Backend services and database migrations ran successfully.

Browser returned errors when OIDC was enabled with invalid/missing domain.

## OIDC Root Cause Analysis

Root Cause:

OIDC (OpenID Connect) enabled without a valid clientSecret or proper domain configuration.

Frontend requires a trusted OIDC issuer; missing configuration caused browser verification to fail.

Backend pods were healthy, so the system ran, but frontend authentication failed.

Debugging Steps:

Verified pods were running:

```bash
kubectl get pods -n openwebui
```

Checked logs for errors:

```bash
kubectl logs -n openwebui statefulset/open-webui
```

Observation: No crash; backend healthy

Confirmed environment variables for OIDC:

```bash
kubectl exec -n openwebui open-webui-0 -- env | grep -E "OIDC|AUTH"
```

Observation: Correct values loaded, but domain invalid

Verified frontend by curl:

```bash
curl -i http://localhost:3002/api/v1/users
```

Observation: Returns HTML splash, OIDC not functional

Identified missing valid domain as root cause

Fix Applied:

Since we do not have a valid domain name for the OIDC issuer, the authentication cannot work. To make the application accessible for assessment purposes, we temporarily disabled OIDC:

```bash
helm upgrade webui open-webui/open-webui \
  --namespace openwebui \
  --set oidc.enabled=false
```

Explanation:

Disabling OIDC allows the frontend to bypass authentication and run normally.

The root cause is absence of a valid OIDC domain, not a Kubernetes or pod issue.

In a real scenario, a proper domain with a valid OIDC provider (e.g., Keycloak) and correct clientId/clientSecret would be required:

```
issuer: "https://auth.example.com/auth/realms/hyperplane/.well-known/openid-configuration"
clientId: "test-client"
clientSecret: "<secure-secret>"
```

## Final Outputs

Nodes:

```bash
kubectl get nodes
```

```
NAME             STATUS   ROLES           AGE   VERSION
anujpal1213145   Ready    control-plane   20m   v1.34.3+k3s1
```

All resources in openwebui namespace:

```bash
kubectl get all -n openwebui
```

```
NAME                                   READY   STATUS    RESTARTS   AGE
open-webui-0                           1/1     Running   0          20m
open-webui-ollama-xxxxx               1/1     Running   0          20m
open-webui-pipelines-xxxxx             1/1     Running   0          20m
open-webui-redis-xxxxx                 1/1     Running   0          20m
```

## Ownership Questions

### Production Readiness

Top 5 risks:

- OIDC misconfiguration
- Insufficient resource limits
- No automated backups
- Security misconfiguration
- Network/firewall issues

First 2 fixes:

- Configure proper OIDC client secret
- Add resource limits and monitoring

### Failure Scenario

Traffic spikes 10x and node down at 2 AM:

Breaks first: Frontend / ingress

Recovery: Scale replicas or redeploy pod on a new node

Next day: Implement autoscaling & monitoring alerts

### Security & Secrets

Manage secrets via Helm values, Kubernetes secrets, or Vault

Never commit secrets to Git

Rotate secrets regularly

### Backups & Recovery

Backup: Databases, configuration, Helm values

Frequency: Daily

Test recovery by restoring to a test namespace

### Cost Ownership (Hetzner)

Keep costs low: Use small VM, single-node k3s

Avoid: Multi-node clusters early

Move away from k3s: When production requires multi-node HA or heavy traffic

## Extra Credit: Custom CA Trust

Add the CA certificate to OS trust store or mount the certificate in the container.

Configure application to trust this CA for HTTPS connections.

Maintainability: Keep a single folder of trusted CAs, and reference it in container configs.

## References

- [Open WebUI Helm Chart](https://open-webui.github.io/helm-charts)
- [Open WebUI Docs](https://docs.openwebui.com/)
- [k3s Official Documentation](https://k3s.io/)
- [Docker Documentation](https://docs.docker.com/)
