# 3J 백엔드 API 명세서

> 기능명세서를 구현하기 위한 서버 API 명세 — [기능명세서](./기능명세서.md) 기반

| 항목 | 내용 |
| --- | --- |
| 서비스명 | 3J |
| 문서명 | 백엔드 API 명세서 |
| 버전 | v2.1 |
| 작성일 | 2026-10-12 |
| 백엔드 | Java 17 + Spring Boot |
| 프런트엔드 | React |
| 기본 포트 | `8080` |

---

## 1. 공통 규칙

### 1.1 요청 헤더

| 헤더 | 필수 | 설명 |
| --- | --- | --- |
| `Content-Type: application/json` | 본문이 있을 때 | JSON 본문 |
| `Authorization: Bearer <accessToken>` | 인증 필요 API | Access Token |

### 1.2 응답 규칙

| 항목 | 내용 |
| --- | --- |
| 성공 | `2xx` + JSON (보고서는 `text/event-stream`) |
| 실패 | `{ "error": "<한국어 메시지>" }` |

### 1.3 공통 에러

| 상태 | 의미 | 예시 |
| --- | --- | --- |
| 400 | 검증 실패 | `아이디를 입력하세요` |
| 401 | 인증 실패/만료 | `로그인이 필요합니다` |
| 404 | 없는 경로 | `요청한 경로가 없습니다` |
| 409 | 중복 | `이미 가입된 아이디입니다` |
| 500 | 서버 오류 | `서버 오류가 발생했습니다` |

```json
{ "error": "로그인이 필요합니다" }
```

---

## 2. 엔드포인트 요약

| # | 메서드 | 경로 | 인증 | 설명 |
| --- | --- | --- | --- | --- |
| 1 | GET | `/api/health` | 불필요 | 서버 상태 + Gemini 키 유무 |
| 2 | POST | `/api/auth/signup` | 불필요 | 회원가입 |
| 3 | POST | `/api/auth/signin` | 불필요 | 로그인 (토큰 발급) |
| 4 | POST | `/api/auth/refresh` | Refresh | Access Token 재발급 |
| 5 | POST | `/api/auth/signout` | Access | 로그아웃 (Refresh 폐기) |
| 6 | GET | `/api/auth/me` | Access | 현재 사용자 조회 |
| 7 | GET | `/api/budget` | Access | 예산·리소스 조회 |
| 8 | PUT | `/api/budget` | Access | 예산·리소스 저장 |
| 9 | POST | `/api/ai/recommend` | Access | AI 절감 추천 생성 |
| 10 | POST | `/api/ai/report` | Access | AI 보고서 생성 (SSE) |

---

## 3. 상세 명세

### 3.1 GET /api/health

**응답 200**

```json
{ "ok": true, "gemini": true }
```

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| ok | boolean | 서버 정상 여부 |
| gemini | boolean | Gemini 키 설정 여부 |

---

### 3.2 POST /api/auth/signup

| 항목 | 내용 |
| --- | --- |
| 설명 | 아이디·비밀번호로 계정 생성. **자동 가입 없음** |
| 아이디 정규화 | `trim()` + 소문자 변환 |

**요청**

| 필드 | 타입 | 필수 | 제약 |
| --- | --- | --- | --- |
| id | string | Y | 4~120자 |
| password | string | Y | 8~200자 |

```json
{ "id": "student@dsm.kr", "password": "secret123" }
```

**응답 201**

```json
{ "user": { "id": "student@dsm.kr", "createdAt": "2026-10-12T00:00:00.000Z" } }
```

| 상태 | 조건 | 메시지 |
| --- | --- | --- |
| 400 | 아이디/비밀번호 누락 또는 길이 오류 | `아이디를 입력하세요` 등 |
| 409 | 이미 존재하는 아이디 | `이미 가입된 아이디입니다` |

---

### 3.3 POST /api/auth/signin

| 항목 | 내용 |
| --- | --- |
| 설명 | 가입된 계정으로 로그인, Access/Refresh 토큰 발급 |

**요청**

```json
{ "id": "student@dsm.kr", "password": "secret123" }
```

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| accessToken | string | JWT, 유효기간 30분 |
| user | User | 사용자 정보 |

> Refresh Token 은 `httpOnly` · `Secure` · `SameSite=Strict` 쿠키로 내려준다. (본문에 노출하지 않음)

```json
{
  "accessToken": "eyJhbGciOi...",
  "user": { "id": "student@dsm.kr", "createdAt": "2026-10-12T00:00:00.000Z" }
}
```

**에러**

| 상태 | 조건 | 메시지 |
| --- | --- | --- |
| 400 | 입력 누락 | `아이디를 입력하세요` |
| 401 | 계정 없음/비밀번호 불일치 | `아이디 또는 비밀번호가 올바르지 않습니다` |

---

### 3.3.1 JWT 토큰 구성

| 토큰 | 형식 | 유효기간 | 전달 | 서버 저장 |
| --- | --- | --- | --- | --- |
| Access Token | JWT (HS256) | 30분 | `Authorization: Bearer` | 저장 안 함 |
| Refresh Token | JWT (HS256) + `jti` | 14일 | httpOnly 쿠키 | `jti` 를 저장·폐기 |

**Access Token payload**

| 클레임 | 설명 |
| --- | --- |
| sub | 사용자 ID |
| iat | 발급 시각 |
| exp | 만료 시각 |

**Refresh Token payload**

| 클레임 | 설명 |
| --- | --- |
| sub | 사용자 ID |
| jti | 토큰 고유 ID (서버 저장·폐기 키) |
| exp | 만료 시각 |

---

### 3.4 POST /api/auth/refresh

| 항목 | 내용 |
| --- | --- |
| 설명 | 쿠키의 Refresh Token 검증 후 Access Token 재발급 (Refresh Token 회전) |
| 요청 본문 | 없음 (쿠키 사용) |

**응답 200**

```json
{ "accessToken": "eyJhbGciOi..." }
```

**동작**

1. 쿠키의 Refresh Token 검증 (서명 + `jti` 존재 + 미만료)
2. 기존 `jti` 폐기, 새 Refresh Token 발급·저장 (rotation)
3. 새 Access Token 반환

**에러**

| 상태 | 조건 |
| --- | --- |
| 401 | Refresh Token 없음/만료/폐기됨 (`로그인이 필요합니다`) |

---

### 3.5 POST /api/auth/signout

| 항목 | 내용 |
| --- | --- |
| 설명 | 서버에 저장된 Refresh Token(`jti`) 폐기 + 쿠키 삭제 |
| 요청 본문 | 없음 |

**응답 204** (본문 없음)

---

### 3.6 GET /api/auth/me

**응답 200**

```json
{ "user": { "id": "student@dsm.kr", "createdAt": "2026-10-12T00:00:00.000Z" } }
```

**에러**: `401` (`로그인이 필요합니다`)

---

### 3.7 GET /api/budget

| 항목 | 내용 |
| --- | --- |
| 설명 | 내 예산·리소스 조회. 신규 사용자는 **빈 값** (초기 템플릿 리소스 없음) |

**응답 200**

```json
{
  "budget": { "resources": [], "limit": 60, "notes": {}, "purpose": "" }
}
```

---

### 3.8 PUT /api/budget

| 항목 | 내용 |
| --- | --- |
| 설명 | 예산·리소스 저장 (본문을 검증·정규화 후 저장본 반환) |

**요청 본문 (Budget)**

| 필드 | 타입 | 제약 |
| --- | --- | --- |
| resources | Resource[] | 최대 200건 |
| limit | number | 양수, 아니면 기존 값 유지 |
| notes | object | `name → string` (500자) |
| purpose | string | 최대 1000자 |

```json
{
  "resources": [
    { "id": "r1", "name": "web-server", "type": "EC2", "unit": "hour",
      "cost": 70.08, "usage": "<5%", "need": "대부분 유휴", "meta": "EC2 · m5.large · 730 시간" }
  ],
  "limit": 60,
  "notes": {},
  "purpose": "고객용 웹 서비스 운영"
}
```

**응답 200**: `{ "budget": { ...저장된 상태... } }`

---

### 3.9 POST /api/ai/recommend

| 항목 | 내용 |
| --- | --- |
| 설명 | 리소스·용도·요금 카탈로그로 절감 추천 생성. 실패 시 규칙 기반 폴백 |

**요청**

```json
{
  "input": {
    "limit": 60,
    "total": 97.2,
    "purpose": "고객용 웹 서비스 운영",
    "resources": [
      { "name": "web-server", "type": "EC2", "cost": 70.08,
        "usage": "<5%", "need": "대부분 유휴", "meta": "EC2 · m5.large · 730 시간", "note": "" }
    ]
  }
}
```

**응답 200**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| recommendations | AiRecommendation[] | 추천 (최대 15건) |
| fallback | boolean | true 면 AI 실패로 규칙 기반 결과 |

```json
{
  "recommendations": [
    {
      "title": "web-server 사양 축소",
      "target": "web-server",
      "savings": 21.02,
      "grounds": ["사용률 <5% / 필요도 대부분 유휴 — 용도 대비 과함", "한 단계 아래 사양 전환 시 요금 감소"],
      "action": "downsizing",
      "kind": "cost"
    }
  ],
  "fallback": false
}
```

> 리소스가 0건이거나 Gemini 키가 없으면 AI 호출 없이 `fallback: true` 로 즉시 응답한다.

---

### 3.10 POST /api/ai/report

| 항목 | 내용 |
| --- | --- |
| 설명 | 개선 보고서 마크다운을 SSE 로 실시간 스트리밍 |
| 응답 형식 | `text/event-stream; charset=utf-8` |

**요청**: `{ "input": {...}, "recommendations": [...] }` (위 recommend 의 input·결과 재사용)

**응답 헤더**

| 헤더 | 값 |
| --- | --- |
| Content-Type | `text/event-stream; charset=utf-8` |
| Cache-Control | `no-cache, no-transform` |
| X-Accel-Buffering | `no` |

**SSE 형식**

```
data: {"delta":"# 클라우드 아키텍처 개선 보고서\n\n"}

data: {"delta":"## 1. 요약\n"}

data: [DONE]
```

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| delta | string | 이어 붙일 텍스트 조각 |
| fallback | boolean | (선택) true 면 폴백/보완 텍스트 |

**동작**: AI 실패·리소스 0건·추천 0건이면 폴백 보고서를 4줄·25ms 단위로 전송. `## 6. 결론` 누락 시 결론만 이어 붙임. 클라이언트가 연결을 끊으면 AI 호출도 취소.

---

## 4. 데이터 타입 및 Enum

**User**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| id | string | 정규화된 아이디 |
| createdAt | string | ISO 8601 가입 시각 |

**Resource**

| 필드 | 타입 | 기본값 |
| --- | --- | --- |
| id | string | `r-{index}` |
| name | string | 필수 (없으면 항목 제외) |
| type | string | `기타` |
| unit | string | `month` |
| cost | number | 0 (소수 2자리 반올림) |
| usage | string | `미상` |
| need | string | `미상` |
| meta | string | `''` |

**AiRecommendation**

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| title | string | 최대 200자, 없으면 `제안 N` |
| target | string | 대상 리소스 |
| savings | number | 양수=절감, 음수=투자, 0=변동 없음 |
| grounds | string[] | 최대 5개, 없으면 `AI 분석 결과` |
| action | enum | 허용 외 값은 `other` |
| kind | enum | 허용 외 값은 `cost` |

**kind**: `cost` · `performance` · `reliability` · `security` · `scalability` · `architecture` · `other`

**action**: `downsizing` · `deletion` · `migration` · `reserved` · `consolidation` · `scaleup` · `tier` · `rightsizing` · `other`

**unit**: `hour` · `lcu_hour` · `month` · `gb_month` · `gb_transfer` · `gb_second` · `million_requests` · `request`

---

## 5. 인증 흐름

```
POST /api/auth/signup  { id, password }        → 계정 생성 (중복 시 409)
POST /api/auth/signin  { id, password }        → { accessToken } + Refresh 쿠키
   │
   ├─ API 요청: Authorization: Bearer <accessToken>
   │
   ├─ (Access 만료) POST /api/auth/refresh     → 새 accessToken + Refresh 회전
   │
   └─ POST /api/auth/signout                   → Refresh 폐기 + 쿠키 삭제
```

| 항목 | 내용 |
| --- | --- |
| 인증 구현 | Spring Security + JWT 필터 (Access Token 검증) |
| 비밀번호 저장 | BCrypt 해시 |
| Access 만료 시 | 401 → 클라이언트가 `/api/auth/refresh` 호출 후 재시도 |
| Refresh 폐기 | 로그아웃, 만료, 회전 시 서버 저장소에서 `jti` 제거 |

---

## 6. 입력 제한

| 항목 | 상한 |
| --- | --- |
| 요청 본문 | 2MB |
| 리소스 / 추천 | 200건 / 15건 |
| 이름 / 제목 / meta | 120자 / 200자 / 200자 |
| note / purpose / 근거 | 500자 / 1000자 / 400자 |

> 검증 실패는 대부분 500 대신 **기본값으로 조용히 정규화**한다.

---

## 7. 환경 변수 (Spring `application.yml`)

| 변수 | 기본값 | 설명 |
| --- | --- | --- |
| `server.port` | `8080` | 서버 포트 |
| `JWT_SECRET` | (필수) | JWT 서명 키 (HS256) |
| `JWT_ACCESS_TTL` | `30m` | Access Token 유효기간 |
| `JWT_REFRESH_TTL` | `14d` | Refresh Token 유효기간 |
| `GEMINI_API_KEY` | (없음) | Gemini 키 (미설정 시 폴백) |
| `GEMINI_MODEL` | `gemini-3.6-flash` | 사용할 모델 |
| `DB_URL` / `DB_USERNAME` / `DB_PASSWORD` | (필수) | 데이터베이스 접속 정보 |
| `CORS_ALLOWED_ORIGINS` | `http://localhost:5173` | CORS 허용 오리진 |
