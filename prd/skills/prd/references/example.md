# PRD 예시: CafeMasters

> 이 예시는 PRD 작성 시 각 섹션의 깊이, 톤, 구체성 수준을 보여줍니다.
> 실제 PRD 작성 시 이 예시와 동일한 수준으로 작성하세요.

## 1. 서비스 개요

| 구성               | 설명                                                                                           |
| ------------------ | ---------------------------------------------------------------------------------------------- |
| **서비스 이름**    | CafeMasters                                                                                    |
| **서비스 타입**    | 카페 탐색 및 컬렉션 관리 웹 애플리케이션                                                       |
| **핵심 가치 제안** | 전국 로스터리 카페를 원두 원산지·로스팅 프로필 기준으로 탐색하고, 나만의 컬렉션으로 큐레이션한다 |
| **타겟 사용자**    | 스페셜티 커피에 관심 있는 2030 직장인·대학생                                                    |
| **예상 사용자 규모** | 초기 MAU 500 → 6개월 후 MAU 3,000                                                            |

## 2. 비즈니스 목표

> 페르소나 리서치 결과: 커피 애호가들이 "어떤 원두를 쓰는지 미리 알 수 없어 방문 후 실망"하는 문제를 반복적으로 경험

### 2.1 성공지표 (KPI)

| 카테고리 | 지표                    | 목표값            | 측정 방법                           |
| -------- | ----------------------- | ----------------- | ----------------------------------- |
| 사용자   | MAU (월간 활성 사용자)  | 3개월 내 1,000명  | Supabase Auth 월간 로그인 수        |
| 사용자   | 컬렉션 생성률           | 가입자의 40% 이상 | 컬렉션 1개 이상 보유 사용자 비율    |
| 기능     | 검색→상세 전환율        | 30% 이상          | 검색 결과 클릭 수 / 검색 수         |
| 기술     | Lighthouse Performance  | 90점 이상         | Lighthouse CI 자동 측정             |

## 3. 아키텍처

### 3.1 기술 스택

| 구성                          | 기술                           | 선택 사유                                              |
| ----------------------------- | ------------------------------ | ------------------------------------------------------ |
| **Frontend**                  | Next.js 16 (App Router)       | RSC 기반 SSR, SEO 최적화, Streaming 지원               |
| **Styling**                   | TailwindCSS 4                  | 유틸리티 기반 빠른 UI 개발, 디자인 토큰 일관성         |
| **State Management**          | TanStack Query v5              | 서버 상태 캐싱, 자동 갱신, Optimistic Update 지원      |
| **Database & Authentication** | Supabase (PostgreSQL + Auth)   | RLS 기반 보안, 실시간 구독, OAuth 내장                 |
| **Map**                       | Kakao Maps SDK                 | 국내 지도 정확도, POI 데이터 풍부                      |
| **Deployment**                | Vercel                         | Next.js 네이티브 지원, Edge Function, Preview Deploy   |

### 3.2 디렉토리 구조 (FSD)

```
src/
├── app/            # Next.js App Router (전역 설정, 라우팅)
│   ├── (auth)/     # 인증 관련 라우트 그룹
│   ├── cafe/       # 카페 상세 페이지
│   └── collection/ # 컬렉션 페이지
├── widgets/        # 독립적 UI 블록 (CafeCard, MapView)
├── features/       # 사용자 기능 (검색, 북마크, 컬렉션)
├── entities/       # 비즈니스 엔티티 (cafe, user, collection)
└── shared/         # 공통 (ui, api, lib, types)
```

### 3.3 Data Flow Architecture

```
[사용자 검색] → [Server Component] → [Supabase RPC] → [PostgreSQL Full-Text Search]
                                                      ↓
[검색 결과 렌더링] ← [RSC Streaming] ← [검색 결과 + 카페 메타데이터]

[북마크 토글] → [Server Action] → [Supabase Insert/Delete] → [RLS 검증]
                                                              ↓
[UI Optimistic Update] ← [TanStack Query Invalidation] ← [성공/실패 응답]
```

## 4. 비즈니스 로직 & 제약조건

### 4.1 비즈니스 규칙

| 규칙 ID | 규칙 설명                                     | 적용 범위     | 예외 조건           |
| ------- | --------------------------------------------- | ------------- | ------------------- |
| BR-001  | 북마크는 로그인 사용자만 가능                  | 북마크 기능   | 없음                |
| BR-002  | 컬렉션당 카페는 최대 50개                      | 컬렉션 기능   | 프리미엄 사용자 100 |
| BR-003  | 카페 정보 수정 요청은 관리자 승인 후 반영      | 카페 데이터   | 없음                |
| BR-004  | 검색 결과는 거리순 > 평점순 > 최신순 기본 정렬 | 검색 기능     | 사용자 정렬 변경 시 |

### 4.2 기술적 제약조건

- Kakao Maps API 일일 호출 제한: 300,000회 (무료 tier)
- Supabase Free tier: DB 500MB, Auth 50,000 MAU, Storage 1GB
- Vercel Hobby: Serverless Function 실행 시간 10초 제한

### 4.3 비즈니스 제약조건

- 1인 개발 → MVP 기능에 집중, 관리자 기능 최소화
- 카페 데이터 초기 수집은 수동 + 크롤링 병행

## 5. 핵심 기능 상세

### 5.1 카페 검색 & 필터링

- **우선순위**: Must-have
- **설명**: 원두 원산지, 로스팅 프로필, 지역 기반으로 카페를 검색하고 필터링한다
- **사용자 시나리오**:
  1. 사용자가 검색창에 "에티오피아 예가체프"를 입력한다
  2. 시스템이 해당 원두를 취급하는 카페 목록을 거리순으로 표시한다
  3. 사용자가 "라이트 로스팅" 필터를 추가하면 결과가 좁혀진다
  4. 각 카페 카드에는 대표 이미지, 이름, 거리, 대표 원두가 표시된다
- **입력**: 검색어(텍스트), 필터(원산지, 로스팅, 지역), 사용자 위치(선택)
- **출력**: 카페 목록 (카드 형태, 페이지네이션 20개 단위)
- **엣지 케이스**:
  - 검색 결과 0건: "조건에 맞는 카페가 없습니다" + 필터 완화 제안
  - 위치 권한 거부: 기본 지역(서울) 기준으로 결과 표시
  - 네트워크 오류: 캐시된 이전 결과 표시 + 재시도 버튼
- **수용 기준 (Acceptance Criteria)**:
  - [ ] 검색어 입력 후 300ms 디바운스 적용, 결과 1초 이내 표시
  - [ ] 필터 복수 선택 시 AND 조건으로 결과 좁혀짐
  - [ ] 검색 결과 0건 시 빈 상태 UI 표시 + 필터 초기화 버튼 노출
  - [ ] 무한 스크롤로 추가 결과 로딩 (20개 단위)

### 5.2 카페 북마크

- **우선순위**: Must-have
- **설명**: 관심 카페를 북마크하여 나중에 빠르게 접근한다
- **사용자 시나리오**:
  1. 사용자가 카페 카드의 하트 아이콘을 탭한다
  2. 하트가 즉시 채워지고 (Optimistic Update), 서버에 저장 요청이 전송된다
  3. 성공 시 상태 유지, 실패 시 하트가 원복되며 토스트 알림 표시
- **입력**: cafe_id, user_id (세션에서 자동 추출)
- **출력**: 북마크 상태 토글 (boolean)
- **엣지 케이스**:
  - 비로그인 사용자: 로그인 모달 표시
  - 네트워크 오류: Optimistic Update 롤백 + "다시 시도해주세요" 토스트
  - 중복 요청 방지: 토글 중 추가 클릭 무시 (debounce)
- **수용 기준 (Acceptance Criteria)**:
  - [ ] 비로그인 시 북마크 클릭하면 로그인 모달 표시
  - [ ] Optimistic Update로 UI 즉시 반영, 실패 시 롤백
  - [ ] 내 북마크 목록에서 확인 가능

### 5.3 나만의 컬렉션 관리

- **우선순위**: Should-have
- **설명**: 북마크한 카페를 주제별 컬렉션으로 분류하여 관리한다
- **사용자 시나리오**:
  1. 사용자가 "컬렉션 만들기" 버튼을 탭한다
  2. 컬렉션 이름("주말 드라이브 코스")과 설명을 입력한다
  3. 북마크 목록에서 카페를 선택하여 컬렉션에 추가한다
  4. 컬렉션 페이지에서 카페 목록을 지도 위에 핀으로 확인한다
- **입력**: 컬렉션명, 설명, 카페 ID 목록
- **출력**: 컬렉션 객체 (카페 목록 포함)
- **엣지 케이스**:
  - 컬렉션당 카페 50개 초과: "최대 50개까지 추가할 수 있습니다" 알림
  - 컬렉션명 중복: 허용 (ID 기반 관리)
  - 빈 컬렉션: 빈 상태 UI + "카페 추가하기" CTA
- **수용 기준 (Acceptance Criteria)**:
  - [ ] 컬렉션 CRUD 정상 동작
  - [ ] 컬렉션당 카페 수 제한 (50개) 적용
  - [ ] 컬렉션 내 카페를 지도에서 확인 가능
  - [ ] 컬렉션 공유 링크 생성 가능

## 6. 유저 접근 가능 페이지

| 경로                    | 페이지명       | 설명                          | 로그인 필요 | 주요 기능                        |
| ----------------------- | -------------- | ----------------------------- | ----------- | -------------------------------- |
| `/`                     | 홈             | 추천 카페, 인기 컬렉션        | X           | 카페 카드 목록, 검색 진입        |
| `/search`               | 검색           | 카페 검색 및 필터링           | X           | 텍스트 검색, 필터, 지도 뷰      |
| `/cafe/[id]`            | 카페 상세      | 카페 정보, 원두, 위치         | X           | 상세 정보, 북마크, 지도          |
| `/collection`           | 내 컬렉션      | 사용자의 컬렉션 목록          | O           | 컬렉션 CRUD                     |
| `/collection/[id]`      | 컬렉션 상세    | 컬렉션 내 카페 목록           | O           | 카페 목록, 지도 뷰, 공유        |
| `/bookmark`             | 내 북마크      | 북마크한 카페 목록            | O           | 북마크 목록, 컬렉션에 추가      |
| `/login`                | 로그인         | OAuth 소셜 로그인             | X           | Google, Kakao 로그인             |
| `/profile`              | 프로필         | 사용자 정보 관리              | O           | 닉네임 수정, 로그아웃           |

## 7. Database 스키마

### 7.1 핵심 테이블

**cafe**

| 컬럼          | 타입         | 제약조건                  | 설명            |
| ------------- | ------------ | ------------------------- | --------------- |
| id            | uuid         | PK, DEFAULT uuid_v4()    | 카페 고유 ID    |
| name          | varchar(100) | NOT NULL                  | 카페명          |
| address       | text         | NOT NULL                  | 주소            |
| lat           | decimal(9,6) | NOT NULL                  | 위도            |
| lng           | decimal(9,6) | NOT NULL                  | 경도            |
| beans         | jsonb        |                           | 취급 원두 정보  |
| image_url     | text         |                           | 대표 이미지 URL |
| created_at    | timestamptz  | DEFAULT now()             | 생성일          |

**bookmark**

| 컬럼       | 타입        | 제약조건                             | 설명         |
| ---------- | ----------- | ------------------------------------ | ------------ |
| id         | bigint      | PK, GENERATED ALWAYS AS IDENTITY    | 북마크 ID    |
| user_id    | uuid        | FK → auth.users(id), NOT NULL       | 사용자 ID    |
| cafe_id    | uuid        | FK → cafe(id), NOT NULL             | 카페 ID      |
| created_at | timestamptz | DEFAULT now()                        | 생성일       |

**collection**

| 컬럼        | 타입         | 제약조건                          | 설명           |
| ----------- | ------------ | --------------------------------- | -------------- |
| id          | uuid         | PK, DEFAULT uuid_v4()            | 컬렉션 ID     |
| user_id     | uuid         | FK → auth.users(id), NOT NULL    | 소유자 ID      |
| name        | varchar(50)  | NOT NULL                          | 컬렉션명       |
| description | text         |                                   | 설명           |
| created_at  | timestamptz  | DEFAULT now()                     | 생성일         |

### 7.2 테이블 관계도

```
auth.users 1:N → bookmark N:1 → cafe
auth.users 1:N → collection 1:N → collection_cafe N:1 → cafe
```

### 7.3 Database Rules

- **RLS 정책**: 모든 테이블에 RLS 활성화
  - bookmark: `user_id = auth.uid()` 조건으로 본인 데이터만 CRUD
  - collection: `user_id = auth.uid()` 조건으로 본인 데이터만 CRUD
  - cafe: 모든 사용자 SELECT 가능, INSERT/UPDATE는 관리자만

## 8. API 설계

### 8.1 Internal API Routes (Server Actions)

| Method        | Endpoint / Action     | 설명                 | Request          | Response              | 인증 |
| ------------- | --------------------- | -------------------- | ---------------- | --------------------- | ---- |
| Server Action | `toggleBookmark`      | 북마크 토글          | cafe_id          | boolean               | O    |
| Server Action | `createCollection`    | 컬렉션 생성          | name, description | Collection            | O    |
| Server Action | `deleteCollection`    | 컬렉션 삭제          | collection_id    | boolean               | O    |
| GET           | `/api/cafe/search`    | 카페 검색            | q, filters, page | CafeCard[]            | X    |
| GET           | `/api/cafe/[id]`      | 카페 상세            | id               | CafeDetail            | X    |

### 8.2 External API Routes

| 서비스     | Endpoint                                    | 용도          | Rate Limit       |
| ---------- | ------------------------------------------- | ------------- | ---------------- |
| Kakao Maps | `//dapi.kakao.com/v2/maps/sdk.js`           | 지도 렌더링   | 300,000/day      |
| Supabase   | `{PROJECT_URL}/rest/v1/`                    | DB CRUD       | 무제한 (자체 DB) |

## 9. 보안

### 9.1 인증/인가

- Supabase Auth OAuth 2.0 (Google, Kakao)
- Session-Token 기반 세션 관리 (httpOnly Cookie)
- 미인증 사용자: 읽기 전용 접근

### 9.2 데이터 보호

- 모든 테이블 RLS 활성화
- 사용자 비밀번호 미저장 (OAuth 전용)
- API Key는 환경변수로만 관리

### 9.3 보안 체크리스트

- [x] RLS (Row Level Security) 적용
- [x] Server Action에서 auth.uid() 검증
- [x] XSS 방지: React 자동 이스케이핑 + dangerouslySetInnerHTML 미사용
- [x] CSRF: Next.js Server Action 기본 보호
- [x] 에러 메시지에 스택 트레이스 미노출

## 10. UI/UX Design System

### 10.1 Theme & Color System

| 용도         | Light Mode   | Dark Mode    |
| ------------ | ------------ | ------------ |
| Primary      | #6F4E37      | #D4A574      |
| Background   | #FAFAF5      | #1A1A2E      |
| Surface      | #FFFFFF      | #2D2D44      |
| Text Primary | #1A1A1A      | #F5F5F5      |

### 10.2 Typography Hierarchy

| 레벨 | 크기   | 용도           |
| ---- | ------ | -------------- |
| H1   | 32px   | 페이지 타이틀  |
| H2   | 24px   | 섹션 타이틀    |
| Body | 16px   | 본문           |
| Caption | 14px | 보조 텍스트   |

### 10.3 Responsive Breakpoints

| Breakpoint | 크기      | 대상 디바이스     |
| ---------- | --------- | ----------------- |
| sm         | >= 640px  | 모바일 가로       |
| md         | >= 768px  | 태블릿            |
| lg         | >= 1024px | 데스크톱          |

### 10.4 Layout Patterns

- 모바일 퍼스트, 카드 그리드 (1열 → 2열 → 3열)
- Bottom Navigation (모바일) / Top Navigation (데스크톱)
- Sticky 검색바 + 필터 칩

## 11. 성능 요구사항

### 11.1 Core Web Vitals 목표

| 지표 | 목표값  | 최적화 전략                             |
| ---- | ------- | --------------------------------------- |
| LCP  | < 2.5s  | next/image + priority, RSC Streaming    |
| FCP  | < 1.8s  | 최소 CSS, font preload                  |
| CLS  | < 0.1   | 이미지 placeholder, skeleton UI         |
| INP  | < 200ms | useTransition, Optimistic Update        |

### 11.2 최적화 전략

- Image: next/image + WebP/AVIF 자동 변환
- Font: next/font/local + display swap
- Bundle: dynamic import로 지도 SDK lazy load
- Cache: TanStack Query staleTime 5분, ISR 1시간

## 12. 테스트 전략

### 12.1 커버리지 목표

| 테스트 유형   | 도구                       | 커버리지 목표 | 범위                     |
| ------------- | -------------------------- | ------------- | ------------------------ |
| Unit          | Vitest                     | 80%           | 유틸 함수, 비즈니스 로직 |
| Component     | Testing Library + Vitest   | 70%           | 핵심 UI 컴포넌트         |
| E2E           | Playwright                 | 핵심 플로우   | 검색, 북마크, 컬렉션     |

## 13. 배포 & CI/CD

### 13.1 배포 환경

- **Production**: Vercel (main 브랜치 자동 배포)
- **Preview**: PR 생성 시 자동 Preview Deploy
- **Database**: Supabase Cloud (ap-northeast-2)

### 13.2 CI/CD Pipeline

```
Push/PR → GitHub Actions → Lint + Type Check → Unit Test → Build → Deploy (Vercel)
```
