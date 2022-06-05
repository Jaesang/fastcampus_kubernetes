# Ch04_015. [실습] Istio controlplane 생성
istio controlplane은 dataplane에서 서비스간의 트래픽을 제대로 라우팅할 수 있도록 프록시를 관리하는데, `istio-system` 네임스페이스에 `isdiod`가 해당 역할을 수행합니다.

`istiod`는 `istioctl`이라는 툴을 사용하여 설치하거나, `helm`차트를 통해 `helm`으로 관리할 수 있습니다.

## `istioctl`을 통해 설치하기
### `istioctl` 설치하기
`istioctl`은 istio의 설치와 설정, 모니터링까지 istio를 사용하는데 있어 필요한 명령어를 제공합니다. istio-operator의 설치 또한 `istioctl`에서 제공하고 있습니다.


```
mkdir istio-installation;cd istio-installation
curl -L https://istio.io/downloadIstio | sh -
export PATH="$PATH:/home/jaesanglee/istio-installation/istio-1.13.4/bin"
```

`istioctl`의 버전 명령을 실행하여, 정상적으로 출력되는지 확인합니다.
```
istioctl version

client version: 1.13.4
control plane version: 1.13.4
data plane version: 1.13.4 (14 proxies)
```
### istiod 설치하기
`istioctl install` 명령을 통해 쿠버네티스 클러스터에 바로 `istiod`를 설치할 수 있습니다.
`install`명령어에서는 istio에서 제공하는 설정 Profile을 선택할 수 있는데, 여기서는 `demo`프로필로 설치합니다. 

```
istioctl install --set profile=demo
kubectl get po -n istio-system
```
프로필 별 차이점은 `istioctl profile diff <profile 1> <profile 2>`명령이나, 공식 [문서](https://istio.io/latest/docs/setup/additional-setup/config-profiles/)에서 확인할 수 있습니다.
```
istioctl profile diff default demo
 gateways:
   egressGateways:
-  - enabled: false
+  - enabled: true
...
     k8s:
        requests:
-          cpu: 100m
-          memory: 128Mi
+          cpu: 10m
+          memory: 40Mi
       strategy:
...
```

## istiod 삭제하기
istiod는 `istioctl x uninstall`명령을 통해 삭제할 수 있습니다. (`istioctl x`명령어는 experimental 명령어의 집합으로 정식 명령어 이전 단계의 명령어를 실행할 수 있습니다.)
```
istioctl x uninstall --purge
istioctl manifest generate | kubectl delete -f -
```

## `helm`차트로 istiod 설치
`istioctl`로 설치하는 것과 동일하게 `helm` 차트로 istiod를 설치할 수 있습니다.

### helm repo 추가
```
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

### helm 차트 설치
1. namespace 생성: helm은 namespace를 자동으로 생성하지 않기때문에 차트가 설치될 네임스페이스를 직접 생성합니다.
```
kubectl create namespace istio-system
```
2.  `istio-base` 차트 설치: `istio-base`는 istio에서 필요한 CRD와 클러스터레벨의 설정(클러스터롤,클러스터롤바인딩,이전 차트용)을 담고 있습니다. [git repo](https://github.com/istio/istio/tree/master/manifests/charts/base/templates)
```
helm install istio-base istio/base -n istio-system
```
3. `istiod` 차트 설치: `istiod`의 deployment와 RBAC용 리소스파일, Confimap등으로 이루어져있습니다. [git repo](https://github.com/istio/istio/tree/master/manifests/charts/istio-control/istio-discovery)
```
helm install istiod istio/istiod -n istio-system
```
4. `istio-ingress` 차트 설치: 
```
kubectl create namespace istio-ingress
kubectl label namespace istio-ingress istio-injection=enabled
helm install istio-ingress istio/gateway -n istio-ingress 
```

### helm 차트 삭제
`helm delete`명령을 통해 손쉽게 삭제할 수 있습니다.
```
helm delete istio-ingress -n istio-ingress
helm delete istiod -n istio-system
helm delete istio-base -n istio-system
```

#### 기타 리소스 삭제
Namespace와 CRD를 삭제합니다.
```
kubectl delete namespace istio-ingress
kubectl delete namespace istio-system
kubectl get crd -oname | grep --color=never 'istio.io' | xargs kubectl delete
```
