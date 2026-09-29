# CLAUDE.md — CHRONO KNIGHT 개발 규칙

이 저장소는 Android 게임 "CHRONO KNIGHT"(가제)를 Claude Code 로 개발한다.
기획의 단일 진실 공급원은 `docs/` 이다. 구현이 문서와 달라지면 문서를 함께 수정한다.

## 문서
- 게임 규칙: `docs/01-game-design.md`
- 경제·소환 수치: `docs/02-economy-gacha.md` (수치는 `app/src/main/assets/balance/*.json` 과 일치해야 함)
- UI·디자인 토큰: `docs/03-design-ux.md`
- 아키텍처: `docs/04-technical-design.md`
- 보안·법규: `docs/05-security-compliance.md`
- 마일스톤·완료 조건: `docs/06-roadmap.md`

## 명령어 (프로젝트 생성 후)
- 전체 검사: `./gradlew check`
- 순수 로직 테스트: `./gradlew :engine:test :game:test`
- 디버그 빌드: `./gradlew assembleDebug`
- 밸런스 시뮬레이션: `./gradlew :engine:simulate` (리포트 → `docs/balance-report.md`)

## 아키텍처 규칙
- 모듈 의존 방향: `app → game → engine`. `engine`, `game` 에 `android.*` import 금지.
- 게임 루프(틱/렌더) 안에서 객체 할당 금지 — 풀과 `FloatArray` 기반 SoA 사용.
- 전투 난수는 반드시 `engine.core.Rng` 사용 (결정성 유지). 소환 난수는 `SecureRandom`.
- 밸런스 수치는 하드코딩 금지 → `assets/balance/*.json` + `BalanceConfig`.
- 세이브 스키마 변경 시 `schemaVersion` 증가 + 마이그레이션 + 이전 버전 픽스처 테스트 추가.
- 사용자 문자열은 `strings.xml`(ko, en) 에만 둔다.

## 보안 규칙
- 키스토어, 서비스 계정 JSON, 운영 광고 ID, 비밀번호 커밋 금지 (`local.properties` / GitHub Secrets).
- 결제 지급은 반드시 서명 검증 → Ledger 중복 확인 → 지급 → 세이브 커밋 → consume/acknowledge 순서.
- 소환은 결과 결정 → 세이브 커밋 → 연출 순서.
- 새 권한 추가 금지 (필요 시 `docs/05-security-compliance.md` §8 먼저 갱신).

## 작업 방식
- 마일스톤 단위로 작업하고 DoD 테스트를 먼저 작성/통과시킨다.
- 커밋 전 `./gradlew check` 통과 필수.
