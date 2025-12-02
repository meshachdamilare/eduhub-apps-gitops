# **EduHub GitOps Repository**

This repository contains the full GitOps configuration for deploying EduHub microservices and platform bootstrap components to Kubernetes using **Argo CD**, **ApplicationSets**, **Helm**, and **Kubernetes manifests**.


#### Repository Structure

```
├── applications
│   ├── dev
│   │   ├── eduhub-auth
│   │   │   ├── external-secret.yaml
│   │   │   └── values.yaml
│   │   ├── eduhub-assignment
│   │   ├── eduhub-catalog
│   │   ├── logging
│   │   │   └── loki-stack-values.yaml
│   │   ├── monitoring
│   │   │   ├── kube-prometheus-stack-values.yaml
│   │   │   └── servicemonitors
│   │   │       └── eduhub-auth-servicemonitor.yaml
│   │   ├── external-secrets
│   │   ├── argocd-secrets
│   └── prod
├── applicationset
│   ├── eduhub-auth.yaml
│   ├── eduhub-catalog.yaml
│   ├── eduhub-assignment.yaml
│   ├── kube-prometheus-stack.yaml
│   ├── loki-stack.yaml
│   ├── external-secrets.yaml
│   └── argocd-secrets.yaml
└── appofapps.yaml
```



The deployment workflow follows the **App-of-Apps pattern**:

```
Layer 1 → Root Application (appofapps.yaml)
Layer 2 → ApplicationSets (per microservice or platform component)
Layer 3 → Applications (Helm releases + manifests)
```

#### Flow Summary

- You apply **one file** → `appofapps.yaml`  
- Argo CD scans the `applicationset/` directory  
- ApplicationSets generate the final Applications  
- Applications deploy Helm charts + manifests  
- Kubernetes and Git remain continuously in sync  

---

### Folder Breakdown

| Folder | Purpose |
|--------|---------|
| `appofapps.yaml` | Root App that bootstraps the whole GitOps workflow. Applied **once**. |
| `applicationset/` | ApplicationSets that generate Apps for each microservice or platform component. |
| `applications/dev/` | Dev environment Helm values and manifests. |
| `applications/prod/` | Production environment Helm values and manifests. |

---

#### Bootstrap Argo CD (One-Time)

Apply the root App-of-Apps:

```bash
kubectl apply -f appofapps.yaml -n argocd
```

It ca also be triggered in a pipeline

This file:

- Loads all ApplicationSets
- Creates all Applications
- Deploys every component automatically

Git becomes the single source of truth.

---

#### ApplicationSets

Each file in `applicationset/` defines how a component is deployed, using:

- Helm chart source (remote chart)
- Git source (values + additional manifests)
- Namespace destination
- Sync policies
- Auto-pruning and auto-healing

Flow:

```
ApplicationSet → creates Application → deploys service
```

ApplicationSets support multiple environments through generators (e.g., `list`, `matrix`).

---

#### The applications/ Directory

This directory contains all configuration for each **environment** (`dev`, `prod`).

Example service:

```
applications/dev/eduhub-auth/
│── values.yaml
│── external-secret.yaml
```

Monitoring Example

```
applications/dev/monitoring/
│── kube-prometheus-stack-values.yaml
│── servicemonitors/
│     └── eduhub-auth-servicemonitor.yaml
```

Logging Example

```
applications/dev/logging/loki-stack-values.yaml
```

---

#### Helm + Manifests (How Argo CD Applies Them)

There are **two patterns** depending on the folder content.


##### A) When Git contains ONLY Helm values → `path: .`

Example:

```
applications/dev/eduhub-auth/values.yaml
```

This folder contains **no Kubernetes manifests**, so:

```yaml
path: .
```

Works because Argo only uses the values file.

---

##### B) When Git contains manifests → `path: '{{ .manifestsPath }}'`

Example:

```
applications/dev/monitoring/servicemonitors/
```

The ApplicationSet sets:

path: . vs path: '{{ .manifestsPath }}' in an Argo CD multi-source ApplicationSet.

- Use path: . when the Git repo is ONLY providing Helm values
```
sources:
  - repoURL: https://grafana.github.io/helm-charts
    chart: loki-stack
    targetRevision: "2.10.2"
    helm:
      valueFiles:
        - $values/{{ .valuesPath }}

  - repoURL: https://github.com/meshachdamilare/eduhub-apps-gitops.git
    targetRevision: main
    ref: values
    path: .
```

In this case:

- valuesPath might be applications/dev/logging/loki-stack-values.yaml

- Argo CD doesn’t apply any YAML from Git as manifests

- Git is only there to feed Helm values.yaml

```
sources:
- repoURL: aksacrusw7.azurecr.io/helm
    chart: eduhub-auth
    targetRevision: "0.1.0"
    helm:
    valueFiles:
        - $values/{{ .valuesPath }}   # e.g. applications/dev/eduhub-auth/values.yaml

- repoURL: https://github.com/meshachdamilare/eduhub-apps-gitops.git
    targetRevision: main
    ref: values
    path: '{{ .manifestsPath }}'      # e.g. applications/dev/eduhub-auth

```
In this case:
- Helm reads overrides from valuesPath
- Argo CD also applies all Kubernetes manifests under applications/dev/eduhub-auth (e.g. external-secret.yaml, future networkpolicy.yaml, etc.)


---

### Pulling Private Helm Charts from Azure ACR (OCI)

Argo CD does **not** automatically authenticate to private OCI registries.  
You must explicitly provide a repository credential.

Create an Argo CD repository secret deployed in the argocd namespace:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repo-acr-credential
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: "myregistry.azurecr.io/helm"
  username: "<client-id or acr-username>"
  password: "<client-secret or acr-password>"
  enableOCI: "true"
type: Opaque
```

Important Notes

- `url:`in secret **must match** the URL used in `repoURL:` in ApplicationSet
- Do **not** prefix with `oci://`  argocd handles that automatically.
- Use External Secrets Operator for managing the secret securely  

---

#### Connecting Argo CD to GitHub via SSH

1. Generate SSH key:

```bash
ssh-keygen -t rsa -b 4096 -C "argocd@meshachdevopsonline.com" -f ~/.ssh/argocd_github_key
```

2. Add the public key (`*.pub`) to GitHub  
**Repo → Settings → Deploy Keys → Add Key**

3. Add the repo to Argo CD:

```bash
argocd repo add git@github.com:meshachdamilare/eduhub-apps-gitops.git   --ssh-private-key-path ~/.ssh/argocd_github_key
```

---

This repository implements a complete GitOps workflow:

✔ One-time bootstrap (`appofapps.yaml`)  
✔ ApplicationSets manage all services  
✔ Helm + manifests deployed via multi-source  
✔ Supports private ACR OCI registries  
✔ GitHub SSH authentication  
✔ Dev/prod separation  
✔ Secure secret handling via External Secrets  
✔ Fully extensible for new microservices  

Git is the single source of truth.  
Argo CD keeps Kubernetes in sync automatically, continuously, and reliably.
