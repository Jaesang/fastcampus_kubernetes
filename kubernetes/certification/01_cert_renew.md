# Ch04_020. [실습] 쿠버네티스 인증서 갱신하기

쿠버네티스에서 클러스터를 kubeadm으로 구축했을 경우, Kubeadm이 생성한 인증서는 kubeadm을 활용해 유효기간을 갱신할 수 있습니다.

## 유효기간 확인
kubeadm은 `/etc/kubernetes/pki/`에 인증서를 생성하고 관리합니다. `check-expiration` 명령어는 인증서 폴더에 있는 인증서에 대한 유효기간을 표시합니다.
```bash
$ kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 May 27, 2023 15:01 UTC   354d            ca                      no
apiserver                  May 27, 2023 15:01 UTC   354d            ca                      no
apiserver-etcd-client      May 27, 2023 15:01 UTC   354d            etcd-ca                 no
apiserver-kubelet-client   May 27, 2023 15:01 UTC   354d            ca                      no
controller-manager.conf    May 27, 2023 15:01 UTC   354d            ca                      no
etcd-healthcheck-client    May 27, 2023 15:01 UTC   354d            etcd-ca                 no
etcd-peer                  May 27, 2023 15:01 UTC   354d            etcd-ca                 no
etcd-server                May 27, 2023 15:01 UTC   354d            etcd-ca                 no
front-proxy-client         May 27, 2023 15:01 UTC   354d            front-proxy-ca          no
scheduler.conf             May 27, 2023 15:01 UTC   354d            ca                      no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      May 24, 2032 15:01 UTC   9y              no
etcd-ca                 May 24, 2032 15:01 UTC   9y              no
front-proxy-ca          May 24, 2032 15:01 UTC   9y              no
```

## 인증서 유효기간 갱신
`kubeadm certs renew` 명령어를 통해 인증서의 유효기간을 연장, 갱신할 수 있습니다. 
```
# kubeadm certs renew all
[renew] Reading configuration from the cluster...
[renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

certificate embedded in the kubeconfig file for the admin to use and for kubeadm itself renewed
certificate for serving the Kubernetes API renewed
certificate the apiserver uses to access etcd renewed
certificate for the API server to connect to kubelet renewed
certificate embedded in the kubeconfig file for the controller manager to use renewed
certificate for liveness probes to healthcheck etcd renewed
certificate for etcd nodes to communicate with each other renewed
certificate for serving etcd renewed
certificate for the front proxy client renewed
certificate embedded in the kubeconfig file for the scheduler manager to use renewed

Done renewing certificates. You must restart the kube-apiserver, kube-controller-manager, kube-scheduler and etcd, so that they can use the new certificates.
```
갱신된 인증서의 유효기간을 확인합니다. 
- 기존: May 27, 2023 15:01 UTC
- 갱신: Jun 07, 2023 01:38 UTC
```
# kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Jun 07, 2023 01:38 UTC   364d            ca                      no
apiserver                  Jun 07, 2023 01:38 UTC   364d            ca                      no
apiserver-etcd-client      Jun 07, 2023 01:38 UTC   364d            etcd-ca                 no
apiserver-kubelet-client   Jun 07, 2023 01:38 UTC   364d            ca                      no
controller-manager.conf    Jun 07, 2023 01:38 UTC   364d            ca                      no
etcd-healthcheck-client    Jun 07, 2023 01:38 UTC   364d            etcd-ca                 no
etcd-peer                  Jun 07, 2023 01:38 UTC   364d            etcd-ca                 no
etcd-server                Jun 07, 2023 01:38 UTC   364d            etcd-ca                 no
front-proxy-client         Jun 07, 2023 01:38 UTC   364d            front-proxy-ca          no
scheduler.conf             Jun 07, 2023 01:38 UTC   364d            ca                      no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      May 24, 2032 15:01 UTC   9y              no
etcd-ca                 May 24, 2032 15:01 UTC   9y              no
front-proxy-ca          May 24, 2032 15:01 UTC   9y              no
```
실행중인 쿠버네티스 컴포넌트가 사용하는 인증서의 유효기간은 커맨드라인명령어로 확인할 수 있습니다.
```
# kube-apiserver
echo -n | openssl s_client -connect localhost:6443 2>&1 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' | openssl x509 -text -noout | grep Not
# kube-controller-managger
echo -n | openssl s_client -connect localhost:10257 2>&1 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' | openssl x509 -text -noout | grep Not
# kube-scheduler
echo -n | openssl s_client -connect localhost:10259 2>&1 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' | openssl x509 -text -noout | grep No
# kubelet
echo -n | openssl s_client -connect localhost:10250 2>&1 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' | openssl x509 -text -noout | grep Not
```

갱신된 인증서를 적용하기 위해 인증서를 사용하는 컴포넌트를 재시작해야합니다.
```
mv /etc/kubernetes/manifests/* ~/
mv ~/*.yaml /etc/kubernetes/manifests/ 
```

`kubeadm certs renew`명령은 명령어를 실행하는 서버의 인증서만 갱신하기 때문에, 쿠버네티스 컨트롤플레인 노드에서 모두 해당 명령어를 각각 실행하고, 서비스를 재시작해야 갱신된 인증서가 적용됩니다.


## 수동 인증서 갱신
`kubeadm certs renew`명령으로 생성한 인증서의 유효기간은 1년으로 고정되어 있어, 이를 변경할 수가 없습니다. 유효기간을 조정하고 싶거나, 혹은 별도의 정보를 추가하기 위해서는 직접 인증서를 만들고 이를 쿠버네티스에 적용할 수 있습니다.

1. 키 파일 생성: 키는 이미 클러스터에 있는 key파일을 사용해도 무방합니다. openssl 명령어를 사용해 새로운 key파일을 만들 수 있습니다.
2. 인증서 요청 파일(CSR) 생성: 키 파일을 사용하여 인증서 요청 파일(CSR)을 생성합니다. CSR은 인증서를 만들기 위해 필요하며, 무엇을 누구를 위한 인증서인지 내용이 포함됩니다. 인증서를 어떻게 만들 것인지 설정파일이 필요합니다.
3. 인증서 생성: CSR을 사용해 인증서를 생성합니다. 이 때 인증서를 인증해주는 CA가 사용됩니다. CSR을 CA를 통해 인증하여 인증서가 생성되는 것입니다.

1. 키 파일 생성
```
openssl genrsa -out apiserver.key 4096
```
2. 인증서 요청 파일(CSR) 설정파일 생성: apiserver_openssl.cnf, 아래 내용은 master01~03노드에 적용할 apiserver용 인증서 설정파일입니다.
alt_names에 명시된 DNS와 IP값은 이 CSR로 생성한 인증서를 사용할 수 있는 호스트 목록이라고 생각하시면 됩니다. 호스트 목록에 포함되지 않은 호스트에서 해당 인증서를 사용한다면 올바르지 않은 인증서로 파악하고, 안전한 연결임을 보장하지 않습니다.
```
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req]
keyUsage=critical, digitalSignature, keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = oper-1 # master01 hostname
DNS.2 = kubernetes
DNS.3 = kubernetes.default
DNS.4 = kubernetes.default.svc
DNS.5 = kubernetes.default.svc.cluster.local
DNS.6 = lb-apiserver.kubernetes.local
DNS.7 = localhost
DNS.8 = oper-2 # master02 hostname
DNS.9 = oper-3 # master03 hostname
 
IP.1 = 10.233.64.1 ### k8s 환경의 service address 대역으로 맞추어야 한다!
IP.2 = 127.0.0.1
IP.3 = 192.168.97.122    # master01
IP.4 = 192.168.97.132    # master02
IP.5 = 192.168.97.135    # master03
```


3. 인증서 요청파일(CSR) 생성: 인증서 설정파일과 키파일을 사용하여 CSR을 생성합니다. 이 때 인증서가 인증하는 주체에 대해 `-subj`에 표시합니다. `kube-apiserver`의 경우 `/CN=kube-apiserver`가 필요합니다. 쿠버네티스에서 사용하는 인증서는 인증서의 subject명을 정해놓았기 때문에, 정해놓은 subject로 인증서를 생성해야합니다. subject정보는 다음 [문서](https://kubernetes.io/ko/docs/setup/best-practices/certificates/#%EB%AA%A8%EB%93%A0-%EC%9D%B8%EC%A6%9D%EC%84%9C)에서 확인할 수 있습니다.

```
openssl req -new -key apiserver.key -out apiserver.csr -subj "/CN=kube-apiserver" -config apiserver_openssl.cnf
```

4. 인증서 생성: 이 때 인증서를 인증해주는 역할을 하는 ca.crt, ca.key가 사용됩니다. `days`옵션을 통해 10년 유효기간으로 인증서 생성
```
openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out apiserver.crt -days 3650 -extensions v3_req -extfile apiserver_openssl.cnf
```

5. 인증서 교체: /etc/kubernetes/pki의 인증서를 교체
```
cp /etc/kubernetes/pki/apiserver.crt /etc/kubernetes/pki/apiserver.crt.bak
cp apiserver.crt /etc/kubernetes/pki/
```

6. 서비스 재시작: Kube-apiserver 재시작
```
cd /etc/kubernetes
mv manifests/kube-apiserver.yaml ./
mv kube-apiserver.yaml manifests/
```

7. 인증서 확인
```
echo -n | openssl s_client -connect localhost:6443 2>&1 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' | openssl x509 -text -noout | grep Not
            Not Before: Jun  7 03:25:03 2022 GMT
            Not After : Jun  4 03:25:03 2032 GMT
```