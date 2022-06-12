# Ch04_026. [실습] Kubernetes 3rd Party Tools
쿠버네티스를 사용할 때 유용한 툴에 대해 정리합니다.

## Lens
`Lens` https://k8slens.dev/ 는 쿠버네티스를 관리할 수 있는 설치형 GUI프로그램입니다. 맥, 윈도우, 리눅스에서 모두 사용할 수 있으며, kubeconfig를 사용해 클러스터를 등록하면 해당 클러스터의 대한 리소스를 모두 파악할 수 있습니다.

렌즈는 특정 리소스에 대해서는 렌즈에서 직접 리소스를 생성하고 수정하는것을 지원합니다. 롤과 롤바인딩, 시크릿 같은 리소스를 템플릿을 제공해주어서 보다 쉽게 생성하고 관리할 수 있습니다.


## k9s
`k9s` https://k9scli.io/ 는 CLI에서 대화형인터페이스를 제공하는 쿠버네티스 관리 툴입니다. 클러스터의 리소스 사용량, 리소스별 디펜던시 관계, Pod 상태와 로그, 그리고 RBAC 리소스를 확인할 수 있습니다.
```
:xray
:purse
```

## Kubernetes for Testing
데스크탑 혹은 저사양 서버에서도 쿠버네티스 클러스터를 설치하여 쿠버네티스 API를 테스트할 수 있습니다. 적은 리소스를 사용해 쿠버네티스 클러스터를 설치할 수 있는 다양한 프로젝트가 존재합니다.

1. [minikube](https://minikube.sigs.k8s.io/docs/start/)
2. [kind](https://kind.sigs.k8s.io/)
3. [k3d](https://k3d.io/), [k3s](https://github.com/k3s-io/k3s)
4. [micro8s](https://microk8s.io/)

Minikube example
```
minikube start
minikube status
minkube stop
```
## Krew
맥OS의 패키지 매니저인 brew의 이름과 유사한 쿠버네티스 플러그인 매니저 `Krew` https://krew.sigs.k8s.io/
194개의 kubectl plugin을 설치할 수 있습니다.
```
(   set -x; cd "$(mktemp -d)" &&   OS="$(uname | tr '[:upper:]' '[:lower:]')" &&   ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&   KREW="krew-${OS}_${ARCH}" &&   curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&   tar zxvf "${KREW}.tar.gz" &&   ./"${KREW}" install krew; )
```

krew 명령어를 사용하기 위해서 krew PATH를 지정합니다. 
```
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
kubectl krew
krew is the kubectl plugin manager.
You can invoke krew through kubectl: "kubectl krew [command]..."

Usage:
  kubectl krew [command]

Available Commands:
  completion  generate the autocompletion script for the specified shell
  help        Help about any command
  index       Manage custom plugin indexes
  info        Show information about an available plugin
  install     Install kubectl plugins
  list        List installed kubectl plugins
  search      Discover kubectl plugins
  uninstall   Uninstall plugins
  update      Update the local copy of the plugin index
  upgrade     Upgrade installed plugins to newer versions
  version     Show krew version and diagnostics

Flags:
  -h, --help      help for krew
  -v, --v Level   number for the log level verbosity

Use "kubectl krew [command] --help" for more information about a command.
```
## kubectx, kubens
https://github.com/ahmetb/kubectx
하나의 kubeconfig에 여러 Context를 사용하는 경우 kubectx 명령을 사용해 Context Switching을 쉽게 할 수 있습니다. kubens는 현재 Context에서 default 네임스페이스를 지정할 수 있어 kubectl 명령에서 네임스페이스 옵션을 생략할 수 있게 도와줍니다.
```
kubectl krew install ctx
kubectl krew install ns
```
## kube-ps1
https://github.com/jonmosco/kube-ps1

shell prompt에서 현재 컨텍스트와 기본 네임스페이스를 표시해주는 유틸리티로 kubens, kubectx로 선택한 현재 컨텍스트에 대해 확인할 수 있습니다.
```
source /path/to/kube-ps1.sh
PS1='[\u@\h \W $(kube_ps1)]\$ '

```

## Shell Customize
새로운 툴을 설치하는 것은 아니지만, kubectl을 좀 더 빠르게 실행할 수 있도록 쉘의 alias와 bash completion을 적용하는 것은 매우 유용합니다.

### bash completion
kubectl은 TAB키를 통해 실행할 수 있는 명령어를 자동완성해주는 [bash completion](https://kubernetes.io/ko/docs/tasks/tools/included/_print/#kubectl-%EC%9E%90%EB%8F%99-%EC%99%84%EC%84%B1-%ED%99%9C%EC%84%B1%ED%99%94)기능을 제공합니다.
`kubectl completion bash` 명령어로 출력된 스크립트를 `.bashrc`에 추가하여 적용할 수 있습니다.
```
source <(kubectl completion bash)
```

### Alias
쉘의 Alias기능을 사용하여 kubectl 명령어를 짧게 축약할수 있습니다. 주로 `k` 한글자로 alias를 사용합니다.
alias를 사용해도 bash completion이 적용할 수 있도록 `complete -F __start_kubectl k`도 같이 bashrc에 추가합니다.
```
alias k=kubectl
complete -F __start_kubectl k
```
kubectx와 kubens에 대한 alias를 사용해 더 명령어를 축약합니다.
```
alias kctx=kubectl ctx
alias kns=kubectl ns
```
