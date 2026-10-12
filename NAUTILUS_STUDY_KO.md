# NautilusTrader 전수조사 & 수익화 연구 노트 (한국어)

> 대화 정리본 · 작성일 2026-10-12

## 📎 저장소 / 참고 주소

| 구분 | 주소 |
| :-- | :-- |
| 내 포크 저장소 | <https://github.com/bmshin94/nautilus_trader> |
| 원본(공식) 저장소 | <https://github.com/nautechsystems/nautilus_trader> |
| 공식 문서 | <https://nautilustrader.io/docs/> |
| 공식 웹사이트 | <https://nautilustrader.io> |
| PyPI 패키지 | <https://pypi.org/project/nautilus_trader/> |
| crates.io (코어) | <https://crates.io/crates/nautilus-core> |
| 커뮤니티 (Discord) | <https://discord.gg/NautilusTrader> |
| 상표 정책 문의 | <legal@nautechsystems.io> |
| 보안 신고 | <security@nautechsystems.io> |

- 분석 기준 버전: `v2.0.0rc7` (v2 릴리스 후보)
- 라이선스: **LGPL-3.0-only** + 별도 상표 정책(`TRADEMARK.md`)
- 개발사: Nautech Systems Pty Ltd (호주, ABN 88 609 589 237)

> ⚠️ **면책**: 이 문서는 학습·아이디어 정리용이며 투자 조언도, 법률 자문도 아니다.
> 실제 자금 투입 전 테스트넷 → 소액 → 확대 순서를 반드시 지킬 것. 원금 손실 가능.

---

## 1. 이게 뭐하는 건가 (전수조사 결과)

**한 줄 정의: 연구(백테스트)부터 실거래까지 하나의 이벤트 기반 아키텍처로 처리하는 오픈소스
알고리즘 트레이딩 엔진.** 코어는 Rust, 전략 작성은 Python(또는 Rust).

### 1.1 규모 (실측)

| 항목 | 수치 |
| :-- | :-- |
| Rust 크레이트 | 25개 |
| Rust 소스 | `.rs` 2,813개 파일 / 약 218만 줄 |
| Python 소스 | 약 10.9만 줄 |
| 문서 | `.md` 214개 |
| 거래소/데이터 어댑터 | 18개 디렉터리 (통합 17개 + sandbox) |
| 지원 플랫폼 | Linux(x86_64/ARM64), macOS(ARM64), Windows(x86_64) |
| 요구 버전 | Rust 1.99.0+ / Python 3.13–3.14 |

### 1.2 폴더 구조

```
nautilus_trader/
├── crates/              # Rust 코어 (25 크레이트)
│   ├── core/            기초 타입, UUID, 나노초 시간, 메시지
│   ├── model/           도메인 모델: Price / Quantity / Money / Order / Position / Instrument
│   ├── common/          Clock, Logger, Cache, 컴포넌트 생명주기
│   ├── data/            DataEngine (시세 수집·배분)
│   ├── execution/       ExecutionEngine (주문 집행·체결 관리)
│   ├── risk/            RiskEngine (주문 전 리스크 차단)
│   ├── portfolio/       포지션·손익(PnL)·잔고 집계
│   ├── trading/         Strategy / Actor 베이스
│   ├── backtest/        백테스트 엔진 + 거래소 시뮬레이터
│   ├── live/            실거래 노드 (LiveNode)
│   ├── indicators/      기술지표 (average / momentum / volatility / ratio / book)
│   ├── analysis/        성과 분석 (샤프, 소르티노, 승률, MDD 등)
│   ├── persistence/     Parquet 데이터 카탈로그
│   ├── event_store/     이벤트 소싱 (전 이벤트 기록·재생)
│   ├── infrastructure/  Redis / PostgreSQL 백엔드
│   ├── network/         WebSocket / HTTP / 소켓 클라이언트
│   ├── serialization/   MessagePack, Arrow
│   ├── cryptography/    Ed25519 서명, HMAC (거래소 인증)
│   ├── system/          NautilusKernel (전체 오케스트레이션)
│   ├── pyo3/            Rust ↔ Python 바인딩
│   ├── plugin/          동적 플러그인(cdylib) 로딩 계약 + nautilus_plugin! 매크로
│   ├── cli/             커맨드라인 도구
│   ├── macros/          매크로
│   ├── testkit/         테스트 도구 + ExecTester 등 내장 전략
│   └── adapters/        거래소 연결 모듈 18개
├── python/              # PyPI 패키지 (python/.venv 를 환경으로 사용)
├── docs/                # concepts / tutorials / how_to / integrations / api_reference
├── examples/            # backtest 15개 + 거래소별 live 예제 + _template
├── test_data/           # 샘플 시세 (binance, bybit, fxcm, databento, tardis, xcme ...)
├── scripts/  Makefile   # 빌드·검증 자동화
└── AGENTS.md / AI_POLICY.md / CONTRIBUTING.md / TRADEMARK.md / SECURITY.md
```

### 1.3 아키텍처 (NautilusKernel 내부)

```
거래소 ──시세──▶ DataEngine ──▶ MessageBus ──▶ Strategy / Actor (내 전략)
                                   │                   │
                                 Cache             주문 명령
                                   │                   ▼
거래소 ◀──주문── ExecutionEngine ◀── RiskEngine (한도 초과 주문 차단)
                      │
                      └──▶ Portfolio (포지션·손익 실시간 집계)
          Cache / MessageBus ◀──▶ 선택적 백엔드 (Redis, PostgreSQL)
```

적용된 설계 기법: 도메인 주도 설계(DDD), 이벤트 기반 아키텍처, 메시징 패턴(pub/sub ·
req/res · point-to-point), 포트앤어댑터(헥사고날), crash-only 설계.

### 1.4 핵심 특징

| 특징 | 설명 |
| :-- | :-- |
| 백테스트 = 실거래 동일 코드 | 같은 전략 소스를 수정 없이 백테스트/실거래에 사용 |
| 이벤트 기반 | 틱 단위 순차 처리 → look-ahead bias(미래 참조 치팅) 구조적 차단 |
| 나노초 정밀도 | quote / trade / bar / order book / 커스텀 데이터 정밀 재현 |
| 정확한 금액 연산 | Price·Quantity·Money를 고정소수 정수로 처리 (고정밀 128bit / 표준 64bit) |
| 멀티 벤뉴·멀티 전략 | 여러 거래소·종목·전략 동시 운용, 크로스 벤뉴 전략 가능 |
| 고급 주문 | IOC/FOK/GTC/GTD/DAY/AT_THE_OPEN/AT_THE_CLOSE, post-only, reduce-only, 아이스버그, OCO/OUO/OTO |
| 공급망 보안 | cargo-vet, cargo-deny, Gitleaks, CodeQL, OSV, SLSA 프로비넌스, Sigstore 서명 SBOM |
| AI 학습 지원 | RL/ES 트레이딩 에이전트 학습이 가능한 속도 (도구는 미포함) |

### 1.5 지원 통합 (모두 `stable`)

- **CEX**: Binance, Bybit, Coinbase, Deribit, Kraken, OKX
- **DEX**: dYdX, Hyperliquid, Lighter, Lighter on Robinhood, Derive
- **전통 금융**: Interactive Brokers(주식·선물·옵션·FX), AX Exchange(파생)
- **베팅/예측시장**: Betfair, Polymarket
- **데이터 제공자**: Databento, Tardis
- **기타**: sandbox(모의 벤뉴), blockchain(온체인 데이터)
- ❗ **한국 거래소(업비트·빗썸·한국투자증권·키움) 미지원** → 수익화 기회

### 1.6 어떨 때 쓰나

| 상황 | 활용 |
| :-- | :-- |
| 전략 아이디어 검증 | 과거 데이터로 나노초 정밀 백테스트 |
| 24시간 자동매매 | 코인/선물/주식 봇 무인 운용 |
| 마켓메이킹·차익거래 | 멀티 거래소 동시 호가·가격차 포착 |
| 리스크 관리 | 주문 전 엔진 단계 한도 검증 |
| AI 트레이딩 학습 | 결정론적 고속 시뮬레이션 환경 |
| 퀀트 리서치 | Parquet 카탈로그 기반 대용량 시세 관리 |

### 1.7 나에게 주는 가치

1. 자동 수익 파이프라인의 **엔진**을 무료로 확보 (직접 만들면 수년치 작업)
2. 프로덕션급 **Rust + Python 코드 교과서** (DDD, 이벤트 소싱, PyO3, 공급망 보안)
3. **수익화 소재 창고** (아래 4장)
4. AI 에이전트의 안전한 **실행 레이어** (RiskEngine이 폭주 방어선)

---

## 2. 더 쉬운 설명

### 2.1 비유

- **F1 조립 키트**: 엔진·변속기·브레이크·안전벨트는 전부 제공됨. 내가 쓸 건 "핸들 조작법(전략)" 뿐.
- **라면집 주방**: 식재료 배달 = DataEngine / 주방 게시판 = MessageBus / 요리사 = **Strategy(내 담당)**
  / 사장 검수 = RiskEngine / 서빙 = ExecutionEngine / 금고 = Portfolio / 연습용 손님 = BacktestEngine.

### 2.2 실제로 쓰는 코드량 (`docs/tutorials/ema_cross.py` 요약)

```python
class EMACross(Strategy):
    def __init__(self, config):
        super().__init__(config)
        self.fast_ema = ExponentialMovingAverage(config.fast_ema_period)
        self.slow_ema = ExponentialMovingAverage(config.slow_ema_period)

    def on_start(self):                                   # 시작 시 1회
        self.register_indicator_for_bars(self.config.bar_type, self.fast_ema)
        self.register_indicator_for_bars(self.config.bar_type, self.slow_ema)
        self.subscribe_bars(self.config.bar_type)

    def on_bar(self, bar):                                # 봉마다 호출
        if self.fast_ema.value >= self.slow_ema.value:
            self.buy()
        else:
            self.sell()
```

WebSocket 연결·재접속·주문 추적·손익 계산은 모두 엔진이 처리한다.

### 2.3 사용 흐름 3단계

```
1단계 백테스트         →   2단계 모의(sandbox/testnet)   →   3단계 실거래(LiveNode)
과거 데이터 시뮬레이션       가짜 돈으로 실시간 검증            소액부터 실자금
```
세 단계 모두 **같은 전략 파일 하나**로 동작한다.

### 2.4 Rust + Python 조합 이유

- Rust = 초고속·안정 → **엔진**
- Python = 빠른 개발 → **전략/설정/오케스트레이션(컨트롤 플레인)**
- Python 단독이면 느리고, Rust 단독이면 전략 수정이 번거롭다.

### 2.5 한계

- **전략은 포함돼 있지 않다.** 알파(수익원)는 사용자가 만들어야 한다.
- 코딩 + 금융 지식 둘 다 필요. 초기 2~4주는 백테스트 연습 권장.
- 한국 거래소 미지원 → 직접 어댑터 개발 필요.
- v2 전환 중이라 릴리스 간 호환성 깨짐(breaking change) 가능.

---

## 3. Q&A 정리

### Q1. 설치 및 사용법

**방법 A — 권장 (바이너리 wheel)**
```bash
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -U nautilus_trader
pip install -U "nautilus_trader[docker,ib]"          # 선택 기능
# nightly: pip install -U nautilus_trader --index-url=https://packages.nautechsystems.io/simple
```
공식 wheel은 고정밀(128bit) 모드로 빌드돼 있어 Rust 툴체인이 필요 없다.

**방법 B — 소스 빌드 (이 저장소 직접 개발)**
```bash
make install          # 또는 make install-debug
make format           # 포맷
make pre-commit       # 린트 + 타입체크
make pytest           # 파이썬 테스트
make cargo-test       # 러스트 테스트
make pre-flight       # 전체 검증 (가장 확실)
```
사전 준비물: Rust 1.99+, uv, clang, cmake, make.
환경 규칙(`AGENTS.md`): 가상환경은 **`python/.venv`** 사용. 루트 `.venv` 금지.

**첫 실행**
```bash
python examples/backtest/crypto_ema_cross_ethusdt_trade_ticks.py
```
스크립트 흐름: `BacktestEngine` 생성 → `add_venue`(시드·수수료 모델) → `add_instrument` →
`add_data` → `add_strategy` → `run()`.

**학습 로드맵**

| 주차 | 할 일 | 파일 |
| :-- | :-- | :-- |
| 1주 | 설치 + 예제 실행 | `docs/getting_started/quickstart.py` |
| 1주 | 고수준 백테스트 | `docs/getting_started/backtest_high_level.py` |
| 2주 | 저수준 백테스트 | `docs/getting_started/backtest_low_level.py` |
| 2주 | 전략 작성 | `docs/concepts/strategies.md`, `docs/tutorials/ema_cross.py` |
| 3주 | 데이터 카탈로그 | `docs/how_to/loading_external_data.py` |
| 3주 | 오더북 전략 | `docs/tutorials/orderbook_imbalance.py` |
| 4주 | 모의 거래 | `examples/live/sandbox/` |
| 5주+ | 테스트넷 실거래 | `examples/live/binance/exec_tester.py` (`DRY_RUN=True`) |

### Q2. 플러그인 / 스킬 / MCP 중 무엇인가

**셋 다 아니다. 독립 실행 라이브러리 + 프레임워크다.**

| 구분 | 해당 여부 |
| :-- | :-- |
| 플러그인(타 앱 확장) | ❌ |
| Claude 스킬(.md 지침) | ❌ |
| MCP 서버 | ❌ |
| **라이브러리 / 프레임워크** | ✅ |

배포 형태: PyPI 패키지, crates.io 크레이트 25개, Docker 이미지(Sigstore 서명).

"플러그인"이라는 말이 세 맥락에서 등장한다:
1. **어댑터** — 거래소 연결 모듈. `examples/live/_template/`에 신규 어댑터 템플릿이 있다.
2. **`crates/plugin`** — 실제 동적 플러그인 시스템. 독립 컴파일된 Rust cdylib이 호스트에
   자기 신원을 알리는 공개 계약으로, ABI 버전이 맞지 않으면 로딩을 거부한다.
   → **비공개 전략을 .so/.dll로 배포하는 길**이 열려 있다.
3. **직접 MCP 서버로 감쌀 수 있다** — 공식 기능은 아니며 사용자가 만드는 영역(수익화 아이디어 3).

### Q3. API 토큰이 필요한가

| 용도 | 필요 여부 |
| :-- | :-- |
| 백테스트(과거 데이터) | ❌ 불필요 |
| 공개 시세 구독 | ⭕ 대체로 불필요 |
| **주문 / 잔고 조회** | ✅ 필수 |
| 유료 데이터(Databento, Tardis) | ✅ 필수 |
| DEX 거래(dYdX, Hyperliquid, Derive) | ✅ 지갑 **개인키** |

코드에서 확인된 환경변수 (일부):
```
BINANCE_API_KEY / BINANCE_API_SECRET
BINANCE_TESTNET_API_KEY / BINANCE_FUTURES_TESTNET_API_KEY   # 연습용
BYBIT_API_KEY / BYBIT_DEMO_API_KEY / BYBIT_TESTNET_API_KEY
COINBASE_API_KEY / DERIBIT_API_KEY (+ TESTNET) / AX_API_KEY / KRAKEN / OKX
DYDX_PRIVATE_KEY / DYDX_TESTNET_PRIVATE_KEY                 # 지갑 개인키
DERIVE_SESSION_PRIVATE_KEY / BLOCKCHAIN_PRIVATE_KEY
DATABENTO_API_KEY / ENVIO_API_TOKEN
```

**보안 수칙**
1. 키를 코드에 하드코딩하지 말고 `.env` 사용 (`.env.example`만 커밋됨, `.gitignore` 확인)
2. 거래소 키 발급 시 **출금 권한 해제**, 거래 권한만 부여
3. IP 화이트리스트 설정
4. testnet/demo 키로 1~2주 선행 검증
5. 이 저장소는 Gitleaks가 pre-commit 단계에서 비밀 유출을 차단한다

### Q4. AI 에이전트 구축에 도움이 되는가

**매우 도움이 되지만 "AI 도구 내장"은 아니다.** README는
`AI Training: Engine fast enough to train AI trading agents (RL/ES)`를 명시하고,
`ROADMAP.md`의 Out of scope는 `Integrated hyper-parameter optimization or built-in AI/ML tooling`
을 제외한다. 즉 **엔진은 빠르게, AI 도구는 사용자가 붙이는** 구조다 → 외부 기회.

**활용 3가지**
1. **RL 학습 환경** — Gymnasium 스타일 래퍼를 만들어 `reset()`은 BacktestEngine 리셋,
   `step(action)`은 주문 제출 + 1틱 진행, reward는 Portfolio PnL 변화로 구성.
   이벤트 기반이라 look-ahead 치팅이 불가능해 학습 결과의 실전 신뢰도가 높다.
2. **LLM 에이전트의 실행 도구(MCP)** — `LLM(판단) → MCP 서버 → NautilusTrader(실행)`.
   LLM 환각 위험을 `RiskEngine`이 주문 전 단계에서 차단한다.
3. **전략 자동 생성·검증 루프** — LLM이 전략 코드 생성 → 백테스트 자동 실행 →
   `crates/analysis` 성과 분석 → 피드백으로 개선.

**AI 친화적 요소**: `event_store`(결정 로그 추적·재생), 결정론적 시뮬레이션(재현성),
`MessageBus`(에이전트를 Actor로 부착), `RiskEngine`(안전장치), 나노초 시계(지연 시뮬),
Parquet 카탈로그(학습 데이터 저장).

### Q5. 수익화 아이디어 요약

한국 거래소 어댑터 / 웹 대시보드 / AI MCP 서버 / 한국어 교육 / 컨설팅 / 전략 마켓플레이스
→ 상세는 4장.

### Q6. React나 PHP로 만들 수 있는가

**엔진 재구현은 비권장. 프론트엔드는 React로 완전히 가능.**

엔진 불가 사유: 초당 수십만 이벤트 처리(GC 일시정지 문제), JS `number`는 float64라 금액 계산에
부적합, 스레드 안전성, 218만 줄 + 어댑터 18개 재구현 비용. PHP는 요청-응답 모델로 상시
WebSocket 유지에 부적합.

**권장 아키텍처**
```
React + TypeScript (차트·대시보드·전략 제어)
        │  WebSocket + REST
FastAPI 브릿지 (얇게, NautilusTrader import / MessageBus 이벤트 중계)
        │
NautilusTrader 엔진 + Redis(상태 공유) + PostgreSQL
```
근거: 공식 로드맵이 **UI 대시보드를 Out of scope**로 명시 → 공식이 만들지 않는 영역.

스택 추천: React + TS + Vite + TanStack Query + Recharts 또는 TradingView Lightweight Charts /
FastAPI + websockets(엔진과 **별 프로세스**로 분리해 이벤트 루프 블로킹 방지) /
Redis Pub/Sub(`crates/infrastructure`가 이미 지원) / PostgreSQL / Docker Compose.

PHP 적합 영역: 회원·결제(Laravel), 강의 플랫폼, 마켓플레이스 쇼핑몰, 리포트 조회 페이지.

### Q7. 유튜브 강의 영상 제작이 가능한가

**가능하고, 한국어 시장은 거의 비어 있다.**

**법적 확인**

| 항목 | 결과 |
| :-- | :-- |
| LGPL-3.0-only | 교육·강의·영상 제작 자유, 유료 강의 판매 가능 |
| 상표(`TRADEMARK.md`) | "NautilusTrader", "nautilus_trader" 등을 **제목 맨 앞/프로젝트명 선두**로 사용 금지 |

- ❌ `NautilusTrader 마스터 클래스` / ✅ `파이썬 자동매매 입문 — NautilusTrader 활용하기`
- 확실히 하려면 `legal@nautechsystems.io` 사전 문의

**시리즈 커리큘럼 20편**

- 시즌 1 입문(1~6): 프로젝트 소개 데모 / 설치 / 첫 백테스트 / 성과지표 읽기(샤프·MDD·승률) /
  `ema_cross.py` 해부 / RSI 전략 직접 작성
- 시즌 2 중급(7~13): look-ahead bias와 이벤트 기반 / 수수료·슬리피지(`MakerTakerFeeModel`) /
  Parquet 카탈로그 / 오더북 임밸런스 / RiskEngine / 포지션·손익 / 멀티 벤뉴·멀티 전략
- 시즌 3 실전(14~20): 테스트넷 API 키 + 보안 / sandbox 24시간 운용 / 실거래 체크리스트 /
  Docker 클라우드 배포 / React 대시보드 / RL 에이전트 학습 / MCP로 AI 백테스트

**제작 팁**: "수익 보장" 류 문구 금지, 매 영상 투자조언 아님 고지, 계좌 잔고 모자이크,
영상당 1개 개념, 포크 저장소에 강의용 예제 폴더 동봉, 수익 모델은
애드센스 → 멤버십 → 유료 강의 → 1:1 컨설팅 순 확장.

---

## 4. 수익화 아이디어 상세

> 전략적 통찰: `ROADMAP.md`의 **Out of scope** 목록이 곧 **경쟁자 없는 시장**이다.
> UI 대시보드 ❌ / 분산 백테스트 오케스트레이션 ❌ / AI·ML 내장 도구 ❌ / 클라우드·DB·모니터링 연동 ❌

### TIER S

**① 한국 거래소 어댑터 (업비트·빗썸·한국투자증권·키움)**

| 항목 | 내용 |
| :-- | :-- |
| 핵심 | 어댑터 18개 중 한국 거래소 0개, 한국 개인 퀀트 수요는 큼 |
| 모델 | OSS 공개 → 명성 → 컨설팅 / 유료 라이선스(월 3~10만원) / 기업 커스텀(500만~3,000만원) |
| 난이도 | ★★★★☆ |
| 시작점 | `examples/live/_template/` (factories.py, data.py, execution.py, providers.py) |
| 기간 | 거래소 1개당 2~3개월 |
| 주의 | 패키지명 `nautilus-upbit` ❌ → `upbit-adapter-for-nautilus` 등 ⭕ |

**② 실시간 트레이딩 대시보드 (SaaS)**

| 항목 | 내용 |
| :-- | :-- |
| 핵심 | 공식이 UI를 영구 제외 → 무주공산 |
| 모델 | Freemium: 무료(1전략) / Pro $19월 / Team $99월 / 셀프호스팅 일회성 $199 |
| 스택 | React + TS + FastAPI + Redis + PostgreSQL |
| 난이도 | ★★★☆☆ · MVP 1~2개월 |
| 로드맵 | 포지션·손익 테이블 → 차트에 매매 시점 → 백테스트 비교 → 텔레그램·디스코드 알림 → 전략 원격 제어 |

**③ AI 트레이딩 MCP 서버**

| 항목 | 내용 |
| :-- | :-- |
| 핵심 | Claude/GPT가 엔진을 직접 조작하는 브릿지, 현재 사실상 공백 |
| 도구 설계 | `run_backtest` / `list_strategies` / `get_portfolio` / `analyze_performance` / `optimize_params` / `submit_order`(사람 승인 필수) |
| 모델 | OSS → 유료 호스팅 $29월 → 기업 라이선스 |
| 난이도 | ★★★☆☆ · MVP 2~4주 |
| 설득 포인트 | RiskEngine이 AI 폭주를 막는 안전성 스토리 |

### TIER A

**④ 한국어 교육 콘텐츠**

| 단계 | 내용 | 수익 |
| :-- | :-- | :-- |
| 1 | 유튜브 무료 20편 | 애드센스 월 10~100만원 |
| 2 | 채널 멤버십(예제 + Q&A) | 월 5천~1만원 × 구독자 |
| 3 | 온라인 강의 플랫폼 | 1인 15~30만원 |
| 4 | 전자책 / Notion 템플릿 | 1~3만원 × N |
| 5 | 라이브 부트캠프(주말 4주) | 1인 50~100만원 |

**⑤ 기술 컨설팅 & 외주 개발**

| 서비스 | 단가 |
| :-- | :-- |
| 설치·환경 구축 대행 | 30~50만원 |
| 전략 코드 구현 | 100~500만원 |
| 백테스트 검증 리포트 | 50~150만원 |
| 커스텀 어댑터 개발 | 500~3,000만원 |
| 월 운영 유지보수 | 월 50~200만원 |

Upwork/Fiverr 영어 시장은 단가 2~3배. 가장 빠른 현금화 경로.

**⑥ 전략 백테스트 SaaS** — 전략 업로드 → 클라우드 대량 백테스트 → 리포트.
공식 Out of scope인 분산 오케스트레이션 영역. 1회 $1 크레딧 / 월 $49 무제한. ★★★★☆
(클라우드 비용 관리가 핵심 리스크)

### TIER B

**⑦ 전략 마켓플레이스** — `crates/plugin` cdylib으로 전략을 .so/.dll 바이너리로 배포해 코드
유출 없이 판매. 수수료 20~30%. ★★★★★ · ⚠️ 투자자문업 해당 여부 등 금융 규제 검토 필수.

**⑧ 마켓메이킹·차익거래 직접 운용** — 실전 전략 튜토리얼이 이미 존재
(`grid_market_maker_dydx.md`, `fx_mean_reversion_ax.md`, `hurst_vpin_kraken.md`,
`lighter_rwa_composite_mm.md`, `delta_neutral_options_bybit.md`).
수익 상한 무제한 / ⚠️ 원금 전액 손실 가능. ★★★★★

**⑨ 데이터 서비스** — Tardis/Databento 원본을 정제 Parquet 카탈로그로 재가공.
⚠️ 원본 제공자 재배포 약관 확인 필요(대부분 금지) → "데이터 파이프라인 구축 서비스"로 우회 권장.

**⑩ 기업용 지원 패키지** — 소형 헤지펀드·운용사 대상 구축 + 교육 + SLA.
연 3,000만~1억원. ★★★★★ (레퍼런스·법인 필요, 장기 목표)

### 추천 실행 순서

```
0~1개월    학습 + 백테스트 마스터 (투자금 0)
1~3개월    유튜브 시작 (④) — 신뢰·브랜드
2~4개월    MCP 서버 OSS 공개 (③) — 기술력 증명·바이럴
3~6개월    React 대시보드 MVP (②)
4~8개월    컨설팅 수주 (⑤) — 첫 현금흐름
6~12개월   한국 거래소 어댑터 (①) — 최대 임팩트
12개월+    SaaS 본격화 / 기업 계약 (⑥, ⑩)
```

**첫 달 과제 3개**
1. `pip install nautilus_trader` 후 예제 1개 실행
2. 유튜브 1편 촬영 (설치 + 첫 백테스트)
3. MCP 서버 뼈대 작성 후 깃허브 공개

### 법적 유의사항

- **LGPL-3.0-only**: 엔진 코드를 **수정해 배포**하면 수정분 공개 의무. 단순 `import` 사용은 자유.
- **상표**: 제품명·패키지명 선두에 NautilusTrader 계열 명칭 사용 금지.
- **금융 규제**: 타인 자금 운용·투자자문·전략 판매는 자본시장법 검토 필요(전문가 상담 권장).
- 본 문서는 투자 조언이 아니다.

---

## 5. 참고 파일 색인

| 목적 | 경로 |
| :-- | :-- |
| 프로젝트 개요·설치 | `README.md` |
| 아키텍처 | `docs/concepts/architecture.md` |
| 전략 작성 | `docs/concepts/strategies.md`, `docs/tutorials/ema_cross.py` |
| 입문 예제 | `docs/getting_started/quickstart.py`, `backtest_high_level.py`, `backtest_low_level.py` |
| 백테스트 예제 | `examples/backtest/` (15개) |
| 실거래 예제 | `examples/live/<venue>/data_tester.py`, `exec_tester.py` |
| 신규 어댑터 템플릿 | `examples/live/_template/` |
| 어댑터 개발 가이드 | `ADAPTERS.md`, `docs/concepts/adapters.md` |
| Rust 전략/액터 작성 | `docs/how_to/write_rust_strategy.md`, `write_rust_actor.md` |
| 데이터 적재 | `docs/how_to/loading_external_data.py`, `data_catalog_databento.py` |
| 실거래 설정 | `docs/how_to/configure_live_trading.md`, `docs/concepts/live.md` |
| 리스크·포트폴리오 | `docs/concepts/portfolio.md`, `positions.md`, `accounting.md` |
| 기여 규칙 | `CONTRIBUTING.md`, `AGENTS.md`, `AI_POLICY.md` |
| 보안 | `SECURITY.md`, `docs/developer_guide/security.md` |
| 상표 | `TRADEMARK.md` |
| 로드맵·범위 | `ROADMAP.md` |
| v1 → v2 마이그레이션 | `MIGRATION_V2.md` |
| 이전 분석 노트 | `ANALYSIS_KO.md` |
