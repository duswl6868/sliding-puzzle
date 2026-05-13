# Sliding Puzzle (슬라이딩 퍼즐)

서버 없음 · 빌드 없음 · 단일 HTML PWA. 다크 무드 / 미니멀 / 감각적.

## 폴더 구조

```
sliding_puzzle/
  index.html              # 마크업·스타일·로직 전부
  manifest.webmanifest    # PWA 설치 메타
  CLAUDE.md               # 본 문서 (스펙)
```

연관 디자인 파일:
- `../pencil/sliding_puzzle.pen` — 디자인 마스터 (Pencil)

## 실행

1. `index.html` 더블클릭 → 브라우저에서 즉시 실행
2. GitHub Pages 등 정적 호스팅에 폴더 통째로 업로드 → PWA 설치 가능

## 핵심 결정

| 항목 | 결정 |
|---|---|
| 그리드 | 3×3 / 4×4 / 5×5 선택 |
| 모드 | 숫자 / 그라디언트 / 패턴 |
| 입력 | 인접 타일 탭 + 화살표 키 + 모바일 스와이프 |
| 셔플 | 완성 상태에서 합법 이동 N회 (항상 solvable) |
| 기록 | 시간 + 이동 횟수, best는 (난이도×모드)별 시간 기준 |
| 저장 | LocalStorage 단일 키 `sliding_puzzle_v1` |
| 사운드/진동 | v1 제외 (의도적 침묵) |

## 화면 구성

단일 화면, 세로 정렬. **앱 컨테이너 `max-width: 390px`** 안에 모든 콘텐츠가 들어감 (페이지 배경 `#07080A` 위 `--bg` 컨테이너, 내부 `inset 1px rgba(255,255,255,0.06)` 보더, `border-radius: 32`).

1. **상단 컨트롤 바** — 좌측 난이도 칩(3·4·5), 우측 모드 토글(숫자·그라디언트·패턴)
2. **라이브 스탯** — `time` `moves` 두 칼럼, mono 폰트 24px tabular-nums, gap 48
3. **보드** — 320px 정사각형 (모바일에서는 컨테이너 폭 내 100%)
4. **새 게임 버튼** — 단일 액션
5. **best 라벨** — 현재 (난이도, 모드) 조합의 최고 기록, 갱신 시 `new!` 펄스

화면 전환 없음.

## 디자인 토큰 (다크 고정)

```
--bg:         #0E0F12
--surface:    #16181D
--tile:       #1E2128
--tile-edge:  rgba(255,255,255,0.06)
--text:       #E8EAED
--text-dim:   rgba(232,234,237,0.45)
--text-faint: rgba(232,234,237,0.25)
--accent:     #9EB8FF
--success:    #7CE0B3
--tile-radius:  14px
--board-radius: 16px
--ease-spring: cubic-bezier(0.32, 0.72, 0.24, 1)
--slide-ms:   180ms
```

타이포:
- 숫자/시간: `'SF Mono', 'JetBrains Mono', ui-monospace` + `font-variant-numeric: tabular-nums`
- UI 라벨: 시스템 산세리프

타일 폰트 크기: `tileSize × 0.32` (최소 14px)

## 모션

| 인터랙션 | 모션 |
|---|---|
| 타일 슬라이드 | `transform: translate()` 180ms `--ease-spring` |
| 타일 프레스 | `scale(0.97)`, 80ms ease-out |
| 셔플 | `size² × 8` (최소 60)회 합법 이동 — DOM 일괄 적용 후 단발 리렌더 |
| 완성 보드 | `--accent` 외곽 글로우 + `scale(1.02)` 호흡 (500ms) → 1.0 |
| 완성 타일 | 내부 `--accent` 1px 테두리 페이드인 (300ms) |
| 팬텀 타일 (그라디언트/패턴만) | 빈 칸 위치에 마지막 슬라이스 페이드인 (500ms, 100ms delay) → 그림 완성 |
| best 갱신 시 | 라이브 시간 텍스트가 `--success`로 펄스 |

`prefers-reduced-motion: reduce` 시 모든 트랜지션 0ms.

## 게임 로직

### 데이터 표현
- `tiles: number[]` — 길이 `size²`, 각 칸의 현재 값(0은 빈 칸)
- `order: Map<value, {row, col}>` — 값별 현재 위치 (DOM 적용용)
- `emptyPos: number` — 빈 칸의 1차원 인덱스

### 이동
- 입력 → `tryMoveByDirection(dir)` 또는 `tryTapTile(value)` → 인접/합법성 검사 → `doMove(pos)`
- `doMove`: `applyMove` (상태 갱신) → `moves++` → `layout(true)` → `isSolved()` 시 `onSolved`

### 화살표 키 의미
- ↑ : 빈 칸의 **아래쪽** 타일이 위로 올라옴 (= 빈 칸이 아래로 이동)
- ↓ ← → 동일 규칙

### 스와이프
- `pointerdown`/`pointerup` 좌표 차이, threshold 24px
- 가장 큰 축으로 방향 결정 후 `tryMoveByDirection`

### 셔플 (solvability 보장)
- 완성 상태에서 시작 → `size² × 8` (최소 60)회 무작위 **합법** 이동
- 직전 위치로 되돌리는 이동 제외 (의미 없는 진동 방지)
- 종료 후 우연히 solved면 1회 추가 이동

### 승리 판정
- `tiles[i] === i + 1` (i = 0..N-2) 이면 solved

## 데이터 모델 (LocalStorage)

키: `sliding_puzzle_v1`

```ts
{
  v: 1,
  best: {
    "3:numbers":  { timeMs: number, moves: number },
    "3:gradient": { ... },
    "3:pattern":  { ... },
    "4:numbers":  { ... },
    "4:gradient": { ... },
    "4:pattern":  { ... },
    "5:numbers":  { ... },
    "5:gradient": { ... },
    "5:pattern":  { ... }
  },
  lastDifficulty: 3 | 4 | 5,
  lastMode: "numbers" | "gradient" | "pattern"
}
```

- best 갱신 기준: **시간(ms) 더 짧을 때만**. 이동 횟수는 부수 기록.
- LS 파싱 실패 시 기본값으로 안전 폴백.

## 그라디언트 / 패턴 모드

- 보드 진입 시 프리셋 풀에서 1개 랜덤 선택 (`currentArt`)
- 숫자는 우상단 mono 작은 글씨로 옅게 표기 (값 식별 가능)

### 그라디언트 모드 — 단일 색 모자이크
- 각 타일은 단일 솔리드 색으로 채워짐 (그라디언트 슬라이스가 아님)
- 색은 `from → to` 두 색의 보간값. 보간 비율 `t = (targetCol + targetRow) / (2 × (size-1))`
- 즉 좌상단 = `from`, 우하단 = `to`, 대각선 진행
- 완성 시 대각선으로 부드럽게 색이 변하는 모자이크가 드러남
- 프리셋 4종 내장 (Indigo→Pink, Cyan→Violet, Amber→Red, Emerald→Blue)

### 패턴 모드 — 연속 패턴 슬라이스
- 큰 SVG 패턴(보드 전체 크기)을 타일의 **목표 위치 기준**으로 잘라 보임
- 셔플되면 패턴이 흩어지고, 완성 시 자연스럽게 하나의 패턴으로 합쳐짐
- 패턴 프리셋(SVG 데이터 URL) 3종 내장 (사선 그리드, 동심원, 도트)

## 의도적 제외 (YAGNI, v1)

- 사운드, 진동
- 힌트, undo, redo
- 일별/누적 통계
- 이미지 업로드 모드
- 다중 사용자/클라우드 동기화
- 행/열 동시 슬라이드(여러 타일 한 번에 이동)
- 라이트 테마

## 검증 체크리스트

커밋/배포 전:
- [ ] `index.html` 더블클릭 시 즉시 실행
- [ ] 3×3 / 4×4 / 5×5 모두 셔플 후 풀이 가능
- [ ] 숫자 / 그라디언트 / 패턴 모두 정상 렌더 (그라디언트는 완성 시 그림이 합쳐짐)
- [ ] 탭 / 화살표키 / 스와이프 모두 동작
- [ ] best 갱신 시 `new!` 표시 및 라이브 시간 success 컬러
- [ ] 새로고침 후 마지막 (난이도, 모드) 및 best 복원
- [ ] LS 오염 시(키 강제 변조)에도 크래시 없이 기본값 동작
- [ ] `prefers-reduced-motion` 환경에서 트랜지션 없음
