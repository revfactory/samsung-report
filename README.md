# Samsung Electronics Equity Research Harness

삼성전자(005930)의 주식 가치를 **펀더멘털·기술적·산업·거시·시장심리** 5개 관점에서 병렬 분석하고, 사실·추정·의견이 명확히 구분되며 출처가 추적 가능한 통합 보고서를 산출하는 Claude Code 하네스입니다.

> 본 저장소의 모든 산출물은 학습·연구 목적이며, 특정 종목의 매수·매도를 권유하지 않습니다. 투자 판단과 그 결과에 대한 책임은 전적으로 투자자 본인에게 있습니다.

---

## 산출물

| 파일 | 설명 |
|------|------|
| [`삼성전자_주식분석_보고서_20260512.md`](./삼성전자_주식분석_보고서_20260512.md) | 5개 영역을 통합한 최종 투자 보고서 (2026-05-12 기준) |
| [`index.html`](./index.html) | 보고서 시각화 페이지 (GitHub Pages 서빙 가능) |
| [`_workspace/`](./_workspace) | 영역별 분석 원본 + QA 체크리스트 (감사 추적용) |

### `_workspace/` 구조

```
_workspace/
├── 00_input/request.md             # 사용자 요청 + 파싱 결과
├── 02_fundamental_analysis.md      # 펀더멘털 (재무·밸류에이션)
├── 02_technical_analysis.md        # 기술적 (차트·모멘텀)
├── 02_industry_analysis.md         # 산업 (반도체·디스플레이·모바일)
├── 02_macro_analysis.md            # 거시 (환율·금리·CapEx)
├── 02_sentiment_analysis.md        # 심리 (컨센서스·수급·뉴스)
├── 99_qa_checklist.md              # 통합·QA 체크리스트
└── 99_final_report.md              # 최종 통합 보고서 (마스터본)
```

---

## 하네스 구성

Claude Code의 **에이전트 팀 + 스킬** 패턴으로 구성된 팬아웃/팬인 + 생성-검증 복합 하네스입니다.

### 팀 구성 (5 + 1)

| 팀원 | 에이전트 | 역할 |
|------|----------|------|
| fundamental | `fundamental-analyst` | 재무 추세, 사업부문 분해, PER/PBR/EV·EBITDA, SOTP, 간이 DCF |
| technical | `technical-analyst` | 추세·이동평균·RSI·MACD·지지저항, 3시나리오 가격대 |
| industry | `industry-analyst` | 메모리 사이클, HBM 경쟁, 파운드리, 디스플레이, MX |
| macro | `macro-analyst` | 환율·금리·빅테크 CapEx·외인 흐름·SOX 상관 |
| sentiment | `sentiment-analyst` | 컨센서스 분포·수급·공매도·뉴스·SNS 시그널 |
| **reviewer** | `integrator-reviewer` | 5개 산출물 통합·교차검증·QA·최종 보고서 |

5명 애널리스트가 **병렬**로 영역별 분석을 수행하고, 가정값(환율, 메모리 ASP, 컨센서스 EPS 등)은 `SendMessage`로 직접 교환합니다. 의존성이 걸린 reviewer가 마지막에 통합·QA를 수행합니다.

### 워크플로우

```
[Phase 0] 컨텍스트 확인 (_workspace 존재 → 부분/전체 재실행 판정)
   ↓
[Phase 1] 분석 기준일·요청 파싱·작업 디렉토리 준비
   ↓
[Phase 2] TeamCreate(5+1) + TaskCreate(6, reviewer 의존성)
   ↓
[Phase 3] 5명 병렬 분석 — SendMessage 페어 교환
          ├─ fundamental ↔ industry   (사업부문/메모리 ASP)
          ├─ fundamental ↔ sentiment  (EPS vs 컨센서스)
          ├─ technical   ↔ sentiment  (거래량/수급)
          ├─ macro       ↔ industry   (CapEx → 수요)
          ├─ macro       ↔ fundamental(환율 가정)
          └─ macro       ↔ sentiment  (외인 흐름)
   ↓
[Phase 4] reviewer 통합 — QA 8항목 + 모순 시 1회 재작업 요청
   ↓
[Phase 5] 최종 보고서 출력 + TeamDelete + 요약 보고
```

### 공통 방법론 스킬

모든 에이전트는 [`equity-research-method`](./.claude/skills/equity-research-method/) 스킬을 로드하여 다음 공통 원칙을 준수합니다.

- **사실(F) / 추정(E) / 의견(O) 3단 라벨링**
- 모든 수치에 **출처 + 시점** 표기
- 단정적 매수·매도 권유 금지 → 시나리오·범위·트리거로 제시
- 면책 조항 및 데이터 한계 부록 필수

영역별 산식·지표·인용 규칙은 `references/` 하위 6개 문서에 정의되어 있습니다 (valuation, technical, semiconductor-industry, macro, sentiment, citations-and-disclaimers).

### 실행 (트리거)

Claude Code 세션에서 다음과 같이 요청하면 `samsung-stock-orchestrator` 스킬이 동작합니다.

- `삼성전자 주식 가치 분석해줘`
- `005930 적정주가 보고서 작성`
- `equity research 삼성전자`

부분 재실행:

- `기술적 분석만 다시 해줘` → `_workspace/02_technical_analysis.md` 갱신 후 `99_final_report.md` 재통합

---

## 디렉토리 구조

```
.
├── README.md
├── CLAUDE.md                       # 하네스 사용 트리거 정의
├── 삼성전자_주식분석_보고서_*.md   # 출력 보고서
├── index.html              # 보고서 시각화
├── _workspace/                     # 영역별 분석 원본
└── .claude/
    ├── agents/                     # 6개 전문가 에이전트 정의
    │   ├── fundamental-analyst.md
    │   ├── technical-analyst.md
    │   ├── industry-analyst.md
    │   ├── macro-analyst.md
    │   ├── sentiment-analyst.md
    │   └── integrator-reviewer.md
    └── skills/
        ├── samsung-stock-orchestrator/   # 오케스트레이션 워크플로우
        └── equity-research-method/       # 공통 분석 방법론 + references
```

---

## 가드레일

- 단정적 매수/매도 추천 금지 — 시나리오·범위·트리거로만 제시
- 임의 추정 금지 — 모든 수치는 출처를 갖거나 `확인 필요`로 명시
- 실시간 데이터 미확보 시 마지막 확인 가능 공식 자료 기준으로 작성하고 데이터 시점 부록 강화
- 최종 보고서에 면책 + 데이터 한계 부록 필수

---

## 라이선스 / 면책

본 저장소는 개인 학습·연구 목적의 산출물이며 어떠한 종목의 매수·매도도 권유하지 않습니다. 보고서에 포함된 수치·전망·의견은 작성 시점 기준 공개 자료에 기반하며 사후 변경될 수 있습니다.
