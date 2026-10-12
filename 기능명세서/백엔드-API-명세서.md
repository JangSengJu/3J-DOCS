# Costly 백엔드 API 명세서

> 기능명세서를 구현하기 위한 서버 API 명세 — [기능명세서](./기능명세서.md) 기반

| 항목 | 내용 |
| --- | --- |
| 문서명 | 백엔드 API 명세서 |
| 버전 | v1.0 |
| 작성일 | 2026-10-12 |
| 서버 | Node.js 내장 `node:http` (런타임 의존성 0) |
| 기본 포트 | `8787` |
| 관련 문서 | [기능명세서](./기능명세서.md) |

---

## 1. 공통 규칙

### 1.1 Base URL

| 환경 | URL |
| --- | --- |
| 로컬 개발 | `http://localhost:8787` |
| 프런트 프록시 | 프런트의 `/api/*` 요청이 동일 오리진에서 서버로 프록시됨 |

### 1.2 요청 헤더

| 헤더 | 필수 | 설명 |
| --- | --- | --- |
| `Content-Type: application/json` | 본문이 있는 경우 | JSON 본문 전송 |
| `Authorization: Bearer <token>` | 인증 필요 API | 세션 토큰 |

### 1.3 응답 규칙

| 항목 | 내용 |
| --- | --- |
| 성공 | `2xx` + JSON (SSE 는 `text/event-stream`) |
| 실패 | `{ "error": "<한국어 메시지>" }` |
| 캐시 | JSON 응답은 `Cache-Control: no-store` |

### 1.4 공통 에러 응답

| 상태 | 의미 | 예시 메시지 |
| --- | --- | --- |
| 400 | 잘못된 요청 (검증 실패) | `아이디를 입력하세요`, `JSON 파싱에 실패했습니다`, `요청 본문이 너무 큽니다` |
| 401 | 인증 실패/만료 | `로그인이 필요합니다` |
| 404 | 없는 경로 | `요청한 경로가 없습니다` |
| 500 | 서버 오류 | `서버 오류가 발생했습니다` |

에러 본문 형식:

```json
{ "error": "로그인이 필요합니다" }
```

### 1.5 CORS

허용 오리진(기본): `http://localhost:5173`, `http://127.0.0.1:5173`
허용 메서드: `GET, POST, PUT, DELETE, OPTIONS`
허용 헤더: `Content-Type, Authorization`

---

## 2. 엔드포인트 요약

| # | 메서드 | 경로 | 인증 | 설명 | 관련 기능 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `/api/health` | 불필요 | 서버 상태 + Gemini 키 유무 | F-40 |
| 2 | POST | `/api/auth/signin` | 불필요 | 로그인 (없으면 자동 가입) | F-01 |
| 3 | POST | `/api/auth/signout` | 불필요 | 로그아웃 (토큰 폐기) | F-03 |
| 4 | GET | `/api/auth/me` | 필요 | 현재 세션 사용자 조회 | F-02 |
| 5 | GET | `/api/budget` | 필요 | 내 예산·리소스 조회 (첫 방문 시 시드) | F-41, F-42 |
| 6 | PUT | `/api/budget` | 필요 | 예산·리소스 저장 | F-10~F-14, F-41 |
| 7 | POST | `/api/ai/recommend` | 필요 | AI 절감 추천 생성 | F-30, F-36 |
| 8 | POST | `/api/ai/report` | 필요 | AI 개선 보고서 생성 (SSE) | F-33, F-36 |

---

## 3. 상세 명세

### 3.1 GET /api/health

| 항목 | 내용 |
| --- | --- |
| 설명 | 서버 정상 여부와 Gemini 키 설정 여부 반환 |
| 인증 | 불필요 |
| 요청 본문 | 없음 |

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| ok | boolean | 서버 정상 여부 (항상 true) |
| gemini | boolean | Gemini API 키가 설정되어 있는지 |

```json
{ "ok": true, "gemini": true }
```

---

### 3.2 POST /api/auth/signin

| 항목 | 내용 |
| --- | --- |
| 설명 | 아이디/비밀번호로 로그인. 아이디가 없으면 자동 가입 후 세션 발급 |
| 인증 | 불필요 |
| 아이디 정규화 | `trim()` + 소문자 변환 (예: `Student@DSM.kr` → `student@dsm.kr`) |

**요청 본문**

| 필드 | 타입 | 필수 | 제약 |
| --- | --- | --- | --- |
| id | string | Y | 1~120자, 공백/대소문자 정규화 |
| password | string | Y | 1~200자 |

```json
{ "id": "student@dsm.kr", "password": "secret123" }
```

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| token | string | 세션 토큰 (64자 hex, 유효기간 7일) |
| user | PublicUser | 사용자 정보 |

```json
{
  "token": "a1b2c3...64자...",
  "user": { "id": "student@dsm.kr", "createdAt": "2026-10-12T00:00:00.000Z" }
}
```

**에러**

| 상태 | 조건 | 메시지 |
| --- | --- | --- |
| 400 | 아이디 누락 | `아이디를 입력하세요` |
| 400 | 비밀번호 누락 | `비밀번호를 입력하세요` |
| 400 | 아이디 길이 초과 | `아이디가 너무 깁니다` |
| 400 | 비밀번호 길이 초과 | `비밀번호가 너무 깁니다` |
| 400 | 비밀번호 불일치 | `비밀번호가 일치하지 않습니다` |

---

### 3.3 POST /api/auth/signout

| 항목 | 내용 |
| --- | --- |
| 설명 | 현재 세션 토큰을 서버에서 폐기 |
| 인증 | 토큰이 있으면 폐기, 없어도 성공 처리 |

**응답 204** (본문 없음)

---

### 3.4 GET /api/auth/me

| 항목 | 내용 |
| --- | --- |
| 설명 | 저장된 토큰으로 현재 세션 사용자 조회 (새로고침 후 세션 복구) |
| 인증 | 필요 |

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| user | PublicUser | 현재 로그인 사용자 |

```json
{ "user": { "id": "student@dsm.kr", "createdAt": "2026-10-12T00:00:00.000Z" } }
```

**에러**

| 상태 | 조건 |
| --- | --- |
| 401 | 토큰 없음/만료/유효하지 않음 (`로그인이 필요합니다`) |

---

### 3.5 GET /api/budget

| 항목 | 내용 |
| --- | --- |
| 설명 | 사용자별 예산·리소스 상태 조회. 저장된 값이 없으면 시드 3건을 생성해 반환 |
| 인증 | 필요 |

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| budget | BudgetDto | 예산·리소스 상태 (아래 스키마) |

```json
{
  "budget": {
    "resources": [
      {
        "id": "r1",
        "name": "web-server",
        "type": "EC2",
        "unit": "hour",
        "cost": 70.08,
        "usage": "<5%",
        "need": "대부분 유휴",
        "meta": "EC2 · m5.large · 730 시간"
      }
    ],
    "limit": 60,
    "notes": {},
    "purpose": ""
  }
}
```

**에러**

| 상태 | 조건 |
| --- | --- |
| 401 | 인증 필요 |

---

### 3.6 PUT /api/budget

| 항목 | 내용 |
| --- | --- |
| 설명 | 사용자 예산·리소스 상태 저장 (본문을 검증·정규화 후 저장, 저장본 반환) |
| 인증 | 필요 |
| 특성 | 서버가 신뢰할 수 없는 본문을 화이트리스트 기준으로 정규화. 잘못된 값은 500 대신 기본값으로 대체 |

**요청 본문 (BudgetDto)**

| 필드 | 타입 | 필수 | 제약 |
| --- | --- | --- | --- |
| resources | Resource[] | N | 최대 200건, `name` 없는 항목은 제외 |
| limit | number | N | 양수·유한값만, 아니면 기존 값 유지 |
| notes | object | N | `name → string` (문자열 최대 500자) |
| purpose | string | N | 최대 1000자 |

Resource 필드 제약:

| 필드 | 타입 | 제약 |
| --- | --- | --- |
| id | string | 최대 64자, 없으면 `r-{index}` |
| name | string | 최대 120자, 필수(공백 제외), 없으면 항목 제외 |
| type | string | 최대 32자, 기본 `기타` |
| unit | string | 최대 32자, 기본 `month` |
| cost | number | 0 이상, 소수 2자리 반올림 |
| usage | string | 최대 16자, 기본 `미상` |
| need | string | 최대 32자, 기본 `미상` |
| meta | string | 최대 200자 |

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| budget | BudgetDto | 저장된(정규화된) 상태 |

**에러**

| 상태 | 조건 |
| --- | --- |
| 400 | 본문이 JSON 객체가 아님 / 2MB 초과 |
| 401 | 인증 필요 |

---

### 3.7 POST /api/ai/recommend

| 항목 | 내용 |
| --- | --- |
| 설명 | 등록 리소스·용도·요금 카탈로그를 바탕으로 AI 절감/개선 추천 생성. 실패 시 규칙 기반 폴백 반환 |
| 인증 | 필요 |
| 특성 | 예외를 반환하지 않음 — 항상 추천 배열 반환. 추천은 최대 15건, 같은 action 은 최대 3건 |

**요청 본문**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| input | AnalysisInput | Y | 분석 입력 (아래 스키마) |

AnalysisInput:

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| limit | number | 예산 한도(USD) |
| total | number | 현재 총 예상 비용(USD) |
| purpose | string | 사용 용도 (최대 1000자) |
| resources | AnalysisResource[] | 분석 대상 리소스 (최대 200건) |

AnalysisResource:

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| name | string | 리소스 이름 (필수, 없으면 항목 제외) |
| type | string | 유형 (기본 `기타`) |
| cost | number | 월 비용 (0 이상) |
| usage | string | 사용률 (기본 `미상`) |
| need | string | 필요도 (기본 `미상`) |
| meta | string | 사양 요약 (최대 200자) |
| note | string | AI 노트 (최대 500자) |

```json
{
  "input": {
    "limit": 60,
    "total": 97.2,
    "purpose": "고객용 웹 서비스 운영",
    "resources": [
      { "name": "web-server", "type": "EC2", "cost": 70.08, "usage": "<5%", "need": "대부분 유휴", "meta": "EC2 · m5.large · 730 시간", "note": "" }
    ]
  }
}
```

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| recommendations | AiRecommendation[] | 추천 목록 (최대 15건) |
| fallback | boolean | true 면 AI 실패로 규칙 기반 추천 사용 |

```json
{
  "recommendations": [
    {
      "title": "중복 EC2 삭제 · web-server로 통합",
      "target": "web-server-2",
      "savings": 63.07,
      "grounds": ["같은 유형(EC2) 리소스가 3대로 운영 중 — 공통 용도에 비해 중복", "web-server 하나로 목적 달성 가능"],
      "action": "deletion",
      "kind": "cost"
    }
  ],
  "fallback": false
}
```

**에러**

| 상태 | 조건 |
| --- | --- |
| 401 | 인증 필요 |

> 참고: 리소스가 0건이거나 Gemini 키가 없으면 AI 호출 없이 `fallback: true` 로 즉시 응답한다.

---

### 3.7.1 AiRecommendation 스키마

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| title | string | 제안 제목 (최대 200자, 없으면 `제안 N`) |
| target | string | 대상 리소스 (없으면 빈 문자열) |
| savings | number | 월 비용 변동(USD). 양수=절감, 음수=투자, 0=변동 없음 |
| grounds | string[] | 근거 (최대 5개, 각 최대 400자, 없으면 `AI 분석 결과`) |
| action | enum | 허용 목록 외 값은 `other` 로 대체 |
| kind | enum | 허용 목록 외 값은 `cost` 로 대체 |

---

### 3.8 POST /api/ai/report

| 항목 | 내용 |
| --- | --- |
| 설명 | 개선 보고서 마크다운을 SSE 로 실시간 스트리밍 |
| 인증 | 필요 |
| 응답 형식 | `text/event-stream; charset=utf-8` |

**요청 본문**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| input | AnalysisInput | Y | 분석 입력 (3.7 과 동일) |
| recommendations | AiRecommendation[] | Y | 보고서에 반영할 추천 (최대 15건) |

**응답 헤더**

| 헤더 | 값 |
| --- | --- |
| Content-Type | `text/event-stream; charset=utf-8` |
| Cache-Control | `no-cache, no-transform` |
| Connection | `keep-alive` |
| X-Accel-Buffering | `no` |

**SSE 이벤트**

| 이벤트 | 형식 | 설명 |
| --- | --- | --- |
| 데이터 조각 | `data: {"delta":"...","fallback":false}\n\n` | 보고서 텍스트 조각 |
| 하트비트 | `: ping\n\n` | 15초 간격, 클라이언트 무시 |
| 종료 | `data: [DONE]` | 스트림 종료 신호 |

**ReportChunk 필드**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| delta | string | 이어 붙일 텍스트 조각 |
| fallback | boolean | (선택) true 면 폴백 보고서/보완 텍스트 |

**스트리밍 예시**

```
data: {"delta":"# 클라우드 아키텍처 개선 보고서\n\n"}

data: {"delta":"## 1. 요약\n"}

: ping

data: [DONE]
```

**동작 규칙**

| 조건 | 동작 |
| --- | --- |
| Gemini 키 없음 / 리소스 0건 / 추천 0건 | 폴백 보고서를 4줄·25ms 단위로 스트리밍 |
| AI 호출 실패 | 폴백 보고서로 대체 |
| 본문 20자 미만 | 폴백으로 대체 |
| `## 6. 결론` 섹션 누락 | 결론 섹션만 이어 붙임 (fallback: true) |
| 클라이언트 연결 종료 | 상류 AI 호출 취소 |

**에러**

| 상태 | 조건 |
| --- | --- |
| 401 | 인증 필요 (스트리밍 시작 전) |

---

## 4. 데이터 타입 및 Enum

### 4.1 PublicUser

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| id | string | 정규화된 사용자 아이디 |
| createdAt | string | ISO 8601 가입 시각 |

### 4.2 BudgetDto

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| resources | Resource[] | 리소스 목록 |
| limit | number | 월 예산 한도(USD) |
| notes | object | 리소스 이름 → AI 노트 |
| purpose | string | 공통 사용 용도 |

### 4.3 Resource

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| id | string | 고유 ID |
| name | string | 표시 이름 |
| type | string | 유형 |
| unit | string | 과금 단위 |
| cost | number | 월 예상 비용(USD) |
| usage | string | 사용률 |
| need | string | 필요도 |
| meta | string | 사양 요약 |

### 4.4 kind (제안 대주제)

`cost`, `performance`, `reliability`, `security`, `scalability`, `architecture`, `other`

### 4.5 action (변경 방식)

`downsizing`, `deletion`, `migration`, `reserved`, `consolidation`, `scaleup`, `tier`, `rightsizing`, `other`

### 4.6 unit (과금 단위)

`hour`, `month`, `gb_month`, `gb_transfer`, `million_requests`, `request`, `gb_second`, `lcu_hour`

### 4.7 ReportChunk (SSE 조각)

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| delta | string | 텍스트 조각 |
| fallback | boolean | 폴백 여부 (선택) |

---

## 5. 입력 검증 및 제한

| 항목 | 상한 | 처리 방식 |
| --- | --- | --- |
| 요청 본문 | 2MB | 초과 시 400 |
| 리소스 개수 | 200 | 초과분 잘라냄 |
| 추천 개수 | 15 | 초과분 잘라냄 |
| 이름 | 120자 | 초과분 잘라냄 |
| 제안 제목 | 200자 | 초과분 잘라냄 |
| meta | 200자 | 초과분 잘라냄 |
| note | 500자 | 초과분 잘라냄 |
| purpose | 1000자 | 초과분 잘라냄 |
| 근거 | 400자 | 초과분 잘라냄 |
| 금액 | — | 음수/비유한값 → 0, 소수 2자리 반올림 |

> 검증 실패 시 대부분 **500 이 아니라 조용히 정규화**한다. 클라이언트가 비정상 값을 보내도 서버는 기본값으로 대체해 응답한다.

---

## 6. 인증 흐름

```
POST /api/auth/signin { id, password }
   │  아이디 없음 → 자동 가입 (scrypt salt+hash 저장)
   │  아이디 있음 → 비밀번호 검증 (timingSafeEqual)
   ▼
{ token, user }  ── 클라이언트는 token 만 저장
   │
   ├─ GET /api/auth/me          (헤더: Authorization: Bearer <token>)
   ├─ GET/PUT /api/budget
   └─ POST /api/ai/recommend, /api/ai/report
   │
POST /api/auth/signout  → 토큰 폐기 (204)
```

| 항목 | 내용 |
| --- | --- |
| 토큰 형식 | `randomBytes(32).toString('hex')` (64자) |
| 유효기간 | 기본 7일 (`SESSION_TTL_HOURS` 로 조정) |
| 비밀번호 저장 | scrypt(salt 16바이트, keylen 64) |
| 만료 처리 | 조회 시점에 만료 세션 정리, 401 반환 |

---

## 7. 서버 환경 변수

| 변수 | 기본값 | 설명 |
| --- | --- | --- |
| `PORT` | `8787` | 서버 포트 |
| `GEMINI_API_KEY` | (없음) | Gemini API 키 (서버 전용, 미설정 시 폴백) |
| `GEMINI_MODEL` | `gemini-3.6-flash` | 사용할 Gemini 모델 |
| `DATA_FILE` | `apps/server/data/db.json` | JSON 파일 DB 경로 |
| `ALLOWED_ORIGINS` | `http://localhost:5173,http://127.0.0.1:5173` | CORS 허용 오리진 |
| `SESSION_TTL_HOURS` | `168` (7일) | 세션 토큰 유효 시간 |

---

## 8. 기능명세서 ↔ API 매핑

| 기능 ID | 기능 | API |
| --- | --- | --- |
| F-01 | 자동 가입 로그인 | `POST /api/auth/signin` |
| F-02 | 세션 복구 | `GET /api/auth/me` |
| F-03 | 로그아웃 | `POST /api/auth/signout` |
| F-10 ~ F-14 | 리소스 등록/삭제, 노트, 용도, 한도 | `GET/PUT /api/budget` |
| F-30, F-31, F-32, F-36 | AI 추천 생성·카드·요약·폴백 | `POST /api/ai/recommend` |
| F-33, F-34, F-36 | 보고서 스트리밍·진행률·폴백 | `POST /api/ai/report` |
| F-40 | 상태 확인 | `GET /api/health` |
| F-41, F-42 | 사용자별 저장·시드 | `GET/PUT /api/budget` |
