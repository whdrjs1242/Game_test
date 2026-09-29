# 04. 기술 설계

> 핵심 제약: **Claude Code 로 100% 개발** → GUI 에디터 없이 CLI(Gradle)만으로 빌드·테스트·서명·배포가 가능해야 한다.
> 핵심 원칙: **게임 로직은 안드로이드와 분리된 순수 Kotlin** → 기기 없이 JVM 단위 테스트/시뮬레이션으로 검증.

---

## 1. 기술 스택 결정

| 영역 | 선택 | 선택 이유 / 비교 |
|---|---|---|
| 언어 | Kotlin 2.x | Android 1순위 언어, Google 라이브러리 1st-party 지원 |
| 메타 UI | Jetpack Compose (Material3 기반 커스텀 테마) | 선언형·텍스트 기반, 프리뷰/스크린샷 테스트 가능 |
| 게임 렌더링 | 자체 경량 2D 엔진: `SurfaceView` + 전용 게임 스레드 + `lockHardwareCanvas()` | 기하학 아트 스타일에 충분(적 250 + 투사체 400 @60fps 목표), 외부 엔진 의존 없음 |
| 게임 로직 | `:engine` 순수 Kotlin(JVM) 모듈 | 안드로이드 API 금지 → JUnit 으로 전투 전체 시뮬레이션 가능 |
| 직렬화 | kotlinx.serialization (JSON) | 세이브·밸런스 데이터 공용 |
| 비동기 | Kotlin Coroutines / Flow | |
| DI | 수동 DI (`AppContainer`) | 규모 대비 Hilt 불필요, 빌드 단순화 |
| 로컬 저장 | 파일(암호화 세이브) + Jetpack DataStore(설정) | Room 불필요 (세이브 1개 문서) |
| 구글 계정 연동 | **Google Play Games Services v2** (자동 로그인, 저장된 게임, 리더보드, 업적) | **무료**, 사용자 Drive 에 저장 → 서버 비용 0 |
| 결제 | Google Play Billing Library (출시 시점 최신 메이저, 8.x 이상) | |
| 광고 | Google Mobile Ads SDK (AdMob) + UMP(동의 관리) | 보상형만 |
| 원격 설정·분석 | Firebase Remote Config, Analytics, Crashlytics (Spark 무료 플랜) | 서버 없이 라이브 운영·크래시 추적 |
| 알림 | WorkManager + 로컬 Notification | 서버 푸시 불필요 |
| 빌드 | Gradle (Kotlin DSL) + Version Catalog, AGP 최신 안정 | |
| 품질 | ktlint, detekt, Android Lint, JUnit5, Robolectric, Compose UI test | |
| CI | GitHub Actions | 무료(공개/개인 한도 내) |

**대안 검토**
- Unity: 품질·생태계 우수하나 씬/프리팹이 에디터 중심 → Claude Code 단독 개발에 부적합, 라이선스 정책 리스크.
- Godot: 텍스트 씬이라 가능성 있으나 PGS·Billing 을 서드파티 플러그인에 의존, 안드로이드 빌드 템플릿 관리 부담.
- libGDX: 성능 우수, 다만 Compose 메타 UI 와의 결합 복잡. → 성능 병목 발생 시 **렌더러만 OpenGL ES 로 교체**할 수 있게 렌더러 인터페이스를 추상화해 둔다.

### 1.1 SDK 버전
| 항목 | 값 |
|---|---|
| `minSdk` | 26 (Android 8.0) — 국내 활성 기기 대부분 커버 |
| `targetSdk` / `compileSdk` | Google Play 요구 최신 (2026년 기준 36 이상) |
| ABI | arm64-v8a, armeabi-v7a (네이티브 코드 없음 → 순수 JVM) |
| 배포 형식 | Android App Bundle (AAB) + Play App Signing |

---

## 2. 프로젝트 구조

```
Game_test/
├─ settings.gradle.kts
├─ gradle/libs.versions.toml
├─ engine/                      # 순수 Kotlin(JVM) — Android 의존 금지
│  └─ src/main/kotlin/com/chronoknight/engine/
│     ├─ core/        GameLoop(고정 틱), Rng(시드), Vec2, Time
│     ├─ ecs/         EntityStore(SoA 배열), Pools
│     ├─ world/       Arena, SpatialHashGrid, Camera
│     ├─ combat/      Damage, StatusEffect, Projectile, Collision
│     ├─ enemy/       EnemyTypes, Spawner, BossPatterns
│     ├─ rune/        RuneDef, RuneClock, ReactionTable, RuneEffects
│     ├─ shadow/      ShadowRecorder, ShadowPlayer, MirrorKnightAi
│     ├─ run/         RunState, LevelUp, RunResult
│     └─ sim/         HeadlessSimulator, BotPolicy (밸런스 검증용)
├─ game/                        # 메타 게임 규칙 — 순수 Kotlin
│  └─ src/main/kotlin/com/chronoknight/game/
│     ├─ model/       SaveGame, Knight, Equipment, Currency, Quest...
│     ├─ economy/     Gacha, Merge, Shop, OfflineReward
│     ├─ balance/     BalanceConfig (JSON 로드·검증)
│     └─ progress/    Stage, Dungeon, Tower, Season
├─ app/                         # Android
│  └─ src/main/
│     ├─ kotlin/com/chronoknight/app/
│     │  ├─ di/        AppContainer
│     │  ├─ render/    GameSurfaceView, Renderer(Canvas), AtlasBaker, Particles
│     │  ├─ input/     FloatingJoystick
│     │  ├─ audio/     SfxPlayer, BgmPlayer
│     │  ├─ data/      SaveRepository, LocalSaveStore(암호화), CloudSaveStore(PGS)
│     │  ├─ platform/  PlayGames, Billing, Ads, RemoteConfig, Analytics, Notifications
│     │  ├─ security/  CryptoBox, IntegrityGuard, ObfuscatedInt
│     │  └─ ui/        theme/, lobby/, battle/, shop/, knight/, rune/, event/, settings/
│     ├─ assets/balance/*.json   # 밸런스 데이터
│     └─ res/                    # vector drawable, strings(ko, en), fonts
├─ tools/                       # 사운드 생성, 밸런스 리포트 스크립트
├─ docs/
└─ .github/workflows/ci.yml
```

의존 방향: `app → game → engine` (역방향 금지, detekt 규칙 + Gradle 모듈 경계로 강제)

---

## 3. 게임 엔진 설계

### 3.1 루프
- **고정 시뮬레이션 틱 30Hz**(33.3ms) + 렌더 60fps 보간 → 저사양에서도 결정적 결과, 배터리 절약.
- 게임 스레드: `while(running) { accumulate dt; while(acc >= TICK) sim.step(); render(alpha) }`
- 백그라운드 전환(`onPause`) 시 즉시 일시정지 + 스레드 정지.

### 3.2 성능 규칙 (필수)
| 규칙 | 내용 |
|---|---|
| 할당 금지 | 게임 루프 내 객체 생성 0 (Pool 사용, `FloatArray` SoA 구조) — 테스트에서 할당 측정 |
| 충돌 | 균일 격자 Spatial Hash (셀 = 적 평균 지름 2배) |
| 상한 | 적 250, 투사체 400, 파티클 600, 데미지 숫자 40 (초과 시 오래된 것부터 재활용) |
| 렌더 | 미리 베이크한 아틀라스 `Bitmap` + `drawBitmap`(src/dst rect), 발광은 가산 합성 스프라이트 |
| 품질 옵션 | 높음(60fps, 파티클 100%) / 보통(60fps, 50%) / 절전(30fps, 25%) — 첫 실행 시 벤치마크로 자동 선택 |
| 목표 | 중급 기기(스냅드래곤 7 계열)에서 60fps, 보급형(2020년대 초 저가형)에서 30fps 안정, 메모리 < 300MB |

### 3.3 결정성(Determinism)
- 전투 난수는 `Rng(seed)` (xoshiro128++ 등 경량 구현) 단일 소스. 시드는 전투 시작 시 기록.
- 동일 시드 + 동일 입력 로그 → 동일 결과. 이용처: 버그 재현, 밸런스 시뮬레이션, 리더보드 기록의 로컬 재검증.

### 3.4 룬 시계 구현 요약
```kotlin
class RuneClock(val slots: Array<RuneInstance?> = arrayOfNulls(12)) {
    var angle = 0f                 // 0..12 (칸 단위)
    var speed = 12f / 6f           // 칸/초 (6초에 한 바퀴)
    fun tick(dt: Float, ctx: CombatContext) {
        val prev = angle
        angle = (angle + speed * dt) % 12f
        forEachCrossedSlot(prev, angle) { idx ->
            val rune = slots[idx] ?: return@forEachCrossedSlot
            val prevRune = slots[(idx + 11) % 12]
            val reaction = ReactionTable.find(prevRune?.element, rune.element)
            val mult = (if (idx == 11) 2f else 1f) * resonanceMult(idx)   // idx 11 = 12시
            RuneEffects.fire(rune, reaction, mult, ctx)
        }
    }
}
```

---

## 4. 데이터 · 세이브

### 4.1 세이브 모델
```kotlin
@Serializable data class SaveGame(
    val schemaVersion: Int,          // 마이그레이션용
    val saveCounter: Long,           // 저장할 때마다 +1 (충돌 해결·롤백 탐지)
    val updatedAtEpochMs: Long,
    val progressScore: Long,         // 충돌 해결용 진행도 점수 (최고 스테이지*1e6 + 계정레벨*1e3 + ...)
    val knight: Knight, val inventory: Inventory, val wallet: Wallet,
    val runes: RuneProgress, val stages: StageProgress, val gacha: GachaState,
    val quests: QuestState, val season: SeasonState, val shadows: ShadowIndex,
    val purchases: PurchaseLedger,   // 처리 완료된 주문 ID 목록 (중복 지급 방지)
    val settings: GameplaySettings
)
```
- 그림자 경로 데이터는 세이브와 분리된 파일(`shadows/*.bin`)로 관리, 클라우드에는 최근 20개만 포함.
- 스키마 변경 시 `Migrations.kt` 에 `vN → vN+1` 함수 추가 + 이전 버전 세이브 픽스처로 테스트.

### 4.2 저장 계층
```
          SaveRepository (단일 진입점, 메모리 상의 SaveGame 소유)
           ├─ LocalSaveStore   : 암호화 파일, 원자적 쓰기(tmp → fsync → rename), 백업 2세대 보관
           └─ CloudSaveStore   : PGS Saved Games (Snapshot)
```
| 시점 | 로컬 저장 | 클라우드 저장 |
|---|---|---|
| 전투 종료, 소환, 결제 지급, 합성 등 중요 이벤트 | 즉시 | 디바운스 후 업로드 |
| 일반 변경 | 5초 디바운스 | — |
| 앱 백그라운드 전환 | 즉시 | 즉시(가능할 때) |
| 주기 | — | 최대 5분 간격 |

### 4.3 구글 계정 연동 — Google Play 게임즈 서비스 v2
- 앱 시작 시 PGS v2 **자동 로그인**(별도 로그인 버튼 불필요, 실패 시 설정 화면의 "로그인" 버튼으로 수동 시도).
- **저장된 게임(Snapshots)**: 슬롯 1개(`main`), 데이터 = 압축(Deflate) + 암호화된 세이브, 커버 이미지·설명(“챕터 5-3 · Lv.42”) 포함.
- Play Console 설정: 게임즈 서비스 프로젝트 생성, **저장된 게임 사용 설정 ON**, OAuth 동의 화면, 앱 서명 키 SHA-1 등록.
- 비용: 없음 (데이터는 사용자 Google Drive 의 앱 전용 숨김 영역에 저장).

**동기화 흐름**
```
앱 시작 → 로컬 로드(즉시 플레이 가능) → PGS 로그인 성공 시 클라우드 스냅샷 조회
  ├─ 클라우드 없음            → 로컬을 업로드
  ├─ 클라우드 == 로컬(동일 counter) → 아무것도 안 함
  └─ 서로 다름                → 충돌 해결
```
**충돌 해결 정책**
1. 같은 기기 계보(`deviceLineageId` 동일)이고 counter 가 큰 쪽 채택.
2. 다른 기기끼리 충돌 → `progressScore` 가 큰 쪽을 기본 선택으로 한 **선택 팝업** 표시
   (양쪽 요약: 최고 스테이지, 레벨, 크리스탈, 마지막 저장 시각).
3. 결제 기록(`PurchaseLedger`)과 유료 크리스탈 잔액은 **양쪽 합집합 기준으로 보정** — 결제 손실 방지.
4. 새 기기 설치 시: 클라우드 세이브 발견 → "이전 진행을 불러올까요?" 확인 후 복원.

> Android Auto Backup 은 Keystore 키가 기기 종속이라 복원 불가 → **세이브 파일은 백업 대상에서 제외**
> (`data_extraction_rules.xml`), 기기 이전은 PGS 클라우드로만 처리.

### 4.4 밸런스 데이터
- `assets/balance/*.json` (enemies, runes, reactions, stages, gacha, shop, rewards)
- 앱 시작 시 스키마 검증 → 실패 시 내장 기본값 사용 + Crashlytics 비치명 로그.
- Remote Config 는 **허용된 키(배율·이벤트 on/off·픽업 대상)만** 덮어쓸 수 있게 화이트리스트 제한.
  (확률표 원본은 앱 내 데이터로 두고 원격 변경 시 공시 절차를 거친다 — [05 문서](05-security-compliance.md) 참고)

---

## 5. 결제 (Google Play Billing)

```
구매 버튼 → BillingClient.launchBillingFlow
  → onPurchasesUpdated(PURCHASED)
     1) 서명 검증 (로컬, 공개키) — 실패 시 지급 안 함
     2) orderId 가 PurchaseLedger 에 없으면 → 아이템 지급 + Ledger 기록 → 세이브 즉시 커밋(로컬+클라우드)
     3) 소모성: consumeAsync / 비소모성·구독: acknowledgePurchase
  → 앱 시작·재개 시 queryPurchasesAsync 로 미처리 구매 복구 (지급 누락 방지)
```
- 상품: 소모성(크리스탈·패키지), 비소모성(광고 제거, 스타터 팩), 월정액은 **소모성 30일권**으로 구현
  (구독보다 구현·환불 처리가 단순. 구독 전환은 라이브 단계에서 검토).
- PENDING 상태(현금 결제 등) 처리, 환불 대응: Voided Purchases API 는 서버가 필요하므로 1단계는 미적용, 2단계(Firebase Functions)에서 검토.
- 테스트: Play Console 라이선스 테스터 + 테스트 카드.

---

## 6. 광고 (AdMob)
- UMP SDK 로 동의 폼(EEA/UK 등 필요 지역) → 동의 후 광고 SDK 초기화.
- 보상형 광고 1개 광고 단위 + 사전 로드 1개 유지, 실패 시 지수 백오프 재시도.
- 보상 지급은 `onUserEarnedReward` 콜백 기준, 일일 한도는 세이브에 기록.
- 광고 제거권 보유 시 SDK 호출 없이 즉시 보상.
- 아동 대상 아님 설정(Families 정책 비대상), 광고 콘텐츠 등급 상한 설정(예: T 등급 이하).

---

## 7. 분석 · 운영
| 이벤트 | 파라미터 |
|---|---|
| `tutorial_step` | step |
| `run_start` / `run_end` | stage, result, duration, level, runes, reactions_count |
| `level_up_choice` | offered[3], picked, slot, auto_placed |
| `gacha_pull` | banner, count, results_grade |
| `purchase` | sku, price (Play 자동 수집과 중복 확인) |
| `ad_reward` | placement |
| `shadow_overtake` | stage, stacks |
- 개인 식별 정보 전송 금지. 퍼널: 설치 → 튜토리얼 완료 → 1-5 → 첫 소환 → 첫 결제.

---

## 8. 테스트 전략 (Claude Code 가 자율 검증 가능하도록)
| 레벨 | 도구 | 대상 |
|---|---|---|
| 단위 | JUnit5 (`:engine`, `:game`) | 룬 시계 발동 순서, 반응 표, 데미지 공식, 소환 확률(100만 회), 천장, 합성, 오프라인 보상, 세이브 마이그레이션, 충돌 해결 |
| 시뮬레이션 | `HeadlessSimulator` | 스테이지 클리어율 곡선, 한 판 평균 레벨, 성능(틱당 처리 시간) |
| 결정성 | 같은 시드 2회 실행 결과 해시 비교 | |
| 안드로이드 | Robolectric + Compose UI test | 화면 전환, 결제 흐름(가짜 BillingClient), 세이브 저장/복원 |
| 스크린샷 | Compose 스크린샷 테스트(Roborazzi 등) | 디자인 회귀 |
| 수동 | 실기기 체크리스트 (`docs/qa-checklist.md`) | 결제·PGS·광고 실제 연동 |

---

## 9. CI/CD
```
GitHub Actions (push / PR):
  ktlint → detekt → :engine:test :game:test → :app:testDebugUnitTest → lintDebug → assembleDebug
  (주 1회) 밸런스 시뮬레이션 → docs/balance-report.md 아티팩트
릴리스 (태그 v*):
  bundleRelease (업로드 키는 GitHub Secrets) → Play Console 내부 테스트 트랙 업로드(Gradle Play Publisher 또는 수동)
```
- 업로드 키스토어·비밀번호·서비스 계정 JSON 은 **저장소에 절대 커밋 금지**, GitHub Secrets 사용.

---

## 10. 2단계 선택 확장 (비용 발생 가능 — 필요 시에만)
| 기능 | 방식 | 비용 |
|---|---|---|
| 서버 영수증 검증, 환불 회수 | Firebase Cloud Functions + Play Developer API | Blaze(종량제) — 소규모 시 무료 한도 내 가능성 높음 |
| Play Integrity 서버 검증 | Functions 에서 토큰 복호화 | Integrity API 일일 무료 할당량 내 |
| 쿠폰 코드 | Firestore | 무료 한도 내 |
