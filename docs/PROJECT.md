# FINCH

**내 계좌와 성향을 읽고 투자 판단을 돕는 AI 비서**다. 실제 시세로 국내 주식을 매매하고, 그 결과를 AI 가 사용자의 원장과 시장 데이터에 근거해 설명한다. 실제 금전 이동은 없고 가상 예수금으로 거래한다.

| | |
|---|---|
| 서비스 | **https://finchapp.org** (설치형 PWA) |
| API 문서 | https://swagger.finchapp.org (Basic 인증) |

## 무엇이 다른가

**1. 원장이 단일 진실 공급원이다.** 잔고, 손익, 수익률은 저장된 값이 아니라 `ledger_entry` 에서 계산한 값이다. 클라이언트가 보낸 계산값을 신뢰하지 않는다.

**2. AI 가 원장을 쓰지 못한다.** 읽기 전용 내부 계약(`/internal/v1`)으로만 접근한다. AI 가 잔고를 바꿀 경로는 구조적으로 존재하지 않는다.

**3. AI 가 수치를 지어내지 못한다.** 모든 숫자는 엔진이 계산해 `{{자리표시자}}` 로 넘기고 서버가 치환한다. 출력단 가드레일 9종이 치환되지 않은 자리표시자, 직접 쓴 숫자, 근거 없는 인용을 잡아 막는다.

**4. 되돌리는 기능을 넣지 않았다.** 계좌 초기화도 충전 취소도 없다. 실제 투자 서비스에 없는 편의를 넣으면 연습용 도구로 읽힌다.

## 기능

**투자** — 카카오 로그인, 카카오페이 테스트 결제로 예수금 충전, 시장가 매수와 매도, 보유 종목과 평가손익, 매매 내역

**시세** — 한국투자증권 OpenAPI 실시간 수신, 일봉과 주봉과 월봉 차트, 관심 종목

**AI 6종** — 응답은 전부 근거를 함께 낸다

| 기능 | 화면 |
|---|---|
| 종목 분석 | 종목 상세 |
| Ask My Portfolio (채팅) | 채팅 |
| 포트폴리오 진단 | 포트폴리오 |
| 수익률 원인 분석 | 홈, 포트폴리오 |
| 주문 전 점검 | 주문 |
| 데일리 브리핑 | 홈 |

## 범위 경계

평가 전에 알아두면 좋은 것들이다.

| | |
|---|---|
| **서비스 종목 30개** | 실시간 시세 세션 한도(41건) 때문에 종목을 30개로 골라 뒀다. **그 밖의 종목은 검색에 나오지 않는다** — 고장이 아니다. 목록은 `backend/src/main/resources/application.yaml` 의 `finch.universe.codes` |
| **시장가 전용** | 지정가와 미체결 관리는 범위 밖이다 |
| **거래 시간** | 평일 09:00~15:30, 16:00~20:00 (KRX 애프터마켓 포함). 그 밖에는 주문이 막힌다 |
| **모의 결제** | 카카오페이 테스트 CID 다. 결제창과 승인 흐름은 진짜지만 실제 금전 이동이 없다 |
| **수수료와 세금** | 적용하지 않는다 |

## 구성

```
 사용자 브라우저 (PWA)
      │  HTTPS
      ▼
  Cloudflare Tunnel ──► k3s Ingress (nginx)
      │
      ├─ frontend   React 19 + Vite 8 + TypeScript (nginx 정적 서빙)
      │
      └─ backend    Spring Boot 4.1 / Java 21
             ├── PostgreSQL 17   원장, 계정, 종목
             ├── Redis           멱등성 키, 캐시
             ├── 외부 API        카카오, 카카오페이, 한국투자증권
             │
             └── AI 중계 ──► ai   FastAPI / Python 3.12
                       ├── PostgreSQL + pgvector   문서, 임베딩, 시세
                       └── 외부 API   GMS, DART, KRX, ECOS, 네이버
```

**프론트는 AI 서버를 직접 호출하지 않는다.** 모든 AI 호출은 백엔드가 중계한다 — 인증 주체를 하나로 두고, 신뢰 헤더(`X-User-Id`) 위조 경로를 막기 위해서다.

**운영은 k3s 다.** Helm 차트로 앱(`finch`)과 관측 스택(`finch-observability`)을 별도 릴리스로 올린다. Jenkins 가 master 머지마다 변경 파트를 감지해 이미지를 빌드하고 `helm upgrade --install --atomic` 으로 배포한다. Docker Compose 구성은 롤백 경로로 남아 있다.

**관측** — Prometheus, Grafana, Loki, Alloy, node-exporter, kube-state-metrics, json-exporter

## 디렉터리

| | |
|---|---|
| `backend/` | Spring Boot. 원장, 인증, 시세 중계, AI 중계, 결제 |
| `ai/` | FastAPI. 분석 생성, RAG 검색, 근거 적재, 가드레일 |
| `frontend/` | React. 화면, 라우팅, 상태 관리, MSW 목 서버 |
| `infra/` | Docker, Helm 차트, Jenkins, 관측 스택, 운영 스크립트 |
| `docs/` | 명세와 계약 |
| `prototype/` | 화면 프로토타입 |

**파트 디렉터리 소유권은 ADR-0002 를 따른다.** 다른 파트의 디렉터리를 직접 수정하지 않는다.

## 문서

**어느 것을 먼저 읽어야 하는지**가 문서 수보다 중요하다.

| 알고 싶은 것 | 읽을 것 |
|---|---|
| 무엇을 만들기로 했나 | `docs/spec/featureSpec.md` (기능 명세) |
| 무엇이 어디까지 됐나 | `docs/spec/requirementsSpec.md` (요구사항별 구현 상태) |
| 백엔드 API 계약 | `docs/api/apiSpec.md` — **이것이 정본이다** |
| AI API 계약 | `ai/docs/api-spec.md` — **이것이 정본이다** |
| 데이터 모델 | `docs/erd/erd.md` |
| 화면 설계 | `docs/design/screenDesign.md`, `frontend/docs/ia.md`, `frontend/docs/design.md` |
| 인프라와 배포 | `infra/README.md`, `infra/k8s/README.md` |
| 커밋과 MR 규칙 | `docs/convention/gitConvention.md`, `docs/convention/mrConvention.md` |

> **`docs/api/aiApiSpec.md` 는 2026-08-20 협의용 초안이다.** 이름이 비슷해 헷갈리기 쉬운데, 살아 있는 AI 계약은 `ai/docs/api-spec.md` 다.

## 로컬에서 돌리기

각 파트가 독립적으로 뜬다. 자세한 것은 파트별 README 를 본다.

```bash
# frontend — MSW 목 서버가 붙어 백엔드 없이도 화면이 전부 돈다
cd frontend && npm ci && npm run dev

# backend — Testcontainers 가 PostgreSQL 과 Redis 를 자동으로 띄운다
cd backend && ./gradlew bootTestRun

# ai — .env 작성이 선행이다 (ai/README.md 참고)
cd ai && pip install -r requirements.txt && uvicorn app.api.main:app --reload
```

전체 스택을 한 번에 띄우려면 `infra/README.md` 의 '서버 첫 구축 순서' 를 따른다.

## 팀

| 파트 | 담당 범위 |
|---|---|
| Backend | 원장, 인증, 시세 중계, AI 중계, 외부 결제 연동 |
| AI | 분석 생성, RAG 검색, 근거 데이터 적재, 가드레일 |
| Frontend | 화면, 라우팅, 상태 관리, 목 서버 |
| Infra | 서버, 컨테이너, CI/CD, 쿠버네티스, 관측, 배치 스케줄 |
