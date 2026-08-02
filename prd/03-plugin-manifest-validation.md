# PRD-03: plugin.json 구조 검증 및 실사용 로드 확인

**우선순위:** Medium · **작성일:** 2026-08-02 · **예상 규모:** 작음 (검증 작업 + 필요 시 매니페스트 수정)

## 배경

이번 사이클에서 `.claude-plugin/plugin.json`이 새로 추가됐다 (아직 git add 되지 않은 untracked
상태). README.md 37번째 줄은 이를 근거로 "플러그인으로 설치" 방법을 안내한다:

> "`.claude-plugin/plugin.json`이 포함돼 있어, 로컬 경로에서 `--plugin-dir <이 저장소 경로>`로
> 로드하거나, marketplace에 등록해 `/plugin marketplace add`로 설치할 수 있습니다."

현재 `plugin.json` 내용:

```json
{
  "name": "payment-launch-orchestrator",
  "displayName": "Payment Launch Orchestrator",
  "version": "0.1.0",
  "description": "...",
  "author": { "name": "SeokRae" },
  "license": "MIT",
  "keywords": ["payment", "pg", "compliance", "launch", "orchestrator", "qa"],
  "agents": "./.claude/agents/",
  "skills": "./.claude/skills/"
}
```

## 문제

이 매니페스트가 **실제로 로드되는지 한 번도 검증되지 않았다.** 2026-08-02 조사에서
사용자 로컬에 설치된 다른 실제 Claude Code 플러그인 30여 종(공식 Anthropic 다수 + 사용자
본인의 sr-harness·sr-blog-harness·flowcast·harness-marketplace)과 대조한 결과, 이 매니페스트는
셋 다 어디에서도 선례를 찾지 못한 패턴을 쓴다:

1. **top-level `displayName`** — 조사한 어떤 실제 plugin.json에도 없음. 유일하게 비슷한 사례는
   사용자의 sr-harness `.codex-plugin/plugin.json` 안 `interface.displayName`인데, 이건 Codex
   전용 중첩 객체지 Claude Code 매니페스트의 최상위 필드가 아니다.
2. **`agents`/`skills`를 단일 디렉터리 문자열로 지정** — 비교 가능한 유일한 사례인 사용자
   본인의 sr-harness Claude Code용 plugin.json은 `skills`를 개별 스킬 폴더 경로의 **배열**로
   선언한다 (예: `["./skills/start", "./skills/brainstorm", ...]`). 단일 디렉터리 문자열이 아니다.
3. **`.claude/agents/`, `.claude/skills/`처럼 `.claude/` 하위 중첩 경로** — 조사한 실제 플러그인
   30여 종은 예외 없이 플러그인 루트 바로 아래 `agents/`, `skills/`를 두거나, 필드 자체를
   생략하고 기본 경로에 의존한다. `.claude/` 프리픽스를 쓰는 사례는 하나도 없었다.

이 구조는 "프로젝트에 직접 복사해서 쓰기"와 "플러그인으로 설치하기" 두 용도를 `.claude/` 하위
경로 하나로 겸하려는 의도로 보이나(README의 "디렉터리 복사" 방법과 동일 경로를 씀), 실제
`--plugin-dir` 로더가 이 구조를 인식하는지는 저장소 안의 정적 증거만으로는 확인할 수 없다.

부수적으로: `plugin.json`의 `author` 객체에 `email`이 없다. 사용자 본인의 다른 플러그인 3개
(sr-harness, sr-blog-harness, flowcast)는 모두 `author.email`(`kslbsh@gmail.com`)을 포함한다.

## 목표

`plugin.json`이 실제 Claude Code 세션에서 `--plugin-dir`로 로드되고, 10개 agent와 3개 skill이
모두 정상 인식되는지 실제로 확인한다. 안 되면 검증된 다른 플러그인(sr-harness 등)이 쓰는
표준 구조로 맞춘다.

## Non-goals

- marketplace.json 신규 작성 (이 저장소를 marketplace에 등록하는 것은 별도 작업 — 필요 여부만
  이 PRD에서 판단하고 실제 작성은 범위 밖)
- `.claude/agents/`·`.claude/skills/` 자체를 플러그인 루트로 옮기는 대규모 리팩터 — 검증 결과
  현재 구조로도 로드된다면 손대지 않는다

## 요구사항

1. 실제 세션에서 로드 시도:
   ```
   claude --plugin-dir /Users/sr/IdeaProjects/claude-tools/payment-launch-orchestrator
   ```
   (정확한 플래그/명령은 당시 Claude Code CLI 버전의 `claude --help`로 재확인)
2. 로드된 세션에서 10개 `payment-*` 에이전트, 3개 스킬(`payment-launch-orchestrator`,
   `payment-compliance-review`, `payment-qa-review`)이 실제로 노출되는지 확인
   (예: `/agents`, 스킬 트리거 문구로 테스트 요청 넣어보기)
3. **성공한 경우**: 이 PRD를 "검증 완료(날짜)"로 닫고, `CLAUDE.md` 변경이력에 "plugin.json
   실사용 로드 검증 완료" 한 줄을 추가한다. `author.email`을 다른 플러그인과 일관되게 추가한다.
4. **실패한 경우**: sr-harness의 실제 동작하는 plugin.json 구조(스킬을 배열로 명시, 필요하면
   경로를 플러그인 루트 바로 아래로 조정)를 참고해 `plugin.json`을 수정하고, 1~2번을 재시도한다.
   이 경우 README.md의 설치 안내도 실제 동작하는 방법에 맞춰 함께 수정한다.
5. README.md 38번째 줄의 "marketplace에 등록해 설치할 수 있습니다" 서술이 이 저장소만으로
   가능한 것처럼 오해되지 않도록, marketplace 등록에는 별도 `marketplace.json`이 필요하다는
   점을 한 문장으로 명확히 구분해 적는다 (marketplace.json을 실제로 만들지는 Non-goals에 따라
   별도 판단).

## 완료 기준

- [ ] `--plugin-dir`로 로드 시도 결과가 기록됨 (성공/실패 어느 쪽이든)
- [ ] 성공 시: 10개 agent + 3개 skill 전부 인식 확인, `author.email` 추가, CLAUDE.md에 검증
      완료 기록
- [ ] 실패 시: 표준 구조로 수정 후 재검증 성공, 또는 실패 원인을 ISSUES.md에 새 항목으로 이관
- [ ] README.md의 marketplace 관련 서술이 실제 저장소 상태(marketplace.json 유무)와 일치

## 영향 파일

수정 가능성: `.claude-plugin/plugin.json`, `README.md`, `CLAUDE.md`
참조: 사용자 로컬의 다른 plugin.json 예시 (sr-harness 등, 정확한 경로는 `~/.claude/plugins/` 하위 검색)

## 완료 (2026-08-02)

**실패 경로로 진행**(요구사항 4) — 조사 결과 `agents` 필드 자체가 Claude Code 플러그인 스키마에 없다는 결정적 근거를 찾았다: flowcast(같은 작성자의 실제 배포 중인 플러그인)에 자체 매니페스트 검증 스크립트(`scripts/validate_plugin_manifest.py`)가 포함돼 있었고, 여기서 허용 필드를 `name/version/description/author/homepage/repository/license/keywords/skills`로 명시적으로 제한하고 있었다 — `agents`·`displayName` 둘 다 없다. 커스텀 서브에이전트는 플러그인 루트의 `agents/` 디렉터리에서 파일명=frontmatter `name` 규칙으로 자동 인식된다(flowcast의 `agents/diagram-router.md` 등으로 실증).

이에 따라:
- `agents/`·`skills/`를 저장소 루트로 이동(`.claude/` 접두어 제거)
- `plugin.json`을 표준 필드만으로 재작성(`agents`·`displayName` 제거, `skills`를 배열로, `author.email` 추가)
- 스킬 3개는 `payment-` 접두어도 제거(`launch-orchestrator`·`compliance-review`·`qa-review`) — 별도 요청으로 추가 진행
- README의 "디렉터리 복사" 설치법을 `agents/`→`.claude/agents/`, `skills/`→`.claude/skills/`로 이름을 바꿔 복사하는 절차로 재작성

**실사용 로드 검증(요구사항 1·2)도 완료**: `claude plugin validate .`(Claude Code 내장 검증기)를 실행해 구조적 문제를 확인하는 과정에서 `compliance-review`·`qa-review` 두 스킬의 YAML frontmatter가 실제 파싱 에러 상태였던 것을 발견 — description을 큰따옴표로 감싸지 않아 내부 인용부호·`Keywords:` 콜론 때문에 깨져 있었다(수정 전에는 두 스킬 모두 플러그인 로드 시 메타데이터가 통째로 비워지는 상태). 수정 후 `claude plugin validate .`는 경고 1건(CLAUDE.md는 플러그인 루트에서 프로젝트 컨텍스트로 로드되지 않는다는 안내, 조치 불필요)만 남기고 통과. 이어서 `claude --plugin-dir . -p "..."`로 실제 세션을 띄워 스킬 3개(`payments:launch-orchestrator`·`payments:compliance-review`·`payments:qa-review`)와 에이전트 10개(`payments:payment-architect` 등) 전부 정상 노출을 직접 확인했다.

`author.email` 추가 완료, CLAUDE.md 검증 완료 기록 완료. README.md의 marketplace 서술도 "marketplace 등록에는 별도 marketplace.json이 필요하다(이 저장소엔 아직 없음)"는 문장으로 명확히 구분해뒀다.
