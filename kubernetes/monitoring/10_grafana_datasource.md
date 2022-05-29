# Ch04_010. [실습] Prometheus와 Grafana 연동
kube-prometheus 프로젝트에서 Grafana와 Prometheus를 연동하는 방법

## kube-prometheus 저장소
https://github.com/prometheus-operator/kube-prometheus

## Datasource
grafana에서 메트릭 데이터를 수집하는 타겟을 데이터소스라 하는데, 프로메테우스도 grafana에서 지원하는 다양한 데이터 소스 중 하나이다. 따라서 grafana에서 프로메테우스의 메트릭을 활용해 대시보드를 사용하기 위해서는 프로메테우스를 데이터소스로 등록해야한다.

## grafana-dashboardDatasources.yaml
`grafana-dashboardDatasources.yaml`는 kube-prometheus에서 제공하는 `secret`리소스로 `grafana-deployment.yaml`이 실행될 때 볼륨으로 마운트하여 사용한다. `secret`리소스에는 `datasources.yaml`이라는 데이터가 저장되어 있는데, 데이터소스로써 프로메테우스 이름과 URL정보를 담고있다. 
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
이 정보를 바탕으로 Grafana와 Prometheus가 연동되며, 이는 Grafana Web UI에서 `Configuration`, `Data sources`메뉴에서 확인할 수 있다. 

## Grafana Web UI에서 확인하기
Grafana Web UI에서 `Configuration`, `Data sources`메뉴에서 현재 설정되어 있는 데이터소스에 대한 설정, 그리고 새로운 데이터소스를 추가할 수 있다.

처음 설치시 바로 Prometheus와 연동하기 위해 `grafana-dashboardDatasources.yaml`를 제공하고 있기 때문에, Web UI에 들어가면 프로메테우스 데이터소스가 추가된 것을 확인할 수 있다.

프로메테우스 데이터소스는 Web UI를 통해 추가한 데이터소스가 아닌, grafana config를 통해 최초 실행할 때부터 적용된 데이터소스이기 때문에 Web UI에서 수정은 할 수 없고, 맨 아래 Test 버튼을 통해 연결상태를 확인할 수 있다.

## Grafana Service Monitor
프로메테우스에서 Grafana의 메트릭을 수집할 수 있도록 서비스 모니터 `grafana-serviceMonitor.yaml`을 사용, Grafana는 어플리케이션 자체적으로 프로메테우스용 메트릭 url을 제공하고 있기 때문에 서비스 모니터만 등록하면 프로메테우스에서 바로 Grafana의 모니터링 메트릭을 사용 가능

## Prometheus Web UI에서 확인하기
프로메테우스 Web UI에서 Status->Service Monitor, Target에서 Grafana 서비스모니터를 확인할 수 있고, Graph 메뉴를 통해 직접 메트릭을 확인할 수 있다.

# 마무리
`kube-prometheus`는 Grafana에 Prometheus를 연동할 수 있도록 데이터소스 내용을 담은 secret이 제공되며, 또 Prometheus에서 Grafana를 모니터링할 수 있도록 serviceMonitor리소스 또한 제공되고 있기 때문에, 설치와 동시에 두개의 어플리케이션을 바로 연동하여 사용 가능.  