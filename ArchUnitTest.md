---

ArchUnit Test에 대해 알아보고 적용해보자.

내 프로젝트 아키텍처가 얼마나 **일관성 있는지** 학습해보자.

---

> 1\. 히스토리

현재 기능을 먼저 개발하고 테스트 커버리지를 올리고있는 프로젝트가 있다.

해당 프로젝트의 디렉토리 구조나 아키텍처를 손바닥 뒤집듯 고치다보니

얼마나 일관성 있는 구조로 완성되고있는지 테스트가 필요해졌다.

---

## ArchUnit이란?

> Java 단위 테스트 프레임워크를 사용하여 Java 코드의 아키텍처를 검사할 수 있는 무료의 간편하고 확장 가능한 라이브러리이자  
> 패키지와 클래스, 레이어와 슬라이스 간의 종속성을 검사하고, 순환 종속성을 검사하는 등 다양한 기능을 포함하고있다  
>   
> 고 한다.  
>   
>   
> 우선 무슨말인지 잘 모르겠다.

선 3줄요악

1\. Java / Kotlin 프로젝트 아키텍처 규칙을 검증

2\. 코드레벨에서 프로젝트 원칙을 강제 할 수 있음

3\. 그래서 쓰면 좋음

테스트 작업은 다음 다이어그램의 순서를 따른다.

<img width="1280" height="729" alt="1" src="https://github.com/user-attachments/assets/58d2ef70-f430-4fff-81cb-780d9ec2212f" />


---

> 2\. 테스트 구현 (패키지 구조 테스트)

2.1 ArchUnit 의존성 추가 

<img width="518" height="68" alt="2" src="https://github.com/user-attachments/assets/5e901697-cc44-498d-9877-3898a7d87359" />


2.2 ~Service 파일들이 /service 디렉토리 내부에 있는지 테스트

<img width="549" height="329" alt="3" src="https://github.com/user-attachments/assets/98659db9-c8f6-4a59-b2fb-afece40c43c1" />


-   com.pro.salehero 내부에 있는 아키텍처 테스트한다는 스코프 명시
-   @ArchTest 어노테이션을 추가하고
-   \*Service 파일들이 service 패키지내에 있는지 테스트

결과:

<img width="594" height="304" alt="4" src="https://github.com/user-attachments/assets/cdfca993-36f1-4e49-97ae-c07a8a138899" />


테스트 특)

\- 한가지 테스트하면 성공하든 실패하든 불안함

\- 2.3으로 개시

2.3  2.2와 같은 맥락으로 컨트롤러 테스트

<img width="554" height="476" alt="5" src="https://github.com/user-attachments/assets/c59fd8d0-6521-431a-ba42-f839e7c64265" />


결과:

<img width="424" height="131" alt="6" src="https://github.com/user-attachments/assets/e27296b9-ad6c-49cd-be5d-70ee01650889" />


다행히(?) 새로 추가한 테스트가 실패했다.

실패했으니 실패 로그를 한번 살펴보자.

<img width="1280" height="293" alt="7" src="https://github.com/user-attachments/assets/f69a93a6-bbee-45fa-ab8a-7bb7f1f23a0c" />


(EmailTemplateController.kt:)

<img width="652" height="427" alt="8" src="https://github.com/user-attachments/assets/0069248e-6046-4f17-bb75-642649374364" />


테스트가 실패한 이유는 실제로 내가 의도한거처럼 \*Controller 클래스가 /controller 패키지 내부에 있지 않아서였으며

위 이미지와 같이 /dto 내부에 있어서 테스트가 실패한거였다.

.... 이후 controller 하위로 수정하여 개선

2.4 리포지토리 테스트

<img width="532" height="597" alt="9" src="https://github.com/user-attachments/assets/49d03422-480a-4268-bbb5-08ef9ce311d8" />


\* 리포지토리는 엔티티와 함께 보관하는 구조를 따르고 있어서

\* 리포지토리 테스트는 domain 패키지 내부에 있는 경우로 테스트를 진행했다.

대부분 정상적으로 동작했고, 테스트가 한번 실패해서 확인했더니

<img width="280" height="127" alt="10" src="https://github.com/user-attachments/assets/515d71ee-e2c7-4b5c-9dc2-2a616194c887" />


domain -> doomain으로 잘못적음..

[##_Image|kage@cizYWZ/btsPMSxJCLT/AAAAAAAAAAAAAAAAAAAAANbSndqlbVM82SWjMg6gwCLDx3W6JIEKFqSYJIhXYzwC/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1759244399&amp;allow_ip=&amp;allow_referer=&amp;signature=Jo5CMxWqAe1tYAvJIsXnWHq%2FIF0%3D|CDM|1.3|{"originWidth":541,"originHeight":393,"style":"alignCenter","caption":"컨트롤러, 서비스, 리포지토리 테스트"}_##]

---

> 3\. 의존성 규칙 테스트

2번 항목에서 패키지안에 알맞은 네이밍 형태를 따르는지 테스트를 해보았다.

ArchUnit테스트로 할 수 있는 테스트들은 더 많다.

이번에는 어떤 네이밍 파일들이 어떤 패키지 파일들에서 호출되는지 테스트 해보았다.

ex) ~Repository를 ~service 패키지 내부의 파일들이 호출되어야 맞지, ~controller에서 호출하면 안됨

3.1 의존성 규칙 테스트용 테스트 생성

\* 테스트시 모킹을 위해 의존성 주입을 했을 수 있으니 테스트는 제외하도록 한다.

[##_Image|kage@bTqJrx/btsPOFxBxmi/AAAAAAAAAAAAAAAAAAAAADg1FqjcnLup99jA9UoPqN_ohzWxfd1VsyP-ryd1qKkw/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1759244399&amp;allow_ip=&amp;allow_referer=&amp;signature=7nQpZkUJ4aM%2FXV0cQKuzGd%2BfSKg%3D|CDM|1.3|{"originWidth":471,"originHeight":230,"style":"alignCenter"}_##]

3.2 테스트 구현부

[##_Image|kage@14fiP/btsPMVgYoOd/AAAAAAAAAAAAAAAAAAAAAJjHe2sj6Z_ZRdjuzEiCEEZPrDzf1VfRNEt_eBffegXi/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1759244399&amp;allow_ip=&amp;allow_referer=&amp;signature=RcKouN0zr8DRDR9fFVH0rTjMYcE%3D|CDM|1.3|{"originWidth":799,"originHeight":477,"style":"alignCenter","width":700,"height":418,"caption":"테스트"}_##]

> 주요 흐름은 다음과 같다.  
>   
> \*Repository, Service 등의 이름을 가진 파일들이  
> ..service.. 등의 패키지구조에서 호출되는지 테스트

[##_Image|kage@boncB1/btsPNO9rXDa/AAAAAAAAAAAAAAAAAAAAAPZY_sLy5bBu5IXVtIhX90Bh6PUsFfkMPmapDs0JFIRQ/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1759244399&amp;allow_ip=&amp;allow_referer=&amp;signature=uD%2F1s3VsrrZBTsm%2BpY%2BF59KBMos%3D|CDM|1.3|{"originWidth":1022,"originHeight":316,"style":"alignCenter","width":700,"height":216,"caption":"테스트 성공"}_##]

> 결론:  
>   
>   
> 손바닥 뒤집듯 고친 내 아키텍처들은 매우 불안정한 상태에 있었고  
> ArchUnit 테스트를 통해 개선 및 구조의 규칙을 확보할 수 있었다.

출처 ([https://www.archunit.org/](https://www.archunit.org/))
