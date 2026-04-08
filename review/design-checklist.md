# 디자인 리뷰 체크리스트 (간략판)

> **DESIGN_METHODOLOGY의 하위 집합** — 여기에 항목을 추가할 때는 `scripts/gen-skill-docs.ts`의 `generateDesignMethodology()`도 함께 업데이트하세요. 그 반대도 마찬가지입니다.

## 안내

이 체크리스트는 **diff에 포함된 소스 코드**에 적용됩니다 — 렌더링된 출력이 아닙니다. 변경된 각 프론트엔드 파일을 (diff 헝크만이 아닌 전체 파일을) 읽고 안티패턴을 지적하세요.

**트리거:** diff가 프론트엔드 파일을 수정하는 경우에만 이 체크리스트를 실행합니다. `gstack-diff-scope`를 사용하여 감지하세요:

```bash
source <(~/.claude/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)
```

`SCOPE_FRONTEND=false`이면 전체 디자인 리뷰를 조용히 건너뛰세요.

**DESIGN.md 보정:** 저장소 루트에 `DESIGN.md` 또는 `design-system.md`가 존재하면 먼저 읽으세요. 모든 발견 사항은 프로젝트의 명시된 디자인 시스템을 기준으로 보정됩니다. DESIGN.md에서 명시적으로 승인된 패턴은 지적하지 마세요. DESIGN.md가 없으면 범용 디자인 원칙을 적용합니다.

---

## 신뢰도 등급(Confidence Tiers)

각 항목에는 탐지 신뢰도 수준이 태그됩니다:

- **[HIGH]** — grep/패턴 매칭으로 안정적으로 탐지 가능. 확정적인 발견 사항.
- **[MEDIUM]** — 패턴 집계 또는 휴리스틱으로 탐지 가능. 발견 사항으로 지적하되 일부 노이즈가 있을 수 있음.
- **[LOW]** — 시각적 의도 이해 필요. 다음과 같이 제시: "가능성 있는 문제 — 시각적으로 확인하거나 /design-review를 실행하세요."

---

## 분류

**AUTO-FIX** (기계적 CSS 수정만 해당 — HIGH 신뢰도, 디자인 판단 불필요):
- `outline: none`에 대체 없음 → `outline: revert` 또는 `&:focus-visible { outline: 2px solid currentColor; }` 추가
- 새 CSS에 `!important` → 제거하고 특이성(specificity) 수정
- 본문 텍스트 `font-size` < 16px → 16px로 상향

**ASK** (나머지 모든 항목 — 디자인 판단 필요):
- 모든 AI 슬롭(slop) 발견 사항, 타이포그래피 구조, 간격 선택, 인터랙션 상태 공백, DESIGN.md 위반

**LOW 신뢰도 항목** → "가능성: [설명]. 시각적으로 확인하거나 /design-review를 실행하세요."로 제시. 절대 AUTO-FIX하지 마세요.

---

## 출력 형식

```
Design Review: N issues (X auto-fixable, Y need input, Z possible)

**AUTO-FIXED:**
- [file:line] 문제 → 적용된 수정

**NEEDS INPUT:**
- [file:line] 문제 설명
  Recommended fix: 제안된 수정

**POSSIBLE (시각적으로 확인):**
- [file:line] 가능성 있는 문제 — /design-review로 확인
```

선택 사항: `test_stub` — 프로젝트의 테스트 프레임워크를 사용한 이 발견 사항의 스켈레톤 테스트 코드.

문제가 없으면: `Design Review: No issues found.`

프론트엔드 파일 변경이 없으면: 조용히 건너뛰기, 출력 없음.

---

## 카테고리

### 1. AI 슬롭(Slop) 탐지 (6개 항목) — 최우선

존경받는 스튜디오의 디자이너라면 절대 출시하지 않을 AI 생성 UI의 전형적인 징후입니다.

- **[MEDIUM]** 보라색/바이올렛/인디고 그라데이션 배경 또는 파란색-보라색 색상 구성. `#6366f1`–`#8b5cf6` 범위의 값을 가진 `linear-gradient`, 또는 보라색/바이올렛으로 확인되는 CSS 커스텀 속성을 찾으세요.

- **[LOW]** 3열 기능 그리드: 색상 원 안의 아이콘 + 굵은 제목 + 2줄 설명이 3개 대칭으로 반복. 원형 요소 + 제목 + 단락을 각각 포함하는 정확히 3개의 자식을 가진 grid/flex 컨테이너를 찾으세요.

- **[LOW]** 섹션 장식으로 사용되는 색상 원 안의 아이콘. `border-radius: 50%` + 배경색이 있는 요소가 아이콘의 장식 컨테이너로 사용되는지 찾으세요.

- **[HIGH]** 모든 것 가운데 정렬: 모든 제목, 설명, 카드에 `text-align: center`. `text-align: center` 밀도를 grep하세요 — 텍스트 컨테이너의 60% 이상이 가운데 정렬을 사용하면 지적.

- **[MEDIUM]** 모든 요소에 균일한 버블형 border-radius: 카드, 버튼, 입력란, 컨테이너에 동일한 큰 반경(16px+)이 일괄 적용. `border-radius` 값을 집계하세요 — 80% 이상이 동일한 값 ≥16px을 사용하면 지적.

- **[MEDIUM]** 제네릭 히어로 카피: "Welcome to [X]", "Unlock the power of...", "Your all-in-one solution for...", "Revolutionize your...", "Streamline your workflow". HTML/JSX 콘텐츠에서 이 패턴을 grep하세요.

### 2. 타이포그래피(Typography) (4개 항목)

- **[HIGH]** 본문 텍스트 `font-size` < 16px. `body`, `p`, `.text` 또는 기본 스타일에서 `font-size` 선언을 grep. 기준이 16px일 때 16px(또는 1rem) 미만 값은 지적.

- **[HIGH]** diff에 3개 이상의 폰트 패밀리 도입. 고유한 `font-family` 선언 수를 세세요. 변경된 파일에 3개 이상의 고유 패밀리가 나타나면 지적.

- **[HIGH]** 제목 계층 레벨 건너뛰기: 같은 파일/컴포넌트에서 `h2` 없이 `h1` 다음에 `h3` 등장. HTML/JSX의 제목 태그를 확인.

- **[HIGH]** 블랙리스트 폰트: Papyrus, Comic Sans, Lobster, Impact, Jokerman. `font-family`에서 이 이름들을 grep.

### 3. 간격 및 레이아웃(Spacing & Layout) (4개 항목)

- **[MEDIUM]** DESIGN.md가 간격 스케일을 지정할 때, 4px 또는 8px 스케일에 맞지 않는 임의 간격 값. 명시된 스케일에 대해 `margin`, `padding`, `gap` 값을 확인. DESIGN.md가 스케일을 정의하는 경우에만 지적.

- **[MEDIUM]** 반응형 처리 없는 고정 너비: `max-width` 또는 `@media` 브레이크포인트 없이 컨테이너에 `width: NNNpx`. 모바일에서 가로 스크롤 위험.

- **[MEDIUM]** 텍스트 컨테이너에 `max-width` 누락: 본문 텍스트 또는 단락 컨테이너에 `max-width`가 설정되지 않아 한 줄이 75자를 초과할 수 있음. 텍스트 래퍼에 `max-width` 확인.

- **[HIGH]** 새 CSS 규칙에 `!important`. 추가된 줄에서 `!important`를 grep. 거의 항상 적절히 수정해야 할 특이성 탈출구(specificity escape hatch).

### 4. 인터랙션 상태(Interaction States) (3개 항목)

- **[MEDIUM]** hover/focus 상태가 없는 인터랙티브 요소(버튼, 링크, 입력란). 새 인터랙티브 요소 스타일에 `:hover`와 `:focus` 또는 `:focus-visible` 의사 클래스가 있는지 확인.

- **[HIGH]** 대체 포커스 인디케이터 없는 `outline: none` 또는 `outline: 0`. `outline:\s*none` 또는 `outline:\s*0`을 grep. 이는 키보드 접근성을 제거합니다.

- **[LOW]** 인터랙티브 요소의 터치 타겟 < 44px. 버튼과 링크의 `min-height`/`min-width`/`padding`을 확인. 여러 속성에서 유효 크기를 계산해야 함 — 코드만으로는 낮은 신뢰도.

### 5. DESIGN.md 위반 (3개 항목, 조건부)

`DESIGN.md` 또는 `design-system.md`가 존재하는 경우에만 적용:

- **[MEDIUM]** 명시된 팔레트에 없는 색상. 변경된 CSS의 색상 값을 DESIGN.md에 정의된 팔레트와 비교.

- **[MEDIUM]** 명시된 타이포그래피 섹션에 없는 폰트. `font-family` 값을 DESIGN.md의 폰트 목록과 비교.

- **[MEDIUM]** 명시된 스케일 외의 간격 값. `margin`/`padding`/`gap` 값을 DESIGN.md의 간격 스케일과 비교.

---

## 억제(Suppressions)

지적하지 마세요:
- DESIGN.md에 의도적 선택으로 명시적으로 문서화된 패턴
- 서드파티/벤더 CSS 파일 (node_modules, vendor 디렉토리)
- CSS 리셋 또는 normalize 스타일시트
- 테스트 픽스처 파일
- 생성된/미니파이된 CSS
