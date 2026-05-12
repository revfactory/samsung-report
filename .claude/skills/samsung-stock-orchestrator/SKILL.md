---
name: samsung-stock-orchestrator
description: "삼성전자(005930) 주식 가치를 다관점에서 분석해 통합 보고서를 작성하는 에이전트 팀 오케스트레이터. 펀더멘털·기술적·산업·거시·시장심리 5개 영역을 병렬 분석한 뒤 통합·교차검증해 최종 보고서를 생성한다. 트리거: '삼성전자 주식 분석', '삼성전자 가치 평가', '삼성전자 적정주가', '삼성전자 투자 보고서', '005930 분석', 'equity research 삼성전자'. 후속 작업: 삼성전자 보고서 다시 실행, 부분 재실행, 업데이트, 수정, 보완, 이전 결과 개선, '펀더멘털만 다시', '기술적 부분만 갱신' 등 요청 시에도 반드시 이 스킬을 사용. 단순 시세 조회·뉴스 요약은 이 스킬의 범위가 아님(이때는 직접 응답)."
---

# Samsung Stock Orchestrator

삼성전자(005930)의 주식 가치를 5명의 전문 애널리스트 + 1명의 통합/QA 리뷰어로 구성된 에이전트 팀으로 분석하여, 사실·추정·의견이 명확히 구분된 최종 투자 보고서를 생성하는 오케스트레이터.

## 실행 모드: 에이전트 팀

5명 애널리스트가 병렬로 영역별 분석을 수행하고, SendMessage로 교차 검증·가정값 공유를 거친 후, 통합 리뷰어가 합본을 작성한다. 팬아웃/팬인 + 생성-검증의 복합 패턴.

## 에이전트 구성

| 팀원 | 에이전트 타입 | 역할 | 사용 스킬/레퍼런스 | 출력 파일 |
|------|-------------|------|------------------|---------|
| fundamental | `fundamental-analyst` | 재무·밸류에이션 | `equity-research-method/references/valuation.md` | `_workspace/02_fundamental_analysis.md` |
| technical | `technical-analyst` | 차트·모멘텀 | `equity-research-method/references/technical.md` | `_workspace/02_technical_analysis.md` |
| industry | `industry-analyst` | 반도체 산업·경쟁 | `equity-research-method/references/semiconductor-industry.md` | `_workspace/02_industry_analysis.md` |
| macro | `macro-analyst` | 거시·환율·금리 | `equity-research-method/references/macro.md` | `_workspace/02_macro_analysis.md` |
| sentiment | `sentiment-analyst` | 컨센서스·수급·심리 | `equity-research-method/references/sentiment.md` | `_workspace/02_sentiment_analysis.md` |
| reviewer | `integrator-reviewer` | 통합·QA·최종 보고서 | `equity-research-method/references/citations-and-disclaimers.md` | `_workspace/99_final_report.md` |

모든 에이전트는 `equity-research-method` 스킬의 공통 원칙(사실/추정/의견 3단 구분, 출처·시점 표기, 면책 처리)을 준수한다.

## 워크플로우

### Phase 0: 컨텍스트 확인 (후속 작업 지원)

`_workspace/` 디렉토리 존재 여부와 사용자 요청 유형을 확인하여 실행 모드를 결정한다.

1. `_workspace/` 존재 여부 확인 (Bash `ls _workspace/ 2>/dev/null`)
2. 실행 모드 결정:
   - **`_workspace/` 미존재** → 초기 실행, Phase 1로 진행
   - **`_workspace/` 존재 + 사용자가 부분 수정 요청** (예: "기술적 분석만 다시", "산업 부분만 갱신") → 부분 재실행. 해당 에이전트만 재호출하고, 이전 산출물 경로를 프롬프트에 포함하여 기존 결과를 읽고 갱신하도록 지시. Phase 2의 팀 구성에서 해당 에이전트만 포함.
   - **`_workspace/` 존재 + 전체 재실행 또는 새 분석 기준일** → 새 실행. 기존 `_workspace/`를 `_workspace_{YYYYMMDD_HHMMSS}/`로 이동 후 Phase 1 진행
3. 결정된 모드를 사용자에게 한 줄 보고

### Phase 1: 준비

1. **분석 기준일 결정**: 사용자가 명시하지 않으면 "최근 거래일 종가 기준"으로 설정. 거래소 휴장일·주말 고려.
2. **사용자 요청 파싱**:
   - 강조 영역 (예: "거시 영향 중심으로")
   - 시나리오 가중치 (예: "Bullish 시나리오 더 자세히")
   - 출력 경로 (default: 작업 디렉토리의 `삼성전자_주식분석_보고서_{YYYYMMDD}.md`)
3. **작업 디렉토리 준비**:
   - 초기 실행: `_workspace/`, `_workspace/00_input/` 생성
   - 새 실행: 기존 `_workspace/` → `_workspace_{타임스탬프}/`로 이동한 직후 새 `_workspace/` 생성
4. **입력 노트 저장**: `_workspace/00_input/request.md`에 사용자 원본 요청 + 파싱 결과 기록

### Phase 2: 팀 구성

**팀 생성** (전체 실행 시):

```
TeamCreate(
  team_name: "samsung-equity-team",
  members: [
    { name: "fundamental", agent_type: "fundamental-analyst", model: "opus",
      prompt: "삼성전자 펀더멘털·밸류에이션 분석. equity-research-method 스킬과 references/valuation.md 로드. 출력: _workspace/02_fundamental_analysis.md. 분석 기준일: {기준일}. 입력 노트: _workspace/00_input/request.md" },
    { name: "technical", agent_type: "technical-analyst", model: "opus",
      prompt: "삼성전자 기술적 분석. equity-research-method 스킬과 references/technical.md 로드. 출력: _workspace/02_technical_analysis.md. 분석 기준일: {기준일}." },
    { name: "industry", agent_type: "industry-analyst", model: "opus",
      prompt: "반도체·디스플레이·모바일 산업 분석. equity-research-method 스킬과 references/semiconductor-industry.md 로드. 출력: _workspace/02_industry_analysis.md. fundamental의 사업부문 매출이 들어오는 대로 활용." },
    { name: "macro", agent_type: "macro-analyst", model: "opus",
      prompt: "거시 환경 분석. equity-research-method 스킬과 references/macro.md 로드. 출력: _workspace/02_macro_analysis.md." },
    { name: "sentiment", agent_type: "sentiment-analyst", model: "opus",
      prompt: "시장 심리·수급·컨센서스 분석. equity-research-method 스킬과 references/sentiment.md 로드. 출력: _workspace/02_sentiment_analysis.md." }
  ]
)
```

> 부분 재실행 시: 해당 에이전트만 members에 포함하고 프롬프트에 "이전 산출물 _workspace/02_*.md를 읽고 갱신/수정하라"를 추가.

**작업 등록**:

```
TaskCreate(tasks: [
  { title: "펀더멘털 분석", assignee: "fundamental", description: "재무 추세 + 사업부문 분해 + 멀티플 + SOTP + 간이 DCF" },
  { title: "기술적 분석", assignee: "technical", description: "추세·MA·모멘텀·지지저항·3시나리오" },
  { title: "산업 분석", assignee: "industry", description: "메모리 사이클 + HBM + 파운드리 + 경쟁 매트릭스" },
  { title: "거시 분석", assignee: "macro", description: "환율·금리·CapEx·외인흐름·SOX 상관" },
  { title: "심리 분석", assignee: "sentiment", description: "컨센서스 분포 + 수급 + 공매도 + 뉴스" },
  { title: "통합·QA 보고서", assignee: "reviewer", description: "5개 산출물 통합 + 교차검증 + 최종 보고서", depends_on: ["펀더멘털 분석","기술적 분석","산업 분석","거시 분석","심리 분석"] }
])
```

> reviewer는 의존성 때문에 Phase 3에서는 대기. Phase 4에서 별도 팀으로 재구성하거나 그대로 팀에 합류시킬 수 있다 — 본 오케스트레이터는 reviewer를 같은 팀에 포함하여 의존성만 걸어둔다.

### Phase 3: 병렬 분석 (5명 자체 조율)

**실행 방식**: 팀원들이 공유 작업 목록에서 작업을 요청(claim)하고 독립 수행. 리더는 진행 상황을 모니터링.

**팀원 간 통신 규칙** (반드시 에이전트 정의에 명시된 대로):
- `fundamental` ↔ `industry`: 사업부문 매출 / 메모리 ASP 가정값 교환
- `fundamental` ↔ `sentiment`: 컨센서스 EPS vs 실제 / 멀티플 적정가 vs 목표주가
- `technical` ↔ `sentiment`: 거래량 급증 구간의 이벤트 조사 / 외인 수급 시계열
- `macro` ↔ `industry`: 빅테크 CapEx → 메모리 수요 채널
- `macro` ↔ `fundamental`: 환율 가정값 → 실적 추정 사용
- `macro` ↔ `sentiment`: 외인 자금 흐름 추세 교차 검증

**브로드캐스트는 사용하지 않는다** (`SendMessage(to: "all")` 금지) — 비용이 높고 소음을 만든다.

**리더 모니터링**:
- 팀원이 유휴 상태가 되면 자동 알림 수신
- 특정 팀원이 막혔을 때(예: 데이터 미확보로 30분 이상 진척 없음) SendMessage로 "현 시점까지 확인 가능한 자료로만 작성하고 미확보 부분은 'X 미확보' 명시" 지시
- 진행률 점검: TaskList / TaskGet

**산출물 저장** (Phase 3 종료 시점):

| 팀원 | 출력 경로 |
|------|----------|
| fundamental | `_workspace/02_fundamental_analysis.md` |
| technical | `_workspace/02_technical_analysis.md` |
| industry | `_workspace/02_industry_analysis.md` |
| macro | `_workspace/02_macro_analysis.md` |
| sentiment | `_workspace/02_sentiment_analysis.md` |

### Phase 4: 통합 및 QA

1. 5명의 작업 완료 대기 (TaskGet으로 전부 completed 확인)
2. `reviewer` 작업 할당 (이미 의존성 등록됨, 자동으로 진행 가능 상태)
3. reviewer에게 SendMessage:
   ```
   5개 산출물(_workspace/02_*.md)을 모두 Read한 뒤
   integrator-reviewer 에이전트 정의의 검증 체크리스트 8개 항목을 수행하라.
   결과를 _workspace/99_qa_checklist.md에 기록하고,
   최종 보고서를 _workspace/99_final_report.md에 작성한 뒤,
   사용자 지정 경로 또는 default 경로(삼성전자_주식분석_보고서_{YYYYMMDD}.md)에 복사하라.
   ```
4. reviewer가 모순/누락을 발견하면 해당 애널리스트에게 SendMessage로 1회 재작업 요청 가능. 1회 요청 후에도 미해결이면 reviewer는 리더에게 알리고 최종 보고서에 "미해결 이슈" 섹션으로 명시.

### Phase 5: 정리

1. reviewer 완료 알림 수신 후, 최종 보고서가 사용자 지정 경로에 생성되었는지 확인 (Bash `ls`)
2. 팀 정리:
   ```
   TeamDelete(team_name: "samsung-equity-team")
   ```
3. `_workspace/` 디렉토리는 **보존**한다 — 사후 검증, 부분 재실행, 감사 추적에 사용
4. 사용자에게 결과 요약 보고:
   - 최종 보고서 경로
   - 핵심 적정주가 범위, 종합 의견
   - 미해결 이슈 / 데이터 미확보 영역 명시
   - 후속 작업 가능 명령 안내 (예: "기술적 부분만 다시" 요청 시 부분 재실행 가능)

## 데이터 흐름

```
[리더]
  ↓ TeamCreate + TaskCreate
[fundamental] ←─SendMessage─→ [industry]
     ↕                            ↕
[macro] ←──SendMessage────→ [sentiment]
     ↕                            ↕
     └──[technical]────────────────┘
              │
              ↓ (각자 파일로 저장)
   _workspace/02_*.md (5개)
              │
              ↓ (의존성 충족 → reviewer claim)
         [reviewer]
              ↓
   _workspace/99_qa_checklist.md
   _workspace/99_final_report.md
              ↓
   {사용자 지정 경로 또는 default}.md
```

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| 5명 중 1명 실패/중지 | 리더가 유휴 알림 수신 → SendMessage로 상태 확인 → 재시작 1회. 재실패 시 reviewer에게 "{영역} 데이터 미확보" 명시하고 진행 |
| 5명 중 과반 실패 | 사용자에게 알리고 진행 여부 확인 (1차 자료 접근이 환경적으로 불가능한 상황 의심) |
| 타임아웃 (전체 진행이 지나치게 느림) | 현재까지 수집된 부분 결과로 reviewer 진행, 미수집 영역은 보고서에 명시 |
| 팀원 간 데이터 충돌 | reviewer가 출처와 함께 병기. 삭제·임의 결정 금지 |
| reviewer의 1회 재작업 요청 거절 | 미해결 이슈 섹션으로 보고서에 명시 후 종료 |
| 실시간 데이터 접근 불가 (웹 차단 환경) | 모든 에이전트에게 "마지막 확인 가능 공식 자료 기준" 으로 작성하도록 지시. 보고서에 "데이터 시점" 부록 강화 |

## 테스트 시나리오

### 정상 흐름
1. 사용자: "현재 삼성전자 주식의 가치를 다관점으로 분석해 보고서를 작성해줘"
2. Phase 0: `_workspace/` 미존재 → 초기 실행
3. Phase 1: 분석 기준일 = 최근 거래일, 출력 경로 = default
4. Phase 2: 5명 + reviewer 팀 구성, 6개 작업 등록 (reviewer는 5개 의존)
5. Phase 3: 5명이 병렬 분석, SendMessage로 환율 가정값·메모리 ASP·컨센서스 EPS 등 공유
6. Phase 4: reviewer가 모든 산출물 통합, QA 체크리스트 통과, 보고서 작성
7. Phase 5: 최종 보고서 사용자 경로 생성, 팀 정리, 요약 보고
8. 예상 결과: `삼성전자_주식분석_보고서_{YYYYMMDD}.md` 생성, `_workspace/` 6개 산출물 보존

### 부분 재실행 흐름
1. 사용자: "기술적 분석만 갱신해줘"
2. Phase 0: `_workspace/` 존재 + 부분 수정 키워드 감지 → 부분 재실행
3. Phase 2: `technical` 팀원만 포함하여 팀 구성, "이전 산출물을 갱신하라" 프롬프트
4. Phase 3: technical이 `_workspace/02_technical_analysis.md`를 갱신
5. Phase 4: reviewer가 갱신된 기술 분석 + 기존 4개 분석을 다시 통합, `_workspace/99_final_report.md` 갱신
6. Phase 5: 최종 보고서 갱신, 사용자에게 변경 요약 보고

### 에러 흐름
1. Phase 3에서 `macro` 에이전트가 외부 데이터 접근 실패로 30분 이상 진척 없음
2. 리더가 유휴 알림 수신
3. SendMessage로 "마지막 확인 가능 공식 자료 기준으로 정성 평가만 작성하고 미확보 부분 명시" 지시
4. macro가 부분 산출물 완성 (정성 위주, 미확보 항목 명시)
5. Phase 4: reviewer가 5개 산출물 통합, "거시 정량 데이터 일부 미확보" 섹션 추가
6. Phase 5: 최종 보고서에 데이터 한계 부록 강화하여 출력

## 가드레일

- **단정적 매수/매도 추천 금지** — reviewer는 최종 의견을 시나리오·범위·트리거로 제시
- **임의 추정 금지** — 모든 수치는 출처를 갖거나 "확인 필요"로 명시
- **면책 조항 누락 금지** — 최종 보고서에 면책 + 데이터 한계 부록 필수
- **세션당 1팀 제약 준수** — 새 팀 구성 전 반드시 `TeamDelete`
- **모든 Agent 호출에 `model: "opus"` 명시**
