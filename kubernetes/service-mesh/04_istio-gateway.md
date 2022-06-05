# Ch04_016. [실습] gateway 생성
istio로 관리하는 서비스를 외부에서 사용하기 위해서는 외부 트래픽에 대한 North-South 프록시 역할을 하는 `istio-gateway`를 설치해야합니다. 

`istio-gateway` 설치 이후에는 외부와 istio가 관리하는 서비스를 연결할 수 있도록 `Gateway`리소스를 생성해야합니다.

## `istio-gateway` 설치
`istio-gateway`는 `istioctl`을 통해 직접 설치, 혹은 `helm` 차트를 통해 `helm`으로 관리할 수 있습니다.

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
