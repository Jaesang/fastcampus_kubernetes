1. canary 샘플 제거
kubectl delete -f canary/ -n msa
2. msa 제거
cd ~/microservices-demo/
kubectl delete -f release/istio-manifests.yaml -n msa
kubectl delete -f release/kubernetes-manifests.yaml -n msa
3. kiali , jaeger 제거
cd ~/istio-installation/istio-1.13.4/
kubectl delete -f samples/addons/kiali.yaml
kubectl delete -f samples/addons/jaeger.yaml
4. prometheus 관련설정 제거
kubectl delete -f ~/istio/prometheus-rbac.yaml
kubectl delete -f ~/istio-installation/istio-1.13.4/samples/addons/extras/prometheus-operator.yaml
5. istio 제거
istioctl x uninstall --purge

6. 네임스페이스 제거
kubectl delete ns msa
kubectl delete ns istio-system
kubectl delete ns istio-operator