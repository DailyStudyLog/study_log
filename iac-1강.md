Infrastructure as code, 코드로써의 인프라

Infrastructure as code, 즉 코드로써의 인프라는 인프라를 이루는 서버, 미들웨어 그리고 서비스 등,
인프라 구성요소들을 코드를 통해 구축하는 것.
IaC는 코드로써의 장점, 즉 작성용이성, 재사용성 유지보수 등 장점을 가진다.

테라폼은 인프라를 만들고, 변경하고, 기록하는 iac를 위해 만들어진 도구, 
문법이 쉬워 비교적 다루기 쉽고 사용자가 많아 참고할 수 있는 예제가 많다.
.tf 형식의 파일구조를 가진다.

구성요소

provider
- 테라폼으로 생성할 인프라의 종류
resource
- 테라폼으로 실제로 생성할 인프라 자원을 의미
state
- 테라폼을 통해 생성한 자원의 상태를 의미
output
- 테라폼으로 만든 자원을 변수 형태로 state에 저장하는 것을 의미
module
- 공통적으로 활용할 수 있는 코드를 문자 그대로 모듈 형태로 정의하는 것을 의미
remote
- 다른 경로의 state를 참조하는 것을 의미 output 변수를 불러올 때 주로 사용 

### Provider

```provider.tf

# AWS provider
provider "aws" {
	region = "ap-northeast-2"
	version = "~> 3.0"
}

```

Provider 안에서 다양한 Arguments를 가집니다.
AWS resource 를 다루기 위한 파일들을 다운로드 하는 역할을 합니다.

main.tf, vpc.tf 등 원하는 형태로 파일이름을 사용합니다.
```tf
#Create a VPC
resource "aws_vpc" "example" {
	cidr_block = "10.0.0.0/16"
	# cidr_block 이외에도 수많은 인자가 존재합니다.
}

```
테라폼으로 VPC를 생성하는 코드이며 VPC역시 다양한 Arguments 와 다른 구성요소가 존재합니다.

### state

# terraform.tfstate 라는 파일명을 가집니다.
{
	"version": 4,
	"terraform_version": "0.12.24",
	"serial": 3,
	"lineage": "3c77XXXX-2de4-7736-1447-038974a3c187",
	"outputs": {},
	"resources": [
		{...},
	]
}

테라폼 state이며 현재 인프라의 상태를 의미하는 것은 아닙니다.
state는 원격 저장소인 backend 에도 저장될 수 있습니다.
	-> 대부분은 현업에서 backend에 저장되고 있습니다.
state 파일과 현재 인프라의 상태를 똑같이 유지하는 게 중요한 포인트 입니다.

### output

resource "aws_vpc" "default" {
	cidr_block = "10.0.0.0/16"
	# cidr_block 이외에도 수많은 인자가 존재합니다.
}

output "vpc_id" {
	value = aws_vpc.default.id
}

output "cidr_block"{
	value = aws_vpc.default.cidr_block
}

테라폼 output이며 VPC 역시 다양한 argumentes와 다른 구성요소가 존재하지만 추후 구체적으로 파악합니다.

### module

module "vpc" {
	source = "../_modules/vpc"

	cidr_block = "10.0.0.0/16"
}

테라폼 module 이며 한번 만들어진 테라폼 코드로 같은 형태를 반복적으로 만들어낼 때 주로 사용됩니다.

실제 모듈 경로, 코드는 source에 경로로서 포함하게 되고 인자값은 cidr_block의 형태로 제공됩니다.

### remote

#### remote는 원격 참조 개념으로 이해하면 좋습니다.

data " terraform_remote_state" "vpc" {
	backend = "remote"

	config = {
		bucket = "terraform-s3-bucket"
		region = "ap-northeast-2"
		key    = "terraform/vpc/terraform.tfstate"
	}
}

테라폼 module이고 remote state 는 key 값에 명시한 state에서 변수를 가져옵니다.
특정 region의 bucket에서 키값을 명시하게 되면 저장된 state값을 참조할 수 있게 되는 구조입니다.

### 테라폼 명령어

init 테라폼 명령어 사용을 위한 각종 설정을 진행합니다.
plan 테라폼으로 작성한 코드가 실제로 어떻게 만들어질지에 대한 예측 결과를 보여줍니다.
apply 테라폼 코드로 실제 인프라르 생성하는 명령어입니다.
import 이미 만들어진 자원을 테라폼 state 파일로 옮겨주는 명령어입니다.
state 테라폼 state를 다루는 명령어입니다. 하위 명령어로 mv, push 와 같은 명령어가 있습니다.
destroy 생성된 자원들을 state 파일 기준으로 모두 삭제하는 명령어입니다.

init은 최초의 테라폼 명령어 수행시 해주어야 하는 명령어이며 테라폼 명령어 사용을 위해 해주어야 하는 각종 설정을 진행하는 명령어입니다.

### 테라폼 process



aws ec2

- Amazon Elastic Compute Cloud(Ec2)는 안전하고 크기 조정이 가능한 컴퓨팅 용량을 클라우드에서 제공하는 웹 서비스

사용자는 간단한 web interface를 통해 간편하게 필요한 용량으로 서버를 구성할 수  있다.
컴퓨팅 리소스에 대한 포괄적인 제어권을 제공, Amazon의 검증된 컴퓨팅 인프라에서 실행할 수 있다.

ssh
Secure Shell Protocol은 네트워크 프로토콜 중 하나로 보통 클라이언트(컴퓨터)와 서버(컴퓨터)가 인터넷과 같은 network를 통해 통신할 때, 보안적으로 안전하게 통신을 하기위해 사용하는 프로토콜
보통 Password 인증과 RSA 공개키 암호화 방식으로 연결합니다.

ec2에 접속하려고 할때 접속이 되지 않는다고 하면
1. public ip확인
2. ssh 포트(서버의 listen port), 클라이언트의 접속 Port확인
3. secure group 확인
4. 퍼블릭 서브넷에 ec2의 ip가 위치해있는지 확인
	- public subnet인지 확인은 해당 서브넷이 연결된 라우팅 테이블에서 0.0.0.0/0 igw-xxxx가 있는지 확인
5. nacl도 확인

ec2를 만들때 ami를 선택하게 되는데 ami는 Os를 의미
vpc를 선택하고 subnet을 선택하게 될 때 subnet은 zone에 종속적인데 서버의 물리적 거리 data center를 의미


#### 
ssh-add로 키를 지정하면 현재 세션에 키가 등록되어 -i옵션을 사용하지 않고도 접속이 가능

서버의 .ssh/authorization_host에 public key가 있고

클라이언트의 .ssh/known_hosts엔 public key를 다운로드 받게 되어있음


#### zsh, oh-my-zsh
zsh은 Z shell의 약자로써, shell의 확장버전. 다양한 테마를 제공하고 shell의 확장기능르 제공함으로써 사용성을 높일 수 있음
(auto completion, correction)

oh-my-zsh은 zsh 설정을 관리해주는 오픈소스 프레임워크. Terminal 환경을 보다 이쁘고 효율적으로 만들어주는 역할을 합니다.

zsh는 apt-get install zsh, 등으로 설치할 수 있고
chsh(필요시 설치)로 chsh -s /bin/zsh로 손쉽게 쉘 환경을 변경할 수 있습니다.

명령어 이후 방향키로 히스토리를 검색할 수 있는데 손쉽게 지원 가능

5강

awscli , terraform 설치

aws cli는 v1, v2가 있지만 v2에 더 많은 기능이 있어 버전2로 설치하는 것을 권장합니다.


mac
https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/install-cliv2-mac.html
https://awscli.amazonaws.com/AWSCLIV2.pkg

linux

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

#### aws cli, terraform 설치 참고 

https://terraform101.inflearn.devopsart.dev/preparation/install-terraform-aws/

### 6강

aws resource등은 모두 api를 통해 생성할 수 있으며 aws cli, terraform이 사용하는 sdk등 모두 다양한 방식으로 api를 활용하며 사용하려면 계정의 권한을 사용하여 접근할 수 있게 AWS_ACCESS_KEY_ID와 AWS_SECRET_ACCESS_KEY 를 통해 aws configure로 등록해주어야 합니다.

#### 권장 방법
루트 계정을 생성하여 해당 계정의 접근 권한을 사용하지 않고 iam에서 user를 생성한 뒤 access, secret키를 발급해 사용하는것이 정석 방법

aws sts get-caller-identity 
- 접속한 사용자를 알게 해주는 명령어


terraform 작동원리

1. Local 코드 : 현재 개발자가 작성/수정하고 있는 코드
2. AWS 실제 인프라 : 실제로 AWS에 배포되어 있는 인프라
3. Backend에 저장된 상태 : 가장 최근에 배포된 테라폼 코드 형상

`가장 중요한 것`
- **AWS의 실제 인프라**와 **Backend에 저장된 상태**가 100% 일치하도록 만드는 것입니다.
- 테라폼을 운영하며 이 두가지가 100% 동일하도록 유지하는 것이 중요한데 이를 위해 import, state 등 여러 명령어를 제공합니다.

terraform.io/intro/index.html

  
Terraform init
- 지정한 backend에 상태 저장을 위한 .tfstate 파일을 생성합니다. 여기에는 가장 마지막에 적용한 테라폼 내역이 저장됩니다.
- init 작업을 완료하면, local에는 .tfstate에 정의된 내용을 담은 .terraform 파일이 생성됩니다.
- 기존에 다른 개발자가 이미 .tfstate에 인프라를 정의해 놓은 것이 있다면, 다른 개발자는 init작업을 통해서 local에 sync를 맞출 수 있습니다.
Terraform plan
- 정의한 코드가 어떤 인프라를 만들게 되는지 미리 예측 결과를 보여줍니다. 단, plan을 한 내용에 에러가 없다고 하더라도, 실제 적용되었을 때는 에러가 발생할 수 있습니다.
- Plan 명령어는 어떠한 형상에도 변화를 주지 않습니다.
Terraform apply
- 실제로 인프라를 배포하기 위한 명령어입니다. apply를 완료하면, AWS 상에 실제로 해당 인프라가 생성되고 작업 결과가 backend의 .tfstate 파일에 저장됩니다.
- 해당 결과는 local의 .terraform 파일에도 저장됩니다.
Terraform import
- AWS 인프라에 배포된 리소스를 terraform state로 옮겨주는 작업입니다.
- 이는 local의 .terraform에 해당 리소스의 상태 정보를 저장해주는 역할을 합니다. (절대 코드를 생성해주지 않습니다.)
- Apply 전까지는 backend에 저장되지 않습니다.
- Import 이후에 plan을 하면 로컬에 해당 코드가 없기 때문에 리소스가 삭제 또는 변경된다는 결과를 보여줍니다. 이 결과를 바탕으로 코드를 작성하실 수 있습니다.



IAC는 코드를 통해 인프라를 정의하고 배포함으로써 수동 작업을 줄이고 자동화된 구축 및 반복성을 가능하게 합니다. 이는 개발 및 배포 속도를 높여주죠.
Provider는 테라폼이 특정 인프라 유형(클라우드, 서비스 등)과 상호작용할 수 있게 해주는 플러그인입니다. 실제 생성될 자원은 Resource에 정의됩니다.
State 파일은 테라폼이 어떤 자원을 어떤 상태로 관리하고 있는지 추적하는 역할을 합니다. `.tfvars`는 변수를, `.terraform` 폴더는 초기화 정보를 담고 있습니다.

aws 7강부터

### VPC

`Amazon VPC`
- Amazon에서 제공하는 Private한 네트워크 망입니다.

`핵심 구성요소`

- **Virtual Private Cloud(VPC)** 
	- 사용자의 AWS 계정 전용 가상 네트워크입니다.
- **서브넷(subnet)**
	- VPC의 IP 주소 범위입니다.
- **인터넷 게이트웨이**
	- VPC의 리소스와 인터넷 간의 통신을 활성화하기 위해 VPC에 연결하는 게이트웨이입니다.
- **NAT 게이트웨이**
	- 네트워크 주소 변환을 통해 프라이빗 서브넷에서 인터넷 또는 기타 AWS 서비스에 연결하는 게이트웨이 입니다.
- **Security Group**
	- 보안 그룹은 인스턴스에 대한 인바운드 및 아웃바운드 트래픽을 제어하는 가상 방화벽 역할을 하는 규칙 집합입니다.
- **VPC 엔드포인트**
	- 인터넷 게이트웨이, NAT 디바이스, VPN 연결 또는 AWS Direct Connect 연결을 필요로 하지 않고 PrivateLink 구동 지원 AWS 서비스 및 VPC 엔드포인트 서비스에 VPC를 비공개로 연결할 수 있습니다.
	- VPC의 인스턴스는 서비스의 리소스와 통신하는 데 퍼블릭 IP 주소를 필요로 하지 않습니다.
	- VPC와 기타 서비스 간의 트래픽은 Amazon 네트워크를 벗어나지 않습니다.

콘솔로 aws의 리소스를 만들게되면 필요한 리소스를 한꺼번에 만들게 되지만 한계가 존재하여 코드를 통해 만들어야 합니다.
실무에선 사용하는 구조가 다르기 때문에 원하는 형태로 만들 수 있기 때문이며 코드로 만들때는 관련 리소스를 한꺼번에 만들어주지 않기 때문에 하나하나 전부 명시하여 생성해야 합니다.

```tf
provider "aws" {
  region  = "ap-northeast-2"
}

resource "aws_vpc" "main" {
  cidr_block       = "10.0.0.0/16"

  tags = {
    Name = "terraform-101"
  }
}
```

```tf
provider "aws" {
  region  = "ap-northeast-2"
}

resource "aws_vpc" "main" {
  cidr_block       = "10.0.0.0/16"

  tags = {
    Name = "terraform-101"
  }
}

resource "aws_subnet" "first_subnet" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"

  availability_zone = "ap-northeast-2a"

  tags = {
    Name = "101subnet-1"
  }
}


resource "aws_subnet" "second_subnet" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"

  availability_zone = "ap-northeast-2b"

  tags = {
    Name = "101subnet-2"
  }
}
```

```tf

resource "aws_internet_gateway" "igw" {
	
	vpc_id = aws_vpc.main.id

	tags = {
		Name = "terraform-igw-test"
	}
}

```

resource "aws_eip" "nat" {
  vpc   = true

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_nat_gateway" "nat_gateway" {
  allocation_id = aws_eip.nat.id

  # Private subnet이 아니라 public subnet을 연결하셔야 합니다.
  subnet_id = aws_subnet.first_subnet.id

  tags = {
    Name = "NAT-GW-1"
  }
}



route table을 만들면서 안에 cidr을 생성하는걸 Inner_rule이라고 명명합니다.

테라폼으로 plan, apply를 하게되면 실제 인프라를 검사하기 때문에 변경사항을 추적할 수 있게 됩니다.
