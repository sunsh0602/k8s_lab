# 02-ingress

MetalLB · Istio Gateway 등 클러스터 외부 노출 구성.

네트워크: `192.168.137.0/24` (master `.10`, worker1 `.11`)

## 디렉터리

```
02-ingress/
├── metallb/
│   ├── helm/values.yaml      # MetalLB Helm 오버라이드
│   └── yaml/                 # IPAddressPool, L2Advertisement
├── istio/
│   └── gateway-values.yaml
└── ceph-dashboard-gateway.yaml

argocd-apps/dev/
└── app-metallb-dev.yaml      # dev 클러스터 App of Apps 하위

argocd-apps/prod/
└── app-metallb-prod.yaml     # prod 클러스터 App of Apps 하위 (설정 동일, name만 분리)
```

## 1. MetalLB (Argo CD — 권장)

`root-app-dev`가 `argocd-apps/dev/`를 감시하므로, `app-metallb-dev.yaml`만 Git에 push하면 자동 등록됩니다.

```bash
# 최초 1회만 (아직 root-app-dev 미적용 시)
kubectl apply -f argocd-apps/root-app-dev.yaml

# 이후: Git push → root-app-dev → app-metallb-dev sync
argocd app sync metallb-dev
```

자세한 설명: [metallb/README.md](metallb/README.md)

### 수동 설치 (Argo CD bootstrap 전)

[metallb/helm/README.md](metallb/helm/README.md) 참고.

## 2. Istio

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

helm install istio-base istio/base -n istio-system --create-namespace
helm install istiod istio/istiod -n istio-system --wait
helm install istio-ingressgateway istio/gateway -n istio-system -f istio/gateway-values.yaml
```

## 3. Ceph Dashboard 라우팅

Ceph 클러스터가 Ready이고 `rook-ceph-mgr-dashboard` 서비스가 생긴 뒤:

```bash
kubectl apply -f ceph-dashboard-gateway.yaml
```

## 4. 접속

```bash
export CEPH_LB=$(kubectl -n istio-system get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "http://${CEPH_LB}/"

kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath='{.data.password}' | base64 -d && echo
```

브라우저: `http://<MetalLB가 준 IP>/` · 사용자 `admin`
