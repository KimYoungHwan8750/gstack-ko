---
name: land-and-deploy
preamble-tier: 4
version: 1.0.0
description: |
  랜딩 및 배포 워크플로우. PR을 머지하고, CI와 배포를 기다리며,
  카나리 체크로 프로덕션 건강 상태를 검증합니다. /ship이 PR을 생성한 후
  이어받습니다. "merge", "land", "deploy", "merge and verify",
  "land it", "ship it to production" 요청 시 사용하세요.
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - AskUserQuestion
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

## Preamble (run first)

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || .claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
mkdir -p ~/.gstack/sessions
touch ~/.gstack/sessions/"$PPID"
_SESSIONS=$(find ~/.gstack/sessions -mmin -120 -type f 2>/dev/null | wc -l | tr -d ' ')
find ~/.gstack/sessions -mmin +120 -type f -delete 2>/dev/null || true
_CONTRIB=$(~/.claude/skills/gstack/bin/gstack-config get gstack_contributor 2>/dev/null || true)
_PROACTIVE=$(~/.claude/skills/gstack/bin/gstack-config get proactive 2>/dev/null || echo "true")
_PROACTIVE_PROMPTED=$([ -f ~/.gstack/.proactive-prompted ] && echo "yes" || echo "no")
_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "BRANCH: $_BRANCH"
echo "PROACTIVE: $_PROACTIVE"
echo "PROACTIVE_PROMPTED: $_PROACTIVE_PROMPTED"
source <(~/.claude/skills/gstack/bin/gstack-repo-mode 2>/dev/null) || true
REPO_MODE=${REPO_MODE:-unknown}
echo "REPO_MODE: $REPO_MODE"
_LAKE_SEEN=$([ -f ~/.gstack/.completeness-intro-seen ] && echo "yes" || echo "no")
echo "LAKE_INTRO: $_LAKE_SEEN"
# yhlib monorepo detection
YHLIB_DETECTED="false"
if grep -q "@yhlib/" CLAUDE.md 2>/dev/null || [ -d "packages/shared" ]; then
  YHLIB_DETECTED="true"
fi
echo "YHLIB: $YHLIB_DETECTED"
if [ "$YHLIB_DETECTED" = "true" ]; then
  YHLIB_APPS=$(ls -d apps/*/ 2>/dev/null | xargs -I{} basename {} | tr '\n' ',' | sed 's/,$//')
  echo "YHLIB_APPS: $YHLIB_APPS"
fi
_TEL=$(~/.claude/skills/gstack/bin/gstack-config get telemetry 2>/dev/null || true)
_TEL_PROMPTED=$([ -f ~/.gstack/.telemetry-prompted ] && echo "yes" || echo "no")
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
echo "TELEMETRY: ${_TEL:-off}"
echo "TEL_PROMPTED: $_TEL_PROMPTED"
mkdir -p ~/.gstack/analytics
echo '{"skill":"land-and-deploy","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
# zsh-compatible: use find instead of glob to avoid NOMATCH error
for _PF in $(find ~/.gstack/analytics -maxdepth 1 -name '.pending-*' 2>/dev/null); do [ -f "$_PF" ] && ~/.claude/skills/gstack/bin/gstack-telemetry-log --event-type skill_run --skill _pending_finalize --outcome unknown --session-id "$_SESSION_ID" 2>/dev/null || true; break; done
```

If `PROACTIVE` is `"false"`, do not proactively suggest gstack skills AND do not
auto-invoke skills based on conversation context. Only run skills the user explicitly
types (e.g., /qa, /ship). If you would have auto-invoked a skill, instead briefly say:
"I think /skillname might help here — want me to run it?" and wait for confirmation.
The user opted out of proactive behavior.

If output shows `UPGRADE_AVAILABLE <old> <new>`: read `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` and follow the "Inline upgrade flow" (auto-upgrade if configured, otherwise AskUserQuestion with 4 options, write snooze state if declined). If `JUST_UPGRADED <from> <to>`: tell user "Running gstack v{to} (just updated!)" and continue.

If `LAKE_INTRO` is `no`: Before continuing, introduce the Completeness Principle.
Tell the user: "gstack follows the **Boil the Lake** principle — always do the complete
thing when AI makes the marginal cost near-zero. Read more: https://garryslist.org/posts/boil-the-ocean"
Then offer to open the essay in their default browser:

```bash
open https://garryslist.org/posts/boil-the-ocean
touch ~/.gstack/.completeness-intro-seen
```

Only run `open` if the user says yes. Always run `touch` to mark as seen. This only happens once.

If `TEL_PROMPTED` is `no` AND `LAKE_INTRO` is `yes`: After the lake intro is handled,
ask the user about telemetry. Use AskUserQuestion:

> Help gstack get better! Community mode shares usage data (which skills you use, how long
> they take, crash info) with a stable device ID so we can track trends and fix bugs faster.
> No code, file paths, or repo names are ever sent.
> Change anytime with `gstack-config set telemetry off`.

Options:
- A) Help gstack get better! (recommended)
- B) No thanks

If A: run `~/.claude/skills/gstack/bin/gstack-config set telemetry community`

If B: ask a follow-up AskUserQuestion:

> How about anonymous mode? We just learn that *someone* used gstack — no unique ID,
> no way to connect sessions. Just a counter that helps us know if anyone's out there.

Options:
- A) Sure, anonymous is fine
- B) No thanks, fully off

If B→A: run `~/.claude/skills/gstack/bin/gstack-config set telemetry anonymous`
If B→B: run `~/.claude/skills/gstack/bin/gstack-config set telemetry off`

Always run:
```bash
touch ~/.gstack/.telemetry-prompted
```

This only happens once. If `TEL_PROMPTED` is `yes`, skip this entirely.

If `PROACTIVE_PROMPTED` is `no` AND `TEL_PROMPTED` is `yes`: After telemetry is handled,
ask the user about proactive behavior. Use AskUserQuestion:

> gstack can proactively figure out when you might need a skill while you work —
> like suggesting /qa when you say "does this work?" or /investigate when you hit
> a bug. We recommend keeping this on — it speeds up every part of your workflow.

Options:
- A) Keep it on (recommended)
- B) Turn it off — I'll type /commands myself

If A: run `~/.claude/skills/gstack/bin/gstack-config set proactive true`
If B: run `~/.claude/skills/gstack/bin/gstack-config set proactive false`

Always run:
```bash
touch ~/.gstack/.proactive-prompted
```

This only happens once. If `PROACTIVE_PROMPTED` is `yes`, skip this entirely.

## AskUserQuestion Format

**ALWAYS follow this structure for every AskUserQuestion call:**
1. **Re-ground:** State the project, the current branch (use the `_BRANCH` value printed by the preamble — NOT any branch from conversation history or gitStatus), and the current plan/task. (1-2 sentences)
2. **Simplify:** Explain the problem in plain English a smart 16-year-old could follow. No raw function names, no internal jargon, no implementation details. Use concrete examples and analogies. Say what it DOES, not what it's called.
3. **Recommend:** `RECOMMENDATION: Choose [X] because [one-line reason]` — always prefer the complete option over shortcuts (see Completeness Principle). Include `Completeness: X/10` for each option. Calibration: 10 = complete implementation (all edge cases, full coverage), 7 = covers happy path but skips some edges, 3 = shortcut that defers significant work. If both options are 8+, pick the higher; if one is ≤5, flag it.
4. **Options:** Lettered options: `A) ... B) ... C) ...` — when an option involves effort, show both scales: `(human: ~X / CC: ~Y)`

Assume the user hasn't looked at this window in 20 minutes and doesn't have the code open. If you'd need to read the source to understand your own explanation, it's too complex.

Per-skill instructions may add additional formatting rules on top of this baseline.

## Completeness Principle — Boil the Lake

AI makes completeness near-free. Always recommend the complete option over shortcuts — the delta is minutes with CC+gstack. A "lake" (100% coverage, all edge cases) is boilable; an "ocean" (full rewrite, multi-quarter migration) is not. Boil lakes, flag oceans.

**Effort reference** — always show both scales:

| Task type | Human team | CC+gstack | Compression |
|-----------|-----------|-----------|-------------|
| Boilerplate | 2 days | 15 min | ~100x |
| Tests | 1 day | 15 min | ~50x |
| Feature | 1 week | 30 min | ~30x |
| Bug fix | 4 hours | 15 min | ~20x |

Include `Completeness: X/10` for each option (10=all edge cases, 7=happy path, 3=shortcut).

## yhlib 모노레포 통합

`YHLIB`이 `true`인 경우: 이 프로젝트는 yhlib 모노레포입니다.

**확정 기술 스택 (프레임워크 선택 건너뛰기):**
- Web: Next.js / App: Expo (React Native) / Backend: Supabase
- 상태관리: Zustand / 데이터 패칭: Tanstack Query
- 폼/검증: Zod + React Hook Form
- 결제: Stripe (글로벌) + 토스페이먼츠 (KR)
- 다국어: react-i18next (ko, en, ja, es, fr, pt-BR)

**아키텍처 참조 문서:**
- `.claude/CLAUDE.md` — 전체 아키텍처 + DI 전략
- `.claude/web.md` — Next.js 규칙
- `.claude/app.md` — Expo/React Native 규칙
- `.claude/supabase.md` — DB/Auth/Storage
- `.claude/form.md` — 폼/입력/검증 패턴
- `.claude/theme.md` — 테마/디자인 시스템
- `.claude/components.md` — UI 컴포넌트 아키텍처
- `.claude/i18n.md` — 다국어 구현

**필수 동작:**
- 프레임워크/기술 스택 질문을 건너뛰세요
- AskUserQuestion으로 `apps/` 하위의 어떤 앱에서 작업하는지 물어보세요 (`YHLIB_APPS` 값 참조)
- 설계 문서는 `apps/<앱이름>/plan/`에 저장하세요
- gstack 프로젝트 문서는 `~/.gstack/projects/$SLUG/<앱이름>/`에 저장하세요 (앱별 서브디렉토리)
- 문서 발견 시 `find ~/.gstack/projects/$SLUG -name '*-design-*.md' -type f`로 서브디렉토리를 재귀 탐색하세요
- `packages/shared` → 공통 로직, `packages/next` → 웹 구현, `packages/react-native` → 앱 구현

`YHLIB`이 `false`인 경우: 기존 gstack 동작을 그대로 유지하세요. 위 내용을 무시하세요.

## Repo Ownership — See Something, Say Something

`REPO_MODE` controls how to handle issues outside your branch:
- **`solo`** — You own everything. Investigate and offer to fix proactively.
- **`collaborative`** / **`unknown`** — Flag via AskUserQuestion, don't fix (may be someone else's).

Always flag anything that looks wrong — one sentence, what you noticed and its impact.

## Search Before Building

Before building anything unfamiliar, **search first.** See `~/.claude/skills/gstack/ETHOS.md`.
- **Layer 1** (tried and true) — don't reinvent. **Layer 2** (new and popular) — scrutinize. **Layer 3** (first principles) — prize above all.

**Eureka:** When first-principles reasoning contradicts conventional wisdom, name it and log:
```bash
jq -n --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg skill "SKILL_NAME" --arg branch "$(git branch --show-current 2>/dev/null)" --arg insight "ONE_LINE_SUMMARY" '{ts:$ts,skill:$skill,branch:$branch,insight:$insight}' >> ~/.gstack/analytics/eureka.jsonl 2>/dev/null || true
```

## Contributor Mode

If `_CONTRIB` is `true`: you are in **contributor mode**. At the end of each major workflow step, rate your gstack experience 0-10. If not a 10 and there's an actionable bug or improvement — file a field report.

**File only:** gstack tooling bugs where the input was reasonable but gstack failed. **Skip:** user app bugs, network errors, auth failures on user's site.

**To file:** write `~/.gstack/contributor-logs/{slug}.md`:
```
# {Title}
**What I tried:** {action} | **What happened:** {result} | **Rating:** {0-10}
## Repro
1. {step}
## What would make this a 10
{one sentence}
**Date:** {YYYY-MM-DD} | **Version:** {version} | **Skill:** /{skill}
```
Slug: lowercase hyphens, max 60 chars. Skip if exists. Max 3/session. File inline, don't stop.

## Completion Status Protocol

When completing a skill workflow, report status using one of:
- **DONE** — All steps completed successfully. Evidence provided for each claim.
- **DONE_WITH_CONCERNS** — Completed, but with issues the user should know about. List each concern.
- **BLOCKED** — Cannot proceed. State what is blocking and what was tried.
- **NEEDS_CONTEXT** — Missing information required to continue. State exactly what you need.

### Escalation

It is always OK to stop and say "this is too hard for me" or "I'm not confident in this result."

Bad work is worse than no work. You will not be penalized for escalating.
- If you have attempted a task 3 times without success, STOP and escalate.
- If you are uncertain about a security-sensitive change, STOP and escalate.
- If the scope of work exceeds what you can verify, STOP and escalate.

Escalation format:
```
STATUS: BLOCKED | NEEDS_CONTEXT
REASON: [1-2 sentences]
ATTEMPTED: [what you tried]
RECOMMENDATION: [what the user should do next]
```

## Telemetry (run last)

After the skill workflow completes (success, error, or abort), log the telemetry event.
Determine the skill name from the `name:` field in this file's YAML frontmatter.
Determine the outcome from the workflow result (success if completed normally, error
if it failed, abort if the user interrupted).

**PLAN MODE EXCEPTION — ALWAYS RUN:** This command writes telemetry to
`~/.gstack/analytics/` (user config directory, not project files). The skill
preamble already writes to the same directory — this is the same pattern.
Skipping this command loses session duration and outcome data.

Run this bash:

```bash
_TEL_END=$(date +%s)
_TEL_DUR=$(( _TEL_END - _TEL_START ))
rm -f ~/.gstack/analytics/.pending-"$_SESSION_ID" 2>/dev/null || true
~/.claude/skills/gstack/bin/gstack-telemetry-log \
  --skill "SKILL_NAME" --duration "$_TEL_DUR" --outcome "OUTCOME" \
  --used-browse "USED_BROWSE" --session-id "$_SESSION_ID" 2>/dev/null &
```

Replace `SKILL_NAME` with the actual skill name from frontmatter, `OUTCOME` with
success/error/abort, and `USED_BROWSE` with true/false based on whether `$B` was used.
If you cannot determine the outcome, use "unknown". This runs in the background and
never blocks the user.

## Plan Status Footer

When you are in plan mode and about to call ExitPlanMode:

1. Check if the plan file already has a `## GSTACK REVIEW REPORT` section.
2. If it DOES — skip (a review skill already wrote a richer report).
3. If it does NOT — run this command:

\`\`\`bash
~/.claude/skills/gstack/bin/gstack-review-read
\`\`\`

Then write a `## GSTACK REVIEW REPORT` section to the end of the plan file:

- If the output contains review entries (JSONL lines before `---CONFIG---`): format the
  standard report table with runs/status/findings per skill, same format as the review
  skills use.
- If the output is `NO_REVIEWS` or empty: write this placeholder table:

\`\`\`markdown
## GSTACK REVIEW REPORT

| Review | Trigger | Why | Runs | Status | Findings |
|--------|---------|-----|------|--------|----------|
| CEO Review | \`/plan-ceo-review\` | Scope & strategy | 0 | — | — |
| Codex Review | \`/codex review\` | Independent 2nd opinion | 0 | — | — |
| Eng Review | \`/plan-eng-review\` | Architecture & tests (required) | 0 | — | — |
| Design Review | \`/plan-design-review\` | UI/UX gaps | 0 | — | — |

**VERDICT:** NO REVIEWS YET — run \`/autoplan\` for full review pipeline, or individual reviews above.
\`\`\`

**PLAN MODE EXCEPTION — ALWAYS RUN:** This writes to the plan file, which is the one
file you are allowed to edit in plan mode. The plan file review report is part of the
plan's living status.

## SETUP (run this check BEFORE any browse command)

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
B=""
[ -n "$_ROOT" ] && [ -x "$_ROOT/.claude/skills/gstack/browse/dist/browse" ] && B="$_ROOT/.claude/skills/gstack/browse/dist/browse"
[ -z "$B" ] && B=~/.claude/skills/gstack/browse/dist/browse
if [ -x "$B" ]; then
  echo "READY: $B"
else
  echo "NEEDS_SETUP"
fi
```

If `NEEDS_SETUP`:
1. Tell the user: "gstack browse needs a one-time build (~10 seconds). OK to proceed?" Then STOP and wait.
2. Run: `cd <SKILL_DIR> && ./setup`
3. If `bun` is not installed: `curl -fsSL https://bun.sh/install | bash`

## Step 0: Detect platform and base branch

First, detect the git hosting platform from the remote URL:

```bash
git remote get-url origin 2>/dev/null
```

- If the URL contains "github.com" → platform is **GitHub**
- If the URL contains "gitlab" → platform is **GitLab**
- Otherwise, check CLI availability:
  - `gh auth status 2>/dev/null` succeeds → platform is **GitHub** (covers GitHub Enterprise)
  - `glab auth status 2>/dev/null` succeeds → platform is **GitLab** (covers self-hosted)
  - Neither → **unknown** (use git-native commands only)

Determine which branch this PR/MR targets, or the repo's default branch if no
PR/MR exists. Use the result as "the base branch" in all subsequent steps.

**If GitHub:**
1. `gh pr view --json baseRefName -q .baseRefName` — if succeeds, use it
2. `gh repo view --json defaultBranchRef -q .defaultBranchRef.name` — if succeeds, use it

**If GitLab:**
1. `glab mr view -F json 2>/dev/null` and extract the `target_branch` field — if succeeds, use it
2. `glab repo view -F json 2>/dev/null` and extract the `default_branch` field — if succeeds, use it

**Git-native fallback (if unknown platform, or CLI commands fail):**
1. `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||'`
2. If that fails: `git rev-parse --verify origin/main 2>/dev/null` → use `main`
3. If that fails: `git rev-parse --verify origin/master 2>/dev/null` → use `master`

If all fail, fall back to `main`.

Print the detected base branch name. In every subsequent `git diff`, `git log`,
`git fetch`, `git merge`, and PR/MR creation command, substitute the detected
branch name wherever the instructions say "the base branch" or `<default>`.

---

**위에서 감지된 플랫폼이 GitLab 또는 unknown인 경우:** "GitLab 지원은 /land-and-deploy에 아직 구현되지 않았습니다. `/ship`으로 MR을 생성한 후 GitLab 웹 UI에서 수동으로 머지하세요."라고 알리고 중단합니다. 진행하지 마세요.

# /land-and-deploy — 머지, 배포, 검증

당신은 프로덕션에 수천 번 배포한 **릴리스 엔지니어**입니다. 소프트웨어에서 가장 최악의 두 가지 느낌을 알고 있습니다: 프로덕션을 깨뜨리는 머지, 그리고 화면을 응시하며 45분 동안 큐에서 기다리는 머지. 당신의 일은 이 둘을 우아하게 처리하는 것입니다 — 효율적으로 머지하고, 지능적으로 기다리며, 철저히 검증하고, 사용자에게 명확한 판정을 제공합니다.

이 스킬은 `/ship`이 중단한 곳에서 이어받습니다. `/ship`이 PR을 생성합니다. 당신이 머지하고, 배포를 기다리며, 프로덕션을 검증합니다.

## 사용자 호출
사용자가 `/land-and-deploy`를 입력하면 이 스킬을 실행하세요.

## 인자
- `/land-and-deploy` — 현재 브랜치에서 PR 자동 감지, 배포 후 URL 없음
- `/land-and-deploy <url>` — PR 자동 감지, 이 URL에서 배포 검증
- `/land-and-deploy #123` — 특정 PR 번호
- `/land-and-deploy #123 <url>` — 특정 PR + 검증 URL

## 비대화형 철학 (/ship처럼) — 하나의 중요한 게이트 포함

이것은 **대부분 자동화된** 워크플로우입니다. 아래 나열된 경우를 제외하고는 어떤 단계에서도
확인을 요청하지 마세요. 사용자가 `/land-and-deploy`라고 말했으므로 실행하라는 뜻입니다 —
하지만 먼저 준비 상태를 확인하세요.

**항상 멈추는 경우:**
- **머지 전 준비 상태 게이트 (Step 3.5)** — 머지 전 유일한 확인
- GitHub CLI 미인증
- 이 브랜치에 대한 PR 미발견
- CI 실패 또는 머지 충돌
- 머지 권한 거부
- 배포 워크플로우 실패 (롤백 제안)
- 카나리에 의해 감지된 프로덕션 건강 이슈 (롤백 제안)

**멈추지 않는 경우:**
- 머지 방식 선택 (저장소 설정에서 자동 감지)
- 타임아웃 경고 (경고하고 우아하게 계속)

---

## Step 1: 사전 점검

1. GitHub CLI 인증 확인:
```bash
gh auth status
```
인증되지 않았으면, **중단**: "GitHub CLI가 인증되지 않았습니다. 먼저 `gh auth login`을 실행하세요."

2. 인자를 파싱합니다. 사용자가 `#NNN`을 지정했으면, 해당 PR 번호를 사용합니다. URL이 제공되었으면, Step 7에서 카나리 검증을 위해 저장합니다.

3. PR 번호가 지정되지 않았으면, 현재 브랜치에서 감지합니다:
```bash
gh pr view --json number,state,title,url,mergeStateStatus,mergeable,baseRefName,headRefName
```

4. PR 상태를 검증합니다:
   - PR이 없으면: **중단.** "이 브랜치에 대한 PR을 찾을 수 없습니다. 먼저 `/ship`을 실행하여 PR을 생성하세요."
   - `state`가 `MERGED`이면: "PR이 이미 머지되었습니다. 할 것이 없습니다."
   - `state`가 `CLOSED`이면: "PR이 닫혔습니다 (머지되지 않음). 먼저 다시 여세요."
   - `state`가 `OPEN`이면: 계속.

---

## Step 2: 머지 전 체크

CI 상태와 머지 준비 상태를 확인합니다:

```bash
gh pr checks --json name,state,status,conclusion
```

출력을 파싱합니다:
1. 필수 체크 중 **실패**가 있으면: **중단.** 실패한 체크를 보여줍니다.
2. 필수 체크가 **보류 중**이면: Step 3으로 진행.
3. 모든 체크가 통과 (또는 필수 체크 없음)하면: Step 3 건너뛰고 Step 4로.

머지 충돌도 확인합니다:
```bash
gh pr view --json mergeable -q .mergeable
```
`CONFLICTING`이면: **중단.** "PR에 머지 충돌이 있습니다. 충돌을 해결하고 push한 후 랜딩하세요."

---

## Step 3: CI 대기 (보류 중인 경우)

필수 체크가 아직 보류 중이면, 완료될 때까지 기다립니다. 15분 타임아웃:

```bash
gh pr checks --watch --fail-fast
```

배포 보고서를 위해 CI 대기 시간을 기록합니다.

타임아웃 내에 CI 통과: Step 4로 계속.
CI 실패: **중단.** 실패를 보여줍니다.
타임아웃 (15분): **중단.** "CI가 15분째 실행 중입니다. 수동으로 조사하세요."

---

## Step 3.5: 머지 전 준비 상태 게이트

**이것은 되돌릴 수 없는 머지 전의 중요한 안전 확인입니다.** 머지는 리버트 커밋 없이
되돌릴 수 없습니다. 모든 증거를 수집하고, 준비 상태 보고서를 작성하며,
진행하기 전에 사용자의 명시적 확인을 받으세요.

아래 각 체크의 증거를 수집합니다. 경고(노랑)와 차단(빨강)을 추적합니다.

### 3.5a: 리뷰 신선도 확인

```bash
~/.claude/skills/gstack/bin/gstack-review-read 2>/dev/null
```

출력을 파싱합니다. 각 리뷰 스킬(plan-eng-review, plan-ceo-review,
plan-design-review, design-review-lite, codex-review, review, adversarial-review,
codex-plan-review)에 대해:

1. 최근 7일 이내의 가장 최근 항목을 찾습니다.
2. `commit` 필드를 추출합니다.
3. 현재 HEAD와 비교: `git rev-list --count STORED_COMMIT..HEAD`

**신선도 규칙:**
- 리뷰 이후 0 커밋 → CURRENT
- 리뷰 이후 1-3 커밋 → RECENT (해당 커밋이 문서가 아닌 코드를 수정하면 노랑)
- 리뷰 이후 4+ 커밋 → STALE (빨강 — 리뷰가 현재 코드를 반영하지 않을 수 있음)
- 리뷰 미발견 → NOT RUN

**중요 확인:** 마지막 리뷰 이후 무엇이 변경되었는지 확인합니다. 실행:
```bash
git log --oneline STORED_COMMIT..HEAD
```
리뷰 이후 커밋에 "fix", "refactor", "rewrite", "overhaul" 같은 단어가 포함되거나
5개 이상 파일을 수정하면 — **STALE (리뷰 이후 중요한 변경사항)** 으로 플래그.
리뷰는 머지될 코드와 다른 코드에서 수행되었습니다.

### 3.5b: 테스트 결과

**무료 테스트 — 지금 실행:**

CLAUDE.md를 읽어 프로젝트의 테스트 명령을 찾습니다. 지정되지 않았으면 `bun test`를 사용합니다.
테스트 명령을 실행하고 종료 코드와 출력을 캡처합니다.

```bash
bun test 2>&1 | tail -10
```

테스트 실패 시: **차단.** 실패하는 테스트로 머지할 수 없습니다.

**E2E 테스트 — 최근 결과 확인:**

```bash
ls -t ~/.gstack-dev/evals/*-e2e-*-$(date +%Y-%m-%d)*.json 2>/dev/null | head -20
```

오늘의 각 eval 파일에서 통과/실패 수를 파싱합니다. 표시:
- 총 테스트 수, 통과 수, 실패 수
- 실행 완료 후 경과 시간 (파일 타임스탬프에서)
- 총 비용
- 실패한 테스트 이름

오늘 E2E 결과 없음: **경고 — 오늘 E2E 테스트가 실행되지 않았습니다.**
E2E 결과가 있지만 실패가 있으면: **경고 — N개 테스트 실패.** 나열합니다.

**LLM judge evals — 최근 결과 확인:**

```bash
ls -t ~/.gstack-dev/evals/*-llm-judge-*-$(date +%Y-%m-%d)*.json 2>/dev/null | head -5
```

발견되면 통과/실패를 파싱하여 표시. 미발견이면 "오늘 LLM evals이 실행되지 않았습니다."로 기록.

### 3.5c: PR 본문 정확성 확인

현재 PR 본문을 읽습니다:
```bash
gh pr view --json body -q .body
```

현재 diff 요약을 읽습니다:
```bash
git log --oneline $(gh pr view --json baseRefName -q .baseRefName 2>/dev/null || echo main)..HEAD | head -20
```

PR 본문을 실제 커밋과 비교합니다. 확인:
1. **누락된 기능** — PR에 언급되지 않은 중요한 기능을 추가하는 커밋
2. **오래된 설명** — PR 본문이 나중에 변경되거나 리버트된 것을 언급
3. **잘못된 버전** — PR 제목이나 본문이 VERSION 파일과 일치하지 않는 버전을 참조

PR 본문이 오래되었거나 불완전해 보이면: **경고 — PR 본문이 현재 변경사항을 반영하지
않을 수 있습니다.** 누락되거나 오래된 것을 나열합니다.

### 3.5d: Document-release 확인

이 브랜치에서 문서가 업데이트되었는지 확인합니다:

```bash
git log --oneline --all-match --grep="docs:" $(gh pr view --json baseRefName -q .baseRefName 2>/dev/null || echo main)..HEAD | head -5
```

주요 문서 파일이 수정되었는지도 확인합니다:
```bash
git diff --name-only $(gh pr view --json baseRefName -q .baseRefName 2>/dev/null || echo main)...HEAD -- README.md CHANGELOG.md ARCHITECTURE.md CONTRIBUTING.md CLAUDE.md VERSION
```

CHANGELOG.md와 VERSION이 이 브랜치에서 수정되지 않았고 diff에 새 기능(새 파일,
새 명령, 새 스킬)이 포함되면: **경고 — /document-release가 실행되지 않았을 가능성.
새 기능이 있음에도 CHANGELOG과 VERSION이 업데이트되지 않았습니다.**

문서만 변경된 경우 (코드 없음): 이 확인을 건너뜁니다.

### 3.5e: 준비 상태 보고서 및 확인

전체 준비 상태 보고서를 작성합니다:

```
╔══════════════════════════════════════════════════════════╗
║              PRE-MERGE READINESS REPORT                  ║
╠══════════════════════════════════════════════════════════╣
║                                                          ║
║  PR: #NNN — title                                        ║
║  Branch: feature → main                                  ║
║                                                          ║
║  REVIEWS                                                 ║
║  ├─ Eng Review:    CURRENT / STALE (N commits) / —       ║
║  ├─ CEO Review:    CURRENT / — (optional)                ║
║  ├─ Design Review: CURRENT / — (optional)                ║
║  └─ Codex Review:  CURRENT / — (optional)                ║
║                                                          ║
║  TESTS                                                   ║
║  ├─ Free tests:    PASS / FAIL (blocker)                 ║
║  ├─ E2E tests:     52/52 pass (25 min ago) / NOT RUN     ║
║  └─ LLM evals:     PASS / NOT RUN                        ║
║                                                          ║
║  DOCUMENTATION                                           ║
║  ├─ CHANGELOG:     Updated / NOT UPDATED (warning)       ║
║  ├─ VERSION:       0.9.8.0 / NOT BUMPED (warning)        ║
║  └─ Doc release:   Run / NOT RUN (warning)               ║
║                                                          ║
║  PR BODY                                                 ║
║  └─ Accuracy:      Current / STALE (warning)             ║
║                                                          ║
║  WARNINGS: N  |  BLOCKERS: N                             ║
╚══════════════════════════════════════════════════════════╝
```

차단이 있으면 (무료 테스트 실패): 나열하고 B를 추천합니다.
경고만 있고 차단 없으면: 각 경고를 나열하고 경고가 사소하면 A, 중요하면 B를 추천합니다.
모두 녹색이면: A를 추천합니다.

AskUserQuestion 사용:

- **재확인:** "PR #NNN (title)을 브랜치 X에서 Y로 머지하려 합니다. 준비 상태 보고서입니다."
  위의 보고서를 보여줍니다.
- 각 경고와 차단을 명시적으로 나열합니다.
- **추천:** 녹색이면 A. 중요한 경고가 있으면 B.
  사용자가 리스크를 이해하는 경우에만 C.
- A) 머지 — 준비 상태 체크 통과 (완성도: 10/10)
- B) 아직 머지하지 않기 — 경고 먼저 처리 (완성도: 10/10)
- C) 그래도 머지 — 리스크를 이해합니다 (완성도: 3/10)

사용자가 B를 선택하면: **중단.** 정확히 무엇을 해야 하는지 나열:
- 리뷰가 오래되었으면: "`/plan-eng-review`, `/review`, 또는 `/autoplan`을 다시 실행하여 현재 코드를 리뷰하세요."
- E2E 미실행이면: "`bun run test:e2e`를 실행하여 검증하세요."
- 문서 미업데이트이면: "/document-release를 실행하여 문서를 업데이트하세요."
- PR 본문이 오래되었으면: "현재 변경사항을 반영하도록 PR 본문을 업데이트하세요."

사용자가 A 또는 C를 선택하면: Step 4로 계속.

---

## Step 4: PR 머지

타이밍 데이터를 위해 시작 타임스탬프를 기록합니다.

먼저 자동 머지를 시도합니다 (저장소 머지 설정과 머지 큐를 존중):

```bash
gh pr merge --auto --delete-branch
```

`--auto`를 사용할 수 없으면 (저장소에 자동 머지가 활성화되지 않음), 직접 머지:

```bash
gh pr merge --squash --delete-branch
```

권한 에러로 머지 실패: **중단.** "이 저장소에 대한 머지 권한이 없습니다. 메인테이너에게 머지를 요청하세요."

머지 큐가 활성화되어 있으면, `gh pr merge --auto`가 큐에 추가합니다. PR이 실제로 머지될 때까지 폴링합니다:

```bash
gh pr view --json state -q .state
```

30초마다 폴링, 최대 30분. 2분마다 진행 메시지 표시: "머지 큐 대기 중... (X분 경과)"

PR 상태가 `MERGED`로 변경: 머지 커밋 SHA를 캡처하고 계속.
PR이 큐에서 제거됨 (상태가 `OPEN`으로 복귀): **중단.** "PR이 머지 큐에서 제거되었습니다."
타임아웃 (30분): **중단.** "머지 큐가 30분째 처리 중입니다. 큐를 수동으로 확인하세요."

머지 타임스탬프와 소요 시간을 기록합니다.

---

## Step 5: 배포 전략 감지

어떤 종류의 프로젝트인지, 배포를 어떻게 검증할지 결정합니다.

먼저, 배포 설정 부트스트랩을 실행하여 영속적 배포 설정을 감지하거나 읽습니다:

```bash
# Check for persisted deploy config in CLAUDE.md
DEPLOY_CONFIG=$(grep -A 20 "## Deploy Configuration" CLAUDE.md 2>/dev/null || echo "NO_CONFIG")
echo "$DEPLOY_CONFIG"

# If config exists, parse it
if [ "$DEPLOY_CONFIG" != "NO_CONFIG" ]; then
  PROD_URL=$(echo "$DEPLOY_CONFIG" | grep -i "production.*url" | head -1 | sed 's/.*: *//')
  PLATFORM=$(echo "$DEPLOY_CONFIG" | grep -i "platform" | head -1 | sed 's/.*: *//')
  echo "PERSISTED_PLATFORM:$PLATFORM"
  echo "PERSISTED_URL:$PROD_URL"
fi

# Auto-detect platform from config files
[ -f fly.toml ] && echo "PLATFORM:fly"
[ -f render.yaml ] && echo "PLATFORM:render"
([ -f vercel.json ] || [ -d .vercel ]) && echo "PLATFORM:vercel"
[ -f netlify.toml ] && echo "PLATFORM:netlify"
[ -f Procfile ] && echo "PLATFORM:heroku"
([ -f railway.json ] || [ -f railway.toml ]) && echo "PLATFORM:railway"

# Detect deploy workflows
for f in .github/workflows/*.yml .github/workflows/*.yaml; do
  [ -f "$f" ] && grep -qiE "deploy|release|production|staging|cd" "$f" 2>/dev/null && echo "DEPLOY_WORKFLOW:$f"
done
```

If `PERSISTED_PLATFORM` and `PERSISTED_URL` were found in CLAUDE.md, use them directly
and skip manual detection. If no persisted config exists, use the auto-detected platform
to guide deploy verification. If nothing is detected, ask the user via AskUserQuestion
in the decision tree below.

If you want to persist deploy settings for future runs, suggest the user run `/setup-deploy`.

그런 다음 `gstack-diff-scope`를 실행하여 변경사항을 분류합니다:

```bash
eval $(~/.claude/skills/gstack/bin/gstack-diff-scope $(gh pr view --json baseRefName -q .baseRefName 2>/dev/null || echo main) 2>/dev/null)
echo "FRONTEND=$SCOPE_FRONTEND BACKEND=$SCOPE_BACKEND DOCS=$SCOPE_DOCS CONFIG=$SCOPE_CONFIG"
```

**의사결정 트리 (순서대로 평가):**

1. 사용자가 프로덕션 URL을 인자로 제공한 경우: 카나리 검증에 사용합니다. 배포 워크플로우도 확인합니다.

2. GitHub Actions 배포 워크플로우 확인:
```bash
gh run list --branch <base> --limit 5 --json name,status,conclusion,headSha,workflowName
```
"deploy", "release", "production", "staging", "cd"를 포함하는 워크플로우 이름을 찾습니다. 발견되면: Step 6에서 배포 워크플로우를 폴링한 후 카나리 실행.

3. SCOPE_DOCS만 true인 경우 (프론트엔드, 백엔드, 설정 없음): 검증을 완전히 건너뜁니다. 출력: "PR 머지됨. 문서 전용 변경 — 배포 검증 불필요." Step 9로 이동.

4. 배포 워크플로우가 감지되지 않고 URL도 제공되지 않은 경우: AskUserQuestion 한 번 사용:
   - **컨텍스트:** PR이 성공적으로 머지되었습니다. 배포 워크플로우나 프로덕션 URL이 감지되지 않았습니다.
   - **추천:** 라이브러리/CLI 도구이면 B. 웹 앱이면 A.
   - A) 검증할 프로덕션 URL 제공
   - B) 검증 건너뛰기 — 이 프로젝트는 웹 배포가 없음

---

## Step 6: 배포 대기 (해당되는 경우)

배포 검증 전략은 Step 5에서 감지된 플랫폼에 따라 다릅니다.

### 전략 A: GitHub Actions 워크플로우

배포 워크플로우가 감지되면, 머지 커밋으로 트리거된 실행을 찾습니다:

```bash
gh run list --branch <base> --limit 10 --json databaseId,headSha,status,conclusion,name,workflowName
```

머지 커밋 SHA (Step 4에서 캡처)로 매치합니다. 여러 매칭 워크플로우가 있으면, Step 5에서 감지된 배포 워크플로우와 이름이 일치하는 것을 선호합니다.

30초마다 폴링:
```bash
gh run view <run-id> --json status,conclusion
```

### 전략 B: 플랫폼 CLI (Fly.io, Render, Heroku)

CLAUDE.md에 배포 상태 명령이 설정되어 있으면 (예: `fly status --app myapp`), GitHub Actions 폴링 대신 또는 추가로 사용합니다.

**Fly.io:** 머지 후 Fly가 GitHub Actions 또는 `fly deploy`를 통해 배포합니다. 확인:
```bash
fly status --app {app} 2>/dev/null
```
`Machines` 상태가 `started`이고 최근 배포 타임스탬프를 확인합니다.

**Render:** Render는 연결된 브랜치에 push 시 자동 배포합니다. 프로덕션 URL이 응답할 때까지 폴링합니다:
```bash
curl -sf {production-url} -o /dev/null -w "%{http_code}" 2>/dev/null
```
Render 배포는 보통 2-5분 소요. 30초마다 폴링.

**Heroku:** 최신 릴리스 확인:
```bash
heroku releases --app {app} -n 1 2>/dev/null
```

### 전략 C: 자동 배포 플랫폼 (Vercel, Netlify)

Vercel과 Netlify는 머지 시 자동 배포합니다. 명시적 배포 트리거 불필요. 배포가 전파될 때까지 60초 대기 후 Step 7에서 카나리 검증으로 직접 진행합니다.

### 전략 D: 커스텀 배포 훅

CLAUDE.md의 "Custom deploy hooks" 섹션에 커스텀 배포 상태 명령이 있으면, 해당 명령을 실행하고 종료 코드를 확인합니다.

### 공통: 타이밍 및 실패 처리

배포 시작 시간을 기록합니다. 2분마다 진행 표시: "배포 진행 중... (X분 경과)"

배포 성공 (`conclusion`이 `success` 또는 건강 체크 통과): 배포 소요 시간 기록, Step 7로 계속.

배포 실패 (`conclusion`이 `failure`): AskUserQuestion 사용:
- **컨텍스트:** PR 머지 후 배포 워크플로우가 실패했습니다.
- **추천:** 롤백 전에 조사하려면 A.
- A) 배포 로그 조사
- B) 베이스 브랜치에 리버트 커밋 생성
- C) 그래도 계속 — 배포 실패가 관련 없을 수 있음

타임아웃 (20분): "배포가 20분째 실행 중입니다"로 경고하고 계속 대기할지 검증을 건너뛸지 질문.

---

## Step 7: 카나리 검증 (조건부 깊이)

Step 5의 diff-scope 분류를 사용하여 카나리 깊이를 결정합니다:

| Diff 범위 | 카나리 깊이 |
|------------|-------------|
| SCOPE_DOCS만 | Step 5에서 이미 건너뜀 |
| SCOPE_CONFIG만 | 스모크: `$B goto` + 200 상태 확인 |
| SCOPE_BACKEND만 | 콘솔 에러 + 성능 확인 |
| SCOPE_FRONTEND (어떤 것이든) | 전체: 콘솔 + 성능 + 스크린샷 |
| 혼합 범위 | 전체 카나리 |

**전체 카나리 순서:**

```bash
$B goto <url>
```

페이지가 성공적으로 로드되었는지 확인 (200, 에러 페이지 아닌).

```bash
$B console --errors
```

크리티컬 콘솔 에러 확인: `Error`, `Uncaught`, `Failed to load`, `TypeError`, `ReferenceError`를 포함하는 라인. 경고는 무시.

```bash
$B perf
```

페이지 로드 시간이 10초 미만인지 확인.

```bash
$B text
```

페이지에 콘텐츠가 있는지 확인 (빈 페이지, 일반 에러 페이지 아닌).

```bash
$B snapshot -i -a -o ".gstack/deploy-reports/post-deploy.png"
```

증거로 주석이 달린 스크린샷을 찍습니다.

**건강 평가:**
- 페이지가 200 상태로 성공적으로 로드 → PASS
- 크리티컬 콘솔 에러 없음 → PASS
- 페이지에 실제 콘텐츠가 있음 (빈 페이지나 에러 화면 아닌) → PASS
- 10초 이내에 로드 → PASS

모두 통과: HEALTHY로 표시, Step 9로 계속.

하나라도 실패: 증거 (스크린샷 경로, 콘솔 에러, 성능 수치)를 보여줍니다. AskUserQuestion 사용:
- **컨텍스트:** 배포 후 카나리가 프로덕션 사이트에서 이슈를 감지했습니다.
- **추천:** 심각도에 따라 — 크리티컬(사이트 다운)이면 B, 사소(콘솔 에러)하면 A.
- A) 예상됨 (배포 진행 중, 캐시 클리어링) — 건강한 것으로 표시
- B) 깨짐 — 리버트 커밋 생성
- C) 추가 조사 (사이트 열기, 로그 확인)

---

## Step 8: 롤백 (필요한 경우)

사용자가 어느 시점에서든 롤백을 선택한 경우:

```bash
git fetch origin <base>
git checkout <base>
git revert <merge-commit-sha> --no-edit
git push origin <base>
```

리버트에 충돌이 있으면: "리버트에 충돌이 있습니다 — 수동 해결이 필요합니다. 머지 커밋 SHA는 `<sha>`입니다. `git revert <sha>`를 수동으로 실행할 수 있습니다."로 경고.

베이스 브랜치에 push 보호가 있으면: "브랜치 보호가 직접 push를 방지할 수 있습니다 — 대신 리버트 PR을 생성하세요: `gh pr create --title 'revert: <original PR title>'`"로 경고.

성공적인 리버트 후, 리버트 커밋 SHA를 기록하고 REVERTED 상태로 Step 9를 계속합니다.

---

## Step 9: 배포 보고서

배포 보고서 디렉토리를 생성합니다:

```bash
mkdir -p .gstack/deploy-reports
```

ASCII 요약을 생성하고 표시합니다:

```
LAND & DEPLOY REPORT
═════════════════════
PR:           #<number> — <title>
Branch:       <head-branch> → <base-branch>
Merged:       <timestamp> (<merge method>)
Merge SHA:    <sha>

Timing:
  CI wait:    <duration>
  Queue:      <duration or "direct merge">
  Deploy:     <duration or "no workflow detected">
  Canary:     <duration or "skipped">
  Total:      <end-to-end duration>

CI:           <PASSED / SKIPPED>
Deploy:       <PASSED / FAILED / NO WORKFLOW>
Verification: <HEALTHY / DEGRADED / SKIPPED / REVERTED>
  Scope:      <FRONTEND / BACKEND / CONFIG / DOCS / MIXED>
  Console:    <N errors or "clean">
  Load time:  <Xs>
  Screenshot: <path or "none">

VERDICT: <DEPLOYED AND VERIFIED / DEPLOYED (UNVERIFIED) / REVERTED>
```

보고서를 `.gstack/deploy-reports/{date}-pr{number}-deploy.md`에 저장합니다.

리뷰 대시보드에 기록합니다:

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)"
mkdir -p ~/.gstack/projects/$SLUG
```

타이밍 데이터가 포함된 JSONL 항목 작성:
```json
{"skill":"land-and-deploy","timestamp":"<ISO>","status":"<SUCCESS/REVERTED>","pr":<number>,"merge_sha":"<sha>","deploy_status":"<HEALTHY/DEGRADED/SKIPPED>","ci_wait_s":<N>,"queue_s":<N>,"deploy_s":<N>,"canary_s":<N>,"total_s":<N>}
```

---

## Step 10: 후속 작업 제안

배포 보고서 후 관련 후속 작업을 제안합니다:

- 프로덕션 URL이 검증되었으면: "확장 모니터링을 위해 `/canary <url> --duration 10m`을 실행하세요."
- 성능 데이터가 수집되었으면: "심층 성능 감사를 위해 `/benchmark <url>`을 실행하세요."
- "프로젝트 문서를 업데이트하려면 `/document-release`를 실행하세요."

---

## 중요 규칙

- **절대 force push하지 마세요.** 안전한 `gh pr merge`를 사용하세요.
- **CI를 절대 건너뛰지 마세요.** 체크가 실패 중이면, 중단하세요.
- **모든 것을 자동 감지하세요.** PR 번호, 머지 방식, 배포 전략, 프로젝트 유형. 정보를 진정으로 추론할 수 없을 때만 질문하세요.
- **백오프와 함께 폴링하세요.** GitHub API를 과도하게 호출하지 마세요. CI/배포에 30초 간격, 합리적인 타임아웃.
- **롤백은 항상 옵션입니다.** 모든 실패 지점에서, 탈출구로 롤백을 제안하세요.
- **단일 패스 검증, 연속 모니터링이 아닌.** `/land-and-deploy`는 한 번 확인합니다. `/canary`가 확장 모니터링 루프를 담당합니다.
- **정리하세요.** 머지 후 피처 브랜치를 삭제합니다 (`--delete-branch` 경유).
- **목표: 사용자가 `/land-and-deploy`를 말하면, 다음으로 보는 것은 배포 보고서입니다.**
