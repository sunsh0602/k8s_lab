## 1. 사전 필수 단계

Ceph는 분산 스토리지이기 때문에, 데이터를 저장할 포맷되지 않은 빈 디스크(Raw Block Device)가 필요

root@worker1:~# lsblk -f
> NAME        FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
> sr0
> nvme0n1
> ├─nvme0n1p1 vfat   FAT32       A381-CF10                                 1G     1% /boot/efi
> ├─nvme0n1p2 ext4   1.0         96828836-3c23-40cf-a305-abccbc94ea1f  708.7M    20% /boot
> └─nvme0n1p3 ext4   1.0         9cc1b82a-1f20-4bd2-8d00-218ed16f3d24     85G     6% /
> nvme0n2

nvme0n2nvme0n2는 미사용 디스크라 ceph 초기화용으로 쓸 수 있다.

## 2. Ceph Helm 다운로드

helm repo add rook-release [https://charts.rook.io/release](https://charts.rook.io/release)
helm repo update
helm search repo rook-ceph

| 차트 | 역할 |
|------|------|
| rook-ceph | operator 설치 |
| rook-ceph-cluster | 실제 ceph cluster 생성 |


## 3. helm chart 다운로드만 하기
```
#헬름 차트는 rook-ceph-operator로 설치하는 것이 좋다.
helm pull rook-release/rook-ceph
```


## 4. Ceph Cluster Values 작성
./rook-ceph/single-node-values.yaml 파일을 만든다.

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
      - name: worker1
        devices:
          - name: nvme0n2

cephBlockPools:
  - name: ceph-blockpool
    spec:
      replicated:
        size: 1
```

흐름
- rook-ceph → Rook Operator + CRD + (요즘 버전은) CSI operator 서브차트
- rook-ceph-cluster → CephCluster CR 등을 만들어 MON/OSD/MGR 실제 클러스터 구성
- 지금 nvme0n2 같은 디스크 지정은 2번 rook-ceph-cluster의 cephClusterSpec.storage 쪽입니다. Operator 차트 values.yaml만 고쳐서는 OSD가 생기지 않습니다

```
1) Operator 설치
# 1. 생성한 파일 적용하여 클러스터 배포
helm install rook-ceph-cluster rook-release/rook-ceph-cluster \
  --namespace rook-ceph \
  -f values-single-node-single-disk.yaml

# 2. 클러스터 구성 완료 후 (OSD 파드들이 다 뜬 후) Istio 라우팅 적용
kubectl apply -f ceph-dashboard-istio.yaml

```