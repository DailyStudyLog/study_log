## argocd in actions 3장까지 읽던 내용


GitOps 원칙과 

GitOps 방식에서는 모든 변경 사항을 Pull Request(PR) 기반으로 관리하는 걸 기본 원칙으로 삼는다.
바로 push하지 않고 PR을 거치도록 해서, 동료 리뷰가 가능하고 변경 이력이 명확히 남는다.
운영 중인 리소스도 Git 상태에 따라 동기화되기 때문에, Git이 곧 실제 상태를 정의하는 셈이다.

Argo CD 사용 시 유의할 점
1. 첫 접속 시 인증서 경고 발생 가능
Argo CD는 기본적으로 self-signed 인증서를 쓴다. 
그래서 브라우저로 처음 접속할 때 보안 경고가 뜨는 경우가 있다.

운영 환경에서는 다음처럼 구성하는 걸 권장한다:

LoadBalancer나 Ingress에 인증서를 붙이는 방식

Argo CD 앞단에서 TLS를 처리하게 해서, 인증서 관리를 서비스 외부에서 하도록 구성

이렇게 하면 인증서 문제를 Argo CD 내부로 끌고 가지 않아도 되고, 운영도 편해진다.

Argo CD Application 구성 시 주요 항목
Application 리소스는 크게 아래 세 가지 파트로 나뉜다:

항목	설명
destination	배포 대상 클러스터와 네임스페이스를 지정하는 부분
project	제한 조건을 정의할 때 사용. 클러스터나 리소스 종류 제한 등에 활용
source	Git 저장소 정보, 브랜치, 디렉터리 경로 등을 정의. Helm/Kustomize 설정도 여기에 포함됨

Git 저장소 변경 감지 주기 조정
Argo CD 2.1 이상부터는 Git 상태를 얼마나 자주 체크할지 설정할 수 있다.
아래처럼 argocd-cm ConfigMap에서 timeout.reconciliation 값을 지정하면 된다:

yaml
복사
편집
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  timeout.reconciliation: 300s  # 5분 간격으로 Git 상태 확인
단위는 초(s)

운영 환경에 따라 짧게 설정하면 변경을 빠르게 감지할 수 있음


kubectl rollout restart -n argocd deployment argocd-repo-server

일반적인 gitops flow는 변화를 갖고있는 pr을 생성해 그것들이 동료에의해 리뷰될 수 있게 하는 것을 포함하도록 한다.


Argo CD의 다양한 템플릿 기능으로 인해 매니페스트 생성이 간단하지 않다


`api server`
- 외부와 상호작용하는 컴포넌트, ha manifests는 2개의 파드를 생성

`Repostory server`
- cluster에 적용하기 위한 최종 manifests를 생성하는 책임을 갖고있는 컴포넌트
- Helm, kustomize, jsoonet등 argocd가 지원하는 템플릿 등으로 manifests 생성은 복잡하다.
- ha에선 2개생성

`Application Controller`
- 제어루프가 구현되고 작업이 시작되고, 어플리케이션 동기화가 이뤄지는곳 
- Ha에선 클러스터당 하나씩 두고 공유할 수 있다.

`Redis cache`
- manifest 생성은 비용이 비싸 캐시를 쓰려고 하는데 일반 모드에선 캐시를 계산하다 실패할 수 있어 성능저하가 발생할 수 있다.
- ha에선 3개의 redis 복제본과 하나의 haproxy를 생성하는데 이는 1개의 master 2개의 slave구조를 의미한다.

`Dex server`
- ldap, saml, oidc등 외부 식별 제공자를 사용할때 유저 인증에 대한 책임을 가진다.
- 옵션이지만 github이나 google account를 argocd에 연결하길 원할경우 이 컴포넌트가 필요하다.


API Server

- api server는 모든 요청(cli, Ui등 요청의 유형과 관계없이)에 대한 진입 지점이다. 
- 상태를 가지지 않기 때문에 부하에 따라 확장하거나 줄일 수 있다.
- ha에선 복제본 개수 설정 외에도 ARGOCD_API_SERVER_REPLICAS라는 env를 수정해 사용할 복제본의 개수를 똑같이 설정이 가능하다.
- ARGOCD_API_SERVER_REPLICAS 환경변수는 동시 로그인 요청 제한을 나누기 위해 사용된다.


Repository server
- 일반적으로 간단한 manifests를 사용하지않고 helm등 template engine을 사용한 파일을 사용하는데 이 컴포넌트는 그런 템플릿을 파일로 변환해 kubectl apply 커맨드와 함께 적용될 수 있게 준비되도록 한다.
- 이 컴포넌트가 특정 템플릿엔진(helm,kustomize)가 뭔지 알고있는 깃 Repo의 내용을 가져온다면 이후 어떤 템플릿 엔진이 있는지 알아내고 helm template, kustomize build를 사용해 최종 manifests를 만들어낸다.
- 외부 의존성을 가져오기 전 helm dep update를 해야할 수 도 있다.
- 내부적으로 많은 작업을 하기 때문에 여러개의 인스턴스를 실행한다면 병렬로 작업 수행이 가능하다.
- oom, cpu thorttling 때문에 장애가 발생하지 않도록 충분한 리소스를 주는것이 타당하다
- 만약 너가 수천개의 앱을 argocd를 통해 배포한다면 넌 10개 이상의 repo server를 배포하고 각각 cpu 4~5, 8~10 ram을 줘야할거다.
- 로컬이라 리소스 제한량, 한계량을 설정하지 않지만 운영상황에선 안정성때문에라도 적용하길 강력히 권장한다.

ARGOCD_EXEC_TIMEOUT이란 중요한 파라미터가 있는데 이건 helm, kustomize같은 템플릿 엔진의 타임아웃이다.
	- 만약 kube-prometheus-stack, istio등 큰 차트를 사용하거나 kustomize가 큰 원격 레포를 사용한다면 90초로 설정된 제한 시간이 적어 파일을 생성하지 못할 수 있기 때문에 테스트하고 수정해야 한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: argocd-repo-server
          env:
            - name: "ARGOCD_EXEC_TIMEOUT"
              value: "3m"
```

Application Controller
- 처음엔 모든 동기화를 시작하는 제어루프때문에 복제본이 한개 이상 가질 수 없게 설계되어 있었다.
	- 따라서 2개 이상 있을 때 2개 이상의 동기화가 시작될 가능성이 존재했었다.
	- 1.8버전 이후 각 서버가 argocd에 등록된 클러스터의 일부분을 처리하는 1개 이상 복제본을 둘 수 있게 됐다.
- Ha에서 복제본을 조정하는것은 필요하지 않지만 컨트롤러가 다운됐을때 전 클러스터로 영향을 미치지 않게 하는쪽으론 도움을 줄 수 있다.
	- 이건 다양한 서버로 부하를 분산함으로써 argocd의 성능을 향상시키는데 도움을 준다.

argoc application controller가 얼마나 많은 샤드(혹은 서버)를 가질 수 있는지 알려주려면 
ARGOCD_CONTROLLER_REPLICAS 환경변수를 statefuleset에 설정해야 한다.

--operation-processors, --status-processors, --kubectl-pallelism-limit 플래그를 꼭 체크해야한다. 그리고 더 많은 플리케이션을 다룰 수 있도록 너의 인스턴스에게 높은 값을 설정해야한다.


ps
환경변수의 복제본은 api server, application controller 최소 두곳에서 사용되는데 k8s api를 활용해 개수를 확인하는 것 보다 간단하기 때문에 두 곳 모두 설정해야 하는 오버헤드를 감안해도 설정하는것이 좋다.

`Redis Cache`
- Redis cache는 일회용 캐시로 사용된다.
	- 설치되지 않거나 장애가 있을때 성능에 영향이 있을지라도 시스템은 여전히 작동하지만 고성능을 위해선 선택하는 컴포넌트다.

왜냐면 git repo로부터 생성되는 manifets는 redis cache에 유지될거기 때문이다.
- 만약 Redis가 없다면 매 동기화 요청마다 재생성될거다.
캐시는 git repo에 새로운 커밋이 있을때만 사라지고 캐시가 손실되면 모든 데이터를 다시 생성해야 하며, 애플리케이션은 여전히 동작하긴 하지만 성능 저하가 발생한다.

ha에선 3개의 redis statefulset(1 master, 2 slave)와 redis앞단에 위치하는 haproxy deployment를 배포한다.
- 만약 master가 장애가 생긴다면 slave중 하나가 master로 승격되고 haproxy는 클라이언트에게 영향을 주지않도록 처리해준다.


`dex server`
- Dex는 OIDC같은 외부 시스템이 관련되있다면 인증을 위임하는데 사용된다.
- Dex는 인메모리 구조를 사용하기 때문에 1개 이상의 복제본을 사용할 수 없다. -> 아니라면 불일치를 감수해야 하기 때문
- Dex가 종료되더라도 argocd의 모든 작업엔 문제가 없어서 git 연결, k8s cluster연결등엔 문제가 없고 단순히 일시적으로 외부 시스템으로 로그인하는것만 장애가 생기지만 이것 역시 deployment로 배포되기 때문에 빠르게 해결될 것이다.

만약 로컬 유저를 사용하거나 특별한 관리자 계정을 사용한다면 replica를 0으로 만들어 사용하지 않아도 된다.

ha는 너의 서비스의 중단 위험을 낮출 수 있는 최선의 방법이다.
따라서 argocd에 증원력과 복원력을 제공해 단일 장애 지점을 제거해야하는데 Ha는 이것들을 기본적으로 제공해준다.
그리고 ha가 기본적으로 제공해주더라도 각 컴포넌트의 기능을 이해한다면 더 많은 복제본을 두거나 수정하는 등 ha의 기본설정만으로 충분하지 않다면 더 많은 조치를 취할 수 있다.


장애 복구 계획

argocd는 db를 직접 쓰지않는다.(Redis는 오직 캐쉬로만 사용되서)
이건 상태를 가지지 않는걸 의미하지만 우린 k8s cluster에 어떻게 접근하는지 어떻게 git에 접근하는지 등 argocd의 상태를 구성하는 이러한 것들은 k8s resource로 유지된다. 접속 정보를 위한 secret처럼 

네임스페이스를 지우는 등 휴먼 오류가 발생하거나, 클러스터를 이전하거나 Kubeadm같이 지원되지 않는 기술에서 cloud provider로 이전하는 경우 같은 재해가 잇을 수 있다.

gitops라면 모든것이 git에 있지 않을까 생각할 수 있지만 
1. 보안상의 이유로 새로운 클러스터의 접속정보를 git에 두지 않을 수 있다.
2. 수천개의 앱, 수백개의 클러스터, 수천개의 git repo를 모두 재생성하는건 시간이 머무낳이들어서 이것들을 backup을 해두는것이 더 빠르고 좋다.

argocd는 backup을 yaml파일로 만들거나 기존 백업에서 데이터를 가져오는데 사용되는 main cli의 유용한 부분을 제공한다.
이 cli는 메인 도커 이미지에서 확인될 수 있거나 따로 설치할 수 잇따.

백업 시나리오 

- 

```sh
argocd admin export -n argocd > backup-$(date +"%Y-%m-%d_%H:%M").yml
```

백업하고 싶은 cluster에 접속해 위 cli를 입력하면 현재 날짜와 시간을 비롯한 커스텀한 네임을 기반으로 yml파일이 생성될것이다.

argocd만 있는 클러스터일지라도 꽤 큰 백업파일이 생성되는데(거의 1000라인에 해당한다.) argocd application 자체가 꽤 크고 동기화가 될때마다 모든 히스토리를 유지하기 때문이다. 
다음은 이 파일을 보안에 신경써서 storage(s3등)에 보관한다.

복원 시나리오

복원하기 위해 목적지 cluster에 argocd를 설치해야 하는데 백업엔 초기 설치시 우리가 변경한  모든 설정(configmap, secret)등이 존재하지만 백업엔 실제 Deployment나 statefulset, crd등을 포함하고 있지 않아 미리 설치해야 하기 때문이다.
우리는 application의 인스턴스와 appproject에 대한걸 가질거지만 custom resources의 목적지는 가지지 못할것이다.

그래서 새로운 클러스터에 동일한 설치를 수행하기 위해선 이전의 kustomize와 함께한 Ha설치를 먼저 수행해야 한다. 이후엔 아래 커맨드로 복원을 수행한다.

```sh
argocd admin import - < backup-2021-09-15_18:16.yml
```

이렇게 하면 우리가 설치햇떤 모든 상태(application, cluster, git repo)등 모든건 설치되지만 Redis cache에 있던건 복원되지 않기에 처음 몇분동안 시스템 저하를 일으키며 git repo로부터 모든 manifests를 재생성할 것이다. 이후엔 똑같이 작업이 될 것 이다.

재해가 발생하는건 새벽2시, 오후 2시와 상관없이 희귀하지만 우리는 자주 경험하고 복원 시나리오를 갖춰야 하기 때문에 관측에 대한 좋은 전략이 필요하다.

추후 해당 내용 다시 수정하여 깃에 올리자.
