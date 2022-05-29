# prometheus 설정
kube-prometheus 프로젝트에서 Prometheus를 설정하는 방법에 대해 알아본다.

## kube-prometheus 저장소
https://github.com/prometheus-operator/kube-prometheus

## prometheus 설정파일
kube-prometheus/manifests/prometheus-*.yaml파일을 참고한다.
```
cd kube-prometheus/manifests
ls prometheus-*.yaml -alh
prometheus-clusterRole.yaml
prometheus-clusterRoleBinding.yaml
prometheus-podDisruptionBudget.yaml
prometheus-prometheus.yaml
prometheus-prometheusRule.yaml
prometheus-roleBindingConfig.yaml
prometheus-roleBindingSpecificNamespaces.yaml
prometheus-roleConfig.yaml
prometheus-roleSpecificNamespaces.yaml
prometheus-service.yaml
prometheus-serviceAccount.yaml
prometheus-serviceMonitor.yaml
```

## prometheus 서비스 설정
prometheus 어플리케이션을 사용하기 위해서는 외부에서 접속할 수 있도록 서비스 설정이 필요하다. `prometheus-service.yaml`은 기본적으로 clusterIP로 설정되어있는데, `LoadBalancer`나 `NodePort`, 혹은 Ingress를 통해 외부에서 접속할 수 있도록 변경한다. 아래 파일은 `LoadBalancer`를 사용하도록 설정한 예시이다.

```
$ cat prometheus-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.32.1
  name: prometheus-k8s
  namespace: monitoring
spec:
  ports:
  - name: web
    port: 9090
    targetPort: web
  - name: reloader-web
    port: 8080
    targetPort: reloader-web
  selector:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
  sessionAffinity: ClientIP
  type: LoadBalancer
```

## prometheus-prometheus.yaml
prometheus CR을 설정한다.

![Prometheus Operator Architecture](https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/Documentation/user-guides/images/architecture.png "Prometheus Operator Architecture")

Prometheus CR을 통해 prometheus를 어떻게 배포할지, 프로메테우스에 적용할 서비스모니터, 파드모니터, 룰 CR의 레이블을 설정한다.

프로메테우스 어플리케이션을 배포할 때 replica 개수, 사용하는 컨테이너 이미지, request memory, security context등을 설정할 수 있다.

기본 설정은 아무런 레이블을 적용하지 않았기 때문에 모든 네임스페이스에 생성된 서비스모니터, 파드모니터, 룰등의 CR을 프로메테우스에 적용하는데, 특정 레이블을 갖고 있는 CR만 적용할 경우 셀렉터에 레이블을 설정할 수 있다.
```
spec:
  alerting:
    alertmanagers:
    - apiVersion: v2
      name: alertmanager-main
      namespace: monitoring
      port: web
  enableFeatures: []
  externalLabels: {}
  image: quay.io/prometheus/prometheus:v2.32.1
  nodeSelector:
    kubernetes.io/os: linux
  podMetadata:
    labels:
      app.kubernetes.io/component: prometheus
      app.kubernetes.io/instance: k8s
      app.kubernetes.io/name: prometheus
      app.kubernetes.io/part-of: kube-prometheus
      app.kubernetes.io/version: 2.32.1
  podMonitorNamespaceSelector: {}
  podMonitorSelector: {}
  probeNamespaceSelector: {}
  probeSelector: {}
  replicas: 2
  resources:
    requests:
      memory: 400Mi
  ruleNamespaceSelector: {}
  ruleSelector: {}
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  version: 2.32.1
```
### 예시) ServiceMonitor Namespace Selector 설정
prometheus가 모니터링 타겟을 특정 네임스페이스의 서비스모니터만 사용하도록 변경

`monitor=enbled` 레이블이 있는 네임스페이스의 서비스모니터만 사용할 경우
```
  serviceMonitorNamespaceSelector:
    matchLabels:
      monitor: enabled  
```

`prometheus-operator`와 `prometheus` log 확인
```
kubectl logs -n monitoring prometheus-operator-7ddc6877d5-ksbmt -f
kubectl logs -n monitoring prometheus-k8s-0
```
Prometheus Web UI의 Status->Service Discovery 메뉴 확인

## prometheus Rule
prometheusRule CR을 설정한다.
prometheus에서 alert이나 recording rule을 적용하기 위해서는 prometheusRule CR을 생성해야한다. manifests폴더에 `*-prometheusRule.yaml` 파일들을 보면 kube-prometheus에서 제공하는 prometheusRule CR 내용을 볼 수 있는데, 새로 CR을 생성하여 Rule을 추가하거나, 기존 Rule CR파일을 수정할 수 있다.
```
$ ls *-prometheusRule.yaml -1
alertmanager-prometheusRule.yaml
kubePrometheus-prometheusRule.yaml
kubeStateMetrics-prometheusRule.yaml
kubernetesControlPlane-prometheusRule.yaml
nodeExporter-prometheusRule.yaml
prometheus-prometheusRule.yaml
prometheusOperator-prometheusRule.yaml
```

## prometheus serviceMonitor
serviceMonitor CR을 설정한다.
prometheus에서 기본적으로 모니터링하는 대상을 쿠버네티스 서비스를 사용하여 검색한다. manifests폴더에 `*-serviceMonitor.yaml` 파일들을 보면 kube-prometheus에서 제공하는 serivce monitor CR 내용을 볼 수 있는데, 새로 CR을 생성하여 service monitor를 생성하거나, 기존 serivce monitor CR파일을 수정할 수 있다.

```
$ ls *-serviceMonitor.yaml -1
alertmanager-serviceMonitor.yaml
blackboxExporter-serviceMonitor.yaml
grafana-serviceMonitor.yaml
kubeStateMetrics-serviceMonitor.yaml
nodeExporter-serviceMonitor.yaml
prometheus-serviceMonitor.yaml
prometheusAdapter-serviceMonitor.yaml
prometheusOperator-serviceMonitor.yaml
```

# 마무리
`kube-prometheus`의 `Prometheus-operator`는 prometheus, prometheusRule, serviceMonitor Custrom Resource를 사용하고 있다. 따라서 프로메테우스를 설정하기 위해서는 
해당 리소스를 변경하고, 이를 kubectl로 적용하여 환경설정이 변경되는지`prometheus-operator`와 `prometheus` pod 로그를 통해 확인할 수 있다.

