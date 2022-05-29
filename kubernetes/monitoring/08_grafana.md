# Grafana 설치
kube-prometheus 프로젝트에서 Grafana를 설치하는 방법

## kube-prometheus 저장소
https://github.com/prometheus-operator/kube-prometheus

## Grafana Manifests
kube-prometheus/manifests/grafana-*.yaml파일을 참고한다.
```
cd kube-prometheus/manifests
ls grafana-*.yaml -1
grafana-config.yaml
grafana-dashboardDatasources.yaml
grafana-dashboardDefinitions.yaml
grafana-dashboardSources.yaml
grafana-deployment.yaml
grafana-service.yaml
grafana-serviceAccount.yaml
grafana-serviceMonitor.yaml
```
### grafana-deployment.yaml
grafana deployment 매니페스트, grafana를 실행시키기 위한 컨테이너 이미지와 제공되는 대시보드 설정파일이 명시되어 있다.

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
특히 미리 제공되는 대시보드를 모두 configmap으로 생성하고, 이를 컨테이너에 마운트하고 있다. 
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

## Grafana 설치
기본적으로 kube-prometheus에서 제공하는 manifests는 모두 쿠버네티스에 바로 실행될 수 있도록 작성되어 있기 때문에 kubectl apply 명령으로 바로 설치할 수 있다.
```
$ for f in grafana-*.yaml
do
  kubectl apply -f $f
done
```

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

## Grafana Login
grafana service를 통해 웹브라우저에서 접근할 IP를 획득한다. grafana의 서비스 포트는 3000이다.
```
  $ kubectl get svc -n monitoring grafana
NAME      TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)          AGE
grafana   LoadBalancer   10.32.0.238   54.164.69.14   3000:30597/TCP   11d
```
`http://<GRAFANA_IP>:3000`으로 접속하면, grafana 로그인 페이지가 뜨는데, 최초 admin 계정은 `admin`/`admin`으로 접속하여, 비밀번호를 초기화한다.



 