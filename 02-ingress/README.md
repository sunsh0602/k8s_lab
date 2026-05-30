## MetalLB + Istio Gateway로 Ceph Dashboard 노출

네트워크: `192.168.137.0/24` (master `.10`, worker1 `.11`)

### 1. MetalLB

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
helm install metallb metallb/metallb -n metallb-system --create-namespace

kubectl apply -f metallb/ipaddresspool.yaml
kubectl apply -f metallb/l2advertisement.yaml
```

### 2. Istio

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

helm install istio-base istio/base -n istio-system --create-namespace
helm install istiod istio/istiod -n istio-system --wait
helm install istio-ingressgateway istio/gateway -n istio-system -f istio/gateway-values.yaml
```

### 3. Ceph Dashboard 라우팅

Ceph 클러스터(`rook-ceph-cluster`)가 올라와 `rook-ceph-mgr-dashboard` 서비스가 생긴 뒤:

```bash
kubectl apply -f ceph-dashboard-gateway.yaml
```

### 4. 접속

```bash
export CEPH_LB=$(kubectl -n istio-system get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "http://${CEPH_LB}/"

# 대시보드 admin 비밀번호
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath='{.data.password}' | base64 -d && echo
```

브라우저에서 `http://192.168.137.200/` (MetalLB가 할당한 IP) 로 접속. 사용자 `admin`.
