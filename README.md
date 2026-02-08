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

Create the kind cluster using:

```bash
kind create cluster --config kind-zeus.yaml
kubectl cluster-info --context kind-zeus
```

Firewall: allow inbound 80, 443, 6334 to the host (ideally 6334 allowlisted).

Configure the following file layout:

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

### 3a Install Argo CD once (imperative bootstrap)

You can bootstrap Argo with Helm quickly (then everything else becomes GitOps).

```bash
kubectl create ns argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd -n argocd \
  --set configs.params."server\.insecure"=true
```

`server.insecure=true` is important because Gateway will terminate TLS; Argo CD will serve plain HTTP behind it.

### 3b Apply the “app-of-apps” manifest

apply the `clusters/zeus/bootstrap/app-of-apps.yaml` file.

```bash
kubectl apply -f gitops/clusters/zeus/bootstrap/app-of-apps.yaml
```
