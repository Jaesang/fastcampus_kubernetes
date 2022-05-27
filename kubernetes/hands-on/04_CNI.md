
# Ch03_09. [실습] Calico CNI 설치
쿠버네티스 클러스터에 CNI를 설치하여 Pod Network를 구축한다.

## CNI 설치
`calico` CNI를 설치하기 위해, 홈페이지에서 제공하는 매니페스트를 저장하여, 이를 kubectl로 실행합니다. calico pod이 모두 실행된 뒤에, kubectl get node 명령을 통해 node status가 Ready를 확인할 수 있습니다. 
```
curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O
kubectl apply -f calico.yaml
kubectl get po -A
kubectl get nodes
```

