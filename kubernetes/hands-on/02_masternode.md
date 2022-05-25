
# Ch02_011. [실습] Master Node 구축
가상머신을 생성하고, kubeadm으로 마스터노드를 구축한다.

## 컨트롤플레인 VM 생성
컨트롤플레인(마스터) 노드용 VM 3개를 생성한다.
```
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-standard-2 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-nodes \
    --tags fastcampus-k8s,controller
done
```
## 가상머신 접속
생성된 가상머신에 `gcloud` cli를 사용하여 ssh 접속하여, 아래 작업을 수행한다.
```
gcloud compute ssh controller-0~2
```
## OS 설정
이제부터는 생성된 가상머신에서 수행하는 작업이다.
tmux를 사용해 마스터 노드 3대 VM에 한 화면에 접속하고, `Ctrl+b+e`명령어를 통해 동시에 명령어를 수행하면 빠르게 설치, 설정을 진행할 수 있다. (`apt` 명령이 실패가 나는 경우에는 여러번 재시도한다.)

쿠버네티스 패키지를 추가, 설치하고, 리눅스 swap 기능을 비활성화한다.
```
sudo apt update
sudo apt -y install curl apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt -y install vim git curl wget
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

## cloud config 생성
모든 마스터 노드에서 각자 GCP에 사용하는 프로젝트 id가 들어간 `cloud-config`를 생성한다. 예)project-id = "my-test-project-viqzx"
```
cat << EOF | sudo tee /etc/kubernetes/cloud-config
[Global]
project-id = "<your-project-id>"
EOF
```

## 쿠버네티스 로드밸런서 설정
쿠버네티스를 설치하기 전에 마스터노드의 API서버용 로드밸런서용 설정을 GCP에 생성한다. 이 작업은 이전에 `gcloud`를 실행한 서버에서 실행한다.
```
gcloud compute firewall-rules create fw-allow-network-lb-health-checks \
    --network=fastcampus-k8s \
    --action=ALLOW \
    --direction=INGRESS \
    --source-ranges=35.191.0.0/16,209.85.152.0/22,209.85.204.0/22 \
    --target-tags=allow-network-lb-health-checks \
    --rules=tcp

gcloud compute instance-groups unmanaged create k8s-master \
    --zone=asia-northeast3-a

# 컨트롤러0만 등록, 추후에 1,2도 등록해야
gcloud compute instance-groups unmanaged add-instances k8s-master \
    --zone=asia-northeast3-a \
    --instances=controller-0

gcloud compute health-checks create https k8s-controller-hc --check-interval=5 \
    --enable-logging \
    --request-path=/healthz \
    --port=6443 \
    --region=asia-northeast3

gcloud compute backend-services create k8s-service \
    --protocol TCP \
    --health-checks k8s-controller-hc \
    --health-checks-region asia-northeast3 \
    --region asia-northeast3

# add the instance group to your backend service
gcloud compute backend-services add-backend k8s-service \
    --instance-group k8s-master \
    --instance-group-zone asia-northeast3-a \
    --region asia-northeast3

gcloud compute forwarding-rules create k8s-forwarding-rule \
    --load-balancing-scheme external \
    --region asia-northeast3 \
    --ports 6443 \
    --address fastcampus-k8s \
    --backend-service k8s-service
```

## kubeadm config
최초의 마스터노드로 실행할 controller-0에 접속하여 아래 명령을 수행한다. 이 때 controlPlaneEndpoint의 IP 어드레스는 이전에 생성한 퍼블릭 IP를 사용한다. 아래 명령으로 확인할 수 있다.
퍼블릭 IP 확인
```
gcloud compute addresses list 
```
kubeadmcfg.yaml 생성
```
cat << EOF > kubeadmcfg.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
nodeRegistration:
  criSocket: "/run/containerd/containerd.sock"
  kubeletExtraArgs:
    cloud-provider: "gce"
    cloud-config: "/etc/kubernetes/cloud-config"
    cgroup-driver: "systemd"
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
apiServer:
  extraArgs:
    cloud-provider: "gce"
    cloud-config: "/etc/kubernetes/cloud-config"
  extraVolumes:
  - name: cloud
    hostPath: "/etc/kubernetes/cloud-config"
    mountPath: "/etc/kubernetes/cloud-config"
controllerManager:
  extraArgs:
    cloud-provider: "gce"
    cloud-config: "/etc/kubernetes/cloud-config"
  extraVolumes:
  - name: cloud
    hostPath: "/etc/kubernetes/cloud-config"
    mountPath: "/etc/kubernetes/cloud-config"
networking:
  podSubnet: "192.168.0.0/16"
  serviceSubnet: "10.32.0.0/24"
controlPlaneEndpoint: "34.64.80.42:6443"
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
serverTLSBootstrap: true
EOF
```
## kubeadm init
생성한 `kubeadmcfg.yaml`를 사용해 `kubeadm init` 명령을 수행한다. 이를 통해 쿠버네티스 마스터 노드용 컴포넌트가 실행된다.
```
sudo kubeadm init --upload-certs --config kubeadmcfg.yaml
```
정상적으로 완료가 되면 아래와 같은 성공메시지가 표시된다.
```
our Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 34.64.247.182:6443 --token rg8lnq.6useng7zfvdd4d53 \
        --discovery-token-ca-cert-hash sha256:8398f1ba595ee3b6044fdcaafdeecd4ee2eaea5570722b03fa387cb554f7ec44 \
        --control-plane --certificate-key f0759a0b8fc8e0fb9e1bff49c4f0c88d37511ee879e40d2c739274bb6c9d1729

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 34.64.247.182:6443 --token rg8lnq.6useng7zfvdd4d53 \
        --discovery-token-ca-cert-hash sha256:8398f1ba595ee3b6044fdcaafdeecd4ee2eaea5570722b03fa387cb554f7ec44
```

이 메시지에는 kubectl을 사용할 때 필요한 kubeconfig파일, 마스터노드, 워커노드를 추가할 수 있는 명령어가 표시되기에 따로 저장하여 활용해야한다.

## kubeconfig 복사
위 kubeadm init 출력 메시지에 있는 내용을 사용한다.
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 마스터노드 추가
controller-0이 최초 마스터노드 역할을 하고있는 상태에서 controller-1,2에 접속하여 controller-0에 자기자신을 추가하는 `kubeadm join`명령을 수행한다. 

아래는 명령어 예시이며, 각자 위에 controller-0에 `kubeadm init` 명령어에서 나온 출력 메시지에 있는 명령어를 root유저로 접속하여 실행한다. 
```
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 34.64.247.182:6443 --token rg8lnq.6useng7zfvdd4d53 \
        --discovery-token-ca-cert-hash sha256:8398f1ba595ee3b6044fdcaafdeecd4ee2eaea5570722b03fa387cb554f7ec44 \
        --control-plane --certificate-key f0759a0b8fc8e0fb9e1bff49c4f0c88d37511ee879e40d2c739274bb6c9d1729
```

# 마무리
이번 실습에서는 쿠버네티스 클러스터에서 컨트롤플레인 노드역할을 수행하는 마스터노드를 kubeadm을 통해 구축하였습니다. 다음에는 여기에 워커노드를 추가하는 실습을 진행하도록 하겠습니다.