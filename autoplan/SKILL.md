---
name: autoplan
preamble-tier: 3
version: 1.0.0
description: |
  자동 리뷰 파이프라인 — CEO, 디자인, 엔지니어링 리뷰 스킬 전체를 디스크에서 읽고
  6가지 의사결정 원칙을 사용하여 자동 결정하며 순차적으로 실행합니다. 최종 승인
  게이트에서 취향 결정(근접한 접근법, 경계선 범위, codex 이견)을 표면화합니다.
  하나의 명령어로 완전히 리뷰된 플랜을 산출합니다.
  "auto review", "autoplan", "run all reviews", "review this plan automatically",
  "make the decisions for me" 요청 시 사용합니다.
  사용자가 플랜 파일을 가지고 있고 15-30개의 중간 질문에 답하지 않고 전체 리뷰를
  실행하고 싶어할 때 적극적으로 제안하세요.
benefits-from: [office-hours]
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - WebSearch
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
_TEL=$(~/.claude/skills/gstack/bin/gstack-config get telemetry 2>/dev/null || true)
_TEL_PROMPTED=$([ -f ~/.gstack/.telemetry-prompted ] && echo "yes" || echo "no")
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
echo "TELEMETRY: ${_TEL:-off}"
echo "TEL_PROMPTED: $_TEL_PROMPTED"
mkdir -p ~/.gstack/analytics
echo '{"skill":"autoplan","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
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

## Prerequisite Skill Offer

When the design doc check above prints "No design doc found," offer the prerequisite
skill before proceeding.

Say to the user via AskUserQuestion:

> "No design doc found for this branch. `/office-hours` produces a structured problem
> statement, premise challenge, and explored alternatives — it gives this review much
> sharper input to work with. Takes about 10 minutes. The design doc is per-feature,
> not per-product — it captures the thinking behind this specific change."

Options:
- A) Run /office-hours now (we'll pick up the review right after)
- B) Skip — proceed with standard review

If they skip: "No worries — standard review. If you ever want sharper input, try
/office-hours first next time." Then proceed normally. Do not re-offer later in the session.

If they choose A:

Say: "Running /office-hours inline. Once the design doc is ready, I'll pick up
the review right where we left off."

Read the office-hours skill file from disk using the Read tool:
`~/.claude/skills/gstack/office-hours/SKILL.md`

Follow it inline, **skipping these sections** (already handled by the parent skill):
- Preamble (run first)
- AskUserQuestion Format
- Completeness Principle — Boil the Lake
- Search Before Building
- Contributor Mode
- Completion Status Protocol
- Telemetry (run last)

If the Read fails (file not found), say:
"Could not load /office-hours — proceeding with standard review."

After /office-hours completes, re-run the design doc check:
```bash
SLUG=$(~/.claude/skills/gstack/browse/bin/remote-slug 2>/dev/null || basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)")
BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null | tr '/' '-' || echo 'no-branch')
DESIGN=$(ls -t ~/.gstack/projects/$SLUG/*-$BRANCH-design-*.md 2>/dev/null | head -1)
[ -z "$DESIGN" ] && DESIGN=$(ls -t ~/.gstack/projects/$SLUG/*-design-*.md 2>/dev/null | head -1)
[ -n "$DESIGN" ] && echo "Design doc found: $DESIGN" || echo "No design doc found"
```

If a design doc is now found, read it and continue the review.
If none was produced (user may have cancelled), proceed with standard review.

# /autoplan — 자동 리뷰 파이프라인

하나의 명령어. 초안 플랜 입력, 완전히 리뷰된 플랜 출력.

/autoplan은 CEO, 디자인, 엔지니어링 리뷰 스킬 파일 전체를 디스크에서 읽고 전체
깊이로 따릅니다 — 각 스킬을 수동으로 실행하는 것과 동일한 엄격함, 동일한 섹션,
동일한 방법론입니다. 유일한 차이점: 중간 AskUserQuestion 호출이 아래 6가지 원칙을
사용하여 자동 결정됩니다. 취향 결정(합리적인 사람들이 의견을 달리할 수 있는 경우)은
최종 승인 게이트에서 표면화됩니다.

---

## 6가지 의사결정 원칙

이 규칙들이 모든 중간 질문에 자동 응답합니다:

1. **완전성 선택** — 전체를 출시합니다. 더 많은 엣지 케이스를 커버하는 접근법을 선택합니다.
2. **끝까지 해결** — 폭발 반경(이 플랜이 수정하는 파일 + 직접 임포터) 내의 모든 것을 수정합니다. 폭발 반경 내이면서 CC 작업량 1일 미만(파일 5개 미만, 새 인프라 없음)인 확장은 자동 승인합니다.
3. **실용적** — 두 옵션이 같은 것을 수정한다면 더 깔끔한 것을 선택합니다. 선택에 5초, 5분이 아닙니다.
4. **DRY** — 기존 기능과 중복? 거부합니다. 기존 것을 재사용합니다.
5. **명시적 > 영리한** — 10줄의 명확한 수정 > 200줄의 추상화. 새 기여자가 30초 안에 읽을 수 있는 것을 선택합니다.
6. **행동 편향** — 머지 > 리뷰 사이클 > 오래된 논의. 우려를 표시하되 차단하지 않습니다.

**충돌 해결 (컨텍스트 의존 타이브레이커):**
- **CEO 단계:** P1 (완전성) + P2 (끝까지 해결)가 우선합니다.
- **Eng 단계:** P5 (명시적) + P3 (실용적)가 우선합니다.
- **Design 단계:** P5 (명시적) + P1 (완전성)가 우선합니다.

---

## 의사결정 분류

모든 자동 결정은 분류됩니다:

**기계적** — 명확히 맞는 답이 하나. 자동 결정하고 조용히 진행합니다.
예: codex 실행 (항상 예), eval 실행 (항상 예), 완성된 플랜의 범위 축소 (항상 아니오).

**취향** — 합리적인 사람들이 의견을 달리할 수 있음. 추천과 함께 자동 결정하되, 최종 게이트에서 표면화합니다. 세 가지 자연적 원인:
1. **근접한 접근법** — 상위 두 개가 모두 다른 트레이드오프로 실현 가능.
2. **경계선 범위** — 폭발 반경 내이지만 3-5개 파일, 또는 모호한 반경.
3. **Codex 이견** — codex가 다르게 추천하며 유효한 논점이 있음.

---

## 순차 실행 — 필수

단계는 반드시 엄격한 순서로 실행해야 합니다: CEO → Design → Eng.
각 단계는 다음 단계가 시작되기 전에 완전히 완료되어야 합니다.
절대 단계를 병렬로 실행하지 마세요 — 각 단계는 이전 단계를 기반으로 합니다.

각 단계 사이에 단계 전환 요약을 출력하고, 다음 단계를 시작하기 전에 이전 단계의
모든 필수 출력이 작성되었는지 확인하세요.

---

## "자동 결정"의 의미

자동 결정은 사용자의 판단을 6가지 원칙으로 대체합니다. 분석을 대체하는 것이
아닙니다. 로드된 스킬 파일의 모든 섹션은 여전히 인터랙티브 버전과 동일한 깊이로
실행되어야 합니다. 변경되는 유일한 것은 AskUserQuestion에 누가 답하는가입니다:
사용자 대신 여러분이 6가지 원칙을 사용하여 답합니다.

**반드시 해야 할 것:**
- 각 섹션이 참조하는 실제 코드, diff, 파일을 읽기
- 섹션이 요구하는 모든 출력물 생성 (다이어그램, 테이블, 레지스트리, 아티팩트)
- 섹션이 포착하도록 설계된 모든 이슈 식별
- 6가지 원칙을 사용하여 각 이슈 결정 (사용자에게 묻는 대신)
- 각 결정을 감사 추적에 기록
- 모든 필수 아티팩트를 디스크에 작성

**해서는 안 될 것:**
- 리뷰 섹션을 테이블의 한 줄 요약으로 압축
- 무엇을 검토했는지 보여주지 않고 "이슈 없음" 작성
- 무엇을 확인했고 왜 해당하지 않는지 명시하지 않고 "적용되지 않음"으로 섹션 건너뛰기
- 필수 출력 대신 요약 생성 (예: 섹션이 요구하는 ASCII 종속성 그래프 대신 "아키텍처 양호")

"이슈 없음"은 섹션의 유효한 출력입니다 — 단, 분석을 수행한 후에만 가능합니다.
무엇을 검토했고 왜 아무것도 플래그되지 않았는지 명시하세요 (최소 1-2문장).
"건너뜀"은 스킵 목록에 없는 섹션에서는 절대 유효하지 않습니다.

---

## 0단계: 접수 + 복원 지점

### 1단계: 복원 지점 캡처

다른 작업 전에, 플랜 파일의 현재 상태를 외부 파일에 저장합니다:

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)" && mkdir -p ~/.gstack/projects/$SLUG
BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null | tr '/' '-')
DATETIME=$(date +%Y%m%d-%H%M%S)
echo "RESTORE_PATH=$HOME/.gstack/projects/$SLUG/${BRANCH}-autoplan-restore-${DATETIME}.md"
```

복원 경로에 플랜 파일의 전체 내용을 다음 헤더와 함께 작성합니다:
```
# /autoplan Restore Point
Captured: [timestamp] | Branch: [branch] | Commit: [short hash]

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan

## Original Plan State
[verbatim plan file contents]
```

그런 다음 플랜 파일에 한 줄 HTML 주석을 맨 앞에 추가합니다:
`<!-- /autoplan restore point: [RESTORE_PATH] -->`

### 2단계: 컨텍스트 읽기

- CLAUDE.md, TODOS.md, git log -30, 베이스 브랜치에 대한 git diff --stat 읽기
- 디자인 문서 탐색: `ls -t ~/.gstack/projects/$SLUG/*-design-*.md 2>/dev/null | head -1`
- UI 범위 감지: 플랜에서 뷰/렌더링 관련 용어 (component, screen, form,
  button, modal, layout, dashboard, sidebar, nav, dialog) grep. 2개 이상 일치 필요.
  오탐 제외 (단독 "page", 약어 내 "UI").

### 3단계: 디스크에서 스킬 파일 로드

Read 도구를 사용하여 각 파일을 읽습니다:
- `~/.claude/skills/gstack/plan-ceo-review/SKILL.md`
- `~/.claude/skills/gstack/plan-design-review/SKILL.md` (UI 범위가 감지된 경우에만)
- `~/.claude/skills/gstack/plan-eng-review/SKILL.md`

**섹션 스킵 목록 — 로드된 스킬 파일을 따를 때, 다음 섹션은 건너뜁니다
(/autoplan이 이미 처리함):**
- Preamble (먼저 실행)
- AskUserQuestion Format
- Completeness Principle — Boil the Lake
- Search Before Building
- Contributor Mode
- Completion Status Protocol
- Telemetry (마지막에 실행)
- Step 0: Detect base branch
- Review Readiness Dashboard
- Plan File Review Report
- Prerequisite Skill Offer (BENEFITS_FROM)
- Outside Voice — Independent Plan Challenge
- Design Outside Voices (parallel)

리뷰별 방법론, 섹션, 필수 출력만 따르세요.

출력: "작업 대상은 다음과 같습니다: [플랜 요약]. UI 범위: [예/아니오].
디스크에서 리뷰 스킬을 로드했습니다. 자동 결정으로 전체 리뷰 파이프라인을 시작합니다."

---

## 1단계: CEO 리뷰 (전략 및 범위)

plan-ceo-review/SKILL.md를 따릅니다 — 모든 섹션, 전체 깊이.
재정의: 모든 AskUserQuestion → 6가지 원칙을 사용하여 자동 결정.

**재정의 규칙:**
- 모드 선택: SELECTIVE EXPANSION
- 전제: 합리적인 것은 수용 (P6), 명확히 잘못된 것만 도전
- **게이트: 전제를 사용자에게 확인 요청** — 자동 결정되지 않는 유일한 AskUserQuestion입니다. 전제는 인간의 판단이 필요합니다.
- 대안: 가장 높은 완전성 선택 (P1). 동점이면 가장 단순한 것 선택 (P5).
  상위 2개가 근접하면 → 취향 결정으로 표시.
- 범위 확장: 폭발 반경 내 + 1일 미만 CC → 승인 (P2). 외부 → TODOS.md로 연기 (P3).
  중복 → 거부 (P4). 경계선 (3-5개 파일) → 취향 결정으로 표시.
- 10개 리뷰 섹션 전체: 완전히 실행, 각 이슈 자동 결정, 모든 결정 기록.
- 이중 음성: Codex가 사용 가능하면 항상 Claude 서브에이전트와 Codex 모두 실행 (P6).
  동시에 실행합니다 (서브에이전트는 Agent 도구, Codex는 Bash).

  **Codex CEO 음성** (Bash 통해):
  명령어: `codex exec "You are a CEO/founder advisor reviewing a development plan.
  Challenge the strategic foundations: Are the premises valid or assumed? Is this the
  right problem to solve, or is there a reframing that would be 10x more impactful?
  What alternatives were dismissed too quickly? What competitive or market risks are
  unaddressed? What scope decisions will look foolish in 6 months? Be adversarial.
  No compliments. Just the strategic blind spots.
  File: <plan_path>" -C "$(git rev-parse --show-toplevel)" -s read-only --enable web_search_cached`
  타임아웃: 10분

  **Claude CEO 서브에이전트** (Agent 도구 통해):
  "Read the plan file at <plan_path>. You are an independent CEO/strategist
  reviewing this plan. You have NOT seen any prior review. Evaluate:
  1. Is this the right problem to solve? Could a reframing yield 10x impact?
  2. Are the premises stated or just assumed? Which ones could be wrong?
  3. What's the 6-month regret scenario — what will look foolish?
  4. What alternatives were dismissed without sufficient analysis?
  5. What's the competitive risk — could someone else solve this first/better?
  For each finding: what's wrong, severity (critical/high/medium), and the fix."

  **오류 처리:** 모두 논블로킹. Codex 인증/타임아웃/빈 응답 → Claude 서브에이전트만으로
  진행, `[single-model]` 태그 부착. Claude 서브에이전트도 실패 →
  "외부 음성 사용 불가 — 기본 리뷰로 계속합니다."

  **성능 저하 매트릭스:** 둘 다 실패 → "single-reviewer mode". Codex만 →
  `[codex-only]` 태그. 서브에이전트만 → `[subagent-only]` 태그.

- 전략적 선택: codex가 유효한 전략적 이유로 전제나 범위 결정에 이의 → 취향 결정.

**필수 실행 체크리스트 (CEO):**

Step 0 (0A-0F) — 각 하위 단계를 실행하고 산출:
- 0A: 특정 전제를 명명하고 평가한 전제 도전
- 0B: 기존 코드 활용 맵 (하위 문제 → 기존 코드)
- 0C: 드림 상태 다이어그램 (현재 → 이 플랜 → 12개월 이상적)
- 0C-bis: 구현 대안 테이블 (2-3개 접근법의 노력/위험/장단점)
- 0D: 모드별 분석과 범위 결정 기록
- 0E: 시간적 심문 (1시간 차 → 6시간 이후)
- 0F: 모드 선택 확인

Step 0.5 (이중 음성): Claude 서브에이전트와 Codex를 동시에 실행합니다. Codex 출력은
CODEX SAYS (CEO — strategy challenge) 헤더 아래에 제시합니다. 서브에이전트 출력은
CLAUDE SUBAGENT (CEO — strategic independence) 헤더 아래에 제시합니다. CEO
합의 테이블을 생성합니다:

```
CEO DUAL VOICES — CONSENSUS TABLE:
═══════════════════════════════════════════════════════════════
  Dimension                           Claude  Codex  Consensus
  ──────────────────────────────────── ─────── ─────── ─────────
  1. Premises valid?                   —       —      —
  2. Right problem to solve?           —       —      —
  3. Scope calibration correct?        —       —      —
  4. Alternatives sufficiently explored?—      —      —
  5. Competitive/market risks covered? —       —      —
  6. 6-month trajectory sound?         —       —      —
═══════════════════════════════════════════════════════════════
CONFIRMED = both agree. DISAGREE = models differ (→ taste decision).
Missing voice = N/A (not CONFIRMED). Single critical finding from one voice = flagged regardless.
```

섹션 1-10 — 각 섹션에 대해, 로드된 스킬 파일의 평가 기준을 실행합니다:
- 발견 사항이 있는 섹션: 전체 분석, 각 이슈 자동 결정, 감사 추적에 기록
- 발견 사항이 없는 섹션: 무엇을 검토했고 왜 아무것도 플래그되지 않았는지 1-2문장으로 명시.
  절대 섹션을 테이블 행의 이름만으로 압축하지 마세요.
- 섹션 11 (Design): 0단계에서 UI 범위가 감지된 경우에만 실행

**1단계 필수 출력물:**
- "범위 외" 섹션 — 연기된 항목과 근거
- "이미 존재하는 것" 섹션 — 하위 문제를 기존 코드에 매핑
- Error & Rescue Registry 테이블 (섹션 2에서)
- Failure Modes Registry 테이블 (리뷰 섹션에서)
- 드림 상태 델타 (이 플랜이 우리를 어디에 두는지 vs 12개월 이상적)
- Completion Summary (CEO 스킬의 전체 요약 테이블)

**1단계 완료.** 단계 전환 요약을 출력합니다:
> **1단계 완료.** Codex: [N개 우려]. Claude 서브에이전트: [N개 이슈].
> 합의: [X/6 확인, Y개 이견 → 게이트에서 표면화].
> 2단계로 전달.

모든 1단계 출력물이 플랜 파일에 작성되고 전제 게이트가 통과될 때까지
2단계를 시작하지 마세요.

---

**2단계 사전 체크리스트 (시작 전 확인):**
- [ ] CEO completion summary가 플랜 파일에 작성됨
- [ ] CEO 이중 음성 실행됨 (Codex + Claude 서브에이전트, 또는 사용 불가 명시)
- [ ] CEO 합의 테이블 생성됨
- [ ] 전제 게이트 통과됨 (사용자 확인)
- [ ] 단계 전환 요약 출력됨

## 2단계: 디자인 리뷰 (조건부 — UI 범위 없으면 건너뜀)

plan-design-review/SKILL.md를 따릅니다 — 7개 차원 전체, 전체 깊이.
재정의: 모든 AskUserQuestion → 6가지 원칙을 사용하여 자동 결정.

**재정의 규칙:**
- 집중 영역: 관련된 모든 차원 (P1)
- 구조적 이슈 (누락된 상태, 깨진 계층): 자동 수정 (P5)
- 미적/취향 이슈: 취향 결정으로 표시
- 디자인 시스템 정렬: DESIGN.md가 존재하고 수정이 명확하면 자동 수정
- 이중 음성: Codex가 사용 가능하면 항상 Claude 서브에이전트와 Codex 모두 실행 (P6).

  **Codex 디자인 음성** (Bash 통해):
  명령어: `codex exec "Read the plan file at <plan_path>. Evaluate this plan's
  UI/UX design decisions.

  Also consider these findings from the CEO review phase:
  <insert CEO dual voice findings summary — key concerns, disagreements>

  Does the information hierarchy serve the user or the developer? Are interaction
  states (loading, empty, error, partial) specified or left to the implementer's
  imagination? Is the responsive strategy intentional or afterthought? Are
  accessibility requirements (keyboard nav, contrast, touch targets) specified or
  aspirational? Does the plan describe specific UI decisions or generic patterns?
  What design decisions will haunt the implementer if left ambiguous?
  Be opinionated. No hedging." -C "$(git rev-parse --show-toplevel)" -s read-only --enable web_search_cached`
  타임아웃: 10분

  **Claude 디자인 서브에이전트** (Agent 도구 통해):
  "Read the plan file at <plan_path>. You are an independent senior product designer
  reviewing this plan. You have NOT seen any prior review. Evaluate:
  1. Information hierarchy: what does the user see first, second, third? Is it right?
  2. Missing states: loading, empty, error, success, partial — which are unspecified?
  3. User journey: what's the emotional arc? Where does it break?
  4. Specificity: does the plan describe SPECIFIC UI or generic patterns?
  5. What design decisions will haunt the implementer if left ambiguous?
  For each finding: what's wrong, severity (critical/high/medium), and the fix."
  이전 단계 컨텍스트 없음 — 서브에이전트는 진정한 독립성을 유지해야 합니다.

  오류 처리: 1단계와 동일 (논블로킹, 성능 저하 매트릭스 적용).

- 디자인 선택: codex가 유효한 UX 근거로 디자인 결정에 이의 → 취향 결정.

**필수 실행 체크리스트 (Design):**

1. Step 0 (디자인 범위): 완전성 0-10점 평가. DESIGN.md 확인. 기존 패턴 매핑.

2. Step 0.5 (이중 음성): Claude 서브에이전트와 Codex를 동시에 실행합니다.
   CODEX SAYS (design — UX challenge)와 CLAUDE SUBAGENT (design — independent review)
   헤더 아래에 제시합니다. 디자인 리트머스 스코어카드 (합의 테이블)를 생성합니다.
   plan-design-review의 리트머스 스코어카드 형식을 사용합니다. CEO 단계 발견 사항은
   Codex 프롬프트에만 포함합니다 (Claude 서브에이전트는 포함하지 않음 — 독립성 유지).

3. Pass 1-7: 로드된 스킬에서 각각 실행합니다. 0-10점 평가. 각 이슈 자동 결정.
   스코어카드의 DISAGREE 항목 → 관련 pass에서 양쪽 관점과 함께 제기.

**2단계 완료.** 단계 전환 요약을 출력합니다:
> **2단계 완료.** Codex: [N개 우려]. Claude 서브에이전트: [N개 이슈].
> 합의: [X/Y 확인, Z개 이견 → 게이트에서 표면화].
> 3단계로 전달.

모든 2단계 출력물이 (실행된 경우) 플랜 파일에 작성될 때까지 3단계를 시작하지 마세요.

---

**3단계 사전 체크리스트 (시작 전 확인):**
- [ ] 위의 1단계 항목 전체 확인
- [ ] 디자인 completion summary 작성됨 (또는 "건너뜀, UI 범위 없음")
- [ ] 디자인 이중 음성 실행됨 (2단계가 실행된 경우)
- [ ] 디자인 합의 테이블 생성됨 (2단계가 실행된 경우)
- [ ] 단계 전환 요약 출력됨

## 3단계: Eng 리뷰 + 이중 음성

plan-eng-review/SKILL.md를 따릅니다 — 모든 섹션, 전체 깊이.
재정의: 모든 AskUserQuestion → 6가지 원칙을 사용하여 자동 결정.

**재정의 규칙:**
- 범위 도전: 절대 축소하지 않음 (P2)
- 이중 음성: Codex가 사용 가능하면 항상 Claude 서브에이전트와 Codex 모두 실행 (P6).

  **Codex eng 음성** (Bash 통해):
  명령어: `codex exec "Review this plan for architectural issues, missing edge cases,
  and hidden complexity. Be adversarial.

  Also consider these findings from prior review phases:
  CEO: <insert CEO consensus table summary — key concerns, DISAGREEs>
  Design: <insert Design consensus table summary, or 'skipped, no UI scope'>

  File: <plan_path>" -C "$(git rev-parse --show-toplevel)" -s read-only --enable web_search_cached`
  타임아웃: 10분

  **Claude eng 서브에이전트** (Agent 도구 통해):
  "Read the plan file at <plan_path>. You are an independent senior engineer
  reviewing this plan. You have NOT seen any prior review. Evaluate:
  1. Architecture: Is the component structure sound? Coupling concerns?
  2. Edge cases: What breaks under 10x load? What's the nil/empty/error path?
  3. Tests: What's missing from the test plan? What would break at 2am Friday?
  4. Security: New attack surface? Auth boundaries? Input validation?
  5. Hidden complexity: What looks simple but isn't?
  For each finding: what's wrong, severity, and the fix."
  이전 단계 컨텍스트 없음 — 서브에이전트는 진정한 독립성을 유지해야 합니다.

  오류 처리: 1단계와 동일 (논블로킹, 성능 저하 매트릭스 적용).

- 아키텍처 선택: 명시적 > 영리한 (P5). codex가 유효한 이유로 이의 → 취향 결정.
- Eval: 항상 관련된 모든 스위트 포함 (P1)
- 테스트 플랜: `~/.gstack/projects/$SLUG/{user}-{branch}-test-plan-{datetime}.md`에 아티팩트 생성
- TODOS.md: 1단계에서 연기된 모든 범위 확장을 수집하여 자동 작성

**필수 실행 체크리스트 (Eng):**

1. Step 0 (범위 도전): 플랜이 참조하는 실제 코드를 읽습니다. 각 하위 문제를
   기존 코드에 매핑합니다. 복잡도 검사를 실행합니다. 구체적인 발견 사항을 산출합니다.

2. Step 0.5 (이중 음성): Claude 서브에이전트와 Codex를 동시에 실행합니다. Codex 출력은
   CODEX SAYS (eng — architecture challenge) 헤더 아래에 제시합니다. 서브에이전트 출력은
   CLAUDE SUBAGENT (eng — independent review) 헤더 아래에 제시합니다. eng 합의
   테이블을 생성합니다:

```
ENG DUAL VOICES — CONSENSUS TABLE:
═══════════════════════════════════════════════════════════════
  Dimension                           Claude  Codex  Consensus
  ──────────────────────────────────── ─────── ─────── ─────────
  1. Architecture sound?               —       —      —
  2. Test coverage sufficient?         —       —      —
  3. Performance risks addressed?      —       —      —
  4. Security threats covered?         —       —      —
  5. Error paths handled?              —       —      —
  6. Deployment risk manageable?       —       —      —
═══════════════════════════════════════════════════════════════
CONFIRMED = both agree. DISAGREE = models differ (→ taste decision).
Missing voice = N/A (not CONFIRMED). Single critical finding from one voice = flagged regardless.
```

3. 섹션 1 (아키텍처): 새 컴포넌트와 기존 컴포넌트의 관계를 보여주는 ASCII 종속성
   그래프를 생성합니다. 결합도, 확장성, 보안을 평가합니다.

4. 섹션 2 (코드 품질): DRY 위반, 네이밍 이슈, 복잡도를 식별합니다.
   특정 파일과 패턴을 참조합니다. 각 발견 사항을 자동 결정합니다.

5. **섹션 3 (테스트 리뷰) — 절대 건너뛰거나 압축하지 마세요.**
   이 섹션은 기억에서 요약하는 것이 아닌 실제 코드를 읽어야 합니다.
   - diff 또는 플랜의 영향받는 파일을 읽기
   - 테스트 다이어그램 구축: 모든 새로운 UX 플로우, 데이터 플로우, 코드 경로, 브랜치 나열
   - 다이어그램의 각 항목에 대해: 어떤 유형의 테스트가 커버하는가? 존재하는가? 갭은?
   - LLM/프롬프트 변경 시: 어떤 eval 스위트를 실행해야 하는가?
   - 테스트 갭 자동 결정의 의미: 갭 식별 → 테스트 추가 또는 연기 결정
     (근거와 원칙 포함) → 결정 기록. 분석을 건너뛰는 것이 아닙니다.
   - 테스트 플랜 아티팩트를 디스크에 작성

6. 섹션 4 (성능): N+1 쿼리, 메모리, 캐싱, 느린 경로를 평가합니다.

**3단계 필수 출력물:**
- "범위 외" 섹션
- "이미 존재하는 것" 섹션
- 아키텍처 ASCII 다이어그램 (섹션 1)
- 코드 경로를 커버리지에 매핑하는 테스트 다이어그램 (섹션 3)
- 디스크에 작성된 테스트 플랜 아티팩트 (섹션 3)
- 치명적 갭 플래그가 있는 Failure modes registry
- Completion Summary (Eng 스킬의 전체 요약)
- TODOS.md 업데이트 (모든 단계에서 수집)

---

## 의사결정 감사 추적

각 자동 결정 후, Edit를 사용하여 플랜 파일에 행을 추가합니다:

```markdown
<!-- AUTONOMOUS DECISION LOG -->
## Decision Audit Trail

| # | Phase | Decision | Principle | Rationale | Rejected |
|---|-------|----------|-----------|-----------|----------|
```

결정마다 한 행씩 점진적으로 작성합니다 (Edit 사용). 이렇게 하면 감사 추적이
대화 컨텍스트가 아닌 디스크에 유지됩니다.

---

## 게이트 전 검증

최종 승인 게이트를 제시하기 전에, 필수 출력물이 실제로 생성되었는지 확인합니다.
플랜 파일과 대화에서 각 항목을 확인합니다.

**1단계 (CEO) 출력물:**
- [ ] 특정 전제를 명명한 전제 도전 ("전제 수용" 이상)
- [ ] 해당하는 모든 리뷰 섹션에 발견 사항 또는 명시적 "X를 검토했으나 플래그 없음"
- [ ] Error & Rescue Registry 테이블 생성됨 (또는 이유와 함께 N/A 명시)
- [ ] Failure Modes Registry 테이블 생성됨 (또는 이유와 함께 N/A 명시)
- [ ] "범위 외" 섹션 작성됨
- [ ] "이미 존재하는 것" 섹션 작성됨
- [ ] 드림 상태 델타 작성됨
- [ ] Completion Summary 생성됨
- [ ] 이중 음성 실행됨 (Codex + Claude 서브에이전트, 또는 사용 불가 명시)
- [ ] CEO 합의 테이블 생성됨

**2단계 (Design) 출력물 — UI 범위가 감지된 경우에만:**
- [ ] 7개 차원 전체 점수와 함께 평가됨
- [ ] 이슈가 식별되고 자동 결정됨
- [ ] 이중 음성 실행됨 (또는 사용 불가/단계와 함께 건너뜀 명시)
- [ ] 디자인 리트머스 스코어카드 생성됨

**3단계 (Eng) 출력물:**
- [ ] 실제 코드 분석이 포함된 범위 도전 ("범위 문제 없음" 이상)
- [ ] 아키텍처 ASCII 다이어그램 생성됨
- [ ] 코드 경로를 테스트 커버리지에 매핑하는 테스트 다이어그램
- [ ] ~/.gstack/projects/$SLUG/에 테스트 플랜 아티팩트 작성됨
- [ ] "범위 외" 섹션 작성됨
- [ ] "이미 존재하는 것" 섹션 작성됨
- [ ] 치명적 갭 평가가 포함된 Failure modes registry
- [ ] Completion Summary 생성됨
- [ ] 이중 음성 실행됨 (Codex + Claude 서브에이전트, 또는 사용 불가 명시)
- [ ] Eng 합의 테이블 생성됨

**크로스 단계:**
- [ ] 크로스 단계 테마 섹션 작성됨

**감사 추적:**
- [ ] Decision Audit Trail에 자동 결정당 최소 한 행 있음 (비어 있지 않음)

위 체크박스 중 누락된 것이 있으면 돌아가서 누락된 출력물을 생성합니다. 최대 2회
시도 — 두 번 재시도 후에도 누락이면 어떤 항목이 미완료인지 경고와 함께 게이트로
진행합니다. 무한 반복하지 마세요.

---

## 4단계: 최종 승인 게이트

**여기서 멈추고 사용자에게 최종 상태를 제시합니다.**

메시지로 제시한 다음 AskUserQuestion을 사용합니다:

```
## /autoplan Review Complete

### Plan Summary
[1-3문장 요약]

### Decisions Made: [N] total ([M] auto-decided, [K] choices for you)

### Your Choices (taste decisions)
[각 취향 결정에 대해:]
**Choice [N]: [제목]** (from [단계])
[X]를 추천합니다 — [원칙]. 하지만 [Y]도 실현 가능합니다:
  [Y를 선택할 경우 1문장 후속 영향]

### Auto-Decided: [M] decisions [see Decision Audit Trail in plan file]

### Review Scores
- CEO: [요약]
- CEO Voices: Codex [요약], Claude subagent [요약], Consensus [X/6 confirmed]
- Design: [요약 또는 "skipped, no UI scope"]
- Design Voices: Codex [요약], Claude subagent [요약], Consensus [X/7 confirmed] (또는 "skipped")
- Eng: [요약]
- Eng Voices: Codex [요약], Claude subagent [요약], Consensus [X/6 confirmed]

### Cross-Phase Themes
[2개 이상 단계의 이중 음성에서 독립적으로 나타난 우려에 대해:]
**Theme: [주제]** — [Phase 1, Phase 3]에서 플래그됨. 높은 신뢰도 신호.
[단계를 가로지르는 테마가 없으면:] "크로스 단계 테마 없음 — 각 단계의 우려는 별개였습니다."

### Deferred to TODOS.md
[이유와 함께 자동 연기된 항목]
```

**인지 부하 관리:**
- 취향 결정 0개: "Your Choices" 섹션 건너뛰기
- 취향 결정 1-7개: 평면 목록
- 8개 이상: 단계별 그룹화. 경고 추가: "이 플랜은 비정상적으로 높은 모호성을 가졌습니다 ([N]개 취향 결정). 주의 깊게 검토하세요."

AskUserQuestion 옵션:
- A) 현재 상태로 승인 (모든 추천 수용)
- B) 재정의와 함께 승인 (변경할 취향 결정 지정)
- C) 질의 (특정 결정에 대해 질문)
- D) 수정 (플랜 자체에 변경 필요)
- E) 거부 (처음부터 다시)

**옵션 처리:**
- A: APPROVED 표시, 리뷰 로그 작성, /ship 제안
- B: 어떤 재정의인지 확인, 적용, 게이트 재제시
- C: 자유 형식 답변, 게이트 재제시
- D: 변경 후 영향받는 단계 재실행 (범위→1B, 디자인→2, 테스트 플랜→3, 아키텍처→3). 최대 3 사이클.
- E: 처음부터 다시

---

## 완료: 리뷰 로그 작성

승인 시, /ship의 대시보드가 인식하도록 3개의 별도 리뷰 로그 항목을 작성합니다.
TIMESTAMP, STATUS, N을 각 리뷰 단계의 실제 값으로 교체하세요.
STATUS는 미해결 이슈가 없으면 "clean", 있으면 "issues_open"입니다.

```bash
COMMIT=$(git rev-parse --short HEAD 2>/dev/null)
TIMESTAMP=$(date -u +%Y-%m-%dT%H:%M:%SZ)

~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"plan-ceo-review","timestamp":"'"$TIMESTAMP"'","status":"STATUS","unresolved":N,"critical_gaps":N,"mode":"SELECTIVE_EXPANSION","via":"autoplan","commit":"'"$COMMIT"'"}'

~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"plan-eng-review","timestamp":"'"$TIMESTAMP"'","status":"STATUS","unresolved":N,"critical_gaps":N,"issues_found":N,"mode":"FULL_REVIEW","via":"autoplan","commit":"'"$COMMIT"'"}'
```

2단계가 실행된 경우 (UI 범위):
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"plan-design-review","timestamp":"'"$TIMESTAMP"'","status":"STATUS","unresolved":N,"via":"autoplan","commit":"'"$COMMIT"'"}'
```

이중 음성 로그 (실행된 각 단계당 하나):
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"autoplan-voices","timestamp":"'"$TIMESTAMP"'","status":"STATUS","source":"SOURCE","phase":"ceo","via":"autoplan","consensus_confirmed":N,"consensus_disagree":N,"commit":"'"$COMMIT"'"}'

~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"autoplan-voices","timestamp":"'"$TIMESTAMP"'","status":"STATUS","source":"SOURCE","phase":"eng","via":"autoplan","consensus_confirmed":N,"consensus_disagree":N,"commit":"'"$COMMIT"'"}'
```

2단계가 실행된 경우 (UI 범위), 추가 로그:
```bash
~/.claude/skills/gstack/bin/gstack-review-log '{"skill":"autoplan-voices","timestamp":"'"$TIMESTAMP"'","status":"STATUS","source":"SOURCE","phase":"design","via":"autoplan","consensus_confirmed":N,"consensus_disagree":N,"commit":"'"$COMMIT"'"}'
```

SOURCE = "codex+subagent", "codex-only", "subagent-only", 또는 "unavailable".
N 값을 테이블의 실제 합의 카운트로 교체하세요.

다음 단계 제안: PR을 생성할 준비가 되면 `/ship`.

---

## 중요 규칙

- **절대 중단하지 마세요.** 사용자가 /autoplan을 선택했습니다. 그 선택을 존중하세요. 모든 취향 결정을 표면화하되, 절대 인터랙티브 리뷰로 리디렉트하지 마세요.
- **전제가 유일한 게이트입니다.** 자동 결정되지 않는 유일한 AskUserQuestion은 1단계의 전제 확인입니다.
- **모든 결정을 기록하세요.** 암묵적 자동 결정은 없습니다. 모든 선택은 감사 추적에 행을 가집니다.
- **전체 깊이는 전체 깊이입니다.** 로드된 스킬 파일의 섹션을 압축하거나 건너뛰지 마세요 (0단계의 스킵 목록 제외). "전체 깊이"의 의미: 섹션이 읽으라는 코드를 읽고, 섹션이 요구하는 출력을 생성하고, 모든 이슈를 식별하고, 각각을 결정합니다. 섹션의 한 문장 요약은 "전체 깊이"가 아닙니다 — 건너뛰기입니다. 리뷰 섹션에 3문장 미만을 쓰고 있다면, 아마 압축하고 있는 것입니다.
- **아티팩트는 산출물입니다.** 테스트 플랜 아티팩트, failure modes registry, error/rescue 테이블, ASCII 다이어그램 — 리뷰 완료 시 디스크 또는 플랜 파일에 존재해야 합니다. 존재하지 않으면 리뷰는 미완료입니다.
- **순차 순서.** CEO → Design → Eng. 각 단계는 이전 단계를 기반으로 합니다.
