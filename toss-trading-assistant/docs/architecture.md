# 시스템 아키텍처 & 프로세스 설계

## 1. 기술 스택 (권장)

| 영역 | 선택 | 이유 |
|------|------|------|
| 언어 | Python 3.12+ | 금융 데이터 생태계(pandas), 빠른 반복 개발 |
| HTTP | `httpx` | async 지원, 타임아웃·재시도 제어 용이 |
| 모델/검증 | `pydantic` v2 | API 응답 정규화 + 설정 스키마 검증 겸용 |
| 스케줄러 | `APScheduler` | 폴링 주기·장 시간 크론 관리 |
| 저장소 | SQLite (`sqlmodel`) | 단일 파일, 백업 쉬움, 개인용에 충분 |
| UI | **웹 (확정)** — FastAPI 백엔드 + React/Next.js 프론트 | 사용자 결정. WebSocket(또는 SSE)으로 시세·포지션 실시간 푸시 |

웹 UI 구성: FastAPI가 서비스 계층을 REST(`/portfolio`, `/rules`, `/logs`, `/settings`) + WebSocket(`/stream` — 시세·주문 이벤트)으로 노출하고, React 대시보드가 이를 소비한다. UI는 서비스 계층 함수만 호출하고 비즈니스 로직을 갖지 않는다 (아래 계층 규칙). 미국장 시간 동안 상시 구동해야 하므로(스톱 감시, strategy.md §4) 서버 프로세스(엔진+API)와 브라우저(뷰)가 분리된 웹 구조가 요구사항과도 맞다.

토스 API 클라이언트는 공식 `openapi.json`으로 `openapi-generator` 자동 생성 후 얇은 래퍼를 씌우는 방식 권장 (스펙 변경 추적이 쉬움).

## 2. 계층 구조

```
┌────────────────────────────────────────────────┐
│  UI (Streamlit 대시보드)                         │  표시·입력만. 로직 없음
├────────────────────────────────────────────────┤
│  Service Layer                                  │
│  · PortfolioService   (잔고+시세+환율 → 손익)      │
│  · StrategyEngine     (규칙 평가 → 주문 의도 생성)  │
│  · RiskManager        (주문 의도 검증·킬스위치)     │
│  · OrderService       (주문 실행·체결 추적)        │
│  · FinancialAnalyzer  (DART → 지표·등급)          │
├────────────────────────────────────────────────┤
│  API Client Layer                               │
│  · TossClient  (auth·시세·잔고·주문, rate limiter) │
│  · DartClient  (재무제표, 캐시)                    │
├────────────────────────────────────────────────┤
│  Storage                                        │
│  · SQLite: 주문/감사로그·손익스냅샷·규칙 실행이력     │
│  · config.yaml + .env(시크릿)                    │
└────────────────────────────────────────────────┘
```

계층 규칙:
- UI → Service만 호출. Service → Client/Storage만 호출. 역방향·건너뛰기 금지
- **주문 API 호출은 `OrderService` 단 한 곳**에만 존재 (dry-run 분기·감사 로그를 한 지점에서 강제)
- `StrategyEngine`은 주문을 직접 내지 않고 `OrderIntent`(주문 의도)만 생성 → `RiskManager` 통과분만 `OrderService`로 전달

### 디렉터리 스켈레톤

```
src/
  clients/    toss_client.py  dart_client.py  rate_limiter.py  auth.py
  services/   portfolio.py  strategy_engine.py  risk_manager.py
              order_service.py  financial_analyzer.py
  models/     position.py  quote.py  order.py  rule.py  financials.py
  storage/    db.py  repositories.py
  config/     settings.py (pydantic-settings)  config.yaml
  api/        app.py  routes/  ws.py      # FastAPI
  scheduler/  jobs.py
web/          # React/Next.js 대시보드
tests/
```

## 3. 핵심 프로세스 설계

### 3-1. 메인 루프 (장중, 폴링 기반)

```
[APScheduler, 5~10초 주기, 장중에만]
        │
        ▼
① MarketDataCollector: 보유+감시종목 시세 일괄 조회 → 캐시 갱신
        │                (UI와 전략 엔진이 같은 캐시를 읽음)
        ▼
② StrategyEngine: 활성 규칙 × 최신 시세/포지션 평가
        │   충족 없음 → 다음 틱 대기
        │   충족 → OrderIntent 생성 (규칙명·조건값·시세 스냅샷 포함)
        ▼
③ RiskManager 검증 (순서 고정):
        킬스위치 확인 → 중복(미체결·기실행) 확인 → 금액 상한
        → 일일 횟수 → 일일 손실 한도 → 보유 비중
        │   하나라도 실패 → REJECTED 로그 + 사유, 끝
        ▼
④ OrderService:
        감사 로그 선기록 (INTENT 상태)
        → dry-run이면 SIMULATED 기록 후 끝
        → live면 주문 API 호출 → 응답으로 로그 갱신
        ▼
⑤ 체결 추적 (별도 주기 10~30초):
        미체결 주문 상태 폴링 → FILLED 시 포지션 갱신 + 규칙 실행 카운트 차감
```

설계 포인트:
- **①~④는 한 틱 안에서 동기적으로 완결** (시세 시점과 주문 판단 시점의 괴리 최소화)
- 틱 처리 중 다음 틱이 오면 스킵 (재진입 금지 — 중복 주문의 주요 원인)
- 장 시간 판정은 시장정보 API의 휴장일 + 거래소별 장 시간 테이블로 스케줄러 레벨에서 차단

### 3-2. 주문 상태 머신

```
INTENT ──리스크 통과──▶ SUBMITTED ──▶ PARTIALLY_FILLED ──▶ FILLED
   │                      │                                  
   ├──리스크 거부──▶ REJECTED_RISK   ├──API 거부──▶ REJECTED_API
   └──dry-run──▶ SIMULATED          └──취소──▶ CANCELLED
```

모든 상태 전이는 SQLite 감사 로그에 타임스탬프와 함께 기록. "REJECTED_RISK가 왜 났는지"까지 사유 문자열 저장.

### 3-3. 인증 토큰 수명주기

```
기동 → 토큰 발급 → 메모리 보관 (디스크 저장 금지)
    → 만료 60초 전 백그라운드 갱신
    → 갱신 실패: 조회는 캐시로 유지(스테일 표기), 전략 엔진은 일시정지
    → 401 응답: 1회 즉시 재발급 후 재시도, 재실패 시 킬스위치
```

### 3-4. Rate Limiter

- 토큰 버킷 2개: 시세용(넉넉히), 주문용(보수적 — 스펙 확인 후 한도의 50%로 설정)
- 모든 API 호출이 클라이언트 레이어에서 버킷을 통과. 429 수신 시 지수 백오프(2s→4s→8s) + 로그
- 폴링 주기 자동 계산: `종목 수 / 주기 ≤ 시세 한도 × 0.7` 위반 시 기동 경고

### 3-5. 재무제표 평가 파이프라인 (일배치/수동)

```
보유+감시종목 → 종목코드→DART corp_code 매핑(전체 기업목록 캐시)
→ 최신 보고서 재무제표 조회 → 지표 계산(시총은 토스 시세 결합)
→ 점수화·등급 → SQLite 캐시 (보고서 기준일 포함) → 대시보드 표시
```

실시간 루프와 완전 분리 — 재무 배치 실패가 트레이딩 루프에 영향 주지 않음.

### 3-6. 시작/종료 시퀀스

기동: 설정 로드·검증 → DB 마이그레이션 → 토큰 발급 → 잔고 1회 조회 → **미체결 주문·전날 킬스위치 상태 복원** → 스케줄러 시작 → UI 서빙
종료: 스케줄러 정지 → 진행 중 틱 완료 대기 → 상태 플러시. (미체결 주문은 취소하지 않고 다음 기동 시 복원이 기본)

## 4. 에이전트 스킬 활용 (개발 프로세스)

"에이전트 스킬"은 **런타임 구성요소가 아니라 개발 가속 도구**로 쓰는 것을 권장:

1. 토스 공식 `llms.txt` + `openapi.json`을 Claude Code 컨텍스트에 넣고 클라이언트 래퍼·모델 코드 생성
2. [BEOKS/tossinvest-skill](https://github.com/BEOKS/tossinvest-skill) (`npx skills add BEOKS/tossinvest-skill`): 개발 중 API 탐색·응답 확인·dry-run 주문 테스트를 자연어로 수행
3. 운영 단계에서 에이전트에게 실주문 권한을 위임하는 것은 **비권장** — 자동매매는 결정적(deterministic) 규칙 엔진으로 두고, 에이전트는 분석·리포트 생성까지만

## 5. 테스트 전략

- 단위: StrategyEngine·RiskManager는 **API 없이 순수 함수로 테스트 가능**하게 설계 (시세·포지션을 인자로 주입)
- 통합: TossClient는 응답 픽스처(mock) 기반. 실 API 스모크 테스트는 조회 전용으로 분리
- 시나리오: "조건 충족 → 리스크 거부", "부분 체결", "429 백오프", "토큰 만료 중 주문" 등을 픽스처로 재현
- **리플레이 모드**: 저장한 시세 이력으로 전략을 재실행해 규칙 검증 (간이 백테스트, 2단계)
