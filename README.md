# OpsGPT Helm Chart

This Helm chart deploys the OpsGPT microservices stack on **Azure Kubernetes Service (AKS)**.

## Chart Details

*   **Chart Name**: `opsgpt`
*   **Description**: Helm chart for OpsGPT microservices on Azure AKS
*   **Version**: `1.0.0`
*   **AppVersion**: `1.0.0`

---

## Architectural Configuration

### 1. Azure Key Vault & CSI Secret Provider
Secrets and environment variables are not hardcoded or defined in static manifests. Instead, they are dynamically loaded at runtime from **Azure Key Vault** using the **Secrets Store CSI Driver** and a `SecretProviderClass` built per service.
The CSI driver mounts the secrets under `/mnt/secrets-store` and syncs them to standard Kubernetes Secret resources so FastAPI services can consume them as environment variables via `valueFrom.secretKeyRef`.

### 2. Workload Identity
To authorize the pods to fetch secrets from Key Vault without hardcoded credentials:
*   Each service uses its own dedicated `ServiceAccount`.
*   The ServiceAccount is annotated with `azure.workload.identity/client-id`.
*   Pods are injected with the Workload Identity sidecar using the `azure.workload.identity/use: "true"` label.

### 3. Ingress Routing via AGIC
The entry point is configured through **Azure Application Gateway Ingress Controller (AGIC)**, using path-based routing:
*   `/` → `frontend-service` (port 3000)
*   `/api/core` → `core-api-service` (port 8001)
*   `/api/alerts` → `alert-ingestion-service` (port 8002)
*   `/api/analysis` → `ai-analysis-service` (port 8003)
*   `/api/notifications` → `notification-service` (port 8004)
*   `/alerts/webhook` → `alert-ingestion-service` (port 8002)

### 4. Stateless Design
The services are strictly stateless. No Persistent Volumes (PVs) or Persistent Volume Claims (PVCs) are managed by this chart.

### 5. Autoscaling (HPA) and Resource Limits
Each service is backed by an HPA using `autoscaling/v2` with target CPU utilization set to `70%`.
*   `dev`: 2 min / 4 max replicas
*   `prod`: 3 min / 10 max replicas

Resource requests and limits are configured as:
*   Requests: `100m` CPU, `128Mi` Memory
*   Limits: `500m` CPU, `512Mi` Memory

### 6. Pod Security Context
In compliance with requirements, no `podSecurityContext` or container `securityContext` parameters (such as `runAsNonRoot`, `readOnlyRootFilesystem`, etc.) are declared in the templates.
*   **Reason**: The backend and frontend containers are already pre-configured to run as the non-root system user `appuser` (or standard `nginx` user) in their respective Dockerfiles. Redundant K8s-level overrides are avoided.

---

## Compatibility Warning

> [!WARNING]
> **Health Probe Endpoints Compatibility**:
> This Helm chart configures `livenessProbe` and `readinessProbe` to hit `/healthz` and `/ready` paths.
> However, the microservices code currently only exposes a `GET /health` endpoint.
> Prior to deployment, you must either:
> 1. Update the application source code routes to support `/healthz` and `/ready`.
> 2. Customize `values.yaml` to override `probes.livenessPath` and `probes.readinessPath` to target `/health`.

---

## Deployment Documentation

Use the environment-specific values files to target different namespaces.

### Deploy to Development Environment
```bash
helm upgrade --install opsgpt-dev helm/opsgpt -n dev -f helm/opsgpt/values-dev.yaml --create-namespace
```

### Deploy to Production Environment
```bash
helm upgrade --install opsgpt-prod helm/opsgpt -n prod -f helm/opsgpt/values-prod.yaml --create-namespace
```
