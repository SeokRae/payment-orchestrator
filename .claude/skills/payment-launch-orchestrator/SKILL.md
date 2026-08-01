---
name: payment-launch-orchestrator
description: "신규 결제/PG 서비스 출시(런칭)를 기획부터 구축 보고서까지 자동화하는 에이전트 팀 오케스트레이터. '신규 결제 서비스 출시', 'PG 서비스 구축 하네스', '새 결제 서비스 기획/설계/스펙/운영 문서 만들어줘', '결제 서비스 런칭 준비', '이 서비스 출시 문서 세트 만들어줘' 요청 시 사용. 후속 작업 — '제약분석만 다시', '아키텍처 수정해줘', '세션설계 보완', '스펙 갱신', '보고서 다시 종합', '이전 결과 기반으로 계속', '결제 서비스 출시 하네스 업데이트' 요청 시에도 반드시 이 스킬을 사용한다. 설계 문서 간 정합성을 각 단계 완료 직후 점진적으로 QA 검증한다. 20-areas/payment/{서비스}/docs/ 표준 구조(kickoff·constraints·architecture·session-design·operations·specs·adr·reports)를 실제 도메인 내용으로 채운다."
---

# Payment Launch Orchestrator — 결제 서비스 출시 에이전트 팀 오케스트레이터

신규 결제/PG 서비스가 기획 단계부터 구축 보고서를 낼 수 있는 상태까지 가는 과정을, 10명의 전문 에이전트 팀으로 자동화하는 통합 스킬.

## 실행 모드: 에이전트 팀

파이프라인(기획→아키텍처→스펙) + 팬아웃(인프라/세션설계/운영설계 병렬) + 생성-검증(각 생산자 직후 QA 게이트) + 횡단(ADR·규제질의 on-call) 복합 패턴. 단계마다 팀을 새로 만들지 않고, **팀 1개를 처음에 구성**하고 `TaskCreate`의 `depends_on`으로 순서를 강제한 뒤 팀원들이 공유 작업 목록에서 자체적으로 작업을 요청(claim)하며 진행한다.

## 에이전트 구성

| 팀원 | 에이전트 타입 | 역할 | 출력 |
|------|-------------|------|------|
| planner | payment-planner | 킥오프 + 제약분석 | `docs/kickoff/`, `docs/constraints/` |
| compliance | payment-compliance-analyst | 규제 준수 심층 검토 (1회) + 상시 대기(on-call) | `docs/constraints/` (02번 보강 문서) |
| architect | payment-architect | 논리 컴포넌트 아키텍처 설계 | `docs/architecture/01-*` |
| infra | payment-infra-builder | 물리 인프라 설계 (서버 사이징·이중화·존·DR) | `docs/architecture/02-infra-sizing.md` |
| session | payment-session-designer | 트랜잭션 세션 시나리오 설계 | `docs/session-design/` |
| ops | payment-ops-designer | 정산·모니터링·테스트 전략 | `docs/operations/` |
| spec | payment-spec-writer | 인터페이스·API 스펙 | `docs/specs/` |
| qa | payment-launch-qa | 설계 문서 정합성 점진적 검증 (생성-검증 게이트) | `_launch-workspace/qa/` (docs/에는 쓰지 않음) |
| adr | payment-decision-recorder | 의사결정 기록 (상시 대기) | `docs/adr/` |
| reporter | payment-launch-reporter | 구축 보고서 종합 | `docs/reports/` |

의존 관계: `planner` → `compliance` → `architect` → **QA(아키텍처)** → (`infra` ∥ `session` ∥ `ops`) → **QA(각 산출물)** → `spec`(session의 QA 통과 후) → **QA(스펙)** → `reporter`. `architect`는 논리 설계만 담당하고 물리 인프라(서버 대수·이중화·존·DR)는 `infra`로 분리되어 있다 — 실제 서비스 구축 현장에서도 인프라 구축과 어플리케이션(서비스) 구축을 별도 트랙으로 다루는 경우가 흔한 것을 반영한 구조다. `qa`는 매 생산자 직후 `payment-qa-review` 스킬로 경계면 교차검증을 수행하는 생성-검증 게이트이며, "문서 QA"만 담당한다(실제 코드 QA는 범위 밖 — `payment-qa-review` 스킬 참조). `compliance`는 planner 완료 직후 1회 심층 검토를 수행한 뒤, `adr`과 마찬가지로 고정 순서 없이 전 팀원의 SendMessage 규제 질의에 상시 응답한다. `adr`은 고정 순서 없이 전 팀원의 SendMessage 요청에 응답.

## 워크플로우

### Phase 0: 컨텍스트 확인 (후속 작업 지원)

1. 사용자 발화에서 서비스명을 파악하고 슬러그(`{service-slug}`, kebab-case — 예: `xyz-pay`)를 정한다 — 불명확하면 사용자에게 확인
2. `20-areas/payment/{service-slug}/docs/` 존재 여부 확인:
   - **미존재** → 초기 실행. Phase 1로 진행
   - **존재 + 사용자가 특정 문서·팀원만 재작업 요청** ("제약분석만 다시", "아키텍처 수정해줘" 등) → **부분 재실행**. 해당 팀원 1명만 서브 에이전트로 재호출(팀 구성 불필요 — 아래 "부분 재실행 절차" 참조)
   - **존재 + 서비스 방향이 근본적으로 바뀜(새 입력)** → **새 실행**. 기존 `docs/`를 건드리지 않고 `_launch-workspace/`만 `_launch-workspace_{YYYYMMDD}/`로 이동 후 Phase 1부터 재실행 (기존 docs는 사용자 확인 후 덮어쓸지 결정)
3. 부분 재실행 시: 재작업 대상 팀원에게 기존 산출물 경로 + 사용자 피드백을 프롬프트에 포함해 Agent 도구로 단독 호출

**부분 재실행 절차 (팀 불필요):**
```
Agent(subagent_type: "payment-{역할}", model: "opus", prompt:
  "기존 산출물 {경로}를 Read하고, 다음 피드백을 반영해 수정하라: {사용자 피드백}")
```
하류 팀원에게 영향이 있으면(예: 아키텍처 변경 시 세션설계·운영설계도 영향) 사용자에게 연쇄 재실행 필요 여부를 확인한다.

### Phase 1: 준비

1. 사용자 입력(서비스 개요, 연동 대상, 목표, 알려진 제약)을 정리
2. `20-areas/payment/{service-slug}/payment-{service-slug} 프로젝트 현황.md` 허브 노트 존재 확인
   - 없으면 `sr-obsidian:hub` 스킬(또는 `sr-obsidian:scaffold`)로 먼저 생성할지 사용자에게 확인 후 실행 — **이 오케스트레이터가 허브/WBS 생성 로직을 직접 구현하지 않는다** (기존 스킬 재사용)
3. `20-areas/payment/{service-slug}/_launch-workspace/00_input/개요.md`에 사용자 입력 저장 (감사 추적용 — 최종 산출물은 아님)

### Phase 2: 팀 구성

```
TeamCreate(
  team_name: "payment-launch-team",
  members: [
    { name: "planner",    agent_type: "payment-planner",             model: "opus", prompt: "{서비스 개요 + Phase 1 입력}" },
    { name: "compliance", agent_type: "payment-compliance-analyst",  model: "opus", prompt: "planner 완료 후 5개 규제 도메인(전자금융거래법·PCI-DSS·개인정보보호법·특금법·해외PSP연동) 심층 검토를 1회 수행하고, 이후 팀 전체의 규제 질의에 상시 대기한다" },
    { name: "architect",  agent_type: "payment-architect",           model: "opus", prompt: "..." },
    { name: "infra",      agent_type: "payment-infra-builder",       model: "opus", prompt: "architect 완료 후 물리 인프라(서버 사이징·이중화·존·DR)를 설계한다. ITSM 상신·서버 발급 등 실제 실행은 범위 밖 — 설계 문서만 작성" },
    { name: "session",    agent_type: "payment-session-designer",    model: "opus", prompt: "..." },
    { name: "ops",        agent_type: "payment-ops-designer",        model: "opus", prompt: "..." },
    { name: "spec",       agent_type: "payment-spec-writer",         model: "opus", prompt: "..." },
    { name: "qa",         agent_type: "payment-launch-qa",           model: "opus", prompt: "architect·infra·session·ops·spec 각 팀원이 완료될 때마다 payment-qa-review 스킬로 경계면 교차검증을 수행하고, 불일치는 해당 생산자에게 직접 SendMessage로 피드백한다(재검증 최대 2회)" },
    { name: "adr",        agent_type: "payment-decision-recorder",   model: "opus", prompt: "요청 기반 상시 대기, 스스로 작업을 claim하지 않는다" },
    { name: "reporter",   agent_type: "payment-launch-reporter",     model: "opus", prompt: "..." }
  ]
)

TaskCreate(tasks: [
  { title: "킥오프+제약분석",              assignee: "planner" },
  { title: "규제 준수 심층 검토",           assignee: "compliance", depends_on: ["킥오프+제약분석"] },
  { title: "컴포넌트 아키텍처 설계",         assignee: "architect",  depends_on: ["규제 준수 심층 검토"] },
  { title: "QA: 아키텍처 정합성 검증",       assignee: "qa",         depends_on: ["컴포넌트 아키텍처 설계"] },
  { title: "인프라 설계",                  assignee: "infra",      depends_on: ["QA: 아키텍처 정합성 검증"] },
  { title: "세션 시나리오 설계",            assignee: "session",    depends_on: ["QA: 아키텍처 정합성 검증"] },
  { title: "운영·정산·테스트 전략",         assignee: "ops",        depends_on: ["QA: 아키텍처 정합성 검증"] },
  { title: "QA: 인프라 정합성 검증",        assignee: "qa",         depends_on: ["인프라 설계"] },
  { title: "QA: 세션설계 정합성 검증",      assignee: "qa",         depends_on: ["세션 시나리오 설계"] },
  { title: "QA: 운영전략 정합성 검증",      assignee: "qa",         depends_on: ["운영·정산·테스트 전략"] },
  { title: "인터페이스·API 스펙",           assignee: "spec",       depends_on: ["QA: 세션설계 정합성 검증"] },
  { title: "QA: 스펙 정합성 검증",          assignee: "qa",         depends_on: ["인터페이스·API 스펙"] },
  { title: "구축 보고서 종합",             assignee: "reporter",   depends_on: ["QA: 인프라 정합성 검증", "QA: 운영전략 정합성 검증", "QA: 스펙 정합성 검증"] }
])
```

> `adr`은 고정 작업을 받지 않는다 — 팀원 누구든 되돌리기 어려운 결정이 나오면 `SendMessage(to: "adr", ...)`로 즉시 기록을 요청하는 방식으로 동작한다.
>
> `compliance`는 `규제 준수 심층 검토` 작업을 1회 수행한 뒤에는 `adr`처럼 고정 작업을 받지 않는다 — 이후 팀원 누구든 규제 관련 질의가 생기면 `SendMessage(to: "compliance", ...)`로 즉시 응답을 요청하는 방식으로 동작한다. `architect`의 `depends_on`이 `킥오프+제약분석`이 아니라 `규제 준수 심층 검토`인 것에 주의 — architect가 규제 심층 검토 없이 시작하는 경로를 원천 차단하는 핵심 장치다.
>
> `qa`는 5개 작업을 갖지만 전부 "생산자 직후" 트리거되는 게이트다 — `infra`·`session`·`ops` 3갈래가 `architect`가 아니라 `QA: 아키텍처 정합성 검증`에 의존하는 것에 주의(아키텍처 정합성이 확인되지 않은 채 3명이 각자 다른 전제로 병렬 작업을 시작하는 사고를 막는 장치). 마찬가지로 `spec`은 `QA: 세션설계 정합성 검증`에, `reporter`는 3개 QA 완료 작업에 의존한다. `qa`가 FIX 판정을 내려 생산자에게 되돌려보내는 것은 `TaskCreate` 그래프에 나타나지 않는 팀원 간 SendMessage 루프이며(최대 2회), 이 루프가 끝나야 해당 QA 작업이 `completed`로 표시된다.

### Phase 3: 실행 (팀원 자체 조율)

**실행 방식:** 팀원들이 공유 작업 목록에서 `depends_on`이 해소된 작업을 스스로 claim해 진행. 리더는 모니터링하며 병목이 생기면 개입.

**팀원 간 통신 규칙 (에이전트 정의 파일에 상술됨):**
- planner → compliance: H급 제약·확정 스코프 전달 (규제 심층 검토의 입력)
- compliance → architect: 규제 심층 검토 완료 시 H급 규제 항목 전달 (H급 규제 없이 설계 시작하지 않도록)
- architect → infra, session, ops: 컴포넌트 구조 동시 전달 (팬아웃, 3갈래)
- session ↔ spec: 시나리오별 확정 필드 전달, spec이 세션 설계에 없는 API가 필요하면 session에게 역으로 확인 질의
- session ↔ ops: 타임아웃/임계값 정합성 상호 확인
- infra ↔ ops: 서버 존 구성과 운영 전환 로드맵(배포·모니터링 대상)이 어긋나지 않도록 상호 확인
- qa → architect·infra·session·ops·spec: FIX/GAP 판정 시 구체적 위치·불일치 내용 전달, 수정 완료 알림 수신 즉시 재검증 (최대 2회)
- qa → adr: 불일치가 실은 미결정 때문이면(예: 이중화 여부 미정으로 infra·ops가 서로 다른 전제) 먼저 ADR 기록 요청 후 그 결정 기준으로 재검증
- qa → 리더: REDO급(전면 재설계 필요) 불일치 발견 시 판정 없이 즉시 에스컬레이션
- 전 팀원 → adr: 되돌리기 어려운 결정 발생 시 기록 요청
- 전 팀원 → compliance: 규제 관련 질의 발생 시 SendMessage로 문의 (adr과 동일한 on-call 패턴)
- 전 팀원 → reporter: 각자 산출물 완료 알림
- qa → reporter: 각 체크포인트 판정 결과(강제 PASS 포함) 요약 전달 — 리스크 레지스터의 "QA 확인 필요" 항목 근거
- reporter → adr: ADR 전체 목록을 SendMessage로 요청 (Phase 4 시작 시점) — adr → reporter: 목록 전달
- reporter → compliance: "법무 확인 필요 목록"(H급 규제 항목 포함)을 SendMessage로 요청

**리더 모니터링:**
- 팀원이 유휴 상태가 되면 자동 알림 수신 → 다음 작업이 의존성 대기 중인지, 막힌 것인지 확인
- `TaskGet`으로 전체 진행률 확인, 특정 팀원이 30분 이상 진행 없이 막히면 `SendMessage`로 상태 확인

**산출물 저장 (최종 경로 — `_launch-workspace/`가 아닌 실제 문서 저장 위치):**

| 팀원 | 출력 경로 |
|------|----------|
| planner | `20-areas/payment/{service-slug}/docs/kickoff/`, `docs/constraints/` |
| compliance | `20-areas/payment/{service-slug}/docs/constraints/` (02번 보강 문서 — planner와 동일 폴더, 별도 파일) |
| architect | `20-areas/payment/{service-slug}/docs/architecture/` (01번 컴포넌트 문서) |
| infra | `20-areas/payment/{service-slug}/docs/architecture/` (02번 인프라 사이징 문서 — architect와 동일 폴더, 별도 파일) |
| session | `20-areas/payment/{service-slug}/docs/session-design/` |
| ops | `20-areas/payment/{service-slug}/docs/operations/` |
| spec | `20-areas/payment/{service-slug}/docs/specs/` |
| qa | `20-areas/payment/{service-slug}/_launch-workspace/qa/{NN}-{체크포인트명}.md` (감사 기록 — `docs/`에는 쓰지 않는다) |
| adr | `20-areas/payment/{service-slug}/docs/adr/` |
| reporter | `20-areas/payment/{service-slug}/docs/reports/` |

(주의: `docs/` 표준 구조에 새 top-level 폴더를 추가하지 않는다 — compliance 산출물은 이미 존재하는 `constraints/` 폴더 안에 `02-*.md`로 들어가므로, 대상 프로젝트에 문서유형 표준이 있다면 그 세트를 그대로 유지한다.)

### Phase 4: 통합 확인

1. `reporter` 작업(`구축 보고서 종합`) 완료 대기 (`TaskGet`)
2. 리더가 `docs/reports/{service-slug}-플랫폼-구축-보고서.md`를 Read하여 리스크 레지스터에 각 팀원의 미결 항목이 실제로 반영됐는지 교차 확인
3. `_launch-workspace/qa/`의 5개 체크포인트 기록을 Read하여 강제 PASS(2회 초과 미해소) 항목이 리스크 레지스터에 빠짐없이 반영됐는지 확인
4. 누락 발견 시 reporter에게 SendMessage로 보완 요청 (1회)

### Phase 5: 정리

1. 팀원 전원에게 종료 알림 (SendMessage)
2. `TeamDelete`
3. `_launch-workspace/` 보존 (사후 감사용, 삭제하지 않음)
4. 사용자에게 결과 요약: 생성된 문서 목록, 미결 항목 개수, ADR 개수

## 데이터 흐름

```
[리더] → TeamCreate(10명) + TaskCreate(depends_on 체인)
                │
        planner ──SendMessage→ compliance ──SendMessage→ architect
                  │                  ↑
                  │                  └──(전 팀원, 규제 질의 시 on-call)──┘
                  │
              architect → [QA: 아키텍처 정합성]
                  │
                  ├─SendMessage→ infra   → [QA: 인프라]
                  ├─SendMessage→ session → [QA: 세션]  ──→ spec → [QA: 스펙]
                  └─SendMessage→ ops    → [QA: 운영]
                  (infra ↔ ops · session ↔ ops 정합성 상호 확인)
                  │
        (전 팀원) ──SendMessage(결정 발생시)→ adr
        (전 팀원) ──SendMessage(규제 질의시)→ compliance
        (QA가 FIX 판정) ──SendMessage(구체적 불일치)→ 해당 생산자 (재검증 루프, 최대 2회)
                  │
        [QA: 인프라] + [QA: 운영] + [QA: 스펙] 완료 ──SendMessage→ reporter
                                        │
                                  구축 보고서 종합
                                        │
                              [리더: 교차 확인] → 정리
```

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| 팀원 1명 실패/중지 | 리더가 감지 → SendMessage로 상태 확인 → 재시작. 재실패 시 해당 산출물 없이 하류 작업 진행, reporter에게 누락 명시 지시 |
| architect가 H급 제약·H급 규제 항목 없이 시작(planner·compliance 지연) | 리더가 개입해 architect를 대기시키고 planner·compliance 상태 확인 (TaskCreate의 `depends_on: ["규제 준수 심층 검토"]`가 정상 작동하면 원천적으로 발생하지 않아야 함 — 발생 시 의존관계 설정 오류 의심) |
| infra/session/ops 중 일부만 지연 | 나머지는 계속 진행, spec은 session의 QA 통과만으로 시작 가능(infra·ops의 QA는 spec의 하드 의존성 아님). 단 reporter는 infra·ops·spec 3개 QA 모두 완료돼야 시작 |
| qa가 같은 불일치를 2회 재검증 후에도 발견 | 강제 PASS 처리 + 생산자에게 SendMessage로 해당 문서에 "미결 항목(QA 확인 필요)" 섹션 추가를 요청(qa 자신은 docs/에 쓰지 않는다) + reporter에게 SendMessage 통지. 3회차 재시도를 리더가 임의로 지시하지 않는다(무한 루프 방지가 우선) |
| qa가 REDO급(전면 재설계 필요) 불일치를 발견 | qa는 판정하지 않고 즉시 리더에게 에스컬레이션 → 리더가 영향 범위(팬아웃 3갈래 전체 재작업 여부 등) 판단 후 해당 팀원(들)에게 직접 지시 |
| adr에 결정 기록 폭주(팀원 다수가 동시 요청) | adr이 순차 처리, 리더는 개입하지 않음 (병목이 아니라 정상 동작) |
| 외부 연동 스펙 전반 미확보 | 전체를 막지 않고 "미확보" 표기로 진행 — 스펙 확보는 이 하네스 범위 밖(사용자가 별도로 확보) |
| 팀원 과반 실패 | 사용자에게 알리고 진행 여부 확인 |

## 테스트 시나리오

### 정상 흐름
1. 사용자: "새 결제 서비스 'xyz-pay' 출시 준비 문서 만들어줘 — 해외 카드사 다이렉트 연동, Q3 목표"
2. Phase 0: `20-areas/payment/xyz-pay/docs/` 미존재 → 초기 실행
3. Phase 1: 허브 노트 없음 확인 → 사용자에게 `sr-obsidian:hub` 선행 실행 제안 → 승인 시 실행
4. Phase 2: 10명 팀 구성, 13개 작업 등록(생산 8 + QA 5)
5. Phase 3: planner 완료 → compliance 규제 심층 검토 시작 → compliance 완료(H급 규제 항목을 architect에 SendMessage) → architect 완료 → qa가 "QA: 아키텍처 정합성 검증" 수행, PASS → infra/session/ops 3갈래 병렬 시작 → 각각 완료 시마다 qa가 해당 체크포인트 검증 → session의 QA가 FIX 판정(예: "해외카드 시나리오가 architect의 미결 항목과 모순") → qa가 session에게 SendMessage로 구체적 위치 지적 → session 수정 → qa 재검증 PASS → spec 시작 → qa가 스펙 검증 → 중간에 infra가 "이중화 여부"를 adr에게 기록 요청, session이 해외 결제수단 관련 규제 질의를 compliance에게 SendMessage로 문의
6. Phase 4: infra·ops·spec 3개 QA 완료 확인 후 reporter가 구축 보고서 완성, 리더가 리스크 레지스터 교차 확인(QA 강제 PASS 항목 포함 여부까지)
7. Phase 5: 팀 정리, 사용자에게 결과 요약
8. 예상 결과: `20-areas/payment/xyz-pay/docs/{kickoff,constraints,architecture,session-design,operations,specs,adr,reports}/` 전체 생성

### 에러 흐름 1 — 산출물 미확보
1. Phase 3에서 spec이 외부 카드사 연동 스펙 미확보로 절반만 완료
2. spec이 "미확보" 표기 후 작업 완료로 보고 (전체 실패가 아님 — 부분 완료도 유효)
3. qa가 "QA: 스펙 정합성 검증"에서 이를 GAP이 아니라 정상적인 "미확보 표기"로 인식(스펙 문서 자체가 이미 명시했으므로) → PASS
4. reporter가 spec의 미확보 항목을 리스크 레지스터에 반영
5. 최종 보고서에 "인터페이스 스펙 일부 미확보 — 연동사 문서 확보 후 보완 필요" 명시

### 에러 흐름 2 — QA 재검증 루프 소진
1. Phase 3에서 qa가 "QA: 운영전략 정합성 검증"에서 ops 문서의 타임아웃 값이 session 문서와 다름을 발견 → FIX 판정, ops에게 SendMessage
2. ops가 수정 후 완료 알림 → qa 재검증 → 여전히 다른 값(이번엔 반대 방향으로 어긋남) → FIX 2번째 판정
3. ops가 다시 수정 → qa 재검증(3번째 확인) → 아직도 불일치
4. qa는 3번째부터 재시도하지 않고 강제 PASS 처리 + ops에게 SendMessage로 "미결 항목(QA 확인 필요) — 타임아웃 값이 session과 상충, 수동 확인 필요" 섹션을 문서에 추가하도록 요청(qa 자신은 docs/에 쓰지 않는다) + reporter에게 SendMessage 통지
5. reporter가 이 항목을 리스크 레지스터 최상단(H급 취급)에 반영

## 결제 도메인 참고 자료 (팀원이 필요 시 Read)

- 문서 구조·포맷 참고용(내용 복사 금지): 대상 프로젝트에 이미 만들어둔 유사 결제 서비스 문서가 있다면 구조만 참고
- 도메인 지식: 대상 프로젝트에 벤더별(카드사·VAN·PSP) 연동 문서가 있다면 활용 (있는 범위 내)
- 규제 판단 근거: `.claude/skills/payment-compliance-review/references/regulatory-checklist.md` (compliance 팀원이 Skill 도구로 호출, 다른 팀원이 규제 맥락을 직접 확인할 때도 Read 가능)
- QA 검증 매트릭스·판정 기준: `.claude/skills/payment-qa-review/SKILL.md` (qa 팀원이 Skill 도구로 호출 — **"문서 QA"만 다룬다. 실제 구현 코드 테스트("개발 QA")는 이 하네스 범위 밖**이며 별도 코드베이스에서 진행됨을 팀원 전원이 인지)
- 문서화 표준: 대상 프로젝트에 문서 구조 표준이 있다면 참고
