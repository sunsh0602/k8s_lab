## 1. 사전 필수 단계

Ceph는 분산 스토리지이기 때문에, 데이터를 저장할 포맷되지 않은 빈 디스크(Raw Block Device)가 필요

```
root@worker1:~# lsblk -f
NAME        FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sr0
nvme0n1
├─nvme0n1p1 vfat   FAT32       A381-CF10                                 1G     1% /boot/efi
├─nvme0n1p2 ext4   1.0         96828836-3c23-40cf-a305-abccbc94ea1f  708.7M    20% /boot
└─nvme0n1p3 ext4   1.0         9cc1b82a-1f20-4bd2-8d00-218ed16f3d24     85G     6% /
nvme0n2
```

`nvme0n2`는 미사용 디스크라 ceph 초기화용으로 쓸 수 있다.

## 2. Ceph Helm repo 등록

```bash
helm repo add rook-release https://charts.rook.io/release
helm repo update
helm search repo rook-release
```

| 차트 | 역할 |
|------|------|
| `rook-ceph` | operator + CRD 설치 |
| `rook-ceph-cluster` | 실제 Ceph 클러스터(CephCluster CR 등) 생성 |

> ⚠️ **순서 중요:** `rook-ceph-cluster` 차트는 CR만 만든다. 이 CR을 실제 MON/OSD/MGR 파드로 만들어주는 주체가 `rook-ceph` operator다.
> **operator를 먼저 설치하지 않으면 CR만 생기고 파드는 하나도 안 뜬다.**

## 3. Ceph Cluster Values 작성

`values/values-dev-1node-1disk.yaml` (1노드/1디스크 개발용).

> ⚠️ `rook-ceph-cluster` 차트는 클러스터 spec을 **`cephClusterSpec:` 아래에 중첩**해서 받는다.
> `mon` / `mgr` / `storage` 등을 최상위(top-level)에 두면 **전부 무시되고 차트 기본값**(mon 3개, `useAllNodes/useAllDevices=true` → 모든 노드의 빈 디스크를 전부 OSD로 점유)이 적용된다.

```yaml
cephClusterSpec:
  mon:
    count: 1
    allowMultiplePerNode: true

  mgr:
    count: 1
    allowMultiplePerNode: true

  storage:
    useAllNodes: false
    useAllDevices: false
    nodes:
      - name: worker1          # 실제 노드 이름과 정확히 일치해야 함 (kubectl get nodes)
        devices:
          - name: nvme0n2      # 포맷 안 된 raw 디스크

cephBlockPools:
  - name: ceph-blockpool
    spec:
      replicated:
        size: 1                # 단일 디스크라 복제본 1 (운영 부적합, 테스트 전용)
```

## 4. 설치 순서

```bash
# 1) Operator 먼저 설치 (CRD + operator + CSI)
helm install rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph --create-namespace

# operator 파드가 Running 될 때까지 대기
kubectl -n rook-ceph rollout status deploy/rook-ceph-operator

# 2) 그 다음 Ceph 클러스터 설치
helm install rook-ceph-cluster rook-release/rook-ceph-cluster \
  --namespace rook-ceph \
  -f values/values-dev-1node-1disk.yaml

# 3) OSD/MON/MGR 파드가 전부 뜬 뒤 Istio 라우팅(대시보드) 적용
kubectl apply -f manifests/dev/ceph-dashboard-istio.yaml
```

## 5. 확인

```bash
kubectl -n rook-ceph get pods
kubectl -n rook-ceph get cephcluster        # PHASE=Ready, HEALTH=HEALTH_OK 목표
kubectl get sc                              # rook-ceph-block StorageClass 생성 확인
```

정상이면 `rook-ceph-osd-*`, `rook-ceph-mon-*`, `rook-ceph-mgr-*` 파드가 뜨고
CephCluster `PHASE`가 `Ready`가 된다.

## 참고: 삭제가 Terminating에서 멈출 때

operator보다 CR을 먼저 정리하지 못하면 finalizer 때문에 네임스페이스가 Terminating에 멈춘다.
반드시 **CR(및 helm cluster 릴리스) → operator → namespace** 순서로 삭제할 것.
이미 멈췄다면 남은 CR들의 finalizer를 제거한다:

```bash
kubectl patch -n rook-ceph cephcluster/rook-ceph --type merge -p '{"metadata":{"finalizers":null}}'
# cephblockpool, cephfilesystem, cephobjectstore 등 남은 CR도 동일하게 처리
```

## ⚠️⚠️ 재설치 전 반드시 노드 데이터 정리 (가장 중요)

`dataDirHostPath: /var/lib/rook`와 OSD 디스크(`nvme0n2`)는 **노드 로컬(HostPath/raw disk)** 이다.
네임스페이스·CR·helm 릴리스를 지워도 **노드의 이 데이터는 남는다.**

정리 없이 재설치하면 새 operator가 만든 keyring과 노드에 남은 옛 keyring이 **불일치**하여
아래 증상으로 **mon quorum에 영원히 도달하지 못한다** (mgr/osd 단계로 못 넘어감):

```
mon quorum a: exceeded max retry count waiting for monitors to reach quorum
handle_auth_bad_method server allowed_methods [2] but i only support [2,1]
[errno 13] RADOS permission denied (error connecting to the cluster)
```

**증상이 보이면 몇 번을 재설치해도 똑같이 막힌다. 반드시 노드 데이터부터 지운다.**

**정리 순서:**

```bash
# 1) master: helm 릴리스 + 네임스페이스 제거
helm uninstall rook-ceph-cluster -n rook-ceph
helm uninstall rook-ceph -n rook-ceph
kubectl delete ns rook-ceph --wait=false
#   (Terminating에서 멈추면 위 finalizer 제거 절차 수행)

# 2) 스토리지 노드(worker1)에 SSH로 직접 접속해서 실행  ← master에서는 안 됨
sudo rm -rf /var/lib/rook/*          # 옛 mon/설정/keyring 데이터 제거
sudo sgdisk --zap-all /dev/nvme0n2   # OSD 디스크 파티션/시그니처 제거
sudo dd if=/dev/zero of=/dev/nvme0n2 bs=1M count=100 oflag=direct
sudo wipefs -a /dev/nvme0n2
```

그 다음 위 **"4. 설치 순서"** (operator → cluster)를 다시 수행한다.
