# 하네스 개선 필요 항목 (로컬 트래킹)

> GitHub remote가 아직 없어 임시로 이 파일에 기록한다. remote가 생기면 남은 항목을 GitHub Issue로 옮기고 이 파일에서는 지운다.

## 해결됨

- ~~`scenario-{시나리오명}.md` 파일명에 service-slug 누락~~ → `payment-session-designer.md`에서 `scenario-{service-slug}-{시나리오명}.md`로 수정
- ~~게이트가 영구적으로 충족 불가능하다고 판단될 때의 에스컬레이션 규칙 부재~~ → orchestrator SKILL.md 에러 핸들링 표에 행 추가 완료 (**2026-08-02 재확인**: 실제 파일(170행)에 반영돼 있음을 재확인 — 번호 목록에 중복 남아있던 항목 정리)
- ~~on-call 팀원(adr·compliance) 참여 여부를 리더가 확인할 방법 없음~~ → 생산자 완료 알림에 "ADR 필요 여부 검토함" 한 줄 의무화 + reporter가 adr 목록과 대조 + 리더 Phase 4에 교차확인 단계 추가. **2026-08-02 갱신**: 최초 반영은 reporter·SKILL.md에만 있고 정작 그 문구를 써야 할 7개 생산자 agent 파일에는 지시가 없어 실효성이 없었음 → 7개 파일 전체에 문구 추가 완료로 재확인
- ~~QA 재검증 "최대 2회" 규칙의 카운팅 단위 미정의 (심각 — 종료 보장 없음)~~ → "같은 불일치"를 같은 항목(필드/컴포넌트/섹션) 기준으로 정의 + 항목당 최초 1회+재검증 최대 2회(총 3회 확인)에 더해 체크포인트당 재검증 왕복 총 5회 상한을 신설(`qa-review/SKILL.md`, `payment-launch-qa.md`, `launch-orchestrator/SKILL.md`). "에러 흐름 2" 테스트 시나리오와 규칙 서술의 계산 방식 불일치도 함께 정리. **2026-08-02 독립 검증 완료(RESOLVED)**
- ~~생산자 영구 실패 시 reporter 착수 게이트(G5)가 영구 미해소 (심각 — 교착)~~ → 생산자 영구 실패 확정 시 해당 체크포인트를 "미수행(대상 부재)"으로 종결·완결로 간주해 G5를 비롯한 하류 게이트 산정에서 제외하는 규칙을 "완결"의 정의·에러 핸들링 표(`launch-orchestrator/SKILL.md`)·`payment-launch-qa.md`·`payment-launch-reporter.md`(완료 알림 대기 예외)에 추가. **2026-08-02 독립 검증 완료(RESOLVED)** — 잔여 참고: 상류(architect 등) 생산자가 영구 실패하면 하류가 원본 없이 시작하는 문제는 이번 이슈(G5 교착)와는 다른 사안으로 별도 관찰됨
- ~~QA→ADR 요청 사이클에서 "결정 주체"가 정의돼 있지 않음 (심각 — 대기 사슬)~~ → qa가 adr에게 바로 ADR을 요청하지 않고, 먼저 결정 확정 책임자(관련 생산자, 애매하면 리더)에게 SendMessage로 확정을 요청한 뒤 확정된 내용을 받아 adr에게 전달하도록 `launch-orchestrator/SKILL.md`·`payment-launch-qa.md` 수정. **2026-08-02 독립 검증 완료(RESOLVED)**
- ~~QA 완결 이후 정당한 문서 변경이 기존 PASS 판정을 무효화하지 않음 (상태 불일치)~~ → "완결 후 전제 문서 변경 시 무효화(팀 실행 중)" 규범을 `launch-orchestrator/SKILL.md`에 추가하고, `payment-session-designer.md`·`payment-infra-builder.md`가 qa에게 변경을 통지 → qa가 1회 한정 재수행하는 절차로 연결. 팀 해체 후(Phase 5 이후) 갱신되는 경우는 Phase 0 "부분 재실행 절차"로 연결하도록 명시. **2026-08-02 독립 검증 완료(RESOLVED)**
- ~~부분 재실행(후속 작업) 시 QA 재검증과 팀 통신 상대(qa·adr·compliance·reporter)가 실제로는 없음 (상태 불일치)~~ → Phase 0 부분 재실행 절차에 (1) 재소집 프롬프트에서 "qa 재검증을 받는다"·"reporter에게 알린다" 류 지시를 따르지 말라고 명시, (2) 재작업 완료 후 리더가 직접 기존 QA 기록에 "재작업으로 무효화됨" 표기 + 체크포인트 재수행 여부 판단(필요 시 `qa-review` 단독 트리거) + 하류 영향·ADR·리포트 보완 절차를 추가. **2026-08-02 독립 검증 완료(RESOLVED)**
- ~~ADR이 superseded될 때 그 결정을 전제로 작성된 문서의 재확인 규칙 없음 (상태 불일치)~~ → `payment-decision-recorder.md`에 supersede 처리 시 원 ADR을 요청했던 팀원(특정 불가 시 리더)에게 SendMessage로 재확인을 요청하는 절차 추가. **2026-08-02 독립 검증 완료(RESOLVED)**
- ~~compliance의 on-call 발견사항이 최종 리스크 레지스터에 반영되는 경로가 없음 (상태 불일치)~~ → `payment-compliance-analyst.md`에 on-call 응답 중 H급/M급 신규 판정을 내리면 02번 문서("법무 확인 필요 목록" 포함)를 compliance 자신이 직접 갱신하도록 명시. **2026-08-02 독립 검증 완료(RESOLVED)**
- ~~팀원 간 양방향 상호 확인에 "정본(source of truth)"과 상한이 없음 (경미)~~ → session↔ops(정본: session 문서)·infra↔ops(정본: infra 문서) 소유권 + 3회 왕복 후 리더 에스컬레이션 규칙을 `launch-orchestrator/SKILL.md`·`payment-ops-designer.md`에 추가. **2026-08-02 갱신**: 최초 반영이 SKILL.md·ops-designer.md에만 있고 정작 정본을 쥔 `payment-session-designer.md`·`payment-infra-builder.md`에는 빠져 있어(PRD-01과 동일 유형의 결함) 독립 검증에서 PARTIAL 판정 → 두 파일에도 정본·왕복 상한 문구 추가해 재확인 완료
- ~~문서 간 표현 편차 — reporter 착수 조건, QA 카운팅 서술 (경미)~~ → `payment-launch-reporter.md`의 착수 조건을 `launch-orchestrator/SKILL.md`의 게이트 G5 참조로 통일(영구 실패 예외 포함), QA 재검증 표현을 "최초 검증 1회 + 재검증 최대 2회 = 총 3회 확인"으로 관련 파일 전체 통일. **2026-08-02 독립 검증 완료(RESOLVED)**
- ~~Phase 0 "새 실행" 분기의 문구 자기모순 (경미)~~ → "기존 docs를 건드리지 않고 ... (기존 docs는 사용자 확인 후 덮어쓸지 결정)"처럼 "안 건드림(고정)"과 "덮어쓸지 결정함(가변)"이 한 문장에서 충돌하던 서술을 "기본값은 보존 — 덮어쓰기는 사용자가 명시적으로 승인했을 때만 예외적으로 진행" 두 절로 분리해 명확화(`launch-orchestrator/SKILL.md` 43행). **2026-08-02 수정 완료**

## 1. 전체 파이프라인 end-to-end 실행 검증 — 부분 확인, 나머지는 미검증

- **부분 검증 완료**: `payment-launch-qa`의 판정 로직 자체는 단위 테스트로 확인했다 — 일부러 어긋나는 가짜 문서 2개(compliance의 "PAN 비저장" H급 항목 vs architect의 "카드번호 포함 로그 저장" 컴포넌트)를 주고 체크포인트 1을 수행시킨 결과, FIX 판정과 정확한 위치 인용(항목 ID·섹션명)을 정확히 해냈다. 의도적으로 심어두지 않은 부수 불일치(compliance 문서의 "✅ 확인됨" 표기가 실제로는 인프라 단계 미확정 상태와 어긋난다는 점)까지 스스로 짚어냈다.
- **여전히 미검증**: session→spec 핸드오프(architect의 "통신 방향" 표를 spec이 실제로 올바르게 엔드포인트로 구체화하는지), reporter의 리스크 레지스터 취합, QA 재검증 루프(FIX→수정→재검증)의 실제 왕복.
- **왜 전체 완주는 안 했나**: 전체 완주는 "하네스 검증"이 아니라 "가상 서비스 하나를 실제로 구현"하는 것에 가까워, 이번 세션에서도 사용자 지시로 범위에서 재차 제외했다(**2026-08-02 재확인**).
- **제안 방향**: 필요해지면 나머지 구간도 QA 테스트와 같은 방식(전체 완주가 아니라 해당 구간만 떼어낸 좁은 단위 테스트)으로 검증.
