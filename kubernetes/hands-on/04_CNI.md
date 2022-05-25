
# Ch03_09. [실습] Calico CNI 설치
쿠버네티스 클러스터에 CNI를 설치하여 Pod Network를 구축한다.

## CNI 설치
`calico` CNI를 설치하기 위해, 홈페이지에서 제공하는 매니페스트를 저장하여, 이를 kubectl로 실행합니다. calico pod이 모두 실행된 뒤에, kubectl get node 명령을 통해 node status가 Ready를 확인할 수 있습니다. 
```
curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O
kubectl apply -f calico.yaml
kubectl get po -A
kubectl get nodes
```

## nginx ingress 설정
nginx ingress를 설치하고 
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


