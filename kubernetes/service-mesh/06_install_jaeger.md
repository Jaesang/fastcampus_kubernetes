# Ch04_018. [실습] Jaeger 설치
`jaeger`는 복잡한 분산 시스템에서 트래픽을 모니터링 하고 문제를 해결할 수 있도록 돕는 종단 간 분산 추적(tracing) 시스템입니다. 

## `jaeger` 설치
`jaeger`는 istio 저장소의 `samples/addons/jaeger.yaml` 파일로 설치할 수 있습니다.
```
cd istio-installation/istio-1.13.4/
kubectl apply -f samples/addons/jaeger.yaml
```

## `jaeger` 접속
`jaeger`는 웹 UI를 제공하며, 쿠버네티스 서비스 타입을 ClusterIp에서 LoadBalancer로 변경하여 접속합니다. 서비스이름은 `tracing`이며 기본 포트는 80입니다.
```
kubectl edit svc -n istio-sysem tracing
...
  type: LoadBalancer
...
```

## `jaeger` 삭제
설치 시 사용했던 `samples/addons/jaeger.yaml` 파일을 통해 삭제합니다.
```
cd ~/istio-installation/istio-1.13.4/
kubectl delete -f samples/addons/jaeger.yaml
```