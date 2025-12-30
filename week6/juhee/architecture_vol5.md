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
```
