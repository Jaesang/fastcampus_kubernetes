# Ch04_016. [실습] gateway 생성
istio로 관리하는 서비스를 외부에서 사용하기 위해서는 외부 트래픽에 대한 North-South 프록시 역할을 하는 `istio-gateway`를 설치해야합니다. 

`istio-gateway` 설치 이후에는 외부와 istio가 관리하는 서비스를 연결할 수 있도록 `Gateway`리소스를 생성해야합니다.

## `istio-gateway` 설치
`istio-gateway`는 `istioctl`을 통해 직접 설치, 혹은 `helm` 차트를 통해 `helm`으로 관리할 수 있습니다.

이번 실습에서는 이전에 istio operator를 활용을 한 환경을 사용해 istio gateway를 설정, 테스트해보도록 하겠습니다.

cat ~/istio/istio-demo.yaml

이처럼 데모 프로필을 사용해서 이스티오를 설치하면 istio-system 네임스페이스에 istio gateway가 같이 설치가 되어있습니다.

istio system 네임스페이스에 svc를 조회해보면 istio-ingress에 외부 IP가 할당된 것을 볼 수 있습니다. 즉 이 외부 IP가 쿠버네티스 내부 서비스가 외부에 노출되는 IP역할을 하게 되는 것입니다.

그럼 이제 내부에 서비스를 외부에 노출하기 위해서 필요한 작업을 하도록 하겠습니다.


## `istioctl`을 통해 설치하기
`istioctl`은 install 명령을 통해 istio controlplane을 설치하는데 이 때 사용하는 다양한 config profile을 제공합니다. (프로필의 대한 설명은 공식 [문서](https://istio.io/latest/docs/setup/additional-setup/config-profiles/)에서 확인할 수 있습니다.)
프로필 중 `demo`프로필에서 `istio-gateway`를 설치를 지원하기 때문에 설치 시 해당 프로필을 추가해줍니다. `istioctl install`은 `istio-gateway`만 독립해서 설치하는 것은 지원하지 않습니다.
```
istioctl install --set profile=demo
kubectl get po -n istio-system
```

## `helm`차트를 통한 설치
### helm repo 추가
```
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

### helm 차트 설치
`helm`차트의 경우 차트가 설치될 네임스페이스를 미리 만들어주어야합니다. 
```
kubectl create namespace istio-ingress
kubectl label namespace istio-ingress istio-injection=enabled
helm install istio-ingress istio/gateway -n istio-ingress 
```

## Istio-gateway를 통해 외부로 서비스 노출

이스티오는 외부로부터 내부 서비스를 연결시켜줄 리소스로 Gateway를 제공하고 있습니다.
이 파일은 프론트엔드 게이트웨이로써 이스티도: ingresssgateway라는 레이블 가지고 있는 서비스에 들어오는 요청을 처리합니다. 그리고 hosts규칙을 보면 어떤 hosts를 요청하든지 80포트로 요청이 들어오는 것을 처리하는 것을 의미합니다.

그러면 이제 이 게이트웨이로 부터 내부 서비스를 연결할 수 있는데, 그러한 역할은 우리가 이전에 카나리 배포 테스트시 사용됐던 virtual 서비스가 담당합니다.

프론트엔드 게이트웨이로 부터 들어오는 트래픽은 어떤 호스트에 상관없이 frontend서비스의 80포트로 라우팅 경로를 지정해주는 것입니다.


이제 Gateway와 Virtual Service 리소스를 생성하여 istio가 관리하는 어플리케이션을 외부에 노출할 수 있습니다. 이 때 helm차트로 `istio-gateway`를 설치하였다면 gateway selector에 `istio: ingress`를 사용하고, istioctl로 설치하였다면 `istio: ingressgateway`를 사용한다.

vi istio-gateway.yaml
```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: frontend-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: frontend-ingress
spec:
  hosts:
  - "*"
  gateways:
  - frontend-gateway
  http:
  - route:
    - destination:
        host: frontend
        port:
          number: 80
```

```
kubectl apply -f istio-gateway.yaml -n msa
```

##  helm 차트 삭제
```
helm delete -n istio-ingress istio-ingress
kubectl delete ns istio-ingress
```
