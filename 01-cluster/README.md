## 1. Swap 영구 비활성화
```
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

## 2. 커널 모듈 설정
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

#로드
sudo modprobe overlay
sudo modprobe br_netfilter
```

## 3. sysctl 설정
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 적용
sudo sysctl --system
```

## 4. containerd 설치
```
sudo apt update
sudo apt install -y containerd

# 기본 설정 생성
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

## 5. systemd cgroup 변경
```
sudo vi /etc/containerd/config.toml

# true로 변경
SystemdCgroup = true

# 재기동
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## 6. Kubernetes 패키지 설치
```
# 패키지 저장소 추가
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```

## 7. kubelet, kubeadm, kubectl 설치
sudo apt update
```
sudo apt install -y kubelet kubeadm kubectl

# 버전 고정
sudo apt-mark hold kubelet kubeadm kubectl
```

## 8. Master 초기화
master 노드에서만. Calico 기본 CIDR 안 쓰고 바꿔보자.
```
# pod cidr 172.16.0.0/16
sudo kubeadm init --pod-network-cidr=172.16.0.0/16
calico yaml 다운로드
```

## 9. Worker join
```
kubeadm token create --print-join-command
```
