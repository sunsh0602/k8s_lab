# 02-argocd

GitOps 엔진. 이후 모든 컴포넌트(metallb, istio, ceph, jenkins ...)는
`argocd-apps/`의 Application으로 ArgoCD가 배포·동기화한다.

> ⚠️ **ArgoCD 자체는 GitOps로 못 깐다** — "ArgoCD를 배포할 ArgoCD"가 없기 때문.
> 최초 1회는 아래처럼 **수동 부트스트랩(kubectl)** 으로 설치한다.

## 1. 설치 (부트스트랩)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### ⚠️ CRD annotation 크기 오류 시

`install.yaml`의 `applicationsets.argoproj.io` CRD가 커서 아래 오류가 날 수 있다:

```
The CustomResourceDefinition "applicationsets.argoproj.io" is invalid:
metadata.annotations: Too long: must have at most 262144 bytes
```

`kubectl apply`가 마지막 적용 상태를 annotation(`last-applied-configuration`)에
통째로 저장하는데, CRD가 그 한도를 넘어 생기는 문제다. **server-side apply로 재적용**하면 해결:

```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 기동 확인

```bash
kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd get pods            # 7개 파드 모두 Running 목표
```

| 파드 | 역할 |
|------|------|
| `argocd-server` | API / 웹 UI |
| `argocd-application-controller` | Application reconcile(sync·selfHeal) 담당 핵심 |
| `argocd-repo-server` | Git clone + helm/kustomize 렌더링 |
| `argocd-applicationset-controller` | ApplicationSet(대량 앱 생성)용 |
| `argocd-redis` | 캐시 |
| `argocd-dex-server` | SSO(OIDC) 연동용 |
| `argocd-notifications-controller` | 알림 |

## 2. 초기 admin 비밀번호

설치 시 자동 생성된 비밀번호가 Secret에 저장된다 (id: `admin`):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

> 로그인 후 비밀번호를 변경하고, `argocd-initial-admin-secret`은 삭제 권장.

## 3. UI 접속

기본적으로 `argocd-server`는 ClusterIP라 외부 노출이 안 된다. 두 가지 방법:

**(a) 포트포워딩 (빠른 확인용)**
```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
# https://localhost:8080  (자체 서명 인증서 경고 무시)
```

**(b) LoadBalancer (MetalLB 있을 때 — 권장)**
```bash
kubectl -n argocd patch svc argocd-server \
  -p '{"spec":{"type":"LoadBalancer"}}'
kubectl -n argocd get svc argocd-server   # EXTERNAL-IP 확인 (192.168.137.128~)
```

## 3-1. 전역 ignoreDifferences (webhook caBundle drift 방지) — 권장

istio·metallb·cert-manager 등은 컨트롤러가 런타임에 webhook/CRD의 `caBundle`·`failurePolicy`를
주입한다. Git엔 그 값이 없어서 ArgoCD가 영구 `OutOfSync`(selfHeal 무한루프)로 본다.
앱마다 `ignoreDifferences`를 넣는 대신 **argocd-cm에 전역으로 한 번** 설정하면 모든 앱에 적용된다.

```bash
kubectl -n argocd patch cm argocd-cm --type merge \
  --patch-file argocd-cm-ignoredifferences.yaml
kubectl -n argocd rollout restart statefulset argocd-application-controller
```

설정 내용: [argocd-cm-ignoredifferences.yaml](argocd-cm-ignoredifferences.yaml)
(Validating/MutatingWebhookConfiguration의 caBundle·failurePolicy, CRD conversion caBundle 무시)

> 이걸 적용하면 각 `app-*.yaml`에 개별 `ignoreDifferences`를 넣을 필요가 없다.

## 4. CLI 로그인 (선택)

```bash
argocd login <EXTERNAL-IP 또는 localhost:8080> --username admin --insecure
```

## 다음 단계 — 컴포넌트 배포 (GitOps)

ArgoCD가 준비되면 각 Application을 등록한다. 예) MetalLB:

```bash
# (충돌 방지) helm 수동 설치본이 있으면 먼저 제거
helm uninstall metallb -n metallb-system

kubectl apply -f ../argocd-apps/dev/app-metallb-dev.yaml
kubectl -n argocd get applications        # Synced / Healthy 확인
```

> Application은 원격 helm 차트(버전 고정) + `NN-*/values` + `NN-*/manifests`를
> 묶어 배포하고, 이후 `selfHeal`로 Git 상태에 맞춰 지속 복구한다.
