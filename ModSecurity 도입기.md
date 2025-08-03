현재 운영 중인 환경은 Nginx 컨테이너를 기반으로 클라이언트 서비스를 배포하여 구성되어 있다.  
사용자가 많지 않더라도, 외부에서 언제든 다양한 방식의 스크립트 공격이나 서버 침투 시도가 발생할 수 있는 구조다.

이러한 보안 위협에 대비하기 위한 여러 방법 중 하나로 **ModSecurity**를 도입해보았다.

---

**ModSecurity**는 WAS 앞단에서 동작하는 \*\*웹 애플리케이션 방화벽(WAF)\*\*이다.  
일반적인 네트워크 방화벽과는 달리, 웹 애플리케이션 특화 공격에 대응하도록 설계되어 있다는 점에서 차별점이 있다.

ModSecurity는 대표적으로 다음과 같은 웹 공격을 탐지하고 차단할 수 있다:

-   SQL Injection
-   XSS(Cross-site Scripting)
-   CSRF(Cross-site Request Forgery)
-   JavaScript 기반의 악성 요청 등

특히, 단순 차단에 그치지 않고 **모든 의심 요청을 로그로 기록**하며,  
환경에 맞게 룰셋을 커스터마이징할 수 있다는 점에서 유연한 운영이 가능하다.

---

> AS-IS

<img width="542" height="172" alt="스크린샷 2025-08-02 오후 3 17 40" src="https://github.com/user-attachments/assets/b64492c5-4e56-49f7-926a-c29620779a83" />


현재 시스템 구성은 다음과 같다.

-   **Nginx**를 통해 프론트엔드 프로젝트를 정적 파일로 서빙하고 있으며
-   **백엔드 서버**는 별도로 분리되어 동작 중이다.

다만, 보안 관련 설정이 별도로 적용되어 있지 않아,  
SQL Injection이나 URL에 스크립트를 삽입하는 방식의 공격에 대해 **취약한 상태**다.  
외부 요청에 대한 필터링이나 차단 기능이 없기 때문에, 기본적인 웹 공격에도 노출될 가능성이 존재한다.

<img width="1195" height="80" alt="224" src="https://github.com/user-attachments/assets/c5dde81d-6f75-4485-ac8d-a49631f47475" />

- 그대로 노출되는 공격들

<br />

<img width="3840" height="2010" alt="as-is_diagram" src="https://github.com/user-attachments/assets/929a2cfd-7aea-46cd-b0c8-abcf34c1d1c4" />

---

> TO-BE

그래서 이런 구조적인 문제를 조금이라도 개선해보려고 보안 쪽 방법을 찾아보다가,  
**ModSecurity**라는 걸 알게 됐다.

코드 자체를 일일이 수정하거나, 백엔드에 복잡한 인증 시스템을 얹는 것보단  
앞단에서 공격을 먼저 막아주는 구조가 필요하다고 생각했다.

웹 요청을 들어오는 시점에 실시간으로 검사하고, 악성 요청은 서버에 도달하기 전에 차단한다는 점에서 도입이 꼭 필요하다고 생각들었다.

ModSecurity는 **nginx 앞단에서 동작한다는점,**  
게다가 기존에 쓰고 있던 **nginx 도커 이미지를 ModSecurity 포함 이미지로 교체만 해주면 적용 가능**해서  
도입 자체도 어렵지 않았다.

일반 방화벽이 포트나 IP를 기준으로 막는다면,  
ModSecurity는 웹 요청 안의 내용(쿼리, 스크립트 등)을 분석해서  
XSS나 SQL Injection 같은 공격을 사전에 차단할 수 있기 때문이다.

어떤 요청이 차단됐는지도 로그로 전부 기록해주기 때문에  
나중에 문제를 분석하거나 룰을 조정할 때도 꽤 유용할것이라고 생각이 들었고,

어쨌든 지금의 단순한 구조에 보안을 덧붙이기엔  
ModSecurity가 꽤 괜찮은 선택지라고 생각해서, 실제로 도입해보기로 했다.


<img width="3840" height="2893" alt="diagram_tobe" src="https://github.com/user-attachments/assets/271ab3a0-b590-48e2-8104-3077ac3c5f87" />


1\. NGINX 이미지 변경

```
# 변경전
  nginx:
    container_name: nginx
    image: nginx:latest  # 이부분 변경
    ...



# 변경후
  nginx:
    container_name: nginx
    image: owasp/modsecurity-crs:nginx   # 이부분 변경
    ...
```

<img width="637" height="175" alt="225" src="https://github.com/user-attachments/assets/974b9bc9-e140-4cf2-8a65-a9349a478077" />
- 변경된 nginx 이미지

<br />

1.2 이미지를 변경했지만 여전히 공격이 허용되는 NGINX...

<img width="1286" height="223" alt="스크린샷 2025-08-02 오후 3 59 17" src="https://github.com/user-attachments/assets/001dc7f2-33fb-4b60-9a2c-6a01ddbafc2b" />
- 여전히 SQL Injection과 스크립트 공격이 허용되고있다..

---

무엇이 문제일까?

1.3 owasp/modsecurity-crs:nginx 컨테이너 내 **SecRuleEngine **DetectionOnly(디폴트 모드)**** 

(On | Off | DetectionOnly)

/etc/modsecurity.d/modsecurity.conf 속성의

**SecRuleEngine 속성이 On 일때 악성 요청 차단**

```
해결책


1. 로컬 경로에 modsecurity.conf 생성
(도커 내 modsecurity.conf 복사)


2. SecRuleEngine Detection -> On 으로 변경


3. 도커 컴포즈 사용 시 Nginx 볼륨 마운트 설정
(로컬 modsecurity.conf / 도커 경로)




4. CRS 셋업 및 보안 규칙 관련 정보 main.conf에 추가 후 마찬가지로 볼륨 마운트
(
	~마운트 경로/crs-setup.conf
	~마운트 경로/rules/*.conf
)
```

+

nginx.conf 내 악성 요청 방지를 원하는 블록에 modsecurity on 설정 추가!

> 그리고..
<img width="1908" height="773" alt="sql_injection" src="https://github.com/user-attachments/assets/0069a6a6-10bf-4c16-b8be-85c9bdc0f7ce" />
- 403으로 처리되는 SQL Injection

<br />


<img width="1705" height="790" alt="스크립트공격" src="https://github.com/user-attachments/assets/9d17212b-3020-45a3-90b7-c6ab74e5f1c8" />
- 마찬가지로 403으로 처리되는 스크립트 공격

<br />
<img width="1270" height="92" alt="스크린샷 2025-08-02 오후 9 09 35" src="https://github.com/user-attachments/assets/93373893-e56b-45e5-b5e2-0fba6fdd0c44" />
- 403 처리 후 로그도 남는 모습

<br />

> 설정을 마친 후
>
> 불온하게 요청오는 SQL Injection 공격이나 스크립트 공격등을  
> 잘 처리해주는 모습을 보았다.
>
> 이후로도 ModSecurity를 활용해서 행복하게 잘 먹고 잘 살았다고 한다.
>
> 끝

+

### Trouble Shooting

> 다 전환하고 다음 날 관리자 페이지 수정 과정에서  
> 조회를 제외한 API가 동작하지 않는 문제 발생..
>

<img width="989" height="418" alt="스크린샷 2025-08-03 오후 1 56 19" src="https://github.com/user-attachments/assets/e4205fcb-43fa-41d8-a579-9ca15e55c7fb" />
- 어디서 많이 보던 403 에러 발생..

<br />
<br />


모두 403 처리가 되고 있길래 ModSecurity 관련 보안 문제일거라 바로 판단.

<img width="1271" height="147" alt="스크린샷 2025-08-03 오후 1 58 54" src="https://github.com/user-attachments/assets/35d01f44-fdc5-427a-b88c-1fc70c3be657" />

로그를 확인해보니, ModSecurity의 불온 요청 감지 임계값을 초과 해서 생긴 문제였음.

<br />
<br />

<img width="735" height="249" alt="스크린샷 2025-08-03 오후 2 01 16" src="https://github.com/user-attachments/assets/cd477bc9-0291-4631-a4c7-685b797fbc02" />
/api로 요청을 보내는 서버쪽 요청은 받을 수 있게끔 정규식 추가

<br />
<br />

<img width="1016" height="260" alt="스크린샷 2025-08-03 오후 2 02 36" src="https://github.com/user-attachments/assets/28836617-d963-4030-aacf-0dace6bcaa21" />
- 다시 정상 동작하는 API

<br />
<br />


> 결론:
>
>
>
>
> 다시 ModSecurity를 활용해서 행복하게 잘 먹고 잘 살았다고 한다.
>
> 끝
