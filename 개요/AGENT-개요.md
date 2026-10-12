# AGENT-개요 — 3J

> AI 에이전트가 이 프로젝트를 빠르게 이해하기 위한 압축 개요. 사람이 읽는 상세 문서는 [프로젝트-개요.md](./프로젝트-개요.md) 및 하위 문서 참고.

---

## META

```yaml
service_name: 3J
type: web_service
domain: aws_cost_optimization
summary: AWS 리소스를 등록하면 월 예상 요금 계산, 예산 초과 확인, AI 절감 추천, 개선 보고서(PDF) 생성
language_docs: korean
version: v1.0
updated: 2026-10-12
```

---

## STACK

```yaml
backend: Java 17 + Spring Boot
frontend: React
auth: Spring Security + JWT (access + refresh)
database: RDBMS
ai: Gemini API (server-side only, key never exposed)
report_stream: SSE (text/event-stream)
```

---

## ACTORS

| 액터 | 권한 |
| --- | --- |
| 비로그인 | 회원가입 · 로그인 |
| 로그인 사용자 | 본인 예산·리소스 조회/저장, AI 추천·보고서 |

---

## AUTH

```yaml
signup_required: true          # 자동 가입 없음
password_hash: BCrypt
access_token:
  type: JWT (HS256)
  ttl: 30m
  transport: "Authorization: Bearer <token>"
  stored_server_side: false
refresh_token:
  type: JWT (HS256) + jti
  ttl: 14d
  transport: httpOnly Secure SameSite=Strict cookie
  stored_server_side: true     # jti 저장·폐기
rotation: refresh 시 refresh token 회전
id_normalize: "trim() + lowercase"
```

**Auth endpoints**

| 메서드 | 경로 | 요청 | 응답 |
| --- | --- | --- | --- |
| POST | `/api/auth/signup` | `{ id, password }` | `{ user }` / 409 중복 |
| POST | `/api/auth/signin` | `{ id, password }` | `{ accessToken, user }` + Refresh 쿠키 |
| POST | `/api/auth/refresh` | (쿠키) | `{ accessToken }` |
| POST | `/api/auth/signout` | – | 204 |
| GET | `/api/auth/me` | (Bearer) | `{ user }` |

---

## DOMAIN MODEL

```yaml
User:
  id: string            # 정규화된 아이디
  createdAt: string     # ISO 8601

Resource:
  id: string
  name: string          # 예: ec2-m5-large
  type: string          # 20종 중 하나
  unit: string          # 8종 중 하나
  cost: number          # 월 예상 비용 USD
  usage: string         # '<5%' | '5~20%' | '>20%'
  need: string          # '대부분 유휴' | '항상 필요'
  meta: string          # 사양 요약

Budget:
  resources: Resource[] # 최대 200
  limit: number         # 월 예산 한도 USD
  notes: map<string,string>  # name -> AI 노트 (500자)
  purpose: string       # 공통 용도 (1000자)

AiRecommendation:
  title: string         # 최대 200자
  target: string        # 대상 리소스명
  savings: number       # 양수=절감, 음수=투자, 0=변동없음
  grounds: string[]     # 최대 5개
  action: string
  kind: string

NEW_USER_BUDGET: { resources: [], limit: 60, notes: {}, purpose: "" }   # 시드 없음
```

---

## API SURFACE

| 메서드 | 경로 | 인증 | 목적 |
| --- | --- | --- | --- |
| GET | `/api/health` | – | `{ ok, gemini }` |
| POST | `/api/auth/signup` | – | 회원가입 |
| POST | `/api/auth/signin` | – | 로그인 |
| POST | `/api/auth/refresh` | Refresh | 토큰 재발급 |
| POST | `/api/auth/signout` | Access | 로그아웃 |
| GET | `/api/auth/me` | Access | 현재 사용자 |
| GET | `/api/budget` | Access | 예산 조회 |
| PUT | `/api/budget` | Access | 예산 저장 |
| POST | `/api/ai/recommend` | Access | 추천 생성 |
| POST | `/api/ai/report` | Access | 보고서 SSE |

**Error shape**: `{ "error": "<korean message>" }` · codes: 400 / 401 / 404 / 409 / 500

---

## ENUMS

```yaml
kind: [cost, performance, reliability, security, scalability, architecture, other]
action: [downsizing, deletion, migration, reserved, consolidation, scaleup, tier, rightsizing, other]
unit: [hour, lcu_hour, month, gb_month, gb_transfer, gb_second, million_requests, request]
usage: ['<5%', '5~20%', '>20%']
need: ['대부분 유휴', '항상 필요']
```

---

## COST RULES

```yaml
formula: "max(0, qty - free_tier) * unit_price"
free_tier:
  million_requests: 1
  gb_second: 400000
  gb_transfer: 100
runtime_presets:            # 시간당 과금(hour, lcu_hour)에 사용
  "24시간 상시": 730
  "평일 9–18": 176
  "야간 20–08": 365
  "주말만": 209
default_limit_usd: 60
name_rule: "{type 소문자}-{사양 소문자}, 영숫자 외 '-'로 치환"
```

---

## RESOURCE CATALOG (요약)

```yaml
types_count: 20
types: [EC2, EKS, RDS, S3, Lambda, ECS, ECR, DynamoDB, ElastiCache, VPC, ALB, CloudFront, Route53, APIGateway, SQS, SNS, CloudWatch, CloudTrail, IAM, SecretsManager]
unit_count: 8
pricing_options: 47
region: ap-northeast-2 (서울) 온디맨드 참고가
detail: ../리소스/리소스.md
```

---

## AI PIPELINE

**추천 (`POST /api/ai/recommend`)**

```yaml
input: AnalysisInput { limit, total, purpose, resources[] }
process:
  - buildPrompt: 리소스 현황 + 사용 용도 + 서울 리전 요금 카탈로그
  - Gemini: responseMimeType=json, responseSchema 강제
  - sanitize: 허용 외 값은 other/cost 로 대체, 기본값 채움
  - capByAction: 같은 action 최대 3건
  - 최대 추천 15건
output: { recommendations: AiRecommendation[], fallback: boolean }
fallback_when: [키 없음, 리소스 0건, 호출 실패, 유효 추천 0건]
fallback_logic: 유형별 대표 1개 남기고 삭제, 오버스펙 축소, EC2 다수는 서버리스/정적 호스팅 1건 통합
never_throws: true
```

**Report (SSE)**

```yaml
path: POST /api/ai/report
request: { input, recommendations }
response: text/event-stream
sections: [요약, 비용·아키텍처 현황, 개선 제안(표), 세부 검토, 실행 계획, 결론]
chunk: 'data: {"delta":"...","fallback":true?}'
done: 'data: [DONE]'
heartbeat: ': ping' (15s)
fallback_when: [키 없음, 리소스 0건, 추천 0건, 호출 실패, 본문 20자 미만]
truncated_tail: "'## 6. 결론' 없으면 결론 섹션 이어 붙임"
client_close: 상류 AI 호출 취소
```

---

## CONSTRAINTS

| 항목 | 상한 |
| --- | --- |
| 요청 본문 | 2MB |
| 리소스 / 추천 | 200 / 15 |
| 이름 / 제목 / meta | 120 / 200 / 200자 |
| note / purpose / 근거 | 500 / 1000 / 400자 |
| 아이디 / 비밀번호 | 4~120 / 8~200자 |

> 검증 실패는 500 대신 **기본값으로 조용히 정규화**하는 것이 원칙.

---

## CONVENTIONS

```yaml
api_prefix: /api
error_language: korean
money_rounding: 소수 2자리
budget_save_debounce: 400ms
unknown_enum_value_fallback: { action: other, kind: cost }
report_first_h1_removed: true    # PDF 제목 중복 방지
```

---

## DOC MAP

| 문서 | 경로 | 내용 |
| --- | --- | --- |
| 프로젝트 개요 | `개요/프로젝트-개요.md` | 사람용 전체 개요 |
| 기능명세서 | `기능명세서/기능명세서.md` | 기능·화면·데이터 |
| 백엔드 API 명세서 | `기능명세서/백엔드-API-명세서.md` | API 상세 |
| 리소스 카탈로그 | `리소스/리소스.md` | 리소스 20종 요금표 |

---

## NON_GOALS

```yaml
excluded: [비밀번호 변경, 회원 탈퇴, AWS 계정 실 연동, 실 청구 데이터 수집, 보고서 공유, 팀 협업, 결제]
```
