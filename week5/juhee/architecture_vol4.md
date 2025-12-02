# 📘 Chapter 9 — 웹 크롤러 설계 (Web Crawler Design)

System Design Interview 책 9장을 기반으로 작성된 **웹 크롤러(Web Crawler) 설계 요약 문서**입니다.
크롤러가 갖춰야 하는 요구사항, URL Frontier 설계, 성능·확장성·안정성에 대해 설명합니다.

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
