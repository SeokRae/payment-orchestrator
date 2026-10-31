# 게이트 기록과 스폰 머리말

`launch-orchestrator` 리더가 쓰는 서식 두 가지와, 작업 목록 도구를 쓰지 않는 근거를 담습니다. 게이트 규범은 SKILL.md의 Phase 2에, 호출 예시는 부록 A에 있어요.

## 1. 게이트 기록: `{service-root}/_launch-workspace/gates.md`

리더가 Phase 2 시작 때 만들고, 게이트를 판정하거나 팀원을 스폰하거나 완료 보고를 받을 때마다 줄을 덧붙입니다. 지난 줄은 고치지 않아요. 리더 컨텍스트가 끊기면 이 파일과 `qa/`, `user-checkpoints/`를 Read해 마지막 줄부터 이어갑니다.

```markdown
# {서비스명} 게이트 기록

- service-root: {경로}
- service-slug: {slug}
- 실행 유형: 초기 실행 | 새 실행

## 게이트와 스폰

| 시각 | 구분 | 판정 | 근거 | 리더 조치 |
|------|------|------|------|----------|
| {HH:MM} | 시작 | - | Phase 1 입력: _launch-workspace/00_input/개요.md | 스폰 adr, planner |
| {HH:MM} | G1 | 열림 | docs/kickoff/01-{slug}-착수-개요.md, docs/constraints/01-{slug}-제약-분석.md 존재 | 스폰 compliance |
| {HH:MM} | G2 | 열림 | docs/constraints/02-{slug}-규제-준수-체크리스트.md에 `## 법무 확인 필요 목록` 있음 | 스폰 architect |
| {HH:MM} | C1 요청 | - | architect 완료 보고, docs/architecture/01-{slug}-컴포넌트-역할-정의.md 존재 | 스폰 qa |
| {HH:MM} | G3 | 열림 | qa/01-*.md 마지막 판정에 FIX, GAP, 보류 없음 | 스폰 infra, session, ops |
| {HH:MM} | C3 무효화 | - | qa 보고: session 문서 변경 | G4 이후 게이트 판정 보류 |

구분: 시작 | G1~G5 | C1~C5 요청 | C{N} 완결 | C{N} 무효화 | 실패 | 종료
판정: 열림 | 닫힘({사유}) | -

## 완료 보고

| 시각 | 팀원 | 산출물 | ADR 검토 | 하류 전달 사항 |
|------|------|--------|----------|----------------|
| {HH:MM} | planner | docs/kickoff/..., docs/constraints/01-... | 불필요 | H급 제약 {N}건, 확정 스코프 요약 |
```

- `근거` 칸에는 리더가 직접 Glob, Grep, Read로 확인한 파일과 그 결과를 씁니다. 팀원의 자기보고만으로 게이트를 열지 않습니다
- `완료 보고` 표는 reporter가 G5 뒤에 읽는 입력입니다. 7명의 `ADR 검토` 값이 여기 모입니다
- 영구 실패로 확정한 팀원은 `게이트와 스폰` 표에 `실패` 줄을 남기고, 그 팀원을 비교 대상으로 삼는 체크포인트를 qa에 "미수행(대상 부재)"으로 처리하도록 알립니다

## 2. 스폰 프롬프트 머리말

리더는 모든 스폰 프롬프트 맨 앞에 아래 블록을 채워 넣습니다. 부분 재실행 때도 같은 블록을 쓰되, 둘째 줄을 "이번은 부분 재실행이라 다른 팀원은 없다. 모든 보고는 main으로"로 바꿉니다.

```text
[팀 통신 머리말]
- 너의 이름: {name}. 리더 주소: main
- 팀원 주소: planner, compliance, architect, infra, session, ops, spec, qa, adr, reporter
- service-root: {경로} / service-slug: {slug}
- 게이트 근거: {이 팀원을 연 게이트와 근거 파일 경로. 게이트가 없으면 "없음"}
- 보낼 상대가 아직 없어 SendMessage가 실패하면, 그 내용을 main에게 보낸다. 리더가 상대를 스폰할 때 넘긴다
- 일이 남은 채 턴을 끝낼 때는 main에게 대기 보고를 먼저 보낸다
- 메시지 첫 줄 형식:
  - 완료 보고 (→ main): 완료: {이름} | 산출물: {경로, 쉼표 구분} | ADR 검토: 불필요 또는 adr-{번호}
    둘째 줄부터 하류 전달 사항(다음 팀원이 알아야 할 H급 항목, 확정 구조 요약)
  - 대기 보고 (→ main): 대기: {이름} | 상대: {이름} | 사유: {한 줄}
  - QA 지적 (qa → 생산자): FIX C{N}-{항목}: 위치 {파일}:{절} | 주장 {문서가 말하는 값} | 근거 {비교 문서}:{절} | 기대값 {값}
    GAP이면 첫 단어만 GAP
  - 재검증 요청 (생산자 → qa): 재검증 요청 C{N}: {고친 파일}:{절}
  - 변경 통지 (→ qa): 변경 통지 C{N}: {바뀐 파일}:{절} | 사유: {한 줄}
  - QA 보고 (qa → main): QA C{N}: PASS 또는 강제 PASS 또는 미수행(대상 부재) 또는 무효화 | 기록: {경로}
```


## 3. 작업 목록 도구 노출 조건

SKILL.md "전제 도구"가 작업 목록 도구를 게이트에 쓰지 않는 근거입니다.

- Claude Code v2.1.268 이후 기본값으로는 `TaskCreate`, `TaskGet`, `TaskUpdate`, `TaskList`, `TodoWrite`가 Opus 4.7, Sonnet 4.6, Haiku 4.5 이하 모델에만 들어갑니다
- 그 밖의 모델(Opus 4.8, Sonnet 5, Fable 5, Mythos 5 계열 이후 등)은 다음 중 하나로 켜야 해요: `CLAUDE_CODE_ENABLE_TODO_TOOLS=1`, `--allowedTools`에 도구 이름 지정, `--tools`에 나열, Agent SDK의 `allowedTools`나 `tools` 옵션
- 백그라운드 세션과 클라우드 세션은 모델과 무관하게 이 도구를 제공합니다. 같은 플러그인을 클라우드에서 돌리면 도구가 보일 수 있다는 뜻이고, 그래서 "보여도 게이트 판정에는 쓰지 않는다"를 규칙으로 둡니다
- 서브에이전트는 부모 세션에 도구가 있을 때만 받습니다. 서브에이전트가 어떤 모델로 돌든 같아요
- 에이전트 팀 실험 플래그는 켜는 방법에 들어 있지 않습니다. 2026-10-31 세션에서도 플래그를 켠 상태로 도구가 없었어요
- 출처: https://code.claude.com/docs/en/tools-reference ("Task tool availability" 절), https://code.claude.com/docs/en/whats-new/2026-w33 (Opus 4.8, Sonnet 5, Fable 5, Mythos 5 이후 계열에서 빠졌고 `CLAUDE_CODE_ENABLE_TODO_TOOLS=1`로 다시 켠다는 공지)
