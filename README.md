<div align="center">

<img src="https://raw.githubusercontent.com/Team-FINCH/finch-frontend/master/public/brand/finch-logo.png" alt="FINCH" width="160" />

# FINCH

**내 계좌를 읽고, 숫자로 설명하는 AI 투자 비서**

실제 시세로 국내 주식을 모의 매매하고, AI 가 내 원장과 시장 데이터를 근거로 "무슨 일이 있었는지"를 설명합니다.

[**서비스 바로가기 → finchapp.org**](https://finchapp.org)

![React](https://img.shields.io/badge/React_19-20232A?logo=react&logoColor=61DAFB)
![Spring Boot](https://img.shields.io/badge/Spring_Boot_4-6DB33F?logo=springboot&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL_+_pgvector-4169E1?logo=postgresql&logoColor=white)
![Kubernetes](https://img.shields.io/badge/k3s-FFC61C?logo=k3s&logoColor=black)

</div>

## 한눈에 보기

| | |
|---|---|
| **기간** | 2026.08 – 2026.09 (6주) |
| **팀** | 5명 — Frontend 2 · Backend 1 · AI 1 · Infra 1 |
| **규모** | 커밋 1,400+ · 자동화 테스트 1,300+ (AI 708 · Backend 654) |
| **운영** | 실서비스 배포 (k3s, Cloudflare Tunnel, 설치형 PWA) |

## 풀고 싶었던 문제

주식 앱은 **"얼마가 됐는지"** 는 보여 주지만 **"왜 그렇게 됐는지"** 는 알려 주지 않습니다.
그렇다고 LLM 에게 그냥 물으면 **숫자를 지어내고, 투자를 권유합니다.** 금융 서비스에서는 둘 다 치명적입니다.

## FINCH 의 답

**1. AI 는 숫자를 쓰지 않습니다.**
수익률·비중·손익은 계산 엔진이 만들고, AI 는 `{{자리표시자}}` 로만 문장을 씁니다. 서버가 엔진 값으로 치환합니다.
AI 가 직접 쓴 숫자는 자동 검사에서 폐기됩니다.

**2. 출력은 내보내기 전에 10가지 자동 검사를 거칩니다.**
지어낸 숫자, 근거 없는 인용, 인과 단정, 매수·매도 권유를 잡아 재생성하거나 차단합니다. 틀린 답을 내느니 답하지 않습니다.

**3. AI 는 돈을 움직일 수 없습니다.**
원장은 백엔드만 쓰고, AI 는 읽기 전용 내부 API 로만 접근합니다. 잔고를 바꾸는 경로가 구조적으로 없습니다.

**4. 모든 설명에 근거가 붙습니다.**
뉴스·공시를 매일 수집해 벡터 검색(RAG)으로 찾고, 답변마다 출처 각주를 답니다.

## 주요 기능

| 기능 | 설명 |
|---|---|
| 🤖 **AI 채팅** | "오늘 내 주식 왜 떨어졌어?" — 도구를 골라 포트폴리오·수익률 분해·뉴스를 조회하고 답합니다 |
| 📰 **데일리 브리핑** | 매일 아침 내 보유 종목에 관련된 소식을 골라 요약합니다 |
| 📊 **포트폴리오 진단** | 집중도·변동성 등 위험 지표를 계산하고 쉬운 말로 풀어 줍니다 |
| 📈 **수익률 원인 분석** | 시장 대비 초과수익을 종목별 기여도로 나눠 보여 줍니다 |
| 🔎 **종목 분석** | 공시·재무·뉴스를 근거로 종목을 요약합니다 |
| 🛒 **모의 매매** | 실시간 시세(한국투자증권 OpenAPI)로 시장가 매수·매도, 카카오페이 테스트 결제로 충전 |

## 아키텍처

```
  사용자 (PWA)
      │ HTTPS
  Cloudflare Tunnel ─► k3s Ingress
      ├─ Frontend   React 19 · Vite · TypeScript
      └─ Backend    Spring Boot 4 · Java 21 ── PostgreSQL · Redis
             │                                  └ 카카오 · 카카오페이 · 한국투자증권
             └─ AI 중계 ─► AI   FastAPI · Python 3.12 ── PostgreSQL + pgvector
                                                  └ OpenAI · DART · KRX · ECOS · 네이버 뉴스
```

프론트는 AI 를 직접 부르지 않습니다. 인증 주체를 백엔드 하나로 두기 위해 모든 AI 호출은 백엔드가 중계합니다.

## 저장소

| 저장소 | 내용 |
|---|---|
| [finch-docs](https://github.com/Team-FINCH/finch-docs) | 프로젝트 소개, 기획·명세, 설계 문서 |
| [finch-frontend](https://github.com/Team-FINCH/finch-frontend) | React 19 PWA |
| [finch-backend](https://github.com/Team-FINCH/finch-backend) | Spring Boot 4 · 원장 · 주문 · 결제 |
| [finch-ai](https://github.com/Team-FINCH/finch-ai) | FastAPI · LLM 에이전트 · RAG · 가드레일 |
| [finch-infra](https://github.com/Team-FINCH/finch-infra) | k3s · Helm · CI/CD · 관측 |

## 팀

| 이름 | GitHub | 역할 |
|---|---|---|
| 유승주 | [@TrossYou](https://github.com/TrossYou) | Frontend |
| 안서진 | [@xxj15](https://github.com/xxj15) | Frontend |
| 서동혁 | [@weeast1521](https://github.com/weeast1521) | Backend |
| 김세민 | [@tpals0409](https://github.com/tpals0409) | AI |
| 장준환 | [@prgmd](https://github.com/prgmd) | Infra |

---

<sub>상세 기획·범위·문서 지도는 [docs/PROJECT.md](docs/PROJECT.md) 에 있습니다.</sub>
