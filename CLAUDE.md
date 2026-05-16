# Sliding Puzzle (슬라이딩 퍼즐)

서버 없음 · 빌드 없음 · 단일 HTML PWA. **라이트 글라스모피즘** 톤.

## 폴더 구조

```
sliding_puzzle/
  index.html              # 마크업·스타일·로직 전부
  manifest.webmanifest    # PWA 설치 메타
  apple-touch-icon.png    # iOS 홈 화면 아이콘 (180×180)
  CLAUDE.md               # 본 문서 (스펙)
```

## 실행 / 배포

- 로컬: `index.html` 더블클릭 → 브라우저에서 즉시 실행
- 배포: GitHub Pages — https://duswl6868.github.io/sliding-puzzle/
- iOS PWA: Safari → 공유 → **홈 화면에 추가** → 풀스크린 독립 실행
- 업데이트 흐름: `git push` → Pages 자동 빌드 1~2분 후 반영

## 핵심 결정

| 항목 | 결정 |
|---|---|
| 톤 | 라이트 글라스모피즘 (메시 그라디언트 배경 + 화이트 글라스 카드 + 블루 액센트) |
| 그리드 | 일반 모드 홈 카드는 **3×3 / 4×4**. 게임 화면 `N × N ▼` 모달로 **3·4·5 디폴트 + 클리어한 사이즈의 +1 해금** (5 클리어 → 6 열림, 8 클리어 → 9 열림). 무한 모드는 3부터 ∞ 진행 |
| 입력 | 인접 타일 탭 + 화살표 키 + 모바일 스와이프 |
| 셔플 | 완성 상태에서 합법 이동 N회 (항상 solvable). N = `max(70, size² × 10)` |
| 기록 | 시간 + 이동. best는 사이즈별 시간 단축 시 갱신 |
| 저장 | LocalStorage 단일 키 `sliding_puzzle_glass_v1` |
| 사운드 / 진동 | 미적용 |

## 모드

### 일반 모드 (3×3 / 4×4 + 모달로 5+ 도달)
- 홈 카드 클릭 → 3×3 또는 4×4 진입
- 완성 win 화면 → `다시 플레이` / `메인으로` (단일 게임 흐름, 자동 다음 사이즈 진행 없음)
- 헤더의 `N × N ▼` 클릭 → **사이즈 선택 모달** (아래 별도 섹션)
- 모달에서 선택한 사이즈로 즉시 게임 재시작 (`infiniteMode = false` 유지). win 시 동일한 일반 모드 win 화면

### 무한의 퍼즐
- 3×3부터 시작 → 완성 win 화면 → `다음 (N+1 × N+1)` / `다시하기` / `홈` 세로 배치
- 다음 누르면 사이즈 +1 후 새 게임. 끝없이 진행
- 헤더 ▼ 사이즈 토글은 무효 (모드 보호)
- ← 뒤로 / 홈 누르면 무한 모드 종료

### 내 사진 퍼즐
- 홈 카드 → hidden file input (`accept="image/*"`) 트리거 → 파일 선택
- 캔버스로 **가운데 정사각 크롭** (max 1024×1024, JPEG 92%)
- 각 타일은 전체 이미지를 그리드만큼 잘라 보임. `background-position`은 타일 숫자의 **목표 위치** 기준 → 셔플되면 그림이 흩어지고, 풀면 합쳐짐
- 우상단에 작은 흰색 mono 숫자 오버레이 (식별용)
- 사이즈는 마지막 사용한 값 (3 또는 4)
- LS에는 저장하지 않음 (메모리 휘발)

### 사이즈 선택 모달 (`#sizeModal`)
- 진입: 게임 화면(`#playScreen`) 상단 `N × N ▼` 버튼. 일반 모드만 활성, 무한 모드에서는 무효.
- 레이아웃: 글라스 카드 + `repeat(3, 1fr)` 그리드. 각 칸은 정사각 (`aspect-ratio: 1`), `N × N` 텍스트. 현재 사이즈는 블루 활성 톤(`.current`).
- 표시 사이즈 풀 — `computeSizeOptions()`:
  - 디폴트: `3, 4, 5` (베스트 기록이 없어도 항상 노출)
  - 해금: `Object.keys(LS.best)` 중 최대값 N이 있으면 `N+1` 까지 표시. 중간 사이즈는 연속해서 채워짐.
  - 예: `best={3,4,5}` → `[3,4,5,6]` · `best={3,4,5,7}` → `[3,4,5,6,7,8]` · `best={}` → `[3,4,5]`
- 닫기: 빈 백드롭 클릭. 사이즈 칸 클릭 시 `pickSize(n)` → `infiniteMode=false` + `startGame(n)`. 같은 사이즈 클릭 시 단순 재셔플.
- 응답성:
  - **표시 방식**: `display: none` 토글 대신 `visibility + opacity` 패턴. backdrop-filter layer를 미리 유지해 첫 표시 시 paint 비용 회피. opacity 120ms 페이드.
  - **`pickSize` RAF yield**: `sizeModal.classList.remove('active')` 직후 `requestAnimationFrame`으로 `startGame`/`shuffle`을 한 프레임 미룸 → 모달 닫기 paint가 무거운 빌드+셔플로 막히지 않음.

## 화면 구조

3개의 풀스크린 `.screen`, `.active` 클래스로 전환.

### Home (`#homeScreen`)
- 상단 topbar: ≡ 설정 (현재 alert stub) / 우측 비어있음
- 히어로: 큰 타이틀 "슬라이딩 퍼즐" + 부제 "집중하고, 정렬하고, 완성하세요."
- 난이도 카드 4장 (글라스 카드):
  1. **3 × 3** — 8 퍼즐, mini-grid 시각화
  2. **4 × 4** — 15 퍼즐, mini-grid 시각화
  3. **무한의 퍼즐** — 보라 그라디언트 ∞ 썸네일
  4. **내 사진 퍼즐** — 블루 그라디언트 카메라 썸네일 (이미지 선택 후 썸네일로 교체)
- 하단 home-stats 글라스 카드 (3컬럼): `오늘 플레이` / `최고 기록` / `완료`

### Play (`#playScreen`)
- 상단 topbar: ← 뒤로 / `N × N ▼` (일반 모드에선 사이즈 선택 모달 / 무한 모드에선 무효) / 디버그 + ↻ 다시하기
- stats-row: 시간 (좌) / 이동 (우) — 라벨 12px + 값 17px 굵게
- 보드 (글라스 카드, slate 빈칸 비침)
- 하단 플로팅 toolbar (글라스 카드, position: fixed): `↶ 되돌리기` / `💡 힌트 (badge)` / `↻ 다시하기` (셔플 아이콘 유지 + 라벨만 "다시하기" — Win 화면 "다시 플레이"와 톤 통일, 사용자 의도와 직접 매칭)

### Win (`#winScreen`)
- 컨페티 28개 (1.8s fall: translateY + rotate 230deg)
- "완성!" + 시간 + 이동 표시
- 솔브드 보드 미리보기 (`#winBoard`)
- 액션 영역 (모드별):
  - 일반: `다시 플레이` (primary) / `메인으로` (링크)
  - 무한: `다음 (N+1 × N+1)` (primary) / `다시하기` (secondary 글라스) / `홈` (링크)

## 디자인 토큰 (라이트 글라스)

`:root` 핵심 변수:

```
--bg0:          #f7f9fd
--bg1:          #eef3fa
--ink:          #1f314d
--ink-soft:     #40516b
--muted:        #7b899c
--faint:        #a3afbf
--line:         rgba(170,184,205,.28)
--glass:        rgba(255,255,255,.58)
--glass-strong: rgba(255,255,255,.74)
--glass-soft:   rgba(255,255,255,.38)
--empty:        rgba(211,224,240,.58)
--blue:         #1877ff
--blue2:        #4ba0ff
--blue-soft:    rgba(24,119,255,.16)
--blue-mid:     rgba(24,119,255,.34)
--ease:         cubic-bezier(.22,.9,.2,1)
--spring:       cubic-bezier(.32,.72,.24,1)
--slide-ms:     190ms
--font:         Pretendard Variable, Pretendard, -apple-system, ... system-ui
```

페이지 배경: 다중 radial highlight + 145° linear gradient(`#f8fbff → #eef3fa → #f7faff`). 34px 그리드 텍스처(`body::before`, opacity 0.42, 중앙 radial mask).

`.glass-card` 공통:
```
background: linear-gradient(145deg, rgba(255,255,255,.76), rgba(255,255,255,.42));
border: 1px solid rgba(255,255,255,.72);
box-shadow: inset 0 1px 0 rgba(255,255,255,.92), inset 0 -1px 0 rgba(195,208,226,.18), 0 16px 36px rgba(35,55,90,.09);
backdrop-filter: blur(22px) saturate(150%);
```

`.tile` 공통:
```
border-radius: 10px;
background: linear-gradient(145deg, rgba(255,255,255,.70), rgba(255,255,255,.46));
border: 1px solid rgba(255,255,255,.78);
box-shadow: inset 0 1px 0 rgba(255,255,255,.95), inset 0 -1px 0 rgba(164,184,209,.18), 0 2px 5px rgba(35,55,86,.12);
backdrop-filter: blur(18px) saturate(150%);
```

타이포: 시스템 + Pretendard 우선. 숫자는 `font-variant-numeric: tabular-nums`, weight 780~850.

**타일 폰트/라운드/간격 + 보드 라운드 — size별 단계 적용** (`sizeFonts` + `gapFor`):

| size | ratio | min px | weight | tile r | board r | gap | 메모 |
|---|---|---|---|---|---|---|---|
| 3 ~ 6 | 0.36 | 24 | 780 | 10px | 14px | 4px | 기본 — 한·두 자릿수, 굵게 |
| 7 ~ 8 | 0.30 | 17 | 720 | 8px | 12px | 4px | 두 자릿수 본격 — 호흡 확보 |
| 9 ~ 11 | 0.26 | 14 | 720 | 6px | 8px | 3px | 세 자릿수 가능 (10×10 = 99), 라운드·간격 ↓ |
| 12 ~ 14 | 0.22 | 12 | 680 | 5px | 6px | 2px | 세 자릿수 빈도 ↑ (12×12 = 143) — 추가 축소 + 얇게 |
| 15 ~ 16 | 0.18 | 10 | 640 | 5px | 6px | 2px | 폰트 추가 축소 (15×15 = 224, 자릿수 많음) |
| 17 | 0.15 | 9 | 620 | 5px | 6px | 2px | 폰트 더 축소 |
| 18 | 0.14 | 9 | 600 | 4px | 5px | 2px | 라운드도 점진 축소 시작 |
| 19 | 0.13 | 8 | 580 | 3px | 4px | 2px | 라운드 더 축소 |
| 20 이상 | 0.12 | 8 | 560 | 2px | 3px | 2px | 사실상 직사각형 톤 — 매우 큰 그리드 가독성 우선 |

- 폰트 사이즈 = `Math.max(minPx, tileSize × ratio)`. 큰 그리드는 ratio 작아져 자릿수가 많아도 빽빽하지 않게 표현. 7부터 weight 720, 12부터 680, 15부터 640, 17부터 620, 18~20+ 600/580/560으로 점진 축소.
- 타일 라운드는 타일이 작아질수록 비율상 동글동글해 보이는 현상을 방지하려고 단계적으로 줄임. CSS의 `.tile { border-radius: 10px }`는 기본값(3~6 단계 기준), 7 이상은 JS에서 인라인으로 덮어씀.
- 보드 라운드는 타일 라운드보다 살짝 더 큰 톤으로 단계화 — 사용자 결정. CSS의 `.board { border-radius: 14px }`는 기본값(3~6 단계), 7 이상은 JS에서 인라인으로 덮어씀. `.board::after`는 `border-radius: inherit`이라 자동 반영.
- gap은 `gapFor()`로 size에 따라 동적으로 계산. `tileSize`/`tilePos`/`applyImageBg` 함수 내에서 `const GAP = gapFor()`로 사용. size 변경 시 자동 반영 (별도 갱신 호출 불필요).

## 모션

| 인터랙션 | 모션 |
|---|---|
| 타일 슬라이드 | `transform: translate()` 190ms `--spring` |
| 타일 프레스 | `scale(0.965)`, 80ms |
| 타일 spawn (셔플 직후) | 460ms `--ease`. opacity 0→1 + scale .86→1.035→1.0 + blur 2px→0. staggered 10ms (max 100ms) — 큰 사이즈 시작 응답성 우선 |
| 다시하기 버튼 press | `soft-pop` 360ms (scale .965 → 1.018 → 1.0) |
| Win 컨페티 | 28개 `<i>`, 1.8s fall |
| Difficulty card press | `scale(0.985)`, 130ms |
| `prefers-reduced-motion` | 모든 전환 0ms, 컨페티·spawn·pop 무효화 |

## 게임 로직

### 데이터
- `tiles: number[]` 길이 size², 각 칸 값 (0 = 빈칸)
- `order: Map<value, {row, col}>` 값별 현재 위치
- `emptyPos` 빈칸 1차원 인덱스
- `history: { tileTo, emptyTo }[]` 되돌리기 스택
- `hints: number` 남은 힌트 (라운드 시작 시 3)
- `imageMode: boolean`, `currentImage: dataUrl | null`
- `infiniteMode: boolean`
- `accumulatedMs: number` 라운드 시작부터 누적된 진행 시간
- `lastResumeTs: number | null` 마지막 타이머 재개 시점 (`null`이면 일시정지·미시작 상태)

### 좌표
- `GAP = gapFor()` — size 단계별: 3~8 = 4px, 9~11 = 3px, 12 이상 = 2px (큰 그리드 빽빽함 완화)
- `tileSize(board) = (boardPx - GAP × (size+1)) / size`
- `tilePos(row, col) = { x: GAP + col×(ts+GAP), y: GAP + row×(ts+GAP), ts }`

### 입력
- 탭: `tryTapTile(value)` → 인접/합법 검사 → `doMove(pos, true)`
- 화살표: `tryMoveByDirection(dir)`
  - ↑ : 빈칸의 아래쪽 타일이 위로 (= 빈칸이 아래로 이동)
  - ↓ ← → 동일
- 스와이프: `pointerdown`/`pointerup` 좌표 차, threshold 24px, 큰 축으로 방향 결정
- ⌘/Ctrl+Z: 되돌리기

### 되돌리기
- 직전 이동만 한 칸 reverse, moves-- (음수 방지)
- won 또는 history 비어있을 때 비활성

### 힌트
- 인접 타일들 중 **이동 후 맨해튼 거리 합이 최소**인 타일에 `hint` 클래스 (블루 강조)
- 카운트 -1. 다음 이동 시 자동 클리어
- 라운드 시작 시 3으로 reset
- **이미 hint 표시 중이면 재클릭 무효** (early return) — 같은 보드 상태에서 동일한 후보를 반복 노출하는데 카운트가 차감되는 손해 방지. 한 번 이동해 hint가 자동 클리어된 뒤 다시 누르면 정상 차감

### 타이머 / 일시정지

- 단순 `startTs` 단일 변수 모델 폐기. **`accumulatedMs` + `lastResumeTs` 2개 변수**로 pause/resume 지원
- `elapsedMs() = accumulatedMs + (lastResumeTs === null ? 0 : performance.now() - lastResumeTs)`
- `startTimer()` — 첫 이동 시 호출 (가드: `lastResumeTs === null && accumulatedMs === 0`). `accumulatedMs = 0`, `lastResumeTs = now`, `setInterval(tickTimer, 200)`
- `stopTimer()` — won/홈 복귀 등 라운드 종료. 누적 확정 (`accumulatedMs += now - lastResumeTs`) + `lastResumeTs = null` + interval 정지
- `pauseTimer()` — 가시성 hidden 시. 누적만 갱신, interval은 그대로 (브라우저 throttle)
- `resumeTimer()` — 가시성 visible 복귀. 가드: `lastResumeTs === null && !won && playScreen.active && (accumulatedMs > 0 || timerHandle !== null)`. `lastResumeTs = now`
- `document.addEventListener('visibilitychange', ...)` — 화면 잠금 / 앱 스위치 / 다른 탭 이동 모두 동일 처리
- **효과**: 백그라운드에서 시간이 흐르지 않아 베스트 기록이 부풀어 자동 갱신되는 문제 방지 (특히 무한 모드)

### 셔플
- 완성 상태에서 무작위 합법 이동 `max(70, size² × 10)`회
- 직전 위치로 되돌리는 이동 제외
- 종료 후 우연히 solved면 1회 추가
- `spawnTiles()` 호출 → 타일 톡톡 페이드인

### 승리 판정 / onSolved
1. `tiles[i] === i+1` (i = 0..N-2)
2. won=true, 입력 락, 타이머 stop
3. `setBestIfBetter(elapsedMs, moves)` (사이즈별 시간 단축 시 갱신, completed +1)
4. 모든 타일에 `complete` 클래스
5. `configureWinActions()`로 모드별 액션 버튼 텍스트/표시 조정
6. 300ms 후 winBoardEl 빌드 + `layoutSolved` + 컨페티 + `show(winScreen)`
7. winScreen 활성화 시 RAF 안에서 `layoutSolved(winBoardEl)` 다시 호출 (display:none 상태에서 첫 호출이 noop이라 보정 필요)

## 데이터 모델 (LocalStorage)

키: `sliding_puzzle_glass_v1`

```ts
{
  v: 1,
  lastSize: 3 | 4 | 5 | 6 | ...,    // 무한 모드에서 5+ 가능
  best: {
    "3": { timeMs, moves },
    "4": { ... },
    "5": { ... },
    // 사이즈별 자동 생성
  },
  today: { date: "YYYY-MM-DD", plays },   // 날짜 바뀌면 자동 초기화
  completed: number                        // 누적 완료 횟수
}
```

- 파싱 실패 / 버전 불일치 시 `defaultStore()`로 폴백
- 사진 모드는 LS에 안 저장 (메모리 휘발)
- `mutateLS(fn)`으로 일관된 read-modify-write 후 `renderHomeStats()` 갱신

## 의도적 제외 (v1)

- 사운드 / 진동
- 다크 테마 토글
- 5×5 이상 **홈 카드** (5+는 게임 화면 사이즈 선택 모달 또는 무한 모드로 도달)
- 그라디언트 / 패턴 모드 (이전 다크 버전 기능, 제거)
- 클라우드 동기화
- 행/열 동시 슬라이드 (한 번에 한 타일)
- 이미지 크롭 UI (자동 가운데 크롭만)

## 검증 체크리스트 (커밋 전)

- [ ] `index.html` 더블클릭 시 즉시 실행
- [ ] Home → 3×3 / 4×4 / 무한 / 사진 4가지 진입 OK
- [ ] 사진 모드: 가로/세로 사진 모두 가운데 정사각 크롭으로 들어옴
- [ ] 무한 모드: 3 → 4 → 5 → … 다음 버튼으로 진행
- [ ] 탭 / 화살표 / 스와이프 동작
- [ ] 되돌리기: 직전 한 칸 정확히 reverse, 비활성 처리
- [ ] 힌트: 3회 후 비활성, 다음 이동 시 클리어
- [ ] 타일 spawn 애니메이션 셔플 시 매번 발현
- [ ] 다시하기 버튼 누를 때 pop 효과
- [ ] best 갱신 시 LS 업데이트 + 홈 통계 반영
- [ ] LS 강제 변조해도 크래시 없이 기본값 복원
- [ ] `prefers-reduced-motion` 시 애니메이션 정지
