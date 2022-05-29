# Ch04_09. [실습] Grafana 설정
kube-prometheus 프로젝트에서 Grafana를 설정하는 방법에 대해 알아본다.

## kube-prometheus 저장소
https://github.com/prometheus-operator/kube-prometheus

## Grafana 서비스 설정
Grafana 어플리케이션을 사용하기 위해서는 외부에서 접속할 수 있도록 서비스 설정이 필요하다. `grafana-service.yaml`은 기본적으로 clusterIP로 설정되어있는데, LoadBalancer나 NodePort, 혹은 Ingress를 통해 외부에서 접속할 수 있도록 변경한다.
```
$ cat grafana-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 8.3.3
  name: grafana
  namespace: monitoring
spec:
  ports:
  - name: http
    port: 3000
    targetPort: http
  selector:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
  type: LoadBalancer

  $ kubectl apply -f grafana-service.yaml

```

## Grafana 설정파일
`grafana-service.yaml`처럼 Grafana 어플리케이션에서 사용하는 설정파일은 `kube-prometheus/manifests/grafana-*.yaml`파일을 참고한다.
```
grafana-config.yaml
grafana-dashboardDatasources.yaml
grafana-dashboardDefinitions.yaml
grafana-dashboardSources.yaml
grafana-deployment.yaml
grafana-service.yaml
grafana-serviceAccount.yaml
grafana-serviceMonitor.yaml
```

### grafana-config.yaml
Grafana 어플리케이션의 설정파일인 `grafana.ini`에 대해 설정하며, `Secret`리소스로 저장된다. `kube-prometheus`에서 제공하는 grafana.ini는 기본 타임존에 대해서만 설정하고 있는데, 다른 설정값에 대한 정보는 [grafana 공식문서](https://grafana.com/docs/grafana/latest/administration/configuration/)에서 확인할 수 있다.
```
apiVersion: v1
kind: Secret
metadata:
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 8.3.3
  name: grafana-config
  namespace: monitoring
stringData:
  grafana.ini: |
    [date_formats]
    default_timezone = UTC
type: Opaque
```
### grafana-deployment.yaml
grafana deployment 매니페스트, grafana를 실행시키기 위한 컨테이너 이미지와 `grafana.ini`, 그리고 제공되는 대시보드 설정파일이 명시되어 있다.

```
$ cat grafana-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 8.3.3
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: grafana
      app.kubernetes.io/name: grafana
      app.kubernetes.io/part-of: kube-prometheus
  template:
...

```
grafana 설정 파일들이 secret이나 configMap으로 제공되는 이를 볼륨으로 지정하고 컨테이너에 마운트하고 있다. 
```
     volumes:
      - emptyDir: {}
        name: grafana-storage
      - name: grafana-datasources
        secret:
          secretName: grafana-datasources
      - configMap:
          name: grafana-dashboards
        name: grafana-dashboards
      - configMap:
          name: grafana-dashboard-alertmanager-overview
        name: grafana-dashboard-alertmanager-overview
      - configMap:
          name: grafana-dashboard-apiserver
        name: grafana-dashboard-apiserver
      - configMap:
          name: grafana-dashboard-cluster-total
        name: grafana-dashboard-cluster-total
      - configMap:
          name: grafana-dashboard-controller-manager
        name: grafana-dashboard-controller-manager
      - configMap:
          name: grafana-dashboard-k8s-resources-cluster
        name: grafana-dashboard-k8s-resources-cluster
      - configMap:
          name: grafana-dashboard-k8s-resources-namespace
        name: grafana-dashboard-k8s-resources-namespace
      - configMap:
          name: grafana-dashboard-k8s-resources-node
        name: grafana-dashboard-k8s-resources-node
      - configMap:
          name: grafana-dashboard-k8s-resources-pod
        name: grafana-dashboard-k8s-resources-pod
      - configMap:
``` 
### grafana-dashboardDefinitions.yaml
grafana 대시보드를 configmap 리소스로 제공하며, 이를 grafana deployment에서 볼륨 마운트하여 사용한다. 23개의 대시보드가 json 형식으로 제공된다.

### grafana-dashboardDatasources.yaml
grafana에서 메트릭 데이터를 수집하는 시스템을 데이터소스라 하는데, 프로메테우스와 연동하기 위한 데이터소스 설정을 담고 있다. `secret`리소스로 생성되며, 이것을 grafana-deployment에서 마운트하여 실행된다.
```
apiVersion: v1
kind: Secret
metadata:
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 8.3.3
  name: grafana-datasources
  namespace: monitoring
stringData:
  datasources.yaml: |-
    {
        "apiVersion": 1,
        "datasources": [
            {
                "access": "proxy",
                "editable": false,
                "name": "prometheus",
                "orgId": 1,
                "type": "prometheus",
                "url": "http://prometheus-k8s.monitoring.svc:9090",
                "version": 1
            }
        ]
    }
type: Opaque
```

### grafana-dashboardSources.yaml
grafana 대시보드에서 사용할 기본 폴더를 지정한다. 여기서는 `Default`라는 폴더를 생성하는데, kube-prometheus에서 제공하는 23개의 grafana 대시보드가 `Default` 폴더 안에 위치하게 된다. 
```
apiVersion: v1
data:
  dashboards.yaml: |-
    {
        "apiVersion": 1,
        "providers": [
            {
                "folder": "Default",
                "folderUid": "",
                "name": "0",
                "options": {
                    "path": "/grafana-dashboard-definitions/0"
                },
                "orgId": 1,
                "type": "file"
            }
        ]
    }
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 8.3.3
  name: grafana-dashboards
  namespace: monitoring
```
### grafana-serviceAccount.yaml
Grafana Pod가 사용할 service account 리소스를 설정한다.
```
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 8.3.3
  name: grafana
  namespace: monitoring
```
### grafana-serviceMonitor.yaml
Prometheus에서 grafana 서비스를 모니터링할 수 있도록 serviceMonitor리소스를 설정한다.

```apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 8.3.3
  name: grafana
  namespace: monitoring
spec:
  endpoints:
  - interval: 15s
    port: http
  selector:
    matchLabels:
      app.kubernetes.io/name: grafana
```

 