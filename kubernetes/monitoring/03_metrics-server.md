# Ch04_03. [실습] metrics-server를 활용한 리소스 모니터링

Metrics-server는 쿠버네티스 모니터링에서 리소스 메트릭 파이프라인으로 개발된 모니터링 도구로써
kubelet을 통해 리소스 메트릭을 수집하고 이를 kube-apiserver의 `metrics` API로 제공합니다.

Metrics server는 쿠버네티스에서 자원의 사용량에 따라 Pod의 개수나 용량을 조절하는 오토스케일을 위해
사용되는데, Pod개수를 조절하는 HPA, Pod의 용량을 조절하는 VPA가 metrics server가 제공하는 Pod의
CPU, Memory 사용량을 보고, 오토스케일 룰에 따라 Pod를 변경시킵니다.

## url

https://github.com/kubernetes-sigs/metrics-server

# 설치

```
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl apply -f components.yaml
```

# 실행
```
kubectl top node
```

# API 리퀘스트

```
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" |jq '.'
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes/worker-4" |jq '.'
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/kube-system/pods" |jq '.'
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/kube-system/pods/kube-scheduler-controller-2" |jq '.'
```

# hpa 테스트

https://kubernetes.io/ko/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/

## 테스트 deployment 설치 

```
kubectl apply -f https://k8s.io/examples/application/php-apache.yaml

kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

## 부하 발생

```
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```
