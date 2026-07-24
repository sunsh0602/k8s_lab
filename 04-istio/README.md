# Istio (Argo CD GitOps)

```
istio/
├── values/
│   ├── dev/
│   │   ├── base.yaml      # istio/base
│   │   ├── istiod.yaml    # istio/istiod
│   │   └── gateway.yaml   # istio/gateway (LoadBalancer)
│   └── prod/
│       ├── base.yaml
│       ├── istiod.yaml
│       └── gateway.yaml
└── manifests/
    ├── dev/
    │   └── gateway.yaml   # Gateway CR (istio-ingressgateway)
    └── prod/
        └── gateway.yaml
```

Argo CD Application:
- dev: [`../argocd-apps/dev/app-istio-dev.yaml`](../argocd-apps/dev/app-istio-dev.yaml)
- prod: [`../argocd-apps/prod/app-istio-prod.yaml`](../argocd-apps/prod/app-istio-prod.yaml)

| source | 차트/경로 | 역할 |
|--------|-----------|------|
| Helm | `istio/base` | CRD, 클러스터 리소스 |
| Helm | `istio/istiod` | 컨트롤 플레인 |
| Helm | `istio/gateway` | Ingress Gateway Pod + LoadBalancer Service |
| Git path | `manifests/{dev,prod}/` | Gateway CR (HTTP :80) |

## apply 순서

| 순서 | Application |
|------|-------------|
| 1 | metallb-dev |
| 2 | istio-dev |
| (이후) | rook-ceph 등 |

MetalLB Ready → Istio Gateway LoadBalancer IP 할당.

## 확인

```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system istio-ingressgateway
kubectl get gateway -n istio-system
```

Ceph dashboard 라우팅(VirtualService)은 [`05-rook-ceph/manifests/`](../05-rook-ceph/manifests/) 에서 관리.

## 수동 설치

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

helm install istio-base istio/base -n istio-system --create-namespace \
  -f values/dev/base.yaml
helm install istiod istio/istiod -n istio-system \
  -f values/dev/istiod.yaml --wait
helm install istio-ingressgateway istio/gateway -n istio-system \
  -f values/dev/gateway.yaml

kubectl apply -f manifests/dev/
```
