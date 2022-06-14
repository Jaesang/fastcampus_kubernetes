# Ch04_06. [실습] Prometheus AlarmManager을 통한 알람 수신

프로메테우스 UI에서 Alerts를 보면 여러 알림 내역을 확인할 수 있는데, 이는 프로메테우스가 수집하는 메트릭을 기반으로 어떠한 조건에 부합되는 상황이 유지되면
이를 알림으로 보내도록 룰 설정이 되어있기 때문에 알람이 발생한다. 
프로메테우스에서 설정된 알람 룰은 Status - Rules를 통해 어떤 알림이 등록되었는지 알 수 있다.
이러한 룰 설정은 kube-prometheus의 매니페스트를 통해 사전에 등록한 것이다.

```
kubectl get prometheusrules -n monitoring
kubectl get prometheusrules -n monitoring -o yaml node-exporter-rules
vi manifests/nodeExporter-prometheusRule.yaml
```

Prometheus는 정해진 알람의 룰에 따라 알람을 발생시키며, 이를 여러 메시지 수단으로 전달하는 것은 `alertmanager`의 역할이다.

# AlertManager
alertmanager는 알람 발생 시 정해진 메시지 수단으로 알람을 보내는 역할을 수행한다.
prometheus-operator에서 제공하는 alertmanager의 설정은 `manifests/alertmanager-secret.yaml`를 통해 설정한다.

```
vi manifests/alertmanager-secret.yaml

kubectl apply -f manifests/alertmanager-secret.yaml
```
이 설정파일에 slack에 대한 설정을 추가하면 이제 알림을 슬랙으로 받을 수 있다.

# Slack으로 알림 보내기
alertmanager 설정에 slack을 메시지 수단으로 추가할 수 있다. 이를 위해 slack의 설정이 필요한데, 테스트용으로 자신의 슬랙 채널의 설정에서 수신 webhook을 활성화하고 webhook url을 획득해야한다.

webhook을 활성화하고, 메시지를 보낼 채널을 설정한다.

## alertmanager 설정
위의 webhook정보를 이용해 alertmanager를 설정한다.
```
vi manifests/alertmanager-secret.yaml
kubectl apply -f manifests/alertmanager-secret.yaml
```

## 테스트용 Rule 설정
slack으로 보낼 테스트 알람 룰을 생성한다. default 네임스페이스의 Pod의 컨테이너를 모니터링하면서 CPU사용율이 25%를 넘어가면 알람을 발생한다
.
`manifests/kubernetesControlPlane-prometheusRule.yaml`에서 `CPUThrottlingHigh` 참고

```
kubectl get prometheusrules -n monitoring
vi test-prometheusRule.yaml
spec:
  groups:
  - name: test
    rules:
    - alert: CPUThrottlingHighTest
      annotations:
        description: '{{ $value | humanizePercentage }} throttling of CPU in namespace
          {{ $labels.namespace }} for container {{ $labels.container }} in pod {{
          $labels.pod }}.'
        runbook_url: https://runbooks.prometheus-operator.dev/runbooks/kubernetes/cputhrottlinghigh
        summary: Processes experience elevated CPU throttling.
      expr: |
        sum(increase(container_cpu_cfs_throttled_periods_total{container!="", namespace="default" }[5m])) by (container, pod, namespace)
          /
        sum(increase(container_cpu_cfs_periods_total{}[5m])) by (container, pod, namespace)
          > ( 25 / 100 )
      for: 1m
      labels:
        severity: critical

kubectl apply -f test-prometheusRule.yaml
```
prometheusRule을 생성하고, 프로메데우스 ui에서 이를 적용되는지 기다린다.

## 부하발생
`metrics-server` 실습에서 사용한 loadgenerator를 사용해 pod에 부하를 주면서 알림 발생을 테스트한다.
