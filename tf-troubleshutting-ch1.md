╷ │ Error: Unsupported argument │ │ on vpc.tf line 39, in resource "aws_eip" "nat_1": │ 39: vpc = true │ │ An argument named "vpc" is not expected here. ╵


예전(≈v2.x)에선 이렇게 사용(강의)

resource "aws_eip" "nat_1" {
  vpc = true
}

하지만 현재(예: AWS provider v5.x)에서는 이 필드가 기본값으로 VPC 전용이라 더 이상 vpc 속성을 받지 않는다.
→ 삭제해야 한다.

정확히 그렇다. 구체적으로 설명하면 다음과 같다.

과거 Terraform AWS provider(v2.x 이하)는

EC2-Classic(예전 AWS 네트워크)과

EC2-VPC(현재 표준 네트워크)
두 환경을 모두 지원했다.

그래서 aws_eip 리소스에서

vpc = true


라고 명시해야 “이 EIP는 VPC용이다”라고 구분했다.

하지만 2019년 이후 AWS가 EC2-Classic을 완전히 폐기하면서
Terraform provider v3.x 이후에는 모든 Elastic IP가 VPC 전용이 되었다.
→ 즉, vpc 필드는 항상 true로 간주되어 인자 자체가 제거되었다.

그래서 지금 버전(v5.x 기준)에서는 vpc 쓰면

“Unsupported argument”
오류가 발생한다.

결론

예전: EC2-Classic과 VPC를 구분해야 해서 vpc=true 필요

지금: VPC만 남았기 때문에 기본값으로 처리되고 vpc 키워드 자체 삭제됨
