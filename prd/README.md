# PRD 인덱스 — 2026-08-02 하네스 사이클 분석 후속 작업

2026-08-02 세션에서 uncommitted 변경분(agents 8개·skills 3개·CLAUDE.md·README.md·신규 ISSUES.md·plugin.json)을
전수 분석하고 CLAUDE.md 체인지로그 주장을 교차검증한 결과 도출된 후속 작업 5건이다. 각 PRD는 별도 세션에서
이 문서만 읽고 바로 착수할 수 있도록 배경·근거·요구사항·완료 기준을 자기완결적으로 담았다.

| # | 제목 | 우선순위 | 상태 | 파일 |
|---|---|---|---|---|
| 01 | ADR 필요 여부 검토 지시를 생산자 7개 에이전트에 반영 | **High** | ✅ 완료 (2026-08-02) | [01-adr-review-instruction-in-producers.md](01-adr-review-instruction-in-producers.md) |
| 02 | "드라이런 1회 검증" 등 검증 상태 표현 정정 | Medium | ✅ 완료 (2026-08-02) | [02-verification-status-wording.md](02-verification-status-wording.md) |
| 03 | plugin.json 구조 검증 및 실사용 로드 확인 | Medium | ✅ 완료 (2026-08-02) — 구조 전면 재작성 + 실사용 로드 검증까지 완료 | [03-plugin-manifest-validation.md](03-plugin-manifest-validation.md) |
| 04 | SKILL.md 본문에 남은 툴 종속 서술을 부록 A로 완전 이동 | Low | ✅ 완료 (2026-08-02) | [04-skill-md-tool-coupling-cleanup.md](04-skill-md-tool-coupling-cleanup.md) |
| 05 | 나열형 경로의 `{service-root}/` 접두어 누락 정리 | Low | ✅ 완료 (2026-08-02) | [05-service-root-prefix-consistency.md](05-service-root-prefix-consistency.md) |

## 착수 순서 권장

01 → 02 → 03 → 04 → 05. 01은 실제 동작(ADR 추적 로직)에 영향을 주는 유일한 항목이라 가장 먼저,
04·05는 문서 정합성 문제라 가장 나중에 처리해도 무방하다.

## 처리 후

각 항목이 완료되면 이 PRD 파일은 지우거나 `## 완료` 절을 추가해 남겨두고,
`ISSUES.md`에 동일 항목이 있다면 함께 정리한다(GitHub remote가 생기기 전까지는 ISSUES.md가 1차 트래킹 파일).
