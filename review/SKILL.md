---
name: review
preamble-tier: 4
version: 1.0.0
description: |
  사전 착륙(pre-landing) PR 리뷰. base 브랜치 대비 diff를 분석하여 SQL 안전성, LLM 신뢰 경계
  위반, 조건부 부작용, 기타 구조적 문제를 검출합니다. "review this PR", "code review",
  "pre-landing review", "check my diff" 등의 요청 시 사용합니다.
  사용자가 코드를 merge하거나 land하려 할 때 선제적으로 제안합니다.
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
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
echo '{"skill":"review","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
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

# 사전 착륙 PR 리뷰(Pre-Landing PR Review)

`/review` 워크플로우를 실행합니다. 현재 브랜치의 diff를 base 브랜치와 비교하여 테스트가 잡아내지 못하는 구조적 문제를 분석합니다.

---

## Step 1: 브랜치 확인

1. `git branch --show-current`를 실행하여 현재 브랜치를 확인합니다.
2. base 브랜치에 있다면 다음을 출력하고 중단합니다: **"리뷰할 내용이 없습니다 — base 브랜치에 있거나 변경사항이 없습니다."**
3. `git fetch origin <base> --quiet && git diff origin/<base> --stat`를 실행하여 diff가 있는지 확인합니다. diff가 없으면 같은 메시지를 출력하고 중단합니다.

---

## Step 1.5: 범위 이탈 감지(Scope Drift Detection)

코드 품질을 리뷰하기 전에 먼저 확인합니다: **요청된 것만 정확히 구현했는가 — 그 이상도 이하도 아닌가?**

1. `TODOS.md`(존재하는 경우)를 읽습니다. PR 설명(`gh pr view --json body --jq .body 2>/dev/null || true`)을 읽습니다.
   커밋 메시지(`git log origin/<base>..HEAD --oneline`)를 읽습니다.
   **PR이 없는 경우:** 커밋 메시지와 TODOS.md를 기반으로 의도를 파악합니다 — /review는 /ship이 PR을 생성하기 전에 실행되므로 이것이 일반적인 케이스입니다.
2. **명시된 의도**를 파악합니다 — 이 브랜치가 달성해야 할 목표는 무엇이었는가?
3. `git diff origin/<base>...HEAD --stat`를 실행하고 변경된 파일을 명시된 의도와 비교합니다.

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

### Integration with Scope Drift Detection

The plan completion results augment the existing Scope Drift Detection. If a plan file is found:

- **NOT DONE items** become additional evidence for **MISSING REQUIREMENTS** in the scope drift report.
- **Items in the diff that don't match any plan item** become evidence for **SCOPE CREEP** detection.

This is **INFORMATIONAL** — does not block the review (consistent with existing scope drift behavior).

Update the scope drift output to include plan file context:

```
Scope Check: [CLEAN / DRIFT DETECTED / REQUIREMENTS MISSING]
Intent: <from plan file — 1-line summary>
Plan: <plan file path>
Delivered: <1-line summary of what the diff actually does>
Plan items: N DONE, M PARTIAL, K NOT DONE
[If NOT DONE: list each missing item]
[If scope creep: list each out-of-scope change not in the plan]
```

**No plan file found:** Fall back to existing scope drift behavior (check TODOS.md and PR description only).

4. 회의적 시각으로 평가합니다 (가능한 경우 플랜 완료 결과를 반영):

   **범위 이탈(SCOPE CREEP) 감지:**
   - 명시된 의도와 무관한 파일 변경
   - 플랜에 언급되지 않은 새 기능이나 리팩토링
   - "이 김에..." 식의 변경으로 영향 범위가 확대된 경우

   **누락 요구사항(MISSING REQUIREMENTS) 감지:**
   - TODOS.md/PR 설명의 요구사항 중 diff에서 다뤄지지 않은 항목
   - 명시된 요구사항에 대한 테스트 커버리지 부족
   - 부분 구현 (시작했으나 완료하지 않은 경우)

5. 출력 (메인 리뷰 시작 전):
   ```
   Scope Check: [CLEAN / DRIFT DETECTED / REQUIREMENTS MISSING]
   Intent: <요청된 내용 1줄 요약>
   Delivered: <diff가 실제로 수행하는 내용 1줄 요약>
   [이탈 시: 범위 외 변경사항 각각 나열]
   [누락 시: 미해결 요구사항 각각 나열]
   ```

6. 이 단계는 **정보 제공용**이며 리뷰를 차단하지 않습니다. Step 2로 진행합니다.

---

## Step 2: 체크리스트 읽기

`.claude/skills/review/checklist.md`를 읽습니다.

**파일을 읽을 수 없으면 즉시 중단하고 오류를 보고합니다.** 체크리스트 없이 진행하지 마십시오.

---

## Step 2.5: Greptile 리뷰 코멘트 확인

`.claude/skills/review/greptile-triage.md`를 읽고 fetch, filter, classify 및 **에스컬레이션 감지(escalation detection)** 단계를 따릅니다.

**PR이 없거나, `gh`가 실패하거나, API가 오류를 반환하거나, Greptile 코멘트가 없는 경우:** 이 단계를 조용히 건너뜁니다. Greptile 통합은 부가적 기능이며 — 없어도 리뷰는 정상 동작합니다.

**Greptile 코멘트가 발견된 경우:** 분류 결과(VALID & ACTIONABLE, VALID BUT ALREADY FIXED, FALSE POSITIVE, SUPPRESSED)를 저장합니다 — Step 5에서 필요합니다.

---

## Step 3: diff 가져오기

오래된 로컬 상태로 인한 오탐을 방지하기 위해 최신 base 브랜치를 fetch합니다:

```bash
git fetch origin <base> --quiet
```

`git diff origin/<base>`를 실행하여 전체 diff를 가져옵니다. 여기에는 최신 base 브랜치 대비 커밋된 변경사항과 커밋되지 않은 변경사항이 모두 포함됩니다.

---

## Step 4: 2단계 리뷰(Two-pass review)

체크리스트를 diff에 대해 2단계로 적용합니다:

1. **Pass 1 (CRITICAL):** SQL & 데이터 안전성, 경쟁 조건(Race Conditions) & 동시성(Concurrency), LLM 출력 신뢰 경계(Trust Boundary), Enum & 값 완전성(Value Completeness)
2. **Pass 2 (INFORMATIONAL):** 조건부 부작용(Conditional Side Effects), 매직 넘버 & 문자열 결합(String Coupling), 죽은 코드 & 일관성, LLM 프롬프트 문제, 테스트 갭, 뷰/프론트엔드, 성능 & 번들 영향

**Enum & 값 완전성은 diff 외부의 코드를 읽어야 합니다.** diff가 새로운 enum 값, 상태, 티어 또는 타입 상수를 도입하면, Grep을 사용하여 형제 값을 참조하는 모든 파일을 찾은 다음 해당 파일을 Read하여 새 값이 처리되는지 확인합니다. 이것이 diff 내 리뷰만으로는 불충분한 유일한 카테고리입니다.

**추천 전 검색(Search-before-recommending):** 수정 패턴을 추천할 때 (특히 동시성, 캐싱, 인증 또는 프레임워크별 동작의 경우):
- 해당 패턴이 사용 중인 프레임워크 버전의 현재 모범 사례인지 확인합니다
- 우회 방법을 추천하기 전에 최신 버전에 내장 솔루션이 있는지 확인합니다
- 현재 문서 대비 API 시그니처를 검증합니다 (API는 버전 간 변경됩니다)

몇 초면 되지만, 구식 패턴 추천을 방지합니다. WebSearch를 사용할 수 없는 경우 해당 사실을 언급하고 기존 지식으로 진행합니다.

체크리스트에 지정된 출력 형식을 따릅니다. 억제 항목(suppressions)을 준수합니다 — "DO NOT flag" 섹션에 나열된 항목은 지적하지 마십시오.

---

## Step 4.5: 디자인 리뷰 (조건부)

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

디자인 관련 발견사항을 Step 4의 발견사항과 함께 포함합니다. 이들은 Step 5의 동일한 Fix-First 흐름을 따릅니다 — 기계적 CSS 수정은 AUTO-FIX, 나머지는 ASK입니다.

---

## Step 4.75: 테스트 커버리지 다이어그램

100% coverage is the goal. Evaluate every codepath changed in the diff and identify test gaps. Gaps become INFORMATIONAL findings that follow the Fix-First flow.

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

3. **If no framework detected:** still produce the coverage diagram, but skip test generation.

**Step 1. Trace every codepath changed** using `git diff origin/<base>...HEAD`:

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

**Step 2. Map user flows, interactions, and error states:**

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

**Step 3. Check each branch against existing tests:**

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

**Step 4. Output ASCII coverage diagram:**

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

**Fast path:** All paths covered → "Step 4.75: All new code paths have test coverage ✓" Continue.

**Step 5. Generate tests for gaps (Fix-First):**

If test framework is detected and gaps were identified:
- Classify each gap as AUTO-FIX or ASK per the Fix-First Heuristic:
  - **AUTO-FIX:** Simple unit tests for pure functions, edge cases of existing tested functions
  - **ASK:** E2E tests, tests requiring new test infrastructure, tests for ambiguous behavior
- For AUTO-FIX gaps: generate the test, run it, commit as `test: coverage for {feature}`
- For ASK gaps: include in the Fix-First batch question with the other review findings
- For paths marked [→E2E]: always ASK (E2E tests are higher-effort and need user confirmation)
- For paths marked [→EVAL]: always ASK (eval tests need user confirmation on quality criteria)

If no test framework detected → include gaps as INFORMATIONAL findings only, no generation.

**Diff is test-only changes:** Skip Step 4.75 entirely: "No new application code paths to audit."

### Coverage Warning

After producing the coverage diagram, check the coverage percentage. Read CLAUDE.md for a `## Test Coverage` section with a `Minimum:` field. If not found, use default: 60%.

If coverage is below the minimum threshold, output a prominent warning **before** the regular review findings:

```
⚠️ COVERAGE WARNING: AI-assessed coverage is {X}%. {N} code paths untested.
Consider writing tests before running /ship.
```

This is INFORMATIONAL — does not block /review. But it makes low coverage visible early so the developer can address it before reaching the /ship coverage gate.

If coverage percentage cannot be determined, skip the warning silently.

이 단계는 Pass 2의 "테스트 갭" 카테고리를 대체합니다 — 체크리스트의 테스트 갭 항목과 이 커버리지 다이어그램 간에 발견사항을 중복하지 마십시오. 커버리지 갭은 Step 4 및 Step 4.5의 발견사항과 함께 포함합니다. 이들은 동일한 Fix-First 흐름을 따릅니다 — 갭은 INFORMATIONAL 발견사항입니다.

---

## Step 5: Fix-First 리뷰

**모든 발견사항에 대해 조치를 취합니다 — 중요한 것만이 아닙니다.**

요약 헤더를 출력합니다: `Pre-Landing Review: N issues (X critical, Y informational)`

### Step 5a: 각 발견사항 분류

각 발견사항에 대해 checklist.md의 Fix-First 휴리스틱에 따라 AUTO-FIX 또는 ASK로 분류합니다. 중요(Critical) 발견사항은 ASK 쪽으로, 정보성(Informational) 발견사항은 AUTO-FIX 쪽으로 기울입니다.

### Step 5b: 모든 AUTO-FIX 항목 자동 수정

각 수정을 직접 적용합니다. 각 항목에 대해 한 줄 요약을 출력합니다:
`[AUTO-FIXED] [file:line] 문제 → 수행한 조치`

### Step 5c: ASK 항목 일괄 질문

ASK 항목이 남아있으면 하나의 AskUserQuestion으로 일괄 제시합니다:

- 각 항목에 번호, 심각도 라벨, 문제, 권장 수정안을 함께 나열합니다
- 각 항목에 옵션을 제공합니다: A) 권장대로 수정, B) 건너뛰기
- 전체 RECOMMENDATION을 포함합니다

예시 형식:
```
5개 이슈를 자동 수정했습니다. 2개에 대해 판단이 필요합니다:

1. [CRITICAL] app/models/post.rb:42 — 상태 전환에서 경쟁 조건(Race condition)
   수정: UPDATE에 `WHERE status = 'draft'` 추가
   → A) 수정  B) 건너뛰기

2. [INFORMATIONAL] app/services/generator.rb:88 — LLM 출력이 DB 쓰기 전에 타입 검사되지 않음
   수정: JSON 스키마 유효성 검증 추가
   → A) 수정  B) 건너뛰기

RECOMMENDATION: 둘 다 수정 — #1은 실제 경쟁 조건이고, #2는 무음 데이터 손상을 방지합니다.
```

ASK 항목이 3개 이하이면 일괄 처리 대신 개별 AskUserQuestion 호출을 사용할 수 있습니다.

### Step 5d: 사용자 승인 수정 적용

사용자가 "수정"을 선택한 항목에 대해 수정을 적용합니다. 수정된 내용을 출력합니다.

ASK 항목이 없으면 (모두 AUTO-FIX인 경우) 질문을 완전히 건너뜁니다.

### 주장의 검증

최종 리뷰 출력을 생성하기 전에:
- "이 패턴은 안전하다"고 주장하는 경우 → 안전성을 증명하는 구체적인 줄을 인용합니다
- "다른 곳에서 처리된다"고 주장하는 경우 → 처리 코드를 읽고 인용합니다
- "테스트가 이를 커버한다"고 주장하는 경우 → 테스트 파일명과 메서드명을 명시합니다
- "아마 처리되어 있을 것" 또는 "아마 테스트되어 있을 것"이라고 절대 말하지 마십시오 — 검증하거나 미확인으로 표시합니다

**합리화 방지:** "괜찮아 보입니다"는 발견사항이 아닙니다. 괜찮다는 증거를 인용하거나, 미검증으로 표시합니다.

### Greptile 코멘트 해결

자체 발견사항을 출력한 후, Step 2.5에서 Greptile 코멘트가 분류된 경우:

**출력 헤더에 Greptile 요약을 포함합니다:** `+ N Greptile comments (X valid, Y fixed, Z FP)`

코멘트에 답변하기 전에 greptile-triage.md의 **에스컬레이션 감지(Escalation Detection)** 알고리즘을 실행하여 Tier 1 (우호적) 또는 Tier 2 (단호한) 답변 템플릿 중 어떤 것을 사용할지 결정합니다.

1. **VALID & ACTIONABLE 코멘트:** 이들은 발견사항에 포함됩니다 — Fix-First 흐름을 따릅니다 (기계적이면 자동 수정, 아니면 ASK로 일괄 처리) (A: 지금 수정, B: 확인, C: 오탐). 사용자가 A(수정)를 선택하면 greptile-triage.md의 **Fix reply template**을 사용하여 답변합니다 (인라인 diff + 설명 포함). 사용자가 C(오탐)를 선택하면 **False Positive reply template**을 사용하여 답변하고 (증거 + 재랭크 제안 포함), 프로젝트별 및 글로벌 greptile-history에 저장합니다.

2. **FALSE POSITIVE 코멘트:** 각각을 AskUserQuestion으로 제시합니다:
   - Greptile 코멘트를 표시합니다: file:line (또는 [top-level]) + 본문 요약 + 퍼머링크 URL
   - 왜 오탐인지 간결하게 설명합니다
   - 옵션:
     - A) Greptile에 왜 잘못된 것인지 답변 (명백히 틀린 경우 권장)
     - B) 그래도 수정 (적은 노력이고 무해한 경우)
     - C) 무시 — 답변도 수정도 하지 않음

   사용자가 A를 선택하면 greptile-triage.md의 **False Positive reply template**을 사용하여 답변하고 (증거 + 재랭크 제안 포함), 프로젝트별 및 글로벌 greptile-history에 저장합니다.

3. **VALID BUT ALREADY FIXED 코멘트:** greptile-triage.md의 **Already Fixed reply template**을 사용하여 답변합니다 — AskUserQuestion 불필요:
   - 수행된 작업과 수정 커밋 SHA를 포함합니다
   - 프로젝트별 및 글로벌 greptile-history에 저장합니다

4. **SUPPRESSED 코멘트:** 조용히 건너뜁니다 — 이전 분류에서 알려진 오탐입니다.

---

## Step 5.5: TODOS 교차 참조

저장소 루트의 `TODOS.md`(존재하는 경우)를 읽습니다. PR을 열린 TODO와 교차 참조합니다:

- **이 PR이 열린 TODO를 닫는가?** 그렇다면 출력에 해당 항목을 표시합니다: "This PR addresses TODO: <title>"
- **이 PR이 TODO가 되어야 할 작업을 생성하는가?** 그렇다면 정보성 발견사항으로 표시합니다.
- **이 리뷰에 맥락을 제공하는 관련 TODO가 있는가?** 그렇다면 관련 발견사항을 논의할 때 참조합니다.

TODOS.md가 존재하지 않으면 이 단계를 조용히 건너뜁니다.

---

## Step 5.6: 문서 최신성 검사

diff를 문서 파일과 교차 참조합니다. 저장소 루트의 각 `.md` 파일 (README.md, ARCHITECTURE.md, CONTRIBUTING.md, CLAUDE.md 등)에 대해:

1. diff의 코드 변경이 해당 문서 파일에 설명된 기능, 컴포넌트 또는 워크플로우에 영향을 미치는지 확인합니다.
2. 해당 문서 파일이 이 브랜치에서 업데이트되지 않았지만 설명하는 코드가 변경된 경우, INFORMATIONAL 발견사항으로 표시합니다:
   "문서가 오래되었을 수 있습니다: [file]이 [feature/component]를 설명하지만 이 브랜치에서 코드가 변경되었습니다. `/document-release` 실행을 고려하십시오."

이것은 정보 제공 전용이며 — 절대 critical이 아닙니다. 수정 액션은 `/document-release`입니다.

문서 파일이 없으면 이 단계를 조용히 건너뜁니다.

---

## Step 5.7: Adversarial review (auto-scaled)

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

If A: address the findings. Re-run `codex review` to verify.

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

## Step 5.8: Eng Review 결과 영속화

모든 리뷰 패스가 완료된 후, 최종 `/review` 결과를 영속화하여 `/ship`이 이 브랜치에서 Eng Review가 실행되었음을 인식할 수 있게 합니다.

실행:

```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"review","timestamp":"TIMESTAMP","status":"STATUS","issues_found":N,"critical":N,"informational":N,"commit":"COMMIT"}'
```

치환할 값:
- `TIMESTAMP` = ISO 8601 날짜시간
- `STATUS` = Fix-First 처리 및 적대적 리뷰(adversarial review) 후 미해결 발견사항이 없으면 `"clean"`, 그렇지 않으면 `"issues_found"`
- `issues_found` = 총 미해결 발견사항 수
- `critical` = 미해결 critical 발견사항 수
- `informational` = 미해결 informational 발견사항 수
- `COMMIT` = `git rev-parse --short HEAD`의 출력

실제 리뷰가 완료되기 전에 조기 종료되는 경우 (예: base 브랜치 대비 diff 없음) 이 항목을 기록하지 **마십시오**.

## 중요 규칙

- **전체 diff를 읽은 후 코멘트합니다.** diff에서 이미 해결된 문제를 지적하지 마십시오.
- **읽기 전용이 아닌 Fix-first.** AUTO-FIX 항목은 직접 적용합니다. ASK 항목은 사용자 승인 후에만 적용합니다. 절대 커밋, push 또는 PR 생성을 하지 마십시오 — 그것은 /ship의 역할입니다.
- **간결하게.** 한 줄로 문제, 한 줄로 수정. 서두 없이.
- **실제 문제만 지적합니다.** 괜찮은 것은 건너뜁니다.
- **greptile-triage.md의 Greptile 답변 템플릿을 사용합니다.** 모든 답변에 증거를 포함합니다. 모호한 답변은 절대 게시하지 마십시오.
