
6장

argo rollout과 친숙해지기

main topics
- motivation
- deployment strategies
- keeping secrets safe
- real ci/cd pipeline
- msa ci/cd in practice

> External Secrets? https://github.com/external-secrets/external-secrets

개요

현대엔 대부분의 회사가 (aws ecs나 azure ci)등의 그들의 컨테이너 오케스트레이션 환경이나 cloud vm에 있는 서비스를 k8s cluster로 옮기려고 이미 노력했는데 가장 큰 문제점 중 하난 더 정교한 배포 방법을 설정하는 것이다.

k8s엔 무료로 rolling update를 지원하는게 존재하긴 하지만 하단의 시나리오는 대응이 힘들다.

- 새로운 버전을 업데이트 하고싶지만 기능을 테스트 하고 변경하고 업데이트하면서 병렬로 이전 버전을 없애려면 어떻게 해야할까
이런건 buie/green 배포전략이라고 불리고 이건 우리에게 순단 시간을 더 줄이는 것을 줌과 동시 나나 유저에게 큰 영향을 주지 않는다.

#### 간단하게 k8s에서 bule/green 배포 구현

v1에 대한 deployment를 배포한 뒤 v2를 배포하고 이후 V2를 테스트하여 괜찮다면 service의 label selector를 바꾸는걸로 blue/green이 되겠지만 이것은 자동화 해야한다.

Blue/Green 전환을 자동화하면 → 장기적으로 복잡하고 유지보수 어려움이 생길 수 있다.
하지만 지금처럼 수동으로 label selector를 바꾸는 방식은 → GitOps 원칙을 따르지 않는 방식이다.

그래서 Argo Rollouts이라는 프로젝트가 있는데 이 k8s controller는 여기서 설명한 것보다 더 복잡한 방식으로 서비스를 점진적으로 제공하는데 도울 것 이다.

#### Argo Rollout이란?

Argo Rollout은 k8s Deployment object와 유사한 K8s controller지만 argo project 팀에 의해 개발된 CRD이다.

이 CRD는 기능을 확장해 디플로이먼트를 점진적으로 제공할것이며 그 배포전략은 하위와 같다.

- Blue-green deployment
- Canary deployment
- Weighted traffic shift
- Automatic rollbacks and promotions
- Metric analysis

#### 왜 Argo Rollout인가?

문제 인식
- 기본적인 k8s deployment 객체는 오직 RollingUpdate 전략을 지원하며 업데이트시 readiness probe같은 deployment의 기본 안전 준비 사항만 지원하는데 이것의 많은 한계는 아래와 같다.

- rollout 속도를 제어할 방법이 존재하지 않는다.
- canary, blue-green과 같은것처럼 새로운 버전으로 변경할때 트래픽을 전환할 수 없다.
- rollout 조건으로 metric을 사용할 수 없다.
- 진행을 중단하는 방법만 존재하고 업데이트를 중단하거나 롤백하는 방법은 존재하지 않는다.

대규모 환경에서 Rolling update는 deployment의 폭발 반경을 제어하지 못하는것처럼 위험하다.

#### Argo Rollout Architecture

Argo Rollout Controller는 일반 k8s deployment 객체가 그러하듯 ReplicaSets의 생명주기를 관리하는 책임이 있다.

`Argo Rollouts controller` 
	- k8s의 모든 컨트롤러처럼 Rollout이라 불리는 유형의 객체 리소스를 관측하고 Rollout의 선언을 확인하고 클러스터로 선언된 상태로 가져오도록 노력한다.
`Rolluts resource(CRD)` 
	- 전형적인 K8s deployment 객체와 호환되지만  blue-green이나 canary같은 배포 전략의 단계 및 임계값을 제어할 수 있는 추가 필드를 포함하고 있다.
	- Argo Rollouts의 이점을 활용하려면, Rollout이라는 CRD로 수정하여 Migration을 해야 그들이 관리되고 이점을 누릴 수 있다.
`ReplicaSets`
	- Argo Rollout 컨트롤러가 deployment/application의 서로 다른 버전을 추적하기 위해 추가 메타이데이터를 포함한 k8s ReplicaSet 리소스이다.
`Analysis`
	- 우리가 선호하는 메트릭 제공자와 연동해 Rollout 컨트롤러와 통신하는 `지능형(의사결정) 모듈`이다.
	- 업데이트가 성공적이거나 실패하거나 결정할 수 있도록 metric을 정의할 수 있다.
		- 매트릭이 검증되고 유효하면 배포가 진행되는 반면 실패하면 롤백되고 metric provider의 응답이 없으면 배포가 중단된다.
	- `AnalysisTemplate`, `AnalysisRun`이란 2가지 템플릿이 필요하다.
		- AnalysisTemplate은 쿼리할 매트릭에 대한 정보가 들어있어 AnalaysisRun이라 불리는 결과를 가지고 올 수 있다.
		- 이 템플릿은 특별한 rollout에서 또는 클러스터에서 전역적으로 설정되어 여러개의 rollouts에서 공유되어질 수 있다.
`Metrics providers`
	- Analysis 컴포넌트에서 사용할 수 있는 prometheus, datadog 같은 툴과 통합되고 자동으로 롤아웃하거나 롤백하는 등 역할을 수행한다.


#### Rollout - Blue/Green 배포 전략

이전 예시같이 blue/green deployment는 동시에 2개의 버전을 병렬로 가진다.
블루-그린에서는 운영 트래픽을 먼저 구버전에 유지해 두고, 새 버전 테스트(수동/자동)를 수행한 뒤에 트래픽을 새 버전으로 넘긴다.

이걸 Argo Rollout Crd와 함께 구성하기 위한 예제는 다음과 같다.

```yaml

apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
	name: rollout-bluegreen
spec:
	replicas: 2
	revisionHistoryLimit: 2
	selector:
		matchLabels:
			app: bluegreen
	template:
		metadata:
			labels:
				app: bluegreen
		spec:
			containers:
			- name: bluegreen-demo
			  image: spirosoik/cho06:v1.0
			  imagePullPolicy: Always
			  ports:
			  - containerPort: 8080
	strategy:
		blueGreen:
			activeService: bluegreen-v1
			previewService: bluegreen-v2
			autoPromotionEnabled: false

```

k8s Deployment와 다른 점은 우리가 배포 전략을 정의하는 strategy와 blue-green version에 사용되는 2개의 서비스다.

blueGreen에서 중요한 필드는 하위다.
- `activeService`
	- 새 버전을 배포가기까지 제공될 현재 blue version
- `previewService`
	- 새 버전을 배포하면 제공될 새로운 green version
- `autoPromotionEnabled`
	- ReplicaSet이 완전히 준비되고 가용하면 즉시 green version을 배포할지 결정하는 필드

#### Algo Rollout - Canary 배포 전략

카나리아 배포의 핵심은 일부 사용자에게만 **새 버전(또는 새 배포)**을 제공하고, 나머지 트래픽은 기존 버전으로 계속 서빙하는 데 있다.
카나리아 배포는 현실에서 새로운 버전이 정확하게 동작하는지 검사할 수 있다는데 있고 그리고 최종 사용자에게서의 채택(비율)을 단계적으로 늘려, 결국 기존 버전을 완전히 대체할 수 있다.

이걸 달성하기 위한 예제는 하위와 같다.

```yaml

apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
	name: rollout-canary
spec:
	replicas: 5
	revisionHistoryLimit: 2
	selector:
		matchLabels:
			app: rollout-canary
	template:
		metadata:
			labels:
				app: rollout-canary
		spec:
			containers:
			- name: rollout-demo
			  image: spirosoik/cho06:blue
			  imagePullPolicy: Always
			  ports:
			  - containerPort: 8080
	strategy:
		canary:
			steps:
			- setWeight: 20
			- pause: {}
			- setWeight: 40
			- pause: {duration: 40s}
			- setWeight: 60
			- pause: {duration: 20s}
			- setWeight: 80
			- pause: {duration: 20s}
```

canary의 주요한 다른점은 
현재 롤아웃은 카나리 트래픽을 20%로 시작하고, pause가 {}로 설정되어 있어 무기한 일시정지된다(자동으로 다음 단계로 진행되지 않는다)

#### Ci-CD with 점진적 배포

중앙 통제에만 의존하지 않고, 팀 차원에서 필요에 따라 Argo 애플리케이션을 유연하게 관리·배포할 수 있길 원한다

우린 그래서 Argo Project, Project roles, tokens를 ci에서 사용할 수 있고 이 접근법을 통하면 argocd의 멀티테넌시를 구현할 수 있다.

Argo Project(예시)
```yaml

apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
	name: team
spec:
	destinations:
	- namespace: team-*
	  server: '*'
	sourceRepos:
	- https://github.com/PacktPublishing/ArgoCD-in-Practice.git
	roles:
	- name: team-admin
	  policies:
	  - p, proj:team:team-admin, applications, *, team/*, allow
	- name: ci-role
	  description: Create and Sync app
	  policies
	  - p, proj:team:ci-role, applications, sync, team/*, allow
	  - p, proj:team:ci-role, applications, get, team/*, allow
	  - p, proj:team:ci-role, applications, create, team/*, allow
	  - p, proj:team:ci-role, applications, update, team/*, allow
	  - p, proj:team:ci-role, applications, delete, team/*, allow

```

이렇게 프로젝트를 생성하고 나면 ci에서 사용할 Project token을 생성해야 한다. 더 구체적으로 (ci-role이란 이름으로)
	- create,update,delete,sync 라는 권한만 제한적으로 갖고있으며 team-* 정규식으로 매칭되는 네임스페이스에만 가능하다.

```sh

argocd proj role create-token team ci-role -e id # token 생성 이건 어디에도 저장되지 않는다.

```

`ci`


대강의 workflow는 list, build이후 argo cli를 설치하여 각 팀에 해당하는 argocd app을 배포하는 것

이 외 ci에 관련된 예제 - https://github.com/PacktPublishing/ArgoCD-inPractice ch06/automated-blue-green/.github/workflows/ci.yaml에 있다.
 
`cd`

새로운 태그의 아티펙트를 생성하고 bule-green 배포전략으로 배포하는 것인데 smoke test를 거치고 모든것이 정상이면 Green 버전으로 배포

cd workflow는 대강
- deploy
- smoke test

deploy에선 새로운 버전을 배포하여 2개의 버전이 둘다 동시에 뜨게 하고 이후 green version에 대해 생각하는 응답을 받는(smoke test)를 받으면 옮기는 시나리오인데 보통의 운영은 통합테스트가 존재하여 그런 테스트를 진행한 후 옮기는것이 좋다.

이러한 smoke test를 진행하려면 외부 접속이 안되는 서비스일경우 port-forward를 통해 진행하는등 테스트를 수행하는 법에 대해 고민해야 하지만 sync-wave와 함꼐 resource hook을 사용하면 외부 ci/cd에서 테스트 시 kubeconfig가 필요한 경우 등등을 모두 없애고 smoke test를 하게 만들 수 있다.


이 경우 별도의 app을 만들어 sync단계에선 통합 테스트를 진행하고 문제가 없을 시 post-sync에서 새로운 버전이 배포되도록 해당 단계를 이용한다. 이를 이용해 실패하더라도 배포의 실패를 최소화할 수 있고 이땐 post-sync 단계는 절대 진행되지 않을 것이다. 
그리고 해당 훅을 이용하면 container를 유지해 이슈에 대한 디버깅을 하기위해 사용할 수 있다.

통합 테스트를 하기 위한 manifests이다.

```yaml

apiVersion: batch/v1
kind: Job
metadata:
	generateName: integration-tests
	namespace: team-demo
	annotations:
		argocd.argoproj.io/hook: Sync
		argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
	template:
		spec:
			containers:
			- name: run-test
			  image: curlimages/curl
			  command: ["/bin/sh", "-c"]
			  args:
			   - if [ $(curl -s -o /dev/null -w '%{http_code}' rollout-bluegreen-preview/version) != "200"]; then exit 22; fi
			   - if [[ "$(curl -s rollout-bluegreen-preview/version"!= "APP_VERSION"]]; then exit 22; fi;
			     echo "tests completed successfully"
			restartPolicy: Never
	backoffLimit: 2

```

argocd.argoproj.io/hook: Sync
argocd.argoproj.io/hook-delete-policy: HookSucceeded

이 두가지 어노테이션이 job에서 가장 중요한데 두가지는 sync phase에서 실행되는것과 만약 성공한다면 job이 자동적으로 HookSucceeded resource hook에 의해 사라질것임을 나타낸다.

마지막부분은 앱의 PostSync 단계에서 배포하는것이다.

```yaml

apiVersion: batch/v1
kind: Job
metadata:
	generateName: rollout-promote
	namespace: team-demo
	annotations:
		argocd.argoproj.io/hook: PostSync
		argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
	template:
		spec:
			containers:
			- name: promote-green
			  image: quay.io/argoproj/kubectl-argo-rollouts:v1.1.1
			  command: ["/bin/sh", "-c"]
			  args:
			   - kubectl-argo-rollouts promote app -n team-demo;
			restartPolicy: Never
	backoffLimit: 2

```

argocd.argoproj.io/hook: PostSync
argocd.argoproj.io/hook-delete-policy: HookSucceeded
역시나 여기서 제일 중요한 어노테이션은 이 2가지이다.

### secret을 안전하게 유지하기

- gitops와 선언적 구성을 얘기할때 첫번째 문제가 어디에 secret을 안전하게 저장할것인지에 관해서이다.

그것들을 안전하게 유지하기 위한 대부분의 방법은 Vault, Aws Secrets Manager, Azure Key Vault 등의 management tool을 이용해 관리하는 것이다.
하지만 이것들을 이용하면서 어떻게 gitops원칙을 유지할 수 있을지 고민이 될 수 있는데 이때 `External Secrets Operator`라고 불리는 툴이 그 대안으로 가능하다.

이것은 자동화를 염두하여 설계되어있으며 External Secrets Operator는 더 특별하게 secrets을 Aws Secret Manager, Vault와 같은 external API로부터 k8s Cluster Resource로 동기화할 수 있다.

전체적인 아이디어는 secret을 정의하고 완벽하게 동기화하기 위한 방법을 정의하는 몇몇 k8s custom resource가 있다는 것이다.

resource model은 아래와 같다.
- `secretStore`:  
	- external API부분에서 인증을 담당하여 우리가 실제 secret을 가져올 수 있게 해준다.
	- 이건 동일한 네임스페이스에서 새로 생긴 secret리소스가 있는지 확인하지만 최종적으로 동일 네임스페이스에서만 참조될 수 있다.
- `ExternalSecret`:
	- external API로 부터 가져오기 위한 데이터를 정의하는 방법이고 `SecretStore`와 상호작용한다.
- `ClusterSecretStore`
	- global secret이고 클러스터에서 어느 네임스페이스에서든 참조될 수 있다.

secretStore의 예제

```yaml

apiVersion: external-secrets.io/v1alpha1
kind: SecretStore
metadata:
	name: secretstore-sre
spec:
	controller: dev
	provider:
	  aws: 
	    service: Secretsmanager
	    role: arn:aws:iam::123456788912:role/sre-team
	    region: us-east-1
	    auth:
	      secretRef:
	      	accessKeyIDSecretRef:
	      	  name:  awssm-secret
	      	  key: access-key
	      	secretAccessKeySecretRef:
	      	  name: awssm-secret
	      	  key: secret-access-key

```

ExternalSecret 예제

```yaml

apiversion: external-secrets.io/v1alpha1
kind: ExternalSecret
metadata:
	name: db-password
spec:
	refreshInterval: 1h
	secretStoreRef:
	  name: secretstore-sre
	  kind: SecretStore
	target:
	  name: secret-sre
	  creationPolicy: Owner
	data:
	- secretKey: dbpassword
	  remoteRef:
	    key: devops-rds-credentials
property: db.password

```

external secrets의 아이디어는 argocd가 상태를 항상 동기화하고 유지하고 계싼하는것과 유사하지만 External Secrets controller는 동일한 상태를 유지하기 위해 특정 secret manager에서 변경될때마다 secrets을 매번 업데이트한다.


#### External Operator

우리는 git에 평문의 비밀번호등을 저장할 수 없는데 External secrets Operator는 이런 문제를 해결하고 여러 이점을 제공한다.

- 보안 위험을 최소화
- 완전 자동화
- 동일한 툴을 이용해 모든 환경에 전달

6장 끝

배울 아이디어

argocd와 argo rollout을 결합해 실패를 최소화하고 배포하는것을 배웠고 gitops방식을 활용해 blue-green배포를 구현하고 AppProject를 활용해 논리적 그룹 분리를 수행하고 project token을 활용해 제한적 권한을 부여했다.


통합 테스트를 실행하고 테스트가 실패하면 파이프라인을 실패시키고 사전 동기화 단계에서 실패 원인을 디버깅하기 위해 리소스를 유지하기 위해 syc-wave, reousrce hook을 활용했다. 중복된 성공 테스트로부터 클러스터를 깨끗하게 유지될 수 있게 하고 사후 동기화 단계에서 통합 테스트 성공 시 최신 버전의 애플리케이션을 배포할 수 있게 된다.


7장 trouble shutting

각 팀당 특정 네임스페이스를 관리하는 경우거나 클러스터를 관리하려고 인스턴스를 분리하여 설치하는 경우처럼 동일한 클러스터에 여러 argocd를 배포하려는 경우 일반적인 네임스페이스를 활용하는걸로는 문제가 생길 수 있다.

이 외에도 주의해야할 사항이 있는데

argocd를 배포하기 위해 CRD를 배포하게 되는데 여러개의 argocd 구축 중 하나만 배포하도록 만들어야 한다.

일반적으로 서비스를 안전하게 유지하기위해 배포하는 `prometheus operator`, helm 저장소인 `chartMuseam`, 무료 오픈 소스 ci/cd인 `Tekton`과 같이 플랫폼을 구성하기 위한것들을 platform cluster라고 두면 3번째 argocd를 배포하여 이 argocd가 플랫폼 서비스와 prod, 등 환경별로 클러스터를 구축하게 하는 유즈케이스가 대안일 수 있다.

이렇게 배포할때 gitops를 준수하기 위해선 각기 다른 브렌치로 배포하거나 폴더를 다르게 유지하는 방법이 존재하는데 
브렌치를 다르게 한다면 생성하는 Pr이 기본 브렌치로 가는지등을 항상 주시해야 하고 실수가 발생할 수 있다.
그리하여 방법 중에선 폴더로 유지하는것이 더 쉽고 명시적일 수 있는 것이다.

helm에서 배포할땐 기본적으론 클러스터 단위로 관측하고 배포할 수 있는 권한이 있지만 kustomize등으로 배포한다면 argocd로만 권한이 한정되어있어 이것을 ClusterRoleBinding으로 업데이트하고 권한을 더 부여해야 한다.

helm에선 네임스페이스 단위로 할거라면 관련된 필드를 수정하여 네임스페이스로 범위를 줄이고 네임스페이스를 지정할 수 있따.

팁

일반적으로 동기화에 문제가 생긴다면 하위의 command로 applcation controller를 재시작하라

```sh

kubectl rollout restart sts argocd-applications-controller -n arogcd

```

만약 Helm이라면 기본적으로 Deployment일거라 그땐 위의 명령어를 참고해 재시작하라

repo server가 manifests를 생성하는데 이슈가 있으면 하단의 명령어로 재시작하라

```sh

kubectl rollout restart deploy argocd-repo-server -n argocd

```

웹 UI가 멈추거나 cli가 멈춘다면 argocd-server를 재시작

sso로그인이 멈춘다면 dex서버를 재시작

다만 Redis 서버는 재시작하면 캐시를 모두 비워 성능에 패널티가 발생하기 때문에 권장하지 않는다.

argocd는 자체 컨테이너 이미지 내의 주요 템플릿 툴을 포함한다. 그래서 각 개별 argocd 버전은 특정한 helm, kustomize의 버전을 포함하고있다.
- 2.0의 argocd는 helm 3.5.1, kustomize 3.9.4를 가진다
그래서 argocd의 버전이 새로 나와도 migraition등의 위험이 존재하여 템플릿 엔진만 특정한 것으로 사용하기 위한 방법은 argocd container image에 특정한 버전의 어떤 툴이라도 갖고있는 이미지를 생성해 사용하는 것이다.
	- 하지만 이는 이미지를 생성하고 푸쉬하고 다시 적용하는 등 어려움이 많다.

helm이나 다른 템플릿 엔진의 특정 버전을 사용하기 위한 더 쉬운 방법은 tool binary를 다운로드하고 container image내의 것을 대체할 수 있도록 repository server deployment안에 init container를 사용하는 것이다.
- Init container는 메인 컨테이너와 분리되어 있고 복잡해보이지만 그렇지 않아 항상 동일한 방식으로 툴의 버전을 변경할 수 있다.

이걸 활용하여 custom-tools라고 불리는 볼륨을 생성하고 Emptydir타입으로 생성할건데 용량은 적고 파드가 완전 삭제되지않는한 지워지지않는 볼륨으로 활용할 것이다.

한번 생성한다면 기존의 helm이 위치한 것을 비우고 다운로드한 binary를 임시볼륨을 통해 공유하여 덮어씌우는식으로 기능한다.

```yaml

repoServer:
	volumes:
		-name: custom-tools
		 emptydir: {}
    volumeMounts:
      - mountPath: /usr/local/bin/helm #기존의 Helm binary 대체
      	name: custom-tools
      	subPath: helm
  	initContiners:
  	  - name: download-tools
  	    imagE: alpine:3.15
  	    command: [sh, -c]
  	    args:
  	      - wget -qO- https://get.helm.sh/helm-v3.5.1-linux-amd64.tar.gz | tar -xvzf - && mv linux-amd64/helm /custom-tools/
  	    volumeMounts:
  	      - mountPath: /custom-tools
  	        name: custom-tools

```

성능 개선

`Applciation Controller`
- 동기화 성능을 개선하기 위해선 동기화 작업에 영향을 줄 수 있도록 시작할때 컨테이너로 전달할 수 있는 몇몇 파라미터가 있다.

- --status-processors
	- 얼마나 많은 앱이 동시에 계산될 수 있는지 특정하는 파라미터다.
		- 만약 너의 인스턴스가 많은 앱을 다루고 있다면 그 값이 커야하는데 공식 문서에서 제안하기를 1000개의 앱엔 50으로 설정하길 권장하고 기본값은 20으로 되어있다.
- --operation-processors
	- 기본값은 10인데 동시에 수행할 수 있는 동기화 작업을 조정할 수 있게 해주며 1000개의 앱일땐 25를 두도록 권장한다.
- --repo-server-timeout-seconds
	- application controller는 manifests를 생성하도록 repo server로 작업을 전달하는데 이 때 얼마나 오랜시간 생성할 수 있을지를 설정하는 파라미터이다.
	- 인터넷, 파일생성수에 따라 `Context deadline exceeded`라는 실패를 볼 수 있어 늘리는 것이 좋고 기본값은 60초다.
- --kubectl-parallelism-limit
	- 파일이 완벽히 생성된 이후 클러스터에 적용되야 하는데 이 파라미터는 클러스터에서 얼마나 많이 동시에 kubectl 커맨드를 사용할 수 있을지 특정하는 파라미터이며 기본값은 20초다.
	- 더 많은 앱이 동시에 적용되길 바란다면 늘리는것이 좋지만 OOM이 발생할 수 있어 metric에 주의를 기울여야 한다.

`Repository Server`
- --parallelismlimit 
	- 얼마나 많은 manifests가 동시에 생성될지 이며 기본값은 0인데 이건 제한이 없다는것을 의미한다.
		- 이것으로 인해 OOM이 발생할 수 있고 컨테이너가 재시작될 수 있어 OOM을 추적하는데 이 필드가 중요한 이유이다.
	- flag대신 사용할 수 있는 환경변수인 ARGOCD_REPO_SERVER_PARALLELISM_LIMIT이 있다.
- --repo-cache-expiration
	- 이건 manifests가 캐시되는 시간을 의미하고 기본값은 24시간이다.
	- 변경할일이 잘 없지만 배포가 빈번하다면 값을 올리고 적게 값을 설정하는건 캐쉬의 이점을 줄일 뿐이다.
- ARGOCD_GIT_ATTEMPTS_COUNT 환경 변수
	- git 저장소의 기본 tag, branch reference, 기본 브렌치를 commit Sha로 일치시켜 변환시킬때 얼마나 많이 시도할 수 있는지를 의미하며 기본값은 1이라 재시작하지않지만 안전하게 하기 위해 더 높일 수 있다.
- ARGOCD_EXEC_TIMEOUT 환경 변수
	- helm이나 kustomize같은 템플릿엔진으로 manifest가 생성될때 아마도 다운로드가 필요할 수 있다.
		- 큰 helm chart 등 의 소스로부터 가져오는 시간으 증가시키기위해 필요할 수 있다.
	- 우린 이값을 제어해 repository server로 manifest 생성시간의 타임아웃을 조정할 수 있고 기본값은 90초이다.
	- 만약 파일을 생성하는데 빈번하게 실패한다면 값을 증가시켜라
	- 이 값은 application controller의 --repo-server-timeout-seconds 값과 연결되므로 그들의 값은 유사해야 한다.

#### Yaml and Kubernetes Manifests 

helm이나 kusotmize같은 메인 템플릿 엔진의 옵션에 대해 알아볼 건데 지식이 부족하다면 해당 공식문서를 참고해라
helm
- https://helm.sh/docs/intro/queckstart
kustomize
- https://kubectl.docs.kubernetes.io/guides

#### helm

helm은 아마 k8s manifests를 위해 가장 많이 사용되는 템플릿 옵션이다. 매우 유명하고 가장 널리 채택되어있고 helm을 쉽게 사용하는건 argocd에 내장된 선언적 방법을 이용하는 것이다.

하지만 argocd의 기본 내장된 application spec에 helm 차트를 오버라이드 하기위해 파일을 계속 늘리면 너무 파일이 커지고 결합도가 커지기 때문에 더 나은 방법은 앱 정의와 helm 차트 정의를 나누는 것이다.

이 방법으론 helm차트를 하위 차트로 활용하거나 umbrella chart라고 불리는 방법을 활용하는 것인데 이건 필요한 차트에 대한 종속성을 가지는 차트를 정의해 서브 차트가 되거나 기본 차트가 umbrella chart가 되는 것이다.

```yaml

name: traefik-umbrella
apiVersion: v2
version: 0.1.0
dependencies:
- name: traefik
  version: 9.14.3
  repository: "https://helm.traefik.io/traefik"
  alias: traefik

```

```yaml

traefik:
  image:
    tag: "2.4.2"
  deployment:
    replicas: 3
  podDisruptionBudget: 
    enabled: true
  logs:
    general:
      level: "INFO"
    access:
      enabled: true

```

이 두가지 파일을 가리키는 application을 만들면 helm차트인지 argocd가 인지핮 이후 helm dependencies update 커맨드로 의존성을 다운로드하고 모든 파일을 만들어 적용하려 할 것 이다.
이런 서브차트 방식을 차용하면 버전같은걸 바꾸거나 리소스를 추가하는등 영향을 끼칠 수 있고 새로운 버전의 변경에 대한 지표가 될 수 있다.

Helm yaml의 오류를 포착하기 위해 파이프라인에 스크립트를 추가하여 Pr에서 생긴 브렌치의 템플릿 결과와 main 브렌치의 결과를 diff로 비교하여 우리에겐 차이점이 아니라 오류가 아니지만 suffix로 비교하여 미리 결과물을 확인할 수 있다.

```sh

helm depndency update traefik-umbrella
helm template traefik-umbrella -include-crds --values traefik-umbrella/values.yml --namespace traefik --output-dir out 
git checkout main
helm dependency update traefik-umbrella
helm template traefik-umbrella -include-crds --values traefik-umbrella/values.yml --namespace traefik --output-dir out-
default-branch
diff -r out-default-branch out || true

```

이런식으로의 파이프라인을 빠르게 도입한다면 yaml 유효성을 검증하여 오류를 잡아내고 유효성을 검증하게되는 이점을 얻을 수 있을 것이다.
