---
name: ship
preamble-tier: 4
version: 1.0.0
description: |
  Ship 워크플로우: 베이스 브랜치 감지 + 병합, 테스트 실행, diff 리뷰, VERSION 범프, CHANGELOG 업데이트, 커밋, 푸시, PR 생성. "ship", "deploy", "push to main", "create a PR", "merge and push"라고 요청할 때 사용합니다.
  사용자가 코드가 준비됐다고 하거나 배포에 대해 물을 때 사전 제안합니다.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
  - WebSearch
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
_TEL=$(~/.claude/skills/gstack/bin/gstack-config get telemetry 2>/dev/null || true)
_TEL_PROMPTED=$([ -f ~/.gstack/.telemetry-prompted ] && echo "yes" || echo "no")
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
echo "TELEMETRY: ${_TEL:-off}"
echo "TEL_PROMPTED: $_TEL_PROMPTED"
mkdir -p ~/.gstack/analytics
echo '{"skill":"ship","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
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
- AskUserQuestion으로 `apps/` 하위의 어떤 앱에서 작업하는지 물어보세요
- 설계 문서는 `apps/<앱이름>/plan/`에 저장하세요
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

# Ship: 완전 자동화된 Ship 워크플로우

`/ship` 워크플로우를 실행 중입니다. 이것은 **비대화형, 완전 자동화** 워크플로우입니다. 어떤 단계에서도 확인을 요청하지 마십시오. 사용자가 `/ship`이라고 했으면 그냥 실행하라는 뜻입니다. 끝까지 쭉 실행하고 마지막에 PR URL을 출력하십시오.

**다음 경우에만 중단:**
- 베이스 브랜치에 있을 때 (중단)
- 자동 해결할 수 없는 병합 충돌 (중단, 충돌 표시)
- 인브랜치 테스트 실패 (기존 실패는 분류하며, 자동 차단하지 않음)
- 사전 착륙 검증(pre-landing verification)에서 사용자 판단이 필요한 ASK 항목 발견
- MINOR 또는 MAJOR 버전 범프 필요 (물어봄 — Step 4 참조)
- 사용자 결정이 필요한 Greptile 리뷰 코멘트 (복잡한 수정, 오탐)
- AI 평가 커버리지가 최소 임계값 미만 (사용자 재정의 가능한 하드 게이트 — Step 3.4 참조)
- 사용자 재정의 없이 NOT DONE인 플랜 항목 (Step 3.45 참조)
- 플랜 검증 실패 (Step 3.47 참조)
- TODOS.md가 없고 사용자가 생성을 원할 때 (물어봄 — Step 5.5 참조)
- TODOS.md가 정리되지 않았고 사용자가 재정리를 원할 때 (물어봄 — Step 5.5 참조)

**절대 중단하지 않는 경우:**
- 커밋되지 않은 변경사항 (항상 포함)
- 버전 범프 선택 (자동으로 MICRO 또는 PATCH 선택 — Step 4 참조)
- CHANGELOG 내용 (diff에서 자동 생성)
- 커밋 메시지 승인 (자동 커밋)
- 다중 파일 변경셋 (이등분 가능(bisectable) 커밋으로 자동 분할)
- TODOS.md 완료 항목 감지 (자동 표시)
- 자동 수정 가능한 리뷰 발견사항 (데드 코드, N+1, 오래된 주석 — 자동 수정)
- 목표 임계값 내의 테스트 커버리지 갭 (자동 생성 및 커밋, 또는 PR 본문에 표기)

---

## Step 1: 사전 비행 점검

1. 현재 브랜치를 확인합니다. 베이스 브랜치 또는 저장소의 기본 브랜치에 있으면 **중단**: "You're on the base branch. Ship from a feature branch."

2. `git status`를 실행합니다 (`-uall`은 절대 사용하지 않음). 커밋되지 않은 변경사항은 항상 포함됩니다 — 물어볼 필요 없습니다.

3. `git diff <base>...HEAD --stat`과 `git log <base>..HEAD --oneline`을 실행하여 무엇이 배포되는지 파악합니다.

4. 리뷰 준비 상태를 확인합니다:

## Review Readiness Dashboard

After completing the review, read the review log and config to display the dashboard.

```bash
~/.claude/skills/gstack/bin/gstack-review-read
```

Parse the output. Find the most recent entry for each skill (plan-ceo-review, plan-eng-review, review, plan-design-review, design-review-lite, adversarial-review, codex-review, codex-plan-review). Ignore entries with timestamps older than 7 days. For the Eng Review row, show whichever is more recent between `review` (diff-scoped pre-landing review) and `plan-eng-review` (plan-stage architecture review). Append "(DIFF)" or "(PLAN)" to the status to distinguish. For the Adversarial row, show whichever is more recent between `adversarial-review` (new auto-scaled) and `codex-review` (legacy). For Design Review, show whichever is more recent between `plan-design-review` (full visual audit) and `design-review-lite` (code-level check). Append "(FULL)" or "(LITE)" to the status to distinguish. For the Outside Voice row, show the most recent `codex-plan-review` entry — this captures outside voices from both /plan-ceo-review and /plan-eng-review.

**Source attribution:** If the most recent entry for a skill has a \`"via"\` field, append it to the status label in parentheses. Examples: `plan-eng-review` with `via:"autoplan"` shows as "CLEAR (PLAN via /autoplan)". `review` with `via:"ship"` shows as "CLEAR (DIFF via /ship)". Entries without a `via` field show as "CLEAR (PLAN)" or "CLEAR (DIFF)" as before.

Note: `autoplan-voices` and `design-outside-voices` entries are audit-trail-only (forensic data for cross-model consensus analysis). They do not appear in the dashboard and are not checked by any consumer.

Display:

```
+====================================================================+
|                    REVIEW READINESS DASHBOARD                       |
+====================================================================+
| Review          | Runs | Last Run            | Status    | Required |
|-----------------|------|---------------------|-----------|----------|
| Eng Review      |  1   | 2026-03-16 15:00    | CLEAR     | YES      |
| CEO Review      |  0   | —                   | —         | no       |
| Design Review   |  0   | —                   | —         | no       |
| Adversarial     |  0   | —                   | —         | no       |
| Outside Voice   |  0   | —                   | —         | no       |
+--------------------------------------------------------------------+
| VERDICT: CLEARED — Eng Review passed                                |
+====================================================================+
```

**Review tiers:**
- **Eng Review (required by default):** The only review that gates shipping. Covers architecture, code quality, tests, performance. Can be disabled globally with \`gstack-config set skip_eng_review true\` (the "don't bother me" setting).
- **CEO Review (optional):** Use your judgment. Recommend it for big product/business changes, new user-facing features, or scope decisions. Skip for bug fixes, refactors, infra, and cleanup.
- **Design Review (optional):** Use your judgment. Recommend it for UI/UX changes. Skip for backend-only, infra, or prompt-only changes.
- **Adversarial Review (automatic):** Auto-scales by diff size. Small diffs (<50 lines) skip adversarial. Medium diffs (50–199) get cross-model adversarial. Large diffs (200+) get all 4 passes: Claude structured, Codex structured, Claude adversarial subagent, Codex adversarial. No configuration needed.
- **Outside Voice (optional):** Independent plan review from a different AI model. Offered after all review sections complete in /plan-ceo-review and /plan-eng-review. Falls back to Claude subagent if Codex is unavailable. Never gates shipping.

**Verdict logic:**
- **CLEARED**: Eng Review has >= 1 entry within 7 days from either \`review\` or \`plan-eng-review\` with status "clean" (or \`skip_eng_review\` is \`true\`)
- **NOT CLEARED**: Eng Review missing, stale (>7 days), or has open issues
- CEO, Design, and Codex reviews are shown for context but never block shipping
- If \`skip_eng_review\` config is \`true\`, Eng Review shows "SKIPPED (global)" and verdict is CLEARED

**Staleness detection:** After displaying the dashboard, check if any existing reviews may be stale:
- Parse the \`---HEAD---\` section from the bash output to get the current HEAD commit hash
- For each review entry that has a \`commit\` field: compare it against the current HEAD. If different, count elapsed commits: \`git rev-list --count STORED_COMMIT..HEAD\`. Display: "Note: {skill} review from {date} may be stale — {N} commits since review"
- For entries without a \`commit\` field (legacy entries): display "Note: {skill} review from {date} has no commit tracking — consider re-running for accurate staleness detection"
- If all reviews match the current HEAD, do not display any staleness notes

Eng Review가 "CLEAR"가 아닌 경우:

출력: "No prior eng review found — ship will run its own pre-landing review in Step 3.5."

diff 크기를 확인합니다: `git diff <base>...HEAD --stat | tail -1`. diff가 200줄을 초과하면 추가: "Note: This is a large diff. Consider running `/plan-eng-review` or `/autoplan` for architecture-level review before shipping."

CEO Review가 없으면 정보 제공용으로 언급합니다 ("CEO Review not run — recommended for product changes") 하지만 절대 차단하지 않습니다.

Design Review의 경우: `source <(~/.claude/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)`를 실행합니다. `SCOPE_FRONTEND=true`이고 대시보드에 디자인 리뷰(plan-design-review 또는 design-review-lite)가 없으면 언급합니다: "Design Review not run — this PR changes frontend code. The lite design check will run automatically in Step 3.5, but consider running /design-review for a full visual audit post-implementation." 여전히 절대 차단하지 않습니다.

Step 1.5로 계속합니다 — 차단하거나 물어보지 마십시오. Ship은 Step 3.5에서 자체 리뷰를 실행합니다.

---

## Step 1.5: 배포 파이프라인 확인

diff가 새로운 독립 실행형 아티팩트(CLI 바이너리, 라이브러리 패키지, 도구)를 도입하는 경우 — 기존 배포가 있는 웹 서비스가 아닌 — 배포 파이프라인이 존재하는지 확인합니다.

1. diff가 새로운 `cmd/` 디렉토리, `main.go`, 또는 `bin/` 진입점을 추가하는지 확인합니다:
   ```bash
   git diff origin/<base> --name-only | grep -E '(cmd/.*/main\.go|bin/|Cargo\.toml|setup\.py|package\.json)' | head -5
   ```

2. 새 아티팩트가 감지되면 릴리스 워크플로우가 있는지 확인합니다:
   ```bash
   ls .github/workflows/ 2>/dev/null | grep -iE 'release|publish|dist'
   grep -qE 'release|publish|deploy' .gitlab-ci.yml 2>/dev/null && echo "GITLAB_CI_RELEASE"
   ```

3. **릴리스 파이프라인이 없고 새 아티팩트가 추가된 경우:** AskUserQuestion을 사용합니다:
   - "This PR adds a new binary/tool but there's no CI/CD pipeline to build and publish it.
     Users won't be able to download the artifact after merge."
   - A) 지금 릴리스 워크플로우 추가 (CI/CD 릴리스 파이프라인 — 플랫폼에 따라 GitHub Actions 또는 GitLab CI)
   - B) 나중으로 미룸 — TODOS.md에 추가
   - C) 불필요 — 내부용/웹 전용이며 기존 배포가 처리함

4. **릴리스 파이프라인이 존재하면:** 조용히 계속합니다.
5. **새 아티팩트가 감지되지 않으면:** 조용히 건너뜁니다.

---

## Step 2: 베이스 브랜치 병합 (테스트 전에)

베이스 브랜치를 피처 브랜치에 페치 및 병합하여 테스트가 병합된 상태에서 실행되도록 합니다:

```bash
git fetch origin <base> && git merge origin/<base> --no-edit
```

**병합 충돌이 있는 경우:** 단순한 충돌(VERSION, schema.rb, CHANGELOG 순서)은 자동 해결을 시도합니다. 충돌이 복잡하거나 모호하면 **중단**하고 보여줍니다.

**이미 최신 상태인 경우:** 조용히 계속합니다.

---

## Step 2.5: 테스트 프레임워크 부트스트랩

## Test Framework Bootstrap

**Detect existing test framework and project runtime:**

```bash
# Detect project runtime
[ -f Gemfile ] && echo "RUNTIME:ruby"
[ -f package.json ] && echo "RUNTIME:node"
[ -f requirements.txt ] || [ -f pyproject.toml ] && echo "RUNTIME:python"
[ -f go.mod ] && echo "RUNTIME:go"
[ -f Cargo.toml ] && echo "RUNTIME:rust"
[ -f composer.json ] && echo "RUNTIME:php"
[ -f mix.exs ] && echo "RUNTIME:elixir"
# Detect sub-frameworks
[ -f Gemfile ] && grep -q "rails" Gemfile 2>/dev/null && echo "FRAMEWORK:rails"
[ -f package.json ] && grep -q '"next"' package.json 2>/dev/null && echo "FRAMEWORK:nextjs"
# Check for existing test infrastructure
ls jest.config.* vitest.config.* playwright.config.* .rspec pytest.ini pyproject.toml phpunit.xml 2>/dev/null
ls -d test/ tests/ spec/ __tests__/ cypress/ e2e/ 2>/dev/null
# Check opt-out marker
[ -f .gstack/no-test-bootstrap ] && echo "BOOTSTRAP_DECLINED"
```

**If test framework detected** (config files or test directories found):
Print "Test framework detected: {name} ({N} existing tests). Skipping bootstrap."
Read 2-3 existing test files to learn conventions (naming, imports, assertion style, setup patterns).
Store conventions as prose context for use in Phase 8e.5 or Step 3.4. **Skip the rest of bootstrap.**

**If BOOTSTRAP_DECLINED** appears: Print "Test bootstrap previously declined — skipping." **Skip the rest of bootstrap.**

**If NO runtime detected** (no config files found): Use AskUserQuestion:
"I couldn't detect your project's language. What runtime are you using?"
Options: A) Node.js/TypeScript B) Ruby/Rails C) Python D) Go E) Rust F) PHP G) Elixir H) This project doesn't need tests.
If user picks H → write `.gstack/no-test-bootstrap` and continue without tests.

**If runtime detected but no test framework — bootstrap:**

### B2. Research best practices

Use WebSearch to find current best practices for the detected runtime:
- `"[runtime] best test framework 2025 2026"`
- `"[framework A] vs [framework B] comparison"`

If WebSearch is unavailable, use this built-in knowledge table:

| Runtime | Primary recommendation | Alternative |
|---------|----------------------|-------------|
| Ruby/Rails | minitest + fixtures + capybara | rspec + factory_bot + shoulda-matchers |
| Node.js | vitest + @testing-library | jest + @testing-library |
| Next.js | vitest + @testing-library/react + playwright | jest + cypress |
| Python | pytest + pytest-cov | unittest |
| Go | stdlib testing + testify | stdlib only |
| Rust | cargo test (built-in) + mockall | — |
| PHP | phpunit + mockery | pest |
| Elixir | ExUnit (built-in) + ex_machina | — |

### B3. Framework selection

Use AskUserQuestion:
"I detected this is a [Runtime/Framework] project with no test framework. I researched current best practices. Here are the options:
A) [Primary] — [rationale]. Includes: [packages]. Supports: unit, integration, smoke, e2e
B) [Alternative] — [rationale]. Includes: [packages]
C) Skip — don't set up testing right now
RECOMMENDATION: Choose A because [reason based on project context]"

If user picks C → write `.gstack/no-test-bootstrap`. Tell user: "If you change your mind later, delete `.gstack/no-test-bootstrap` and re-run." Continue without tests.

If multiple runtimes detected (monorepo) → ask which runtime to set up first, with option to do both sequentially.

### B4. Install and configure

1. Install the chosen packages (npm/bun/gem/pip/etc.)
2. Create minimal config file
3. Create directory structure (test/, spec/, etc.)
4. Create one example test matching the project's code to verify setup works

If package installation fails → debug once. If still failing → revert with `git checkout -- package.json package-lock.json` (or equivalent for the runtime). Warn user and continue without tests.

### B4.5. First real tests

Generate 3-5 real tests for existing code:

1. **Find recently changed files:** `git log --since=30.days --name-only --format="" | sort | uniq -c | sort -rn | head -10`
2. **Prioritize by risk:** Error handlers > business logic with conditionals > API endpoints > pure functions
3. **For each file:** Write one test that tests real behavior with meaningful assertions. Never `expect(x).toBeDefined()` — test what the code DOES.
4. Run each test. Passes → keep. Fails → fix once. Still fails → delete silently.
5. Generate at least 1 test, cap at 5.

Never import secrets, API keys, or credentials in test files. Use environment variables or test fixtures.

### B5. Verify

```bash
# Run the full test suite to confirm everything works
{detected test command}
```

If tests fail → debug once. If still failing → revert all bootstrap changes and warn user.

### B5.5. CI/CD pipeline

```bash
# Check CI provider
ls -d .github/ 2>/dev/null && echo "CI:github"
ls .gitlab-ci.yml .circleci/ bitrise.yml 2>/dev/null
```

If `.github/` exists (or no CI detected — default to GitHub Actions):
Create `.github/workflows/test.yml` with:
- `runs-on: ubuntu-latest`
- Appropriate setup action for the runtime (setup-node, setup-ruby, setup-python, etc.)
- The same test command verified in B5
- Trigger: push + pull_request

If non-GitHub CI detected → skip CI generation with note: "Detected {provider} — CI pipeline generation supports GitHub Actions only. Add test step to your existing pipeline manually."

### B6. Create TESTING.md

First check: If TESTING.md already exists → read it and update/append rather than overwriting. Never destroy existing content.

Write TESTING.md with:
- Philosophy: "100% test coverage is the key to great vibe coding. Tests let you move fast, trust your instincts, and ship with confidence — without them, vibe coding is just yolo coding. With tests, it's a superpower."
- Framework name and version
- How to run tests (the verified command from B5)
- Test layers: Unit tests (what, where, when), Integration tests, Smoke tests, E2E tests
- Conventions: file naming, assertion style, setup/teardown patterns

### B7. Update CLAUDE.md

First check: If CLAUDE.md already has a `## Testing` section → skip. Don't duplicate.

Append a `## Testing` section:
- Run command and test directory
- Reference to TESTING.md
- Test expectations:
  - 100% test coverage is the goal — tests make vibe coding safe
  - When writing new functions, write a corresponding test
  - When fixing a bug, write a regression test
  - When adding error handling, write a test that triggers the error
  - When adding a conditional (if/else, switch), write tests for BOTH paths
  - Never commit code that makes existing tests fail

### B8. Commit

```bash
git status --porcelain
```

Only commit if there are changes. Stage all bootstrap files (config, test directory, TESTING.md, CLAUDE.md, .github/workflows/test.yml if created):
`git commit -m "chore: bootstrap test framework ({framework name})"`

---

---

## Step 3: 테스트 실행 (병합된 코드에서)

**`RAILS_ENV=test bin/rails db:migrate`를 실행하지 마십시오** — `bin/test-lane`이 내부적으로 이미 `db:test:prepare`를 호출하며, 이것이 스키마를 올바른 레인 데이터베이스에 로드합니다.
INSTANCE 없이 베어 테스트 마이그레이션을 실행하면 고아 DB에 접근하여 structure.sql을 손상시킵니다.

두 테스트 스위트를 병렬로 실행합니다:

```bash
bin/test-lane 2>&1 | tee /tmp/ship_tests.txt &
npm run test 2>&1 | tee /tmp/ship_vitest.txt &
wait
```

둘 다 완료되면 출력 파일을 읽고 통과/실패를 확인합니다.

**테스트가 실패하면:** 즉시 중단하지 마십시오. 테스트 실패 소유권 분류를 적용합니다:

## Test Failure Ownership Triage

When tests fail, do NOT immediately stop. First, determine ownership:

### Step T1: Classify each failure

For each failing test:

1. **Get the files changed on this branch:**
   ```bash
   git diff origin/<base>...HEAD --name-only
   ```

2. **Classify the failure:**
   - **In-branch** if: the failing test file itself was modified on this branch, OR the test output references code that was changed on this branch, OR you can trace the failure to a change in the branch diff.
   - **Likely pre-existing** if: neither the test file nor the code it tests was modified on this branch, AND the failure is unrelated to any branch change you can identify.
   - **When ambiguous, default to in-branch.** It is safer to stop the developer than to let a broken test ship. Only classify as pre-existing when you are confident.

   This classification is heuristic — use your judgment reading the diff and the test output. You do not have a programmatic dependency graph.

### Step T2: Handle in-branch failures

**STOP.** These are your failures. Show them and do not proceed. The developer must fix their own broken tests before shipping.

### Step T3: Handle pre-existing failures

Check `REPO_MODE` from the preamble output.

**If REPO_MODE is `solo`:**

Use AskUserQuestion:

> These test failures appear pre-existing (not caused by your branch changes):
>
> [list each failure with file:line and brief error description]
>
> Since this is a solo repo, you're the only one who will fix these.
>
> RECOMMENDATION: Choose A — fix now while the context is fresh. Completeness: 9/10.
> A) Investigate and fix now (human: ~2-4h / CC: ~15min) — Completeness: 10/10
> B) Add as P0 TODO — fix after this branch lands — Completeness: 7/10
> C) Skip — I know about this, ship anyway — Completeness: 3/10

**If REPO_MODE is `collaborative` or `unknown`:**

Use AskUserQuestion:

> These test failures appear pre-existing (not caused by your branch changes):
>
> [list each failure with file:line and brief error description]
>
> This is a collaborative repo — these may be someone else's responsibility.
>
> RECOMMENDATION: Choose B — assign it to whoever broke it so the right person fixes it. Completeness: 9/10.
> A) Investigate and fix now anyway — Completeness: 10/10
> B) Blame + assign GitHub issue to the author — Completeness: 9/10
> C) Add as P0 TODO — Completeness: 7/10
> D) Skip — ship anyway — Completeness: 3/10

### Step T4: Execute the chosen action

**If "Investigate and fix now":**
- Switch to /investigate mindset: root cause first, then minimal fix.
- Fix the pre-existing failure.
- Commit the fix separately from the branch's changes: `git commit -m "fix: pre-existing test failure in <test-file>"`
- Continue with the workflow.

**If "Add as P0 TODO":**
- If `TODOS.md` exists, add the entry following the format in `review/TODOS-format.md` (or `.claude/skills/review/TODOS-format.md`).
- If `TODOS.md` does not exist, create it with the standard header and add the entry.
- Entry should include: title, the error output, which branch it was noticed on, and priority P0.
- Continue with the workflow — treat the pre-existing failure as non-blocking.

**If "Blame + assign GitHub issue" (collaborative only):**
- Find who likely broke it. Check BOTH the test file AND the production code it tests:
  ```bash
  # Who last touched the failing test?
  git log --format="%an (%ae)" -1 -- <failing-test-file>
  # Who last touched the production code the test covers? (often the actual breaker)
  git log --format="%an (%ae)" -1 -- <source-file-under-test>
  ```
  If these are different people, prefer the production code author — they likely introduced the regression.
- Create an issue assigned to that person (use the platform detected in Step 0):
  - **If GitHub:**
    ```bash
    gh issue create \
      --title "Pre-existing test failure: <test-name>" \
      --body "Found failing on branch <current-branch>. Failure is pre-existing.\n\n**Error:**\n```\n<first 10 lines>\n```\n\n**Last modified by:** <author>\n**Noticed by:** gstack /ship on <date>" \
      --assignee "<github-username>"
    ```
  - **If GitLab:**
    ```bash
    glab issue create \
      -t "Pre-existing test failure: <test-name>" \
      -d "Found failing on branch <current-branch>. Failure is pre-existing.\n\n**Error:**\n```\n<first 10 lines>\n```\n\n**Last modified by:** <author>\n**Noticed by:** gstack /ship on <date>" \
      -a "<gitlab-username>"
    ```
- If neither CLI is available or `--assignee`/`-a` fails (user not in org, etc.), create the issue without assignee and note who should look at it in the body.
- Continue with the workflow.

**If "Skip":**
- Continue with the workflow.
- Note in output: "Pre-existing test failure skipped: <test-name>"

**분류 후:** 인브랜치 실패가 수정되지 않고 남아 있으면 **중단**합니다. 진행하지 마십시오. 모든 실패가 기존 것이고 처리된 경우(수정, TODO 등록, 할당 또는 건너뜀) Step 3.25로 계속합니다.

**모두 통과하면:** 조용히 계속합니다 — 카운트만 간략히 메모합니다.

---

## Step 3.25: Eval 스위트 (조건부)

프롬프트 관련 파일이 변경된 경우 Eval은 필수입니다. diff에 프롬프트 파일이 없으면 이 단계를 완전히 건너뜁니다.

**1. diff가 프롬프트 관련 파일을 수정하는지 확인합니다:**

```bash
git diff origin/<base> --name-only
```

다음 패턴과 매칭합니다 (CLAUDE.md에서):
- `app/services/*_prompt_builder.rb`
- `app/services/*_generation_service.rb`, `*_writer_service.rb`, `*_designer_service.rb`
- `app/services/*_evaluator.rb`, `*_scorer.rb`, `*_classifier_service.rb`, `*_analyzer.rb`
- `app/services/concerns/*voice*.rb`, `*writing*.rb`, `*prompt*.rb`, `*token*.rb`
- `app/services/chat_tools/*.rb`, `app/services/x_thread_tools/*.rb`
- `config/system_prompts/*.txt`
- `test/evals/**/*` (eval 인프라 변경은 모든 스위트에 영향)

**매칭 없음:** "No prompt-related files changed — skipping evals."을 출력하고 Step 3.5로 계속합니다.

**2. 영향받는 eval 스위트를 식별합니다:**

각 eval 러너(`test/evals/*_eval_runner.rb`)는 영향을 주는 소스 파일을 나열하는 `PROMPT_SOURCE_FILES`를 선언합니다. 이를 grep하여 변경된 파일과 매칭되는 스위트를 찾습니다:

```bash
grep -l "changed_file_basename" test/evals/*_eval_runner.rb
```

러너 → 테스트 파일 매핑: `post_generation_eval_runner.rb` → `post_generation_eval_test.rb`.

**특수 사례:**
- `test/evals/judges/*.rb`, `test/evals/support/*.rb`, 또는 `test/evals/fixtures/` 변경은 해당 judge/support 파일을 사용하는 모든 스위트에 영향을 줍니다. eval 테스트 파일의 import를 확인하여 어떤 것이 영향받는지 판단합니다.
- `config/system_prompts/*.txt` 변경 — eval 러너에서 프롬프트 파일명을 grep하여 영향받는 스위트를 찾습니다.
- 어떤 스위트가 영향받는지 불확실하면 영향받을 가능성이 있는 모든 스위트를 실행합니다. 과도한 테스트가 회귀를 놓치는 것보다 낫습니다.

**3. 영향받는 스위트를 `EVAL_JUDGE_TIER=full`로 실행합니다:**

`/ship`은 사전 병합 게이트이므로 항상 full 티어를 사용합니다 (Sonnet 구조적 + Opus 페르소나 judge).

```bash
EVAL_JUDGE_TIER=full EVAL_VERBOSE=1 bin/test-lane --eval test/evals/<suite>_eval_test.rb 2>&1 | tee /tmp/ship_evals.txt
```

여러 스위트를 실행해야 하면 순차적으로 실행합니다 (각각 테스트 레인이 필요). 첫 번째 스위트가 실패하면 즉시 중단합니다 — 나머지 스위트에 API 비용을 낭비하지 마십시오.

**4. 결과 확인:**

- **eval이 실패하면:** 실패와 비용 대시보드를 표시하고 **중단**합니다. 진행하지 마십시오.
- **모두 통과하면:** 통과 카운트와 비용을 기록합니다. Step 3.5로 계속합니다.

**5. eval 출력 저장** — eval 결과와 비용 대시보드를 PR 본문에 포함합니다 (Step 8).

**티어 참조 (참고용 — /ship은 항상 `full` 사용):**
| 티어 | 사용 시점 | 속도 (캐시됨) | 비용 |
|------|------|----------------|------|
| `fast` (Haiku) | 개발 반복, 스모크 테스트 | ~5s (14배 빠름) | ~$0.07/실행 |
| `standard` (Sonnet) | 기본 개발, `bin/test-lane --eval` | ~17s (4배 빠름) | ~$0.37/실행 |
| `full` (Opus 페르소나) | **`/ship` 및 사전 병합** | ~72s (기준선) | ~$1.27/실행 |

---

## Step 3.4: 테스트 커버리지 감사

100% coverage is the goal — every untested path is a path where bugs hide and vibe coding becomes yolo coding. Evaluate what was ACTUALLY coded (from the diff), not what was planned.

### Test Framework Detection

Before analyzing coverage, detect the project's test framework:

1. **Read CLAUDE.md** — look for a `## Testing` section with test command and framework name. If found, use that as the authoritative source.
2. **If CLAUDE.md has no testing section, auto-detect:**

```bash
# Detect project runtime
[ -f Gemfile ] && echo "RUNTIME:ruby"
[ -f package.json ] && echo "RUNTIME:node"
[ -f requirements.txt ] || [ -f pyproject.toml ] && echo "RUNTIME:python"
[ -f go.mod ] && echo "RUNTIME:go"
[ -f Cargo.toml ] && echo "RUNTIME:rust"
# Check for existing test infrastructure
ls jest.config.* vitest.config.* playwright.config.* cypress.config.* .rspec pytest.ini phpunit.xml 2>/dev/null
ls -d test/ tests/ spec/ __tests__/ cypress/ e2e/ 2>/dev/null
```

3. **If no framework detected:** falls through to the Test Framework Bootstrap step (Step 2.5) which handles full setup.

**0. Before/after test count:**

```bash
# Count test files before any generation
find . -name '*.test.*' -o -name '*.spec.*' -o -name '*_test.*' -o -name '*_spec.*' | grep -v node_modules | wc -l
```

Store this number for the PR body.

**1. Trace every codepath changed** using `git diff origin/<base>...HEAD`:

Read every changed file. For each one, trace how data flows through the code — don't just list functions, actually follow the execution:

1. **Read the diff.** For each changed file, read the full file (not just the diff hunk) to understand context.
2. **Trace data flow.** Starting from each entry point (route handler, exported function, event listener, component render), follow the data through every branch:
   - Where does input come from? (request params, props, database, API call)
   - What transforms it? (validation, mapping, computation)
   - Where does it go? (database write, API response, rendered output, side effect)
   - What can go wrong at each step? (null/undefined, invalid input, network failure, empty collection)
3. **Diagram the execution.** For each changed file, draw an ASCII diagram showing:
   - Every function/method that was added or modified
   - Every conditional branch (if/else, switch, ternary, guard clause, early return)
   - Every error path (try/catch, rescue, error boundary, fallback)
   - Every call to another function (trace into it — does IT have untested branches?)
   - Every edge: what happens with null input? Empty array? Invalid type?

This is the critical step — you're building a map of every line of code that can execute differently based on input. Every branch in this diagram needs a test.

**2. Map user flows, interactions, and error states:**

Code coverage isn't enough — you need to cover how real users interact with the changed code. For each changed feature, think through:

- **User flows:** What sequence of actions does a user take that touches this code? Map the full journey (e.g., "user clicks 'Pay' → form validates → API call → success/failure screen"). Each step in the journey needs a test.
- **Interaction edge cases:** What happens when the user does something unexpected?
  - Double-click/rapid resubmit
  - Navigate away mid-operation (back button, close tab, click another link)
  - Submit with stale data (page sat open for 30 minutes, session expired)
  - Slow connection (API takes 10 seconds — what does the user see?)
  - Concurrent actions (two tabs, same form)
- **Error states the user can see:** For every error the code handles, what does the user actually experience?
  - Is there a clear error message or a silent failure?
  - Can the user recover (retry, go back, fix input) or are they stuck?
  - What happens with no network? With a 500 from the API? With invalid data from the server?
- **Empty/zero/boundary states:** What does the UI show with zero results? With 10,000 results? With a single character input? With maximum-length input?

Add these to your diagram alongside the code branches. A user flow with no test is just as much a gap as an untested if/else.

**3. Check each branch against existing tests:**

Go through your diagram branch by branch — both code paths AND user flows. For each one, search for a test that exercises it:
- Function `processPayment()` → look for `billing.test.ts`, `billing.spec.ts`, `test/billing_test.rb`
- An if/else → look for tests covering BOTH the true AND false path
- An error handler → look for a test that triggers that specific error condition
- A call to `helperFn()` that has its own branches → those branches need tests too
- A user flow → look for an integration or E2E test that walks through the journey
- An interaction edge case → look for a test that simulates the unexpected action

Quality scoring rubric:
- ★★★  Tests behavior with edge cases AND error paths
- ★★   Tests correct behavior, happy path only
- ★    Smoke test / existence check / trivial assertion (e.g., "it renders", "it doesn't throw")

### E2E Test Decision Matrix

When checking each branch, also determine whether a unit test or E2E/integration test is the right tool:

**RECOMMEND E2E (mark as [→E2E] in the diagram):**
- Common user flow spanning 3+ components/services (e.g., signup → verify email → first login)
- Integration point where mocking hides real failures (e.g., API → queue → worker → DB)
- Auth/payment/data-destruction flows — too important to trust unit tests alone

**RECOMMEND EVAL (mark as [→EVAL] in the diagram):**
- Critical LLM call that needs a quality eval (e.g., prompt change → test output still meets quality bar)
- Changes to prompt templates, system instructions, or tool definitions

**STICK WITH UNIT TESTS:**
- Pure function with clear inputs/outputs
- Internal helper with no side effects
- Edge case of a single function (null input, empty array)
- Obscure/rare flow that isn't customer-facing

### REGRESSION RULE (mandatory)

**IRON RULE:** When the coverage audit identifies a REGRESSION — code that previously worked but the diff broke — a regression test is written immediately. No AskUserQuestion. No skipping. Regressions are the highest-priority test because they prove something broke.

A regression is when:
- The diff modifies existing behavior (not new code)
- The existing test suite (if any) doesn't cover the changed path
- The change introduces a new failure mode for existing callers

When uncertain whether a change is a regression, err on the side of writing the test.

Format: commit as `test: regression test for {what broke}`

**4. Output ASCII coverage diagram:**

Include BOTH code paths and user flows in the same diagram. Mark E2E-worthy and eval-worthy paths:

```
CODE PATH COVERAGE
===========================
[+] src/services/billing.ts
    │
    ├── processPayment()
    │   ├── [★★★ TESTED] Happy path + card declined + timeout — billing.test.ts:42
    │   ├── [GAP]         Network timeout — NO TEST
    │   └── [GAP]         Invalid currency — NO TEST
    │
    └── refundPayment()
        ├── [★★  TESTED] Full refund — billing.test.ts:89
        └── [★   TESTED] Partial refund (checks non-throw only) — billing.test.ts:101

USER FLOW COVERAGE
===========================
[+] Payment checkout flow
    │
    ├── [★★★ TESTED] Complete purchase — checkout.e2e.ts:15
    ├── [GAP] [→E2E] Double-click submit — needs E2E, not just unit
    ├── [GAP]         Navigate away during payment — unit test sufficient
    └── [★   TESTED]  Form validation errors (checks render only) — checkout.test.ts:40

[+] Error states
    │
    ├── [★★  TESTED] Card declined message — billing.test.ts:58
    ├── [GAP]         Network timeout UX (what does user see?) — NO TEST
    └── [GAP]         Empty cart submission — NO TEST

[+] LLM integration
    │
    └── [GAP] [→EVAL] Prompt template change — needs eval test

─────────────────────────────────
COVERAGE: 5/13 paths tested (38%)
  Code paths: 3/5 (60%)
  User flows: 2/8 (25%)
QUALITY:  ★★★: 2  ★★: 2  ★: 1
GAPS: 8 paths need tests (2 need E2E, 1 needs eval)
─────────────────────────────────
```

**Fast path:** All paths covered → "Step 3.4: All new code paths have test coverage ✓" Continue.

**5. Generate tests for uncovered paths:**

If test framework detected (or bootstrapped in Step 2.5):
- Prioritize error handlers and edge cases first (happy paths are more likely already tested)
- Read 2-3 existing test files to match conventions exactly
- Generate unit tests. Mock all external dependencies (DB, API, Redis).
- For paths marked [→E2E]: generate integration/E2E tests using the project's E2E framework (Playwright, Cypress, Capybara, etc.)
- For paths marked [→EVAL]: generate eval tests using the project's eval framework, or flag for manual eval if none exists
- Write tests that exercise the specific uncovered path with real assertions
- Run each test. Passes → commit as `test: coverage for {feature}`
- Fails → fix once. Still fails → revert, note gap in diagram.

Caps: 30 code paths max, 20 tests generated max (code + user flow combined), 2-min per-test exploration cap.

If no test framework AND user declined bootstrap → diagram only, no generation. Note: "Test generation skipped — no test framework configured."

**Diff is test-only changes:** Skip Step 3.4 entirely: "No new application code paths to audit."

**6. After-count and coverage summary:**

```bash
# Count test files after generation
find . -name '*.test.*' -o -name '*.spec.*' -o -name '*_test.*' -o -name '*_spec.*' | grep -v node_modules | wc -l
```

For PR body: `Tests: {before} → {after} (+{delta} new)`
Coverage line: `Test Coverage Audit: N new code paths. M covered (X%). K tests generated, J committed.`

**7. Coverage gate:**

Before proceeding, check CLAUDE.md for a `## Test Coverage` section with `Minimum:` and `Target:` fields. If found, use those percentages. Otherwise use defaults: Minimum = 60%, Target = 80%.

Using the coverage percentage from the diagram in substep 4 (the `COVERAGE: X/Y (Z%)` line):

- **>= target:** Pass. "Coverage gate: PASS ({X}%)." Continue.
- **>= minimum, < target:** Use AskUserQuestion:
  - "AI-assessed coverage is {X}%. {N} code paths are untested. Target is {target}%."
  - RECOMMENDATION: Choose A because untested code paths are where production bugs hide.
  - Options:
    A) Generate more tests for remaining gaps (recommended)
    B) Ship anyway — I accept the coverage risk
    C) These paths don't need tests — mark as intentionally uncovered
  - If A: Loop back to substep 5 (generate tests) targeting the remaining gaps. After second pass, if still below target, present AskUserQuestion again with updated numbers. Maximum 2 generation passes total.
  - If B: Continue. Include in PR body: "Coverage gate: {X}% — user accepted risk."
  - If C: Continue. Include in PR body: "Coverage gate: {X}% — {N} paths intentionally uncovered."

- **< minimum:** Use AskUserQuestion:
  - "AI-assessed coverage is critically low ({X}%). {N} of {M} code paths have no tests. Minimum threshold is {minimum}%."
  - RECOMMENDATION: Choose A because less than {minimum}% means more code is untested than tested.
  - Options:
    A) Generate tests for remaining gaps (recommended)
    B) Override — ship with low coverage (I understand the risk)
  - If A: Loop back to substep 5. Maximum 2 passes. If still below minimum after 2 passes, present the override choice again.
  - If B: Continue. Include in PR body: "Coverage gate: OVERRIDDEN at {X}%."

**Coverage percentage undetermined:** If the coverage diagram doesn't produce a clear numeric percentage (ambiguous output, parse error), **skip the gate** with: "Coverage gate: could not determine percentage — skipping." Do not default to 0% or block.

**Test-only diffs:** Skip the gate (same as the existing fast-path).

**100% coverage:** "Coverage gate: PASS (100%)." Continue.

### Test Plan Artifact

After producing the coverage diagram, write a test plan artifact so `/qa` and `/qa-only` can consume it:

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" && mkdir -p ~/.gstack/projects/$SLUG
USER=$(whoami)
DATETIME=$(date +%Y%m%d-%H%M%S)
```

Write to `~/.gstack/projects/{slug}/{user}-{branch}-ship-test-plan-{datetime}.md`:

```markdown
# Test Plan
Generated by /ship on {date}
Branch: {branch}
Repo: {owner/repo}

## Affected Pages/Routes
- {URL path} — {what to test and why}

## Key Interactions to Verify
- {interaction description} on {page}

## Edge Cases
- {edge case} on {page}

## Critical Paths
- {end-to-end flow that must work}
```

---

## Step 3.45: 플랜 완료 감사

### Plan File Discovery

1. **Conversation context (primary):** Check if there is an active plan file in this conversation — Claude Code system messages include plan file paths when in plan mode. Look for references like `~/.claude/plans/*.md` in system messages. If found, use it directly — this is the most reliable signal.

2. **Content-based search (fallback):** If no plan file is referenced in conversation context, search by content:

```bash
BRANCH=$(git branch --show-current 2>/dev/null | tr '/' '-')
REPO=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)")
# Try branch name match first (most specific)
PLAN=$(ls -t ~/.claude/plans/*.md 2>/dev/null | xargs grep -l "$BRANCH" 2>/dev/null | head -1)
# Fall back to repo name match
[ -z "$PLAN" ] && PLAN=$(ls -t ~/.claude/plans/*.md 2>/dev/null | xargs grep -l "$REPO" 2>/dev/null | head -1)
# Last resort: most recent plan modified in the last 24 hours
[ -z "$PLAN" ] && PLAN=$(find ~/.claude/plans -name '*.md' -mmin -1440 -maxdepth 1 2>/dev/null | xargs ls -t 2>/dev/null | head -1)
[ -n "$PLAN" ] && echo "PLAN_FILE: $PLAN" || echo "NO_PLAN_FILE"
```

3. **Validation:** If a plan file was found via content-based search (not conversation context), read the first 20 lines and verify it is relevant to the current branch's work. If it appears to be from a different project or feature, treat as "no plan file found."

**Error handling:**
- No plan file found → skip with "No plan file detected — skipping."
- Plan file found but unreadable (permissions, encoding) → skip with "Plan file found but unreadable — skipping."

### Actionable Item Extraction

Read the plan file. Extract every actionable item — anything that describes work to be done. Look for:

- **Checkbox items:** `- [ ] ...` or `- [x] ...`
- **Numbered steps** under implementation headings: "1. Create ...", "2. Add ...", "3. Modify ..."
- **Imperative statements:** "Add X to Y", "Create a Z service", "Modify the W controller"
- **File-level specifications:** "New file: path/to/file.ts", "Modify path/to/existing.rb"
- **Test requirements:** "Test that X", "Add test for Y", "Verify Z"
- **Data model changes:** "Add column X to table Y", "Create migration for Z"

**Ignore:**
- Context/Background sections (`## Context`, `## Background`, `## Problem`)
- Questions and open items (marked with ?, "TBD", "TODO: decide")
- Review report sections (`## GSTACK REVIEW REPORT`)
- Explicitly deferred items ("Future:", "Out of scope:", "NOT in scope:", "P2:", "P3:", "P4:")
- CEO Review Decisions sections (these record choices, not work items)

**Cap:** Extract at most 50 items. If the plan has more, note: "Showing top 50 of N plan items — full list in plan file."

**No items found:** If the plan contains no extractable actionable items, skip with: "Plan file contains no actionable items — skipping completion audit."

For each item, note:
- The item text (verbatim or concise summary)
- Its category: CODE | TEST | MIGRATION | CONFIG | DOCS

### Cross-Reference Against Diff

Run `git diff origin/<base>...HEAD` and `git log origin/<base>..HEAD --oneline` to understand what was implemented.

For each extracted plan item, check the diff and classify:

- **DONE** — Clear evidence in the diff that this item was implemented. Cite the specific file(s) changed.
- **PARTIAL** — Some work toward this item exists in the diff but it's incomplete (e.g., model created but controller missing, function exists but edge cases not handled).
- **NOT DONE** — No evidence in the diff that this item was addressed.
- **CHANGED** — The item was implemented using a different approach than the plan described, but the same goal is achieved. Note the difference.

**Be conservative with DONE** — require clear evidence in the diff. A file being touched is not enough; the specific functionality described must be present.
**Be generous with CHANGED** — if the goal is met by different means, that counts as addressed.

### Output Format

```
PLAN COMPLETION AUDIT
═══════════════════════════════
Plan: {plan file path}

## Implementation Items
  [DONE]      Create UserService — src/services/user_service.rb (+142 lines)
  [PARTIAL]   Add validation — model validates but missing controller checks
  [NOT DONE]  Add caching layer — no cache-related changes in diff
  [CHANGED]   "Redis queue" → implemented with Sidekiq instead

## Test Items
  [DONE]      Unit tests for UserService — test/services/user_service_test.rb
  [NOT DONE]  E2E test for signup flow

## Migration Items
  [DONE]      Create users table — db/migrate/20240315_create_users.rb

─────────────────────────────────
COMPLETION: 4/7 DONE, 1 PARTIAL, 1 NOT DONE, 1 CHANGED
─────────────────────────────────
```

### Gate Logic

After producing the completion checklist:

- **All DONE or CHANGED:** Pass. "Plan completion: PASS — all items addressed." Continue.
- **Only PARTIAL items (no NOT DONE):** Continue with a note in the PR body. Not blocking.
- **Any NOT DONE items:** Use AskUserQuestion:
  - Show the completion checklist above
  - "{N} items from the plan are NOT DONE. These were part of the original plan but are missing from the implementation."
  - RECOMMENDATION: depends on item count and severity. If 1-2 minor items (docs, config), recommend B. If core functionality is missing, recommend A.
  - Options:
    A) Stop — implement the missing items before shipping
    B) Ship anyway — defer these to a follow-up (will create P1 TODOs in Step 5.5)
    C) These items were intentionally dropped — remove from scope
  - If A: STOP. List the missing items for the user to implement.
  - If B: Continue. For each NOT DONE item, create a P1 TODO in Step 5.5 with "Deferred from plan: {plan file path}".
  - If C: Continue. Note in PR body: "Plan items intentionally dropped: {list}."

**No plan file found:** Skip entirely. "No plan file detected — skipping plan completion audit."

**Include in PR body (Step 8):** Add a `## Plan Completion` section with the checklist summary.

---

## Step 3.47: Plan Verification

Automatically verify the plan's testing/verification steps using the `/qa-only` skill.

### 1. Check for verification section

Using the plan file already discovered in Step 3.45, look for a verification section. Match any of these headings: `## Verification`, `## Test plan`, `## Testing`, `## How to test`, `## Manual testing`, or any section with verification-flavored items (URLs to visit, things to check visually, interactions to test).

**If no verification section found:** Skip with "No verification steps found in plan — skipping auto-verification."
**If no plan file was found in Step 3.45:** Skip (already handled).

### 2. Check for running dev server

Before invoking browse-based verification, check if a dev server is reachable:

```bash
curl -s -o /dev/null -w '%{http_code}' http://localhost:3000 2>/dev/null || \
curl -s -o /dev/null -w '%{http_code}' http://localhost:8080 2>/dev/null || \
curl -s -o /dev/null -w '%{http_code}' http://localhost:5173 2>/dev/null || \
curl -s -o /dev/null -w '%{http_code}' http://localhost:4000 2>/dev/null || echo "NO_SERVER"
```

**If NO_SERVER:** Skip with "No dev server detected — skipping plan verification. Run /qa separately after deploying."

### 3. Invoke /qa-only inline

Read the `/qa-only` skill from disk:

```bash
cat ${CLAUDE_SKILL_DIR}/../qa-only/SKILL.md
```

**If unreadable:** Skip with "Could not load /qa-only — skipping plan verification."

Follow the /qa-only workflow with these modifications:
- **Skip the preamble** (already handled by /ship)
- **Use the plan's verification section as the primary test input** — treat each verification item as a test case
- **Use the detected dev server URL** as the base URL
- **Skip the fix loop** — this is report-only verification during /ship
- **Cap at the verification items from the plan** — do not expand into general site QA

### 4. Gate logic

- **All verification items PASS:** Continue silently. "Plan verification: PASS."
- **Any FAIL:** Use AskUserQuestion:
  - Show the failures with screenshot evidence
  - RECOMMENDATION: Choose A if failures indicate broken functionality. Choose B if cosmetic only.
  - Options:
    A) Fix the failures before shipping (recommended for functional issues)
    B) Ship anyway — known issues (acceptable for cosmetic issues)
- **No verification section / no server / unreadable skill:** Skip (non-blocking).

### 5. Include in PR body

Add a `## Verification Results` section to the PR body (Step 8):
- If verification ran: summary of results (N PASS, M FAIL, K SKIPPED)
- If skipped: reason for skipping (no plan, no server, no verification section)

---

## Step 3.5: 사전 착륙 검증(Pre-Landing Review)

테스트가 잡지 못하는 구조적 이슈를 diff에서 리뷰합니다.

1. `.claude/skills/review/checklist.md`를 읽습니다. 파일을 읽을 수 없으면 **중단**하고 오류를 보고합니다.

2. `git diff origin/<base>`를 실행하여 전체 diff를 가져옵니다 (새로 페치한 베이스 브랜치 대비 피처 변경사항 범위).

3. 리뷰 체크리스트를 두 패스로 적용합니다:
   - **패스 1 (CRITICAL):** SQL 및 데이터 안전성, LLM 출력 신뢰 경계
   - **패스 2 (INFORMATIONAL):** 나머지 모든 카테고리

## Design Review (conditional, diff-scoped)

Check if the diff touches frontend files using `gstack-diff-scope`:

```bash
source <(~/.claude/skills/gstack/bin/gstack-diff-scope <base> 2>/dev/null)
```

**If `SCOPE_FRONTEND=false`:** Skip design review silently. No output.

**If `SCOPE_FRONTEND=true`:**

1. **Check for DESIGN.md.** If `DESIGN.md` or `design-system.md` exists in the repo root, read it. All design findings are calibrated against it — patterns blessed in DESIGN.md are not flagged. If not found, use universal design principles.

2. **Read `.claude/skills/review/design-checklist.md`.** If the file cannot be read, skip design review with a note: "Design checklist not found — skipping design review."

3. **Read each changed frontend file** (full file, not just diff hunks). Frontend files are identified by the patterns listed in the checklist.

4. **Apply the design checklist** against the changed files. For each item:
   - **[HIGH] mechanical CSS fix** (`outline: none`, `!important`, `font-size < 16px`): classify as AUTO-FIX
   - **[HIGH/MEDIUM] design judgment needed**: classify as ASK
   - **[LOW] intent-based detection**: present as "Possible — verify visually or run /design-review"

5. **Include findings** in the review output under a "Design Review" header, following the output format in the checklist. Design findings merge with code review findings into the same Fix-First flow.

6. **Log the result** for the Review Readiness Dashboard:

```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"design-review-lite","timestamp":"TIMESTAMP","status":"STATUS","findings":N,"auto_fixed":M,"commit":"COMMIT"}'
```

Substitute: TIMESTAMP = ISO 8601 datetime, STATUS = "clean" if 0 findings or "issues_found", N = total findings, M = auto-fixed count, COMMIT = output of `git rev-parse --short HEAD`.

7. **Codex design voice** (optional, automatic if available):

```bash
which codex 2>/dev/null && echo "CODEX_AVAILABLE" || echo "CODEX_NOT_AVAILABLE"
```

If Codex is available, run a lightweight design check on the diff:

```bash
TMPERR_DRL=$(mktemp /tmp/codex-drl-XXXXXXXX)
codex exec "Review the git diff on this branch. Run 7 litmus checks (YES/NO each): 1. Brand/product unmistakable in first screen? 2. One strong visual anchor present? 3. Page understandable by scanning headlines only? 4. Each section has one job? 5. Are cards actually necessary? 6. Does motion improve hierarchy or atmosphere? 7. Would design feel premium with all decorative shadows removed? Flag any hard rejections: 1. Generic SaaS card grid as first impression 2. Beautiful image with weak brand 3. Strong headline with no clear action 4. Busy imagery behind text 5. Sections repeating same mood statement 6. Carousel with no narrative purpose 7. App UI made of stacked cards instead of layout 5 most important design findings only. Reference file:line." -C "$(git rev-parse --show-toplevel)" -s read-only -c 'model_reasoning_effort="high"' --enable web_search_cached 2>"$TMPERR_DRL"
```

Use a 5-minute timeout (`timeout: 300000`). After the command completes, read stderr:
```bash
cat "$TMPERR_DRL" && rm -f "$TMPERR_DRL"
```

**Error handling:** All errors are non-blocking. On auth failure, timeout, or empty response — skip with a brief note and continue.

Present Codex output under a `CODEX (design):` header, merged with the checklist findings above.

   디자인 발견사항을 코드 리뷰 발견사항과 함께 포함합니다. 아래의 Fix-First 흐름을 동일하게 따릅니다.

4. **각 발견사항을 AUTO-FIX 또는 ASK로 분류합니다** — checklist.md의 Fix-First 휴리스틱에 따릅니다. 크리티컬 발견사항은 ASK 쪽으로, 정보성 발견사항은 AUTO-FIX 쪽으로 기울입니다.

5. **모든 AUTO-FIX 항목을 자동 수정합니다.** 각 수정을 적용합니다. 수정당 한 줄 출력:
   `[AUTO-FIXED] [file:line] Problem → what you did`

6. **ASK 항목이 남아 있으면** 하나의 AskUserQuestion으로 제시합니다:
   - 각 항목에 번호, 심각도, 문제, 권장 수정 포함
   - 항목별 옵션: A) 수정  B) 건너뜀
   - 전체 RECOMMENDATION
   - ASK 항목이 3개 이하이면 개별 AskUserQuestion 호출을 사용할 수 있습니다

7. **모든 수정 완료 후 (자동 + 사용자 승인):**
   - 수정이 적용된 경우: 수정된 파일을 이름으로 커밋합니다 (`git add <fixed-files> && git commit -m "fix: pre-landing review fixes"`), 그런 다음 **중단**하고 사용자에게 재테스트를 위해 `/ship`을 다시 실행하라고 알립니다.
   - 수정이 적용되지 않은 경우 (모든 ASK 항목 건너뜀 또는 이슈 없음): Step 4로 계속합니다.

8. 요약 출력: `Pre-Landing Review: N issues — M auto-fixed, K asked (J fixed, L skipped)`

   이슈가 없으면: `Pre-Landing Review: No issues found.`

9. 리뷰 결과를 리뷰 로그에 저장합니다:
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"review","timestamp":"TIMESTAMP","status":"STATUS","issues_found":N,"critical":N,"informational":N,"commit":"'"$(git rev-parse --short HEAD)"'","via":"ship"}'
```
TIMESTAMP(ISO 8601), STATUS(이슈 없으면 "clean", 그 외 "issues_found"), N 값을 위 요약 카운트에서 대입합니다. `via:"ship"`은 독립 실행형 `/review` 실행과 구분합니다.

리뷰 출력을 저장합니다 — Step 8에서 PR 본문에 포함됩니다.

---

## Step 3.75: Greptile 리뷰 코멘트 처리 (PR이 존재하는 경우)

`.claude/skills/review/greptile-triage.md`를 읽고 페치, 필터, 분류, **에스컬레이션 감지** 단계를 따릅니다.

**PR이 존재하지 않거나, `gh`가 실패하거나, API가 오류를 반환하거나, Greptile 코멘트가 없으면:** 이 단계를 조용히 건너뜁니다. Step 4로 계속합니다.

**Greptile 코멘트가 발견되면:**

출력에 Greptile 요약을 포함합니다: `+ N Greptile comments (X valid, Y fixed, Z FP)`

코멘트에 답변하기 전에 greptile-triage.md의 **에스컬레이션 감지** 알고리즘을 실행하여 Tier 1(친절) 또는 Tier 2(단호) 답변 템플릿 중 어느 것을 사용할지 결정합니다.

분류된 각 코멘트에 대해:

**VALID & ACTIONABLE:** AskUserQuestion을 사용합니다:
- 코멘트 (file:line 또는 [top-level] + 본문 요약 + 영구 링크 URL)
- `RECOMMENDATION: Choose A because [한 줄 이유]`
- 옵션: A) 지금 수정, B) 인정하고 그대로 배포, C) 오탐임
- 사용자가 A를 선택하면: 수정을 적용하고, 수정된 파일을 커밋합니다 (`git add <fixed-files> && git commit -m "fix: address Greptile review — <brief description>"`), greptile-triage.md의 **Fix 답변 템플릿**을 사용하여 답변합니다 (인라인 diff + 설명 포함), 프로젝트별 및 전역 greptile-history에 저장합니다 (type: fix).
- 사용자가 C를 선택하면: greptile-triage.md의 **False Positive 답변 템플릿**을 사용하여 답변합니다 (증거 + 재랭크 제안 포함), 프로젝트별 및 전역 greptile-history에 저장합니다 (type: fp).

**VALID BUT ALREADY FIXED:** greptile-triage.md의 **Already Fixed 답변 템플릿**을 사용하여 답변합니다 — AskUserQuestion 불필요:
- 무엇이 수행되었는지와 수정 커밋 SHA를 포함합니다
- 프로젝트별 및 전역 greptile-history에 저장합니다 (type: already-fixed)

**FALSE POSITIVE:** AskUserQuestion을 사용합니다:
- 코멘트와 왜 틀렸다고 생각하는지 보여줍니다 (file:line 또는 [top-level] + 본문 요약 + 영구 링크 URL)
- 옵션:
  - A) Greptile에 오탐 설명 답변 (명확히 틀린 경우 권장)
  - B) 그래도 수정 (사소한 경우)
  - C) 조용히 무시
- 사용자가 A를 선택하면: greptile-triage.md의 **False Positive 답변 템플릿**을 사용하여 답변합니다 (증거 + 재랭크 제안 포함), 프로젝트별 및 전역 greptile-history에 저장합니다 (type: fp)

**SUPPRESSED:** 조용히 건너뜁니다 — 이전 분류에서 알려진 오탐입니다.

**모든 코멘트 해결 후:** 수정이 적용된 경우 Step 3의 테스트가 오래된 것입니다. Step 4로 계속하기 전에 **테스트를 재실행**합니다 (Step 3). 수정이 적용되지 않았으면 Step 4로 계속합니다.

---

## Step 3.8: Adversarial review (auto-scaled)

Adversarial review thoroughness scales automatically based on diff size. No configuration needed.

**Detect diff size and tool availability:**

```bash
DIFF_INS=$(git diff origin/<base> --stat | tail -1 | grep -oE '[0-9]+ insertion' | grep -oE '[0-9]+' || echo "0")
DIFF_DEL=$(git diff origin/<base> --stat | tail -1 | grep -oE '[0-9]+ deletion' | grep -oE '[0-9]+' || echo "0")
DIFF_TOTAL=$((DIFF_INS + DIFF_DEL))
which codex 2>/dev/null && echo "CODEX_AVAILABLE" || echo "CODEX_NOT_AVAILABLE"
# Respect old opt-out
OLD_CFG=$(~/.claude/skills/gstack/bin/gstack-config get codex_reviews 2>/dev/null || true)
echo "DIFF_SIZE: $DIFF_TOTAL"
echo "OLD_CFG: ${OLD_CFG:-not_set}"
```

If `OLD_CFG` is `disabled`: skip this step silently. Continue to the next step.

**User override:** If the user explicitly requested a specific tier (e.g., "run all passes", "paranoid review", "full adversarial", "do all 4 passes", "thorough review"), honor that request regardless of diff size. Jump to the matching tier section.

**Auto-select tier based on diff size:**
- **Small (< 50 lines changed):** Skip adversarial review entirely. Print: "Small diff ($DIFF_TOTAL lines) — adversarial review skipped." Continue to the next step.
- **Medium (50–199 lines changed):** Run Codex adversarial challenge (or Claude adversarial subagent if Codex unavailable). Jump to the "Medium tier" section.
- **Large (200+ lines changed):** Run all remaining passes — Codex structured review + Claude adversarial subagent + Codex adversarial. Jump to the "Large tier" section.

---

### Medium tier (50–199 lines)

Claude's structured review already ran. Now add a **cross-model adversarial challenge**.

**If Codex is available:** run the Codex adversarial challenge. **If Codex is NOT available:** fall back to the Claude adversarial subagent instead.

**Codex adversarial:**

```bash
TMPERR_ADV=$(mktemp /tmp/codex-adv-XXXXXXXX)
codex exec "Review the changes on this branch against the base branch. Run git diff origin/<base> to see the diff. Your job is to find ways this code will fail in production. Think like an attacker and a chaos engineer. Find edge cases, race conditions, security holes, resource leaks, failure modes, and silent data corruption paths. Be adversarial. Be thorough. No compliments — just the problems." -C "$(git rev-parse --show-toplevel)" -s read-only -c 'model_reasoning_effort="xhigh"' --enable web_search_cached 2>"$TMPERR_ADV"
```

Set the Bash tool's `timeout` parameter to `300000` (5 minutes). Do NOT use the `timeout` shell command — it doesn't exist on macOS. After the command completes, read stderr:
```bash
cat "$TMPERR_ADV"
```

Present the full output verbatim. This is informational — it never blocks shipping.

**Error handling:** All errors are non-blocking — adversarial review is a quality enhancement, not a prerequisite.
- **Auth failure:** If stderr contains "auth", "login", "unauthorized", or "API key": "Codex authentication failed. Run \`codex login\` to authenticate."
- **Timeout:** "Codex timed out after 5 minutes."
- **Empty response:** "Codex returned no response. Stderr: <paste relevant error>."

On any Codex error, fall back to the Claude adversarial subagent automatically.

**Claude adversarial subagent** (fallback when Codex unavailable or errored):

Dispatch via the Agent tool. The subagent has fresh context — no checklist bias from the structured review. This genuine independence catches things the primary reviewer is blind to.

Subagent prompt:
"Read the diff for this branch with `git diff origin/<base>`. Think like an attacker and a chaos engineer. Your job is to find ways this code will fail in production. Look for: edge cases, race conditions, security holes, resource leaks, failure modes, silent data corruption, logic errors that produce wrong results silently, error handling that swallows failures, and trust boundary violations. Be adversarial. Be thorough. No compliments — just the problems. For each finding, classify as FIXABLE (you know how to fix it) or INVESTIGATE (needs human judgment)."

Present findings under an `ADVERSARIAL REVIEW (Claude subagent):` header. **FIXABLE findings** flow into the same Fix-First pipeline as the structured review. **INVESTIGATE findings** are presented as informational.

If the subagent fails or times out: "Claude adversarial subagent unavailable. Continuing without adversarial review."

**Persist the review result:**
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"adversarial-review","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","status":"STATUS","source":"SOURCE","tier":"medium","commit":"'"$(git rev-parse --short HEAD)"'"}'
```
Substitute STATUS: "clean" if no findings, "issues_found" if findings exist. SOURCE: "codex" if Codex ran, "claude" if subagent ran. If both failed, do NOT persist.

**Cleanup:** Run `rm -f "$TMPERR_ADV"` after processing (if Codex was used).

---

### Large tier (200+ lines)

Claude's structured review already ran. Now run **all three remaining passes** for maximum coverage:

**1. Codex structured review (if available):**
```bash
TMPERR=$(mktemp /tmp/codex-review-XXXXXXXX)
codex review --base <base> -c 'model_reasoning_effort="xhigh"' --enable web_search_cached 2>"$TMPERR"
```

Set the Bash tool's `timeout` parameter to `300000` (5 minutes). Do NOT use the `timeout` shell command — it doesn't exist on macOS. Present output under `CODEX SAYS (code review):` header.
Check for `[P1]` markers: found → `GATE: FAIL`, not found → `GATE: PASS`.

If GATE is FAIL, use AskUserQuestion:
```
Codex found N critical issues in the diff.

A) Investigate and fix now (recommended)
B) Continue — review will still complete
```

If A: address the findings. After fixing, re-run tests (Step 3) since code has changed. Re-run `codex review` to verify.

Read stderr for errors (same error handling as medium tier).

After stderr: `rm -f "$TMPERR"`

**2. Claude adversarial subagent:** Dispatch a subagent with the adversarial prompt (same prompt as medium tier). This always runs regardless of Codex availability.

**3. Codex adversarial challenge (if available):** Run `codex exec` with the adversarial prompt (same as medium tier).

If Codex is not available for steps 1 and 3, note to the user: "Codex CLI not found — large-diff review ran Claude structured + Claude adversarial (2 of 4 passes). Install Codex for full 4-pass coverage: `npm install -g @openai/codex`"

**Persist the review result AFTER all passes complete** (not after each sub-step):
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"adversarial-review","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","status":"STATUS","source":"SOURCE","tier":"large","gate":"GATE","commit":"'"$(git rev-parse --short HEAD)"'"}'
```
Substitute: STATUS = "clean" if no findings across ALL passes, "issues_found" if any pass found issues. SOURCE = "both" if Codex ran, "claude" if only Claude subagent ran. GATE = the Codex structured review gate result ("pass"/"fail"), or "informational" if Codex was unavailable. If all passes failed, do NOT persist.

---

### Cross-model synthesis (medium and large tiers)

After all passes complete, synthesize findings across all sources:

```
ADVERSARIAL REVIEW SYNTHESIS (auto: TIER, N lines):
════════════════════════════════════════════════════════════
  High confidence (found by multiple sources): [findings agreed on by >1 pass]
  Unique to Claude structured review: [from earlier step]
  Unique to Claude adversarial: [from subagent, if ran]
  Unique to Codex: [from codex adversarial or code review, if ran]
  Models used: Claude structured ✓  Claude adversarial ✓/✗  Codex ✓/✗
════════════════════════════════════════════════════════════
```

High-confidence findings (agreed on by multiple sources) should be prioritized for fixes.

---

## Step 4: 버전 범프 (자동 결정)

1. 현재 `VERSION` 파일을 읽습니다 (4자리 형식: `MAJOR.MINOR.PATCH.MICRO`)

2. **diff를 기반으로 범프 수준을 자동 결정합니다:**
   - 변경된 줄 수를 셉니다 (`git diff origin/<base>...HEAD --stat | tail -1`)
   - **MICRO** (4번째 자릿수): 50줄 미만 변경, 사소한 조정, 오타, 설정
   - **PATCH** (3번째 자릿수): 50줄 이상 변경, 버그 수정, 소-중 규모 기능
   - **MINOR** (2번째 자릿수): **사용자에게 물어봄** — 주요 기능 또는 중대한 아키텍처 변경일 때만
   - **MAJOR** (1번째 자릿수): **사용자에게 물어봄** — 마일스톤 또는 호환성 깨지는 변경일 때만

3. 새 버전을 계산합니다:
   - 자릿수를 범프하면 오른쪽의 모든 자릿수를 0으로 초기화합니다
   - 예시: `0.19.1.0` + PATCH → `0.19.2.0`

4. 새 버전을 `VERSION` 파일에 기록합니다.

---

## Step 5: CHANGELOG (자동 생성)

1. `CHANGELOG.md` 헤더를 읽어 형식을 파악합니다.

2. **브랜치의 모든 커밋**에서 항목을 자동 생성합니다 (최근 것만이 아니라):
   - `git log <base>..HEAD --oneline`으로 배포되는 모든 커밋을 확인합니다
   - `git diff <base>...HEAD`로 베이스 브랜치 대비 전체 diff를 확인합니다
   - CHANGELOG 항목은 PR에 포함되는 모든 변경사항을 포괄해야 합니다
   - 브랜치의 기존 CHANGELOG 항목이 일부 커밋을 이미 다루고 있으면, 새 버전에 대한 하나의 통합 항목으로 교체합니다
   - 변경사항을 해당하는 섹션으로 분류합니다:
     - `### Added` — 새 기능
     - `### Changed` — 기존 기능 변경
     - `### Fixed` — 버그 수정
     - `### Removed` — 제거된 기능
   - 간결하고 설명적인 글머리 기호를 작성합니다
   - 파일 헤더 뒤(5번째 줄)에 오늘 날짜로 삽입합니다
   - 형식: `## [X.Y.Z.W] - YYYY-MM-DD`

**사용자에게 변경사항 설명을 요청하지 마십시오.** diff와 커밋 히스토리에서 추론합니다.

---

## Step 5.5: TODOS.md (자동 업데이트)

프로젝트의 TODOS.md를 배포되는 변경사항과 교차 참조합니다. 완료된 항목은 자동으로 표시합니다; 파일이 없거나 정리되지 않은 경우에만 프롬프트합니다.

`.claude/skills/review/TODOS-format.md`를 읽어 표준 형식 참조를 확인합니다.

**1. TODOS.md가 존재하는지 확인합니다** — 저장소 루트에서.

**TODOS.md가 존재하지 않으면:** AskUserQuestion을 사용합니다:
- 메시지: "GStack recommends maintaining a TODOS.md organized by skill/component, then priority (P0 at top through P4, then Completed at bottom). See TODOS-format.md for the full format. Would you like to create one?"
- 옵션: A) 지금 생성, B) 나중에
- A인 경우: 스켈레톤으로 `TODOS.md`를 생성합니다 (# TODOS 제목 + ## Completed 섹션). step 3으로 계속합니다.
- B인 경우: Step 5.5의 나머지를 건너뜁니다. Step 6으로 계속합니다.

**2. 구조와 정리 상태를 확인합니다:**

TODOS.md를 읽고 권장 구조를 따르는지 확인합니다:
- `## <Skill/Component>` 제목 아래 항목 그룹화
- 각 항목에 P0-P4 값의 `**Priority:**` 필드
- 하단에 `## Completed` 섹션

**정리되지 않은 경우** (우선순위 필드 누락, 컴포넌트 그룹화 없음, Completed 섹션 없음): AskUserQuestion을 사용합니다:
- 메시지: "TODOS.md doesn't follow the recommended structure (skill/component groupings, P0-P4 priority, Completed section). Would you like to reorganize it?"
- 옵션: A) 지금 재정리 (권장), B) 그대로 유지
- A인 경우: TODOS-format.md에 따라 제자리에서 재정리합니다. 모든 내용을 보존합니다 — 구조만 변경하고, 항목을 절대 삭제하지 않습니다.
- B인 경우: 재정리 없이 step 3으로 계속합니다.

**3. 완료된 TODO를 감지합니다:**

이 단계는 완전 자동입니다 — 사용자 상호작용 없음.

이전 단계에서 이미 수집한 diff와 커밋 히스토리를 사용합니다:
- `git diff <base>...HEAD` (베이스 브랜치 대비 전체 diff)
- `git log <base>..HEAD --oneline` (배포되는 모든 커밋)

각 TODO 항목에 대해 이 PR의 변경사항이 완료하는지 확인합니다:
- 커밋 메시지와 TODO 제목 및 설명 매칭
- TODO에서 참조된 파일이 diff에 나타나는지 확인
- TODO에 설명된 작업이 기능적 변경사항과 일치하는지 확인

**보수적으로 판단합니다:** diff에 명확한 증거가 있을 때만 TODO를 완료로 표시합니다. 불확실하면 그대로 둡니다.

**4. 완료된 항목을 이동합니다** — 하단의 `## Completed` 섹션으로. 추가: `**Completed:** vX.Y.Z (YYYY-MM-DD)`

**5. 요약 출력:**
- `TODOS.md: N items marked complete (item1, item2, ...). M items remaining.`
- 또는: `TODOS.md: No completed items detected. M items remaining.`
- 또는: `TODOS.md: Created.` / `TODOS.md: Reorganized.`

**6. 방어적 처리:** TODOS.md를 기록할 수 없으면 (권한 오류, 디스크 풀) 사용자에게 경고하고 계속합니다. TODOS 실패로 ship 워크플로우를 절대 중단하지 않습니다.

이 요약을 저장합니다 — Step 8에서 PR 본문에 포함됩니다.

---

## Step 6: 커밋 (이등분 가능(bisectable) 청크)

**목표:** `git bisect`와 잘 작동하고 LLM이 변경사항을 이해하는 데 도움이 되는 작고 논리적인 커밋을 생성합니다.

1. diff를 분석하고 변경사항을 논리적 커밋으로 그룹화합니다. 각 커밋은 **하나의 일관된 변경** — 하나의 파일이 아니라 하나의 논리적 단위를 나타내야 합니다.

2. **커밋 순서** (앞선 커밋이 먼저):
   - **인프라:** 마이그레이션, 설정 변경, 라우트 추가
   - **모델 & 서비스:** 새 모델, 서비스, concern (테스트 포함)
   - **컨트롤러 & 뷰:** 컨트롤러, 뷰, JS/React 컴포넌트 (테스트 포함)
   - **VERSION + CHANGELOG + TODOS.md:** 항상 마지막 커밋

3. **분할 규칙:**
   - 모델과 해당 테스트 파일은 같은 커밋
   - 서비스와 해당 테스트 파일은 같은 커밋
   - 컨트롤러, 해당 뷰, 해당 테스트는 같은 커밋
   - 마이그레이션은 별도 커밋 (또는 지원하는 모델과 함께 그룹화)
   - 설정/라우트 변경은 활성화하는 기능과 함께 그룹화 가능
   - 전체 diff가 작으면 (4개 미만 파일에 50줄 미만) 단일 커밋으로 충분

4. **각 커밋은 독립적으로 유효해야 합니다** — 깨진 import 없음, 아직 존재하지 않는 코드 참조 없음. 의존성이 먼저 오도록 커밋 순서를 정합니다.

5. 각 커밋 메시지를 작성합니다:
   - 첫 줄: `<type>: <summary>` (type = feat/fix/chore/refactor/docs)
   - 본문: 이 커밋에 포함된 내용의 간략한 설명
   - **마지막 커밋**(VERSION + CHANGELOG)에만 버전 태그와 공동 저자 트레일러를 포함합니다:

```bash
git commit -m "$(cat <<'EOF'
chore: bump version and changelog (vX.Y.Z.W)

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Step 6.5: 검증 게이트

**철칙: 새로운 검증 증거 없이 절대 완료를 주장하지 마십시오.**

푸시 전에 Step 4-6에서 코드가 변경되었는지 재검증합니다:

1. **테스트 검증:** Step 3의 테스트 실행 후 코드가 변경된 경우 (리뷰 발견사항 수정, CHANGELOG 편집은 해당 없음), 테스트 스위트를 재실행합니다. 새로운 출력을 붙여넣습니다. Step 3의 오래된 출력은 허용되지 않습니다.

2. **빌드 검증:** 프로젝트에 빌드 단계가 있으면 실행합니다. 출력을 붙여넣습니다.

3. **합리화 방지:**
   - "이제 작동할 것이다" → 실행하십시오.
   - "확신한다" → 확신은 증거가 아닙니다.
   - "이미 앞서 테스트했다" → 그 이후로 코드가 변경되었습니다. 다시 테스트하십시오.
   - "사소한 변경이다" → 사소한 변경이 프로덕션을 깨뜨립니다.

**여기서 테스트가 실패하면:** 중단합니다. 푸시하지 마십시오. 이슈를 수정하고 Step 3으로 돌아갑니다.

검증 없이 작업 완료를 주장하는 것은 효율이 아니라 부정직입니다.

---

## Step 7: 푸시

업스트림 추적과 함께 리모트에 푸시합니다:

```bash
git push -u origin <branch-name>
```

---

## Step 8: PR/MR 생성

Step 0에서 감지된 플랫폼을 사용하여 풀 리퀘스트(GitHub) 또는 머지 리퀘스트(GitLab)를 생성합니다.

PR/MR 본문에 다음 섹션을 포함해야 합니다:

```
## Summary
<CHANGELOG의 글머리 기호>

## Test Coverage
<Step 3.4의 커버리지 다이어그램, 또는 "All new code paths have test coverage.">
<Step 3.4가 실행된 경우: "Tests: {before} → {after} (+{delta} new)">

## Pre-Landing Review
<Step 3.5 코드 리뷰의 발견사항, 또는 "No issues found.">

## Design Review
<디자인 리뷰가 실행된 경우: "Design Review (lite): N findings — M auto-fixed, K skipped. AI Slop: clean/N issues.">
<프론트엔드 파일 변경 없음: "No frontend files changed — design review skipped.">

## Eval Results
<eval이 실행된 경우: 스위트 이름, 통과/실패 카운트, 비용 대시보드 요약. 건너뛴 경우: "No prompt-related files changed — evals skipped.">

## Greptile Review
<Greptile 코멘트가 발견된 경우: [FIXED] / [FALSE POSITIVE] / [ALREADY FIXED] 태그 + 코멘트당 한 줄 요약의 글머리 기호 목록>
<Greptile 코멘트 없음: "No Greptile comments.">
<Step 3.75에서 PR이 존재하지 않았으면: 이 섹션 전체 생략>

## Plan Completion
<플랜 파일 발견됨: Step 3.45의 완료 체크리스트 요약>
<플랜 파일 없음: "No plan file detected.">
<플랜 항목 보류: 보류된 항목 나열>

## Verification Results
<검증이 실행된 경우: Step 3.47의 요약 (N PASS, M FAIL, K SKIPPED)>
<건너뛴 경우: 이유 (플랜 없음, 서버 없음, 검증 섹션 없음)>
<해당 없음: 이 섹션 생략>

## TODOS
<항목 완료됨: 버전과 함께 완료된 항목의 글머리 기호 목록>
<완료된 항목 없음: "No TODO items completed in this PR.">
<TODOS.md 생성 또는 재정리됨: 해당 내용 메모>
<TODOS.md가 존재하지 않고 사용자가 건너뜀: 이 섹션 생략>

## Test plan
- [x] All Rails tests pass (N runs, 0 failures)
- [x] All Vitest tests pass (N tests)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

**GitHub인 경우:**

```bash
gh pr create --base <base> --title "<type>: <summary>" --body "$(cat <<'EOF'
<위의 PR 본문>
EOF
)"
```

**GitLab인 경우:**

```bash
glab mr create -b <base> -t "<type>: <summary>" -d "$(cat <<'EOF'
<위의 MR 본문>
EOF
)"
```

**두 CLI 모두 사용 불가능한 경우:**
브랜치 이름, 리모트 URL을 출력하고 사용자에게 웹 UI를 통해 수동으로 PR/MR을 생성하도록 안내합니다. 중단하지 않습니다 — 코드는 푸시되어 준비된 상태입니다.

**PR/MR URL을 출력합니다** — 그런 다음 Step 8.5로 진행합니다.

---

## Step 8.5: /document-release 자동 호출

PR이 생성된 후 프로젝트 문서를 자동으로 동기화합니다. `document-release/SKILL.md` 스킬 파일(이 스킬의 디렉토리와 인접)을 읽고 전체 워크플로우를 실행합니다:

1. `/document-release` 스킬을 읽습니다: `cat ${CLAUDE_SKILL_DIR}/../document-release/SKILL.md`
2. 지시사항을 따릅니다 — 프로젝트의 모든 .md 파일을 읽고, diff와 교차 참조하여, 변경된 것들을 업데이트합니다 (README, ARCHITECTURE, CONTRIBUTING, CLAUDE.md, TODOS 등)
3. 문서가 업데이트되면 변경사항을 커밋하고 같은 브랜치에 푸시합니다:
   ```bash
   git add -A && git commit -m "docs: sync documentation with shipped changes" && git push
   ```
4. 업데이트가 필요한 문서가 없으면 "Documentation is current — no updates needed."라고 말합니다.

이 단계는 자동입니다. 사용자에게 확인을 요청하지 마십시오. 목표는 마찰 없는 문서 업데이트입니다 — 사용자가 `/ship`을 실행하면 별도의 명령 없이 문서가 최신 상태를 유지합니다.

---

## Step 8.75: Ship 메트릭 저장

커버리지와 플랜 완료 데이터를 로그에 기록하여 `/retro`가 추세를 추적할 수 있도록 합니다:

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" && mkdir -p ~/.gstack/projects/$SLUG
```

`~/.gstack/projects/$SLUG/$BRANCH-reviews.jsonl`에 추가합니다:

```bash
echo '{"skill":"ship","timestamp":"'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'","coverage_pct":COVERAGE_PCT,"plan_items_total":PLAN_TOTAL,"plan_items_done":PLAN_DONE,"verification_result":"VERIFY_RESULT","version":"VERSION","branch":"BRANCH"}' >> ~/.gstack/projects/$SLUG/$BRANCH-reviews.jsonl
```

이전 단계에서 대입합니다:
- **COVERAGE_PCT**: Step 3.4 다이어그램의 커버리지 퍼센티지 (정수, 판단 불가 시 -1)
- **PLAN_TOTAL**: Step 3.45에서 추출한 전체 플랜 항목 (플랜 파일 없으면 0)
- **PLAN_DONE**: Step 3.45의 DONE + CHANGED 항목 수 (플랜 파일 없으면 0)
- **VERIFY_RESULT**: Step 3.47의 "pass", "fail", 또는 "skipped"
- **VERSION**: VERSION 파일에서
- **BRANCH**: 현재 브랜치 이름

이 단계는 자동입니다 — 절대 건너뛰지 말고, 절대 확인을 요청하지 마십시오.

---

## 중요 규칙

- **절대 테스트를 건너뛰지 마십시오.** 테스트가 실패하면 중단합니다.
- **greptile-triage.md의 Greptile 답변 템플릿을 사용합니다.** 모든 답변에 증거(인라인 diff, 코드 참조, 재랭크 제안)를 포함합니다. 모호한 답변은 절대 게시하지 마십시오.
- **새로운 검증 증거 없이 절대 push하지 마십시오.** Step 3 테스트 후 코드가 변경되었으면 push 전에 재실행합니다.
- **Step 3.4는 커버리지 테스트를 생성합니다.** 커밋 전에 통과해야 합니다. 실패하는 테스트를 절대 커밋하지 마십시오.
- **목표: 사용자가 `/ship`이라고 하면, 다음으로 보는 것은 리뷰 + PR URL + 자동 동기화된 문서입니다.**
