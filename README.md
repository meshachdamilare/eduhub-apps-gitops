# EduHub Apps GitOps Repository

This repository contains the **GitOps configuration** for deploying all EduHub microservices to Kubernetes using **Argo CD**, **OCI-based Helm charts**, and **External Secrets Operator (ESO)**.

It acts as the **single source of truth** for how every EduHub service is deployed in each environment (**DEV** and **PROD**).

The repository stores:

- Environment-specific Helm `values.yaml`
- `ExternalSecret` manifests (Azure Key Vault → Kubernetes Secrets)
- `ClusterSecretStore` configuration
- Argo CD `ApplicationSet` definitions
- Argo CD ACR pull credentials
- Per-service GitOps deployment logic

All deployments are fully automated and reconciled continuously by Argo CD.


### Repository Structure

```text
.
├── applications
│   ├── dev
│   │   ├── argocd-secrets/
│   │   │   └── argocd-pull-secret.yaml
│   │   ├── eduhub-assignment/
│   │   │   ├── external-secret.yaml
│   │   │   └── values.yaml
│   │   ├── eduhub-auth/
│   │   │   ├── external-secret.yaml
│   │   │   └── values.yaml
│   │   ├── eduhub-catalog/
│   │   │   ├── external-secret.yaml
│   │   │   └── values.yaml
│   │   └── external-secrets/
│   │       ├── ClusterSecretStore.yaml
│   │       └── values.yaml
│   └── prod
│       └── ... (same layout as dev)
│
├── applicationset
│   ├── argocd-secrets.yaml
│   ├── eduhub-assignment.yaml
│   ├── eduhub-auth.yaml
│   ├── eduhub-catalog.yaml
│   └── external-secrets.yaml
│
└── README.md
```


### End-to-End Flow

High-level flow:

```text
appofapps.yaml (single Argo CD Application)
        ⇩
ApplicationSets (in this repo under /applicationset)
        ⇩
Argo CD Applications (one per service/env)
        ⇩
Helm charts from ACR + values + ExternalSecrets
        ⇩
Running workloads in Kubernetes
```

More detailed:

1. **`appofapps.yaml`** (App of Apps) points Argo CD to this Git repo and the `applicationset/` folder.
2. Argo CD syncs the `ApplicationSet` manifests in `applicationset/`.
3. Each `ApplicationSet` generates one or more **Argo CD Applications** (e.g., `eduhub-auth-dev`).
4. Each Application:
   - Pulls the correct **Helm chart** from ACR (OCI)
   - Applies **values.yaml** from `applications/<env>/<service>/values.yaml`
   - Applies **ExternalSecret** manifests for secrets
5. External Secrets Operator pulls the real secrets from **Azure Key Vault**.
6. Kubernetes deploys and reconciles services continuously.


To bootstrap everything, you use a single **App of Apps** Argo CD Application. (App of Apps Parttern, One-Time Apply)

Example `appofapps.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: base-application
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/meshachdamilare/eduhub-apps-gitops.git
    targetRevision: main
    path: applicationset
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Once Argo CD is installed in the cluster and the `argocd` namespace exists, apply:

```bash
kubectl apply -f appofapps.yaml -n argocd
```

Note: this is applied once during creation.

After this:

- Argo CD will **continuously watch** the `applicationset/` folder in this repo.
- Any change to ApplicationSets, Applications, values, or ExternalSecrets will be reconciled automatically.
- You **do not** need to re-apply `appofapps.yaml` unless you change the approach of deployment

---

#### Secret Management (External Secrets Operator)

Each service directory contains:

```text
applications/dev/<service>/external-secret.yaml
```

This YAML instructs ESO to:

- Read one or more keys from Azure Key Vault  
- Create/update a Kubernetes Secret automatically  
- Refresh secrets on a schedule  
- Keep secrets out of Git forever  

Examples of synced secrets:

- `POSTGRES_PASSWORD`  
- `DATABASE_URL`  
- `JWT_ACCESS_TOKEN_SECRET`  
- `SMTP_PASSWORD`  
- `REDIS_URL`  

`applications/dev/external-secrets/ClusterSecretStore.yaml` configures how ESO connects to Azure Key Vault (tenant, vault URI, auth method, etc.).

---

#### How Argo CD Pulls Private Helm Charts from ACR

Argo CD requires credentials to pull Helm charts from a **private ACR OCI registry**.

This is handled by:

```text
applications/dev/argocd-secrets/argocd-pull-secret.yaml
```

This file is an `ExternalSecret` that:

- Fetches `username` and `password` (or token) from Azure Key Vault  
- Creates a Secret inside the `argocd` namespace  
- Labels it with: ```argocd.argoproj.io/secret-type: repository ```

This tells Argo CD:

> “Use this secret to authenticate to the ACR Helm OCI registry.”

Once created, Argo CD can access:

```text
oci://aksacrusw7.azurecr.io/helm
```

Without this, Argo CD fails with errors like:

```text
401 unauthorized
failed to resolve revision
authentication required
```

This pattern keeps ACR credentials **out of Git** and **managed via Key Vault + ESO**.

---

#### ApplicationSet Logic

Each service has an `ApplicationSet` definition under `applicationset/`, for example:

```text
applicationset/eduhub-auth.yaml
```

The ApplicationSet:

- Points to the Helm chart in ACR:

  ```yaml
  repoURL: oci://aksacrusw7.azurecr.io/helm
  chart: eduhub-auth
  targetRevision: "0.1.0"
  ```

- Pulls values from this repo:

  ```yaml
  values:
    path: applications/dev/eduhub-auth/values.yaml
  ```

- Applies the corresponding `external-secret.yaml` from:

  ```text
  applications/dev/eduhub-auth/external-secret.yaml
  ```

- Deploys to the configured cluster + namespace.
- Enables:
  - automated sync
  - self-healing
  - pruning
  - namespace creation via `CreateNamespace=true`

Similar `ApplicationSet` definitions exist for all the services and tools deployed into the cluster.


The deployment supports multiple environments, currently `dev` and `prod`:

```text
applications/dev/
applications/prod/
```

Each environment can override:

- image tags  
- namespaces  
- ingress domains  
- autoscaling behavior  
- secret names  
- database endpoints  

Argo CD keeps each environment reconciled independently based on the `ApplicationSet` config.

---
### To deploy a new version of a microservice:

**Update image tag or config**

Edit the appropriate values file, for example:

```text
applications/dev/eduhub-auth/values.yaml
```

Change:

```yaml
image:
  tag: "develop-abc123"
```

Or update env/config values as needed.

**Commit & push**

Once pushed to the Git remote:
- Argo CD detects the Git change  
- ApplicationSet regenerates/spec updates as needed  
- Argo CD syncs the application  
- Helm pulls the correct chart version from ACR  
- Kubernetes rolls out the new version

---

### Work in Progress...
- Deploy and set up monitoring tools like Prometheus and Grafana
- Configure Single-Sign-on for Argocd using Okta... Same for prometheus and grafana
- Deploy other bootstrapping tools like Goldilocks, SonarQube, etc
