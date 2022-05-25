
# Ch03_11. [실습] GCPD(GCP Compute Persistent Disk) CSI 설치
쿠버네티스 클러스터에 CSI를 설치하여 Pod에 Storage Volume을 사용한다.
## 사전준비
golang, gcloud와 kubectl 필요하기 때문에 `controller-0`에서 수행한다.이 3가지가 충족된 다른 노드에서 실행해도 무방하다.

### golang 설치
```
wget https://go.dev/dl/go1.18.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.18.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
mkdir -p go/bin
mkdir go/pkg
mkdir go/src
export GOPATH=$HOME/go
```
### gcloud 설치
`controller-0`이 GCE VM이기 때문에 `gcloud`가 이미 내장되어있다. gcloud가 없는 경우 아래 명령어 혹은 아래 페이지를 참고하여  gcloud cli를 설치한다.
https://cloud.google.com/sdk/docs/quickstarts?hl=ko
```
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
sudo apt-get update && sudo apt-get install google-cloud-cli
```
### gcloud 설정
계정정보와 프로젝트 id, 리전을 설정한다.
```
gcloud init 
```
# CSI 설치
## CSI 다운로드
GCPD CSI는 아래처럼 특정 go path에 위치해야 그 안에 설치 스크립트가 제대로 실행되기 때문에 소스코드 위치를 유의하여 설치한다.
```
git clone https://github.com/kubernetes-sigs/gcp-compute-persistent-disk-csi-driver $GOPATH/src/sigs.k8s.io/gcp-compute-persistent-disk-csi-driver
cd $GOPATH/src/sigs.k8s.io/gcp-compute-persistent-disk-csi-driver

export PROJECT=my-project-for-demo-346601
export GCE_PD_SA_NAME=my-gce-pd-csi-sa
export GCE_PD_SA_DIR=~/gce_pd_sa_dir
export ENABLE_KMS=false

mkdir ~/gce_pd_sa_dir
./deploy/setup-project.sh
```
## CSI 설치
GKE(Google Kubernetes Engine)가 아닌, 인스턴스에 직접 쿠버네티스를 설치하고 여기에 CSI를 설치하는 경우에는 `GCE_PD_DRIVER_VERSION`을 `stable-master`로 설정하고 `deploy-driver.sh`를 실행해야한다.
```
export GCE_PD_DRIVER_VERSION=stable-master
./deploy/kubernetes/deploy-driver.sh
```
# 스토리지 클래스 생성 및 테스트
GCPD CSI드라이버 리포 아래의 `examples`에 있는 스토리지 클래스와 PVC예제를 사용한다.  
```
kubectl apply -f ./examples/kubernetes/demo-zonal-sc.yaml
kubectl apply -f ./examples/kubernetes/demo-pod.yaml
```
## 기본 스토리지 설정
GCPD CSI드라이버를 기본(default) 스토리지클래스로 설정한다.
```
kubectl patch storageclass csi-gce-pd -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class": "true"}}}'
```

# 마무리
테스트 환경이 GCP VM이기 때문에 CSI드라이버를 구글 스토리지를 사용하는 GCPD CSI Driver를 설치해보았습니다. 다양한 스토리지 CSI 드라이버가 있기 때문에 각자 환경에 적합한, 사용가능한 CSI를 설치하여 스토리지 볼륨으로 사용하시기 바랍니다.

