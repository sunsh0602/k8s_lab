# Istio Helm values

Istio는 chart 3개를 순서대로 설치합니다.

| 파일 | chart | 역할 |
|------|-------|------|
| `base.yaml` | istio/base | CRD |
| `istiod.yaml` | istio/istiod | 컨트롤 플레인 |
| `gateway.yaml` | istio/gateway | Ingress Gateway (LoadBalancer) |

Gateway CR(HTTP 라우팅)은 [`../manifests/`](../manifests/) 에서 관리.

Argo CD: [`../../argocd-apps/dev/app-istio-dev.yaml`](../../argocd-apps/dev/app-istio-dev.yaml)
