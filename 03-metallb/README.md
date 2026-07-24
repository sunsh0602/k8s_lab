# MetalLB (Argo CD GitOps)

```
metallb/
├── values/
│   ├── values-dev.yaml
│   └── values-prod.yaml
└── manifests/
    ├── dev/
    │   ├── ipaddresspool.yaml
    │   └── l2advertisement.yaml
    └── prod/
        ├── ipaddresspool.yaml
        └── l2advertisement.yaml
```

Argo CD Application:
- dev: [`../argocd-apps/dev/app-metallb-dev.yaml`](../argocd-apps/dev/app-metallb-dev.yaml)
- prod: [`../argocd-apps/prod/app-metallb-prod.yaml`](../argocd-apps/prod/app-metallb-prod.yaml)

| source | 경로 | 역할 |
|--------|------|------|
| Helm | `metallb/metallb` | controller, speaker, CRD, webhook |
| values | `values/values-{dev,prod}.yaml` | chart 오버라이드 |
| manifests | `manifests/{dev,prod}/` | IPAddressPool, L2Advertisement |

IP 풀: `192.168.137.128` ~ `191`

```bash
argocd app sync metallb-dev
kubectl get ipaddresspool,l2advertisement -n metallb-system
```

---

## 수동 설치 (Helm) — GitOps 없이 직접 설치

> MetalLB는 설치 순서상 가장 먼저다. Istio ingressgateway 등 `type: LoadBalancer`
> 서비스가 External IP를 받으려면 MetalLB가 IP를 공급해줘야 하기 때문.
>
> ⚠️ **2단계로 나뉜다.** Helm 차트는 controller/speaker/CRD/webhook만 설치하고,
> **IP 대역(IPAddressPool)은 chart values로 넣을 수 없어** 별도 CR(manifests)로 적용해야 한다.

### 1) Helm repo 등록

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update metallb
```

### 2) Operator 설치 (controller + speaker + CRD)

```bash
helm install metallb metallb/metallb \
  --namespace metallb-system --create-namespace \
  --version 0.14.9 \
  -f values/values-dev.yaml

# controller Ready 될 때까지 대기 (webhook 준비 포함)
kubectl -n metallb-system rollout status deploy/metallb-controller
```

- `--version 0.14.9` : Argo CD Application(`app-metallb-dev.yaml`)과 버전 일치
- `values-dev.yaml` : `crds.enabled: true`, speaker `tolerateMaster: true`(마스터 노드에도 배치) 등 오버라이드

### 3) IP 풀 / L2 광고 CR 적용

> controller의 **validating webhook** 가 떠야 CR이 승인된다.
> 설치 직후 바로 적용하면 webhook 미기동으로 실패할 수 있어, controller Ready 뒤에 적용한다.

```bash
kubectl apply -f manifests/dev/ipaddresspool.yaml      # IP 대역 192.168.137.128-191
kubectl apply -f manifests/dev/l2advertisement.yaml    # 위 풀을 L2(ARP)로 광고
```

- **IPAddressPool** : LoadBalancer 서비스에 나눠줄 IP 범위. **노드와 같은 서브넷**(`192.168.137.0/24`)이어야 L2 모드에서 정상 동작.
- **L2Advertisement** : 해당 풀을 L2(ARP/NDP) 방식으로 광고 → 같은 네트워크에서 그 IP로 접근 가능.

### 4) 확인

```bash
kubectl -n metallb-system get pods
#   metallb-controller  1/1 Running
#   metallb-speaker     4/4 Running  (노드마다 1개, master 포함)
kubectl -n metallb-system get ipaddresspool,l2advertisement
```

speaker가 노드 수만큼(여기선 master+worker1 = 2개) `4/4 Running` 이면 정상.
이후 `type: LoadBalancer` 서비스를 만들면 `192.168.137.128~191`에서 IP가 자동 할당된다.

> ⚠️ **GitOps와 혼용 주의**: 위 수동 helm 설치와 Argo CD `metallb-dev` 앱을 동시에 쓰면
> 같은 릴리스를 두 곳에서 관리해 충돌한다. 둘 중 하나로 통일할 것.

---

자세한 설명: [values/README.md](values/README.md)
