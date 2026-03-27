# Greptile 댓글 분류(Triage)

GitHub PR에서 Greptile 리뷰 댓글을 가져오고, 필터링하고, 분류하기 위한 공유 참조 문서입니다. `/review` (Step 2.5)와 `/ship` (Step 3.75) 모두 이 문서를 참조합니다.

---

## 가져오기(Fetch)

다음 명령을 실행하여 PR을 감지하고 댓글을 가져옵니다. 두 API 호출은 병렬로 실행됩니다.

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner' 2>/dev/null)
PR_NUMBER=$(gh pr view --json number --jq '.number' 2>/dev/null)
```

**둘 중 하나라도 실패하거나 비어 있으면:** Greptile 분류를 조용히 건너뜁니다. 이 통합은 부가적(additive)입니다 — 워크플로우는 이것 없이도 작동합니다.

```bash
# 줄 수준 리뷰 댓글과 최상위 PR 댓글을 병렬로 가져오기
gh api repos/$REPO/pulls/$PR_NUMBER/comments \
  --jq '.[] | select(.user.login == "greptile-apps[bot]") | select(.position != null) | {id: .id, path: .path, line: .line, body: .body, html_url: .html_url, source: "line-level"}' > /tmp/greptile_line.json &
gh api repos/$REPO/issues/$PR_NUMBER/comments \
  --jq '.[] | select(.user.login == "greptile-apps[bot]") | {id: .id, body: .body, html_url: .html_url, source: "top-level"}' > /tmp/greptile_top.json &
wait
```

**API 오류 또는 양쪽 엔드포인트에서 Greptile 댓글이 0개인 경우:** 조용히 건너뜁니다.

줄 수준 댓글의 `position != null` 필터는 force-push된 코드의 오래된 댓글을 자동으로 건너뜁니다.

---

## 억제 확인(Suppressions Check)

프로젝트별 이력 경로를 도출합니다:
```bash
REMOTE_SLUG=$(browse/bin/remote-slug 2>/dev/null || ~/.claude/skills/gstack/browse/bin/remote-slug 2>/dev/null || basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")
PROJECT_HISTORY="$HOME/.gstack/projects/$REMOTE_SLUG/greptile-history.md"
```

`$PROJECT_HISTORY`가 존재하면 읽습니다(프로젝트별 억제). 각 줄은 이전 분류 결과를 기록합니다:

```
<date> | <repo> | <type:fp|fix|already-fixed> | <file-pattern> | <category>
```

**카테고리** (고정 집합): `race-condition`, `null-check`, `error-handling`, `style`, `type-safety`, `security`, `performance`, `correctness`, `other`

가져온 각 댓글을 다음 조건의 항목과 매칭합니다:
- `type == fp` (알려진 오탐(false positive)만 억제, 이전에 수정된 실제 문제는 아님)
- `repo`가 현재 저장소와 일치
- `file-pattern`이 댓글의 파일 경로와 일치
- `category`가 댓글의 이슈 유형과 일치

매칭된 댓글은 **SUPPRESSED**로 건너뜁니다.

이력 파일이 존재하지 않거나 파싱할 수 없는 줄이 있으면 해당 줄을 건너뛰고 계속합니다 — 잘못된 이력 파일로 인해 절대 실패하지 마세요.

---

## 분류(Classify)

억제되지 않은 각 댓글에 대해:

1. **줄 수준 댓글:** 표시된 `path:line`의 파일과 주변 컨텍스트(±10줄)를 읽습니다
2. **최상위 댓글:** 전체 댓글 본문을 읽습니다
3. 댓글을 전체 diff (`git diff origin/main`)와 리뷰 체크리스트에 대해 교차 참조합니다
4. 분류:
   - **VALID & ACTIONABLE** — 현재 코드에 존재하는 실제 버그, 경쟁 조건, 보안 문제 또는 정확성 문제
   - **VALID BUT ALREADY FIXED** — 브랜치의 후속 커밋에서 해결된 실제 문제. 수정 커밋 SHA를 식별합니다.
   - **FALSE POSITIVE** — 댓글이 코드를 잘못 이해하거나, 다른 곳에서 처리된 것을 지적하거나, 스타일 노이즈
   - **SUPPRESSED** — 위의 억제 확인에서 이미 필터링됨

---

## 응답 API

Greptile 댓글에 응답할 때 댓글 소스에 따라 올바른 엔드포인트를 사용합니다:

**줄 수준 댓글** (`pulls/$PR/comments`에서 가져온 것):
```bash
gh api repos/$REPO/pulls/$PR_NUMBER/comments/$COMMENT_ID/replies \
  -f body="<reply text>"
```

**최상위 댓글** (`issues/$PR/comments`에서 가져온 것):
```bash
gh api repos/$REPO/issues/$PR_NUMBER/comments \
  -f body="<reply text>"
```

**응답 POST가 실패하는 경우** (예: PR이 닫혔거나, 쓰기 권한 없음): 경고하고 계속합니다. 응답 실패로 워크플로우를 중단하지 마세요.

---

## 응답 템플릿

모든 Greptile 응답에 이 템플릿을 사용하세요. 항상 구체적인 근거를 포함하세요 — 모호한 응답을 올리지 마세요.

### Tier 1 (첫 번째 응답) — 친절하고, 근거 포함

**수정(FIXES)의 경우 (사용자가 문제를 수정하기로 선택):**

```
**Fixed** in `<commit-sha>`.

\`\`\`diff
- <이전 문제 줄>
+ <수정된 줄>
\`\`\`

**Why:** <무엇이 잘못되었고 수정이 어떻게 해결하는지 1문장 설명>
```

**이미 수정됨(ALREADY FIXED)의 경우 (브랜치의 이전 커밋에서 이미 해결):**

```
**Already fixed** in `<commit-sha>`.

**What was done:** <기존 커밋이 이 문제를 어떻게 해결하는지 1-2문장 설명>
```

**오탐(FALSE POSITIVES)의 경우 (댓글이 잘못된 경우):**

```
**Not a bug.** <이것이 잘못된 이유를 직접적으로 1문장 서술>

**Evidence:**
- <패턴이 안전/올바름을 보여주는 구체적 코드 참조>
- <예: "nil 확인은 RecordNotFound를 발생시키는 `ActiveRecord::FinderMethods#find`에 의해 처리되며, nil이 아닙니다">

**Suggested re-rank:** 이것은 `<what Greptile called it>`이 아닌 `<style|noise|misread>` 문제로 보입니다. 심각도 하향 조정을 고려하세요.
```

### Tier 2 (Greptile이 이전 응답 후 재지적) — 단호하고, 압도적 근거

아래의 에스컬레이션 탐지에서 같은 스레드에 이전 GStack 응답이 있음을 식별할 때 Tier 2를 사용합니다. 논의를 종결하기 위해 최대한의 근거를 포함합니다.

```
**This has been reviewed and confirmed as [intentional/already-fixed/not-a-bug].**

\`\`\`diff
<변경 또는 안전한 패턴을 보여주는 전체 관련 diff>
\`\`\`

**Evidence chain:**
1. <안전한 패턴 또는 수정을 보여주는 file:line 퍼머링크>
2. <해결된 커밋 SHA, 해당하는 경우>
3. <아키텍처 근거 또는 설계 결정, 해당하는 경우>

**Suggested re-rank:** 재보정을 요청합니다 — 이것은 `<claimed category>`이 아닌 `<actual category>` 문제입니다. [도움이 되면 특정 파일 변경 퍼머링크 링크]
```

---

## 에스컬레이션 탐지(Escalation Detection)

응답을 작성하기 전에 이 댓글 스레드에 이전 GStack 응답이 있는지 확인합니다:

1. **줄 수준 댓글의 경우:** `gh api repos/$REPO/pulls/$PR_NUMBER/comments/$COMMENT_ID/replies`로 응답을 가져옵니다. 응답 본문에 GStack 마커가 포함되어 있는지 확인합니다: `**Fixed**`, `**Not a bug.**`, `**Already fixed**`.

2. **최상위 댓글의 경우:** 가져온 이슈 댓글 중 Greptile 댓글 이후에 게시된, GStack 마커를 포함하는 응답을 검색합니다.

3. **이전 GStack 응답이 있고 Greptile이 같은 파일+카테고리에 다시 게시한 경우:** Tier 2 (단호한) 템플릿을 사용합니다.

4. **이전 GStack 응답이 없는 경우:** Tier 1 (친절한) 템플릿을 사용합니다.

에스컬레이션 탐지가 실패하는 경우 (API 오류, 모호한 스레드): Tier 1로 기본 설정합니다. 모호한 상황에서 절대 에스컬레이션하지 마세요.

---

## 심각도 평가 및 재순위(Severity Assessment & Re-ranking)

댓글을 분류할 때 Greptile의 암시적 심각도가 현실과 일치하는지도 평가합니다:

- Greptile이 **보안/정확성/경쟁 조건** 문제로 지적했지만 실제로는 **스타일/성능** 수준의 사소한 문제인 경우: 응답에 `**Suggested re-rank:**`를 포함하여 카테고리 수정을 요청합니다.
- Greptile이 심각도가 낮은 스타일 문제를 치명적인 것처럼 지적한 경우: 응답에서 반박합니다.
- 재순위가 타당한 이유를 항상 구체적으로 명시합니다 — 의견이 아닌 코드와 줄 번호를 인용합니다.

---

## 이력 파일 쓰기(History File Writes)

쓰기 전에 두 디렉토리가 존재하는지 확인합니다:
```bash
REMOTE_SLUG=$(browse/bin/remote-slug 2>/dev/null || ~/.claude/skills/gstack/browse/bin/remote-slug 2>/dev/null || basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")
mkdir -p "$HOME/.gstack/projects/$REMOTE_SLUG"
mkdir -p ~/.gstack
```

각 분류 결과마다 **두** 파일에 한 줄씩 추가합니다 (프로젝트별은 억제용, 글로벌은 회고용):
- `~/.gstack/projects/$REMOTE_SLUG/greptile-history.md` (프로젝트별)
- `~/.gstack/greptile-history.md` (글로벌 집계)

형식:
```
<YYYY-MM-DD> | <owner/repo> | <type> | <file-pattern> | <category>
```

예시 항목:
```
2026-03-13 | garrytan/myapp | fp | app/services/auth_service.rb | race-condition
2026-03-13 | garrytan/myapp | fix | app/models/user.rb | null-check
2026-03-13 | garrytan/myapp | already-fixed | lib/payments.rb | error-handling
```

---

## 출력 형식

출력 헤더에 Greptile 요약을 포함합니다:
```
+ N Greptile comments (X valid, Y fixed, Z FP)
```

분류된 각 댓글에 대해 표시:
- 분류 태그: `[VALID]`, `[FIXED]`, `[FALSE POSITIVE]`, `[SUPPRESSED]`
- 파일:줄 참조 (줄 수준의 경우) 또는 `[top-level]` (최상위의 경우)
- 한 줄 본문 요약
- 퍼머링크 URL (`html_url` 필드)
