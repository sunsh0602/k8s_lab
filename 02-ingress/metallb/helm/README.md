# MetalLB 설치 (Helm + Static CR)

네트워크: `192.168.137.0/24`  
IP 풀: `192.168.137.128` ~ `192.168.137.191` (64개)

---

## 왜 MetalLB가 필요한가?

클라우드(AWS, GCP 등)에서는 `Service`의 `type: LoadBalancer`를 만들면 클라우드가 **외부 IP를 자동으로** 붙여 줍니다.

이 lab처럼 **온프레미스 / VM Kubernetes**에는 그런 Load Balancer가 없습니다. `LoadBalancer` Service를 만들면 `EXTERNAL-IP`가 `<pending>`으로 남고, 브라우저나 다른 PC에서 Ingress·ArgoCD·Istio Gateway에 **직접 접속할 수 없습니다.**

**MetalLB**는 이 공백을 메우는 **LoadBalancer 구현체**입니다. 클러스터 밖(같은 LAN)에서 접근 가능한 IP를 `LoadBalancer` Service에 할당합니다.

```
[브라우저 / 다른 PC]
        │
        │  http://192.168.137.128  (MetalLB가 준 IP)
        ▼
[LoadBalancer Service]  ← Istio ingress, ArgoCD 등
        │
        ▼
[Pod]
```

이 lab에서는 이후 **Istio Ingress Gateway**, **ArgoCD UI** 등이 `type: LoadBalancer`를 쓰므로 MetalLB를 먼저 두는 것이 일반적입니다.

---

## 구성 요소가 하는 일

MetalLB 설치는 **Helm(컨트롤러)** + **Static CR(IP 정책)** 두 단계로 나뉩니다.

| 구분 | 리소스 | 역할 |
|------|--------|------|
| Helm | `metallb` 차트 | MetalLB 본체(Deployment, RBAC, Webhook 등) 설치 |
| Pod | **controller** | 어떤 Service에 어떤 IP를 줄지 결정·할당 |
| Pod | **speaker** (DaemonSet) | 노드에서 **ARP**로 “이 IP는 이 클러스터 것”이라고 LAN에 알림 (L2 모드) |
| CR | **IPAddressPool** | MetalLB가 **빌려 줄 수 있는 IP 대역** 정의 |
| CR | **L2Advertisement** | 그 풀의 IP를 **L2(같은 스위치/서브넷)** 방식으로 광고할지 연결 |

Helm 차트(기본)만 쓰면 **controller/speaker Pod**까지만 올라갑니다. **어느 IP 대역을 쓸지**는 환경마다 다르므로, 공식 차트는 보통 **IPAddressPool / L2Advertisement를 values나 별도 manifest로** 넣게 합니다. 이 lab은 `yaml/`에 CR을 두었습니다.

---

## GitOps가 깨지나?

**깨지는 경우**와 **안 깨지는 경우**를 구분해야 합니다.

| 방식 | GitOps |
|------|--------|
| Helm만 `helm install` 하고, IP 풀은 손으로 `kubectl apply` | ❌ Git 밖 변경·드리프트 → **깨짐** |
| Helm + `yaml/` CR **둘 다 Git에 두고** Argo CD가 동기화 | ✅ **안 깨짐** (Application 2개 또는 multi-source 1개) |
| 공식 `metallb/metallb` values만으로 IP 풀까지 | ❌ **현재 차트는 미지원** (v0.13+ CRD 방식, `configInline` 폐기) |

즉 **“Helm은 컨트롤러만”** 이라는 말은 **역할 분리**이지, **CR을 Git 밖에 두라는 뜻이 아닙니다.**

### Argo CD로 맞추는 패턴 (권장)

**패턴 A — Application 2개 (흔함)**

```
apps/
├── metallb-controller.yaml   # Helm: metallb/metallb
└── metallb-config.yaml       # Path: 02-ingress/metallb/yaml/
```

- `metallb-controller`: `Sync Wave 0` — Helm으로 operator
- `metallb-config`: `Sync Wave 1` — `IPAddressPool`, `L2Advertisement` (webhook Ready 이후)

둘 다 **같은 Git repo** → 선언적·추적 가능.

**패턴 B — Argo CD Application 1개 + multi-source (원큐에 가깝음)**

공식 `metallb/metallb` 차트는 **IPAddressPool을 values로 렌더링하지 않습니다.**  
대신 Argo CD **한 Application**에서 Helm과 `yaml/`을 같이 지정하면, 운영상 “한 번에” 동기화됩니다.

```yaml
# 예: Argo CD Application (개념)
spec:
  sources:
    - chart: metallb
      repoURL: https://metallb.github.io/metallb
      targetRevision: 0.14.x
      helm:
        releaseName: metallb
    - path: 02-ingress/metallb/yaml
      repoURL: https://github.com/you/k8s_lab.git
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

→ GitOps도 **Application 1개**, MetalLB도 **한 번의 sync**로 맞출 수 있음 (내부는 소스 2개).

**패턴 C — 래퍼 Helm 차트**

`charts/metallb-lab/`에서 `metallb`를 dependency로 두고, `templates/`에 `IPAddressPool` / `L2Advertisement`를 직접 넣으면 진짜 **Helm release 1개**로 원큐 가능. lab 규모에서는 패턴 A/B가 더 흔함.

**패턴 D — 수동 부트스트랩 (지금 README의 `helm install` + `kubectl apply`)**

- **첫 클러스터**에 Argo CD를 올리기 **전**에만 사용
- Argo CD가 생기면 위 A 또는 B로 **옮겨서** Git이 단일 진실 공급원이 되게 함

**“패턴 B = Helm values에 IP 풀”** 은 예전 `configInline` 시절 이야기이고, **지금 공식 차트로는 안 됩니다.** 원큐를 원하면 **Argo multi-source(패턴 B)** 또는 **래퍼 차트(패턴 C)** 를 쓰면 됩니다.

### 정리

- GitOps = **모든 desired state가 Git에 있고**, 컨트롤러(Argo CD)가 클러스터를 맞춤
- Helm(컨트롤러) + YAML(CR) **이원화 ≠ GitOps 위반**
- **수동 kubectl만 하고 Git에 안 넣는 것**이 GitOps 위반

---

## 1. Helm repo 추가

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
```

---

## 2. MetalLB 설치 (Helm)

```bash
helm install metallb metallb/metallb -n metallb-system --create-namespace
```

### 설치 확인

```bash
kubectl get pods -n metallb-system
kubectl get svc -n metallb-system
```

컨트롤러가 Ready 될 때까지 대기 (IP 풀 CR 적용 전에 필수):

```bash
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/component=controller \
  -n metallb-system \
  --timeout=120s
```

---

## 3. IP 풀 / L2 Advertisement (Static CR)

Helm은 컨트롤러만 설치합니다. IP 대역은 아래 manifest로 적용합니다.

### IPAddressPool (`lab-pool`) — 무엇이고 왜 필요한가?

- **의미**: MetalLB가 `LoadBalancer` Service에 **할당해도 되는 IP 목록(풀)**.
- **왜 필요한가**: controller는 Service가 생기면 풀에서 **비어 있는 IP 하나**를 골라 `status.loadBalancer.ingress`에 기록합니다. 풀이 없으면 IP를 줄 수 없습니다.
- **이 lab 설정**: `192.168.137.128` ~ `191` (64개). 노드 IP(`.10`, `.11`)·게이트웨이·DHCP와 **겹치지 않는** 구간을 씁니다.
- **동작**: Service마다 풀에서 IP 1개씩 소비 (예: Istio ingress 1개, ArgoCD 1개).

### L2Advertisement (`lab-l2`) — 무엇이고 왜 필요한가?

- **의미**: “`lab-pool`에 있는 IP는 **L2 모드**로 노출한다”는 **광고 규칙**.
- **왜 필요한가**: MetalLB는 L2 / BGP 등 여러 방식을 지원합니다. **어떤 풀을 어떤 방식으로 쓸지** CR로 지정해야 합니다. lab·단일 서브넷 환경에서는 **L2**가 가장 단순합니다.
- **L2 모드 동작**: `speaker`가 해당 IP에 대한 **ARP 응답**을 해서, LAN의 다른 기기가 그 IP로 패킷을내면 클러스터 노드로 들어오게 합니다. 별도 BGP 라우터 없이 VM lab에 맞습니다.
- **`ipAddressPools: [lab-pool]`**: 위에서 만든 풀만 L2로 광고한다는 연결입니다.

```
IPAddressPool (할당 가능 IP)
        │
        │  참조
        ▼
L2Advertisement (L2로 광고할 풀 지정)
        │
        ▼
speaker Pod → ARP: "192.168.137.xxx 는 여기"
```

```bash
cd /root/k8s_lab/02-ingress/metallb

kubectl apply -f yaml/ipaddresspool.yaml
kubectl apply -f yaml/l2advertisement.yaml
```

### `yaml/ipaddresspool.yaml`

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lab-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.137.128-192.168.137.191
```

### `yaml/l2advertisement.yaml`

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - lab-pool
```

### CR 확인

```bash
kubectl get ipaddresspool,l2advertisement -n metallb-system
```

---

## 4. LoadBalancer 동작 확인

```bash
# LoadBalancer 타입 Service 예시 (Istio ingress 등)
kubectl get svc -A | grep LoadBalancer
```

`EXTERNAL-IP`가 `192.168.137.128` ~ `191` 범위에 있으면 정상입니다.

---

## 5. 주의사항

| 항목 | 내용 |
|------|------|
| IP 충돌 | master `.10`, worker `.11`, 게이트웨이, DHCP와 겹치지 않게 |
| L2 모드 | 노드와 같은 L2 세그먼트(`192.168.137.0/24`)에서만 동작 |
| 적용 순서 | Helm 설치 → Pod Ready → Static CR 적용 (webhook 준비 전에 CR 적용 시 오류 가능) |
| 풀 용도 | Istio Gateway, ArgoCD 등 `type: LoadBalancer` Service마다 IP 1개 사용 |
| MetalLB ≠ Ingress | MetalLB는 **IP 할당**만 함. HTTP 라우팅·TLS는 Istio/Ingress가 담당 |

---

## 6. 업그레이드 / 삭제

```bash
# 업그레이드
helm upgrade metallb metallb/metallb -n metallb-system

# 삭제
kubectl delete -f yaml/ipaddresspool.yaml
kubectl delete -f yaml/l2advertisement.yaml
helm uninstall metallb -n metallb-system
```
