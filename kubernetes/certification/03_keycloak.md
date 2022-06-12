# Ch04_25. [실습] Keycloak을 이용한 사용자 권한설정
Keycloak은 오픈소스 OpenID Connector로써 쿠버네티스의 사용자로 해석될 수 있는 token을 생성한다. keycloak을 통해 유저와 그룹을 생성하고 이를 쿠버네티스에서 사용할 수 있도록 연동하는 실습을 진행한다.

## Keycloak 설치
```
mkdir keycloak; cd keycloak
wget https://raw.githubusercontent.com/keycloak/keycloak-quickstarts/latest/kubernetes-examples/keycloak.yaml

kubectl create ns keycloak
kubectl label namespace keycloak istio-injection=enabled
kubectl create -f keycloak.yaml -n keycloak

```
## Ingress 설정
nip.io를 이용해 인증서를 생성하고, `istio-gateway`를 설정한다.

인증서 생성
```
$ cat cert-prd.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: keycloak-cert-prd
  namespace: istio-system
spec:
  secretName: keycloak-cert-prd
  commonName: keycloak.34.64.78.98.nip.io
  dnsNames:
  - keycloak.34.64.78.98.nip.io
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer

kubectl apply -f cert-prd.yaml -n istio-system
```

게이트웨이 설정
```
$ cat istio-gateway-keycloak.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: keycloak-gateway-tls
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "keycloak.34.64.78.98.nip.io"
    tls:
      credentialName: keycloak-cert-prd
      mode: SIMPLE
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: keycloak-ingress
spec:
  hosts:
  - "keycloak.34.64.78.98.nip.io"
  gateways:
  - keycloak-gateway-tls
  http:
  - route:
    - destination:
        host: keycloak

kubectl apply -f istio-gateway-keycloak.yaml -n keycloak
```

keycloak URL: keycloak.34.64.78.98.nip.io

## Realm 설정
1. realm 생성: 이름 fastcampus
2. client 추가: kuberntes-client
3. role 추가: admin
4. Mapper 추가: 
  - 이름: groups
  - Mapper Type: User Client Role
  - Client ID: kuberntes-client
  - Cliet Role prefix: kubernetes:
  - Token Claim Name: groups
5. Group 생성: admin
6. USer 생성: jaesang, 비밀번호 설정, group 설정, 유저활성화

## 접속 테스트
token URL로 계정정보를 입력하여 id_token을 발급받는지 테스트.
```
curl -X POST https://keycloak.34.64.78.98.nip.io/realms/fastcampus/protocol/openid-connect/token -d grant_type=password -d client_id=kubernetes-client -d username=jaesang -d password="Votmxmzoavjtm^" -d scope=openid

{"access_token":"eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICI2SEVMUHZjeGFfNTRud0sxb2N6TDZxbV8zajc3a1pmZnI2TXVOdkdGVFpRIn0.eyJleHAiOjE2NTQ5NjYzMTIsImlhdCI6MTY1NDk2NjAxMiwianRpIjoiYjllMDc0NjUtNjFlYS00YWY3LTgwYmUtZWZmNjQwNTU2YjdmIiwiaXNzIjoiaHR0cHM6Ly9rZXljbG9hay4zNC42NC43OC45OC5uaXAuaW8vcmVhbG1zL2Zhc3RjYW1wdXMiLCJhdWQiOiJhY2NvdW50Iiwic3ViIjoiYWRlZmEyYzMtMWNhMi00NGI4LWIxMDEtMTFlMzg4YmQ2ZGFiIiwidHlwIjoiQmVhcmVyIiwiYXpwIjoia3ViZXJuZXRlcy1jbGllbnQiLCJzZXNzaW9uX3N0YXRlIjoiMTVkYjZhZDgtZmIxMy00NTc0LWFiMzItMGI0MTE1YTQyOTA3IiwiYWNyIjoiMSIsInJlYWxtX2FjY2VzcyI6eyJyb2xlcyI6WyJvZmZsaW5lX2FjY2VzcyIsInVtYV9hdXRob3JpemF0aW9uIiwiZGVmYXVsdC1yb2xlcy1mYXN0Y2FtcHVzIl19LCJyZXNvdXJjZV9hY2Nlc3MiOnsia3ViZXJuZXRlcy1jbGllbnQiOnsicm9sZXMiOlsiYWRtaW4iXX0sImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoib3BlbmlkIHByb2ZpbGUgZW1haWwiLCJzaWQiOiIxNWRiNmFkOC1mYjEzLTQ1NzQtYWIzMi0wYjQxMTVhNDI5MDciLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsImdyb3VwcyI6WyJrdWJlcm5ldGVzOmFkbWluIl0sInByZWZlcnJlZF91c2VybmFtZSI6ImphZXNhbmcifQ.BGcGamVgdjw5FOIlx5ibICcTg6gIFj6p8xEn1-2VPjvVNhKR0el92oZYb_PBQA_ZPDpu75G57VsFw1wEZ42VKxl46bV1Kxi21b1MA3PpPA2wfKK5fdlqGyvS_K6UGeCULTat_MtM9xlgJryo_HtiAXw5QBi7-AWYtgTPsogCWJ_trCq2H-kCCiPn_MnK29a8F1Id3R5x9UdBXxSVLUdLrwl1pBOHUE29xwQUB5LsJf32dIEoD9dvAYEmHVatsXp7sFfAhb7qFF7MKSBhk1BO3PdeQt4pgo__CtrZjRWEQRzw2X_s7Lenp76uxNsyZwTRxJbCVoXCuStco0NbXCe8Lw","expires_in":300,"refresh_expires_in":1800,"refresh_token":"eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICIwOWE0Njk1My05N2MyLTQxYmUtYjNlZS0xMjY5ZDRmYTZmN2YifQ.eyJleHAiOjE2NTQ5Njc4MTIsImlhdCI6MTY1NDk2NjAxMiwianRpIjoiYjc5OWZhY2ItNGFhMC00NjE3LTlkY2YtMTEzNWU2MGZmZmI3IiwiaXNzIjoiaHR0cHM6Ly9rZXljbG9hay4zNC42NC43OC45OC5uaXAuaW8vcmVhbG1zL2Zhc3RjYW1wdXMiLCJhdWQiOiJodHRwczovL2tleWNsb2FrLjM0LjY0Ljc4Ljk4Lm5pcC5pby9yZWFsbXMvZmFzdGNhbXB1cyIsInN1YiI6ImFkZWZhMmMzLTFjYTItNDRiOC1iMTAxLTExZTM4OGJkNmRhYiIsInR5cCI6IlJlZnJlc2giLCJhenAiOiJrdWJlcm5ldGVzLWNsaWVudCIsInNlc3Npb25fc3RhdGUiOiIxNWRiNmFkOC1mYjEzLTQ1NzQtYWIzMi0wYjQxMTVhNDI5MDciLCJzY29wZSI6Im9wZW5pZCBwcm9maWxlIGVtYWlsIiwic2lkIjoiMTVkYjZhZDgtZmIxMy00NTc0LWFiMzItMGI0MTE1YTQyOTA3In0.ysIRUMBZPUI5GlH-vU5msXwOaU7YmbDXRWAAnv3ZoP8","token_type":"Bearer","id_token":"eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICI2SEVMUHZjeGFfNTRud0sxb2N6TDZxbV8zajc3a1pmZnI2TXVOdkdGVFpRIn0.eyJleHAiOjE2NTQ5NjYzMTIsImlhdCI6MTY1NDk2NjAxMiwiYXV0aF90aW1lIjowLCJqdGkiOiJiNGRmZjQ2YS1mYWEyLTQ2MmMtYTY3OC1mZTg1NjA2OTM4MTIiLCJpc3MiOiJodHRwczovL2tleWNsb2FrLjM0LjY0Ljc4Ljk4Lm5pcC5pby9yZWFsbXMvZmFzdGNhbXB1cyIsImF1ZCI6Imt1YmVybmV0ZXMtY2xpZW50Iiwic3ViIjoiYWRlZmEyYzMtMWNhMi00NGI4LWIxMDEtMTFlMzg4YmQ2ZGFiIiwidHlwIjoiSUQiLCJhenAiOiJrdWJlcm5ldGVzLWNsaWVudCIsInNlc3Npb25fc3RhdGUiOiIxNWRiNmFkOC1mYjEzLTQ1NzQtYWIzMi0wYjQxMTVhNDI5MDciLCJhdF9oYXNoIjoiZjcwNUJBWW9acTZGSDVUZ1hfamVIUSIsImFjciI6IjEiLCJzaWQiOiIxNWRiNmFkOC1mYjEzLTQ1NzQtYWIzMi0wYjQxMTVhNDI5MDciLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsImdyb3VwcyI6WyJrdWJlcm5ldGVzOmFkbWluIl0sInByZWZlcnJlZF91c2VybmFtZSI6ImphZXNhbmcifQ.APsuSE2Aov1-z7HTKnmvg1sCVwGarZZf8TqwjNXdVZOqAz2sCH3O66PFK44MW5giw3CpNTvKl_laVPPin08u1AcwoIeGjLqZ_XYTKUjsfC8D4Mwsi9vHqo6fFr642D837NG1-bGWROYKD8G3MM2E9B5YiqmO6Xrgn-9R0TiKQOu3EzXhd7zn8ZCxLoHUgNtphdZ2I_NC_JRx8YwmcxelYMHebday75ONw7NsSgPOE9m03pETk14P8yYkBCiNVHYybnUxgV7JEj_7g3OOPA4AYHSgwlZZ04k3-wx1om58kjNcOL7rs57XqhZV8lTrafnGJcFzwKUmgz3bZYd3rpjacg","not-before-policy":0,"session_state":"15db6ad8-fb13-4574-ab32-0b4115a42907","scope":"openid profile email"}
```
전달된 token중 id_token을 jwt.io에서 해석한다.
```
{
  "exp": 1654966312,
  "iat": 1654966012,
  "auth_time": 0,
  "jti": "b4dff46a-faa2-462c-a678-fe8560693812",
  "iss": "https://keycloak.34.64.78.98.nip.io/realms/fastcampus",
  "aud": "kubernetes-client",
  "sub": "adefa2c3-1ca2-44b8-b101-11e388bd6dab",
  "typ": "ID",
  "azp": "kubernetes-client",
  "session_state": "15db6ad8-fb13-4574-ab32-0b4115a42907",
  "at_hash": "f705BAYoZq6FH5TgX_jeHQ",
  "acr": "1",
  "sid": "15db6ad8-fb13-4574-ab32-0b4115a42907",
  "email_verified": false,
  "groups": [
    "kubernetes:admin"
  ],
  "preferred_username": "jaesang"
}
```

## Kubernetes 설정
Kube-apiserver에서 Keycloak에서 발행한 token을 사용할 수 있도록 설정
```
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --oidc-issuer-url=https://keycloak.34.64.78.98.nip.io/realms/fastcampus
    - --oidc-client-id=kubernetes-client
    - --oidc-username-claim=preferred_username
    - --oidc-username-prefix=-
    - --oidc-groups-claim=groups
```
## kubelogin 설치
kubelogin은 OIDC 인증을 쉽게 할 수 있는 kubectl plugin으로 keycloak Web UI에 로그인을 함으로써, 쿠버네티스에 API를 요청할 수 있다.
https://github.com/int128/kubelogin
```
brew install int128/kubelogin/kubelogin
```
kubelogin 설정: 브라우저로 유저 로그인을 하면 인증 성공
```
kubectl oidc-login setup --oidc-issuer-url=https://keycloak.34.64.78.98.nip.io/realms/fastcampus --oidc-client-id=kubernetes-client --oidc-client-secret=tetqg
## 2. Verify authentication

You got a token with the following claims:

{
  "exp": 1654967430,
  "iat": 1654967130,
  "auth_time": 1654967130,
  "jti": "b9cae6c7-855b-46ec-a809-7b015abfab6b",
  "iss": "https://keycloak.34.64.78.98.nip.io/realms/fastcampus",
  "aud": "kubernetes-client",
  "sub": "adefa2c3-1ca2-44b8-b101-11e388bd6dab",
  "typ": "ID",
  "azp": "kubernetes-client",
  "nonce": "MSMJNlJnGcFUoZzR7jcMvNIgO14kKoP_HRjuTzKevqU",
  "session_state": "ac919ab1-3439-4131-a522-5213b03cc283",
  "at_hash": "ZqJQzT7WbTWIUt33XU8IxA",
  "acr": "1",
  "sid": "ac919ab1-3439-4131-a522-5213b03cc283",
  "email_verified": false,
  "groups": [
    "kubernetes:admin"
  ],
  "preferred_username": "jaesang"
}

## 3. Bind a cluster role

Run the following command:

	kubectl create clusterrolebinding oidc-cluster-admin --clusterrole=cluster-admin --user='https://keycloak.34.64.78.98.nip.io/realms/fastcampus#adefa2c3-1ca2-44b8-b101-11e388bd6dab'

## 4. Set up the Kubernetes API server

Add the following options to the kube-apiserver:

	--oidc-issuer-url=https://keycloak.34.64.78.98.nip.io/realms/fastcampus
	--oidc-client-id=kubernetes-client

## 5. Set up the kubeconfig

Run the following command:

	kubectl config set-credentials oidc \
	  --exec-api-version=client.authentication.k8s.io/v1beta1 \
	  --exec-command=kubectl \
	  --exec-arg=oidc-login \
	  --exec-arg=get-token \
	  --exec-arg=--oidc-issuer-url=https://keycloak.34.64.78.98.nip.io/realms/fastcampus \
	  --exec-arg=--oidc-client-id=kubernetes-client \
	  --exec-arg=--oidc-client-secret=test

## 6. Verify cluster access

Make sure you can access the Kubernetes cluster.

	kubectl --user=oidc get nodes

You can switch the default context to oidc.

	kubectl config set-context --current --user=oidc

You can share the kubeconfig to your team members for on-boarding.
```

## kubeconfig 설정
```
kubectl config get-contexts
kubectl config set-credentials oidc \
	  --exec-api-version=client.authentication.k8s.io/v1beta1 \
	  --exec-command=kubectl \
	  --exec-arg=oidc-login \
	  --exec-arg=get-token \
	  --exec-arg=--oidc-issuer-url=https://keycloak.34.64.78.98.nip.io/realms/fastcampus \
	  --exec-arg=--oidc-client-id=kubernetes-client \
	  --exec-arg=--oidc-client-secret=test
kubectl --user=oidc get nodes
Error from server (Forbidden): nodes is forbidden: User "jaesang" cannot list resource "nodes" in API group "" at the cluster scope
```

## 롤바인딩

Admin 권한
```
kubectl create clusterrolebinding oidc-cluster-admin --clusterrole=cluster-admin --user=jaesang
kubectl --user=oidc get nodes
```

Configmap 조회만 부여
```
kubectl delete clusterrolebinding oidc-cluster-admin
cat clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: only-configmap
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - get
  - list
  - watch
kubectl create -f clusterrole.yaml
kubectl create clusterrolebinding only-configmap --clusterrole=only-configmap --user=jaesang
kubectl --user=oidc get nodes
kubectl --user=oidc get secrets
kubectl --user=oidc get po
kubectl --user=oidc get cm
```
## 리소스 삭제
```
kubectl delete clusterrolebinding only-configmap
kubectl delete -f clusterrole.yaml
kubectl delete clusterrolebinding oidc-cluster-admin
kubectl delete -f istio-gateway-keycloak.yaml -n keycloak
kubectl delete -f cert-prd.yaml -n istio-system
kubectl delete -f keycloak.yaml -n keycloak
kubectl delete ns keycloak
```