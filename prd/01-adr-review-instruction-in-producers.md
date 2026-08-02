# PRD-01: ADR 필요 여부 검토 지시를 생산자 7개 에이전트에 반영

**우선순위:** High · **작성일:** 2026-08-02 · **예상 규모:** 작음 (7개 파일, 각 1~3줄 추가)

## 배경

`payment-launch-orchestrator` 하네스는 되돌리기 어려운 결정을 `payment-decision-recorder`(adr)가 ADR로
기록하도록 설계돼 있다. `adr`은 고정 태스크가 없는 on-call 팀원이라, 리더가 TaskList만으로는
"정말 ADR로 남겨야 할 결정이 나왔는데 아무도 요청하지 않고 지나갔는지" 확인할 방법이 없다는 문제가
있었다 (`ISSUES.md` "4. on-call 팀원(adr·compliance) 참여 여부를 리더가 확인할 방법 없음").

이를 해결하기 위해 최근 커밋되지 않은 변경에서 다음 장치를 추가했다:

- 산출물을 만드는 7개 에이전트(`planner`·`compliance`·`architect`·`infra`·`session`·`ops`·`spec`)가
  완료 알림에 `"ADR 필요 여부 검토함(불필요 / adr에게 요청함 — adr-{번호})"` 한 줄을 반드시 포함한다
  — `.claude/skills/payment-launch-orchestrator/SKILL.md:93`
- `payment-launch-reporter`가 이 한 줄들을 모아뒀다가 adr의 전체 ADR 목록과 대조해 빠진 게 있으면
  리더에게 즉시 보고한다 — `.claude/agents/payment-launch-reporter.md:37`
- 리더가 Phase 4(통합 확인)에서 "불필요" 판단들이 실제로 타당한지 훑어본다
  — `.claude/skills/payment-launch-orchestrator/SKILL.md:124`

`ISSUES.md`의 "해결됨" 섹션은 이 항목을 완료로 표시했다.

## 문제

**위 장치는 수신 측(reporter)과 오케스트레이터 요약(SKILL.md)에만 구현돼 있고, 정작 그 한 줄을
실제로 작성해야 할 발신 측 — 7개 생산자 에이전트 자신의 `.md` 파일 — 에는 이 지시가 전혀 없다.**

각 서브에이전트는 자신의 `.md` 파일만 시스템 프롬프트로 받는다. SKILL.md는 리더(오케스트레이터)가
읽는 문서이지 서브에이전트 자신이 읽는 문서가 아니다. 따라서 지금 상태로 하네스를 실행하면:

1. 7개 생산자는 완료 시 `"- payment-launch-reporter에게: 산출물 완료 시 SendMessage로 알림"`
   (각 파일에 이미 있는 문구, 예: `.claude/agents/payment-planner.md`)만 보고 알림을 보내며,
   "ADR 필요 여부 검토함" 줄을 포함할 이유를 자기 파일 어디에서도 찾을 수 없다.
2. reporter가 하려는 "완료 알림에 포함된 한 줄을 모아 adr 목록과 대조"는 비교할 데이터 자체가
   없어 사실상 아무 일도 하지 않는다.

### 검증 근거 (2026-08-02 재확인)

```
$ grep -ln "ADR 필요 여부" .claude/agents/payment-planner.md .claude/agents/payment-compliance-analyst.md \
    .claude/agents/payment-architect.md .claude/agents/payment-infra-builder.md \
    .claude/agents/payment-session-designer.md .claude/agents/payment-ops-designer.md \
    .claude/agents/payment-spec-writer.md
(빈 결과 — 7개 파일 중 어디에도 매치 없음)
```

## 목표

7개 생산자 에이전트 파일 각각의 "팀 통신 프로토콜"(또는 이에 준하는, reporter에게 완료 알림을
보내는 절차를 서술한 절)에 "ADR 필요 여부 검토함" 한 줄을 완료 알림 포맷에 포함하라는 지시를 추가한다.

## Non-goals

- reporter 쪽 대조 로직 변경 (이미 완료돼 있음, 손대지 않는다)
- adr·compliance on-call 참여 모델 자체의 재설계
- ADR 판정 기준(무엇이 "되돌리기 어려운 결정"인가) 자체를 새로 정의 — 기존 `payment-decision-recorder.md`
  기준을 그대로 참조

## 요구사항

1. 아래 7개 파일 각각에서, 기존의 "완료 시 reporter에게 SendMessage로 알림" 문구 근처에
   다음 문구를 추가한다 (SKILL.md:93, reporter.md:37과 **정확히 동일한 표기**를 써야
   reporter의 대조 로직이 파싱 가능하다):

   > 완료 알림에는 "ADR 필요 여부 검토함(불필요 / adr에게 요청함 — adr-{번호})" 한 줄을 반드시
   > 포함한다. 이번 산출물에 되돌리기 어려운 결정(예: 이중화 여부, 동기/비동기 처리 방식,
   > 규제 등급 판정처럼 번복 비용이 큰 선택)이 포함됐다고 판단되면 `payment-decision-recorder`에게
   > SendMessage로 ADR 작성을 요청하고 그 번호를 적는다. 아니면 "불필요"로 적는다.

   대상 파일:
   - `.claude/agents/payment-planner.md`
   - `.claude/agents/payment-compliance-analyst.md`
   - `.claude/agents/payment-architect.md`
   - `.claude/agents/payment-infra-builder.md`
   - `.claude/agents/payment-session-designer.md`
   - `.claude/agents/payment-ops-designer.md`
   - `.claude/agents/payment-spec-writer.md`

2. 문구 삽입 위치는 각 파일의 기존 구조(팀 통신 프로토콜 절, 산출물 완료 알림 관련 bullet)를
   따르되, 파일마다 절 이름이 조금씩 다를 수 있으니 `grep -n "reporter에게" <파일>`로 정확한
   위치를 먼저 확인한 뒤 삽입한다.

3. `adr-{번호}` 표기 규칙(예: `adr-001`, `adr-{slug}-01` 등 실제 `payment-decision-recorder.md`가
   ADR 파일에 매기는 번호 형식)을 `payment-decision-recorder.md`에서 확인해 일치시킨다 — 형식이
   다르면 reporter가 "adr 목록에 없다"고 오탐할 수 있다.

## 완료 기준

- [ ] `grep -l "ADR 필요 여부" .claude/agents/payment-{planner,compliance-analyst,architect,infra-builder,session-designer,ops-designer,spec-writer}.md` 결과 7개 파일 전부 매치
- [ ] 삽입한 문구의 괄호 안 표기(`불필요 / adr에게 요청함 — adr-{번호}`)가 SKILL.md:93·reporter.md:37과 문자 그대로 일치
- [ ] (권장) 생산자 에이전트 1명을 짧은 가짜 산출물로 드라이런해 실제로 완료 알림에 그 줄이 포함되는지 확인
- [ ] `ISSUES.md`의 해당 "해결됨" 항목 설명을 "reporter·SKILL.md뿐 아니라 7개 생산자 파일까지 반영 완료"로 갱신하거나, 이 PRD를 완료 표시 후 정리

## 영향 파일

수정: 위 7개 agent 파일
참조만 (변경 없음): `.claude/skills/payment-launch-orchestrator/SKILL.md`, `.claude/agents/payment-launch-reporter.md`, `.claude/agents/payment-decision-recorder.md`

## 완료 (2026-08-02)

7개 생산자 에이전트 파일 모두 "팀 통신 프로토콜" 절의 reporter 알림 bullet에 "ADR 필요 여부 검토함(불필요 / adr에게 요청함 — adr-{번호})" 문구를 SKILL.md:90·reporter.md:37과 동일한 표기로 추가했다. `grep -l "ADR 필요 여부" .claude/agents/payment-{planner,compliance-analyst,architect,infra-builder,session-designer,ops-designer,spec-writer}.md` 결과 7건 매치로 확인. 실제 드라이런(권장 항목)은 미실시.
