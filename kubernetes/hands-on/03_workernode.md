
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

#[실습] Calico CNI 설치
## CNI 설치
# install calico
curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O
kubectl apply -f calico.yaml


# glbc 설정 - 검증 실패, nginx로 변경
gcloud iam service-accounts create glbc-service-account \
  --display-name "Service Account for GLBC"

export PROJECT=my-project-name
gcloud projects add-iam-policy-binding $PROJECT \
  --member serviceAccount:glbc-service-account@${PROJECT}.iam.gserviceaccount.com \
  --role roles/compute.admin

gcloud iam service-accounts keys create key.json --iam-account \
  glbc-service-account@${PROJECT}.iam.gserviceaccount.com

cat << EOF > gce.conf
[global]
token-url = nil
project-id = "${PROJECT}"
network-name =  "fastcampus-k8s"
local-zone = "asia-northeast3-a"
subnetwork-name = "k8s-nodes"
EOF

gcloud compute scp key.json gce.conf controller-0:~/

# controller-0
kubectl create secret generic glbc-gcp-key --from-file=key.json -n kube-system

kubectl create configmap gce-config --from-file=gce.conf -n kube-system

for i in 0 1 2; do
    kubectl label nodes worker-${i} topology.kubernetes.io/zone=asia-northeast3-a
    kubectl label nodes worker-${i} topology.kubernetes.io/region=asia-northeast3
done

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-gce/master/docs/deploy/gke/non-gcp/default-http-backend.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-gce/master/docs/deploy/gke/non-gcp/rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-gce/master/docs/deploy/gke/non-gcp/glbc.yaml

#nginx ingress 설정
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.3/deploy/static/provider/cloud/deploy.yaml

# test-ingress.yaml
# nginx svc의 external IP를 계속사용한다.
$ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.32.0.97   34.64.158.119   80:30168/TCP,443:30722/TCP   19m
ingress-nginx-controller-admission   ClusterIP      10.32.0.54   <none>          443/TCP                      19m

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  namespace: kube-system
spec:
  ingressClassName: nginx
  rules:
  - host: 34.64.158.119.nip.io
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: default-http-backend
            port:
              number: 80


##
Ch03_011. [실습] GCPD(GCP Compute Persistent Disk) CSI 설치

## csi 설치
controller-0에서 수행
golang, gcloud와 kubectl 필요
#golang 설치
wget https://go.dev/dl/go1.18.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.18.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
mkdir -p go/bin
mkdir go/pkg
mkdir go/src
export GOPATH=$HOME/go
#gcloud 설치
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
sudo apt-get update && sudo apt-get install google-cloud-cli

#gcloud 설정
gcloud init 
# csi 다운로드
git clone https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver $GOPATH/src/sigs.k8s.io/gcp-compute-persistent-disk-csi-driver
cd $GOPATH/src/sigs.k8s.io/gcp-compute-persistent-disk-csi-driver
export PROJECT=my-project-for-demo-346601
export GCE_PD_SA_NAME=my-gce-pd-csi-sa
export GCE_PD_SA_DIR=~/gce_pd_sa_dir
export ENABLE_KMS=false
mkdir ~/gce_pd_sa_dir
./deploy/setup-project.sh
# CSI 설치
# GKE가 아닐경우 stablle-master를 사용해야 공개 image를 사용
export GCE_PD_DRIVER_VERSION=stable-master
./deploy/kubernetes/deploy-driver.sh
# 스토리지 클래스 생성 및 테스트
kubectl apply -f ./examples/kubernetes/demo-zonal-sc.yaml
kubectl apply -f ./examples/kubernetes/demo-pod.yaml

#기본 스토리지 설정
kubectl patch storageclass csi-gce-pd -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class": "true"}}}'
#metrics server 설치
#인증서 생성
mkdir metrics-server
cd metrics-server

openssl genrsa -out server.key 2048
openssl req -new -key server.key \
  -subj "/CN=metrics-server.kube-system.svc" \
  -reqexts SAN \ 
  -config <(cat /etc/ssl/openssl.cnf <(printf "\n[SAN]\nsubjectAltName=DNS:metrics-server.kube-system.svc")) \
  -out server.csr
cat > metrics-server.csr.yaml <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: metrics-server.kube-system.svc
spec:
  request: $(cat server.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
kubectl apply -f metrics-server.csr.yaml
kubectl certificate approve metrics-server.kube-system.svc
kubectl -n kube-system get csr metrics-server.kube-system.svc \
  -o jsonpath='{.status.certificate}' | base64 --decode > server.crt
kubectl -n kube-system create secret tls metrics-server-cert --cert=server.crt --key=server.key


