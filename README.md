# payment-launch-orchestrator

> 신규 결제/PG 서비스 출시를 기획부터 구축 보고서까지, **10명의 AI 에이전트**가 팀으로 자동화하는 [Claude Code](https://claude.com/claude-code) 하네스입니다.

기획(킥오프·제약분석) → 규제 준수 심층 검토 → 논리 아키텍처 설계 → (물리 인프라 설계 ∥ 트랜잭션 세션설계 ∥ 운영전략, 각 단계 직후 QA 게이트) → 인터페이스 스펙 → 구축 보고서까지, 결제 서비스 하나를 출시 준비하는 데 필요한 설계 문서 세트를 한 번에 만듭니다.

> 원본은 개인 Obsidian PARA vault(`sr-labs`)에서 `harness:harness` 메타 스킬로 반복 확장하며 만들어졌습니다 — 규제 전문가 분리, 인프라/어플리케이션 역할 분리, 점진적 QA 게이트 순서로 4단계에 걸쳐 진화했습니다. 이 저장소는 그 결과물을 독립적으로 재사용할 수 있게 옮긴 것입니다.

## 팀 구성

| 에이전트 | 역할 | 산출물 |
|---|---|---|
| [`payment-planner`](.claude/agents/payment-planner.md) | 킥오프(사업배경·이해관계자) + 제약분석(H/M/L 등급) | `docs/kickoff/`, `docs/constraints/` |
| [`payment-compliance-analyst`](.claude/agents/payment-compliance-analyst.md) | 규제 준수 심층 검토(전자금융거래법·PCI-DSS·개인정보보호법·특금법·해외연동) 1회 + 상시 대기 | `docs/constraints/` (02번 보강) |
| [`payment-architect`](.claude/agents/payment-architect.md) | 논리 컴포넌트 아키텍처 설계 (경계·인증/승인/정산 흐름) | `docs/architecture/01-*` |
| [`payment-infra-builder`](.claude/agents/payment-infra-builder.md) | 물리 인프라 설계 (서버 사이징·이중화·존·DR) — 실행(ITSM·발급)은 범위 밖 | `docs/architecture/02-infra-sizing.md` |
| [`payment-session-designer`](.claude/agents/payment-session-designer.md) | 트랜잭션 세션 시나리오 설계 (승인·취소·구독·가상계좌·웹훅 등) | `docs/session-design/` |
| [`payment-ops-designer`](.claude/agents/payment-ops-designer.md) | 정산·모니터링·이슈대응·테스트 전략 | `docs/operations/` |
| [`payment-spec-writer`](.claude/agents/payment-spec-writer.md) | 인터페이스·API 스펙 | `docs/specs/` |
| [`payment-launch-qa`](.claude/agents/payment-launch-qa.md) | 생산자 완료 직후 경계면 교차검증(생성-검증 게이트), 불일치는 즉시 되돌려보냄(최대 2회) | `_launch-workspace/qa/` (문서 산출물 아님) |
| [`payment-decision-recorder`](.claude/agents/payment-decision-recorder.md) | 되돌리기 어려운 결정을 ADR로 기록, 상시 대기 | `docs/adr/` |
| [`payment-launch-reporter`](.claude/agents/payment-launch-reporter.md) | 전 팀원 산출물 + QA 결과를 구축 보고서로 종합 | `docs/reports/` |

오케스트레이터는 [`.claude/skills/payment-launch-orchestrator/SKILL.md`](.claude/skills/payment-launch-orchestrator/SKILL.md) 하나입니다. 팀 구성·의존관계(`TaskCreate`의 `depends_on` 체인)·에러 핸들링을 정의하며, 리더 역할은 별도 에이전트 없이 세션 자체가 맡습니다.

**의존 관계:** `planner` → `compliance` → `architect` → **QA** → (`infra` ∥ `session` ∥ `ops`) → **QA(각각)** → `spec`(session QA 통과 후) → **QA** → `reporter`(3개 QA 통과 후). `adr`·`compliance`(2단계 이후)는 고정 순서 없이 전 팀원의 SendMessage 요청에 상시 응답합니다.

### 독립 스킬 2개

팀 전체를 돌리지 않고 단독으로도 쓸 수 있습니다.

- [`payment-compliance-review`](.claude/skills/payment-compliance-review/SKILL.md) — 전자금융거래법 등록/인허가·PCI-DSS·개인정보보호법(국외이전)·특정금융정보법(AML/VASP)·해외 PSP 연동(외국환거래법·GDPR) 5개 규제 도메인 체크리스트. `references/regulatory-checklist.md`에 76개 항목이 확인 상태(✅/⚠️/❓)와 함께 정리돼 있습니다.
- [`payment-qa-review`](.claude/skills/payment-qa-review/SKILL.md) — 설계 문서 간 경계면 교차검증(PASS/FIX/GAP 판정). **"문서 QA"만 다룹니다** — 실제 구현 코드가 스펙대로 동작하는지 확인하는 "개발 QA"는 범위 밖입니다(아래 "설계 범위" 참조).

## 사용법

1. 이 플러그인을 Claude Code 프로젝트에 설치하거나, `.claude/agents/`·`.claude/skills/` 전체를 대상 프로젝트로 복사합니다.
2. 세션에 입력합니다:
   ```
   새 결제 서비스 'xyz-pay' 출시 준비 문서 만들어줘 — 해외 카드사 다이렉트 연동, Q3 목표
   ```
3. `payment-launch-orchestrator` 스킬이 트리거되어 10명 팀이 자동으로 조율하며 작업합니다.
4. 결과는 `20-areas/payment/{서비스}/docs/{kickoff,constraints,architecture,session-design,operations,specs,adr,reports}/`에 생성됩니다.

### 후속 작업

이미 만든 서비스에 대해 "제약분석만 다시", "아키텍처 수정해줘" 같은 요청도 같은 스킬이 처리합니다 — 영향받는 팀원만 골라 재실행합니다(전체 팀 재소집 아님). 자세한 절차는 오케스트레이터 스킬의 "Phase 0: 컨텍스트 확인" 참조.

## 전제 조건 (중요 — 이식 전 필독)

이 하네스는 **Obsidian PARA vault 구조를 전제**로 설계됐습니다. 그대로 쓰려면:

- 대상 저장소가 `20-areas/payment/{서비스}/` 형태의 폴더 구조를 쓰거나, 그렇게 매핑할 의향이 있어야 합니다.
- 허브 노트(`{서비스} 프로젝트 현황.md`)·WBS 생성은 이 저장소에 없는 별도 플러그인([`sr-obsidian`](https://github.com/SeokRae/sr-obsidian))의 `hub`·`wbs` 스킬을 재사용하도록 설계돼 있습니다 — 없으면 Phase 1에서 사용자에게 먼저 만들지 확인하는 절차가 동작하지 않으니, 해당 스킬을 함께 설치하거나 오케스트레이터의 Phase 1을 프로젝트에 맞게 고쳐야 합니다.
- 문서 프론트매터(`type: literature`, `wbs-group`, `wbs-label` 등)는 vault의 `wbs.md` DataviewJS 자동 Gantt 집계 관례를 따릅니다. 이 관례가 없는 프로젝트라면 프론트매터 스펙만 남기고 무시해도 동작에는 지장이 없습니다.

## 내 프로젝트로 이식하기

1. `.claude/agents/payment-*.md` 10개와 `.claude/skills/{payment-launch-orchestrator,payment-compliance-review,payment-qa-review}/` 3개 디렉터리를 그대로 복사합니다.
2. 각 에이전트 파일의 출력 경로(`20-areas/payment/{service-slug}/docs/...`)를 자기 프로젝트의 문서 위치 규칙으로 바꿉니다.
3. `payment-planner.md`·`payment-launch-orchestrator/SKILL.md`의 "허브 노트 확인" 단계를 자기 프로젝트에 맞는 절차로 바꾸거나 제거합니다.
4. `payment-compliance-review`의 체크리스트는 **한국 결제 규제 기준**(2026-08-01 리서치 시점)입니다 — 해외 시장·다른 업종이면 `references/regulatory-checklist.md` 자체를 새로 조사해 교체해야 합니다.
5. `CLAUDE.md`에 하네스 포인터를 프로젝트 컨벤션에 맞춰 추가합니다(이 저장소의 `CLAUDE.md` 형식 참고).

## 설계 범위 — "문서 QA"와 "개발 QA"

이 하네스는 **설계 문서만** 다룹니다. `payment-launch-qa`가 하는 검증은 "세션설계 시나리오가 스펙에 API로 다 옮겨졌나" 같은 문서 대 문서 정합성이며, **실제로 구현된 코드가 그 스펙대로 동작하는지는 확인하지 않습니다** — 코드는 이 저장소 밖의 별도 코드베이스에 있고, 그건 별도의 개발+QA 하네스가 필요한 영역입니다. `payment-ops-designer`가 만드는 "테스트 전략" 문서가 그 다음 단계(개발 QA)로 넘어가는 핸드오프 산출물 역할을 하도록 설계돼 있습니다.

## 설계 원칙

- **생산자와 검증자를 분리한다** — `payment-launch-qa`는 문서를 만들지 않고 교차검증만 합니다. 스스로 만든 문서를 스스로 검증하면 놓치는 게 그대로 남습니다.
- **점진적 QA** — 전체 완성 후 한 번에 몰아서 검증하지 않고, 각 생산자가 끝나는 즉시 바로 확인합니다. 문제를 일찍 잡을수록 되돌리는 비용이 작습니다.
- **재시도에는 상한이 있다** — QA 재검증은 최대 2회, 초과하면 강제 통과 처리 후 미결로 남깁니다. 무한 루프보다 "사람이 확인할 목록"을 남기는 쪽을 택합니다.
- **역할이 다르면 에이전트를 분리한다** — 인프라 구축(물리·실행형)과 어플리케이션 설계(논리·문서형)는 성격이 다르다는 판단으로 별도 에이전트로 나눴습니다. 규제·정책 판단도 마찬가지로 전문화했습니다.
- **스킬(어떻게)과 에이전트(누가)를 분리한다** — 규제 판단 로직(`payment-compliance-review`)과 QA 판정 로직(`payment-qa-review`)은 독립 스킬로 빼서, 팀 밖에서 단독으로도 쓸 수 있게 했습니다.

## 라이선스

[MIT](LICENSE)
