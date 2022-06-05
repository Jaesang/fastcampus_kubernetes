# Ch04_014. [실습] Istio operator 설치
istio는 `istioctl`을 사용해 직접 설치하거나, `helm` 차트를 사용하여 helm으로 관리할 수 있습니다. 이번 실습에서는 `istioctl`을 통해 `istio-operator`를 설치하는 실습을 진행합니다. 다른 설치 방법은 istio [문서](https://istio.io/latest/docs/setup/install/)에서 확인할 수 있습니다.

## istioctl 설치
`istioctl`은 istio의 설치와 설정, 모니터링까지 istio를 사용하는데 있어 필요한 명령어를 제공합니다. istio-operator의 설치 또한 `istioctl`에서 제공하고 있습니다.


```
mkdir istio-installation;cd istio-installation
curl -L https://istio.io/downloadIstio | sh -
export PATH="$PATH:/home/jaesanglee/istio-installation/istio-1.13.4/bin"
```

`istioctl`의 버전 명령을 실행하여, 정상적으로 출력되는지 확인합니다.
```
istioctl version

no running Istio pods in "istio-system"
1.13.4
```
## istio-operator 설치
`istioctl operator` 명령을 통해 istio-operator를 설치합니다. operator를 설치하면 `istio-operator` 네임스페이스가 생성이되고, `istio-operator` deployment가 생성됩니다.
```
istioctl operator init
Installing operator controller in namespace: istio-operator using image: docker.io/istio/operator:1.13.4
Operator controller will watch namespaces: istio-system
✔ Istio operator installed
✔ Installation complete

kubectl get ns
kubectl get po -n istio-operator
```

## istiod 설치
`istio-operator`는 `istio-system`네임스페이스에 `IstioOperator` CR이 생성되는지 모니터링합니다. 해당 네임스페이스에 istioOperator를 생성합니다.

### istio-demo.yaml 생성
istio는 여러개의 설치 Manifest를 제공하는데 이를 Profile이라고 부릅니다. 이번 실습에서는 `demo` 프로필을 사용합니다. 다른 Profile의 대한 정보는 공식[문서](https://istio.io/latest/docs/setup/additional-setup/config-profiles/)에서 확인할 수 있습니다.

istio-demo.yaml파일을 생성합니다.
```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: demo
``` 
istio-demo.yaml파일을 kubectl로 실행합니다. istio-system 네임스페이스에 관련 리소스가 생성된 것을 확인할 수 있습니다.
```
kubectl apply -f istio-demo.yaml
kubectl get po -n istio-system
kubectl get svc -n istio-system
```

# istio 삭제
```
kubectl delete iop -n istio-system example-istiocontrolplane
```

# istio operator 삭제
```
istioctl operator remove
istioctl manifest generate | kubectl delete -f -
```
