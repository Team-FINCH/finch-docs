# 백엔드 API 명세서

- 문서 버전: v0.8.4 (확정판)
- 작성일: 2026-08-20 / 최종 수정: 2026-09-08
- 기준 문서: [기능 명세서 v2.2](../spec/featureSpec.md)
- 범위: MVP 백엔드 API 전체. 프론트엔드가 Mock을 만들 수 있는 수준의 계약을 목표로 한다.
- 변경 이력:
  - v0.1 — 기능 명세서에서 도출한 초안
  - v0.2 — 공통 API 규격 0-5 확정 반영: `[제안]` 항목 일괄 확정, 에러 형식 `{code, message, detail}` 확정,
    토큰 정책 확정(Refresh 쿠키 전용), 등락률 단위 명문화, 페이징 기본/최대 크기 확정,
    다건 시세 파라미터 `stockCodes`로 통일과 `stale` 스키마 정의, 재발급 401 코드 2종 분리,
    AI 중계 API 장 신설(10장),
    실시간 시세 구독 방식 확정(§5.6 — STOMP + 폴링 무신호, KIS 한도는 백엔드 수집 계층이 흡수),
    리뷰 반영(Access 만료 코드 `AUTH_TOKEN_EXPIRED` 확정, AI 중계 에러 최상위 `requestId` 명시,
    STOMP CONNECT 프레임 인증, 다건 시세 최대 50건 확정, 서버 보장/프론트 권장값 표기 분리)
  - v0.3 — 스프링 표준 예외를 정확한 상태 코드로 내리기 위해 공통 에러 코드 2종 추가
    (`METHOD_NOT_ALLOWED` 405, `UNSUPPORTED_MEDIA_TYPE` 415). **기존 코드의 값·상태는 변경 없음**,
    AI 중계 upstream 상태 통과 범위를 13장 6번 결정 항목으로 등록 (10.4 현행 규칙은 유지)
  - v0.4 — **엔드포인트별 발생 코드 표 신설 (§11.2)**. 프론트 contracts P6 회신. 코드의 값·상태는 변경 없음.
    공통 계층 코드의 발생 조건 명문화(§11.1), `Authorization` 헤더 누락 시 코드 확정(§1.2),
    `AI_UPSTREAM_*`에는 `requestId`가 없음을 명시(§10.4 — contracts P16 회신),
    DELETE 계열은 대상이 없어도 `204`로 확정(§11.2), `ORDER_PRICE_CHANGED` 판정 조건을 13장 7번으로 등록
  - v0.5 — 문의 이슈 #10·#11·#12·#19·#22 회신 반영. 코드의 값·상태는 변경 없음.
    AI 재포장 규칙 구체화(§10.3 — `content` 키 유지, 보존 봉투 필드 4종, `ticker` 필드명 유지, 요청 본문도 camelCase),
    AI 위키 3종 중계 경로 추가와 `POST /wiki/theses` 중계 제외 명기(§10.1), AI 중계 단발 요청/응답 명기(§10),
    웹소켓 확정(§5.6 — 인증 실패는 STOMP ERROR 프레임, 연결 유지 중 토큰 만료 무관, 순수 WebSocket 단일),
    전량 매도 시 `holding: null` 확정(§5.2), 상장폐지 종목 검색 제외·구분 필드 없음 확정(§5.1)
  - v0.6 — 문의 이슈 #23 회신 반영. 최근 검색어 응답 명문화(§6.2 — 계정 기준 서버 저장,
    GET 응답 예시 `keywordId`·`keyword`·`searchedAt`, DELETE 는 본문 없이 204·멱등)
  - v0.7 — **투자 회차·계좌 리셋 제거** (요청 이슈 #27). **기존 계약을 깨는 변경이다.**
    엔드포인트 2개 삭제(`POST /account/reset`, `GET /rounds`), 응답의 `roundId`·`currentRoundId` 필드와
    `roundId` 쿼리 파라미터 전부 삭제, 에러 코드 `ROUND_READ_ONLY` 삭제,
    원장 유형 6종 → 4종(`ROUND_OPEN`·`ROUND_CLOSE` 삭제),
    충전 누적 한도의 기준을 회차 → 계정 전체로 변경(§4.1 응답 필드명 포함),
    투자 회차 규칙(§1.6)을 계좌 규칙으로 대체
  - v0.7.1 — `409 IDEMPOTENCY_IN_PROGRESS` 응답에 `Retry-After` 헤더 추가(§1.4). **기존 계약을 깨지 않는다** —
    헤더가 늘어난 것뿐이고, 읽지 않는 클라이언트는 그대로 동작한다. 프론트 리뷰 요청으로 추가했다
    (재시도 간격을 클라이언트가 정하고 있었다)
  - v0.8 — **충전을 2단계로 교체하고 출금을 신설한다. 기존 계약을 깨는 변경이다.**
    ① **§4 충전 전면 교체** — `POST /deposits` 한 방 호출을 삭제하고 `ready`(§4.2) → 승인(§4.3) →
    `confirm`(§4.4) 으로 나눈다. 실제 카카오페이를 쓰기로 하면서 "브라우저가 PG 로 떠났다가 돌아오는"
    흐름이 생겼기 때문이다. `paymentMethod` 열거값이 `VIRTUAL_CARD`·`VIRTUAL_TRANSFER` →
    `KAKAOPAY`·`TRANSFER` 로 바뀐다. 충전 에러 코드 6종 신설.
    ② **§4.5 출금 신설** — `POST /withdrawals`. 에러 코드 2종 신설. §8.2 필터에 `WITHDRAWAL` 추가.
    ③ **§1.4 멱등성 기준 분리** — 주문·출금은 `Idempotency-Key` 헤더, 충전 확정은 PG 가 발급한
    `paymentKey`. 충전에서 헤더가 빠진다.
    ④ **`ORDER_PRICE_CHANGED` 폐기** (구 13장 7번 확정) — 예수금 부족은 전부 `ORDER_INSUFFICIENT_CASH`.
    ⑤ **AI 중계 401·403 → 502 `AI_UPSTREAM_UNAVAILABLE`, 429 → 503 `AI_UPSTREAM_RATE_LIMITED`(신설)**
    로 재포장 (구 13장 6번 확정). 나머지 상태는 그대로 통과.
  - v0.7.2 — **초기 예수금 지급 제거.** 실제 증권 서비스가 가입만으로 돈을 주지 않으므로 계좌는 0원으로
    열리고 사용자는 충전부터 한다 (명세 2.2). **프론트 구현 전제가 바뀐다** — 가입 직후 잔고가 0이고
    매매 내역이 빈 목록이며, `INITIAL_GRANT` 원장 행이 더는 만들어지지 않는다. §2.1 · §8.2 수정.
    원장 유형과 DB 제약에서 값을 지우지는 않았다(지급 정책이 되살아나면 그대로 쓴다)
  - v0.8.1 — `503 AI_UPSTREAM_RATE_LIMITED` 응답에 `Retry-After` 헤더 추가(§10.4). **기존 계약을 깨지 않는다** —
    헤더가 늘어난 것뿐이고, 읽지 않는 클라이언트는 그대로 동작한다. 프론트 요청 이슈 #34 로 추가했다
    (AI 슬롯 여럿이 한 화면에서 각자 임의 백오프로 재시도하면 rate limit 을 더 밀어붙인다). v0.7.1 의
    `IDEMPOTENCY_IN_PROGRESS` 와 같은 원칙 — 재시도 간격은 서버가 정해서 내려준다. 구현은 AI 중계(S11)에서
  - v0.8.2 — 보유 목록에서 **시세가 없는 종목의 평가 필드를 `null`로 명문화**(§8.1). **기존 계약을 깨지 않는다** —
    필드가 사라지거나 이름이 바뀌지 않고, 지금까지 정해져 있지 않던 경우의 값을 정한 것이다. 다만
    **프론트가 `evaluationAmount`를 숫자로 가정하고 있었다면 그 자리에 `null`이 올 수 있다.**
    합계에서 제외·정렬 후순위·`asOf` 기준(가장 오래된 값)도 함께 적었다. §5.4 의 "값 없음"이 보유
    목록에서 갖는 모양이고, 종목 상세 `holding`(§5.2)에도 같은 규칙이 적용된다
  - v0.8.3 — §5.6 에 **KIS 앱키 풀** 명문화. 관계식 4 의 수용량이 `앱키 수 × 초당 한도 × 순회 주기`가 되고,
    실시간 슬롯도 `키 수 × 키당 한도`다. **프론트 계약은 바뀌지 않는다** — 키 수는 서버 설정·시크릿의 문제이고
    페이로드·관심 신호·`asOf` 는 그대로다. 처음은 키 1개로 운영하고 부족하면 키를 더한다 (S10, 이슈 #16)
  - v0.8.4 — §5.3 캔들에 `interval`(`DAY`·`WEEK`·`MONTH`)을 더하고 `period`에 `3Y`를 추가.
    **봉의 크기(`interval`)와 조회 범위(`period`)가 독립인 두 축**이 되고, 주봉·월봉은 `daily_candle`을 읽을 때
    묶어서 만든다 — 테이블도 마이그레이션도 없다. `interval` 기본값이 `DAY`라 **기존 호출은 그대로 동작한다**.
    §13 미결 3에서 주봉·월봉이 빠지고 분봉만 남는다 (이슈 #37, 티켓 169)

> **이 문서의 성격**
> 공통 API 규격(0-5)의 확정 내용을 담은 문서다. 프론트·AI 파트와 어긋나면 이 문서가 기준이며, 수정은 백엔드 파트가 한다.
> `[S0-N]` 표시는 Sprint 0 결정 항목(8/26 결정 회의)에 의존해 아직 확정할 수 없는 부분이다.

---

## 1. 공통 규약

### 1.1 기본 정보

| 항목 | 값 |
|---|---|
| Base URL | `/api/v1` |
| 내부 연동 Base URL | `/internal/v1` (AI 서버 전용, 외부 비공개) |
| 형식 | JSON (`application/json; charset=UTF-8`) |
| 시각 | ISO 8601, KST 오프셋 포함 (`2026-08-20T14:30:00+09:00`) |
| 금액 | 원 단위 정수 (`long`). 소수점·콤마 없이 숫자만 내려보낸다. 천 단위 콤마는 화면에서 처리 |
| 수량 | 정수 (`long`) |
| 등락률·수익률 | **백분율 값** (`number`, 소수 둘째 자리까지). `-1.21`은 −1.21%를 뜻한다. 프론트는 `%` 기호만 붙이고 100을 곱하지 않는다 |
| 그 외 비율 | 0~1 소수 (예: 비중 `0.0512`). % 변환은 화면 표시 계층 담당 |
| 종목코드 | 6자리 **문자열** (`"005930"`). 정수 금지 (앞자리 0 소실) |
| 요청 추적 | 모든 응답에 `X-Request-Id` 헤더를 싣는다. 오류 리포트에 이 값을 포함한다 |

### 1.2 인증

- 카카오 OAuth 2.0으로 로그인하고, 서비스 인증은 JWT를 사용한다. (명세 2.1)
- 인증이 필요한 모든 요청은 헤더에 Access Token을 담는다.

```http
Authorization: Bearer {accessToken}
```

| 토큰 | 만료 | 전달·저장 방식 |
|---|---|---|
| Access Token | 30분 | 응답 **본문**으로 전달. 프론트는 **메모리에만 보관** (`localStorage` 저장 금지) |
| Refresh Token | 14일 | **`HttpOnly + Secure + SameSite=Lax` 쿠키로만** 전달. 응답 본문에 싣지 않으며 JS에서 접근 불가 |

인증 불필요 엔드포인트: `POST /auth/kakao`, `POST /auth/refresh`

**Access Token 만료 시** 서버는 `401 AUTH_TOKEN_EXPIRED`를 반환한다. 프론트 인터셉터는 **이 코드에서만**
`POST /auth/refresh`로 재발급 후 원요청을 재시도한다. `AUTH_INVALID_TOKEN`(변조·무효)은 재시도 없이 로그인 화면으로 보낸다.

인증 필요 엔드포인트에서 두 코드를 가르는 기준은 **서명이 유효한가**다.

| 상황 | 코드 |
|---|---|
| 서명 유효, `exp` 경과 | `401 AUTH_TOKEN_EXPIRED` |
| `Authorization` 헤더 누락 · `Bearer` 형식 아님 · 서명 불일치 · 파싱 불가 | `401 AUTH_INVALID_TOKEN` |

헤더 누락을 `AUTH_TOKEN_EXPIRED`로 주면 프론트가 재발급을 시도하고, 재발급이 성공하면 원래 토큰을 안 붙인 버그가 가려진다. 그래서 누락은 `AUTH_INVALID_TOKEN`이다.

`AUTH_FORBIDDEN`(403)은 "인증은 됐지만 권한 밖"을 위해 예약된 코드다. **MVP에는 이 코드를 내는 경로가 없다** — 모든 리소스가 토큰의 사용자 기준으로 조회되고 역할 구분이 없다. 프론트는 이 코드에 화면을 만들지 않아도 된다.

### 1.3 응답 형식

성공 시 **봉투 없이 리소스를 그대로** 내려준다.

```http
HTTP/1.1 200 OK
```
```json
{ "stockCode": "005930", "stockName": "삼성전자" }
```

실패 시 HTTP 상태 코드와 함께 아래 형태를 반환한다.

```json
{
  "code": "ORDER_INSUFFICIENT_CASH",
  "message": "예수금이 부족합니다",
  "detail": { "required": 735000, "available": 512000 }
}
```

| 필드 | 설명 |
|---|---|
| `code` | 기계가 분기할 식별자 (11장 전체 목록) |
| `message` | 사용자에게 그대로 노출 가능한 한국어 문구 |
| `detail` | 화면 표시에 필요한 부가 값. 없으면 생략. 검증 실패 시 `{필드명: 사유}` 맵 |

> **예외 — AI 중계 경로(§10)의 에러에는 최상위 `requestId` 필드가 추가된다.** `POST /ai/feedback`이
> 이 값으로 원본 응답을 참조한다 (§10.3). 백엔드 자체 에러에는 이 필드가 없다 — 요청 추적은
> `X-Request-Id` 헤더(§1.1)가 담당하며, 두 식별자는 용도가 다르다.

**성공/실패 판단 규칙 (확정)**

1. **성공이냐 실패냐** — HTTP 상태 코드로 판단한다 (2xx = 성공).
2. **어떤 실패냐** — 에러 본문의 **`code` 문자열로만** 분기한다. HTTP 상태 코드 숫자나 `message` 문구로 실패 종류를 분기하지 않는다.

`isSuccess` 같은 성공 여부 필드는 두지 않는다. 성공 응답에 봉투가 없으므로 이 필드는 에러에서 항상
`false`가 되어 정보가 없고, HTTP 상태와 어긋나는 모순 상태만 만들 수 있기 때문이다.

> 프론트엔드는 `message`를 그대로 노출해도 되도록 서버가 문구를 완성해서 내려준다. 화면마다 문구를 다시 만들지 않는다. 배포된 `code`는 변경하지 않는다.

### 1.4 멱등성 (명세 11장)

**주문과 출금**은 멱등성 키가 필수다. **키는 클라이언트가 UUID v4로 생성한다** — 같은 버튼 클릭의 재시도는 같은 키, 새 클릭은 새 키.

**충전 확정(`POST /deposits/confirm`)은 이 헤더를 쓰지 않는다.** 멱등의 기준이 **PG 가 발급한 `paymentKey`** 이고, 같은 키로 다시 오면 `200` 과 최초 응답이 나간다 (§4.4). 클라이언트가 만든 UUID 로는 판정할 수 없다 — 결제창을 거쳐 돌아온 요청이라 최초 호출과 다른 화면·다른 세션일 수 있다.

| 엔드포인트 | 멱등 기준 |
|---|---|
| `POST /orders` | `Idempotency-Key` 헤더 (필수) |
| `POST /withdrawals` | `Idempotency-Key` 헤더 (필수) |
| `POST /deposits/confirm` | `paymentKey` (헤더 없음) |

```http
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

| 상황 | 동작 |
|---|---|
| 헤더 누락 | `400 IDEMPOTENCY_KEY_REQUIRED` |
| 처음 보는 키 | 정상 처리하고 결과를 키와 함께 저장 |
| 이미 처리된 키, 처리 진행 중 | `409 IDEMPOTENCY_IN_PROGRESS` → 클라이언트는 **`Retry-After` 초만큼** 대기 후 **동일 키로** 재시도 |
| 이미 처리된 키, 동일 요청 | **재처리하지 않고 최초 결과를 그대로 반환** (동일 상태 코드) |
| 이미 처리된 키, 다른 요청 본문 | `409 IDEMPOTENCY_CONFLICT` → 클라이언트 버그 신호. 재시도 금지 |

키 보관 기간은 24시간으로 확정한다.

`409 IDEMPOTENCY_IN_PROGRESS` 응답에는 **`Retry-After` 헤더가 실린다** (초 단위 정수, 기본 `1`). 재시도 간격을 서버가 정해서 내려주는 값이고, 클라이언트는 이 값만큼 기다린 뒤 **같은 키로** 다시 보낸다. 헤더가 없는 응답을 만나면 1초로 본다. (v0.7.1 추가 — 값을 클라이언트가 추측하면 파트마다 다른 간격을 고르게 되고, 너무 짧으면 앞선 처리가 끝나기 전에 `409`만 되풀이한다.)

### 1.5 페이징

목록 조회는 커서 기반을 쓴다. (명세 10.3의 trades 규칙을 전체에 통일)

| 구분 | `size` 기본값 | `size` 최대값 |
|---|---|---|
| 공개 API (`/api/v1`) | 30 | 100 |
| 내부 연동 API (`/internal/v1`) | 100 | 100 |

**요청**: `?cursor={nextCursor}&size=30`
**응답**:

```json
{
  "items": [],
  "nextCursor": "eyJpZCI6MTAxfQ==",
  "hasNext": true
}
```

- `nextCursor`가 `null`이면 마지막 페이지다.
- 커서는 **불투명 문자열**이다. 클라이언트는 파싱·조작·해석하지 않고 `nextCursor`를 그대로 되돌려 보낸다. 인코딩 방식은 서버 구현 상세이며 예고 없이 바뀔 수 있다.

### 1.6 계좌 규칙 (명세 2.1)

- 계좌는 **사용자당 하나**이고 계정 생성과 함께 만들어진다.
- 모든 요청은 토큰의 사용자로 계좌를 찾는다. **요청에 계좌 식별자를 받지 않고 응답에도 내려주지 않는다.**
  클라이언트가 지목할 대상이 아니기 때문이다.
- 계좌 초기화(리셋)와 투자 회차는 **없다.** 원장은 계정 생성부터 이어지는 하나의 연속된 시계열이다. (이슈 #27)

---

## 2. 인증 API

### 2.1 카카오 로그인

```
POST /api/v1/auth/kakao
```

인증 불필요. 카카오 인가 코드를 받아 계정을 조회하거나 생성한다.
**최초 로그인이면 계정과 함께 계좌를 생성한다. 예수금은 0원이고 초기 지급은 없다.** (명세 2.1, 2.2)

**Request**
```json
{
  "authorizationCode": "abcd1234",
  "redirectUri": "https://{도메인}/oauth/kakao"
}
```

**Response `200 OK`**
```http
Set-Cookie: refreshToken=...; HttpOnly; Secure; SameSite=Lax; Path=/api/v1/auth; Max-Age=1209600
```
```json
{
  "accessToken": "eyJhbGciOi...",
  "isNewUser": true,
  "user": {
    "userId": 1,
    "nickname": "홍길동",
    "profileImageUrl": "https://..."
  }
}
```

**Refresh Token은 본문에 싣지 않는다.** `Set-Cookie` 헤더로만 내려간다 (§1.2).

**`profileImageUrl` 은 `null` 일 수 있다.** 카카오의 프로필 사진은 선택 동의 항목이라 사용자가 동의하지 않으면 값이 없다.
빈 문자열이 아니라 `null` 로 내려간다 — `<img src="">` 는 브라우저가 현재 페이지를 다시 요청하게 만들고,
"사진 없음" 과 "빈 URL" 을 구분할 수 없게 된다. 프론트는 기본 아바타로 대체한다.


| 에러 | 상태 |
|---|---|
| `AUTH_KAKAO_FAILED` | 401 |

### 2.2 토큰 재발급

```
POST /api/v1/auth/refresh
```

**요청 본문 없음.** Refresh Token은 쿠키에서 읽는다.

**Response `200 OK`**
```http
Set-Cookie: refreshToken=...; HttpOnly; Secure; SameSite=Lax; Path=/api/v1/auth; Max-Age=1209600
```
```json
{ "accessToken": "eyJhbGciOi..." }
```

Refresh Token도 함께 갱신한다(회전 방식). 새 토큰은 `Set-Cookie`로 내려간다.

**실패 코드 2종을 구분한다** — Access Token이 메모리에만 있으므로 프론트는 **앱 부팅 시 이 API를 1회 호출**해 세션을 복구하는데, 최초 방문자(쿠키 없음)와 세션 만료(재로그인 필요)를 같은 코드로 주면 최초 방문자가 로그인 화면으로 튕긴다.

| 에러 | 상태 | 조건 | 프론트 처리 |
|---|---|---|---|
| `AUTH_REFRESH_TOKEN_MISSING` | 401 | 쿠키 자체가 없음 (최초 방문, 로그아웃 후) | **비로그인 상태로 조용히 처리.** 로그인 화면으로 강제 이동하지 않는다 |
| `AUTH_INVALID_TOKEN` | 401 | 쿠키는 있으나 만료·무효·회전 충돌 | 로그인 화면으로 이동 |

### 2.3 로그아웃

```
POST /api/v1/auth/logout
```

**Response `204 No Content`**. 서버는 해당 Refresh Token을 무효화한다.

### 2.4 내 정보 조회

```
GET /api/v1/users/me
```

프론트가 앱 부팅 시 세션을 복구하는 데 쓴다. 사용자 식별자는 토큰에서만 나오며 경로·본문·쿼리로 받지 않는다 (§1.2).

**⚠️ `currentRoundId` 필드가 v0.7 에서 삭제됐다.** 투자 회차가 없어졌고(§1.6), 계좌는 사용자당 하나라
클라이언트가 식별자로 들고 있을 이유가 없다. 계좌 도메인이 붙어도 `accountId` 로 되살리지 않는다.
직전까지 이 필드는 항상 `null` 이었으므로 값을 쓰던 프론트 코드는 없다.

**Response `200 OK`**
```json
{
  "userId": 1,
  "nickname": "홍길동",
  "profileImageUrl": "https://...",
  "joinedAt": "2026-08-25T10:00:00+09:00"
}
```

**`profileImageUrl` 은 `null` 일 수 있다.** 카카오의 프로필 사진은 선택 동의 항목이라 사용자가 동의하지 않으면 값이 없다.
빈 문자열이 아니라 `null` 로 내려간다 — `<img src="">` 는 브라우저가 현재 페이지를 다시 요청하게 만들고,
"사진 없음" 과 "빈 URL" 을 구분할 수 없게 된다. 프론트는 기본 아바타로 대체한다.

---

## 3. 계좌 API

### 3.1 계좌 요약 조회 (명세 9.1)

```
GET /api/v1/account
```

주식 잔고 화면 상단 요약. 모든 값은 원장에서 계산한 서버 값이다. (명세 9.3)

**Response `200 OK`**
```json
{
  "cashBalance": 1250000,
  "evaluationAmount": 735000,
  "totalAsset": 1985000,
  "asOf": "2026-08-20T14:30:00+09:00"
}
```

| 필드 | 설명 |
|---|---|
| `cashBalance` | 예수금 |
| `evaluationAmount` | 평가금액 = Σ(보유 수량 × 현재가) |
| `totalAsset` | 총자산 = 예수금 + 평가금액 |
| `asOf` | 시세 기준 시각. 화면에 "갱신 시각"으로 표시 |

> 포트폴리오 전체 수익률과 총자산 추이는 MVP 범위 밖이다. (명세 9.1)

> **`POST /account/reset`(계좌 리셋)과 `GET /rounds`(회차 목록)는 v0.7 에서 삭제됐다.** 실제 투자
> 서비스에는 계좌를 초기화하는 기능이 없고, 투자 회차는 리셋 시점을 경계로 원장을 나누던 부산물이라
> 함께 사라졌다. (이슈 #27)

---

## 4. 모의 결제 (충전 · 출금) API — 명세 3장

충전은 **결제 준비(ready) → 결제창 → 승인 → 확정(confirm)** 4단계다. v0.7 까지의 `POST /deposits` 한
방 호출을 대체한다.

바뀐 이유 — 실제 PG 를 쓰기로 했다. 카카오페이 결제창은 **사용자의 브라우저가 카카오로 떠났다가
돌아오는** 흐름이고, 그동안 우리 서버는 요청을 붙잡고 있을 수 없다. 그래서 "결제를 준비하고 URL 을
내려주는" 호출과 "돌아온 뒤 원장에 반영하는" 호출이 나뉜다.

```
[ready]  POST /deposits/ready        → checkoutUrl
   │
   ├─ KAKAOPAY   사용자가 checkoutUrl 로 이동 → 카카오 결제 → 우리 서버로 리다이렉트
   │             GET /deposits/kakao/approval  (무인증, 브라우저가 부른다)
   │                 → 302 프론트 성공 URL ?paymentId&paymentKey&amount
   │
   └─ TRANSFER   프론트가 자체 모의 이체 화면을 띄운다
                 POST /deposits/{paymentId}/mock-approve → paymentKey
   │
[confirm] POST /deposits/confirm {paymentId, paymentKey, amount}  → 예수금 반영
```

**결제 서버를 따로 두지 않는다.** `domain/deposit` 안에 `PaymentGateway` 인터페이스로 경계만 긋고,
카카오페이 구현과 모의 이체 구현이 그 뒤에 선다. 분리가 필요해지면 그 인터페이스가 잘리는 선이다.

### 4.1 충전 한도 조회

```
GET /api/v1/deposits/limit
```

**Response `200 OK`**
```json
{
  "perRequestLimit": 10000000,
  "cumulativeLimit": 100000000,
  "depositedAmount": 3000000,
  "remainingAmount": 97000000
}
```

**누적 한도의 기준은 계정 전체다** (명세 1.1). 회차가 없어지면서 한도를 되돌릴 경로도 없어졌다.
v0.6 의 `roundCumulativeLimit`·`roundDepositedAmount` 가 `cumulativeLimit`·`depositedAmount` 로 바뀌었다.

### 4.2 결제 준비 (ready)

```
POST /api/v1/deposits/ready
```

**Request**
```json
{
  "amount": 1000000,
  "paymentMethod": "KAKAOPAY"
}
```

`paymentMethod`: `KAKAOPAY` | `TRANSFER`

**Response `201 Created`**
```json
{
  "paymentId": 77,
  "paymentMethod": "KAKAOPAY",
  "amount": 1000000,
  "checkoutUrl": "https://online-pay.kakao.com/mockup/v1/...",
  "expiresAt": "2026-08-20T14:46:02+09:00"
}
```

- `checkoutUrl` — `KAKAOPAY` 는 카카오가 준 결제창 주소, `TRANSFER` 는 프론트의 모의 이체 화면 주소다.
  프론트는 수단을 구분하지 않고 이 URL 로 보내면 된다.
- `expiresAt` 이 지나면 그 건은 만료된다(서버가 `FAILED` 로 정리한다). 기본 15분.

**판정 순서** — 위에서 걸리면 아래는 보지 않는다.

| # | 조건 | 에러 | 상태 |
|---|---|---|:---:|
| 1 | `paymentMethod` 가 열거값 밖 | `INVALID_REQUEST` | 400 |
| 2 | 금액 0원 이하 | `DEPOSIT_AMOUNT_INVALID` | 400 |
| 3 | 1회 1,000만 원 초과 | `DEPOSIT_PER_REQUEST_LIMIT_EXCEEDED` | 409 |
| 4 | 계정 누적 1억 원 초과 | `DEPOSIT_LIMIT_EXCEEDED` (`detail.remainingAmount`) | 409 |
| 5 | PG 가 결제 준비를 거절·응답 없음 | `DEPOSIT_PG_UNAVAILABLE` | 502 |

> 누적 한도는 **여기서 미리 한 번, `confirm` 에서 다시 한 번** 본다. 여기 검사는 사용자 편의다 —
> 결제창까지 갔다가 마지막에 막히는 것보다 낫다. **진실은 `confirm` 의 판정**이고, 그 사이에 다른 충전이
> 끼어들면 여기서 통과한 건도 `confirm` 에서 막힐 수 있다.

### 4.3 결제 승인

수단에 따라 둘 중 하나가 불린다. **둘 다 예수금을 건드리지 않는다** — 원장 반영은 `confirm` 뿐이다.

#### 4.3.1 카카오페이 승인 콜백 (무인증)

```
GET /api/v1/deposits/kakao/approval?paymentId={id}&pg_token={token}
```

**이 경로는 인증이 없다.** 카카오가 사용자의 브라우저를 이 주소로 리다이렉트하는데, 그 요청에는
`Authorization` 헤더를 붙일 방법이 없다. 프론트가 부르는 API 가 아니다.

- 성공 → `302` 프론트 성공 URL `?paymentId={id}&paymentKey={key}&amount={amount}`
- 실패 → `302` 프론트 실패 URL `?paymentId={id}&code={에러코드}`

두 URL 은 서버 설정값이다. **응답이 302 뿐이라 본문 에러 형식(§1.3)을 쓰지 않는다.**

> **무인증인데 왜 안전한가** — 이 호출이 하는 일은 결제 건을 `APPROVED` 로 표시하는 것뿐이고
> **돈은 움직이지 않는다.** 위조하려면 카카오가 발급한 `pg_token` 이 필요한데 그 검증은 카카오가 한다.
> 남의 `paymentId` 를 넣으면 실패 URL 로 보낸다 — 404 를 주지 않는 이유는 그것 자체가
> "그 번호는 존재한다"는 정보이기 때문이다.

#### 4.3.2 모의 이체 승인

```
POST /api/v1/deposits/{paymentId}/mock-approve
```

**Request** (선택)
```json
{ "scenario": "SUCCESS" }
```

`scenario`: `SUCCESS`(기본) | `INSUFFICIENT_BALANCE` | `LIMIT_EXCEEDED` | `TIMEOUT` — 실패 흐름을
시연·테스트하기 위한 값이다.

**Response `200 OK`**
```json
{ "paymentId": 77, "paymentKey": "mock_pk_9f2c...", "amount": 1000000 }
```

| 에러 | 상태 | 조건 |
|---|---|---|
| `DEPOSIT_NOT_FOUND` | 404 | 없는 `paymentId` 이거나 **내 것이 아님** |
| `DEPOSIT_INVALID_STATE` | 409 | `KAKAOPAY` 건에 호출. 이 API 는 `TRANSFER` 전용이다 |
| `DEPOSIT_PAYMENT_FAILED` | 409 | `scenario` 가 실패값. 그 건은 `FAILED` 로 굳는다 |

### 4.4 충전 확정 (confirm)

```
POST /api/v1/deposits/confirm
```

**Request**
```json
{ "paymentId": 77, "paymentKey": "mock_pk_9f2c...", "amount": 1000000 }
```

**Response `201 Created`** — v0.7 의 `POST /deposits` 응답과 같은 모양이다.
```json
{
  "depositId": 55,
  "amount": 1000000,
  "paymentMethod": "KAKAOPAY",
  "cashBalanceAfter": 2250000,
  "depositedAt": "2026-08-20T14:31:02+09:00"
}
```

**같은 `paymentKey` 로 다시 보내면 `200 OK` 와 함께 최초와 같은 본문이 나간다.** 예수금은 한 번만
늘어난다. 새로고침·네트워크 재시도로 이 호출이 두 번 도착하는 것이 정상 경로다.

**판정 순서**

| # | 조건 | 에러 | 상태 |
|---|---|---|:---:|
| 1 | 없는 `paymentId`/`paymentKey`, 내 것이 아님 | `DEPOSIT_NOT_FOUND` | 404 |
| 2 | 아직 승인 전(`READY`) | `DEPOSIT_NOT_APPROVED` | 409 |
| 3 | 실패로 굳은 건(`FAILED`) | `DEPOSIT_PAYMENT_FAILED` | 409 |
| 4 | 금액이 준비 시점과 다름 | `DEPOSIT_AMOUNT_MISMATCH` (건은 `FAILED`) | 400 |
| 5 | 계정 누적 1억 원 초과 | `DEPOSIT_LIMIT_EXCEEDED` (건은 `FAILED`) | 409 |
| — | 통과 | 원장·예수금 반영 | 201 |

> **`Idempotency-Key` 헤더를 쓰지 않는다** (§1.4). 멱등의 기준이 **PG 가 발급한 `paymentKey`** 다.
> 클라이언트가 만든 UUID 로는 "같은 결제인가"를 판정할 수 없다 — 결제창을 거쳐 돌아온 요청이라
> 최초 호출과 다른 화면·다른 세션일 수 있기 때문이다.

> **충전 취소 API는 제공하지 않는다.** (명세 1.1, 3.2)

### 4.5 출금

```
POST /api/v1/withdrawals
Idempotency-Key: {UUID}   ← 필수
```

**Request**
```json
{ "amount": 500000 }
```

**Response `201 Created`**
```json
{
  "withdrawalId": 12,
  "amount": 500000,
  "cashBalanceAfter": 750000,
  "withdrawnAt": "2026-08-20T15:02:11+09:00"
}
```

**판정 순서**

| # | 조건 | 에러 | 상태 |
|---|---|---|:---:|
| 1 | `Idempotency-Key` 헤더 누락 | `IDEMPOTENCY_KEY_REQUIRED` | 400 |
| 2 | 금액 0원 이하 | `WITHDRAWAL_AMOUNT_INVALID` | 400 |
| 3 | 예수금 부족 | `WITHDRAWAL_INSUFFICIENT_CASH` (`detail.availableAmount`) | 409 |

- **출금 가능액은 예수금 전액이다.** 별도 한도가 없다 — 잔고가 곧 상한이다.
- **출금 한도 조회 API 를 만들지 않는다.** 출금 가능액은 `GET /account` 의 `cashBalance` 다.
- **출금 수단(은행·계좌번호)을 받지 않는다.** 모의 서비스라 받아도 쓰이는 곳이 없다.

> **출금해도 충전 한도는 돌아오지 않는다.** `GET /deposits/limit` 의 `depositedAmount` 는 **계좌 평생
> 누적 충전액**이라 출금과 무관하다. 되돌리면 충전↔출금을 반복해 누적 한도를 무한히 우회할 수 있다.
> **프론트는 출금 후 한도가 그대로인 것을 버그로 신고하지 않도록 이 규칙을 화면에 반영해야 한다.**

---

## 5. 종목 API

### 5.1 종목 검색 · 자동완성 (명세 4장)

```
GET /api/v1/stocks/search?keyword={검색어}&size=10
```

- 2글자 이상부터 호출한다.
- 종목명 또는 종목코드로 검색한다. 초성 검색은 확장 범위.

**Response `200 OK`**
```json
{
  "items": [
    {
      "stockCode": "005930",
      "stockName": "삼성전자",
      "market": "KOSPI",
      "currentPrice": 73500,
      "changeAmount": -900,
      "changeRate": -1.21,
      "suspended": false
    }
  ]
}
```

`market`: `KOSPI` | `KOSDAQ`
`suspended`: 거래정지 여부. `true`면 화면에 뱃지를 노출하고 매수를 막는다.

**상장폐지 종목(`stock.is_active = false`)은 검색 결과에서 제외한다.** 응답에 구분 필드는 없다.
보유 중인 종목이 상장폐지되는 상황은 MVP 범위에서 발생하지 않으므로(종목 마스터는 프로젝트 기간 중 고정 적재)
종목 상세(§5.2)·보유 목록(§8.1) 응답에도 구분 필드를 두지 않는다. (이슈 #19)

### 5.2 종목 상세 (명세 7.1)

```
GET /api/v1/stocks/{stockCode}
```

**이 API 호출 시 서버가 최근 본 종목에 자동 기록한다.** (명세 5장) 별도 등록 API는 없다.

**Response `200 OK`**
```json
{
  "stockCode": "005930",
  "stockName": "삼성전자",
  "market": "KOSPI",
  "currentPrice": 73500,
  "previousClose": 74400,
  "changeAmount": -900,
  "changeRate": -1.21,
  "suspended": false,
  "suspendedReason": null,
  "watched": true,
  "asOf": "2026-08-20T14:30:00+09:00",
  "holding": {
    "quantity": 10,
    "avgBuyPrice": 71200,
    "evaluationProfit": 23000,
    "evaluationProfitRate": 3.23
  }
}
```

- `watched`: 관심 종목 등록 여부 (토글 초기 상태)
- `holding`: 보유하지 않으면 `null`. **전량 매도로 `quantity = 0`인 행이 남아 있는 경우도 `null`이다** —
  잔존 행은 재매수 시 INSERT 경합을 막기 위한 내부 구현(ERD §2.6)이고 API로 노출하지 않는다.
  프론트는 `holding !== null`로 보유 여부를 판단한다. (이슈 #19)

| 에러 | 상태 |
|---|---|
| `STOCK_NOT_FOUND` | 404 |

### 5.3 캔들 차트 (명세 7.2)

```
GET /api/v1/stocks/{stockCode}/candles?period=1Y&interval=WEEK
```

| 파라미터 | 값 | 기본값 | 뜻 |
|---|---|---|---|
| `period` | `1M` \| `3M` \| `1Y` \| `3Y` | `1M` | 오늘(KST)부터 거슬러 올라갈 달력일 (30·90·365·1095) |
| `interval` | `DAY` \| `WEEK` \| `MONTH` | `DAY` | 봉 하나의 크기 |

두 축은 **독립**이다. `1Y`+`DAY`(일봉 250개 근처)도, `3Y`+`MONTH`(월봉 36개)도 된다.
`interval`은 선택이고 기본이 `DAY`라 이 파라미터가 없던 시절의 호출은 그대로 동작한다.
열거값 밖은 둘 다 `INVALID_REQUEST`(400)이고 `detail`은 `{period: 사유}` · `{interval: 사유}`다.

분봉은 이 열거값이 아니라 `minute_candle`을 새로 만드는 확장이며 도입 여부는 `[S0-4]`.

**집계 규칙** — 저장은 언제나 일봉(`daily_candle`)이고 주봉·월봉은 읽을 때 묶는다.

| 필드 | 규칙 |
|---|---|
| `date` | 구간의 **첫 거래일**. 주봉이라고 그 주 월요일이 아니다 — 월요일이 휴장이면 실제 거래가 있었던 첫날 |
| `open` | 첫 거래일의 시가 |
| `high` / `low` | 구간 내 최대 / 최소 |
| `close` | 마지막 거래일의 종가 |
| `volume` | 구간 합계 |

- 주 경계는 **월요일 시작**(ISO), 월 경계는 **1일 시작**. 기준 시간대는 KST.
- 거래일이 하나도 없는 주·달은 **봉을 만들지 않는다.** 빈 봉을 끼워 넣지 않는다.
- **마지막 봉은 진행 중일 수 있다.** 이번 주·이번 달이 안 끝났으면 그 시점까지의 값이다 — 일봉의 마지막 봉이 장중에 움직이는 것과 같다.
- 봉이 없는 종목은 빈 배열이다. 에러가 아니다.

**Response `200 OK`**
```json
{
  "stockCode": "005930",
  "period": "1Y",
  "interval": "WEEK",
  "candles": [
    {
      "date": "2026-08-17",
      "open": 74000, "high": 76200, "low": 73100, "close": 75800,
      "volume": 61234567
    }
  ]
}
```

### 5.4 현재가 조회

```
GET /api/v1/stocks/{stockCode}/price
```

폴링 방식일 때 사용한다. 갱신 주기와 실시간 여부는 `[S0-2]`.

**Response `200 OK`**
```json
{
  "stockCode": "005930",
  "currentPrice": 73500,
  "changeAmount": -900,
  "changeRate": -1.21,
  "asOf": "2026-08-20T14:30:00+09:00",
  "stale": false
}
```

**`stale` 규칙 (확정)** — 시세가 정상이 아닌 상태는 두 가지이고, 필드 값이 다르다.

| 상황 | `currentPrice`·`changeAmount`·`changeRate` | `asOf` | `stale` |
|---|---|---|---|
| 정상 수신 중 | 최신 값 | 최신 수신 시각 | `false` |
| **수신 끊김** (허용 시간 초과) | **마지막 수신 값 유지** | **마지막 수신 시각** | `true` |
| **값 없음** (캐시 미스, 수신 이력 없음) | **전부 `null`** | `null` | `true` |

프론트는 `stale: true`면 "시세 지연"을 표시하고, 값이 `null`이면 가격 영역을 비운다. 허용 시간은 `[S0-3]`.

### 5.5 다건 현재가 조회

```
GET /api/v1/stocks/prices?stockCodes=005930,000660,035720
```

관심 종목 목록, 보유 종목 목록에서 N+1 호출을 막기 위한 벌크 조회다.
파라미터 이름은 필드 표기(`stockCode`)와 같은 계열인 **`stockCodes`로 확정**한다 (`codes`·`tickers` 아님).
**한 번에 최대 50건 (확정)** — 관심 종목 최대 50개 정책과 정합. KIS 실시간 등록 한도는 §5.6의
백엔드 수집 계층이 흡수하므로 이 최대 건수와 무관하다.

**Response `200 OK`**
```json
{
  "items": [
    { "stockCode": "005930", "currentPrice": 73500, "changeAmount": -900, "changeRate": -1.21, "asOf": "2026-08-20T14:30:00+09:00", "stale": false },
    { "stockCode": "000660", "currentPrice": null, "changeAmount": null, "changeRate": null, "asOf": null, "stale": true }
  ]
}
```

- 항목 스키마는 5.4 단건 응답과 동일하며, `stale` 규칙도 5.4를 그대로 따른다. **항목별로** 판정한다 — 일부 종목만 지연이어도 나머지는 정상 값으로 내려간다.
- 요청한 종목 중 존재하지 않는 코드는 `items`에서 제외한다 (전체 요청을 실패시키지 않는다).

### 5.6 실시간 시세 구독 (확정 — STOMP)

최종 구조는 **웹소켓(STOMP) + REST 폴링 폴백**이다. 프론트는 폴링(5.5)을 먼저 구현하고,
`subscribe/unsubscribe` 추상화 계층 뒤에서 STOMP 클라이언트로 교체한다. 두 경로의 시세 페이로드 스키마는 동일하다.

**KIS 한도는 프론트 계약에 노출하지 않는다.** KIS 실시간 등록 한도(`[S0-1]` 실측)는 백엔드 수집 계층이 흡수한다 —
관심도 상위 종목만 KIS 실시간에 등록하고 나머지는 백엔드가 KIS REST를 폴링해, 전 종목을 하나의 시세 캐시에
동일 스키마로 적재한다. 프론트는 종목이 어느 티어인지 알 수 없고 알 필요도 없다. 신선도 차이는 `asOf`로만 드러난다.

#### 웹소켓 (STOMP)

| 항목 | 값 |
|---|---|
| 전송 | **순수 WebSocket 단일. SockJS 폴백은 두지 않는다** — 타깃(모바일 웹, 최신 브라우저)은 전부 WebSocket을 지원하고, WS 불가 환경은 아래 REST 폴링 폴백이 흡수한다. 접속 스킴은 `ws://`(HTTPS 배포 시 `wss://`), `sockjs-client` 불필요 (이슈 #10) |
| 핸드셰이크 | `WS /ws` → STOMP CONNECT. 핸드셰이크 단계는 인증 없이 통과시킨다 |
| 인증 | **CONNECT 프레임의 `Authorization: Bearer {accessToken}` 헤더.** 브라우저 `WebSocket` 생성자는 커스텀 HTTP 헤더를 붙일 수 없으므로 핸드셰이크가 아닌 CONNECT 프레임에서 검증한다. URL 쿼리로 토큰을 보내지 않는다(로그 노출) |
| 인증 실패 | **STOMP ERROR 프레임을 보내고 연결을 닫는다.** ERROR 프레임의 `message` 헤더에 코드 문자열만 싣는다 — 만료는 `AUTH_TOKEN_EXPIRED`, 헤더 누락·변조·무효는 `AUTH_INVALID_TOKEN`(구분 규칙은 §1.2와 동일). §1.3의 JSON 에러 본문 형식은 따르지 않는다. 프론트 분기는 REST 인터셉터와 동일 — 만료면 재발급 후 재CONNECT, 무효면 로그인 화면 (이슈 #10) |
| 토큰 만료 | **검증은 CONNECT 시 1회. 연결 유지 중 Access Token이 만료돼도 끊지 않는다** — 시세 topic은 사용자별 데이터가 아닌 공개 시세다. 재연결 시 CONNECT 재검증에서 만료가 잡히면 위 인증 실패 규칙을 따른다 (이슈 #10) |
| 구독 | `/topic/prices/{stockCode}`에 **SUBSCRIBE 프레임. 프레임 자체가 구독 신호다** — 별도 구독 메시지·REST 신호 엔드포인트는 없다 |
| 해제 | UNSUBSCRIBE 프레임 또는 연결 종료. 연결이 끊기면(탭 종료·네트워크 단절 포함) 서버가 해당 연결의 구독을 전부 회수한다 |
| 수신 페이로드 | 5.4 단건 응답과 **동일 스키마**: `{ "stockCode": "005930", "currentPrice": 73500, "changeAmount": -900, "changeRate": -1.21, "asOf": "...", "stale": false }` |
| 생존 확인 | STOMP 내장 하트비트 **10초/10초 (양방향)**. **3회 미수신(30초)** 시 서버가 연결을 닫고 슬롯을 회수한다. 정상 종료는 disconnect 이벤트로 즉시 회수 |

클라이언트의 해제 신호에 의존하지 않는 구조이므로, 프론트는 `pagehide`·`visibilitychange` 등 이탈 이벤트에서 별도 신호를 보내지 않는다.

#### 폴링 선행·폴백 단계 — 무신호

명시적 구독·해제 신호가 없다. **`GET /stocks/prices?stockCodes=...` 요청 자체를 관심 신호로 간주**하고,
서버는 종목별 마지막 요청 시각을 기록해 **TTL 30초** 경과 시 슬롯을 회수한다. 폴링 중단이 곧 해제다.
프론트 추상화 계층에서 `subscribe()` = 폴링 시작, `unsubscribe()` = 폴링 중단으로 구현한다.

**서버가 보장하는 것(계약)은 TTL 30초뿐이다.** 폴링 주기는 프론트 재량이며 아래는 권장값이다 —
목록 화면 5초, 주문 화면 3초. 캐시 신선도 임계는 주기보다 짧게(주기의 80% 수준) 두기를 권장한다.
권장값은 관계식 1만 지키면 프론트가 자유롭게 조정할 수 있고, 조정 시 이 문서를 고칠 필요가 없다.

#### 수치 기본값과 관계식

아래 값은 기본값이며, `[S0-1]` 실측(KIS 실시간 등록 한도의 정확한 값과 단위, KIS REST 초당 호출 한도,
다건 조회 엔드포인트 유무) 결과에 따라 조정할 수 있다. **조정하더라도 관계식은 유지한다.**

| 항목 | 기본값 |
|---|---|
| STOMP 하트비트 간격 | 10초 (양방향) |
| 웹소켓 회수 타임아웃 | 하트비트 3회 미수신 = 30초 |
| 폴링 티어 TTL | 30초 |
| 프론트 폴링 주기 (권장 — 프론트 재량) | 목록 5초 · 주문 화면 3초 |
| 백엔드 → KIS REST 순회 주기 | 3초 목표 (= 폴링 티어 종목의 시세 지연 상한) |

관계식:

1. **폴링 TTL ≥ 프론트 폴링 주기 × 4~6** — 요청 한두 번 실패로 슬롯이 회수·재등록을 반복(플래핑)하지 않게 한다
2. **웹소켓 회수 타임아웃 = 하트비트 간격 × 3** — 간격을 바꾸면 타임아웃이 따라온다
3. **`stale` 허용 시간(`[S0-3]`) > KIS 순회 주기** — 폴링 티어는 구조적으로 순회 주기만큼 지연되므로, 임계가 그보다 짧으면 폴링 티어 전체가 상시 "시세 지연"으로 표시된다. 순회 3초 기준 최소 10초 이상을 권장
4. **폴링 티어 수용량 = 앱키 수 × KIS REST 초당 호출 한도 × 순회 주기** — 수용량을 넘으면 순회 주기를 늘려(지연 증가) 자연 열화시킨다. KIS 한도는 앱키(계정) 단위라 백엔드가 앱키를 **풀**로 들고 종목을 나눠 호출하면 수용량이 키 수만큼 는다 (v0.8.3)

#### 실시간 슬롯 배정 (서버 내부 규칙)

두 단계 모두 같은 로직을 쓴다 — 종목별 "마지막 관심 시각"(STOMP 구독 존재 여부 또는 폴링 요청 시각) 기준
서버 전체 LRU로 KIS 실시간 슬롯을 배정·회수한다. 슬롯 한도 수치가 바뀌어도 프론트 계약은 변하지 않는다.

**앱키 풀 (v0.8.3).** 백엔드는 KIS 앱키를 목록(`finch.kis.keys[]`)으로 들고, REST 폴링은 관심 종목을 키 수로 나눠 호출하며
실시간 티어는 키마다 웹소켓 세션 하나를 열어 **슬롯 = 키 수 × 키당 한도**로 넓힌다. 처음은 키 1개로 운영하고 부족하면
키를 항목으로 더한다 — 이때 바뀌는 것은 서버 설정과 시크릿뿐이고, 시세 페이로드·관심 신호·`asOf` 계약은 그대로다.
프론트는 키 수를 알 수 없고 알 필요도 없다.

---

## 6. 최근 본 종목 · 최근 검색어 · 관심 종목

### 6.1 최근 본 종목 (명세 5장)

최대 30건 FIFO, 중복 시 최상단 갱신. 계정 기준 서버 저장.

```
GET    /api/v1/stocks/recent
DELETE /api/v1/stocks/recent/{stockCode}
DELETE /api/v1/stocks/recent
```

**GET Response `200 OK`**
```json
{
  "items": [
    { "stockCode": "005930", "stockName": "삼성전자", "currentPrice": 73500, "changeRate": -1.21, "viewedAt": "..." }
  ]
}
```

### 6.2 최근 검색어 (명세 4장)

최대 10건. **계정 기준 서버 저장** (ERD §2.11 `recent_search_keyword` — 계좌가 아니라 `user_id` 귀속).
같은 검색어를 다시 검색하면 새 항목을 만들지 않고 `searchedAt`만 갱신한다.

```
GET    /api/v1/stocks/search/recent
DELETE /api/v1/stocks/search/recent/{keywordId}
DELETE /api/v1/stocks/search/recent
```

**GET Response `200 OK`** — 최신순(`searchedAt` DESC)
```json
{
  "items": [
    { "keywordId": 42, "keyword": "삼성", "searchedAt": "2026-09-02T14:03:00+09:00" }
  ]
}
```

- `keywordId`: `DELETE .../{keywordId}`에 쓰는 식별자
- `keyword`: 검색어 원문. 종목코드로 검색한 경우에도 문자열로 저장한다 — 최근 본 종목(§6.1)과 달리
  시세 필드가 없는 것은 의도다 (검색어는 종목이 아니라 문자열이다)
- `DELETE` 두 경로는 본문 없이 `204 No Content`. **없는 대상을 지워도 `204`** (§11.2의 멱등 규칙).
  다른 사용자의 `keywordId`를 지목한 경우도 구분 없이 `204`다 — 존재 여부 자체가 정보 노출이라
  알려주지 않는다 (이슈 #23)

### 6.3 관심 종목 (명세 6장)

**최대 50개.**

```
GET    /api/v1/watchlist?sort=REGISTERED
POST   /api/v1/watchlist
DELETE /api/v1/watchlist/{stockCode}
```

`sort`: `REGISTERED`(기본) | `NAME` | `CHANGE_RATE`

**GET Response `200 OK`**
```json
{
  "count": 12,
  "maxCount": 50,
  "items": [
    {
      "stockCode": "005930",
      "stockName": "삼성전자",
      "currentPrice": 73500,
      "changeAmount": -900,
      "changeRate": -1.21,
      "held": true,
      "registeredAt": "..."
    }
  ]
}
```

`held`: 보유 중이면 `true` → 화면에 "보유" 뱃지

**POST Request**: `{ "stockCode": "005930" }` → `201 Created`

| 에러 | 상태 | 메시지 |
|---|---|---|
| `WATCHLIST_LIMIT_EXCEEDED` | 409 | "관심 종목은 최대 50개까지 등록할 수 있어요" |
| `WATCHLIST_ALREADY_EXISTS` | 409 | |

---

## 7. 주문 API — 명세 7.3, 7.4

### 7.1 시장가 주문

```
POST /api/v1/orders
Idempotency-Key: {UUID}   ← 필수
```

MVP는 시장가 즉시 체결만 존재한다. 접수와 체결이 분리되지 않는다.

**Request**
```json
{
  "stockCode": "005930",
  "side": "BUY",
  "quantity": 10
}
```

`side`: `BUY` | `SELL`

**Response `201 Created`**
```json
{
  "orderId": 101,
  "stockCode": "005930",
  "stockName": "삼성전자",
  "side": "BUY",
  "quantity": 10,
  "executedPrice": 73500,
  "executedAmount": 735000,
  "executedAt": "2026-08-20T14:31:10+09:00",
  "cashBalanceAfter": 515000,
  "realizedProfit": null
}
```

`realizedProfit`: 매도일 때만 값이 있다.

### 7.2 체결 처리 순서 (명세 7.3)

서버는 아래 순서로 처리한다. **확인 화면의 예상 금액과 실제 체결가는 다를 수 있으므로 접수 시점에 재검증한다.**

```
1. 거래 시간 확인 (09:00~15:30 KST)      → 아니면 ORDER_MARKET_CLOSED
2. 종목 거래정지 확인                     → 정지면 ORDER_STOCK_SUSPENDED
3. 최신 수신 가격 조회                    → 없거나 허용 시간 초과면 ORDER_PRICE_UNAVAILABLE
4. 체결 직전 재검증 (최신 가격 기준 재계산)
     매수: 예수금 >= 수량 × 최신가        → 부족하면 ORDER_INSUFFICIENT_CASH
     매도: 보유 수량 >= 주문 수량          → 부족하면 ORDER_INSUFFICIENT_QUANTITY
5. 원장 기록 + 잔고 반영 (단일 트랜잭션)
     매수: 예수금 차감, 보유 수량 증가, 평균 매수가 가중평균 재계산
     매도: 예수금 증가, 보유 수량 차감, 실현손익 계산, 전량 매도 시 잔고에서 제거
```

> 4번에서 부족하면 **수량을 임의로 줄여 체결하지 않고 주문을 거부한다.** (명세 7.3)
> 5번은 원장 기준 잔액 검증과 트랜잭션으로 동시 주문 시 예수금이 음수가 되지 않도록 한다. (명세 11장)

| 에러 | 상태 | 메시지 |
|---|---|---|
| `ORDER_MARKET_CLOSED` | 409 | "지금은 주문할 수 없어요 (거래 시간 09:00~15:30)" |
| `ORDER_STOCK_SUSPENDED` | 409 | 거래정지 사유 포함 |
| `ORDER_PRICE_UNAVAILABLE` | 503 | "시세를 불러올 수 없어 주문이 제한됩니다" |
| `ORDER_INSUFFICIENT_CASH` | 409 | "예수금이 부족합니다" |
| `ORDER_INSUFFICIENT_QUANTITY` | 409 | "보유 수량이 부족합니다" |
| `ORDER_QUANTITY_INVALID` | 400 | 0 이하 |

### 7.3 주문 가능 정보 조회

주문 화면의 비율 버튼(10%/25%/50%/최대)과 거래 가능 여부 판단에 사용한다.

```
GET /api/v1/orders/available?stockCode=005930&side=BUY
```

**Response `200 OK`**
```json
{
  "tradable": true,
  "reason": null,
  "currentPrice": 73500,
  "availableCash": 1250000,
  "maxQuantity": 17,
  "holdingQuantity": 10
}
```

`tradable`이 `false`면 `reason`에 위 에러 코드 중 하나가 담긴다. 화면은 이 값으로 주문 버튼을 비활성화한다.

---

## 8. 잔고 · 매매 내역 API

### 8.1 보유 종목 목록 (명세 9.2)

```
GET /api/v1/portfolio?sort=EVALUATION
```

`sort`: `EVALUATION`(평가금액순, 기본) | `PROFIT_RATE`(수익률순)

**Response `200 OK`**
```json
{
  "cashBalance": 1250000,
  "evaluationAmount": 735000,
  "totalAsset": 1985000,
  "asOf": "2026-08-20T14:30:00+09:00",
  "holdings": [
    {
      "stockCode": "005930",
      "stockName": "삼성전자",
      "quantity": 10,
      "avgBuyPrice": 71200,
      "currentPrice": 73500,
      "evaluationAmount": 735000,
      "evaluationProfit": 23000,
      "evaluationProfitRate": 3.23
    }
  ]
}
```

계산식 (명세 0장 용어 정의):
- `evaluationAmount` = 보유 수량 × 현재가
- `evaluationProfit` = (현재가 − 평균 매수가) × 보유 수량
- `evaluationProfitRate` = 평가손익 ÷ (평균 매수가 × 보유 수량) × 100

**시세가 없는 종목은 평가 네 필드가 전부 `null`이다** (v0.8.2). `currentPrice` ·
`evaluationAmount` · `evaluationProfit` · `evaluationProfitRate` 가 함께 `null`로 나가고,
`stockCode` · `stockName` · `quantity` · `avgBuyPrice` 는 시세와 무관하게 언제나 값이 있다.
§5.4 의 "값 없음" 상태가 보유 목록에서 갖는 모양이고, 그 종목이 목록에서 빠지지는 않는다 —
가진 주식은 시세를 모를 때에도 보여야 한다.

- **상단 `evaluationAmount` 합계에 그 종목은 더해지지 않는다.** 0으로 치면 자산이 사라진 것처럼
  보이므로 빼고, 대신 해당 종목의 평가 필드가 `null`이라 화면이 합계가 부분값임을 알 수 있다.
  `totalAsset` 도 같은 합계를 쓴다.
- **정렬에서는 뒤로 간다.** `EVALUATION` · `PROFIT_RATE` 둘 다 값 없는 종목이 마지막이다.
- **`asOf` 는 보유 종목들의 시세 기준 시각 중 가장 오래된 값이다.** 화면의 갱신 시각이 실제보다
  신선해 보이지 않게 가장 보수적인 값을 쓴다. 보유가 없거나 전부 시세가 없으면 응답 시각이다.

같은 규칙이 종목 상세의 `holding` 블록(§5.2)에도 적용된다 — 현재가가 없으면 `evaluationProfit` ·
`evaluationProfitRate` 가 `null`이고 `quantity` · `avgBuyPrice` 는 나간다.

### 8.2 매매 내역 (명세 8장)

원장 기반 통합 내역이다. 충전도 함께 조회된다.

```
GET /api/v1/transactions?type=ALL&cursor=&size=30
```

| 파라미터 | 값 |
|---|---|
| `type` | `ALL`(기본) \| `BUY` \| `SELL` \| `DEPOSIT` \| `WITHDRAWAL` |

`type`은 원장 유형이 아니라 **화면 필터 축**이다. 다섯 중 `ALL`은 원장 유형이 아니고, 나머지 넷만 원장
유형과 1:1로 대응한다. 그래서 `INITIAL_GRANT`를 가리키는 필터 값은 없다.

**`type=DEPOSIT`은 원장 유형 `DEPOSIT`(충전)만이다. 출금은 들어가지 않는다.** 이 필터의 합계가
`GET /deposits/limit`의 `depositedAmount`(= 충전액 누적)와 같아야 하기 때문이다.
둘이 어긋나면 "충전 내역을 다 더했는데 한도 화면의 숫자와 다르다"가 된다.

**`type=WITHDRAWAL`은 원장 유형 `WITHDRAWAL`(출금)만이다.** `amount`는 **양수 절대값**으로 내려간다 —
부호는 원장이 갖고 화면은 "출금"이라는 구분으로 방향을 표시한다. 충전 건과 같은 규칙이다.

**`INITIAL_GRANT`는 v0.7.2 부터 발행되지 않는다.** 초기 지급이 없어졌기 때문이다(명세 2.2). 원장 유형과
DB 제약에는 값이 남아 있고 지급 정책이 되살아나면 그때 다시 기록되지만, **지금 만들어지는 계정에는
이 유형의 행이 없다.** 프론트는 `INITIAL_GRANT`를 "올 수도 있는 값"으로 다루면 되고, **반드시 1건 있을
것으로 기대하면 안 된다.**

**갓 가입한 계정의 내역은 빈 목록이다.** 예수금이 0이고 원장에 아무 행도 없다 — `items: []`,
`nextCursor: null`, `hasNext: false`. 오류가 아니라 정상 초기 상태다.

화면의 필터 이름은 **"충전"**과 **"출금"**이다(명세 8장).

**Response `200 OK`**
```json
{
  "items": [
    {
      "transactionId": 301,
      "type": "SELL",
      "occurredAt": "2026-08-20T14:31:10+09:00",
      "stockCode": "005930",
      "stockName": "삼성전자",
      "price": 73500,
      "quantity": 5,
      "amount": 367500,
      "realizedProfit": 11500,
      "realizedProfitRate": 3.23,
      "paymentMethod": null
    },
    {
      "transactionId": 300,
      "type": "WITHDRAWAL",
      "occurredAt": "2026-08-20T14:31:05+09:00",
      "stockCode": null,
      "stockName": null,
      "price": null,
      "quantity": null,
      "amount": 500000,
      "realizedProfit": null,
      "realizedProfitRate": null,
      "paymentMethod": null
    },
    {
      "transactionId": 299,
      "type": "DEPOSIT",
      "occurredAt": "2026-08-20T14:31:02+09:00",
      "stockCode": null,
      "stockName": null,
      "price": null,
      "quantity": null,
      "amount": 1000000,
      "realizedProfit": null,
      "realizedProfitRate": null,
      "paymentMethod": "KAKAOPAY"
    }
  ],
  "nextCursor": "eyJpZCI6Mjk4fQ==",
  "hasNext": true
}
```

`WITHDRAWAL` 행은 `paymentMethod` 가 `null` 이다 — 출금은 수단을 받지 않는다 (§4.5).

`type` 전체 값 (명세 8장 원장 유형): `INITIAL_GRANT` | `DEPOSIT` | `WITHDRAWAL` | `BUY` | `SELL`
(`INITIAL_GRANT`는 현재 발행되지 않는다 — 위 참고)
정렬은 최신순 고정. 기간·종목 필터는 확장 범위.

---

## 9. AI 서버 내부 연동 API — 명세 10.3

- **읽기 전용이다.** AI 서버는 원장을 쓰지 않는다. (명세 10.1)
- 외부에 노출하지 않고 서비스 간 내부 토큰으로 인증한다.
- 필드 최종 확정은 `[S0-5]`.

```http
X-Internal-Token: {서비스 간 공유 토큰}
X-User-Id: 1
```

### 9.1 포트폴리오 조회

```
GET /internal/v1/portfolio
```

```json
{
  "cashBalance": 1250000,
  "asOf": "2026-08-20T14:30:00+09:00",
  "holdings": [
    {
      "stockCode": "005930",
      "stockName": "삼성전자",
      "quantity": 10,
      "avgBuyPrice": 71200,
      "currentPrice": 73500
    }
  ]
}
```

### 9.2 거래 이력 조회

```
GET /internal/v1/trades?cursor=&size=100
```

```json
{
  "trades": [
    {
      "tradeId": 101,
      "stockCode": "005930",
      "side": "BUY",
      "price": 71200,
      "quantity": 10,
      "executedAt": "2026-08-18T10:12:44+09:00"
    }
  ],
  "nextCursor": null,
  "hasNext": false
}
```

기본 100건, 커서 기반 페이징. (명세 10.3)

> **미확정**: 평가손익 같은 파생 지표를 백엔드가 함께 내려줄지 AI가 계산할지는 `[S0-5]`에서 정한다. 현재 초안은 **원본 값만 내려주고 AI가 계산**하는 쪽이다.

---

## 10. AI 중계 API

호출 경로는 **프론트 → 백엔드 → AI 서버**로 확정됐다. 프론트는 AI 서버를 직접 호출하지 않고, 백엔드가 `/api/v1/ai/**` 요청을 AI 서버(`/api/ai/v1/**`)로 중계한다.

**AI 중계는 모두 단발 요청/응답이다. SSE 등 스트리밍은 하지 않는다** ([AI 명세 §2.5](../../ai/docs/api-spec.md) — "스트리밍을 하려면 백엔드가 스트리밍 프록시를 먼저 만들어야 한다. 계획에 없는 작업이다").
`POST /api/v1/ai/chat`처럼 응답이 오래 걸릴 수 있는 경로도 단발이다 — 프론트는 응답이 올 때까지 로딩 상태로 기다린다. (이슈 #12)

### 10.1 경로 매핑

| 프론트가 부르는 경로 | 중계 대상 (AI 서버) | 용도 |
|---|---|---|
| `POST /api/v1/ai/stocks/{stockCode}/analysis` | `POST /api/ai/v1/stocks/{ticker}/analysis` | 종목 분석 |
| `POST /api/v1/ai/chat` | `POST /api/ai/v1/chat` | 대화 에이전트 (용어 설명 포함) |
| `POST /api/v1/ai/portfolio/diagnosis` | `POST /api/ai/v1/portfolio/diagnosis` | 포트폴리오 진단 |
| `POST /api/v1/ai/portfolio/attribution` | `POST /api/ai/v1/portfolio/attribution` | 수익률 원인 분석 |
| `POST /api/v1/ai/orders/preview` | `POST /api/ai/v1/orders/preview` | 주문 전 점검 |
| `GET /api/v1/ai/briefing` | `GET /api/ai/v1/briefing` | 데일리 브리핑 |
| `POST /api/v1/ai/feedback` | `POST /api/ai/v1/feedback` | 응답 피드백 (`requestId` 기반) |
| `GET /api/v1/ai/wiki` | `GET /api/ai/v1/wiki` | 사용자 위키 조회 |
| `PUT /api/v1/ai/wiki/theses/{stockCode}` | `PUT /api/ai/v1/wiki/theses/{ticker}` | 투자 논지 수정 |
| `DELETE /api/v1/ai/wiki/facts/{factId}` | `DELETE /api/ai/v1/wiki/facts/{factId}` | 위키 사실 삭제 |

`POST /api/ai/v1/wiki/theses`는 AI 서비스가 내부에서 스스로 호출하는 경로라 **중계 대상이 아니다** — 프론트 호출 경로가 없다. (이슈 #12, #7 회신 기준)

### 10.2 인증

- 프론트 → 백엔드: 일반 API와 동일하게 `Authorization: Bearer {accessToken}`. **백엔드가 JWT를 검증**한다.
- 백엔드 → AI 서버: 서비스 간 토큰과 사용자 식별 헤더를 싣는다 (AI 명세 §4의 백엔드 신뢰 헤더 방식). AI 서버는 외부에 직접 노출하지 않는다.

### 10.3 응답 재포장 규칙

백엔드는 AI 응답 봉투를 벗겨 **백엔드 형식(camelCase, 성공 시 봉투 없음)으로 재포장**해 내려준다.
재포장은 엔드포인트별 `content` 스키마를 모르는 **제네릭 변환**이다. (이슈 #10·#11·#22 확정)

- **`content` 키는 유지한다** (안쪽 키를 최상위로 평탄화하지 않는다). 봉투 필드 중 아래 보존 4종만 `content` 옆에 남기고, `generated_at`·`model`·`cached` 등 나머지 봉투 필드는 걷어낸다
- 키 표기는 **snake_case → camelCase 재귀 변환만** 적용하고, 필드 **이름 자체는 바꾸지 않는다** — AI 응답 본문의 `ticker`·`relatedTickers` 계열을 `stockCode`로 바꾸지 않는다 (경로 파라미터만 `{stockCode}`, §10.1)
- `content` 안쪽 값은 건드리지 않는다. briefing의 `content.generatedAt`(배치 생성 시각)은 걷어내는 봉투의 `generated_at`과 별개 필드로, 자기 자리에 그대로 남는다

**요청 본문도 camelCase다.** 프론트 → 백엔드 요청은 `requestId`·`linkedTradeId` 등 camelCase로 보내고,
AI 서버로 넘길 때의 snake_case 변환은 백엔드 중계 계층이 담당한다. (이슈 #12)

재포장 후 본문 예시 (`POST /ai/stocks/{stockCode}/analysis`):

```json
{
  "content": { "riskLevel": "high", "riskScore": 72, "summary": {}, "findings": [] },
  "requestId": "req_20260902_0001",
  "dataAsOf": {},
  "citations": [],
  "disclaimer": "본 서비스는 모의투자이며 투자 자문이 아닙니다."
}
```

아래 필드는 화면 노출 필수이므로 **재포장 후에도 본문에 반드시 보존**한다.

| 보존 필드 (camelCase 변환 후) | 이유 |
|---|---|
| `dataAsOf` | 시세·뉴스·거시 지표의 기준 시각. "몇 시 기준" 표기가 없으면 오해를 만든다 |
| `citations` | 답변 근거 출처 |
| `disclaimer` | 투자 자문 회피 문구. 모든 AI 응답에 노출한다 |
| `requestId` | `POST /ai/feedback`이 이 값으로 원본 응답을 찾는다. **에러 응답에도 보존**한다 |

### 10.4 에러 통과 규칙

- **AI 서버가 반환한 에러는 `code`·`message`·`detail`과 HTTP 상태를 그대로 통과**시킨다. 백엔드가 자기 5xx로 뭉개지 않는다. 프론트가 코드별로 다른 화면 처리를 해야 하기 때문이다 — 특히 `INSUFFICIENT_DATA`(409, 재시도 무의미)·`GUARDRAIL_BLOCKED`(422, 재시도 유도 금지)·`RETRIEVAL_FAILED`(502, 재시도 가능)·`LLM_TIMEOUT`(504, 재시도 가능). 전체 목록은 [AI 서비스 API 명세 §3](./aiApiSpec.md)을 따른다.
- **예외가 둘 있다. `401`·`403`과 `429`는 그대로 통과시키지 않고 재포장한다** (v0.8 확정, 구 13장 6번).

| upstream | 내려보내는 코드 | 상태 | 재포장하는 이유 |
|---|---|:---:|---|
| `401` · `403` | `AI_UPSTREAM_UNAVAILABLE` (`detail.reason = "upstream_auth"`) | 502 | 그대로 내려가면 **프론트 인터셉터가 사용자 토큰 만료로 오인해 로그아웃**시킨다. 실제 원인은 백엔드↔AI 사이의 내부 토큰 문제이고 사용자와 무관하다 |
| `429` | `AI_UPSTREAM_RATE_LIMITED` (**`Retry-After` 헤더 동봉**) | 503 | 그대로 내려가면 **사용자가 요청을 너무 많이 보낸 것으로 오인**된다. 실제로는 백엔드 전체의 AI 호출량이 상한에 닿은 것이다 |

  나머지 상태 코드는 규칙 그대로 통과한다. `detail.reason` 은 백엔드가 붙이는 값이고 AI 서버가 준
  `detail` 은 이 두 경우에 버려진다 — 내부 인증 실패 메시지에는 사용자에게 보여줄 것이 없다.

  **`503 AI_UPSTREAM_RATE_LIMITED` 에는 `Retry-After` 헤더가 실린다** (v0.8.1, 이슈 #34). 규칙은 §1.4 의
  `IDEMPOTENCY_IN_PROGRESS` 와 같다 — 재시도 간격은 클라이언트가 추측하지 않고 서버가 내려준다.
  - 값은 **초 단위 정수**다. AI 서버가 429 에 붙여 준 `Retry-After` 를 그대로 옮기고, HTTP 날짜 형식이면 백엔드가 초로 환산한다.
  - AI 서버가 헤더를 주지 않으면 백엔드 기본값을 싣는다 (`finch.ai.rate-limit-retry-after`, 기본 **5초**). 헤더가 없는 503 은 없다.
  - **최소 1초**다. 0 은 "즉시 재시도"라 되풀이를 부른다.
  - 프론트는 이 값만큼 기다린 뒤 같은 요청을 다시 보낸다. AI 슬롯 여럿이 한 화면에 떠 있으면 전부 같은 값을 받으므로
    각자 임의 백오프로 rate limit 을 더 밀어붙이는 일이 없다.
  - `502 AI_UPSTREAM_UNAVAILABLE`·`504 AI_UPSTREAM_TIMEOUT` 에는 붙이지 않는다 — 언제 복구될지 서버도 모른다.

- **백엔드 자신이 AI 서버에 도달하지 못한 경우**는 아래 코드를 새로 내려준다. 프론트는 이 코드로 AI 위젯만 에러 처리하고 나머지 화면(시세·주문)은 살린다.

| 에러 | 상태 | 조건 |
|---|---|---|
| `AI_UPSTREAM_UNAVAILABLE` | 502 | AI 서버 연결 실패·비정상 응답. **위 401·403 재포장분도 이 코드다** |
| `AI_UPSTREAM_RATE_LIMITED` | 503 | AI 서버가 429 를 돌려줌 (v0.8 신설) |
| `AI_UPSTREAM_TIMEOUT` | 504 | 중계 타임아웃. 타임아웃 값은 AI의 LLM 타임아웃보다 길게 잡아 AI가 먼저 `LLM_TIMEOUT`을 돌려주게 한다 |

- **연결 실패·타임아웃 두 코드에는 최상위 `requestId`가 없다.** 백엔드 자체 에러(§1.3)이고, AI 서버가 응답하지 않았으므로 `POST /ai/feedback`으로 찾을 원본 응답 자체가 없다. §10.3의 "에러 응답에도 보존"은 AI 서버가 돌려준 에러에만 해당한다. 프론트는 `requestId` 유무로 피드백 슬롯을 조건부로 만든다.
  - **재포장한 401·403·429 는 AI 서버가 응답은 했으므로 `requestId` 가 있으면 보존한다.** 다만 그 응답으로 피드백을 남길 내용은 없다.

---

## 11. 에러 코드 목록

### 인증
| 코드 | 상태 |
|---|---|
| `AUTH_KAKAO_FAILED` | 401 |
| `AUTH_REFRESH_TOKEN_MISSING` | 401 |
| `AUTH_INVALID_TOKEN` | 401 |
| `AUTH_TOKEN_EXPIRED` | 401 |
| `AUTH_FORBIDDEN` | 403 |

### 공통
| 코드 | 상태 |
|---|---|
| `INVALID_REQUEST` | 400 |
| `RESOURCE_NOT_FOUND` | 404 |
| `IDEMPOTENCY_KEY_REQUIRED` | 400 |
| `IDEMPOTENCY_IN_PROGRESS` | 409 |
| `IDEMPOTENCY_CONFLICT` | 409 |
| `METHOD_NOT_ALLOWED` | 405 |
| `UNSUPPORTED_MEDIA_TYPE` | 415 |
| `INTERNAL_ERROR` | 500 |

### AI 중계 (백엔드 발행분)
| 코드 | 상태 |
|---|---|
| `AI_UPSTREAM_UNAVAILABLE` | 502 |
| `AI_UPSTREAM_RATE_LIMITED` | 503 (`Retry-After` 헤더 동봉, §10.4) |
| `AI_UPSTREAM_TIMEOUT` | 504 |

AI 서버가 발행하는 코드(`INSUFFICIENT_DATA`, `GUARDRAIL_BLOCKED`, `RETRIEVAL_FAILED`, `LLM_TIMEOUT` 등)는 그대로 통과되며, 목록은 [AI 서비스 API 명세 §3](./aiApiSpec.md)이 관리한다. **단 401·403·429 는 통과하지 않고 위 코드로 재포장한다** (§10.4).

### 충전
| 코드 | 상태 |
|---|---|
| `DEPOSIT_AMOUNT_INVALID` | 400 |
| `DEPOSIT_AMOUNT_MISMATCH` | 400 |
| `DEPOSIT_NOT_FOUND` | 404 |
| `DEPOSIT_PER_REQUEST_LIMIT_EXCEEDED` | 409 |
| `DEPOSIT_LIMIT_EXCEEDED` | 409 |
| `DEPOSIT_NOT_APPROVED` | 409 |
| `DEPOSIT_PAYMENT_FAILED` | 409 |
| `DEPOSIT_INVALID_STATE` | 409 |
| `DEPOSIT_PG_UNAVAILABLE` | 502 |

### 출금
| 코드 | 상태 |
|---|---|
| `WITHDRAWAL_AMOUNT_INVALID` | 400 |
| `WITHDRAWAL_INSUFFICIENT_CASH` | 409 |

### 종목 · 관심 종목
| 코드 | 상태 |
|---|---|
| `STOCK_NOT_FOUND` | 404 |
| `WATCHLIST_LIMIT_EXCEEDED` | 409 |
| `WATCHLIST_ALREADY_EXISTS` | 409 |

### 주문
| 코드 | 상태 |
|---|---|
| `ORDER_QUANTITY_INVALID` | 400 |
| `ORDER_MARKET_CLOSED` | 409 |
| `ORDER_STOCK_SUSPENDED` | 409 |
| `ORDER_INSUFFICIENT_CASH` | 409 |
| `ORDER_INSUFFICIENT_QUANTITY` | 409 |
| `ORDER_PRICE_UNAVAILABLE` | 503 |

### 11.1 공통 계층 — 모든 엔드포인트에서 나올 수 있는 코드

아래 코드는 컨트롤러에 닿기 전 공통 계층(예외 핸들러·인증 필터·멱등성 인터셉터)이 내며, §11.2 표에서는 **생략**한다.

**모든 엔드포인트**

| 코드 | 상태 | 조건 |
|---|---|---|
| `INVALID_REQUEST` | 400 | 본문 JSON 파싱 실패 · Bean Validation 실패 · 필수 파라미터 누락 · 파라미터 타입 불일치 · 열거값 밖(`period=2W`, `sort=FOO` 등). `detail`은 `{이름: 사유}` 맵이며 파싱 실패일 때만 생략 |
| `METHOD_NOT_ALLOWED` | 405 | 존재하는 경로에 허용되지 않은 메서드. `Allow` 헤더에 허용 메서드를 싣는다 |
| `UNSUPPORTED_MEDIA_TYPE` | 415 | 본문을 받는 엔드포인트에 `application/json`이 아닌 `Content-Type` |
| `RESOURCE_NOT_FOUND` | 404 | 매핑되지 않은 경로 |
| `INTERNAL_ERROR` | 500 | 처리되지 않은 예외. 원인 메시지는 본문에 싣지 않는다 |

**인증 필요 엔드포인트** (`POST /auth/kakao` · `POST /auth/refresh` 제외 전부)

| 코드 | 상태 | 조건 |
|---|---|---|
| `AUTH_TOKEN_EXPIRED` | 401 | 서명 유효, 만료 (§1.2) |
| `AUTH_INVALID_TOKEN` | 401 | 헤더 누락 · 형식 오류 · 서명 불일치 (§1.2) |

**멱등성 키 필수 엔드포인트** (`POST /orders` · `POST /withdrawals`)

| 코드 | 상태 | 조건 |
|---|---|---|
| `IDEMPOTENCY_KEY_REQUIRED` | 400 | 헤더 누락 (§1.4) |
| `IDEMPOTENCY_IN_PROGRESS` | 409 | 같은 키 처리 중 (§1.4) |
| `IDEMPOTENCY_CONFLICT` | 409 | 같은 키, 다른 본문 (§1.4) |

멱등성 판정은 본문 검증보다 **앞선다** — 키가 없으면 본문이 틀려도 `IDEMPOTENCY_KEY_REQUIRED`다.

### 11.2 엔드포인트별 발생 코드

공통 계층(§11.1)을 제외하고 **그 엔드포인트의 처리 로직이 직접 내는 코드**만 적는다. "—"는 고유 코드가 없다는 뜻이다.
같은 요청에 여러 사유가 겹치면 **비고의 판정 순서에서 먼저 걸리는 코드 하나**만 나간다.

| Method | Path | 고유 코드 | 비고 |
|---|---|---|---|
| POST | `/auth/kakao` | `AUTH_KAKAO_FAILED` | 무인증. 카카오 토큰 교환 실패·카카오 사용자 조회 실패 모두 이 코드. `authorizationCode`·`redirectUri` 누락은 `INVALID_REQUEST` |
| POST | `/auth/refresh` | `AUTH_REFRESH_TOKEN_MISSING` · `AUTH_INVALID_TOKEN` | 무인증. 쿠키 없음 → `MISSING`, 쿠키 있으나 만료·무효·회전 충돌 → `INVALID` (§2.2) |
| POST | `/auth/logout` | — | Refresh 쿠키가 없어도 `204`. Access Token은 있어야 한다 |
| GET | `/users/me` | — | |
| GET | `/account` | — | |
| GET | `/deposits/limit` | — | |
| POST | `/deposits/ready` | `DEPOSIT_AMOUNT_INVALID` · `DEPOSIT_PER_REQUEST_LIMIT_EXCEEDED` · `DEPOSIT_LIMIT_EXCEEDED` · `DEPOSIT_PG_UNAVAILABLE` | 판정 순서: `paymentMethod` 열거값(`INVALID_REQUEST`) → 금액 0 이하 → 1회 한도 → 누적 한도(`detail.remainingAmount`) → PG 준비 실패. **멱등성 헤더 없음** (§4.2) |
| GET | `/deposits/kakao/approval` | — | **무인증.** 실패해도 본문 에러가 아니라 `302` 로 프론트 실패 URL 에 보낸다 (§4.3.1) |
| POST | `/deposits/{paymentId}/mock-approve` | `DEPOSIT_NOT_FOUND` · `DEPOSIT_INVALID_STATE` · `DEPOSIT_PAYMENT_FAILED` | `TRANSFER` 전용. 판정 순서: 내 건인지(`NOT_FOUND`) → 수단이 `TRANSFER` 인지(`INVALID_STATE`) → `scenario` 실패값(`PAYMENT_FAILED`) |
| POST | `/deposits/confirm` | `DEPOSIT_NOT_FOUND` · `DEPOSIT_NOT_APPROVED` · `DEPOSIT_PAYMENT_FAILED` · `DEPOSIT_AMOUNT_MISMATCH` · `DEPOSIT_LIMIT_EXCEEDED` | 판정 순서는 §4.4 표. **멱등 기준은 `paymentKey`** 이고 재전송은 `200` + 최초 본문 |
| POST | `/withdrawals` | `WITHDRAWAL_AMOUNT_INVALID` · `WITHDRAWAL_INSUFFICIENT_CASH` | 판정 순서: 멱등성 → 금액 0 이하 → 예수금 부족(`detail.availableAmount`) |
| GET | `/stocks/search` | — | `keyword` 2글자 미만 · `size` 범위 밖 → `INVALID_REQUEST`. 결과 없음은 빈 `items` |
| GET | `/stocks/{stockCode}` | `STOCK_NOT_FOUND` | 상장폐지 종목 노출 여부는 별도 확정 항목(프론트 contracts P18) |
| GET | `/stocks/{stockCode}/candles` | `STOCK_NOT_FOUND` | `period` 열거값 밖 → `INVALID_REQUEST` |
| GET | `/stocks/{stockCode}/price` | `STOCK_NOT_FOUND` | **시세 없음은 에러가 아니다** — `stale: true` + `null` (§5.4) |
| GET | `/stocks/prices` | — | `stockCodes` 누락·빈 값·50건 초과 → `INVALID_REQUEST`. 없는 종목은 `items`에서 제외하고 실패시키지 않는다 (§5.5) |
| GET | `/stocks/recent` | — | |
| DELETE | `/stocks/recent` · `/stocks/recent/{stockCode}` | — | **대상이 없어도 `204`** (멱등). 목록에 없는 종목·이미 지운 항목을 다시 지워도 실패하지 않는다 |
| GET | `/stocks/search/recent` | — | |
| DELETE | `/stocks/search/recent` · `/stocks/search/recent/{keywordId}` | — | 대상이 없어도 `204` |
| GET | `/watchlist` | — | `sort` 열거값 밖 → `INVALID_REQUEST` |
| POST | `/watchlist` | `STOCK_NOT_FOUND` · `WATCHLIST_ALREADY_EXISTS` · `WATCHLIST_LIMIT_EXCEEDED` | 판정 순서: 종목 존재 → 중복 → 한도. 이미 등록된 종목은 한도가 찼어도 `ALREADY_EXISTS` |
| DELETE | `/watchlist/{stockCode}` | — | 대상이 없어도 `204` |
| POST | `/orders` | `ORDER_QUANTITY_INVALID` · `STOCK_NOT_FOUND` · `ORDER_MARKET_CLOSED` · `ORDER_STOCK_SUSPENDED` · `ORDER_PRICE_UNAVAILABLE` · `ORDER_INSUFFICIENT_CASH` · `ORDER_INSUFFICIENT_QUANTITY` | 판정 순서: 멱등성 → `side` 열거값(`INVALID_REQUEST`) → 수량 0 이하 → 종목 존재 → §7.2 1~5단계. `ORDER_INSUFFICIENT_CASH`는 매수, `ORDER_INSUFFICIENT_QUANTITY`는 매도에서만 |
| GET | `/orders/available` | `STOCK_NOT_FOUND` | `side` 열거값 밖 → `INVALID_REQUEST`. `tradable: false`의 `reason`은 `ORDER_MARKET_CLOSED` · `ORDER_STOCK_SUSPENDED` · `ORDER_PRICE_UNAVAILABLE` 중 하나이며 **HTTP 200**이다 (§7.3) |
| GET | `/portfolio` | — | `sort` 열거값 밖 → `INVALID_REQUEST`. 보유 없음은 빈 `holdings` |
| GET | `/transactions` | — | `type` 열거값 밖 · `cursor` 손상 · `size` 범위 밖 → `INVALID_REQUEST`. 내역 없음은 빈 `items` |
| POST | `/ai/stocks/{stockCode}/analysis` | `AI_UPSTREAM_UNAVAILABLE` · `AI_UPSTREAM_TIMEOUT` + **AI 서버 발행 코드 통과** | AI 서버 코드 목록은 [aiApiSpec §3](./aiApiSpec.md). 백엔드는 종목 존재를 미리 검사하지 않는다 — AI 서버의 `INSTRUMENT_NOT_FOUND`(404)가 그대로 내려간다. `AI_UPSTREAM_*`에는 `requestId`가 없다 (§10.4) |
| POST | `/ai/chat` | 위와 동일 | |
| POST | `/ai/portfolio/diagnosis` | 위와 동일 | 보유 종목 0개는 AI 서버의 `INSUFFICIENT_DATA`(409)이며 정상 거절이다 |
| POST | `/ai/portfolio/attribution` | 위와 동일 | |
| POST | `/ai/orders/preview` | 위와 동일 | |
| GET | `/ai/briefing` | 위와 동일 | |
| POST | `/ai/feedback` | 위와 동일 | 모르는 `requestId`의 처리는 AI 서버 몫이다 (프론트 contracts P14) |
| GET | `/internal/v1/portfolio` · `/internal/v1/trades` | `AUTH_INVALID_TOKEN` · `RESOURCE_NOT_FOUND` | `X-Internal-Token` 누락·불일치 → `401 AUTH_INVALID_TOKEN`. `X-User-Id`에 해당하는 사용자 없음 → `404 RESOURCE_NOT_FOUND`. 사용자 JWT 인증은 적용되지 않는다 |

**표에 없는 코드는 그 엔드포인트에서 나오지 않는다.** 구현 중 새 사유가 생기면 이 표와 §11 목록을 먼저 고치고 코드를 붙인다. 백엔드 테스트(`ErrorCodeContractTest`)가 enum 전체와 §11 목록의 일치를 검사한다.

---

## 12. 엔드포인트 요약

| 분류 | Method | Path | 멱등성 키 |
|---|---|---|:---:|
| 인증 | POST | `/api/v1/auth/kakao` | |
| 인증 | POST | `/api/v1/auth/refresh` | |
| 인증 | POST | `/api/v1/auth/logout` | |
| 인증 | GET | `/api/v1/users/me` | |
| 계좌 | GET | `/api/v1/account` | |
| 충전 | GET | `/api/v1/deposits/limit` | |
| 충전 | POST | `/api/v1/deposits/ready` | |
| 충전 | GET | `/api/v1/deposits/kakao/approval` | 무인증 |
| 충전 | POST | `/api/v1/deposits/{paymentId}/mock-approve` | |
| 충전 | POST | `/api/v1/deposits/confirm` | `paymentKey` |
| 출금 | POST | `/api/v1/withdrawals` | **필수** |
| 종목 | GET | `/api/v1/stocks/search` | |
| 종목 | GET | `/api/v1/stocks/{stockCode}` | |
| 종목 | GET | `/api/v1/stocks/{stockCode}/candles` | |
| 종목 | GET | `/api/v1/stocks/{stockCode}/price` | |
| 종목 | GET | `/api/v1/stocks/prices` | |
| 최근 본 | GET/DELETE | `/api/v1/stocks/recent` | |
| 최근 검색 | GET/DELETE | `/api/v1/stocks/search/recent` | |
| 관심 종목 | GET/POST/DELETE | `/api/v1/watchlist` | |
| 주문 | POST | `/api/v1/orders` | **필수** |
| 주문 | GET | `/api/v1/orders/available` | |
| 잔고 | GET | `/api/v1/portfolio` | |
| 내역 | GET | `/api/v1/transactions` | |
| AI 중계 | POST | `/api/v1/ai/stocks/{stockCode}/analysis` | |
| AI 중계 | POST | `/api/v1/ai/chat` | |
| AI 중계 | POST | `/api/v1/ai/portfolio/diagnosis` | |
| AI 중계 | POST | `/api/v1/ai/portfolio/attribution` | |
| AI 중계 | POST | `/api/v1/ai/orders/preview` | |
| AI 중계 | GET | `/api/v1/ai/briefing` | |
| AI 중계 | POST | `/api/v1/ai/feedback` | |
| AI 내부 | GET | `/internal/v1/portfolio` | |
| AI 내부 | GET | `/internal/v1/trades` | |

---

## 13. 확정이 필요한 항목

응답 봉투·에러 형식·토큰 정책·페이징 크기·멱등성 규칙은 v0.2에서 확정되어 이 목록에서 빠졌다.

| # | 항목 | 의존 | 영향 |
|---|---|---|---|
| 1 | 시세 갱신 주기(폴링 주기·TTL)와 웹소켓 전환 시점 — 방식 자체는 5.6에서 STOMP + 폴링 폴백으로 확정 | `[S0-2]` | 시세 관련 전부 |
| 2 | `stale` 허용 시간과 주문 차단 기준 | `[S0-3]` | 5.4 규칙의 판정 시간, 7.2의 3번 단계 |
| 3 | 분봉 도입 시 `candles`의 `interval` 확장과 `minute_candle` 신설 — 주봉·월봉은 v0.8.4에서 해소(이슈 #37) | `[S0-4]` | 5.3 |
| 4 | AI 응답에 파생 지표를 백엔드가 포함할지 | `[S0-5]` | 9.1, 9.2 |
| 5 | 5.6 수치 기본값(하트비트 10초·3회, TTL 30초, 폴링 5초/3초, KIS 순회 3초)의 실측 검증 — 채널과 기본값은 확정, 관계식 유지 하에 수치만 조정 가능 | `[S0-1]`·`[S0-2]` | 5.6 |

다건 시세 조회 최대 건수는 **50건으로 확정**되어 목록에서 제거했다 (§5.5 — KIS 등록 한도는 백엔드 수집 계층이 흡수하므로 무관).

v0.8 에서 두 항목이 확정되어 목록에서 빠졌다.

- **AI 중계 upstream 상태 코드 통과 범위**(구 6번) — 401·403 → `502 AI_UPSTREAM_UNAVAILABLE`,
  429 → `503 AI_UPSTREAM_RATE_LIMITED`(신설), 나머지는 그대로 통과로 확정 (§10.4).
- **`ORDER_PRICE_CHANGED` 의 판정 조건**(구 7번) — **선택지 (b) 폐기로 확정.** 예수금 부족은 전부
  `ORDER_INSUFFICIENT_CASH` 다. 요청에 `expectedPrice` 를 추가하는 (a) 를 기각한 이유는, 시장가 주문에서
  "확인 화면의 가격"과 "체결가"가 다른 것은 <b>정상</b>이고 사용자가 할 일도 같기 때문이다 —
  어느 쪽이든 다시 시도하거나 수량을 줄인다. 코드를 나눠도 화면이 달라지지 않는다.

---

## 14. 다음 작업

1. 위 13장 항목을 Sprint 0(8/26 결정 회의)에서 확정한다.
2. 확정 후 **OpenAPI 3.0 문서(`openapi.yaml`)로 옮긴다.** 프론트엔드 Mock(MSW)은 이 문서가 아니라 OpenAPI 문서를 기준으로 생성한다.
3. 예시 응답 JSON을 엔드포인트별로 고정해 Mock과 통합 테스트가 같은 값을 쓰게 한다.
