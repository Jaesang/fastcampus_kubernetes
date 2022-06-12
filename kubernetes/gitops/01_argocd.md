# Ch05_03. [실습] ArgoCD를 통한 GitOps 구축

## ArgoCD 설치
```
mkdir ~/argocd;cd ~/argocd
wget https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl create ns argocd
kubectl apply -f install.yaml -n argocd
```
리소스 확인
```
$ k get po
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          55s
argocd-applicationset-controller-66689cbf4b-p7gnm   1/1     Running   0          55s
argocd-dex-server-66fc6c99cc-hlm8l                  1/1     Running   0          55s
argocd-notifications-controller-8f8f46bd6-4dnnk     1/1     Running   0          55s
argocd-redis-d486999b7-cdntq                        1/1     Running   0          55s
argocd-repo-server-7db4cc4b45-mq8zp                 1/1     Running   0          55s
argocd-server-79d86bbb47-9bpb7                      1/1     Running   0          55s
```

초기 비밀번호
admin 유저에 대한 비밀번호 확인
```
k get secret -o yaml argocd-initial-admin-secret|yq '.data.password'|base64 -d
```
웹 UI 접근
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

## Application 등록
https://github.com/argoproj/argocd-example-apps

1. Create App
2. Name: Guestbook
3. URL: https://github.com/Jaesang/argocd-example-apps
4. PATH: guestbook
5. SYNC OPTION: Auto-Create Namespace


## GitOps 테스트

1. SVC 변경: Service Type을 LoadBalancer로 변경
2. Pull Request: PR을 통해 코드 리뷰
3. Merge: 리뷰 승인을 통해 Git source 배포 확인
4. 클러스터에서 직접 수정: Git이 아닌 직접 클러스터의 리소스 변경 후 상태 확인
4. Github Revert: 배포된 어플리케이션의 이전 커밋으로 상태 변경

## 리소스 삭제
```
kubectl delete -f ~/argocd/install.yaml -n argocd
kubectl delete ns argod
rm -rf ~/argocd
```