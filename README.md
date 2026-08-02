# payment-launch-orchestrator

> 신규 결제/PG 서비스 출시를 기획부터 구축 보고서까지, **10명의 AI 에이전트**가 팀으로 자동화하는 [Claude Code](https://claude.com/claude-code) 하네스입니다.

기획(킥오프·제약분석) → 규제 준수 심층 검토 → 논리 아키텍처 설계(QA 게이트) → (물리 인프라 설계 ∥ 트랜잭션 세션설계 ∥ 운영전략, 각각 완료 직후 QA 게이트) → 인터페이스 스펙(QA 게이트) → 구축 보고서까지, 결제 서비스 하나를 출시 준비하는 데 필요한 설계 문서 세트를 한 번에 만듭니다. (QA 게이트는 architect 이후 5개 체크포인트에 적용되며, 그 앞의 기획·규제 심층 검토 단계에는 별도 게이트가 없습니다.)

> 원본은 개인 Obsidian PARA vault에서 `harness:harness` 메타 스킬로 반복 확장하며 만들어졌습니다 — 규제 전문가 분리, 인프라/어플리케이션 역할 분리, 점진적 QA 게이트 순서로 4단계에 걸쳐 진화했습니다. 이 저장소는 그 결과물을 독립적으로 재사용할 수 있게 옮긴 것입니다.

## 팀 구성

| 에이전트 | 역할 | 산출물 |
|---|---|---|
| [`payment-planner`](agents/payment-planner.md) | 킥오프(사업배경·이해관계자) + 제약분석(H/M/L 등급) | `docs/kickoff/`, `docs/constraints/` |
| [`payment-compliance-analyst`](agents/payment-compliance-analyst.md) | 규제 준수 심층 검토(전자금융거래법·PCI-DSS·개인정보보호법·특금법·해외연동) 1회 + 상시 대기 | `docs/constraints/` (02번 보강) |
| [`payment-architect`](agents/payment-architect.md) | 논리 컴포넌트 아키텍처 설계 (경계·인증/승인/정산 흐름) | `docs/architecture/01-*` |
| [`payment-infra-builder`](agents/payment-infra-builder.md) | 아키텍처의 네트워크 인프라 레벨 문서화 (서버 사이징·이중화·존·DR) — 바로 구축에 쓸 수 있는 기준 문서 | `docs/architecture/02-{service-slug}-인프라-사이징.md` |
| [`payment-session-designer`](agents/payment-session-designer.md) | 트랜잭션 세션 시나리오 설계 (승인·취소·구독·가상계좌·웹훅 등) | `docs/session-design/` |
| [`payment-ops-designer`](agents/payment-ops-designer.md) | 정산·모니터링·이슈대응·테스트 전략 | `docs/operations/` |
| [`payment-spec-writer`](agents/payment-spec-writer.md) | 인터페이스·API 스펙 | `docs/specs/` |
| [`payment-launch-qa`](agents/payment-launch-qa.md) | 생산자 완료 직후 경계면 교차검증(생성-검증 게이트), 불일치는 즉시 되돌려보냄(최대 2회) | `{service-root}/_launch-workspace/qa/` (문서 산출물 아님) |
| [`payment-decision-recorder`](agents/payment-decision-recorder.md) | 되돌리기 어려운 결정을 ADR로 기록, 상시 대기 | `docs/adr/` |
| [`payment-launch-reporter`](agents/payment-launch-reporter.md) | 전 팀원 산출물 + QA 결과를 구축 보고서로 종합 | `docs/reports/` |

오케스트레이터는 [`skills/launch-orchestrator/SKILL.md`](skills/launch-orchestrator/SKILL.md) 하나입니다. 팀 구성·착수 게이트(G1~G5)·에러 핸들링을 정의하며, 리더 역할은 별도 에이전트 없이 세션 자체가 맡습니다.

**의존 관계:** `planner` → `compliance` → `architect` → **QA** → (`infra` ∥ `session` ∥ `ops`) → **QA(각각)** → `spec`(session QA 통과 후) → **QA** → `reporter`(3개 QA 통과 후). `adr`·`compliance`(2단계 이후)는 고정 순서 없이 전 팀원의 SendMessage 요청에 상시 응답합니다.

### 스킬 3개

| 스킬 | 트리거 문구 예시 | 역할 |
|---|---|---|
| [`launch-orchestrator`](skills/launch-orchestrator/SKILL.md) | "신규 결제 서비스 출시", "PG 서비스 구축 하네스" | 팀 전체를 조율하는 메인 스킬 — 10명 스폰, 착수 게이트 G1~G5 관리, 부분 재실행(특정 팀원만 재작업) 처리 |
| [`compliance-review`](skills/compliance-review/SKILL.md) | "규제 검토", "PG 등록 대상인지 확인", "PCI-DSS 대응해야 하나" | 전자금융거래법 등록/인허가·PCI-DSS·개인정보보호법(국외이전)·특정금융정보법(AML/VASP)·해외 PSP 연동(외국환거래법·GDPR) 5개 규제 도메인 체크리스트로 H/M/L 등급 판정. `references/regulatory-checklist.md`에 77개 항목이 확인 상태(✅/⚠️/❓)와 함께 정리돼 있습니다. **법률 자문이 아닙니다.** |
| [`qa-review`](skills/qa-review/SKILL.md) | "설계 문서 정합성 검증", "이 문서들 앞뒤가 맞나 확인", "QA 해줘" | 설계 문서 간 경계면 교차검증(PASS/FIX/GAP 판정). **"문서 QA"만 다룹니다** — 실제 구현 코드가 스펙대로 동작하는지 확인하는 "개발 QA"는 범위 밖입니다(아래 "설계 범위" 참조). |

`compliance-review`·`qa-review` 2개는 팀 전체를 돌리지 않고 단독으로도 쓸 수 있습니다 — 오케스트레이터 안에서 각각 compliance·qa 팀원이 내부적으로 호출하는 것과 동일한 스킬입니다. `launch-orchestrator`는 그 자체가 오케스트레이터라 "단독 실행"이라는 구분이 의미가 없습니다.

## 사용법

1. 설치 — 둘 중 편한 방법으로:
   - **플러그인으로 설치**: `.claude-plugin/plugin.json`·`marketplace.json`이 모두 포함돼 있어 아래 두 방법 다 됩니다. 플러그인 이름은 `payments`이며, 설치되면 스킬은 `payments:launch-orchestrator` 형태로 노출됩니다.
     - marketplace 등록: `claude plugin marketplace add <이 저장소 경로 또는 git URL>` 후 `claude plugin install payments@payments` (실사용 검증 완료 — `claude plugin details payments@payments`로 스킬 3개·에이전트 10개 정상 인식 확인됨)
     - 또는 등록 없이 바로 로드: `claude --plugin-dir <이 저장소 경로>`
   - **디렉터리 복사**: 이 저장소의 `agents/`·`skills/` 안의 파일들을 대상 프로젝트의 `.claude/agents/`·`.claude/skills/`로 복사합니다(marketplace 등록 없이 즉시 사용 가능, 이식 방법은 아래 "내 프로젝트로 이식하기" 참조).
2. 세션에 입력합니다:
   ```
   새 결제 서비스 'xyz-pay' 출시 준비 문서 만들어줘 — 해외 카드사 다이렉트 연동, Q3 목표
   ```
3. `launch-orchestrator` 스킬이 트리거되어 10명 팀이 자동으로 조율하며 작업합니다.
4. 결과는 `{service-root}/docs/{kickoff,constraints,architecture,session-design,operations,specs,adr,reports}/`에 생성됩니다.

### 후속 작업

이미 만든 서비스에 대해 "제약분석만 다시", "아키텍처 수정해줘" 같은 요청도 같은 스킬이 처리합니다 — 영향받는 팀원만 골라 재실행합니다(전체 팀 재소집 아님). 자세한 절차는 오케스트레이터 스킬의 "Phase 0: 컨텍스트 확인" 참조.

## 전제 조건

이 하네스는 문서를 어디에 둘지 **특정 폴더 구조를 강제하지 않습니다.** 오케스트레이터 Phase 0가 실행 시점에 `{service-root}`(문서 루트 경로)를 프로젝트에 이미 있는 컨벤션에서 찾거나, 없으면 사용자에게 직접 물어서 정합니다 — Obsidian PARA vault든 일반 저장소든 그대로 동작합니다.

다만 아래 두 가지는 Obsidian vault 관례를 선택적으로 반영합니다(없어도 핵심 동작에는 지장 없습니다):

- 허브 노트(`{service-root}/payment-{service-slug} 프로젝트 현황.md`) 생성은 별도 플러그인([`sr-obsidian`](https://github.com/SeokRae/sr-obsidian))의 `hub` 스킬(대안: `scaffold`)을 재사용하도록 설계돼 있습니다 — 없으면 Phase 1에서 사용자가 건너뛰도록 선택하면 됩니다.
- 문서 프론트매터(`type: literature`, `wbs-group`, `wbs-label` 등)는 Dataview 기반 자동 Gantt 집계를 활용하는 vault 관례입니다. 이런 관례가 없는 프로젝트라면 프론트매터 필드는 무시해도 동작에는 지장이 없습니다.

## 내 프로젝트로 이식하기

1. `agents/payment-*.md` 10개를 대상 프로젝트의 `.claude/agents/`로, `skills/{launch-orchestrator,compliance-review,qa-review}/` 3개 디렉터리를 대상 프로젝트의 `.claude/skills/`로 복사합니다(Claude Code는 프로젝트 로컬 에이전트·스킬을 `.claude/agents/`·`.claude/skills/`에서 찾으므로, 복사할 때 이 경로로 넣어야 합니다).
2. 문서 루트(`{service-root}`)는 Phase 0에서 실행 시점에 물어보므로 파일을 미리 고칠 필요는 없습니다 — 다만 항상 같은 위치를 쓰고 싶다면 각 에이전트 파일의 `{service-root}` 표기를 고정 경로로 바꿔도 됩니다.
3. Obsidian 허브 노트를 쓰지 않는 프로젝트라면 `payment-planner.md`·`launch-orchestrator/SKILL.md`의 "허브 노트 확인" 단계를 생략하도록 안내만 해두면 됩니다(Phase 1에서 사용자가 건너뛰기를 선택할 수 있음).
4. `compliance-review`의 체크리스트는 **한국 결제 규제 기준**(2026-08-01 리서치 시점)입니다 — 해외 시장·다른 업종이면 `references/regulatory-checklist.md` 자체를 새로 조사해 교체해야 합니다.
5. `CLAUDE.md`에 하네스 포인터를 프로젝트 컨벤션에 맞춰 추가합니다(이 저장소의 `CLAUDE.md` 형식 참고).

## 설계 범위 — "문서 QA"와 "개발 QA"

이 하네스는 **설계 문서만** 다룹니다. `payment-launch-qa`가 하는 검증은 "세션설계 시나리오가 스펙에 API로 다 옮겨졌나" 같은 문서 대 문서 정합성이며, **실제로 구현된 코드가 그 스펙대로 동작하는지는 확인하지 않습니다** — 코드는 이 저장소 밖의 별도 코드베이스에 있고, 그건 별도의 개발+QA 하네스가 필요한 영역입니다. `payment-ops-designer`가 만드는 "테스트 전략" 문서가 그 다음 단계(개발 QA)로 넘어가는 핸드오프 산출물 역할을 하도록 설계돼 있습니다.

## 설계 원칙

- **생산자와 검증자를 분리한다** — `payment-launch-qa`는 문서를 만들지 않고 교차검증만 합니다. 스스로 만든 문서를 스스로 검증하면 놓치는 게 그대로 남습니다.
- **점진적 QA** — 전체 완성 후 한 번에 몰아서 검증하지 않고, 각 생산자가 끝나는 즉시 바로 확인합니다. 문제를 일찍 잡을수록 되돌리는 비용이 작습니다.
- **재시도에는 상한이 있다** — QA 재검증은 최대 2회, 초과하면 강제 통과 처리 후 미결로 남깁니다. 무한 루프보다 "사람이 확인할 목록"을 남기는 쪽을 택합니다.
- **역할이 다르면 에이전트를 분리한다** — 인프라 구축(물리·실행형)과 어플리케이션 설계(논리·문서형)는 성격이 다르다는 판단으로 별도 에이전트로 나눴습니다. 규제·정책 판단도 마찬가지로 전문화했습니다.
- **스킬(어떻게)과 에이전트(누가)를 분리한다** — 규제 판단 로직(`compliance-review`)과 QA 판정 로직(`qa-review`)은 독립 스킬로 빼서, 팀 밖에서 단독으로도 쓸 수 있게 했습니다.

## 에이전트 상세

에이전트와 스킬은 역할이 다릅니다 — **에이전트는 "누가 하는가"** (역할·권한이 고정된 팀원, `Agent` 도구로 스폰하면 독립된 컨텍스트에서 작업), **스킬은 "어떻게 하는가"** (여러 호출자가 공유해 꺼내 쓰는 절차서, 현재 컨텍스트에 지침만 로드). 각 에이전트 정의 파일(`agents/*.md`) 자체가 그 역할 전용 업무 매뉴얼이며, 그중 규제 판정(`compliance-review`)과 QA 판정(`qa-review`) 두 절차만 팀 밖에서도 재사용 가치가 있다고 보고 별도 스킬로 분리했습니다. 아래는 10개 에이전트 각각의 요약이며, 팀 통신 프로토콜·에러 핸들링 전체는 링크한 원본 파일을 참고하세요.

### 1. [`payment-planner`](agents/payment-planner.md) — 기획·제약분석
- **착수 조건**: 없음 — 팀의 시작점
- **핵심 역할**: 사업 배경·목표·이해관계자 킥오프 문서 + H/M/L 등급 제약분석
- **핵심 원칙**: 착수 차단급(H) 제약을 최우선 표면화, 심층 규제 판단은 compliance에 위임, 미확인 값은 추정하지 않고 "(미결)" 표기
- **입력 → 출력**: 사용자 제공 서비스 개요 → `docs/kickoff/`, `docs/constraints/01-*`
- **주요 협업**: 완료 즉시 H급 제약·확정 스코프를 compliance에 SendMessage 전달(G1 구현)

### 2. [`payment-compliance-analyst`](agents/payment-compliance-analyst.md) — 규제 준수 심층 검토
- **착수 조건**: G1 — planner의 킥오프+제약분석 완료
- **핵심 역할**: 5개 규제 도메인(전자금융거래법·PCI-DSS·개인정보보호법·특금법·해외연동) 심층 검토 1회, 이후 전 팀원 규제 질의에 상시 대기
- **핵심 원칙**: 법률 자문 아님 caveat 명시, `compliance-review` 스킬 체크리스트만 근거로 판정(자체 지식으로 새 규제 판단 생성 금지)
- **입력 → 출력**: `docs/constraints/01-*` → `docs/constraints/02-*`(H급 규제 항목·법무 확인 필요 목록 포함)
- **주요 협업**: 완료 시 H급 규제 항목을 architect에 전달(G2 구현 — 이 팀의 핵심 안전장치)

### 3. [`payment-architect`](agents/payment-architect.md) — 논리 컴포넌트 아키텍처
- **착수 조건**: G2 — compliance 산출물에 "법무 확인 필요 목록" 섹션 존재
- **핵심 역할**: 논리 컴포넌트 구조 + 컴포넌트 간 통신 방향(호출 방향·동기/비동기) 정의. 물리 인프라는 다루지 않음
- **핵심 원칙**: 트랜잭션 상태 전이 기준으로 경계를 나눔, 구체적 엔드포인트 시그니처는 확정하지 않고 spec에 기준선만 넘김
- **입력 → 출력**: `docs/constraints/`(제약+규제) → `docs/architecture/01-*`
- **주요 협업**: 완료 즉시 infra·session·ops 3명에게 동시 SendMessage(팬아웃 트리거)

### 4. [`payment-infra-builder`](agents/payment-infra-builder.md) — 물리 인프라 설계
- **착수 조건**: G3 — QA 체크포인트1(아키텍처 정합성) 완결, session·ops와 3갈래 병렬
- **핵심 역할**: 서버 사이징·이중화·존 구성·DR을 "바로 구축에 쓰는 단일 기준 문서"로 관리(설계와 실행을 분리하지 않음)
- **핵심 원칙**: 이중화는 기본값이 아니라 결정 사항(미확정이면 ADR 요청), 트래픽 근거 없이 서버 대수를 지어내지 않음
- **입력 → 출력**: architect 컴포넌트 구조 + compliance DR/이중화 규제 판정 → `docs/architecture/02-*`
- **주요 협업**: ops와 서버 존 구성 ↔ 운영 전환 일정 상호 확인

### 5. [`payment-session-designer`](agents/payment-session-designer.md) — 트랜잭션 세션 설계
- **착수 조건**: G3, infra·ops와 3갈래 병렬
- **핵심 역할**: 승인·취소·구독·가상계좌·웹훅 등 시나리오별 요청-응답 시퀀스 + 예외 흐름 설계
- **핵심 원칙**: 정상 흐름만 그리지 않음(타임아웃·중복요청·웹훅유실 등 예외 필수 포함), architect 경계 밖 컴포넌트를 임의로 추가하지 않음
- **입력 → 출력**: `docs/architecture/` → `docs/session-design/`
- **주요 협업**: spec의 직접 상류(하드 의존) — 이 팀원의 QA 통과가 G4(spec 착수)의 조건

### 6. [`payment-ops-designer`](agents/payment-ops-designer.md) — 운영·정산·테스트 전략
- **착수 조건**: G3, infra·session과 3갈래 병렬
- **핵심 역할**: 정산 전략·모니터링/알람·이슈대응 프로세스·테스트 전략·운영 전환 로드맵
- **핵심 원칙**: 정산 정합성은 "승인 건수=정산 건수"가 아니라 상태 전이 정합성으로 검증, 테스트 전략은 향후 "개발 QA" 핸드오프 산출물이므로 실행 가능한 구체성까지 요구
- **입력 → 출력**: `docs/architecture/` → `docs/operations/`
- **주요 협업**: infra와 존 구성 ↔ 운영 로드맵 상호 확인, session과 타임아웃 값 상호 확인

### 7. [`payment-spec-writer`](agents/payment-spec-writer.md) — 인터페이스·API 스펙
- **착수 조건**: G4 — session의 QA(세션설계 정합성) 완결
- **핵심 역할**: architect의 통신 방향 표를 실제 엔드포인트(URL·메서드·요청/응답 필드)로 구체화 + 외부연동 규격 + 구현 API 리스트
- **핵심 원칙**: session 시나리오에 없는 API를 임의로 추가하지 않음(먼저 session에 확인), 외부연동 미확보 필드는 "(미확보)"로 표기
- **입력 → 출력**: `docs/session-design/` + architect 통신방향 표 → `docs/specs/`
- **주요 협업**: session의 직접 하류(단독 실행 불가)

### 8. [`payment-launch-qa`](agents/payment-launch-qa.md) — 설계 문서 정합성 검증
- **착수 조건**: 게이트 없음 — architect·infra·session·ops·spec 각 완료 직후 자동 트리거(5개 체크포인트)
- **핵심 역할**: `qa-review` 스킬로 경계면 교차검증 수행, PASS/FIX/GAP 판정
- **핵심 원칙**: "존재 확인"이 아니라 "교차 비교", 재검증 최대 2회 후 강제 PASS, REDO급 불일치는 판정 없이 즉시 리더 에스컬레이션
- **입력 → 출력**: 인접 문서 쌍 → `_launch-workspace/qa/{NN}-*.md`(`docs/`에는 쓰지 않음 — 감사 기록)
- **주요 협업**: FIX/GAP 시 해당 생산자에게 직접 SendMessage, 불일치가 미결정 때문이면 adr에 먼저 기록 요청 후 재검증

### 9. [`payment-decision-recorder`](agents/payment-decision-recorder.md) — ADR 기록(adr)
- **착수 조건**: 고정 게이트 없음 — 전 팀원 SendMessage에 상시 대기(on-call)
- **핵심 역할**: 되돌리기 어려운 결정을 ADR로 구조화, 상태(`proposed`→`accepted`/`rejected`/`superseded`) 관리
- **핵심 원칙**: "되돌리기 어려운가"만 기록 기준(사소한 선택은 반려), 번호는 순차 증가·재사용 금지, 상충 시 삭제 대신 `superseded` 처리
- **입력 → 출력**: 팀원의 SendMessage(결정 컨텍스트) → `docs/adr/adr-{NNN}-*.md`
- **주요 협업**: reporter가 Phase 4 시작 시 ADR 전체 목록을 요청

### 10. [`payment-launch-reporter`](agents/payment-launch-reporter.md) — 구축 보고서 종합
- **착수 조건**: G5 — 인프라·운영·스펙 3개 QA 체크포인트 모두 완결
- **핵심 역할**: 전 산출물 + QA 결과 + ADR 이력을 표준 목차(배경/개요/아키텍처요약/로드맵/현황/리스크레지스터)로 종합
- **핵심 원칙**: 개별 문서 복붙 금지(관점 재구성), 리스크 레지스터는 전 팀원 미결 항목을 누락 없이 통합, "ADR 필요 여부 검토함" 문구와 adr의 실제 목록을 대조해 누락 감지
- **입력 → 출력**: `docs/{kickoff..specs,adr}/` + QA 기록 → `docs/reports/{service-slug}-플랫폼-구축-보고서.md`
- **주요 협업**: 팀의 마지막 단계 — 리더가 이 산출물을 최종 결과로 사용자에게 보고

## 라이선스

[MIT](LICENSE)
