# MetalLB (Argo CD GitOps)

```
metallb/
├── helm/
│   ├── values.yaml    # metallb/metallb Helm 오버라이드
│   └── README.md      # 수동 설치·개념 설명
└── yaml/
    ├── ipaddresspool.yaml
    └── l2advertisement.yaml
```

Argo CD Application:  
- dev: [`../../argocd-apps/dev/app-metallb-dev.yaml`](../../argocd-apps/dev/app-metallb-dev.yaml) (`root-app-dev`)  
- prod: [`../../argocd-apps/prod/app-metallb-prod.yaml`](../../argocd-apps/prod/app-metallb-prod.yaml) (`root-app-prod`)

| source | 경로 | 역할 |
|--------|------|------|
| Helm (외부) | `metallb.github.io/metallb` | controller, speaker, CRD, webhook |
| `$values` | `helm/values.yaml` | chart values 오버라이드 |
| Git path | `yaml/` | IP 풀·L2 광고 CR |

## Argo CD로 배포

```bash
# root-app-dev 적용 후 Git push면 자동. 수동 sync:
argocd app sync metallb-dev
```

## 확인

```bash
kubectl get pods -n metallb-system
kubectl get ipaddresspool,l2advertisement -n metallb-system
kubectl get svc -A | grep LoadBalancer
```

IP 풀: `192.168.137.128` ~ `191`
