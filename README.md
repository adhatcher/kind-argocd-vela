# kind-argocd-vela

## Order of operations

1. kind cluster “zeus”
1. Gateway API + Envoy Gateway
1. Argo CD
1. KubeVela 1.10.6
1. KubeVela addons: VelaUX + FluxCD
1. Hostnames: argocd.zeus, vela.zeus (internal DNS, accessed from a remote machine)
1. Self-signed TLS terminated at the Gateway
1. Everything (Gateway + Routes + KubeVela + addons) under Argo GitOps, with explicit sync ordering

## Kind config (expose 80/443 + kubeapi 6334)

This makes the kubernetes api reacable remotely on :6334, and forwards host :80/:443 to fixed NodePorts for Envoy.

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: zeus

networking:
  apiServerAddress: "0.0.0.0"
  apiServerPort: 6334

nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 80
        protocol: TCP
      - containerPort: 30443
        hostPort: 443
        protocol: TCP
```

**Create the kind cluster using:**

```bash
kind create cluster --config kind-zeus.yaml
kubectl cluster-info --context kind-zeus
```

Firewall: allow inbound 80, 443, 6334 to the host (ideally 6334 allowlisted).

**Configure the following file layout:**

```bash
gitops/
  clusters/zeus/
    bootstrap/
      app-of-apps.yaml

    infra/
      00-gateway-api-crds/
        kustomization.yaml
        standard-install.yaml

      10-envoy-gateway/
        kustomization.yaml
        helm-values.yaml
        envoyproxy-nodeports.yaml

      20-argocd/
        kustomization.yaml
        helm-values.yaml

      30-edge/
        kustomization.yaml
        tls-secret.yaml
        gateway.yaml
        httproute-argocd.yaml
        httproute-velaux.yaml

      40-kubevela/
        kustomization.yaml
        helm-values.yaml

      50-kubevela-addons/
        kustomization.yaml
        velaux.yaml
        fluxcd.yaml
```

**Notes:**

1. The folder prefixes (00, 10, 20…) match Argo sync waves and make intent obvious.
1. Gateway + Routes are in 30-edge and fully GitOps-managed.
1. KubeVela addons are applied after core is installed.

## 3. Bootstrap Argo CD once, then let it manage the rest

### 3a. Install Argo CD once (imperative bootstrap)

You can bootstrap Argo with Helm quickly (then everything else becomes GitOps).

```bash
kubectl create ns argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd -n argocd \
  --set configs.params."server\.insecure"=true
```

`server.insecure=true` is important because Gateway will terminate TLS; Argo CD will serve plain HTTP behind it.

### 3b. Apply the “app-of-apps” manifest

apply the `clusters/zeus/bootstrap/app-of-apps.yaml` file.

```bash
kubectl apply -f gitops/clusters/zeus/bootstrap/app-of-apps.yaml
```

## 4. Argo sync ordering (the important part)

Inside `gitops/clusters/zeus/infra`, you’ll have one Application per layer with sync-wave annotations.

**Create this aggregator file:**

`gitops/clusters/zeus/infra/kustomization.yaml`

```yaml
resources:
  - app-00-gateway-api-crds.yaml
  - app-10-envoy-gateway.yaml
  - app-30-edge.yaml
  - app-40-kubevela.yaml
  - app-50-kubevela-addons.yaml
```

### 4a. Gateway API CRDs (wave 0)

`gitops/clusters/zeus/infra/app-00-gateway-api-crds.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gateway-api-crds
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  source:
    repoURL: https://github.com/YOURORG/YOURREPO.git
    targetRevision: main
    path: gitops/clusters/zeus/infra/00-gateway-api-crds
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

`gitops/clusters/zeus/infra/00-gateway-api-crds/kustomization.yaml`

```yaml
resources:
  - standard-install.yaml
```

Put the Gateway API standard install YAML in:
`gitops/clusters/zeus/infra/00-gateway-api-crds/standard-install.yaml`

(Download once and commit it so your cluster isn’t dependent on live URLs.)

### 4b. Envoy Gateway (wave 10)

`gitops/clusters/zeus/infra/app-10-envoy-gateway.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: envoy-gateway
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "10"
spec:
  project: default
  source:
    repoURL: https://github.com/YOURORG/YOURREPO.git
    targetRevision: main
    path: gitops/clusters/zeus/infra/10-envoy-gateway
  destination:
    server: https://kubernetes.default.svc
    namespace: envoy-gateway-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Now define 10-envoy-gateway using Helm via Argo (recommended), but since you’re using plain manifests here, simplest is: commit Envoy Gateway install manifests. If you prefer Helm-in-Argo, tell me what chart/version you want pinned and I’ll give that exact spec.source block.

For a pure-manifest approach, you can still include the EnvoyProxy with NodePorts:

`gitops/clusters/zeus/infra/10-envoy-gateway/kustomization.yaml`

```yaml
resources:
  - envoyproxy-nodeports.yaml
  # plus envoy gateway install manifests (vendor/pinned)
```

`gitops/clusters/zeus/infra/10-envoy-gateway/envoyproxy-nodeports.yaml`

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: zeus-proxy
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        type: NodePort
        nodePort:
          ports:
          - port: 80
            nodePort: 30080
          - port: 443
            nodePort: 30443
```

This is what makes host ports 80/443 forward into the Envoy dataplane consistently

## 5 Self-signed cert + Gateway + HTTPRoutes (wave 30, GitOps-managed)

### 5a. Create a wildcard cert for *.zeus

**Run this on your admin machine:**

```bash
mkdir -p certs
openssl req -x509 -newkey rsa:2048 -sha256 -days 825 -nodes \
  -keyout certs/zeus.key \
  -out certs/zeus.crt \
  -subj "/CN=*.zeus" \
  -addext "subjectAltName=DNS:*.zeus,DNS:argocd.zeus,DNS:vela.zeus"
```

### 5b. Create a Kubernetes TLS Secret manifest (commit to repo)

**Generate base64:**

```bash
CRT_B64=$(base64 -w0 certs/zeus.crt)
KEY_B64=$(base64 -w0 certs/zeus.key)
echo $CRT_B64 | head -c 60; echo
echo $KEY_B64 | head -c 60; echo
```

**Put into:**

`gitops/clusters/zeus/infra/30-edge/tls-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: zeus-wildcard-tls
  namespace: envoy-gateway-system
type: kubernetes.io/tls
data:
  tls.crt: <PASTE_CRT_B64>
  tls.key: <PASTE_KEY_B64>
```

This is the simple GitOps path. If you’d rather not store private keys in git, the next step would be SOPS/age or SealedSecrets—say so and I’ll switch the plan.

### 5c. Gateway + Routes

`gitops/clusters/zeus/infra/app-30-edge.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: edge
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "30"
spec:
  project: default
  source:
    repoURL: https://github.com/adhatcher/kind-argo-vela.git
    targetRevision: main
    path: gitops/clusters/zeus/infra/30-edge
  destination:
    server: https://kubernetes.default.svc
    namespace: envoy-gateway-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```



