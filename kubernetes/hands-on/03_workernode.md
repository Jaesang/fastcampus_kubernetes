
# Ch02_012. [실습] Worker Node 구축
가상머신을 생성하고, kubeadm으로 워커노드를 구축한다.

## 워커 노드 VM 생성
워커 노드용 VM 3대를 생성한다.
```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-standard-2 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-nodes \
    --tags fastcampus-k8s,worker
done
```

## 가상머신 접속
생성된 가상머신에 `gcloud` cli를 사용하여 ssh 접속하여, 아래 작업을 수행한다.
```
gcloud compute ssh worker-0~2
```

## OS 설정
이제부터는 생성된 가상머신에서 수행하는 작업이다.
tmux를 사용해 워커 노드 3대 VM에 한 화면에 접속하고, `Ctrl+b+e`명령어를 통해 동시에 명령어를 수행하면 빠르게 설치, 설정을 진행할 수 있다. (`apt` 명령이 실패가 나는 경우에는 여러번 재시도한다.)

쿠버네티스 패키지를 추가, 설치하고, 리눅스 swap 기능을 비활성화한다.
```
sudo apt update
sudo apt -y install curl apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt -y install vim git curl wget kubelet kubeadm kubectl
sudo apt -y install kubelet=1.21.11-00 kubectl=1.21.11-00 kubeadm=1.21.11-00
sudo apt-mark hold kubelet kubeadm kubectl

kubectl version --client && kubeadm version

sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
```

## containerd 설치
컨테이너런타임인 `containerd`를 설치하고 이와 관련된 설정을 진행.
```
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt update
sudo apt install -y containerd.io

sudo su -
mkdir -p /etc/containerd
containerd config default>/etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status  containerd
```

## containerd의 cgroup 설정
`/etc/containerd/config.toml`에서 `SystemdCgroup`을 true로 설정한다.
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true

```
```
sudo systemctl restart containerd
```

## kubelet 활성화
```
sudo systemctl enable kubelet
```

## worker 노드 추가
이제 앞서 `kubeadm init` 실행 시 출력된 메시지에 나왔던 워커 노드 추가 명령어를 root권한으로 실행한다. 아래는 예시이며, 각자 환경에서 출력된 명령어를 사용해야한다.
```
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 34.64.247.182:6443 --token rg8lnq.6useng7zfvdd4d53 \
        --discovery-token-ca-cert-hash sha256:8398f1ba595ee3b6044fdcaafdeecd4ee2eaea5570722b03fa387cb554f7ec44
```

## CSR 승인
처음 마스터노드에 추가된 노드들이 사용하는 인증서를 허가한다. controller-0에서 수행하며, 이 작업을 통해 추가된 노드들과 통신할 수 있다.
```
kubectl get -o wide csr|grep Pen|awk '{print "kubectl certificate approve "$1}'|bash
```
# 마무리
이번 실습을 통해 마스터노드만 구성된 쿠버네티스 클러스터에 워커노드를 추가하였습니다. 이제 쿠버네티스 클러스터의 기본 노드 설치는 다 끝났으며, 추가적으로 네트워크와 스토리지 설정이 필요합니다. 이는 다음 실습에서 진행하도록 하겠습니다.