
sops

sops는 terraform에서 쓰이는 어떤 값을 암호화할때 사용합니다.
IaC는 코드로써 인프라를 관리하는 것이고, 그러면 Github와 같은 Source Repository에 업로드하고, 서로 리뷰를 해야 합니다.
그리고 그 정보는 외부에 저장되는 것과 같습니다.
(Repo가 누구나 접근할 수 있는 Public이면 정말 큰일)


vpc를 만들고 vpc를 서브네팅하여 public subnet, private subnet으로 나누게 되는데 그것을 결정하는건 nat gateway, igw gateway를 통하는 라우팅 테이블을 사용하는지의 여부로 결정되게 된다.

vpc - best pratice

예전엔 온프레미스에서 사용하는 것 처럼 회사의 층, 동 등으로 서브넷을 상세하게 구분하여 사용하는 경우가 잦았음.

하지만 cloud에 서비스를 올리는 관점에서는 ha의 관점에서 az를 구분하는것이 중요!

#### subnetting 관점

과거의 cidr은 24비트로 짤라 사용하는 경우가 잦았음 - 내 케이스도 비슷
 
cloud에선 alb, nlb처럼 public interfacing하는 애들이나 bastion server, nat gateway, public endpoint를 사용하는 경우에만 사용하기 때문에 public subnet을 고민하여 subnetting하는경우는 드뭄
이 외엔 대부분 private subnet을 사용하기 때문에 Public subnet은 /22정도로 크게 잡음 - lb가 public subnet에 많이 위치하여 public ip를 점유해야 하기 때문
private subnet은 az별 /20 - 4000개 정도의 ip를 사용하는게 best pratice

ip를 고정적으로 하드코딩하여 사용하는것이 worst case이며 ip를 고정적으로 사용하는등을 벗어나는게 best pratice임

과거 읽기 편해 /24등 고정적인 subnetting과 ip를 사용했지만 이제 cloud-native환경에선 ip의 대역이 아니라 secrit group의 id를 넣는게 좋은것

화자는 보통

/16의 cidr에서

/22로 az별 public subnet을 만들고 nat gateway가 연결된 Private subnet1, nat gateway(x) private subnet2(보통 db에 해당- 이 경우 nat등을 통하지않고 외부에 나가더라도 aws의 망을 통해 나감? 이건뭐지)
이런식으로 구성한다고함

az의 장애는 실제로 발생하기도 하며 service의 성격, 비지니스의 성격에 따라 각 region의 az를 몇개 쓸지 선택하게됨

az의 개수를 선택하는건 service의 크기, 성격별로 정하는게 나음


Subnet을 나누는 기준은 용도별이 아니라 Routing Table 분할할때 사용하는 것이다.

Route table 에 따라서 서브넷을 종속시키고, 구분시켜야 한다.
