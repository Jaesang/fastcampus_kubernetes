# Ch04_021. [실습] 서비스 외부 오픈용 인증서 관리

쿠버네티스에 배포한 어플리케이션을 외부에서 접속할 때, 특정 도메인을 연결하고 인증서를 사용해 TLS 암호화를 적용할 수 있습니다.

## 어플리케이션 확인
`istio`실습에서 사용한 온라인부띠끄 프로그램에 인증서를 적용합니다. 
```
$ kubectl get po -n msa
NAME                                       READY   STATUS    RESTARTS   AGE
adservice-6f498fc6c6-jjhrp                 2/2     Running   0          5d4h
cartservice-bc9b949b-h8mxb                 2/2     Running   0          5d4h
checkoutservice-598d5b586d-v4fr4           2/2     Running   0          5d4h
currencyservice-6ddbdd4956-9xvjc           2/2     Running   0          5d4h
emailservice-68fc78478-wqgpj               2/2     Running   0          5d4h
frontend-5bd77dd84b-fpw9x                  2/2     Running   0          5d4h
loadgenerator-8f7d5d8d8-lxtb8              2/2     Running   0          5d4h
paymentservice-584567958d-m9rt2            2/2     Running   0          5d4h
productcatalogservice-66897fb6d5-4fhvn     2/2     Running   0          5d2h
productcatalogservice-v2-cb8f96f4f-d74w8   2/2     Running   0          5d1h
recommendationservice-646c88579b-xkcpb     2/2     Running   0          5d4h
redis-cart-5b569cd47-9g5lv                 2/2     Running   0          5d4h
shippingservice-79849ddf8-trkqc            2/2     Running   0          5d4h
$ kubectl get svc -n msa
NAME                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)     AGE
adservice               ClusterIP   10.32.0.27    <none>        9555/TCP    5d4h
cartservice             ClusterIP   10.32.0.235   <none>        7070/TCP    5d4h
checkoutservice         ClusterIP   10.32.0.2     <none>        5050/TCP    5d4h
currencyservice         ClusterIP   10.32.0.214   <none>        7000/TCP    5d4h
emailservice            ClusterIP   10.32.0.118   <none>        5000/TCP    5d4h
frontend                ClusterIP   10.32.0.224   <none>        80/TCP      5d4h
frontend-external       ClusterIP   10.32.0.166   <none>        80/TCP      5d4h
paymentservice          ClusterIP   10.32.0.22    <none>        50051/TCP   5d4h
productcatalogservice   ClusterIP   10.32.0.59    <none>        3550/TCP    5d4h
recommendationservice   ClusterIP   10.32.0.134   <none>        8080/TCP    5d4h
redis-cart              ClusterIP   10.32.0.222   <none>        6379/TCP    5d4h
shippingservice         ClusterIP   10.32.0.182   <none>        50051/TCP   5d4h
$ kubectl get svc -n istio-system
NAME                   TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                                                                      AGE
istio-egressgateway    ClusterIP      10.32.0.225   <none>          80/TCP,443/TCP                                                               5d5h
istio-ingressgateway   LoadBalancer   10.32.0.154   34.64.78.98     15021:30034/TCP,80:31247/TCP,443:30795/TCP,31400:31569/TCP,15443:30900/TCP   5d5h
istiod                 ClusterIP      10.32.0.141   <none>          15010/TCP,15012/TCP,443/TCP,15014/TCP                                        5d5h
jaeger-collector       ClusterIP      10.32.0.242   <none>          14268/TCP,14250/TCP,9411/TCP                                                 5d1h
kiali                  LoadBalancer   10.32.0.75    34.64.200.234   20001:32656/TCP,9090:32095/TCP                                               5d1h
tracing                ClusterIP      10.32.0.92    <none>          80/TCP,16685/TCP                                                             5d1h
zipkin                 ClusterIP      10.32.0.247   <none>          9411/TCP                                                                     5d1h
```
Kiali에 접속하여 `istio-ingressgateway`에 적용되는 `Gateway`와 `VirtualService`를 확인합니다. `istio-ingressgateway`의 External IP를 통해 http 접속을 할 수 있습니다.

## Cert-manager
cert-manager는 X509인증서를 관리하는 쿠버네티스 콘트롤러로, 여러 인증업체로부터 인증서를 발급받고 이를 쿠버네티스 Secret으로 관리할 수 있습니다. 특히 Let's Encrypt를 지원하여 유효한 인증서를 무료로 발급할 수 있는 것이 가장 큰 장점입니다. 또한 인증서의 유효기간 전에 자동으로 연장하는 기능도 포함하여, 인증서의 유효기간에 대한 걱정없이 서비스를 유지할 수 있습니다.

### Cert-manager 설치
```
$ wget https://github.com/cert-manager/cert-manager/releases/download/v1.8.0/cert-manager.yaml
$ kubectl apply -f cert-manager.yaml
$ kubectl get all -n cert-manager
NAME                                           READY   STATUS    RESTARTS   AGE
pod/cert-manager-b4d6fd99b-4w7tp               1/1     Running   0          3d
pod/cert-manager-cainjector-74bfccdfdf-jzjv4   1/1     Running   0          3d
pod/cert-manager-webhook-65b766b5f8-526fz      1/1     Running   0          3d

NAME                           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.32.0.250   <none>        9402/TCP   3d
service/cert-manager-webhook   ClusterIP   10.32.0.119   <none>        443/TCP    3d

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           3d
deployment.apps/cert-manager-cainjector   1/1     1            1           3d
deployment.apps/cert-manager-webhook      1/1     1            1           3d

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-b4d6fd99b               1         1         1       3d
replicaset.apps/cert-manager-cainjector-74bfccdfdf   1         1         1       3d
replicaset.apps/cert-manager-webhook-65b766b5f8      1         1         1       3d
```

### Issuer 생성
`Issuer`는 cert-manager의 커스텀 리소스로 인증서를 인증해주는 Issuer에 대한 정보를 담고 있습니다. 이번 실습에서는 Let's Encrypt를 Issuer로 생성합니다. Let's Encrypt는 무료로 제공하는 상용 인증서 Issuer를 Prod Issuer로 제공하고, 테스트를 위한 용도로 Staging Issuer를 제공하고 있는데, 우선 Staging Issuer로 테스트한 뒤 Prod Issuer를 사용하도록 합니다.
```
$ kubectl apply -f issuer.yaml
$ kubectl get clusterissuer
```
### 도메인 설정
이번 실습에서는 도메인 주소를 `nip.io`를 사용합니다. `nip.io`는 서브도메인으로 외부 접속을 위한 IP 어드레스를 사용하면, 무조건 해당 IP 어드레스로 IP를 반환해주는 서비스입니다. 따라서 도메인을 따로 구입/등록할 필요 없이 바로 도메인과 관련된 작업을 테스트할 수 있습니다. 아래와 같이 사용할 수 있습니다.
```
$ nslookup 100.100.100.100.nip.io
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   100.100.100.100.nip.io
Address: 100.100.100.100

$ nslookup 127.0.0.1.nip.io
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   127.0.0.1.nip.io
Address: 127.0.0.1

$ nslookup test.127.0.0.1.nip.io
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   test.127.0.0.1.nip.io
Address: 127.0.0.1
```

인증서를 발급하기 위해서는 Issuer가 해당 IP로 접속할 수 있어야하기 때문에 인증서를 테스트하기 위해서는 본 실습처럼 외부 IP를 사용해야 인증서를 발급받을 수 있습니다. 외부에서 접속이 불가능한 로컬 IP를 사용할 경우 인증서가 발급되지 않습니다.


### Certificate 생성
`Certificate`는 cert-manager의 커스텀 리소스로 cert-manager가 생성하는 인증서의 대한 오브젝트입니다. 위에서 생성한 `Issuer`와 인증서에 등록할 도메인 주소, 인증서를 저장할 `Secret`을 지정해줍니다. `istio-ingress`에 인증서를 적용하기 위해 같은 네임스페이스인 `istio-system`에 생성합니다. 리소스를 describe하면 인증서 생성 과정을 event에서 확인할 수 있습니다.

```
$ kubectl apply -f cert.yaml -n istio-system
$ kubectl describe certificate ob-cert -n istio-system
Spec:
  Common Name:  jaesang.34.64.78.98.nip.io
  Dns Names:
    jaesang.34.64.78.98.nip.io
  Issuer Ref:
    Kind:       ClusterIssuer
    Name:       letsencrypt-staging
  Secret Name:  ingress-cert
Status:
  Conditions:
    Last Transition Time:  2022-06-10T15:21:57Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2022-09-08T14:21:56Z
  Not Before:              2022-06-10T14:21:57Z
  Renewal Time:            2022-08-09T14:21:56Z
  Revision:                1
Events:
  Type    Reason     Age   From                                       Message
  ----    ------     ----  ----                                       -------
  Normal  Issuing    86s   cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  85s   cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "ob-cert-tl8kb"
  Normal  Requested  85s   cert-manager-certificates-request-manager  Created new CertificateRequest resource "ob-cert-kpz4g"
  Normal  Issuing    35s   cert-manager-certificates-issuing          The certificate has been successfully issued

```

`Issuer`를 통해 정상적으로 인증서가 발급되면 이는 `Secret`으로 확인할 수 있습니다.
```
$ kubectl get secret -n istio-system ingress-cert
NAME           TYPE                DATA   AGE
ingress-cert   kubernetes.io/tls   2      108s
```
### Ingress 연결
이제 발급된 인증서를 `istio-gateway`에 연결합니다. 이제 온라인 부띠끄에 443포트로 https 접속을 수행할 수 있습니다.

```
$ kubectl apply -f istio-gateway-tls.yaml -n msa
$ kubectl apply -f vs.yaml -n msa
```

### Prd Issuer로 인증서 생성
테스트한 인증서는 Let's Encrypt의 Staging Issuer를 사용했기 때문에 브라우저에서 안전한 인증서로 인정하지 않습니다. 이제 Prod Issuer를 생성해서 인증서를 발급해봅니다.
```
$ kubectl apply -f issuer-prd.yaml
$ kubectl apply -f cert-prd.yaml -n istio-system
```
생성된 인증서를 istio gateway에 적용합니다.
```
$ kubectl apply -f istio-gateway-tls-prod.yaml -n msa
```
80포트로 Http요청이 왔을 때 이를 Https로 리다이렉션 해주는 게이트웨이를 생성하면, 항상 https서비스를 기본으로 제공할 수 있습니다.
```
$ kubectl apply -f istio-gateway-redirect.yaml
```

### 리소스 삭제
```
kubectl delete -f istio-gateway-tls.yaml -n msa
kubectl delete -f vs.yaml -n msa
kubectl delete -f istio-gateway-redirect.yaml
kubectl delete certificate ob-cert -n istio-system
kubectl delete certificate ob-cert-prd -n istio-system
kubectl delete secret -n istio-system ingress-cert ingress-cert-prd
kubectl delete -f issuer.yaml
kubectl delete -f issuer-prd.yaml
kubectl delete -f cert-manager.yaml
```

