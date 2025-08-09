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

[##_Image|kage@baxDdT/btsPMRFADCl/AAAAAAAAAAAAAAAAAAAAAIX9xtDZkSD4N2un_iAr23wcsWqGpZ37AZjeqwdSkXAs/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1756652399&amp;allow_ip=&amp;allow_referer=&amp;signature=oUE%2FMb7fi0zKIUVZrIulpaeHCn8%3D|CDM|1.3|{"originWidth":3840,"originHeight":2189,"style":"alignCenter","width":700,"height":399,"caption":"대충 아키텍처를 검사해준다는 이미지"}_##]

---

> 2\. 테스트 구현 (패키지 구조 테스트)

2.1 ArchUnit 의존성 추가 

[##_Image|kage@cMplYv/btsPMIhIwVS/AAAAAAAAAAAAAAAAAAAAAMYgV1_Kl2MjbCAATXoaPWSwKvhaxijPJXuxgb6FZh8R/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1756652399&amp;allow_ip=&amp;allow_referer=&amp;signature=G2DmA7Iis1uvNHHFt2iFQjPLCPU%3D|CDM|1.3|{"originWidth":518,"originHeight":68,"style":"alignCenter"}_##]

2.2 ~Service 파일들이 /service 디렉토리 내부에 있는지 테스트

[##_Image|kage@nIwpZ/btsPM8mTK9z/AAAAAAAAAAAAAAAAAAAAALq8byQGSsjQ3HlrbTKezGQjoG7jm07eQhtgbLuVUDja/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1756652399&amp;allow_ip=&amp;allow_referer=&amp;signature=bkNp40XfCJa75QI8hSg5yY3Ip%2Fc%3D|CDM|1.3|{"originWidth":549,"originHeight":329,"style":"alignCenter"}_##]

-   com.pro.salehero 내부에 있는 아키텍처 테스트한다는 스코프 명시
-   @ArchTest 어노테이션을 추가하고
-   \*Service 파일들이 service 패키지내에 있는지 테스트

결과:

[##_Image|kage@HrJ4G/btsPLMSrlVc/AAAAAAAAAAAAAAAAAAAAAEj2kj4TmrwRF5FFwbb7zofo9ip6sB6_UOtw2VXzdg_N/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1756652399&amp;allow_ip=&amp;allow_referer=&amp;signature=9ecIhQYu57VzyyDBap5fMefaHks%3D|CDM|1.3|{"originWidth":594,"originHeight":304,"style":"alignCenter","caption":"성공"}_##]

테스트 특)

\- 한가지 테스트하면 성공하든 실패하든 불안함

\- 2.3으로 개시

2.3  2.2와 같은 맥락으로 컨트롤러 테스트

[##_Image|kage@d89owD/btsPNEslllr/AAAAAAAAAAAAAAAAAAAAAAnYv_h0LhyDtO0RJ8yBL6AXH0aptyA9aeNyFEOYOMj3/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1756652399&amp;allow_ip=&amp;allow_referer=&amp;signature=nZUGMZncQDXJm3tFkF%2Bv1LDO3ec%3D|CDM|1.3|{"originWidth":554,"originHeight":476,"style":"alignCenter","caption":"하나가 더 추가됨"}_##]

결과:

[##_Image|kage@dYJNen/btsPM97bmlF/AAAAAAAAAAAAAAAAAAAAAB2S7Ew_efx61obTWU_tVl7Xa_fs04bdumgtY90crf9v/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1756652399&amp;allow_ip=&amp;allow_referer=&amp;signature=wOUF245Bsl8BROixkV7Btgzp0O0%3D|CDM|1.3|{"originWidth":424,"originHeight":131,"style":"alignCenter","caption":"새로 추가한 테스트가 실패했다."}_##]

다행히(?) 새로 추가한 테스트가 실패했다.

실패했으니 실패 로그를 한번 살펴보자.

[##_Image|kage@S29FD/btsPNNvQJS7/AAAAAAAAAAAAAAAAAAAAAAV44dQKQz779mez8UJXaH_0Yu-JrBOyW_UI_vP9Df14/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1756652399&amp;allow_ip=&amp;allow_referer=&amp;signature=GBDNf%2FOt%2B4kw%2BQMZQ5qrjwwFM9U%3D|CDM|1.3|{"originWidth":1323,"originHeight":303,"style":"alignCenter","width":750,"height":172,"caption":"EmailTemplateController.kt 가 무언가 문제가 있다고 한다."}_##]

(EmailTemplateController.kt:)

[##_Image|kage@2rC2I/btsPL2ASkpS/AAAAAAAAAAAAAAAAAAAAAIx5-vzFdkrddnqBtDnCii48E7tCE3TrMUwyzTvvIAoG/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1756652399&amp;allow_ip=&amp;allow_referer=&amp;signature=O%2FcaPJDbwpHytUae%2FIUYrAVFP7A%3D|CDM|1.3|{"originWidth":652,"originHeight":427,"style":"alignCenter","caption":"dto 패키지 안에 있던 컨트롤러 클래스..."}_##]

테스트가 실패한 이유는 실제로 내가 의도한거처럼 \*Controller 클래스가 /controller 패키지 내부에 있지 않아서였으며

위 이미지와 같이 /dto 내부에 있어서 테스트가 실패한거였다.

.... 이후 controller 하위로 수정하여 개선

2.4 리포지토리 테스트

[##_Image|kage@DtJIA/btsPMY5Pk34/AAAAAAAAAAAAAAAAAAAAAK5YDm4pR9jEqR3GpjtBsGWuNL4Fk9sUKf8Uyqkf6DkC/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1756652399&amp;allow_ip=&amp;allow_referer=&amp;signature=w9d0jiReNpqtVWPA0JVAW1ppCRk%3D|CDM|1.3|{"originWidth":532,"originHeight":597,"style":"alignCenter","caption":"리포지토리는 새로운 패턴"}_##]

\* 리포지토리는 엔티티와 함께 보관하는 구조를 따르고 있어서

\* 리포지토리 테스트는 domain 패키지 내부에 있는 경우로 테스트를 진행했다.

대부분 정상적으로 동작했고, 테스트가 한번 실패해서 확인했더니

[##_Image|kage@bfsOoj/btsPLV9yvDB/AAAAAAAAAAAAAAAAAAAAAMC-4o5pcy7YfKhq3Qf5II_vo2jzZDj1TNqjD_a0VlBO/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1756652399&amp;allow_ip=&amp;allow_referer=&amp;signature=js0GMX63i4JfvTbMMfEQU%2BXOFFI%3D|CDM|1.3|{"originWidth":280,"originHeight":127,"style":"alignCenter","caption":"오타 발견"}_##]

domain -> doomain으로 잘못적음..

[##_Image|kage@cizYWZ/btsPMSxJCLT/AAAAAAAAAAAAAAAAAAAAANbSndqlbVM82SWjMg6gwCLDx3W6JIEKFqSYJIhXYzwC/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1756652399&amp;allow_ip=&amp;allow_referer=&amp;signature=MLAFaIbNRPhLogTR1bqJNWRC9QI%3D|CDM|1.3|{"originWidth":541,"originHeight":393,"style":"alignCenter","caption":"컨트롤러, 서비스, 리포지토리 테스트"}_##]

---

> 3\. 의존성 규칙 테스트

2번 항목에서 패키지안에 알맞은 네이밍 형태를 따르는지 테스트를 해보았다.

ArchUnit테스트로 할 수 있는 테스트들은 더 많다.

이번에는 어떤 네이밍 파일들이 어떤 패키지 파일들에서 호출되는지 테스트 해보았다.

ex) ~Repository를 ~service 패키지 내부의 파일들이 호출되어야 맞지, ~controller에서 호출하면 안됨

3.1 의존성 규칙 테스트용 테스트 생성

\* 테스트시 모킹을 위해 의존성 주입을 했을 수 있으니 테스트는 제외하도록 한다.

<img width="3840" height="2189" alt="1" src="https://github.com/user-attachments/assets/19e77d48-541e-40b0-8328-751b74902775" />

<br/ >
<br />


3.2 테스트 구현부

[##_Image|kage@14fiP/btsPMVgYoOd/AAAAAAAAAAAAAAAAAAAAAJjHe2sj6Z_ZRdjuzEiCEEZPrDzf1VfRNEt_eBffegXi/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1756652399&amp;allow_ip=&amp;allow_referer=&amp;signature=qroKIEbSYQ0Z5dlwl3xy7BKkxqA%3D|CDM|1.3|{"originWidth":799,"originHeight":477,"style":"alignCenter","width":700,"height":418,"caption":"테스트"}_##]

> 주요 흐름은 다음과 같다.  
>   
> \*Repository, Service 등의 이름을 가진 파일들이  
> ..service.. 등의 패키지구조에서 호출되는지 테스트

[##_Image|kage@boncB1/btsPNO9rXDa/AAAAAAAAAAAAAAAAAAAAAPZY_sLy5bBu5IXVtIhX90Bh6PUsFfkMPmapDs0JFIRQ/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&amp;expires=1756652399&amp;allow_ip=&amp;allow_referer=&amp;signature=vI4fAj9FJKu%2B%2FgYq4LxUdCMniE0%3D|CDM|1.3|{"originWidth":1022,"originHeight":316,"style":"alignCenter","width":700,"height":216,"caption":"테스트 성공"}_##]

> 결론:  
>   
>   
> 손바닥 뒤집듯 고친 내 아키텍처들은 매우 불안정한 상태에 있었고  
> ArchUnit 테스트를 통해 개선 및 구조의 규칙을 확보할 수 있었다.

출처 ([https://www.archunit.org/](https://www.archunit.org/))
