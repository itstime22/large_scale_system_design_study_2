## Chapter 9. 웹 크롤러 설계 (Web Crawler Design)

> **핵심 메시지**  
> 웹 크롤러의 본질은 “많이, 빠르게, 예의 있게” 크롤링하는 것이다. 이를 위해 URL Frontier 설계(우선순위 + 예의성), 분산 크롤링, 문제 콘텐츠 필터링이 핵심이다.

---

### 9.1 문제 정의

웹 크롤러는 다음과 같은 기능을 수행하는 시스템입니다.

1. 주어진 URL을 다운로드한다.
2. HTML에서 새로운 URL을 추출한다.
3. 추출한 URL을 다시 Frontier에 넣어 재크롤링한다.
4. 중복·함정·스팸 등 문제 콘텐츠는 차단한다.
5. 웹 서버에 과도한 부하를 주지 않도록 **예의 있게(politeness)** 동작한다.

---

### 9.2 크롤러의 핵심 속성

#### 9.2.1 규모 확장성 (Scalability)
- 월 **수십억 페이지** 처리
- 여러 데이터센터·수백~수천 대의 크롤러 노드 필요

#### 9.2.2 예의성 (Politeness)
- 동일 도메인에 과도한 요청 금지
- 도메인별 **접근 간격(throttle)** 관리 (예: 도메인당 QPS 제한)

#### 9.2.3 확장성 (Extensibility)
- HTML 외에도 PNG, PDF, JSON 등 다양한 포맷 지원
- **플러그인 모듈** 형태로 콘텐츠 처리기를 쉽게 추가

#### 9.2.4 안정성 (Robustness)
- 비정상 HTML, 깨진 링크, 네트워크 오류에도 크롤러 전체가 죽지 않도록 방어 코드 필요
- 재시도, 백오프(backoff), 장애 노드 격리 등 메커니즘 필요

---

### 9.3 규모 추정 (Capacity Estimation)

- 월 크롤링 목표: **10 billion pages**
- 평균 페이지 크기: **500 KB**
- 월 저장량: \(10^{10} \times 500\text{KB} ≒ 500\text{TB}\)
- 5년 보관: **약 30 PB** (압축·샘플링 전략 필요)
- QPS: **약 400**, 피크 시 **800** (도메인별로 분산)

---

### 9.4 전체 아키텍처 개요

```
시작 URL 수집
      ↓
미수집 URL 저장소 (URL Frontier)
      ↓
도메인 이름 변환기 (DNS)
      ↓
HTML Downloader
      ↓
콘텐츠 파서(Parser)
      ↓
중복 콘텐츠 검사(Deduplication)
      ↓
URL 추출기
      ↓
URL 필터
      ↓
이미 방문한 URL인가? → 방문 URL 저장소
      ↓
신규 URL → 다시 URL Frontier로 전달
```

**핵심 컴포넌트**
- `URL Frontier`: 아직 크롤링하지 않은 URL 큐
- `Downloader`: HTTP 요청/응답 처리, robots.txt 준수
- `Parser`: HTML 파싱, 링크 추출, 메타데이터 수집
- `Dedup`: 콘텐츠/URL 중복 제거
- `URL Filter`: 스팸, 거미덫 등 필터링
- `Visited Store`: 방문한 URL 기록

---

### 9.5 URL Frontier 설계 (핵심)

웹 크롤러의 성능과 안정성의 중심은 **URL Frontier**입니다.

#### 9.5.1 Front Queue (우선순위 기반)

URL 우선순위를 점수화하여 큐를 여러 개(`f1, f2, … fn`)로 분리합니다.

**우선순위 기준 예시**
- PageRank 또는 도메인 신뢰도
- 트래픽(인기 페이지)
- 업데이트 빈도(뉴스 등)
- 신선도(freshness)
- 비즈니스 중요도 (광고, 검색 결과 상위 노출 등)

`Front Queue Selector`가 여러 큐를 순회하며 **점수가 높은 큐에서 더 자주** URL을 꺼냅니다.

#### 9.5.2 Back Queue (예의성 기반)

같은 도메인의 URL은 반드시 **동일 큐**로 보냅니다.

- `b1 = example.com`
- `b2 = amazon.com`
- `b3 = wikipedia.org`

특징:
- 각 Back Queue는 **FIFO 방식**으로 동작
- 동일 도메인에 대한 요청을 **시간 간격을 두고 순차적**으로 처리
- `Back Queue Selector`가 큐를 순차적으로 선택해 **서버 부하를 방지**

#### 9.5.3 Front + Back Queue 결합 구조

```
         입력 URL
            ↓
      우선순위 결정기
            ↓
    f1   f2   f3  ... fn   (Front Queues)
            ↓
     Front Queue Selector
            ↓
    b1   b2   b3  ... bn   (Back Queues)
            ↓
     Back Queue Selector
            ↓
       Worker Threads
            ↓
       페이지 다운로드
```

- **중요도 높은 URL**은 먼저 처리
- **같은 도메인**에는 예의 있게 요청
- 이 구조를 여러 노드로 분산하면 대규모 환경에서도 안정적으로 동작

---

### 9.6 성능 최적화 기법

#### 9.6.1 분산 크롤링 (Distributed Crawling)
- URL 공간을 여러 서버로 분할(샤딩)하고 병렬 처리
- 도메인 해시 기반으로 크롤러 노드에 할당

#### 9.6.2 DNS 최적화
- DNS 결과를 **캐싱**하여 반복 조회 비용 감소
- 비동기 DNS 라이브러리를 사용해 지연 시간 감소

#### 9.6.3 HTTP 연결 재사용
- HTTP Keep-Alive 사용
- Connection Pool 운영
- Timeout/Retry 정책을 적절히 설정

#### 9.6.4 robots.txt 준수
- 사이트별 `robots.txt` 주기적 다운로드
- Disallow 경로/크롤링 속도 규칙 준수

---

### 9.7 플러그인 아키텍처 (확장성)

HTML 외에도 PNG, PDF, JSON 등 새로운 파일 형식을 **플러그인 모듈**로 추가할 수 있게 설계합니다.

예: 콘텐츠 타입별 Downloader

```
HTML Downloader  → HTML 처리  
PNG Downloader   → PNG 파일 처리  
PDF Downloader   → PDF 파일 처리  
JSON Downloader  → API 응답 처리
```

- 새로운 포맷이 생겨도 Downloader/Parser 플러그인만 추가하면 됨
- Web Monitor 모듈로 저작권·상표권 위반 콘텐츠를 감지 가능

---

### 9.8 문제 콘텐츠 감지 및 회피

웹에는 크롤러를 멈추게 하거나 품질을 떨어뜨리는 콘텐츠가 많습니다.

#### 9.8.1 중복 콘텐츠
- 해시 체크섬(MD5/SHA-1 등)을 사용해 본문 해시 비교
- 웹의 **30% 이상이 중복 콘텐츠**라는 연구도 존재
- 중복 페이지는 인덱싱/저장에서 제외해 저장 공간과 대역폭 절약

#### 9.8.2 거미덫 (Spider Trap)

고의적으로 무한히 깊어지는 URL 구조 예:

```
/foo/bar/foo/bar/foo/bar/...
```

대응 방법:
- URL 최대 길이 제한
- 패턴 기반 필터 (반복 패턴 탐지)
- Trap 도메인 블랙리스트 관리

#### 9.8.3 데이터 노이즈
- 광고 스크립트
- 세션 기반 URL (`?session_id=...`)
- 스팸 URL
- 자동 생성된 의미 없는 페이지(doorway page)

→ 필터링을 통해 **대역폭·저장 공간**을 절약하고, 인덱스 품질을 높입니다.

---

### 9.9 추가 고려 사항

#### 서버 측 렌더링 / 동적 렌더링
- JavaScript로 링크가 동적 생성되는 경우 **Headless Browser**(예: Puppeteer, Playwright) 필요

#### 스팸 페이지 필터링
- Anti-spam 컴포넌트로 스코어링
- 머신러닝 기반 분류기 적용 가능

#### 데이터 저장 확장성
- 샤딩(sharding) + 복제(replication)
- 오래된 데이터는 샘플링 또는 요약 저장

#### 수평적 확장 (Horizontal Scalability)
- Stateless 크롤러 워커 설계
- 중앙 메타데이터(Frontier/Visited)는 분산 저장소 사용

#### 분석 시스템 (Analytics)
- 크롤링된 데이터 기반으로:
  - 크롤링 커버리지 분석
  - 실패율·응답 시간 통계
  - Frontier 정책 튜닝

---

### 9.10 요약

- 웹 크롤러는 **확장성, 예의성, 안정성**을 동시에 만족해야 한다.
- URL Frontier(Front/Back Queue) 설계가 성능과 품질의 핵심이다.
- 분산 크롤링, robots.txt 준수, 문제 콘텐츠 필터링, 플러그인 아키텍처를 통해 대규모 웹을 안전하게 크롤링할 수 있다.

---

## Chapter 10. 알림 시스템 설계 (Notification System Design)

> **핵심 메시지**  
> 알림 시스템은 다채널(푸시, SMS, 이메일) 알림을 대량·안정적으로 보내는 인프라다. 채널 분리, 메시지 큐, 재시도·Rate Limiting·템플릿·사용자 설정이 핵심이다.

---

### 10.1 문제 이해 및 설계 범위

#### 10.1.1 지원해야 할 알림 종류
- 푸시 알림 (iOS, Android)
- SMS 메시지
- 이메일

#### 10.1.2 시스템 요구사항
- **연성 실시간(Soft real-time)**
  - 즉시성이 중요하지만 수 초 이내 지연은 허용
- **다양한 단말 지원**
  - iOS, Android, Web/Desktop
- **사용자 설정 지원**
  - 채널별 opt-in / opt-out
- **대량 발송 가능**
  - 초당 수천~수만 건, 하루 수백만 건 이상 처리

---

### 10.2 알림 유형별 전송 절차

#### 10.2.1 iOS 푸시 알림 (APNS)

**구성요소**
- `Provider (알림 서버)`: 알림 데이터를 생성하여 APNS로 전송
- `Device Token`: 특정 단말을 식별하는 고유 키
- `Payload(JSON)`: 전송할 알림 내용

예시 Payload:

```json
{
  "aps": {
    "alert": {
      "title": "Game Request",
      "body": "Bob wants to play chess"
    },
    "badge": 5
  }
}
```

iOS 푸시는 APNS가 큐잉, 재시도, 전송을 모두 관리하므로 **서버는 인증된 요청만 만들면 됨**.

#### 10.2.2 Android 푸시 알림 (FCM)

APNS와 동일 구조이나 전송 서비스만 **FCM**으로 변경됩니다.

```
Provider → FCM → Android Device
```

#### 10.2.3 SMS 메시지

- Twilio, Nexmo 같은 외부 API 서비스를 통해 전송
- 특징:
  - 전송 성공률이 높은 편
  - 건당 과금 구조
  - 2FA, 긴급 알림 등 비즈니스 알림에 널리 사용

#### 10.2.4 이메일

- SendGrid, Mailchimp 등 외부 서비스 사용이 일반적
- 장점:
  - 전송 성공률 높음
  - 스팸 필터링 기술 우수
  - Open/Click 등 Analytics 제공

---

### 10.3 알림 채널 통합 구조

여러 알림 채널을 하나의 Notification System이 통합 관리합니다.

```
          ┌── APNS → iOS
          ├── FCM → Android
Provider ─┤
          ├── SMS 서비스 → Phone
          └── Email 서비스 → Mail Client
```

- Provider(알림 서버)는 **채널에 독립적인 공통 인터페이스**를 제공
- 각 채널별 어댑터가 APNS/FCM/SMS/Email 서비스와 통신

---

### 10.4 연락처 정보 수집 및 저장

#### 10.4.1 필요한 정보
- Device Token (푸시)
- Phone Number (SMS)
- Email Address (이메일)

#### 10.4.2 수집 절차
1. 앱 설치 또는 계정 생성 시 서버로 정보 전송
2. API 서버가 DB에 저장

#### 10.4.3 DB 스키마 예시

`user` 테이블

| 필드           | 타입      | 설명   |
| ------------- | --------- | ------ |
| user_id       | bigint    | PK     |
| email         | varchar   | 이메일 |
| phone_number  | varchar   | 전화번호 |
| ...           |           |        |

`device` 테이블

| 필드                | 타입        | 설명      |
| ------------------- | ----------- | --------- |
| id                  | bigint      | PK        |
| device_token        | varchar     | 단말 토큰 |
| user_id             | bigint      | FK        |
| last_logged_in_at   | timestamp   | 최근 로그인 |

---

### 10.5 알림 전송 절차 (초안)

1. 서비스가 Notification Server API 호출
2. 알림 서버가 메타데이터(DB/Cache) 조회
3. 알림 유형에 맞는 이벤트 생성
4. 이벤트를 메시지 큐에 저장
5. Worker(작업 서버)가 큐에서 이벤트를 소비
6. 제3자 서비스(APNS/FCM/SMS/Email 서비스)로 전송
7. 사용자에게 도착

---

### 10.6 초안 설계의 문제점

- **SPOF(단일 장애점)**: 알림 서버가 하나면 장애 시 전체 서비스 중단
- **확장성 부족**: 모든 알림 채널을 한 서버가 처리
- **성능 병목**: 템플릿 렌더링, 외부 서비스 호출 등 CPU/I/O 부하 집중
- **에러 처리 미비**: 재시도·백오프·DLQ(Dead Letter Queue) 부족
- **채널 간 장애 전파**: SMS 장애가 푸시 알림까지 영향을 줄 수 있음

---

### 10.7 개선된 아키텍처

#### 10.7.1 알림 서버 수평 확장
- 다수의 Notification Server를 구성
- 로드밸런서 뒤에서 병렬 처리
- 서버는 **무상태(stateless)** 로 설계

#### 10.7.2 메시지 큐 분리
- 채널별 큐 구성: iOS 큐, Android 큐, SMS 큐, Email 큐
- 한 채널이 장애가 나도 다른 채널에 영향 없음

#### 10.7.3 Worker(작업 서버) 분리
- 채널별 Worker 프로세스 구성
- 수요 증가 시 특정 채널만 개별 확장 가능 (예: 프로모션 시 이메일만 증가)

#### 10.7.4 캐시/DB 분리
- user/device 정보는 캐시(예: Redis)에서 조회
- DB 부하는 최소화, DB는 source of truth 역할

#### 10.7.5 재시도 큐 추가
- 전송 실패 이벤트는 재시도 큐로 이동
- 지수 백오프(Exponential Backoff) 적용
- 지정 횟수 초과 시 실패 상태로 마킹 후 알림 로그에 기록

---

### 10.8 안정성 고려사항

#### 10.8.1 데이터 손실 방지
- 알림 로그(notification log) DB 유지
- 큐에 적재된 이벤트는 **영속 스토리지 기반 메시지 큐** 사용 (예: Kafka)

#### 10.8.2 중복 전송 방지
- 이벤트 ID 기반 중복 체크
- 멱등성 보장(idempotency key) 도입

#### 10.8.3 Rate Limiting
- 특정 사용자에게 너무 많은 알림을 전송하지 않도록 제한
- 글로벌/사용자/채널별 Rate Limit 설정

---

### 10.9 추가 컴포넌트 및 고려사항

#### 10.9.1 알림 템플릿 시스템
- 파라미터 기반 템플릿으로 알림 생성 자동화

예시:

```
여러분이 꿈꾸던 상품이 다시 입고되었습니다! [item_name]  
지금 예약하세요! [date]까지 가능!
```

#### 10.9.2 사용자 설정(User Preferences)
- 채널별 opt-in / opt-out 관리

예시 스키마:

```
user_id  | channel | opt_in
```

#### 10.9.3 보안(Security)
- `appKey + appSecret` 기반 인증
- 인증된 서비스만 알림 전송 가능
- 감사 로그(audit log) 기록

#### 10.9.4 모니터링 & 추적
- 큐 길이(queue depth)
- open/click 이벤트
- 전송 성공률 및 오류 비율
- 채널별 지연 시간

---

### 10.10 수정된 최종 설계 요약

최종 아키텍처에는 다음 기능이 포함됩니다.

- 인증(Authentication)
- Rate Limiting
- 전송 실패 시 재시도 + DLQ
- 템플릿 시스템
- 알림 로그 DB
- 분석/추적 서비스 연동
- 채널별 큐 및 Worker 구조
- 캐시/DB 분리

전체 알림 시스템이 분산 환경에서 **안정적이고 확장 가능하게** 동작하도록 구성됩니다.

---

### 10.11 정리

이번 장에서는 확장성과 안정성이 높은 알림 시스템을 설계하기 위해 고려해야 할 요소들을 정리했습니다.

- 다양한 알림 전송 채널 통합
- 대규모 트래픽을 처리할 수 있는 분산 아키텍처
- 메시지 큐 기반의 비동기 처리
- 인증·재시도·중복 방지·전송 제한 기능
- 템플릿 및 사용자 설정 관리
- 이벤트 추적 및 실시간 모니터링

이 내용을 기반으로 대규모 서비스에서 안정적이고 신뢰성 있는 알림 시스템을 구현할 수 있습니다.
