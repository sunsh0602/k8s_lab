# 08-jenkins

ArgoCD GitOps로 관리되는 Jenkins (CI 전용). 배포(CD)는 ArgoCD가 담당하므로
Jenkins는 `kubectl apply`를 하지 않고 **빌드 → 이미지 push → Git 매니페스트 수정**만 수행한다.

## 구조

```
08-jenkins/
├── values/
│   ├── dev/jenkins.yaml     # 개발계 Helm values
│   └── prod/jenkins.yaml    # 운영계 Helm values
└── manifests/
    ├── dev/virtualservice.yaml   # Istio로 UI 노출 (dev)
    └── prod/virtualservice.yaml  # Istio로 UI 노출 (prod)
```

ArgoCD Application:
- `argocd-apps/dev/app-jenkins-dev.yaml`
- `argocd-apps/prod/app-jenkins-prod.yaml`

## 사전 확인 사항 (⚠️ 배포 전 조정)

1. **Helm chart 버전** — `app-jenkins-*.yaml`의 `targetRevision`
   `helm repo add jenkinsci https://charts.jenkins.io && helm search repo jenkinsci/jenkins --versions`
2. **storageClass** — `values/*/jenkins.yaml`의 `persistence.storageClass`
   `kubectl get sc` 로 rook-ceph 실제 이름 확인
3. **도메인** — `manifests/*/virtualservice.yaml`의 `hosts`

## 의존성 (apply 순서)

metallb → istio → ceph → **jenkins** → VirtualService

## 활용 방향 (GitOps CI 루프)

```
개발자 push → Jenkins CI (빌드·테스트·이미지 push)
           → Git의 values.yaml image tag 수정 커밋
           → ArgoCD가 감지해 배포 (selfHeal)
```
