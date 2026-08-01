# payment-launch-orchestrator

## 하네스: payment-launch

**목표:** 신규 결제/PG 서비스 출시를 기획(킥오프·제약분석)부터 규제 준수 심층 검토·논리/물리 아키텍처 설계·트랜잭션 세션설계·운영전략·인터페이스 스펙·점진적 QA·구축보고서까지 10명의 에이전트 팀으로 자동화.

**트리거:** 신규 결제 서비스 출시·PG 서비스 구축·결제 서비스 런칭 준비 관련 요청 시 `payment-launch-orchestrator` 스킬을 사용하라. 규제만 단독 검토할 땐 `payment-compliance-review`, 설계 문서 정합성만 검증할 땐 `payment-qa-review`를 직접 트리거해도 된다. 단순 질문은 직접 응답 가능.

**변경 이력:**

| 날짜 | 변경 내용 | 대상 | 사유 |
|---|---|---|---|
| 2026-08-01 | 초기 추출 — 10 에이전트 + 3 스킬을 `sr-labs` Obsidian vault에서 독립 저장소로 분리 | 전체 | 재사용·버전 관리·다른 프로젝트 이식을 위해 별도 배포 |

## 참고

- 원본: `sr-labs`(개인 Obsidian PARA vault)에서 `harness:harness` 메타 스킬로 반복 확장하며 구축(기획→규제→인프라 분리→점진적 QA 4단계).
- 이 저장소는 그 산출물을 독립 실행 가능하도록 옮긴 것이며, vault 전제 조건은 [README.md](README.md)의 "전제 조건" 참조.
