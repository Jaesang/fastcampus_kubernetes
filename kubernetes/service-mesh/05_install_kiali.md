# Ch04_017. [실습] Kiali 설치
`kiali`는 istio의 서비스메시 설정과 검증기증을 갖춘 모니터링 콘솔입니다. 서비스, 앱, 워크로드 간의 트래픽 흐름을 모니터링하여 토폴로지를 추론하고, 오류를 보고함으로써 서비스 메시의 구조와 상태를 이해할 수 있습니다. 

## `kiali` 설치
`kiali`는 istio 저장소의 `samples/addons/kiali.yaml` 파일로 설치할 수 있습니다.
```
cd istio-installation/istio-1.13.4/
kubectl apply -f samples/addons/kiali.yaml
```

## `kiali` 접속
`kiali`는 웹 UI를 제공하며, 쿠버네티스 서비스 타입을 ClusterIp에서 LoadBalancer로 변경하여 접속합니다. 포트는 20001입니다.
```
kubectl edit svc -n istio-sysem kiali
...
  type: LoadBalancer
...
```

### Kiali의 Prometheus 연동
`kiali`의 configMap에 Prometheus 정보를 추가합니다. 
```
kubectl edit configmap kiali -n istio-system
...
data:
  config.yaml: |
  ...
    external_services:
      prometheus:
        url: "http://prometheus-k8s.monitoring:9090/"
      grafana:
        enabled: true
        in_cluster_url: 'http://grafana.monitoring:3000/'
        url: 'http://34.64.222.84:3000'
...
```
## Prometheus 연동
`kiali`는 `Prometheus`에서 수집된 메트릭을 사용하여 그래프와 관련 정보를 표시하기 때문에 Prometheus에서 MSA 어플리케이션의 대한 메트릭을 수집해야합니다. 

### prometheus-operator.yaml
`istio`의 `sample/addons/extra/prometheus-operator.yaml`로 prometheus-operator 환경에서 istio를 모니터링 할 수 있는 `PodMonitor`,`ServiceMonitor` 리소스를 생성할 수 있습니다.
```
kubectl apply -f samples/addons/extras/prometheus-operator.yaml
```

### prometheus rbac
프로메테우스가 istio의 Envoy proxy를 모니터링하기 위해서 프로메테우스의 클러스터 롤이 추가되어야합니다. `sample/addons/prometheus.yaml`에서 clusterrole과 clusterrolebinding을 변경하여 생성합니다. 이 때 clusterrole에 적용할 서비스 어카운트는 Prometheus operator로 설치한 prometheus 어카운트인 `prometheus-k8s`로 변경합니다.
```
---
# Source: prometheus/templates/server/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    component: "server"
    app: prometheus
    release: prometheus
    chart: prometheus-15.0.1
    heritage: Helm
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus-k8s
    namespace: monitoring
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
```

Prometheus가 정상적으로 연동되면, Kiali에서 Graph정보가 표시됩니다.

