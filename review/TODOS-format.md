# TODOS.md 형식 참조

`/ship` (Step 5.5)과 `/plan-ceo-review` (TODOS.md 업데이트 섹션)에서 참조하는 정규 TODOS.md 형식에 대한 공유 참조 문서입니다. 일관된 TODO 항목 구조를 보장합니다.

---

## 파일 구조

```markdown
# TODOS

## <스킬/컴포넌트>     ← 예: ## Browse, ## Ship, ## Review, ## Infrastructure
<항목은 P0 우선, 이후 P1, P2, P3, P4 순으로 정렬>

## Completed
<완료 주석이 포함된 완료 항목>
```

**섹션:** 스킬 또는 컴포넌트별로 구성합니다 (`## Browse`, `## Ship`, `## Review`, `## QA`, `## Retro`, `## Infrastructure`). 각 섹션 내에서 항목은 우선순위별로 정렬합니다 (P0이 최상위).

---

## TODO 항목 형식

각 항목은 해당 섹션 아래의 H3입니다:

```markdown
### <제목>

**What:** 작업에 대한 한 줄 설명.

**Why:** 해결하는 구체적인 문제 또는 얻을 수 있는 가치.

**Context:** 3개월 후 누군가가 이 항목을 맡았을 때 동기, 현재 상태, 시작 지점을 이해할 수 있을 정도의 충분한 상세 내용.

**Effort:** S / M / L / XL
**Priority:** P0 / P1 / P2 / P3 / P4
**Depends on:** <선행 조건, 또는 "None">
```

**필수 필드:** What, Why, Context, Effort, Priority
**선택 필드:** Depends on, Blocked by

---

## 우선순위 정의

- **P0** — 차단(Blocking): 다음 릴리스 전에 반드시 완료해야 함
- **P1** — 핵심(Critical): 이번 사이클에 완료해야 함
- **P2** — 중요(Important): P0/P1이 해결된 후 수행
- **P3** — 있으면 좋은(Nice-to-have): 도입/사용 데이터 확인 후 재검토
- **P4** — 언젠가(Someday): 좋은 아이디어이나 긴급하지 않음

---

## 완료 항목 형식

항목이 완료되면 원래 내용을 유지한 채 `## Completed` 섹션으로 이동하고 다음을 추가합니다:

```markdown
**Completed:** vX.Y.Z (YYYY-MM-DD)
```
