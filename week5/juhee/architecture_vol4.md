# Chapter 9 - 웹 크롤러 설계 (Web Crawler Design)

System Design Interview 책 9장을 기반으로 작성된 **웹 크롤러(Web Crawler) 설계 요약 문서**이다.
크롤러가 갖춰야 하는 요구사항, URL Frontier 설계, 성능·확장성·안정성에 대해 설명한다.

---

## 1. 문제 정의

웹 크롤러는 다음과 같은 기능을 수행하는 시스템입니다.

1. 주어진 URL을 다운로드한다
2. HTML에서 새로운 URL을 추출한다
3. 추출한 URL을 다시 Frontier에 넣어 재크롤링
4. 중복·함정·스팸 등 문제 콘텐츠는 차단
5. 웹 서버에 과도한 부하를 주지 않도록 예의 있게 동작해야 함

---

## 2. 크롤러의 핵심 속성

### 규모 확장성 (Scalability)

* 월 수십억 페이지 처리
* 분산 크롤링 필수

### 예의성 (Politeness)

* 동일 도메인에 과도한 요청 금지
* 도메인별 접근 간격 관리

### 확장성 (Extensibility)

* PNG, PDF 등 새로운 콘텐츠 처리 모듈을 쉽게 추가

### 안정성 (Robustness)

* 비정상 HTML, 네트워크 오류 등에도 시스템이 죽지 않아야 함

---

## 3. 규모 추정

* 월 크롤링 목표: **10 billion pages**
* 평균 페이지: **500 KB**
* 월 저장량: **500 TB**
* 5년 보관: **약 30 PB**
* QPS: **약 400**, 피크 시 **800**

---

## 4. 전체 아키텍처

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

---

## 5. URL Frontier 설계가 핵심이다

웹 크롤러의 성능과 안정성의 중심은 **URL Frontier**이다.

### 5.1 Front Queue (우선순위 기반)

URL 우선순위를 점수화하여 큐를 여러 개(f1, f2, … fn)로 분리.

우선순위 기준:

* PageRank
* 트래픽
* 업데이트 빈도
* 신선도(freshness)
* 콘텐츠 중요도

**Front Queue Selector**가 큐를 순회하며 높은 우선순위 큐에서 더 자주 URL을 꺼냄.

---

### 5.2 Back Queue (예의성 기반)

같은 도메인의 URL은 반드시 동일 큐로 간다.

* b1 = example.com
* b2 = amazon.com
* b3 = wikipedia.org
* …

FIFO 방식
→ 동일 도메인 요청을 시간 간격을 두고 순차적 처리

**Back Queue Selector**가 큐를 순차적으로 선택해 서버 부하를 방지한다.

---

### 5.3 Front + Back Queue 결합 구조 (그림 9-8 개념)

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

* 중요도 높은 URL은 먼저 처리
* 같은 도메인에는 예의 있게 요청
* 분산 크롤링 구조까지 더하면 대규모 환경에서도 안정적

---

## 6. 성능 최적화 기법

### 6.1 분산 크롤링(Distributed Crawling)

URL 공간을 여러 서버로 분할하고 병렬 처리.

### 6.2 DNS 최적화

* DNS 캐싱
* 비동기 DNS 요청 사용

### 6.3 HTTP 연결 재사용

* Keep-alive
* Connection pool
* Timeouts 적절히 설정

### 6.4 robots.txt 준수

* Disallow 경로 차단
* 규칙 변경에 대비해 주기적으로 다운로드

---

## 7. 확장성 (플러그인 아키텍처)

HTML 외에도 PNG, PDF, JSON 등 새로운 파일 형식은
**확장 모듈(plug-in module)** 로 쉽게 추가하도록 설계한다.

예: PNG 다운로드 추가

```
HTML Downloader  → HTML 처리  
PNG Downloader   → PNG 파일 처리  
PDF Downloader   → PDF 파일 처리  
```

웹 모니터(Web Monitor) 모듈은 저작권·상표권 위반 콘텐츠를 감지할 수 있음.

---

## 8. 문제 콘텐츠 감지 및 회피

웹에는 크롤러를 멈추게 하거나 품질을 떨어뜨리는 콘텐츠가 많다.

### 중복 콘텐츠

* 해시 체크섬(MD5/SHA-1) 사용
* 웹의 30%가 중복 콘텐츠

### 거미덫(Spider Trap)

고의적으로 무한히 깊어지는 URL 구조 예:

```
/foo/bar/foo/bar/foo/bar/...
```

대응 방법:

* URL 최대 길이 제한
* 패턴 기반 필터
* Trap 사이트 도메인 블랙리스트

### 데이터 노이즈

* 광고 스크립트
* 세션 기반 URL
* 스팸 URL
* 자동 생성된 의미 없는 페이지

→ 필터링하여 시스템 리소스 절약

---

## 9. 추가 고려 사항

### 서버 측 렌더링 / 동적 렌더링

JavaScript로 링크가 동적 생성되는 경우 → headless browser 필요

### 스팸 페이지 필터링

Anti-spam 컴포넌트 사용

### 데이터 저장 확장성

* 샤딩(sharding)
* 복제(replication)

### 수평적 확장 (Horizontal Scalability)

Stateless 서버 구조를 통해 수백~수천 노드로 확장

### 가용성·일관성·안정성(ACR)

대규모 시스템의 필수 설계 요소

### 분석 시스템(Analytics)

크롤링된 데이터 기반으로 시스템 튜닝이 가능

---

## 10. 마무리

웹 크롤러 설계는 System Design의 전형적인 종합 문제입니다.

* 확장성
* 예의성
* 안정성
* 효율성
* 데이터 처리
* 분산 시스템
* 장애 대응

모든 개념이 통합적으로 반영되어야 하며,
검색 엔진 수준의 크롤러는 수백 개의 컴포넌트와 수천 대의 서버가 필요할 수 있습니다.

---

# Chapter 10 - 알림 시스템 설계 (Notification System Design)

알림 시스템은 사용자에게 중요한 정보를 다양한 채널(푸시 알림, SMS, 이메일)로 전달하는 핵심 인프라이다.  
대규모 트래픽 환경에서 안정적이며 확장 가능한 알림 시스템을 설계하기 위해 필요한 구성요소, 전송 절차, 아키텍처 개선안 등을 정리한다.

---

## 1. 문제 이해 및 설계 범위 규정

### 지원해야 할 알림 종류
- 푸시 알림 (iOS, Android)
- SMS 메시지
- 이메일

### 시스템 요구사항
- 연성 실시간(Soft real-time) 시스템  
  → 즉시성은 중요하지만 약간의 지연은 허용
- 다양한 단말 지원  
  → iOS, Android, Web/Desktop
- 사용자 설정 지원  
  → opt-in / opt-out 지원
- 대량 발송 가능  
  → 초당 수천~수만 건, 하루 수백만 건 이상의 알림 처리 가능

---

## 2. 알림 유형별 전송 절차

### 2.1 iOS 푸시 알림 (APNS)
필요 구성요소:
- **Provider(알림 서버)**: 알림 데이터를 생성하여 APNS로 전송
- **Device Token**: 특정 단말을 식별하는 고유 키
- **Payload(JSON)**: 전송할 알림 내용

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
````

iOS 푸시는 APNS가 큐잉, 재시도, 전송을 모두 관리하므로 서버는 인증된 요청만 만들면 된다.

---

### 2.2 Android 푸시 알림 (FCM)

APNS와 동일 구조이나 전송 서비스만 FCM으로 변경됨.

구조:

```
Provider → FCM → Android Device
```

---

### 2.3 SMS 메시지

Twilio, Nexmo 같은 외부 API 서비스를 사용하여 전송.

특징:

* 전송 성공률 높은 편
* 건당 과금
* 비즈니스 알림에 널리 사용

---

### 2.4 이메일

SendGrid, Mailchimp 등 외부 이메일 서비스를 사용하는 경우가 많다.

장점:

* 전송 성공률 높음
* 스팸 필터링 기술 우수
* Open/Click 등 Analytics 제공

---

## 3. 전체 알림 채널 통합 구조

여러 알림 채널을 하나의 Notification System이 통합 관리한다.

```
          ┌── APNS → iOS
          ├── FCM → Android
Provider ─┤
          ├── SMS 서비스 → Phone
          └── Email 서비스 → Mail Client
```

---

## 4. 연락처 정보 수집 및 저장

알림을 보내기 위해 필요한 정보:

* Device Token (푸시)
* Phone Number (SMS)
* Email Address (이메일)

절차:

1. 앱 설치 또는 계정 생성 시 서버로 정보 전송
2. API 서버가 DB에 저장

DB 구조 예시:

**user 테이블**

| 필드           | 타입      | 설명   |
| ------------ | ------- | ---- |
| user_id      | bigint  | PK   |
| email        | varchar | 이메일  |
| phone_number | varchar | 전화번호 |
| …            |         |      |

**device 테이블**

| 필드                | 타입        | 설명    |
| ----------------- | --------- | ----- |
| id                | bigint    | PK    |
| device_token      | varchar   | 단말 토큰 |
| user_id           | bigint    | FK    |
| last_logged_in_at | timestamp |       |

---

## 5. 알림 전송 절차(초안)

1. 서비스가 Notification Server API 호출
2. 알림 서버가 메타데이터(DB/Cache) 조회
3. 알림 유형에 맞는 이벤트 생성
4. 이벤트를 메시지 큐에 저장
5. Worker(작업 서버)가 큐에서 이벤트를 소비
6. 제3자 서비스(APNS/FCM/SMS/Email 서비스)로 전송
7. 사용자에게 도착

---

## 6. 초안 설계 문제점

* **SPOF(단일 장애점)**
  알림 서버가 하나면 장애 시 전체 서비스 중단
* **확장성 부족**
  모든 알림 채널을 한 서버가 처리
* **성능 병목**
  HTML 템플릿 처리·전송 등 CPU 부하 큼
* **다양한 에러 처리 미비**
  재시도 로직 부족
* **채널 간 장애 전파 위험**
  SMS 장애가 푸시 알림까지 영향을 줄 수 있음

---

## 7. 개선된 아키텍처

개선 핵심:

### 7.1 알림 서버 수평 확장

* 다수의 Notification Server 구성
* 로드밸런서 뒤에서 병렬 처리

### 7.2 메시지 큐 분리

* 채널별 큐(iOS 큐, Android 큐, SMS 큐, Email 큐)
* 한 채널 장애가 다른 채널에 영향을 주지 않음

### 7.3 Worker(작업 서버) 분리

* 채널별 Worker 구성
* 수요 증가 시 개별 확장 가능

### 7.4 캐시/DB 분리

* user/device 정보는 캐시 조회
* DB 부하는 최소화

### 7.5 재시도 큐 추가

* 전송 실패 이벤트는 재시도 큐로 이동
* 지정 횟수만큼 반복 후 실패 처리

---

## 8. 안정성 고려사항

### 8.1 데이터 손실 방지

* 알림 로그(notification log) DB 유지
* 재시도 메커니즘 필수

### 8.2 중복 전송 방지

* 이벤트 ID 기반 중복 체크
* 100% 방지는 어려우나 빈도 최소화 가능

### 8.3 Rate Limiting

* 특정 사용자에게 너무 많은 알림을 전송하지 않도록 제한

---

## 9. 추가 컴포넌트 및 고려사항

### 9.1 알림 템플릿

* 파라미터 기반 템플릿으로 알림 생성 자동화
* 예시:

```
여러분이 꿈꾸던 상품이 다시 입고되었습니다! [item_name]  
지금 예약하세요! [date]까지 가능!
```

### 9.2 사용자 설정(User Preferences)

* 채널별 opt-in / opt-out
* DB 스키마 예:

```
user_id  | channel | opt_in
```

### 9.3 보안(Security)

* appKey + appSecret 인증 필요
* 인증된 서비스만 전송 가능

### 9.4 모니터링 & 추적

* 큐 길이(queue depth)
* open/click 이벤트
* 전송 성공률 및 오류 비율

---

## 10. 수정된 최종 설계안

최종 아키텍처에는 다음 기능 포함됨:

* 인증(Authentication)
* Rate Limiting
* 전송 실패 시 재시도
* 템플릿 시스템
* 알림 로그 DB
* 분석/추적 서비스 연동
* 채널별 큐 및 Worker 구조
* 캐시/DB 분리

전체 알림 시스템이 분산 환경에서 안정적으로 동작하도록 구성됨.

---

## 11. 마무리

이번 장에서는 확장성과 안정성이 높은 알림 시스템을 설계하기 위해 고려해야 할 기술 요소들을 살펴보았다.

핵심은 다음과 같다:

* 다양한 알림 전송 채널 통합
* 대규모 트래픽을 처리할 수 있는 분산 아키텍처
* 메시지 큐 기반의 비동기 처리
* 인증·재시도·중복 방지·전송 제한 기능
* 템플릿 및 사용자 설정 관리
* 이벤트 추적 및 실시간 모니터링

이 장의 내용을 기반으로 대규모 서비스에서 안정적이고 신뢰성 있는 알림 시스템을 구현할 수 있다.

```

