<!--
SPDX-FileCopyrightText: 2024-2026 Pagefault Games

SPDX-License-Identifier: CC-BY-NC-SA-4.0
-->

# PokéRogue 코드베이스 파악 & 기여 가이드 (한국어)

## 프로젝트 개요

- **기술 스택**: TypeScript + Phaser 3 (캔버스 기반 게임 프레임워크)
- **패키지 매니저**: pnpm 10.x (npm/yarn 사용 금지)
- **Node**: >=24.9.0
- **테스트**: Vitest (Jest 스타일 assertion만 사용)
- **린터/포맷터**: Biome (ESLint + Prettier 통합)

### src/ 주요 구조

```
src/
├── data/           # 포켓몬, 기술, 특성 데이터
├── phases/         # 전투 흐름의 Phase 시스템 (핵심!)
├── ui/             # UI 컴포넌트
├── field/          # 필드/배틀 로직
├── battle-scene.ts # 메인 게임 씬
└── overrides.ts    # 테스트용 오버라이드 설정
```

---

## Phase 시스템

전투 흐름의 핵심 추상화. 모든 게임 로직은 Phase 단위로 순차 실행된다.

### 주의사항

- `start()`에서 반드시 `this.end()` 호출 → 안 하면 게임이 멈춤
- `end()`를 두 번 호출하면 충돌 (게임 크래시)
- Phase 오버라이드 시 `super.end()` 반드시 호출

### instanceof 대신 phaseName 문자열 비교를 쓰는 이유

**순환 임포트(Circular Import) 방지** 때문이다.

`instanceof`를 쓰려면 클래스를 임포트해야 하는데, Phase 파일들이 서로를 참조하면
A → B → A 순환이 생긴다. ES Module은 평가 순서가 중요해서 순환 시 한쪽이
`undefined`가 되어 런타임 오류가 발생한다.

```typescript
// src/phase.ts:43
public is<K extends keyof PhaseMap>(phaseName: K): this is PhaseMap[K] {
    return this.phaseName === phaseName;  // 임포트 없이 타입 좁히기 가능
}

// 사용
if (phase.is("MovePhase")) { ... }
```

> **Java/C#과의 차이**: JVM/CLR은 클래스 로딩 순서를 보장하므로 이 문제가 없다.
> TypeScript/JS 특유의 해결책이며 "좋아서" 쓰는 게 아니라 실용적인 타협이다.

`is()`는 서브클래스를 체크하지 않음 — 정확한 타입만 확인한다.

### PhaseTree 실행 구조

```
Level 0 (push queue): [MovePhase(A), MovePhase(B)]

MovePhase(A) 실행 → unshiftPhase 호출
  Level 1: [MoveEffectPhase(A), MoveEndPhase(A)]  ← 먼저 실행
  Level 0: [MovePhase(B)]
```

항상 가장 높은 레벨부터 실행 → 자식 Phase가 완전히 소진된 후 부모 레벨로 복귀.

---

## Biome — 린터/포맷터

ESLint + Prettier를 **하나로 합친** Rust 기반 도구.

| 항목 | ESLint + Prettier | Biome |
|------|-------------------|-------|
| 역할 | 린팅 + 포맷팅 (별도) | 린팅 + 포맷팅 (단일) |
| 언어 | JavaScript | Rust (10~100배 빠름) |
| 설정 파일 | `.eslintrc` + `.prettierrc` | `biome.jsonc` 하나 |

### 동작 방식

- `pnpm install` 시 **Lefthook** pre-commit 훅 자동 설치
- 커밋 전 자동으로 `biome check --write --staged` 실행
- **error 수준 lint 오류 → 커밋 자체가 막힘**
- `warn`/`info` 규칙에 걸리는 새 코드도 작성 금지

---

## 테스트 전략

### 테스트 피라미드

```
      /E2E\        ← 적게 (느리고 불안정)
     /------\
    / 통합테스트 \   ← 중간
   /------------\
  /  단위 테스트  \  ← 많이 (빠르고 안정적)
```

### 각 테스트를 선택하는 기준

| 종류 | 언제 쓰나 |
|------|----------|
| 단위 테스트 | 외부 의존 없는 순수 로직 (계산 함수, 파싱) |
| 통합 테스트 | 여러 모듈이 함께 동작하는 시나리오 |
| 컴포넌트 테스트 | React/Vue 같은 컴포넌트 기반 프레임워크 (Phaser엔 부적합) |
| E2E 테스트 | 유저 시나리오 전체, 크리티컬 플로우만 소수 |

### 포케로그의 테스트 현황

| 종류 | 상태 | 이유 |
|------|------|------|
| 단위 테스트 | 활성 | 계산 로직, 데이터 검증 |
| 통합 테스트 | **주력** | Phase 전체 흐름 (Phaser를 headless로 실제 구동) |
| 컴포넌트 테스트 | 대부분 skip | 유지보수 비용 > 가치 |
| E2E 테스트 | 없음 | Canvas 기반이라 DOM 쿼리 불가 |

### Canvas 기반 UI 테스트가 어려운 이유

```
DOM:    코드 → [HTML 트리] → 픽셀   ← 테스트 도구가 트리를 읽음
Canvas: 코드 → [픽셀만]             ← 중간 표현이 없음
```

Canvas는 **fire-and-forget 모델** — `drawRect()`를 호출하면 픽셀만 칠해지고
"여기에 버튼이 있다"는 정보가 어디에도 남지 않는다.

| 검증 대상 | 가능 여부 |
|----------|----------|
| 내부 상태값 (`visible`, `alpha`, `text`) | 가능 |
| 렌더링을 유발하는 로직 | 가능 (분리 후 단위 테스트) |
| 실제 렌더링 결과 (색, 위치, 모양) | 사실상 불가 |
| 애니메이션 | 매우 어려움 |

전략: **"UI가 어떻게 보이는가" → 수동 테스트 / "UI 뒤의 로직이 올바른가" → 자동 테스트**

### 테스트 작성 규칙

- `pnpm test:create` → 새 테스트 파일 생성 도우미
- `pnpm test:silent` → 테스트 실행
- **Jest 스타일만** (`toBe()`, `toEqual()`) — Chai 스타일 금지
- 결정론적이어야 함 (랜덤 요소 제거)
- 하나의 테스트 케이스에 하나의 검증

---

## PR 제출 규칙

### 브랜치 전략

| 브랜치 | 역할 | PR 대상 |
|--------|------|----------|
| `beta` | 개발 브랜치 (최신) | **모든 기여자 PR** |
| `main` | 안정 릴리즈 | 개발팀만 (hotfix/release) |

→ **반드시 `beta`를 base로 PR 제출**

### PR 제목 형식 — Conventional Commits (자동 검사됨)

```
fix(move): Future Sight no longer crashes
^   ^      ^
|   |      |__ Subject
|   |_________ Scope (optional)
|_____________ Prefix
```

**유효 Prefix**: `fix`, `feat`, `refactor`, `test`, `docs`, `balance`, `chore`, `perf`, `i18n`, `misc`, `dev`, `github`

**유효 Scope**: `move`, `ability`, `battle`, `ui`, `item`, `encounter`, `biomes`, `challenge`, `audio`, `graphics`, `ai`, `event`

**브레이킹 체인지**: 세이브 마이그레이터 추가, 버전 번호 증가 등 기존 세이브와
호환이 깨지는 변경 → 제목에 `!` 추가

```
feat!: Major Update 1.12
```

### 에셋/로컬라이즈 변경 시 PR 순서

```
1. pokerogue-assets (또는 pokerogue-locales) 레포에 PR 제출 & 머지
        ↓
2. pokerogue 메인 레포에서 서브모듈 업데이트 커밋
        ↓
3. pokerogue 메인 레포에 PR 제출 (관련 PR 링크 포함)
```

---

## 개발 팁

### 수동 테스트용 오버라이드

```typescript
// src/overrides.ts — PR 제출 전 반드시 되돌릴 것
const overrides = {
  ABILITY_OVERRIDE: AbilityId.DROUGHT,
  ENEMY_MOVESET_OVERRIDE: MoveId.WATER_GUN,
} satisfies Partial<InstanceType<typeof DefaultOverrides>>;
```

### 개발 세이브 파일

`test/utils/saves/everything.prsv` — 모든 요소 잠금 해제

게임 내: `Menu → Manage Data → Import Data`

### 유용한 명령어

```bash
pnpm start:dev       # 개발 서버 (localhost:8000)
pnpm test:silent     # 테스트 실행
pnpm test:create     # 새 테스트 파일 생성
pnpm biome           # 변경된 파일 린트/포맷
pnpm typecheck       # 타입 검사
```

---

## 참고 링크

- [CONTRIBUTING.md](https://github.com/pagefaultgames/pokerogue/blob/beta/CONTRIBUTING.md)
- [API 문서](https://pagefaultgames.github.io/pokerogue/beta/index.html)
- Discord: `#pokerogue-dev` 채널
- [미구현 기술/특성 목록 (Issue #3503)](https://github.com/pagefaultgames/pokerogue/issues/3503)
- [버그 보드](https://github.com/orgs/pagefaultgames/projects/3)
