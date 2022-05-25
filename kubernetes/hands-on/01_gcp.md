# 실습)KUBEADM 설치
어떤식으로 설치를 할 것인지 오버뷰 페이지 필요



# 사전 준비
## tmux 한번에 입력하기
하나의 창에서 여러 VM에 접속하기 위해 `tmux`를 사용하며, 동시에 명령어를 수행하는 명령어를 `Ctrl+b+e`로 설정한다.
```
vi ~/.tmux.conf
```
```
bind e set-window-option synchronize-panes
```

## golang 설치

```
wget https://go.dev/dl/go1.18.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.18.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
```

# 구글 클라우드 설정
google cloud platform의 freetier 서비스를 사용해 무료로 VM을 생성하고, kubeadm을 통해 쿠버네티스 클러스터를 설치하는 실습을 진행
## gcloud 설치
아래 페이지를 참고하여  gcloud cli를 설치한다.
https://cloud.google.com/sdk/docs/quickstarts?hl=ko

## gcloud 설정
gcp를 사용할 수 있도록 계정정보와 사용 리전을 설정한다.
```
gcloud init
```

## 리전 선택
클라우드 리소스를 설치할 리전으로 서울 리전을 사용한다.
```
gcloud config set compute/region asia-northeast3
```
## VPC 네트워크 생성
가상머신이 사용할 네트워크를 생성하고 설정한다.
```
gcloud compute networks create fastcampus-k8s --subnet-mode custom
gcloud compute networks subnets create k8s-nodes \
  --network fastcampus-k8s \
  --range 10.240.0.0/24
```
## 방화벽 설정
쿠버네티스 클러스터에 접속하고 내부 가상머신끼리 통신을 허용하는 방화벽을 설정
```
gcloud compute firewall-rules create fastcampus-k8s-allow-internal \
  --allow tcp,udp,icmp,ipip \
  --network fastcampus-k8s \
  --source-ranges 10.240.0.0/24
gcloud compute firewall-rules create fastcampus-k8s-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network fastcampus-k8s \
  --source-ranges 0.0.0.0/0
```
## 퍼블릭 IP 생성
쿠버네티스 API서버의 대표 IP로 사용할 퍼블릭 IP 생성
```
gcloud compute addresses create fastcampus-k8s --region $(gcloud config get-value compute/region)
gcloud compute addresses list
```
## Cloud NAT 설정
가상머신에서 외부 인터넷과 연결되도록 NAT 설정
```
gcloud compute routers create k8s-router \
    --network fastcampus-k8s \
    --region asia-northeast3

gcloud compute routers nats create k8s-nat \
    --router-region asia-northeast3 \
    --router k8s-router \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips
```
# 마무리
이로써 쿠버네티스 클러스터를 구글 클라우드 위에 설치하기 위한 사전 설정이 모두 끝났습니다. gcloud cli를 통해 구글 클라우드 설정을 마치고, 인스턴스가 사용할 네트워크 설정을 미리 마쳤습니다.

다음 실습에서는 마스터노드와 워커노드를 위한 인스턴스를 만들고 kubeadm을 활용해 kubernetes를 직접 설치해보도록 하겠습니다.