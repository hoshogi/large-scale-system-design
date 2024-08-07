# 8. URL 단축기 설계

### URL 단축기란?

[https://www.hoshogi.com/q=chatsystem&c-lggedin&v=v3&l=long](https://www.hoshogi.com/q=chatsystem&c-lggedin&v=v3&l=long) 

→ `https://tinyurl.com/y7ke-ocwj`

위와 같이 **긴 URL을 단축 URL로 바꾸어서 제공**하며, **단축 URL을 통해서 접근했을 때 원래의 긴 URL 접속**하도록 하는 것을 말합니다.

## URL 단축기 설계

### 설계 범위

- 단축 URL 구성 : **`숫자(0~9), 영문자(a~z, A~Z)`**
- 쓰기 연산: 매일 1억개의 단축 URL 생성
- 초당 쓰기 연산 : 1억 / 24 / 3600 = 1160
- 초당 읽기 연산 : 쓰기 연산 * 10 = 11600
- URL 단축 서비스를 10년간 운영한다고 했을 때
    - 1억 * 10 * 365 = 3650억개의 레코드를 보관
- 축약전 URL 평균 길이 : 100

→ 10 년 동안 필요한 저장 용량 : 3650억 * 100바이트 = **`36.5TB`**

### API Endpoint

클라이언트는 서버가 제공하는 API 엔드포인트를 통해 서버와 통신한다.

URL 단축기는 기본적으로 **2개의 엔드포인트 필요**

1. **URL 단축용 엔드포인트**
    1. longUrl → 단축 URL
2. **URL 리다이렉션용 엔드포인트**
    1. 단축 URL → 리다이렉션 목적지가 될 원래 longUrl

### Redirection
<img width="1084" alt="스크린샷 2024-08-08 오전 3 59 22" src="https://github.com/user-attachments/assets/5640c33a-daee-45b5-b7b9-af30793a636b">

브라우저에 단축 URL을 받은 서버는 그 URL을 **원래 URL로 바꾸어서 Location 헤더**에 넣고, **301 응답**을 반환

<img width="579" alt="스크린샷 2024-08-08 오전 4 03 50" src="https://github.com/user-attachments/assets/baeb0087-d711-4a93-acaa-fb7b9ce9545c">

- 301 Permanently Moved
    - 해당 URL에 대한 HTTP 요청의 처리가 **영구적으로** Location 헤더에 반환된 URL로 이전되었다는 뜻
    - 영구적으로 이전되었으므로, **브라우저는 이 응답을 캐시**한다.
- 302 Found
    - **일시적으로** Location 헤더가 지정하는 URL에 의해 처리되어야 한다는 응답
    - 클라이언트 요청은 언제나 단축 URL 서버에 먼저 보내진 후에 원래 URL로 리다이렉션 된다.

→ 서버 부하를 줄이는 것이 중요하다면 301 Permanent Moved, 트래픽 분석이 중요할 때는 302 Found

### URL 단축 방법

URL 리다이렉션을 구현할 때는 **해시테이블**을 사용

Key : Value = **단축 URL : 원래 URL**

<img width="333" alt="스크린샷 2024-08-08 오전 4 18 40" src="https://github.com/user-attachments/assets/c1c53979-c078-4beb-ab15-8d338ae7a5e9">

가장 중요한 것은 위 그림 처럼 긴 URL을 해시 값으로 대응시킬 **해시 함수 fx**를 찾는 것

**해시 함수 조건**

- 입력으로 주어지는 긴 URL이 다른 값이면 **해시 값도 달라**야 한다.
- 계산된 해시 값은 원래 입력으로 주어졌던 **긴 URL로 복원**될 수 있어야 한다.

### 데이터 모델

메모리는 유한한데 비싸기 때문에 해시테이블을 사용하기보다 **<단축 URL, 원래 URL>의 순서쌍을 RDBMS에 저장**하는 것이 좋다.

### 해시 함수

- 단축 URL 구성 : **`숫자(0~9), 영문자(a~z, A~Z)`**
    
    → 사용할 수 있는 문자의 개수는 **62개**
    
- URL 단축 서비스를 10년간 운영한다고 했을 때
    - 1억 * 10 * 365 = **3650억개**의 레코드를 보관

해시 충돌을 피하기 위해서는 **$62^n>= 3265억$**을 만족하는 **n의 최소값**을 찾아야한다.

<img width="423" alt="스크린샷 2024-08-08 오전 4 36 40" src="https://github.com/user-attachments/assets/72202ddf-0731-43ba-b84d-185eff9f2057">

**n = 7**

→ 해시 함수 반환값의 길이 : 7

## 해시 함수 구현

### 해시 후 충돌 해소

[https://en.wikipedia.org/wiki/Systems_design](https://en.wikipedia.org/wiki/Systems_design)

<img width="571" alt="스크린샷 2024-08-08 오전 4 35 10" src="https://github.com/user-attachments/assets/08bb6970-449d-4342-ad8c-cc6f0d4d4c92">

위 URL을 다양한 해시 함수를 반환된 값을 보면 가장 짧은 것도 7자리가 넘는다.

자리 수를 줄이기 위해 반환 값의 **앞의 7자리만 사용**하면 해시 충돌 확률이 높아진다.

따라서 충돌이 해소될 때까지 **사전에 정한 문자열을 해시 값에 덧붙**인다.

<img width="835" alt="스크린샷 2024-08-08 오전 5 11 50" src="https://github.com/user-attachments/assets/8191024c-22e0-4892-bb25-732f14b85f39">

이 방법은 충돌은 해소할 수 있지만 **한 번이상의 데이터베이스 접근이 필요하기 때문에 오버헤드가 크**다.

데이터베이스 대신 **`블룸 필터`**를 사용하면 성능을 높일 수 있다.

> **블룸 필터란?**
특정 원소가 집합에 속하는지 검사하는데 사용할 수 있는 **확률형 자료구조**
어떤 **원소가 집합에 속한다고 판단된 경우 실제로는 원소가 집합에 속하지 않는 긍정 오류가 발생**하는 것은 가능하지만
**원소가 집합에 속하지 않는 것으로 판단되었는데 실제로는 원소가 집합에 속하는 부정 오류**는 절대로 발생하지 않는다는 특징

처리능력대비 적은 메모리 공간을 필요로 하는 장점
1억 유저 단위 서비스에서 신규 유저인지 검사하는데 100MB 정도의 메모리만을 사용
> 

### base-62 변환

진법 변환은 URL 단축기를 구현할 때 흔히 사용되는 접근법

**수의 표현 방식이 다른 두 시스템이 같은 수를 공유하여야 하는 경우**에 유용!

62진법을 쓰는 이유는 hashValue에 사용할 수 있는 문자의 개수가 62개이기 때문

| 해시 후 충돌 해소 전략 | base-62 변환 |
| --- | --- |
| 단축 URL의 길이가 고정됨 | 단축 URL의 길이가 가변적 |
| 유일성이 보장되는 ID 생성기가 필요치 않음 | 유일성 보장 ID 생성기가 필요 |
| 충돌이 가능해서 해소 전략이 필요 | ID의 유일성이 보장된 후에야 적용 가능한 전략이라 충돌은 아예 불가능 |
| ID로부터 단축 URL을 계산하는 방식이 아니라서 다음에 쓸 수 있는 URL을 알아내는 것이 불가능 | ID가 1씩 증가하는 값이라고 가정하면 다음에 쓸 수 있는 단축 URL이 무엇인지 쉽게 알아낼 수 있어서 보안상 문제가 될 수 있는 있음 |

### URL 단축기 상세 설계

<img width="643" alt="스크린샷 2024-08-08 오전 5 11 08" src="https://github.com/user-attachments/assets/b284f8f2-0f88-483a-8014-e51d978d7e43">

ID 생성기의 주된 용도는 단축 URL을 만들 때 사용할 ID를 만드는 것
이 ID는 전역적으로 유일성이 보장되어야 한다.

### URL 리다이렉션 상세 설계

<img width="808" alt="스크린샷 2024-08-08 오전 5 13 48" src="https://github.com/user-attachments/assets/ea2208e9-500a-4b77-bdb7-e6b077136d5a">
<단축 URL, 원래 URL>의 쌍을 캐시에 저장하여 성능을 높였다.

**로드밸런서의 동작 흐름**
1. 사용자가 단축 URL을 클릭
2. 로드밸런서가 해당 클릭으로 발생한 요청을 웹 서버에 전달
3. 단축 URL이 이미 캐시에 있는 경우에는 원래 URL을 바로 꺼내서 클라이언트에게 전달
4. 캐시에 해당 단축 URL이 없는 경우에는 데이터베이스에서 꺼낸다. 데이터베이스에 없다면 사용자가 잘못된 단축 URL을 입력한 경우일 것이다.
5. 데이터베이스에서 꺼낸 URL을 캐시에 넣은 후 사용자에게 반환