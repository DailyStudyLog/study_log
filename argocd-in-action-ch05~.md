
5장 aws 구축 with eks

향후 여기서 나오는 코드는 https://github.com/PacktPublishing/ArgoCD-in-Practice ch-05에 존재

# 
대부분의 클라우드 제공자는 관리형 K8s를 구현했고 완전 관리형 control plain을 제공한다.


amazon eks는 aws리전에 다른 Availity Zone에 auto Scaling grupe안의 aws에 의해 관리되는 vpc에 api server, scheduler, kube-controller-manager를 둠으로써 혼란없이 k8s cluster를 유지하고 운영할 수 있도록 돕는다.

aws eks를 구축하기 위해선 하위 5가지를 구축해야 한다.
1. vpc
2. Network Address Translation(NAT) gateway
- private subnet에 있는 eks노드가 우리 vpc외부의 서비스와 통신할 수 있게 돕고 그렇지 않은건 통신되지 않게 만든다.
3.Security group
4. Ec2 Instance
5. Auto Scaling group

#### Provisioning EKS with Terraform

- provider.tf
	- terraform의 제공자는 aws api의 추상화
- network.tf
	- 리소스가 생성될 네트워크 계층
- kubernetes.tf
	- eks 인프라 리소스와 워커 노드 그룹
- variables.tf
	- 테라폼으로 전달할 동적 변수(반복될 수 있음)
- outputs.tf
	- System Console안에 출력될 리소스의 결과물


첫번째로 클라우드에 인프라를 구축하기 전 우리가 사용하길 원하는 클라우드 제공자를 terraform안에 정의해야 한다.

`provider.tf`

```yaml

provider "aws" {
	region = var.region
}

```

terraform의 경우 커뮤니티가 함께 생성, 사용되는 리로스를 추상화한 모듈이 이미 있다.
새로운 네트워크 계층(vpc)를 구축하기 위해 우린 AWS팀에 의해 제공된 모듈을 사용할 것이다.

`network.tf`

```yaml

data "aws_availability_zones" "zones" {}
module "vpc" {
	source = "Terraform-aws-modules/vpc/aws"
	version = "3.11.0"

	name                  = var.network_name
	cidr                  = var.cidr
	azs                   = data.aws_avability_zones.zones.names
	private_subnets       = var.private_subnets
	public_subnets        = var.public_subnets
	enable_nat_gateway    = true
	enable_dns_hostnames  = true

	tags = {
		"kubernetes.io/cluster/${var.cluster_name}" = "shared"
	}
    
    public_subnet_tags = {
    	"kubernetes.io/cluster/${var.cluster_name}" = "shared"
    	"kubernetes.io/role/nlb"
   	}

   	private_subnet_tags = {
   		"kubernetes.io/cluster/${var.cluster_name}" = "shared"
   		"kubernetes.io/role/internal-nlb"           = 1
   	}
}

혹여 메이저 버전이 업그레이드 될 수 있기 때문에 장애가 생기지 않도록 버전을 versions.tf에 명시하는것이 좋다.

terraform {
	required_providers {
		aws= {
			source = "hashicorp/aws"
			version = ">= 3.66.0"
		}

		kubernetes {
			source = "hashicorp/kubernetes"
			version = ">= 2.6.1"
		}
	}
}
```

테라폼을 수행하기 위해 디렉토리를 초기화할 필요가있고 그건 하단의 명령어로 수행이 가능하다.

```sh

$ terraform init

```

초기화 이후 실행 계획을 만들 수 있는데 이건 변화에 대한 미리보기를 할 수 있도록 제공해준다.

```sh

terraform plan -out=plan.out -var=zone_id<your_zone_id>

```

plan.out은 커맨드와 함께 적용될 수 있는데 만약 하단의 커맨드와 함께 성공한다면 terraform을 실행한 폴더에 kubeconfig_packt-cluster(kubeconfig) kubectl 커맨드로 aws를 통해 eks k8s cluster에 접근 가능한 파일이 생성된다.

```sh

terraform apply plan.out

# 이후 성공했다면
export KUBECONFIG=kubeconfig_packt-cluster
kubectl -n kube-system get po # kubeconfig파일에 생성된 파일을 임시 세션 변수로 넣고 명령을 수행하면 생성된 eks cluster에 접근해 명령 수행이 가능하다.

```

지금까진 cluster를 만드는 것이었다면 우리가 만드는 클러스터마다 argocd를 생성할 수 있도록 terraform을 발전시켜야 한다.

##### 준비사항

terraform provider를 kustomize와 argocd를 클러스터에 설치하기 위해 사용할것이다.

첫번째로 kustomization을 위한 extra provider를 provider.tf에 추가할 것이다.

provider "kustomization" {
	kubeconfig_path = "./${module.eks.kubeconfig_filename}"
}

argocd를 구축하이 휘나 argocd.tf파일이다.


data "kustomization_build" "argocd" {
	path = "../k8s-bootstrap/bootstrap"
}
resource "klustomization_resource" "argocd" {
	for_each = data.kustomization_build.argocd.ids
	manifest = data.kustomization_build.argocd.manifests[each.value]
}

#### App of Apps Pattern

실제 환경에선 terraform을 활용하듯 반복적인 k8s cluster와 동일한 서비스를 배포하는것이 필요하다.
  - 이것이 재해복구 시나리오에서 특히 중요하기 때문

# 왜 쓰는가
- 이 패턴은 다른 앱들의 세트를 논리적으로 그룹핑하고 마스터 application을 가질 능력을 준다.
- 다른앱들을 만들 수 있는 master 앱을 만들 수 있으며 이건 선언적으로 app들의 그룹을 관리할 수 있게 해주고 협렵하여 배포될 수 있게 해준다.
- 클러스터를 구축할때 모든 서비스를 함께 배포할때 사용된다.

##### pratice 

aws의 dns를 설정하기 위한 route 53을 설정하기 위해 iam role이 필요한데 그 롤은 iam.tf파일로 하단과 같이 설정한다.

```tf

resource "aws_iam_policy" "external_dns" {
	name                 = "external-dns-policy"
	path                 = "/"
	description          = "Allows access to resources needed to run external-dns."
	policy = << JSON
	{
		"Version": "2012-10-17",
		"Statement": [
			{
				"Effect": "Allow",
				"Action": [
					"route53:ChangeResourceRecordSets"
				],
				"Resource": [
					"${data.aws_route53_zone.zone_selected.arn}"
				]
			},
				"Effect": "Allow",
				"Action": [
					"route53:ListHostedZones",
					"route53:ListResourceRecordSets"
				],
				"Resource": [
					"*"
				]
		]
	}
	JSON
}

```

이렇게 다 배포하고 argocd 를 istio를 기반으로 배포하는 시나리오에서 실제로 istio control plain이 언제 정상적으로 배포될지  argocd가 알 수 없는 사소한 문제가 존재하고 이를 lua언어로 프로그래밍한걸로 체크할 수 있다.

1. argocd-cm.yml(configmap)을 수정
2. argocd 빌트인 과정에 커스텀 script 추가(이건 비추천)

Istio operator는 IstioOperator CRD에 상태 속성을 제공하기 때문에 istio control plain이 성공적으로 배포될때 한번만 업데이트 됨으로 우린 이렇게 argocd-cm을 수정할 수 있다.

```yaml

apiVersion: v1
kind: ConfigMap
metadata:
	name: argocd-cm
	namespace: argocd
	labels:
		app.kubernetes.io/name: argocd-cm
		app.kubernetes.io/part-of: argocd
data:
	resource.customizations: |
		install.istio.io/IstioOperator:
			health.lua: |
				hs = {}
				if obj.status ~= nil then
					if obj.status.status == "HEALTHY" then
						hs.status = "HEALTHY"
						hs.message = "Istio-Operator Ready"
						return hs
					end
				end

				hs.status = "Progressing"
				hs.message = "Waiting for Istio-Operator"
				return hs

```

이렇게 추가되면 주기적으로 argocd에 의해 install.istio.io/IstioOperator 유형의 k8s 리소스 유형의 상태 체크를 하는데 사용된다.

### ApplicationSet

App of Apps의 예제를 봤고 왜 쓰는질 배웠는데 Argo 팀은 이 패턴을 발전시켜 새로운 ApplicationSet이라고 불리는 Argo CD CRD를 만들었다.

#### App of Apps의 단점

이 패턴은 클러스터를 구축할때의 몇가지 단점을 쉽게 없애고 argocd의 수백개의 어플리케이션을 배포하는것을 피하게 도와줬다.
그래서 큰 이점은 수백개의 어플리케이션을 배포하고 관리하고 모니터링하는 하나의 앱만 배포하면 된다는 것이다.

이것 뿐만 아니라 monorepos, multi-repos같은걸 지원하는 문제를 해결해야 한다.
권한 상승없이 멀티테넌트 클러스터에 앱을 배포할 능력을 요즘의 msa 시대에서 어떻게 갖겠냐는 것이다. 물론 수많은 앱을 배포하길 피하면서

#### ApplicationSet이란?

Application Controller는 전형적인 K8s 클러스터이며 argocd app을 관리하기 위해 argocd와 함께 기능한다.
우린 이걸 앱 공장이라고 여길 수 있는 데 가장 큰 특징은 ApplicationSet Controller가 우리에게 다음과 같은 이점을 준다는 것이다.

- argocd를 사용해 여러개의 k8s 클러스터에 배포할 수 있는 하나의 manifest
- multiple source/repo 에서 여러개의 argo app을 배포할 수 있는 하나의 manifest
- argocd에서 하나의 저장소에 여러개의 argocd app이 들어있는 monorepo에 대한 거대한 지원

ApplicationSet은 다른 data source를 지원해 parameters를 만들어내기 위해 Generator를 사용하고 그 종류는 하단과 같다.

- `List`: argo cd 앱들을 대상으로 하는 고정된 클러스터 목록
- `Cluster`: argocd에 의해 관리되거나 미리 정의된 클러스터에 기초하는 동적 리스트
- `Git`: Git Repo의 폴더구조에 기반하거나 git 저장소 내에서 앱을 생성하는 능력
- `Matrix`: 두가지 서로 다른 generator를 이용해 파라미터를 조합
- `Merge`: 두가지 서로 다른 generator를 사용해 파라미터 병합
- `SCM provider`: Source Code Management(SCM) provider(ex: gitlab, github)의 repo를 자동 발견하는 능력
- `Pull Request`: SCM Provider를 이용해 열려있는 Pr을 자동 발견하는 능력
 - `Cluster decision resource`: argocd의 클러스터 리스트를 생성

 #### Generator

 Application의 기본 구성 요소는 generator들로  이건 ApplicationSet의 필드에서 사용되는 파라미터를 생성하는 책임이 있다.

 고정된 클러스터 목록으로 argo 앱을 대상으로할 수 있도록 승인해주는 list generator 예시를 볼 것 이다.

 ```yaml

apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
	name: chaos-engineering
spec:
	generators:
	- list:
			elements:
			- cluster: cloud-dev
			  url: https://1.2.3.4
	        - cluster: cloud-prd
	          url: https://2.4.6.8
	template:
		metadata:
		  name: '{{cluster}}- choas-engineering'
		spec:
		  source:
		    repoUrl: https://github.com/infra-team.git
		    targetRevision: HEAD
		    path: chaos-engineering/{{cluster}}
		    server: '{{url}}'
		    namespace: guestbook

 ```

 하지만 이렇게 고정된 클러스터 목록을 사용하지 않고 싶다면 ClusterGenerator를 사용할 수 있고 그땐 2가지 옵션을 활용할 수 있다.

 1. argocd에서 사용가능한 모든 K8s 클러스터를 대상으로
 2. label seletor에 매칭되는 k8s 클러스터를 대상으로

```yaml

apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
	name: clusterGenerator
spec:
	generators:
	- clusters: {}
	template:
		metadata:
			name: '{{cluster}}-cluster-engineering'
		spec:
			source:
				repoURL: https://github.com/infra-team.git
				targetRevision: HEAD
				path: chaos-engineering/{{cluster}}
				server: '{{url}}'

```

여기엔 {}라는 차이점이 있는데 이것의 의미는 우리가 아무것도 정의하지 않는다면 argocd에서 접근가능한 모든 클러스터에 대해서 앱을 생성하라는 의미이다.

label selector와 함께하면 우린 cluster에 대해 Secret 리소스 안에 label이란 메타데이터를 저장해야 한다.
우리가 해당 클러스터를 대상으로만 배포하고 선택하게 하기 위해서

```yaml

kind: secret
data:
	# etc
metadata:
	labels:
		argocd.argoproj.io/secret-type: cluster
		sre-only: "true"

```

```yaml

kind: ApplicationSet
metadata:
	name: cluster-generator-label-selector
spec:
	generators:
	- clusters:
		selector:
			matchLabels:
				sre-only: true

```
