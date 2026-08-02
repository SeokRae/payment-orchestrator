# PRD-05: 나열형 경로의 `{service-root}/` 접두어 누락 정리

**우선순위:** Low · **작성일:** 2026-08-02 · **예상 규모:** 매우 작음 (4개 파일, 각 1줄)

## 배경

이번 사이클(및 이전 커밋 7b9a489)에서 Obsidian PARA vault 특정 경로(`20-areas/payment/...`)
의존을 없애고 `{service-root}` 플레이스홀더로 일반화하는 작업을 했다. 이 작업 자체는 저장소
전체에서 완전히 일관되게 반영됐다(2026-08-02 확인, `grep -rn "20-areas"` 결과 서술적 언급
2건 제외 실사용 0건).

## 문제

다만 한 줄에 경로를 여러 개 나열하는 문장에서, **`{service-root}/` 접두어가 첫 번째 경로에만
붙고 같은 줄의 나머지 경로에는 빠지는 패턴**이 남아 있다. 문맥상 같은 `service-root` 하위
경로로 읽히긴 하지만, 표기가 일관되지 않아 향후 다른 사람이 "두 번째 경로는 절대경로거나
다른 루트인가?" 하고 헷갈릴 여지가 있다.

### 확인된 사례 (2026-08-02)

```
.claude/agents/payment-compliance-analyst.md:28
- 입력: `{service-root}/docs/kickoff/`, `docs/constraints/01-{service-slug}-제약-분석.md` (payment-planner 산출물)

.claude/agents/payment-architect.md:24
- 입력: `{service-root}/docs/kickoff/`, `docs/constraints/01-{service-slug}-제약-분석.md`(payment-planner), `docs/constraints/02-{service-slug}-규제-준수-체크리스트.md`(payment-compliance-analyst)

.claude/agents/payment-spec-writer.md:22
- 입력: `{service-root}/docs/session-design/` (payment-session-designer 산출물), `docs/architecture/01-{service-slug}-컴포넌트-역할-정의.md`(특히 "컴포넌트 간 통신 방향" 섹션)

.claude/agents/payment-infra-builder.md:23
- 입력: `{service-root}/docs/architecture/01-{service-slug}-컴포넌트-역할-정의.md`(payment-architect), `docs/constraints/02-{service-slug}-규제-준수-체크리스트.md`(payment-compliance-analyst, DR·이중화 관련 항목)
```

네 곳 모두 같은 패턴: 줄의 첫 경로만 `{service-root}/`로 시작하고, 쉼표로 이어지는 나머지
경로는 `docs/...`로 시작해 접두어가 없다.

이번 diff가 새로 만든 문제는 아니고(경로 일반화 작업 중에는 손대지 않고 넘어간 기존 상태),
diff 범위 밖이라 이번 분석에서 "완료"로 카운트하지 않았다.

## 목표

경로를 여러 개 나열하는 문장에서 표기 규칙을 통일한다. 두 방식 중 하나를 택해 4개 파일에
일관 적용한다:

- **방식 A (각 경로에 명시)**: 나열된 모든 경로에 `{service-root}/`를 반복해서 붙인다.
  예: `` `{service-root}/docs/kickoff/`, `{service-root}/docs/constraints/01-{service-slug}-제약-분석.md` ``
- **방식 B (문두에 한 번만 명시)**: 문장 앞에 "이하 모두 `{service-root}/` 하위"라고 한 번
  밝히고, 나열된 경로들은 접두어 없이 `docs/...`로만 쓴다.

방식 A가 각 항목이 코드 인용구(백틱)로 독립적으로 복사/참조되는 이 문서 스타일과 더 잘
맞고, 다른 곳(예: README.md, SKILL.md 데이터 흐름 절)도 대부분 각 경로에 매번
`{service-root}/`를 명시하는 방식을 쓰고 있어 **방식 A를 기본값으로 권장**한다. 다만 착수
전에 SKILL.md·README.md의 지배적 표기 방식을 다시 한번 grep으로 확인해 다수결을 따른다.

## Non-goals

- `{service-root}` 플레이스홀더 자체의 설계(이미 완료, Phase 0 컨텍스트 확인 로직 — 손대지
  않는다)
- 이 4개 파일 외 다른 경로 표기 문제 재조사 (필요하면 별도 PRD로)

## 요구사항

1. `grep -rn '{service-root}/[^,]*, \`\?docs/' .claude/agents/*.md .claude/skills/*/SKILL.md`
   (또는 유사 패턴)으로 같은 문제가 있는 다른 파일이 더 없는지 착수 전에 한 번 더 전수
   확인한다 — 위 4곳은 2026-08-02 시점 확인된 사례이며 그 사이 추가 변경이 있었을 수 있다.
2. 채택한 표기 규칙(방식 A 권장)으로 위 4개 파일의 해당 줄을 수정한다.
3. 수정 후 같은 파일 안의 다른 "입력"/"출력" 줄들과 표기가 일관되는지 육안으로 재확인한다.

## 완료 기준

- [ ] 4개 파일의 해당 줄에서 나열된 모든 경로가 동일한 표기 규칙(방식 A 또는 B)을 따름
- [ ] `grep -rn` 재확인 결과 같은 파일 내 다른 줄과 표기 불일치 없음
- [ ] (선택) 표기 규칙을 CLAUDE.md나 SKILL.md 어딘가에 한 줄로 문서화해 향후 새 에이전트
      파일 작성 시 같은 문제가 재발하지 않게 함

## 영향 파일

수정: `.claude/agents/payment-compliance-analyst.md`, `.claude/agents/payment-architect.md`,
`.claude/agents/payment-spec-writer.md`, `.claude/agents/payment-infra-builder.md`

## 완료 (2026-08-02)

착수 전 재확인(요구사항 1)에서 원래 4곳 외에 `.claude/skills/payment-launch-orchestrator/SKILL.md`의 산출물 저장 경로 표(planner 행)에서도 같은 패턴을 추가로 발견해 총 5곳에 방식 A(각 경로에 `{service-root}/` 반복)를 적용했다. `grep -rn '{service-root}/[^,]*, \`docs/'` 재확인 결과 매치 0건.
