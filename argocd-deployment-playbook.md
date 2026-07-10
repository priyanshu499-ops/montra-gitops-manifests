# ArgoCD Application Onboarding & Deployment Strategies Playbook

## 1. Application Onboarding Playbook
This section covers the standard process for setting up the ArgoCD structure and onboarding new applications to the cluster.

### Commands to Execute
Run the following commands in the specified order to configure ArgoCD projects, parent applications (using the App of Apps pattern), and ApplicationSets:

```bash
# 1. Apply ArgoCD Projects
# This creates the logical grouping, RBAC boundaries, and destination restrictions for applications.
kubectl apply -f projects -n argocd

# 2. Apply Parent Applications (App of Apps pattern)
# This sets up the root/parent applications that are responsible for managing child applications.
kubectl apply -f parents -n argocd

# 3. Apply ApplicationSets
# This automatically generates child ArgoCD Applications based on your Git repositories, cluster secrets, etc., completing the onboarding process.
kubectl apply -f applicationsets -n argocd
```

---

## 2. Advanced Deployment Strategies (Argo Rollouts)

The standard industry approach is to use **Argo Rollouts**. 

ArgoCD natively is a GitOps synchronization tool—it makes the cluster match Git. It doesn't handle progressive traffic shifting by itself. Instead, **Argo Rollouts** acts as a drop-in replacement for the standard Kubernetes `Deployment` object to handle the actual traffic shifting, and ArgoCD seamlessly manages the synchronization and visualizes the process.

### Prerequisites

**1. Install the Argo Rollouts controller on the GKE cluster:**
```bash
# Create the namespace
kubectl create namespace argo-rollouts

# Install the Argo Rollouts controller
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# (Recommended) Install the kubectl plugin for local monitoring and CLI promotions
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64
sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

**2. Install the Argo Rollouts UI extension in ArgoCD:**
To view and promote rollouts directly from the ArgoCD dashboard, you need to load the extension into ArgoCD.

*If you deploy ArgoCD via standard manifests, patch the `argocd-cmd-params-cm` to enable extensions, then use an init container on `argocd-server`. A quicker way for testing is patching `argocd-cm`:*
```bash
kubectl patch configmap/argocd-cm -n argocd --type=merge \
  -p '{"data":{"extension.config":"- name: argo-rollouts\n  backend:\n    services:\n    - url: https://github.com/argoproj/argo-rollouts/releases/latest/download/extension.tar.gz"}}'
```

*If you manage ArgoCD via its official Helm Chart, add this to your ArgoCD `values.yaml`:*
```yaml
server:
  extensions:
    enabled: true
    extensionList:
      - name: argo-rollouts
        env:
          - name: EXTENSION_URL
            value: https://github.com/argoproj/argo-rollouts/releases/latest/download/extension.tar.gz
```

### Strategy 1: Without Helm (Plain Kubernetes Manifests)

When using plain YAML manifests, you simply define a `Rollout` custom resource directly in your GitOps repository instead of a standard `Deployment`.

**Example: Blue-Green Strategy Manifest (`rollout.yaml`)**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-company/my-app:v1.0
  strategy:
    blueGreen: 
      # activeService routes traffic to the live/stable version
      activeService: my-app-active
      # previewService routes traffic to the new version before it is promoted
      previewService: my-app-preview
      autoPromotionEnabled: false # Requires a manual approval click to switch traffic
```
**How it works with ArgoCD:** ArgoCD syncs this `Rollout` object just like a `Deployment`. When you update the image tag in Git, ArgoCD syncs the change, and the Argo Rollouts controller takes over to spin up the new version on the preview service. You can then approve the rollout via the ArgoCD UI.

### Strategy 2: With Helm

When using Helm, the core concept remains exactly the same: your Helm chart must be authored to template out a `Rollout` object instead of a `Deployment`.

**Step 1: Modify the Helm Chart**
In your Helm chart's `templates/` directory, replace your standard `deployment.yaml` with a `rollout.yaml` that parameterizes the rollout strategy.

**`templates/rollout.yaml`:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: {{ include "my-chart.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
  strategy:
    {{- toYaml .Values.rolloutStrategy | nindent 4 }}
```

**Step 2: Define Strategy in `values.yaml`**
You can now define your Canary or Blue-Green strategy in the `values.yaml` of the application.

**`values.yaml` (Canary Example):**
```yaml
image:
  repository: my-company/my-app
  tag: v2.0

rolloutStrategy:
  canary:
    steps:
    - setWeight: 20       # Shift 20% of traffic to the new version
    - pause: {duration: 10m} # Wait 10 minutes
    - setWeight: 50       # Shift 50% of traffic
    - pause: {}           # Pause indefinitely (wait for manual approval)
```

**How it works with ArgoCD:** 
ArgoCD is configured to deploy this Helm chart via standard ArgoCD Applications (or ApplicationSets). When developers bump the image tag or modify the weights in `values.yaml` via a Git commit, ArgoCD runs `helm template`, sees the updated `Rollout` manifest, and syncs it to the cluster. The Argo Rollouts controller then executes the phased canary steps automatically.
