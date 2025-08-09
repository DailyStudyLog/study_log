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

이렇게 하면 우리가 설치했던 모든 상태(application, cluster, git repo)등 모든건 설치되지만 Redis cache에 있던건 복원되지 않기에 처음 몇분동안 시스템 저하를 일으키며 git repo로부터 모든 manifests를 재생성할 것이다. 이후엔 똑같이 작업이 될 것 이다.

재해가 발생하는건 새벽2시, 오후 2시와 상관없이 희귀하지만 우리는 자주 경험하고 복원 시나리오를 갖춰야 하기 때문에 관측에 대한 좋은 전략이 필요하다.

모니터링

관측가능성은 매우 중요한데 이건 시스템의 상태, 성능, 행동을 답변해주기 때문이다.

시나리오
- 10개의 팀이 수천개의 모놀리식, 마이크로 서비스를 클러스터에 배포하려고 할땐 항상 예상대로 진행되지 않을 수 있다.
	- private repo를 쓰면서 ssh키가 없거나 timeout이 걸릴만큼 큰 레포를 쓰거나 불변 변수가 바뀌려고 한다거나 사용하지않는 오래된 버전이 쓰인다거나 하는

argocd는 시스템을 이해할 수 있는 많은 메트릭을 노출하기 때문에 과하게 사용되거나 사용되지 않는 리소스에 대해 행동할 수 있게 되거나 동기화에 실패, 혹은 특정 어플리케이션이 장애가 났을때 해당 개발팀에게 alert을 보낼 수 있게 도와준다.
	- 그리고 이런 alert이 보내지면 argocd로 배포할 책임을 하나의 팀에게 주고 또 하나는 microservice를 관리하도록 하나의 팀에게 책임을 주는 2가지의 방향성으로 분할된다.

이후 챕터에선 CNCF에 관리되는 Prometheus를 배포하는 방법 중 Prometheus Operator라고 불리는 operator패턴을 활용하여 모니터링을 구축하는게 있는데 이 부분은 구축하고 알고있는 부분이라 다른 챕터에서 쓰겠다.

argocd component는 prometheus format으로 Metric을 노출하기 때문에 통합하기는 더 쉽다.
	- 3개의 service가 스크랩이 되야하는데 이건 API server, repo servers, application controller이다.

동기화를 시도하면 argocd 는 repo servers, 와 controller를 이용한다. 
	- 이것들이 시스템 성능을 더 좋게하고 그것들을 관리하는데 중요한 단서가 된다.

metric을 확인하며 알아낸점은 이 두 컴포넌트가 노출하는 가장 중요한 매트릭은 argocd가 노출한게 아니라 노드 os가 컨테이너가 과도하게 리소스를 점유하려고 하여 OOM을 발생시킴으로 인해 노출되는 것임을 알게되었다.

이건 리소스 설정을 충분히 하지 않았는지 혹은 병렬성에 대해 너무 큰 파라미터를 전달했는지에 대한 단서가 된다.
- repo server는 동시에 너무 많은 manifest를 생성하려고 하는 반면 application controller는 동시에 너무많은 파일을 배포하려고 한다.

sum by (pod, container, namespace) (kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}) * on (pod,container) group_left sum by (pod,container) (changes(kube_pod_container_status_restarts_total[5m]()) > 0

이건 5분동안 다시 시작된게 잇는지 아니면 OOM으로 종료된 게 있는지 알아볼 수 있는 쿼리다.(prometheus)

이런 알림이 발생한다면 다음과 같은 조치를 취해야 한다.
	- resource를 늘리기
	- 부하를 분산하기 위해 더 deployment, statefulset의 복제본 개수를 늘리기
	- --parallelismlimit 파라미터(repo server), --kubectl-parallelism-limit(controller)를 줄이기

그래서 OOM과 관련하여 시스템에서 우리가 정의받은 부하 메트릭이 어떤 상관관계가 있는지를 우린 알아야 한다.

repo server는 template 엔진을 사용해 Git repo로 부터 파일을 가져와 manifests를 생성할 책임이 있고 이건 controller로 작업이 이어지지만 동시에 수많은 요청을 했을땐 argocd_repo_pending_request_total이라는 값으로 알 수 있다.

이건 얼마나 많은 요청이 Repo server로 들어왔는지에 대한것인데 짧은 기간내 요청은 상관없지만 그게 아니라면 문제가 있을 수 있어 신경써야 한다.

controller측면에선 시스템 부하를 보여주는 중요한 값으로 argocd_kubectl_exec_pending이라는 Metric이 존재한다.
이건 목적지 클러스터에 노출되어질수있도록 기다리는 apply, auth커맨드가 얼마나 많은지와 관련되어있다.

이것의 최고 숫자는 --kubectl-parallelism-limit 값과 동일하다 왜냐면 병렬 쓰레드가 동시에 대상 클러스터에 명령어를 날릴 수 있기 때문이다.

이것도 역시 부하가 치더라도 짧은 시간은 문제되지않지만 긴 시간이 문제가 된다.

개발팀을 위한 매트릭
- 개발팀에게 알림을 제공한다면 동기화가 제대로 이뤄지지않는 동기화 상태, 예상대로 동작하지 않는어플리케이션 안정 상태(특히 Degraded)와 같은 것들을 활용할 수 있다.

동기화 상태는 UI나 커맨드에 집중하지 않도록 해도 되게끔 알림에 사용되는 유용한 지표다.

도커이미지의 새로운 버전등이 적용되지않을때 받을 수 있게 하는것으로써 argocd_app_sync_total을 사용할 수 있다.

5분동안 실패한 값을 찾는 뭐 command 존재


Application health status는 동기화 상태와 차이점이 있는게 동기화와 무관하게 일어날 수 있다.
- 3개의 statefulset중 2개는 구동되고 하는 긴 시간동안 초기화하고 있거나 중지되어 있으면 스케줄링 되지않아 Pending으로 머물 수 있고 Degraded라는 상태값에 주목한다.

이런 시나리오등이 운영상황에서 일어날 수 있어 동기화 이벤트와 무관하게 발생할 수 있는데 이를 추적하는 메트릭은 argocd_app_info이다.

만약 degraded라고 오면 명확한 장애 신호이므로 바로 대처해야 하고 우리는 이럴때 알림을 보내는 것도 신경써야 한다.

동기화 방법

메뉴얼

- repo에 변경사항이 생겨도 웹 ui, cli등 Trigger가 되기전까진 진행하지않고 되면 그때서야 배포를 하게된다.
자동
- 대부분 사용하는 모드인데 repo에 변경사항이 푸쉬되면 자동으로 재계산해서 배포를 진행하게된다.

그래서 이건 argo Cd Notification 프로젝트인데 이건 argocd를 염두해 두고 개발되었기 때문에 매우 유용할 것이다.

argocd notification는 여러가지로 구축가능 
- 이건 따로 서술

argocd notification cm 은 추후 다시 알아보기

### 접근제어

owasp는 비영리적으로 웹 보안과관련된 많은 활동을 하는 곳이다. 그들의 가장 잘 알려진 탑텐은 애플리케이션 보안과 관련해 가장 잘 알려진 리스트이다. 여기서 계속 5년 안에 드는것이 Broken Access Control인데 이를 통해 최소 권한 원칙을 위반하지 않도록 사용자를 위한 적절한 권한 설정과 모든 사람에게 제공되는 엑세스 유형을 설정하는 것이 중요하다는것을 알 수 있다

argocd가 처음 설치되면 admin user의 비밀번호는 application server pod의 이름이 되는데 이건 누구나 클러스터의 접근이 가능하면 볼 수 있었기 때문에 2.0.0 버전 이후 argocd-initial-admin-secret이라는 시크릿 리소스안에 저장되게 바뀌었다.

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

를 하게되면 해당 시크릿에서 패스워드를 얻어낼 수 있게 된다. 여기서 %를 지워라 왜냐면 쉘이 새로운 라인에서 시작될 수 있도록 CR/LF를 삽입하기 때문

패스워드를 까먹었다면 재설정할 수 있고 bcrypt 해시가 저장된 주요 secret을 직접 변경 할 수 있다.

다만 최소권한원칙을 지키기 위해선 최초 생성되는 계정을 비활성화하는것이 좋다.

다만 그러려면 엑세스 등 권한을 가진 로컬 계정을 생성해야 하는데 그건 

```yaml

apiVersion: v1
kind: ConfigMap
metadata:
	name: argocd-cm
data:
	accounts.{name}: apiKey, login

```

과 같은식으로 로그인과 apikey발급에 대한 권한을 가진 유저를 생성할 수 있다.

생성됐는지 확인하는 명령어는

argocd account list로 조회가 가능하다.

다만 로그인에 필요한 비밀번호를 세팅해야 하여 어드민 비밀번호를 현재 비밀번호로 사용해 세팅이 가능하다.

argocd account update-password --account alina --current-password pOLpcl9ah90dViCD --new-password k8pL-xzE3WMexWm3cT8tmn 
- 다만 이렇게 하면 shell history에 남기 때문에 남지 않는 update-password를 사용해 과정을 남게하지않는것이 안전할 수 있다.

이렇게 했을때 생기는 변화를 확인하는 command와 파일은

kubectl get secret argocd-secret -n argocd -o yaml

```yaml

apiVersion: v1
data:
  accounts.alina.password JDJh~
  accounts.alina.passwordMtime: Mj~=
  accounts.alina.toknes: bnVsbA==
  admin.password: ~
  admin.passwordMtime: ~
  server.secretKey: ~
kind: Secret
metadata:
  labels:
    app.kubernetes.io/instance: argocd
    app.kubernetes.io/name: argocd-secret
    app.kubernetes.io/part-of: argocd
  name: argocd-secret
  namespace: argocd
type: Opaque

```

여기서 유저는 로그인만 가능하고 클러스나 어플리케이션을 볼 수 있는 권한이 없는데 이걸 허용하기위한 2가지 방법이 존재한다.

1. 유저에게 특정 권한을 부여하기
2. 인증 시 특정 권한을 찾을 수 없는 경우 모든 유저에게 제공되는 기본 권한을 세팅하기 

시나리오
 
기본 정책을 읽기 전용으로 세팅하고 엑세스 토큰을 사용할때 특정 권한을 추가하는 방법에 대해 알아볼것이다.

하단의 기술되는 ConfigMap을 통해 진행할건데 여기에 설정할 모든 RBAC rule을 정의한다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
data:
  policy.default: role:readonly

  policy.csv: |
    p, role:user-update, accounts, update, *, allow
    p, role:user-update, accounts, get, *, allow
    g, alina, role:user-update

```

> 여기서 p,g이런게 궁금하다면 https://casbin.org/en/get-started를 가서 확인해라 


```yaml

apiVersion: v1
kind: ConfigMap
metadata:
	name: argocd-cm
data:
	accounts.{name}: apiKey, login
	admin.enabled: "false" # 이렇게 설정하게되면 더 이상 초기에 설정된 관리자 계정으론 접근할 수 없고 운영환경을 준비할 수 있게 된다.

```


serviceAccount

CI/CD 같은 자동화 파트에서 쓰이게 되며 엄격한 권한을 가져야 하는데 사용자에게 연결된다면 추후 사용자를 제거하거나 권한의 이슈가 생길 수 있어 사용자에게 연결하지않고 쓰는게 좋다

생성
- apiKey권한을 주고 login권한을 삭제한 로컬 계정
- project role을 사용하고 그 역할에 토큰을 생성하는것

권한을 주는 방법은 예시가 있고 그렇게 토큰을 생성하는건

```sh

argocd account generate-token -a gitops-ci # 이런식으로 생성

# 생성된 토큰으로 유효한지 확인하는 명령어

argocd account get-user-info --auth-token <token>

```


```yaml

apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: argocd
spec:
  roles:
    - name: read-sync
      description: read and sync privileges
      policies:
        - p, proj:argocd:read-sync, applications, get, argocd/*, allow
        - p, proc:argocd:read-sync, applications, sync, argocd/*, allow
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  description: Project to configure argocd self-manage
application:
  destination:
    - namespace: argocd
      server: https://kubernetes.default.svc
  sourceRepos:
    - https://github.com/PacktPublishing/ArgoCD-in-Practice.git

```

이 뒤에

```sh

argocd proj role create-token argocd read-sync # 이걸로 토큰 생성이 가능하다.

```

생성하는 모든 토큰은 project role에 저장되고 만료일을 지정하거나 관리해야하는 필요성이 생긴다.

이러한건 동기화밖에 하지 못하지만 그렇기 때문에 손상될경우 장애 포인트를 최소화할 수 있어 좋은거라고 생각하고 이러한 것들을 자동화 파이프라인에서 활용할 수 있다고 여긴다.

### Single sign-on
sso를 사용하면 master login을 통해 독립적인 서비스에도 인증을 할 수 있게 된다.(이 경우엔 독립적인 서비스가 해당 마스터 서비스와 통신할 수 있어야 한다.)

sso를 필수로 여기는 회사가 존재하고 이를 통한다면 모든 서비스에 패스워드가 필요하지 않고 하나의 대쉬보드에서 모든 동료의 온보딩,오프보딩을 관장할 수 있게 된다.

다행히 Argocd는 두가지 방법으로 구성할 수 있게 도와주는데 
1. 기본적으로 설치되는 Dex OIDC 제공자를 사용하는 것이다.
2. Dex 설치를 건너띄고 다른 OIDC공급자를 사용하는 경우 직접 argocd를 사용하는 것이다.

argocd는 sso login을 제공하고 한번 sso를 가능하게한다면 관리자는 비활성화되고 로그인 가능한 로컬계정이 없다면 일반 로그인 폼은 사라지고 오직 SSO버튼만 남을것이다.

```sh

argocd login --sso argocd.mycompany.com # cli에서 oidc제공자를 사용해 로그인

```


SSO with Dex

Dex는 어플리케이션이 인증을 분산할 수 있게 승인해주는 식별 시스템이다. 
Dex는 LDAP,SAML과 같은 다른 인증 스키마를 사용할 수 있고 잘 알려진 식별 제공자(google,github)와 통신할 수 있다.
이 이점은 Dex에 한번 연결하면 모든 연결자가 제공하는 것들의 이점을 누릴 수 있다는 것이다.

우리가 SSO를 사용할때 유저는 자동으로 우리의 RBAC그룹에 추가되어질 수 있다.(argocd측에서 설정하는 어떤 개별 유저의 설정없이)
모든것들은 sso system에 의해 통제된다.

이때 구글에서 세팅한 유저정보를 가져올 순 없지만 우리 Rbac그룹에 커스텀한 정보를 세팅한걸 기반으로 유저정보를 노출할 순 있다.

{
	"schemaName": "ArgoCDSSO",
	"displayName": "ArgoCD_SSO_Role",
	"fields": [
		{
			"fieldType": "STRING",
			"filedName": "ArgoCDRole",
			"displayName": "ArgoCD_Role",
			"multiValued": true,
			"readAccessType": "ADMINS_AND_SELF"
		}
	]
}

이러한 형태를 외부 oidc에 제공하고나면 추가 속성에서 볼 수 있다.

이 뒤엔 Google Saml, RBAC groups 사이 매핑을 소개하기위한 작은 변화를 dex.config 섹션에 추가해야 한다.
	- 이때 마지막 groupsAttr 필드가 중요하고 이게 매핑되는 이름과 일치해야만 한다.

url: https://argocd.mycompany.com # google

dex.config: |
   connectors:
     - type: saml
         id: argocd-mycompany-saml-id
         name: google
         config:
         ssoURL: https://accounts.google.com/o/saml2/idp?idpid=<id-provided-by-google>
         caData: | 
           <base64>
         entityIssuer: argocd-mycompany-saml-id
         redirectURI: https://argocd.mycompany.com/api/dex/callback
         usernameAttr: name
         emailAttir: email
         groupsAttr: role

> SSO가 없을땐 계정에 권한을 추가할때 권한을 추가하여 그 권한을 주고싶다면 연결해야 했는데 SSO는 그러지않아도 자동으로 그룹에 추가된다.

엔지니어의 작업에 따라 개발자, sre로 구분할 수 있고 각 역할에 따라 더 세부적인 역할이 나뉠 수 있다.

argocd가 설치되면 Read-Only권한, Full-Access권한을 가진 2개의 Rbac이 세팅된다.


개발자에겐 readonly를 가진 역할을 세팅하고 동기화같은 추가 역할을 세팅해줄 수 있고 sre는 이 역할을 상속하고 그 외 추가적인 역할을 세팅할 수 있는데 그런 설정은 하단과 같이 세팅된다.


```yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
data: 
  policy.csv: |
    g, role:developer, role:readonly
    p, role:developer, applications, sync, */*, allow

    g, role:sre, role:developer

    p, role:sre, applications, create, */*, allow
    p, role:sre, applications, update, */*, allow
    p, role:sre, applications, overried, */*, allow
    p, role:sre, applications, actions, */*, allow
    p, role:sre, projects, create, *, allow
    p, role:sre, projects, update, *, allow
    p, role:sre, repositories, create, *, allow
    p, role:sre, repositories, update, *, allow

    g, Sre, role:sre
    g, Developer, role:developer
    g, Onboarding, role:readonly
 
```

이건 argocd의 접근제어를 정의하는 첫 시작으로 괜찮지만 시간에 따라선 변경이 필요할것이다.

Argocd와 함께(Dex를 제외하고) sso를 설치하는것들중엔 다음과 같은것들도 있다.
- OKTA
- OneLogin
- OpenUnison
- Keycloak

또한 argocd와 외부 SSO를 통합해 쓸때 ConfigMap의 민감 데이터를 Secret에 넣고 참조하게 하는것이 좋은데
이것은 k8s에선 지원하지 않지만 argocd에선 문법을 이해하고 참조할 수 있다

그 외 ExternalSecret 프로젝트는 더 많은 이점을 제공하는데 다양한 데이터 저장소 예론 Aws Parameter Store, Azure Key Vault, Google Cloud Secrets Manager, Hashicorp Vault등을 지원하여 이게 인프라에 더 적합할 수 있다.

우리가 OIDC를 사용해 argocd를 직접 사용하거나 SSO를 안쓴다면 DEX 서버를 지우는것이 우리에게 실용적일 것이다.

방법으로는
1. replicas = 0
2. 배포를 완전히 없애기 위해 kustomize, helm등을 활용할 수 있다.
