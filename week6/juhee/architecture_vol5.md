# Chapter 11. News Feed System Design

## 1. Problem Overview

뉴스 피드는 사용자의 홈 화면에 지속적으로 업데이트되는 콘텐츠 목록이다.

### 포함되는 콘텐츠
- 사용자 상태 업데이트
- 사진 / 비디오
- 링크
- 앱 활동
- 친구, 페이지, 그룹의 게시물

대표 예시:
- Facebook News Feed
- Instagram Feed
- Twitter Timeline

---

## 2. Requirements Clarification

### Functional Requirements
- 사용자는 게시글을 작성할 수 있어야 한다
- 사용자는 친구들의 게시글을 볼 수 있어야 한다
- 뉴스 피드는 최신순으로 정렬된다
- 이미지 및 비디오를 지원한다

### Non-Functional Requirements
- 모바일 앱 + 웹 지원
- 낮은 지연 시간 (low latency)
- 높은 가용성
- 확장 가능성

### Assumptions
- 최대 친구 수: 5,000명
- DAU: 10M
- 피드 정렬: reverse chronological order

---

## 3. High-Level Design

시스템은 크게 두 부분으로 나뉜다.

1. Feed Publishing (쓰기)
2. Feed Reading (읽기)

### APIs

#### Post Feed
```

POST /v1/me/feed

```

#### Read Feed
```

GET /v1/me/feed

```

---

## 4. Feed Publishing Flow

### Overall Flow
1. 사용자가 게시글 작성
2. 로드 밸런서가 요청 분산
3. 웹 서버가 인증 및 처리율 제한 수행
4. 게시글 저장 서비스가 DB + 캐시에 저장
5. Fanout 서비스가 친구들의 피드 갱신
6. 알림 서비스가 푸시 알림 전송

### Web Server Responsibilities
- Authentication
- Rate Limiting
- Request Validation

---

## 5. Fanout Service (Core Component)

Fanout은 한 사용자의 게시글을  
해당 사용자와 연결된 모든 사용자에게 전달하는 과정이다.

### Fanout Flow
1. 그래프 DB에서 친구 ID 목록 조회
2. 사용자 설정 기반 필터링 (mute, privacy)
3. (friend_id, post_id)를 메시지 큐에 적재
4. Worker 서버가 큐를 소비
5. 뉴스 피드 캐시 갱신

---

## 6. Fanout Models

### 6.1 Fanout-on-Write (Push Model)

#### Description
- 게시글 작성 시점에 친구들의 피드를 미리 갱신

#### Pros
- 피드 읽기 속도가 매우 빠름
- 실시간에 가까운 업데이트

#### Cons
- 친구 수가 많을 경우 비용 증가
- 비활성 사용자에게도 작업 수행
- Hotkey 문제 발생 가능

---

### 6.2 Fanout-on-Read (Pull Model)

#### Description
- 사용자가 피드를 읽을 때 필요한 게시글을 조회

#### Pros
- 비활성 사용자에 대한 비용 없음
- Hotkey 문제 없음

#### Cons
- 피드 로딩이 느릴 수 있음
- 계산 비용이 큼

---

### 6.3 Hybrid Approach

- 일반 사용자 → Fanout-on-write
- 팔로워 수가 매우 많은 사용자(셀럽) → Fanout-on-read
- Consistent Hashing을 사용해 부하 분산

---

## 7. Feed Reading Flow

### Reading Steps
1. 클라이언트가 GET /v1/me/feed 호출
2. 로드 밸런서가 웹 서버로 전달
3. 웹 서버가 뉴스 피드 서비스 호출
4. 뉴스 피드 캐시에서 post_id 목록 조회
5. 각 post_id에 대해:
   - 사용자 정보 → User Cache
   - 포스트 내용 → Post Cache
   - 미디어 → CDN
6. JSON 응답 반환
7. 클라이언트 렌더링

---

## 8. Cache Design

뉴스 피드 시스템의 성능 핵심은 캐시 설계이다.

### Cache Layers

1. **News Feed Cache**
   - user_id → post_id list

2. **Content Cache**
   - 포스트 데이터
   - 인기 콘텐츠 별도 관리

3. **Social Graph Cache**
   - 친구 / 팔로워 / 팔로잉 관계

4. **Action Cache**
   - 좋아요, 댓글, 공유

5. **Counter Cache**
   - 좋아요 수, 댓글 수, 팔로워 수

### Cache Strategy
- 캐시는 ID 위주로 저장
- 최신 N개만 유지
- 대부분의 읽기는 캐시 히트

---

## 9. Key Trade-offs

| Topic | Choice |
|------|------|
| Read vs Write | Read optimized |
| Fanout | Hybrid |
| Storage | Cache-first |
| Media | CDN |
| Scalability | Message Queue + Worker |

---

## 10. Summary (Interview Answer)

> 뉴스 피드는 읽기 성능이 가장 중요하기 때문에  
> 대부분의 사용자는 fanout-on-write를 사용하고,  
> 팔로워가 많은 사용자는 fanout-on-read를 적용하는  
> 하이브리드 구조를 선택한다.  
> 캐시는 역할별로 분리하여 확장성과 성능을 확보한다.

---

# Chapter 12. Chat System Design

## 1. Problem Overview

채팅 시스템은 대부분의 사용자에게 익숙하지만,  
요구사항에 따라 전혀 다른 시스템이 될 수 있다.

대표적인 예:
- WhatsApp, Facebook Messenger (1:1 중심)
- Slack (업무용 그룹 채팅)
- Discord (대규모 커뮤니티 + 실시간성)

따라서 설계 전에 **정확한 범위 정의**가 필수이다.

---

## 2. Requirements Clarification

### Functional Requirements
- 1:1 채팅
- 그룹 채팅 (최대 100명)
- 낮은 지연 시간 (low latency)
- 사용자 접속 상태 표시 (online / offline)
- 여러 단말 동시 접속
- 푸시 알림
- 텍스트 메시지 지원

### Non-Functional Requirements
- 모바일 앱 + 웹 지원
- DAU 50M 처리
- 높은 가용성
- 수평 확장 가능

### Out of Scope
- 음성 / 영상 채팅
- 파일 전송
- 종단 간 암호화(E2EE)

---

## 3. High-Level Design

### 기본 원칙
- 클라이언트끼리는 직접 통신하지 않는다
- 모든 메시지는 채팅 서버를 통해 전달된다

### 채팅 서버의 역할
- 메시지 수신
- 수신자 결정
- 메시지 전달
- 오프라인 메시지 저장

---

## 4. Communication Protocols

### HTTP
- 로그인 / 회원가입
- 사용자 프로필
- 설정 관리
- 그룹 관리

### WebSocket
- 실시간 메시지 송수신
- 서버 → 클라이언트 비동기 메시지 전송

실시간성이 필요한 부분만 WebSocket 사용

---

## 5. Polling vs Long Polling vs WebSocket

### Polling
- 주기적으로 서버에 새 메시지 요청
- 비효율적

### Long Polling
- 메시지가 올 때까지 연결 유지
- 폴링보다는 낫지만 확장성 문제 존재

### WebSocket (선택)
- 항구적 연결
- 양방향 통신
- 대규모 채팅 시스템에 가장 적합

---

## 6. Overall Architecture

채팅 시스템은 세 부분으로 나뉜다.

1. **무상태 서비스 (Stateless Services)**
   - HTTP 기반
   - 로그인, 회원가입, 프로필, 그룹 관리
   - 로드밸런서 뒤에 위치

2. **상태 유지 서비스 (Stateful Services)**
   - 채팅 서버
   - WebSocket 연결 유지

3. **제3자 서비스 연동**
   - 푸시 알림 서비스

---

## 7. Storage Design

### 데이터 유형

#### 1) 일반 사용자 데이터
- 사용자 프로필
- 설정
- 친구 목록

관계형 데이터베이스 사용

#### 2) 채팅 이력 데이터
- 데이터 양이 매우 큼
- 최근 메시지 위주 접근
- 읽기/쓰기 비율 ≈ 1:1

키-값 저장소(NoSQL) 선택

### 선택 이유
- 수평 확장 용이
- 낮은 지연 시간
- 대규모 데이터 처리 적합

실제 사례:
- Facebook Messenger → HBase
- Discord → Cassandra

---

## 8. Data Model

### 1:1 채팅 메시지
- Primary Key: message_id
- message_id로 메시지 순서 보장

### 그룹 채팅 메시지
- Composite Key: (channel_id, message_id)
- channel_id = partition key

---

## 9. Message ID Design

### 요구 조건
- 전역 유일성
- 시간 순서 보장

### 선택
- 채널(또는 1:1 세션) 단위 지역 순서 번호 생성기
- 전역 순서 대신 로컬 순서 보장

---

## 10. Service Discovery

### 목적
- 사용자에게 최적의 채팅 서버 선택

### 기준
- 사용자 위치
- 서버 부하

### 구현 예
- Apache ZooKeeper

---

## 11. 1:1 Chat Message Flow

1. 사용자 A → 채팅 서버 1로 메시지 전송
2. 메시지 ID 생성
3. 메시지 동기화 큐로 전달
4. 키-값 저장소에 저장
5. 사용자 B 상태 확인
   - online → 채팅 서버로 전달
   - offline → 푸시 알림
6. WebSocket으로 사용자 B에게 전달

---

## 12. Multi-Device Synchronization

각 단말은 다음 값을 유지한다.

```

cur_max_message_id

```

새 메시지 판단 조건:
- 수신자 ID가 본인
- message_id > cur_max_message_id

단말별로 독립적인 동기화 가능

---

## 13. Group Chat Message Flow (Small Group)

### 소규모 그룹 전략
- 메시지를 수신자별 큐로 복사
- 구현 단순
- 그룹 크기가 작을 때 효율적

제한:
- 그룹이 커지면 fan-out 비용 증가

---

## 14. Presence (Online / Offline Status)

### Presence Server
- WebSocket 기반
- 사용자 상태 관리
- 키-값 저장소 사용

### Login
- WebSocket 연결 시 online 표시

### Logout
- 상태를 offline으로 변경

---

## 15. Heartbeat Mechanism

### 문제
- 일시적인 네트워크 끊김

### 해결
- 클라이언트가 주기적으로 heartbeat 전송
- 일정 시간 동안 미수신 시 offline 처리

예:
- heartbeat 주기: 5초
- timeout: 30초

---

## 16. Presence Update Propagation

### Publish–Subscribe 모델
- 사용자 상태 변경 시
- 친구 관계 채널에 이벤트 발행

### 한계
- 대규모 그룹에서는 비효율적

### 실무적 절충
- 그룹 입장 시 상태 조회
- 수동 갱신 방식

---

## 17. Trade-offs Summary

| Topic | Choice |
|---|---|
| Protocol | WebSocket + HTTP |
| Storage | Key-Value Store |
| Message Order | Channel-local |
| Scalability | Horizontal |
| Presence | Heartbeat + Pub/Sub |

---

## 18. Interview Summary

> 채팅 시스템은 실시간성이 핵심이므로  
> 메시지 송수신에는 WebSocket을 사용하고,  
> 나머지 기능은 HTTP 기반 무상태 서비스로 분리한다.  
> 메시지는 먼저 저장한 뒤 전달하여 안정성을 확보하며,  
> 접속 상태는 heartbeat 기반 presence 서버로 관리한다.

