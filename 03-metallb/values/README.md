# MetalLB Helm values

환경별 `metallb/metallb` chart 오버라이드.

| 파일 | 용도 |
|------|------|
| `values-dev.yaml` | dev 클러스터 |
| `values-prod.yaml` | prod 클러스터 |

IP 대역은 chart values로 설정 불가 → [`../manifests/`](../manifests/) CR 사용.

Argo CD: [`../../argocd-apps/dev/app-metallb-dev.yaml`](../../argocd-apps/dev/app-metallb-dev.yaml)

수동 설치·개념 설명: 이전 `helm/README.md` 내용은 Git 히스토리 참고.
