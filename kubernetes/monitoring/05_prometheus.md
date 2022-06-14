# Ch04_05. [실습] Prometheus와 Grafan를 통한 쿠버네티스 모니터링
이번 시간에는 kube-prometheus 프로젝트를 통해 프로메테우스, 그라파나, alertmanager, node-exporter등 프로메테우스와 관련된 서비스를 설치하고
 사용해본다.

kube-prometheus는 Prometheus-operator를 기본으로 프로메테우스, Alertmanger, node-exporter, kube-state-metrics, grafana 애플리케이션과 관련 설정을 제공하는 프로젝트로, 간편한 설정을 통해 프로메테우스를 관리할 수 있다.

# kube-prometheus 저장소
https://github.com/prometheus-operator/kube-prometheus

# 지원 버전 확인
https://github.com/prometheus-operator/kube-prometheus#compatibility

# 저장소 복사

```
git clone https://github.com/prometheus-operator/kube-prometheus.git -b release-0.10
```

# Manifest, CRD

```
cd kube-prometheus
ls manifests -l
ls manifests/setup -l
```

# 설치

```
kubectl create -f ./manifests/setup/
kubectl create -f ./manifests/
```

# pod 확인

```
kubectl -n monitoring get pods
```

# 서비스 연결
prometheus와 grafana의 svc타입을 `LoadBalancer`로 추가하여 다시 kubernetes에 적용한다.

```
vi manifests/grafana-service.yaml
vi manifests/prometheus-service.yaml
# type 추가
type: LoadBalancer

kubectl apply -f manifests/grafana-service.yaml
kubectl apply -f manifests/prometheus-service.yaml
```

# 서비스 확인

```
kubectl get svc -n monitoring
```

# 프로메테우스 UI

```
kubectl get svc -n monitoring prometheus-k8s -o jsonpath={.status.loadBalancer.ingress[].ip}
```
Web UI에 접속하여, Graph, Status->Target, Rules를 살표본다.


# 프로메테우스 서비스 모니터: 무엇을 모니터링 하나
Web UI에서 Status-Target에 표시되는 것이 ServiceMonitor이다. 이는 처음 설치한 manifests에서 `serviceMonitor`라는 파일로 설정되어있는데, kubectl 명령으로 현재 생성된 서비스모니터를 확인 할 수 있다.
```
ls manifests/*serviceMonitor* -alh

kubectl get servicemonitor -n monitoring
kubectl get servicemonitor -n monitoring -o yaml node-exporter
kubectl get servicemonitor -n monitoring -o yaml node-exporter
```

# metrics 확인
node-exporte 메트릭을 확인해본다.

```
sudo ss -lp|grep 9100
curl http://localhost:9100/metrics
```

# Grafana
그라파나는 프로메테우스에 수집되는 메트릭을 대시보드를 통해 시각화 페이지를 제공한다.
우리가 모니터링 화면이라고 알고있는 다양한 그래프를 그라파나를 통해 볼 수있다.

## 서비스 접근
```
kubectl get svc -n monitoring
```

## Login, 비밀번호 설정
최초 계정정보는 `admin`/`admin`이며, 새 비밀번호를 설정할 수 있다.

## 대시보드
kube-prometheus에서 기본으로 제공하는 대시보드가 있는데 이를 통해 쿠버네티스 클러스터에 대한 모니터링 화면을 확인할 수 있다.
