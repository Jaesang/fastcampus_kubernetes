# Ch04_013. [실습] Istio를 통한 Service Mesh 구현
istio는 

## istioctl 설치
```
mkdir istio-installation;cd istio-installation
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.13.4 sh -
export PATH="$PATH:/home/jaesanglee/istio-installation/istio-1.13.4/bin"
istioctl version
```

# istio-operator 설치
```
istioctl operator init
Installing operator controller in namespace: istio-operator using image: docker.io/istio/operator:1.13.4
Operator controller will watch namespaces: istio-system
✔ Istio operator installed
✔ Installation complete

kubectl get ns
kubectl get po -n istio-operator
```

# istio 설치
istio-demo.yaml 생성
```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: demo
``` 
`kubectl apply -f istio-demo.yaml`

# istio 설치
istioctl install
kubectl get po -n istio-system

# istio-ingress 설치
kubectl create namespace istio-ingress
kubectl label namespace istio-ingress istio-injection=enabled
helm install istio-ingress istio/gateway -n istio-ingress --wait



# istio 삭제
istioctl x uninstall --purge
kubectl delete -f istio-installation/istio-1.13.4/samples/addons/kiali.yaml
kubectl delete -f istio-installation/istio-1.13.4/samples/addons/jaeger.yaml

# istio 설치 by helm
#
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

helm install istio-base istio/base -n istio-system

# Sample App 삭제
kubectl delete -f  https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml -n msa
kubectl delete -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/istio-manifests.yaml -n msa
kubectl delete ns msa

# 카나리 배포테스트
기존 productcatalogservice에 label 버전 추가
```
kubectl patch deployments/productcatalogservice -p '{"spec":{"template":{"metadata":{"labels":{"version":"v1"}}}}}' -n msa
kubectl describe deploy productcatalogservice -n msa
```
destinationRule 추가
```
kubectl apply -f destinationrule.yaml -n msa
```
productcatalog-v2 추가
```
kubectl apply -f productcatalog_v2.yaml -n msa
```
트래픽 분배
```
kubectl apply -f vs-split-traffic.yaml -nmsa
```

