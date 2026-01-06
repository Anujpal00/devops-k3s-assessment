# DevOps K3s Assessment - Open WebUI Deployment

## Table of Contents
1. [VM & Cluster Setup](#vm--cluster-setup)
2. [Docker Installation](#docker-installation)
3. [k3s Installation](#k3s-installation)
4. [Helm Deployment](#helm-deployment)
5. [Verification & Debugging](#verification--debugging)
6. [OIDC Root Cause Analysis](#oidc-root-cause-analysis)
7. [Final Outputs](#final-outputs)
8. [Ownership Questions](#ownership-questions)
9. [Extra Credit: Custom CA Trust](#extra-credit-custom-ca-trust)
10. [References](#references)

---

## VM & Cluster Setup

- Single VM used for deployment: Ubuntu 22.04 LTS
- Lightweight single-node Kubernetes cluster via **k3s**.
- Docker installed for container runtime.

### Why this setup is suitable
For an early-stage startup, this setup is ideal because:
- **Lightweight** and fast to deploy.
- Minimal resource usage.
- Easy to test and prototype applications.
- Allows single developer to manage cluster without overhead.

---

## Docker Installation

- Installed Docker Engine:
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```
- Verified installation:
```bash
docker version
docker ps
```
- User added to Docker group for non-root usage:
```bash
sudo usermod -aG docker $USER
```

---

## k3s Installation

- Installed single-node k3s:
```bash
curl -sfL https://get.k3s.io | sh -
```
- Verified cluster status:
```bash
sudo kubectl get nodes
kubectl get pods -A
```

---

## Helm Deployment

- Added Helm repo and updated:
```bash
helm repo add open-webui https://open-webui.github.io/helm-charts
helm repo update
```
- Initial deployment with OIDC enabled:
```bash
helm upgrade webui open-webui/open-webui \
  --namespace openwebui \
  --values values-oidc.yaml
```
- Disabled OIDC after verification issues:
```bash
helm upgrade webui open-webui/open-webui \
  --namespace openwebui \
  --set oidc.enabled=false
```

---

## Verification & Debugging

- Observed pod status:
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
- Logs inspection:
```bash
kubectl logs -n openwebui statefulset/open-webui
```
- Observation:
  - Backend services and database migrations ran successfully.
- Warning:
```
WARNING: CORS_ALLOW_ORIGIN IS SET TO '*'
```
- Browser returned HTTP 501 due to OIDC requirement.

---

## OIDC Root Cause Analysis

- Root Cause:
  - OIDC (OpenID Connect) enabled without a valid client secret or proper domain configuration.
  - Frontend requires a trusted OIDC issuer; missing configuration caused browser verification to fail.
  - All backend pods were healthy, so the system was running, but browser access was blocked.
- Fix Applied:
  - Disabled OIDC temporarily to complete assessment:
```bash
helm upgrade webui open-webui/open-webui \
  --namespace openwebui \
  --set oidc.enabled=false
```

---

## Final Outputs

- Nodes
```bash
kubectl get nodes
```
```
NAME             STATUS   ROLES           AGE   VERSION
anujpal1213145   Ready    control-plane   20m   v1.34.3+k3s1
```
- All resources in openwebui namespace
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

---

## Ownership Questions

1. **Production Readiness**
   - Top 5 risks:
     - OIDC misconfiguration
     - Insufficient resource limits
     - No automated backups
     - Security misconfiguration
     - Network/firewall issues
   - First 2 fixes:
     - Configure proper OIDC client secret
     - Add resource limits and monitoring

2. **Failure Scenario**
   - Traffic spikes 10x and node down at 2 AM:
     - Breaks first: Frontend / ingress
     - Recovery: Scale replicas or redeploy pod on a new node
     - Next day: Implement autoscaling & monitoring alerts

3. **Security & Secrets**
   - Manage secrets via Helm values, Kubernetes secrets, or Vault
   - Never commit secrets to Git
   - Rotate secrets regularly

4. **Backups & Recovery**
   - Backup: Databases, configuration, Helm values
   - Frequency: Daily
   - Test recovery by restoring to a test namespace

5. **Cost Ownership (Hetzner)**
   - Keep costs low: Use small VM, single-node k3s
   - Avoid: Multi-node clusters early
   - Move away from k3s: When production requires multi-node HA or heavy traffic

---

## Extra Credit: Custom CA Trust

- Add the CA certificate to OS trust store or mount the certificate in the container.
- Configure application to trust this CA for HTTPS connections.
- Maintainability: Keep a single folder of trusted CAs, and reference it in container configs.

---

## References

- [Open WebUI Helm Chart](https://open-webui.github.io/helm-charts)
- [Open WebUI Docs](https://docs.openwebui.com/)
- [k3s Official Documentation](https://k3s.io/)
- [Docker Documentation](https://docs.docker.com/)
