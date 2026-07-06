# Toss Trading Assistant

토스증권 Open API 기반 **개인용 트레이딩 보조 프로그램**.
단순 자동매매를 넘어, 아래 7가지를 **한 화면(대시보드)** 에서 관리하는 것이 목표다.

| # | 기능 | 데이터 출처 |
|---|------|------------|
| 1 | 보유정보 (잔고·보유종목·체결내역) | 토스 Open API — Account |
| 2 | 현재가 (국내 KRX + 미국) | 토스 Open API — Market Data |
| 3 | 평가손익 (원화 환산 포함) | 계산 (보유단가 × 현재가 × 환율) |
| 4 | 자동매수/매도 (조건 기반 전략 실행) | 토스 Open API — Order |
| 5 | 로그 기록 (주문·시세·에러·감사) | 로컬 SQLite + 파일 |
| 6 | 설정 저장 (전략·감시종목·리스크 한도) | 로컬 config (YAML) + 시크릿 분리 |
| 7 | 기업 재무제표 평가 (PER·ROE·부채비율 등) | **DART OpenAPI** (토스 API 미제공 영역) |

## 문서

- [docs/requirements.md](docs/requirements.md) — 기능별 요구사항 구체화 (무엇을 만들지)
- [docs/architecture.md](docs/architecture.md) — 시스템 아키텍처 & 프로세스 설계 (어떻게 만들지)
- [docs/strategy.md](docs/strategy.md) — 매매 규칙 설계 v0.1 (미국 스윙, 1% 리스크, 연 80% 목표의 분해)
- [docs/roadmap.md](docs/roadmap.md) — 단계별 개발 로드맵 (어떤 순서로 만들지)

확정된 방향: **웹 UI (FastAPI + React)** · **미국 시장 우선** · **스윙 매매** · **1회 리스크 = 자본의 1%**

## 전제 조건

- 토스증권 계좌 + [developers.tossinvest.com](https://developers.tossinvest.com/docs) 에서 Open API 신청, `CLIENT_ID` / `CLIENT_SECRET` 발급
- 재무제표용 [DART OpenAPI](https://opendart.fss.or.kr) 인증키 발급 (무료)
- Python 3.12+ (권장 스택은 architecture 문서 참고)

## 안전 원칙 (전 단계 공통)

1. **읽기(조회) → 쓰기(주문) 순서로 개발한다.** 주문 API는 마지막에 붙인다.
2. **주문은 기본 dry-run.** 실주문은 명시적 플래그 + 리스크 체크 통과 시에만 나간다.
3. **모든 주문 시도는 실행 전에 로그부터 남긴다.** (감사 추적)
4. **킬스위치**: 일일 손실 한도·주문 횟수 한도 초과 시 자동매매 전체 정지.

> ⚠️ 본 프로그램은 개인 투자 보조 도구이며, 수익을 보장하지 않는다. 모든 투자 판단과 결과의 책임은 사용자 본인에게 있다.
