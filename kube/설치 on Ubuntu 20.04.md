# Kube

## 용어

- kubeadm : 클러스터 부트스트랩 명령
- kubelet : 클러스터 pod, container 제어 컴포넌트
- kubectl : 클러스터 통신용 CLI 유틸
- CNI : pod 간 네트워크 통신 및 보안 기능 제공.
  - calico, flannel
- istio : service mesh 구현체. 서비스간 통신 및 관리 기능.

## 설치

### multipass

```bash

# 설치 가능 버전 확인: 22.04 는 k8s 설치가 안되어 20.04 설치
multipass find

# ubuntu 20.04, kube 최소 사양: cpu 2, ram 2g
multipass launch -c 2 -m 4G -d 20G -n master 20.04
multipass launch -c 2 -m 2G -d 20G -n node1 20.04
multipass launch -c 2 -m 2G -d 20G -n node2 20.04

# master, node1, node2, node3 접속 하여 설치
multipass shell master
multipass shell node1
multipass shell node2
```

### all master, all node(worker)

#### OS

```bash
# os parameter
SOURCE_FILE="/etc/sysctl.conf"
LINE_INPUT="net.bridge.bridge-nf-call-iptables = 1"
grep -qF "$LINE_INPUT" "$SOURCE_FILE"  || echo "$LINE_INPUT" | sudo tee -a "$SOURCE_FILE"
LINE_INPUT="net.bridge.bridge-nf-call-ip6tables = 1"
grep -qF "$LINE_INPUT" "$SOURCE_FILE"  || echo "$LINE_INPUT" | sudo tee -a "$SOURCE_FILE"
sudo echo '1' | sudo tee /proc/sys/net/ipv4/ip_forward

# 커널 모듈 등록
sudo modprobe overlay
sudo modprobe br_netfilter
sudo sysctl --system

# swap 비활성화
sudo swapoff -a
sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab

# 방화벽 비활성화
sudo ufw disable
sudo systemctl stop ufw.service

# ulimit: 즉시 설정
sudo su
ulimit -f unlimited    # file size 
ulimit -t unlimited    # cpu time 
ulimit -v unlimited    # virtual memory 
ulimit -l unlimited    # locked-in-memory size 
ulimit -n 64000       # open files 
ulimit -m unlimited  # memory size 
ulimit -u 64000      # processes/threads 
ulimit -c unlimited    # core file size

# ulimit: 영구 설정
sudo cat >> /etc/security/limits.conf <<EOF

* soft fsize unlimited 
* hard fsize unlimited 
* soft cpu unlimited 
* hard cpu unlimited 
* soft as unlimited 
* hard as unlimited 
* soft memlock unlimited 
* hard memlock unlimited 
* soft rss unlimited 
* hard rss unlimited 
* soft nproc 64000 
* hard nproc 64000 
* soft nofile 1048576 
* hard nofile 1048576 
* soft core unlimited 
* hard core unlimited 
root      soft   nofile      1048576 
root      hard   nofile      1048576 
root      soft   core        unlimited 
root      hard   core        unlimited 
EOF

```

#### containerd 1.7.2 설치 (all master, all node)

k8s 1.26.7 설치 예정으로 containerd 1.7.0+ 필요

```bash
# bin 설치
wget https://github.com/containerd/containerd/releases/download/v1.7.2/containerd-1.7.2-linux-amd64.tar.gz && \
tar xvfz containerd-1.7.2-linux-amd64.tar.gz && \
sudo mv ./bin/* /usr/local/bin && \
rm -rf ./bin containerd-1.7.2-linux-amd64.tar.gz

# service 생성
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mv containerd.service /usr/lib/systemd/system/

sudo systemctl daemon-reload && \
sudo systemctl enable containerd && \
sudo systemctl start containerd && \
containerd -v
```

### docker 설치

```bash
sudo apt-get update && \
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common && \
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg && \
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null &&\
sudo apt-get update && \
sudo apt-get install -y docker-ce docker-ce-cli && \
sudo systemctl enable docker && \
sudo systemctl start docker && \
docker -v

# enable cri
sudo sed -i '/"cri"/ s/^/#/' /etc/containerd/config.toml && \
sudo systemctl restart containerd
```

### Only Master

#### kubelet, kubeadm, kubectl 설치

master: kubelet, kubeadm, kubectl 설치
node(worker): kubeadm, kubectl 설치

```bash

# kubelet, kubeadm, kubectl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add && \
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"

# 설치 가능 버전 확인
apt-cache madison kubeadm | head -n 10

# 기존 설치 버전 삭제
sudo apt-get autoremove -y --allow-change-held-packages kubeadm kubelet kubectl 

# 설치
sudo apt-get install -y kubeadm=1.26.7-00 && \
sudo apt-get install -y --allow-downgrades kubelet=1.26.7-00 && \
sudo apt-get install -y --allow-downgrades kubectl=1.26.7-00 && \
sudo apt-get install -y --allow-downgrades cri-tools=1.26.0-00

# 자동 업데이트 disable
sudo apt-mark hold kubelet kubeadm kubectl

```

#### kubeadm: master 초기 설정

```bash
sudo kubeadm config images pull --kubernetes-version 1.26.7

# control plane 구성
sudo kubeadm init --pod-network-cidr=10.1.0.0/24 --apiserver-advertise-address $(hostname -I | awk '{print $1}')

# 각 node 에서 실행할 join 명령 저장
#sudo kubeadm join 172.29.103.168:6443 --token 9n613i.9lw7cwq6l0nptkw1 \
        --discovery-token-ca-cert-hash sha256:6dc98cdfae87ebc843ae075564665c0349e2384ffea566d9eb2cab2060633494

# 현재 사용자가 kube 명령 사용 할 수 있도록 설정
mkdir -p ~/.kube && \
sudo cp -i /etc/kubernetes/admin.conf ~/.kube/config && \
sudo chown $(id -u):$(id -g) ~/.kube/config

# kubectl 명령 tab 자동 완성 설정
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

```

#### kube 상태 확인

```bash
# CNI (calico 설치 예정) 이 없으므로 NotReady 상태 이다.
kubectl get nodes -o wide

# CNI 가 없으므로 coredns 가 Pending 으로 나온다.
watch kubectl get pods -A

# kube api server port 6443 이 떠 있는지 확인.
# ubuntu 22.04 에선 api 가 원인 불명으로 죽어서 진행할 수 없었다.
ss -nltp | grep 6443


```

#### calico (CNI) 설치 (master)

```bash
mkdir -p ~/kube/system && cd ~/kube/system && \
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml 

curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml -O  && \
vi custom-resources.yaml
# custom-resources.yaml 의 cidr 을 10.1.0.0/24 로 변경

kubectl create -f custom-resources.yaml

# pods Running 상태 확인
# disk 5GB 로 했을때 용량 부족(df -h) 으로 pods 들이 무수히 생기며 evicted 상태를 보인다.
watch kubectl get pods -A

# node Ready 상태 확인
kubectl get nodes -o wide
```

### Only Node

#### kubeadm, kubectl 설치

```bash
# kubeadm, kubectl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add && \
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"

# 설치 가능 버전 확인
apt-cache madison kubeadm | head -n 10

# 기존 설치 버전 삭제
sudo apt-get autoremove -y --allow-change-held-packages kubeadm kubelet kubectl 

# 설치
sudo apt-get install -y kubeadm=1.26.7-00 && \
sudo apt-get install -y --allow-downgrades kubectl=1.26.7-00 

# 자동 업데이트 disable
sudo apt-mark hold kubeadm kubectl


# sudo kubeadm join 명령 실행
sudo kubeadm join 172.29.103.168:6443 --token 9n613i.9lw7cwq6l0nptkw1 \
        --discovery-token-ca-cert-hash sha256:6dc98cdfae87ebc843ae075564665c0349e2384ffea566d9eb2cab2060633494

# 아후 master 에서 노드 상태 확인: watch kubectl get nodes -o wide 
```
