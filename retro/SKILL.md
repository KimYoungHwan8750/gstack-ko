---
name: retro
preamble-tier: 2
version: 2.0.0
description: |
  주간 엔지니어링 회고. 커밋 히스토리, 작업 패턴, 코드 품질 지표를
  영속적 히스토리 및 추세 추적과 함께 분석합니다.
  팀 인식: 개인별 기여도를 칭찬 및 성장 영역과 함께 분석합니다.
  다음 요청 시 사용: "weekly retro", "what did we ship", "engineering retrospective".
  작업 주 또는 스프린트 종료 시 사전에 제안합니다.
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
_TEL=$(~/.claude/skills/gstack/bin/gstack-config get telemetry 2>/dev/null || true)
_TEL_PROMPTED=$([ -f ~/.gstack/.telemetry-prompted ] && echo "yes" || echo "no")
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
echo "TELEMETRY: ${_TEL:-off}"
echo "TEL_PROMPTED: $_TEL_PROMPTED"
mkdir -p ~/.gstack/analytics
echo '{"skill":"retro","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
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

# /retro — 주간 엔지니어링 회고

커밋 히스토리, 작업 패턴, 코드 품질 지표를 분석하는 포괄적인 엔지니어링 회고를 생성합니다. 팀 인식: 명령을 실행하는 사용자를 식별한 다음 모든 기여자를 개인별 칭찬 및 성장 기회와 함께 분석합니다. Claude Code를 역량 증폭기로 활용하는 시니어 IC/CTO 수준의 빌더를 위해 설계되었습니다.

## 사용자 호출 가능
사용자가 `/retro`를 입력하면 이 스킬을 실행합니다.

## 인수
- `/retro` — 기본: 최근 7일
- `/retro 24h` — 최근 24시간
- `/retro 14d` — 최근 14일
- `/retro 30d` — 최근 30일
- `/retro compare` — 현재 기간과 동일 길이의 이전 기간 비교
- `/retro compare 14d` — 명시적 기간으로 비교
- `/retro global` — 모든 AI 코딩 도구를 아우르는 크로스 프로젝트 회고 (기본 7일)
- `/retro global 14d` — 명시적 기간의 크로스 프로젝트 회고

## 지침

인수를 파싱하여 시간 범위를 결정합니다. 인수가 없으면 7일을 기본값으로 합니다. 모든 시간은 사용자의 **로컬 시간대**로 표시합니다 (시스템 기본값 사용 — `TZ`를 설정하지 마세요).

**자정 정렬 범위:** 일(`d`) 및 주(`w`) 단위의 경우, 상대적 문자열이 아닌 로컬 자정의 절대 시작 날짜를 계산합니다. 예를 들어 오늘이 2026-03-18이고 범위가 7일이면: 시작 날짜는 2026-03-11입니다. git log 쿼리에 `--since="2026-03-11T00:00:00"`을 사용합니다 — 명시적 `T00:00:00` 접미사가 git이 자정부터 시작하도록 보장합니다. 이것 없이는 git이 현재 벽시계 시간을 사용합니다 (예: 오후 11시에 `--since="2026-03-11"`은 자정이 아닌 오후 11시를 의미). 주 단위의 경우, 7을 곱하여 일수를 구합니다 (예: `2w` = 14일 전). 시간(`h`) 단위의 경우, 자정 정렬이 하위 일 범위에 적용되지 않으므로 `--since="N hours ago"`를 사용합니다.

**인수 검증:** 인수가 숫자 뒤에 `d`, `h`, 또는 `w`가 오는 형식, `compare`(선택적으로 범위 뒤따름), 또는 `global`(선택적으로 범위 뒤따름)과 일치하지 않으면 사용법을 표시하고 중단합니다:
```
Usage: /retro [window | compare | global]
  /retro              — last 7 days (default)
  /retro 24h          — last 24 hours
  /retro 14d          — last 14 days
  /retro 30d          — last 30 days
  /retro compare      — compare this period vs prior period
  /retro compare 14d  — compare with explicit window
  /retro global       — cross-project retro across all AI tools (7d default)
  /retro global 14d   — cross-project retro with explicit window
```

**첫 번째 인수가 `global`인 경우:** 일반 저장소 범위 회고 (1-14단계)를 건너뜁니다. 대신 이 문서 끝의 **글로벌 회고** 플로우를 따릅니다. 선택적 두 번째 인수는 시간 범위입니다 (기본 7d). 이 모드는 git 저장소 내에 있을 필요가 없습니다.

### 1단계: 원시 데이터 수집

먼저 origin을 fetch하고 현재 사용자를 식별합니다:
```bash
git fetch origin <default> --quiet
# 회고를 실행하는 사용자 식별
git config user.name
git config user.email
```

`git config user.name`이 반환하는 이름이 **"당신"** — 이 회고를 읽는 사람입니다. 다른 모든 저자는 팀원입니다. 이를 사용하여 내러티브를 구성합니다: "당신의" 커밋 vs 팀원 기여.

다음 git 명령을 모두 병렬로 실행합니다 (독립적입니다):

```bash
# 1. 범위 내 모든 커밋 (타임스탬프, 제목, 해시, 저자, 변경된 파일, 삽입, 삭제 포함)
git log origin/<default> --since="<window>" --format="%H|%aN|%ae|%ai|%s" --shortstat

# 2. 커밋별 테스트 vs 전체 LOC 분석 (저자 포함)
#    각 커밋 블록은 COMMIT:<hash>|<author>로 시작하고, numstat 라인이 뒤따릅니다.
#    테스트 파일 (test/|spec/|__tests__/ 일치)과 프로덕션 파일을 구분합니다.
git log origin/<default> --since="<window>" --format="COMMIT:%H|%aN" --numstat

# 3. 세션 감지 및 시간별 분포를 위한 커밋 타임스탬프 (저자 포함)
git log origin/<default> --since="<window>" --format="%at|%aN|%ai|%s" | sort -n

# 4. 가장 자주 변경된 파일 (핫스팟 분석)
git log origin/<default> --since="<window>" --format="" --name-only | grep -v '^$' | sort | uniq -c | sort -rn

# 5. 커밋 메시지에서 PR/MR 번호 (GitHub #NNN, GitLab !NNN)
git log origin/<default> --since="<window>" --format="%s" | grep -oE '[#!][0-9]+' | sort -t'#' -k1 | uniq

# 6. 저자별 파일 핫스팟 (누가 무엇을 건드리는지)
git log origin/<default> --since="<window>" --format="AUTHOR:%aN" --name-only

# 7. 저자별 커밋 수 (빠른 요약)
git shortlog origin/<default> --since="<window>" -sn --no-merges

# 8. Greptile 트리아지 히스토리 (있는 경우)
cat ~/.gstack/greptile-history.md 2>/dev/null || true

# 9. TODOS.md 백로그 (있는 경우)
cat TODOS.md 2>/dev/null || true

# 10. 테스트 파일 수
find . -name '*.test.*' -o -name '*.spec.*' -o -name '*_test.*' -o -name '*_spec.*' 2>/dev/null | grep -v node_modules | wc -l

# 11. 범위 내 회귀 테스트 커밋
git log origin/<default> --since="<window>" --oneline --grep="test(qa):" --grep="test(design):" --grep="test: coverage"

# 12. gstack 스킬 사용 텔레메트리 (있는 경우)
cat ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true

# 12. 범위 내 변경된 테스트 파일
git log origin/<default> --since="<window>" --format="" --name-only | grep -E '\.(test|spec)\.' | sort -u | wc -l
```

### 2단계: 지표 산출

이 지표를 요약 테이블로 산출하고 표시합니다:

| 지표 | 값 |
|--------|-------|
| main 커밋 수 | N |
| 기여자 수 | N |
| 병합된 PR | N |
| 총 삽입 | N |
| 총 삭제 | N |
| 순 LOC 추가 | N |
| 테스트 LOC (삽입) | N |
| 테스트 LOC 비율 | N% |
| 버전 범위 | vX.Y.Z.W → vX.Y.Z.W |
| 활동 일수 | N |
| 감지된 세션 | N |
| 평균 LOC/세션-시간 | N |
| Greptile 시그널 | N% (Y 캐치, Z FP) |
| 테스트 상태 | 총 N개 테스트 · 이번 기간 M개 추가 · K개 회귀 테스트 |

그런 다음 바로 아래에 **저자별 리더보드**를 표시합니다:

```
Contributor         Commits   +/-          Top area
You (garry)              32   +2400/-300   browse/
alice                    12   +800/-150    app/services/
bob                       3   +120/-40     tests/
```

커밋 수 내림차순으로 정렬합니다. 현재 사용자 (`git config user.name`에서)는 항상 첫 번째로 표시하며, "You (이름)"으로 레이블합니다.

**Greptile 시그널 (히스토리가 있는 경우):** `~/.gstack/greptile-history.md` (1단계, 명령 8에서 가져옴)를 읽습니다. 회고 시간 범위 내 항목을 날짜별로 필터합니다. 유형별로 집계: `fix`, `fp`, `already-fixed`. 시그널 비율 산출: `(fix + already-fixed) / (fix + already-fixed + fp)`. 범위 내 항목이 없거나 파일이 존재하지 않으면 Greptile 지표 행을 건너뜁니다. 파싱할 수 없는 줄은 조용히 건너뜁니다.

**백로그 상태 (TODOS.md가 있는 경우):** `TODOS.md` (1단계, 명령 9에서 가져옴)를 읽습니다. 산출:
- 총 미해결 TODO (`## Completed` 섹션의 항목 제외)
- P0/P1 수 (크리티컬/긴급 항목)
- P2 수 (중요 항목)
- 이번 기간 완료된 항목 (Completed 섹션에서 회고 범위 내 날짜가 있는 항목)
- 이번 기간 추가된 항목 (범위 내 TODOS.md를 수정한 커밋을 교차 참조)

지표 테이블에 포함:
```
| Backlog Health | N open (X P0/P1, Y P2) · Z completed this period |
```

TODOS.md가 존재하지 않으면 Backlog Health 행을 건너뜁니다.

**스킬 사용 (분석 데이터가 있는 경우):** `~/.gstack/analytics/skill-usage.jsonl`이 존재하면 읽습니다. `ts` 필드로 회고 시간 범위 내 항목을 필터합니다. 스킬 활성화 (`event` 필드 없음)와 훅 발동 (`event: "hook_fire"`)을 구분합니다. 스킬 이름별로 집계합니다. 표시:

```
| Skill Usage | /ship(12) /qa(8) /review(5) · 3 safety hook fires |
```

JSONL 파일이 존재하지 않거나 범위 내 항목이 없으면 Skill Usage 행을 건너뜁니다.

**유레카 순간 (기록된 경우):** `~/.gstack/analytics/eureka.jsonl`이 존재하면 읽습니다. `ts` 필드로 회고 시간 범위 내 항목을 필터합니다. 각 유레카 순간에 대해 플래그한 스킬, 브랜치, 인사이트의 한 줄 요약을 표시합니다. 표시:

```
| Eureka Moments | 2 this period |
```

순간이 있으면 나열합니다:
```
  EUREKA /office-hours (branch: garrytan/auth-rethink): "Session tokens don't need server storage — browser crypto API makes client-side JWT validation viable"
  EUREKA /plan-eng-review (branch: garrytan/cache-layer): "Redis isn't needed here — Bun's built-in LRU cache handles this workload"
```

JSONL 파일이 존재하지 않거나 범위 내 항목이 없으면 Eureka Moments 행을 건너뜁니다.

### 3단계: 커밋 시간 분포

로컬 시간 기준 시간별 히스토그램을 막대 차트로 표시합니다:

```
Hour  Commits  ████████████████
 00:    4      ████
 07:    5      █████
 ...
```

다음을 식별하고 지적합니다:
- 피크 시간대
- 데드 존
- 패턴이 이봉형(아침/저녁)인지 연속적인지
- 야간 코딩 집중 (오후 10시 이후)

### 4단계: 작업 세션 감지

연속 커밋 간 **45분 갭** 임계값을 사용하여 세션을 감지합니다. 각 세션에 대해 보고:
- 시작/종료 시간 (Pacific)
- 커밋 수
- 분 단위 지속 시간

세션 분류:
- **딥 세션** (50분 이상)
- **미디엄 세션** (20-50분)
- **마이크로 세션** (20분 미만, 일반적으로 단일 커밋 후 이동)

산출:
- 총 활성 코딩 시간 (세션 지속 시간 합계)
- 평균 세션 길이
- 활성 시간당 LOC

### 5단계: 커밋 유형 분류

기존 커밋 접두사 (feat/fix/refactor/test/chore/docs)로 분류합니다. 백분율 막대로 표시:

```
feat:     20  (40%)  ████████████████████
fix:      27  (54%)  ███████████████████████████
refactor:  2  ( 4%)  ██
```

fix 비율이 50%를 초과하면 플래그합니다 — 이는 리뷰 부족을 나타낼 수 있는 "빠르게 배포, 빠르게 수정" 패턴을 시사합니다.

### 6단계: 핫스팟 분석

가장 많이 변경된 상위 10개 파일을 표시합니다. 플래그:
- 5회 이상 변경된 파일 (변동 핫스팟)
- 핫스팟 목록에서 테스트 파일 vs 프로덕션 파일
- VERSION/CHANGELOG 빈도 (버전 관리 규율 지표)

### 7단계: PR 크기 분포

커밋 diff에서 PR 크기를 추정하고 버킷으로 분류합니다:
- **Small** (<100 LOC)
- **Medium** (100-500 LOC)
- **Large** (500-1500 LOC)
- **XL** (1500+ LOC)

### 8단계: 집중도 점수 + 금주의 배포

**집중도 점수:** 가장 많이 변경된 단일 최상위 디렉토리 (예: `app/services/`, `app/views/`)에 접촉하는 커밋의 백분율을 산출합니다. 높은 점수 = 깊이 집중한 작업. 낮은 점수 = 분산된 컨텍스트 스위칭. 보고: "Focus score: 62% (app/services/)"

**금주의 배포:** 범위 내 가장 높은 LOC의 단일 PR을 자동 식별합니다. 하이라이트:
- PR 번호 및 제목
- 변경된 LOC
- 중요한 이유 (커밋 메시지와 관련 파일에서 추론)

### 9단계: 팀원 분석

각 기여자 (현재 사용자 포함)에 대해 산출:

1. **커밋 및 LOC** — 총 커밋, 삽입, 삭제, 순 LOC
2. **집중 영역** — 가장 많이 접촉한 디렉토리/파일 (상위 3개)
3. **커밋 유형 구성** — 개인 feat/fix/refactor/test 분류
4. **세션 패턴** — 코딩하는 시간대 (피크 시간), 세션 수
5. **테스트 규율** — 개인 테스트 LOC 비율
6. **최대 배포** — 범위 내 가장 영향력 있는 단일 커밋 또는 PR

**현재 사용자 ("당신"):** 이 섹션이 가장 심층적으로 다뤄집니다. 솔로 회고의 모든 세부 사항을 포함합니다 — 세션 분석, 시간 패턴, 집중도 점수. 1인칭으로 구성: "당신의 피크 시간대...", "당신의 최대 배포..."

**각 팀원:** 작업 내용과 패턴에 대한 2-3문장. 그런 다음:

- **칭찬** (1-2개 구체적 사항): 실제 커밋에 근거합니다. "잘했습니다"가 아닌 — 무엇이 좋았는지 정확히 말합니다. 예: "3개의 집중 세션에서 전체 인증 미들웨어 리라이트를 45% 테스트 커버리지로 배포", "모든 PR이 200 LOC 미만 — 규율 있는 분해."
- **성장 기회** (1개 구체적 사항): 비판이 아닌 레벨업 제안으로 프레임합니다. 실제 데이터에 근거합니다. 예: "이번 주 테스트 비율이 12%입니다 — 결제 모듈이 더 복잡해지기 전에 테스트 커버리지에 투자하면 효과가 있을 것입니다", "같은 파일에 5개의 fix 커밋은 원래 PR에 리뷰 패스가 필요했음을 시사합니다."

**기여자가 한 명인 경우 (솔로 저장소):** 팀 분석을 건너뛰고 이전과 같이 진행합니다 — 회고는 개인적입니다.

**Co-Authored-By 트레일러가 있는 경우:** 커밋 메시지에서 `Co-Authored-By:` 라인을 파싱합니다. 해당 저자를 주 저자와 함께 커밋에 기여한 것으로 크레딧합니다. AI 공동 저자 (예: `noreply@anthropic.com`)는 참고하되 팀원으로 포함하지 않습니다 — 대신 "AI 보조 커밋"을 별도 지표로 추적합니다.

### 10단계: 주간 추세 (범위 >= 14일인 경우)

시간 범위가 14일 이상이면 주간 버킷으로 분할하고 추세를 표시합니다:
- 주당 커밋 (전체 및 저자별)
- 주당 LOC
- 주당 테스트 비율
- 주당 fix 비율
- 주당 세션 수

### 11단계: 스트릭 추적

오늘부터 거슬러 올라가며 origin/<default>에 최소 1개의 커밋이 있는 연속 일수를 세합니다. 팀 스트릭과 개인 스트릭 모두 추적:

```bash
# 팀 스트릭: 모든 고유 커밋 날짜 (로컬 시간) — 하드 컷오프 없음
git log origin/<default> --format="%ad" --date=format:"%Y-%m-%d" | sort -u

# 개인 스트릭: 현재 사용자의 커밋만
git log origin/<default> --author="<user_name>" --format="%ad" --date=format:"%Y-%m-%d" | sort -u
```

오늘부터 역으로 세기 — 최소 하나의 커밋이 있는 연속 일수는? 이것은 전체 히스토리를 쿼리하므로 모든 길이의 스트릭이 정확하게 보고됩니다. 둘 다 표시:
- "팀 배포 스트릭: 47일 연속"
- "당신의 배포 스트릭: 32일 연속"

### 12단계: 히스토리 로드 및 비교

새 스냅샷을 저장하기 전에 이전 회고 히스토리를 확인합니다:

```bash
ls -t .context/retros/*.json 2>/dev/null
```

**이전 회고가 있는 경우:** Read 도구를 사용하여 가장 최근 것을 로드합니다. 핵심 지표의 델타를 산출하고 **지난 회고 대비 추세** 섹션을 포함합니다:
```
                    Last        Now         Delta
Test ratio:         22%    →    41%         ↑19pp
Sessions:           10     →    14          ↑4
LOC/hour:           200    →    350         ↑75%
Fix ratio:          54%    →    30%         ↓24pp (improving)
Commits:            32     →    47          ↑47%
Deep sessions:      3      →    5           ↑2
```

**이전 회고가 없는 경우:** 비교 섹션을 건너뛰고 다음을 추가합니다: "첫 번째 회고가 기록되었습니다 — 다음 주에 다시 실행하면 추세를 볼 수 있습니다."

### 13단계: 회고 히스토리 저장

모든 지표 (스트릭 포함)를 산출하고 비교를 위한 이전 히스토리를 로드한 후 JSON 스냅샷을 저장합니다:

```bash
mkdir -p .context/retros
```

오늘의 다음 시퀀스 번호를 결정합니다 (`$(date +%Y-%m-%d)`를 실제 날짜로 대체):
```bash
# 오늘의 기존 회고 수를 세어 다음 시퀀스 번호 결정
today=$(date +%Y-%m-%d)
existing=$(ls .context/retros/${today}-*.json 2>/dev/null | wc -l | tr -d ' ')
next=$((existing + 1))
# .context/retros/${today}-${next}.json으로 저장
```

Write 도구를 사용하여 다음 스키마로 JSON 파일을 저장합니다:
```json
{
  "date": "2026-03-08",
  "window": "7d",
  "metrics": {
    "commits": 47,
    "contributors": 3,
    "prs_merged": 12,
    "insertions": 3200,
    "deletions": 800,
    "net_loc": 2400,
    "test_loc": 1300,
    "test_ratio": 0.41,
    "active_days": 6,
    "sessions": 14,
    "deep_sessions": 5,
    "avg_session_minutes": 42,
    "loc_per_session_hour": 350,
    "feat_pct": 0.40,
    "fix_pct": 0.30,
    "peak_hour": 22,
    "ai_assisted_commits": 32
  },
  "authors": {
    "Garry Tan": { "commits": 32, "insertions": 2400, "deletions": 300, "test_ratio": 0.41, "top_area": "browse/" },
    "Alice": { "commits": 12, "insertions": 800, "deletions": 150, "test_ratio": 0.35, "top_area": "app/services/" }
  },
  "version_range": ["1.16.0.0", "1.16.1.0"],
  "streak_days": 47,
  "tweetable": "Week of Mar 1: 47 commits (3 contributors), 3.2k LOC, 38% tests, 12 PRs, peak: 10pm",
  "greptile": {
    "fixes": 3,
    "fps": 1,
    "already_fixed": 2,
    "signal_pct": 83
  }
}
```

**참고:** `greptile` 필드는 `~/.gstack/greptile-history.md`가 존재하고 시간 범위 내 항목이 있는 경우에만 포함합니다. `backlog` 필드는 `TODOS.md`가 존재하는 경우에만 포함합니다. `test_health` 필드는 테스트 파일이 발견된 경우 (명령 10이 > 0 반환)에만 포함합니다. 데이터가 없으면 해당 필드를 완전히 생략합니다.

테스트 파일이 있으면 JSON에 테스트 상태 데이터를 포함합니다:
```json
  "test_health": {
    "total_test_files": 47,
    "tests_added_this_period": 5,
    "regression_test_commits": 3,
    "test_files_changed": 8
  }
```

TODOS.md가 있으면 JSON에 백로그 데이터를 포함합니다:
```json
  "backlog": {
    "total_open": 28,
    "p0_p1": 2,
    "p2": 8,
    "completed_this_period": 3,
    "added_this_period": 1
  }
```

### 14단계: 내러티브 작성

출력을 다음과 같이 구조화합니다:

---

**트윗 가능한 요약** (첫 줄, 다른 모든 것보다 먼저):
```
Week of Mar 1: 47 commits (3 contributors), 3.2k LOC, 38% tests, 12 PRs, peak: 10pm | Streak: 47d
```

## 엔지니어링 회고: [날짜 범위]

### 요약 테이블
(2단계에서)

### 지난 회고 대비 추세
(11단계에서, 저장 전 로드 — 첫 회고면 건너뜀)

### 시간 및 세션 패턴
(3-4단계에서)

팀 전체 패턴이 의미하는 바를 해석하는 내러티브:
- 가장 생산적인 시간대와 그 원인
- 세션이 시간이 지남에 따라 길어지는지 짧아지는지
- 일일 활성 코딩 시간 추정 (팀 합산)
- 주목할 패턴: 팀원들이 같은 시간에 코딩하는지 교대로 하는지?

### 배포 속도
(5-7단계에서)

다루는 내러티브:
- 커밋 유형 구성과 그것이 드러내는 것
- PR 크기 분포와 그것이 배포 케이던스에 대해 드러내는 것
- Fix 체인 감지 (같은 서브시스템에 대한 연속 fix 커밋)
- 버전 범프 규율

### 코드 품질 시그널
- 테스트 LOC 비율 추세
- 핫스팟 분석 (같은 파일이 계속 변동하는가?)
- Greptile 시그널 비율 및 추세 (히스토리가 있는 경우): "Greptile: X% signal (Y valid catches, Z false positives)"

### 테스트 상태
- 총 테스트 파일: N (명령 10에서)
- 이번 기간 추가된 테스트: M (명령 12에서 — 변경된 테스트 파일)
- 회귀 테스트 커밋: 명령 11에서 `test(qa):`, `test(design):`, `test: coverage` 커밋 나열
- 이전 회고가 존재하고 `test_health`가 있으면: 델타 표시 "테스트 수: {last} → {now} (+{delta})"
- 테스트 비율 < 20%이면: 성장 영역으로 플래그 — "100% 테스트 커버리지가 목표입니다. 테스트는 바이브 코딩을 안전하게 만듭니다."

### 계획 완료
이번 기간의 /ship 실행에서 계획 완료 데이터에 대한 리뷰 JSONL 로그를 확인합니다:

```bash
eval "$(~/.claude/skills/gstack/bin/gstack-slug 2>/dev/null)"
cat ~/.gstack/projects/$SLUG/*-reviews.jsonl 2>/dev/null | grep '"skill":"ship"' | grep '"plan_items_total"' || echo "NO_PLAN_DATA"
```

회고 시간 범위 내에 계획 완료 데이터가 있으면:
- 계획과 함께 배포된 브랜치 수 (`plan_items_total` > 0인 항목)
- 평균 완료율 산출: `plan_items_done` 합계 / `plan_items_total` 합계
- 데이터가 뒷받침하면 가장 많이 건너뛴 항목 카테고리 식별

출력:
```
Plan Completion This Period:
  {N} branches shipped with plans
  Average completion: {X}% ({done}/{total} items)
```

계획 데이터가 없으면 이 섹션을 조용히 건너뜁니다.

### 집중도 및 하이라이트
(8단계에서)
- 집중도 점수와 해석
- 금주의 배포 콜아웃

### 당신의 한 주 (개인 심층 분석)
(9단계에서, 현재 사용자만)

사용자가 가장 관심을 가지는 섹션입니다. 포함:
- 개인 커밋 수, LOC, 테스트 비율
- 세션 패턴 및 피크 시간대
- 집중 영역
- 최대 배포
- **잘한 점** (커밋에 근거한 2-3개 구체적 사항)
- **레벨업할 부분** (1-2개 구체적, 실행 가능한 제안)

### 팀 분석
(9단계에서, 각 팀원 — 솔로 저장소면 건너뜀)

각 팀원 (커밋 내림차순 정렬)에 대해 섹션 작성:

#### [이름]
- **배포한 것**: 기여, 집중 영역, 커밋 패턴에 대한 2-3문장
- **칭찬**: 실제 커밋에 근거한 1-2가지 잘한 점. 진정성 있게 — 1:1에서 실제로 할 말은? 예:
  - "3개의 작고 리뷰 가능한 PR로 전체 인증 모듈을 정리 — 교과서적 분해"
  - "모든 새 엔드포인트에 통합 테스트 추가, 해피 패스만이 아닌"
  - "대시보드에서 2초 로드 시간을 유발하던 N+1 쿼리 수정"
- **성장 기회**: 1가지 구체적이고 건설적인 제안. 비판이 아닌 투자로 프레임. 예:
  - "결제 모듈의 테스트 커버리지가 8% — 다음 기능이 위에 쌓이기 전에 투자할 가치가 있습니다"
  - "대부분의 커밋이 한 번에 몰아서 — 하루에 걸쳐 작업을 분산하면 컨텍스트 스위칭 피로를 줄일 수 있습니다"
  - "모든 커밋이 새벽 1-4시 사이 — 지속 가능한 페이스가 장기적 코드 품질에 중요합니다"

**AI 협업 노트:** 많은 커밋에 `Co-Authored-By` AI 트레일러 (예: Claude, Copilot)가 있으면 AI 보조 커밋 비율을 팀 지표로 기록합니다. 중립적으로 프레임합니다 — "커밋의 N%가 AI 보조" — 판단 없이.

### 팀 상위 3가지 성과
범위 내 전체 팀에서 배포된 가장 영향력 있는 3가지를 식별합니다. 각각에 대해:
- 무엇이었는지
- 누가 배포했는지
- 왜 중요한지 (제품/아키텍처 영향)

### 개선할 3가지
구체적이고, 실행 가능하며, 실제 커밋에 근거합니다. 개인 및 팀 수준 제안을 혼합합니다. "더 나아지려면 팀이..."으로 표현합니다.

### 다음 주 3가지 습관
작고, 실용적이고, 현실적입니다. 각각 채택에 5분 미만이 걸려야 합니다. 최소 하나는 팀 지향적이어야 합니다 (예: "서로의 PR을 당일 리뷰").

### 주간 추세
(해당하는 경우, 10단계에서)

---

## 글로벌 회고 모드

사용자가 `/retro global` (또는 `/retro global 14d`)을 실행할 때, 저장소 범위의 1-14단계 대신 이 플로우를 따릅니다. 이 모드는 어떤 디렉토리에서든 작동합니다 — git 저장소 내에 있을 필요가 없습니다.

### 글로벌 1단계: 시간 범위 산출

일반 회고와 동일한 자정 정렬 로직. 기본 7d. `global` 뒤의 두 번째 인수가 범위입니다 (예: `14d`, `30d`, `24h`).

### 글로벌 2단계: 디스커버리 실행

다음 폴백 체인을 사용하여 디스커버리 스크립트를 찾고 실행합니다:

```bash
DISCOVER_BIN=""
[ -x ~/.claude/skills/gstack/bin/gstack-global-discover ] && DISCOVER_BIN=~/.claude/skills/gstack/bin/gstack-global-discover
[ -z "$DISCOVER_BIN" ] && [ -x .claude/skills/gstack/bin/gstack-global-discover ] && DISCOVER_BIN=.claude/skills/gstack/bin/gstack-global-discover
[ -z "$DISCOVER_BIN" ] && which gstack-global-discover >/dev/null 2>&1 && DISCOVER_BIN=$(which gstack-global-discover)
[ -z "$DISCOVER_BIN" ] && [ -f bin/gstack-global-discover.ts ] && DISCOVER_BIN="bun run bin/gstack-global-discover.ts"
echo "DISCOVER_BIN: $DISCOVER_BIN"
```

바이너리를 찾을 수 없으면 사용자에게 알립니다: "디스커버리 스크립트를 찾을 수 없습니다. gstack 디렉토리에서 `bun run build`를 실행하세요." 그리고 중단합니다.

디스커버리 실행:
```bash
$DISCOVER_BIN --since "<window>" --format json 2>/tmp/gstack-discover-stderr
```

진단 정보를 위해 `/tmp/gstack-discover-stderr`의 stderr 출력을 읽습니다. stdout에서 JSON 출력을 파싱합니다.

`total_sessions`가 0이면 다음을 말합니다: "지난 <window> 동안 AI 코딩 세션을 찾을 수 없습니다. 더 긴 범위를 시도하세요: `/retro global 30d`" 그리고 중단합니다.

### 글로벌 3단계: 발견된 각 저장소에서 git log 실행

디스커버리 JSON의 `repos` 배열에서 각 저장소에 대해, `paths[]`에서 첫 번째 유효한 경로 (`.git/`이 있는 디렉토리)를 찾습니다. 유효한 경로가 없으면 해당 저장소를 건너뛰고 기록합니다.

**로컬 전용 저장소** (`remote`가 `local:`로 시작하는 경우): `git fetch`를 건너뛰고 로컬 기본 브랜치를 사용합니다. `git log origin/$DEFAULT` 대신 `git log HEAD`를 사용합니다.

**원격이 있는 저장소:**

```bash
git -C <path> fetch origin --quiet 2>/dev/null
```

각 저장소의 기본 브랜치를 감지합니다: 먼저 `git symbolic-ref refs/remotes/origin/HEAD`를 시도하고, 일반적인 브랜치 이름 (`main`, `master`)을 확인한 다음, `git rev-parse --abbrev-ref HEAD`로 폴백합니다. 감지된 브랜치를 아래 명령에서 `<default>`로 사용합니다.

```bash
# 통계가 포함된 커밋
git -C <path> log origin/$DEFAULT --since="<start_date>T00:00:00" --format="%H|%aN|%ai|%s" --shortstat

# 세션 감지, 스트릭, 컨텍스트 스위칭을 위한 커밋 타임스탬프
git -C <path> log origin/$DEFAULT --since="<start_date>T00:00:00" --format="%at|%aN|%ai|%s" | sort -n

# 저자별 커밋 수
git -C <path> shortlog origin/$DEFAULT --since="<start_date>T00:00:00" -sn --no-merges

# 커밋 메시지에서 PR/MR 번호 (GitHub #NNN, GitLab !NNN)
git -C <path> log origin/$DEFAULT --since="<start_date>T00:00:00" --format="%s" | grep -oE '[#!][0-9]+' | sort -t'#' -k1 | uniq
```

실패하는 저장소 (삭제된 경로, 네트워크 에러): 건너뛰고 "N개 저장소에 접근할 수 없었습니다."를 기록합니다.

### 글로벌 4단계: 글로벌 배포 스트릭 산출

각 저장소에 대해 커밋 날짜를 가져옵니다 (365일로 제한):

```bash
git -C <path> log origin/$DEFAULT --since="365 days ago" --format="%ad" --date=format:"%Y-%m-%d" | sort -u
```

모든 저장소의 날짜를 합집합합니다. 오늘부터 역으로 세기 — 어떤 저장소에든 최소 하나의 커밋이 있는 연속 일수는? 스트릭이 365일에 도달하면 "365+ days"로 표시합니다.

### 글로벌 5단계: 컨텍스트 스위칭 지표 산출

3단계에서 수집한 커밋 타임스탬프에서 날짜별로 그룹화합니다. 각 날짜에 대해 그날 커밋이 있는 고유 저장소 수를 셉니다. 보고:
- 일일 평균 저장소 수
- 일일 최대 저장소 수
- 집중한 날 (1개 저장소) vs 분산된 날 (3개 이상 저장소)

### 글로벌 6단계: 도구별 생산성 패턴

디스커버리 JSON에서 도구 사용 패턴을 분석합니다:
- 어떤 AI 도구가 어떤 저장소에 사용되는지 (독점 vs 공유)
- 도구별 세션 수
- 행동 패턴 (예: "Codex는 myapp에만 독점 사용, Claude Code는 나머지 모든 곳")

### 글로벌 7단계: 집계 및 내러티브 생성

**공유 가능한 개인 카드를 먼저** 출력하고, 그 아래에 전체
팀/프로젝트 분석을 구조화합니다. 개인 카드는 스크린샷에 적합하게
설계되었습니다 — 누군가가 X/Twitter에서 공유하고 싶은 모든 것이 하나의 깔끔한 블록에.

---

**트윗 가능한 요약** (첫 줄, 다른 모든 것보다 먼저):
```
Week of Mar 14: 5 projects, 138 commits, 250k LOC across 5 repos | 48 AI sessions | Streak: 52d 🔥
```

## 🚀 당신의 한 주: [사용자 이름] — [날짜 범위]

이 섹션은 **공유 가능한 개인 카드**입니다. 현재 사용자의 통계만 포함합니다
— 팀 데이터 없음, 프로젝트 분석 없음. 스크린샷 후 게시하도록 설계되었습니다.

`git config user.name`의 사용자 ID를 사용하여 모든 저장소별 git 데이터를 필터합니다.
모든 저장소에 걸쳐 집계하여 개인 합계를 산출합니다.

시각적으로 깔끔한 단일 블록으로 렌더링합니다. 왼쪽 테두리만 — 오른쪽 테두리 없음 (LLM이
오른쪽 테두리를 안정적으로 정렬할 수 없음). 열이 깔끔하게 정렬되도록 저장소 이름을
가장 긴 이름에 맞춰 패딩합니다. 프로젝트 이름을 절대 잘라내지 마세요.

```
╔═══════════════════════════════════════════════════════════════
║  [USER NAME] — Week of [date]
╠═══════════════════════════════════════════════════════════════
║
║  [N] commits across [M] projects
║  +[X]k LOC added · [Y]k LOC deleted · [Z]k net
║  [N] AI coding sessions (CC: X, Codex: Y, Gemini: Z)
║  [N]-day shipping streak 🔥
║
║  PROJECTS
║  ─────────────────────────────────────────────────────────
║  [repo_name_full]        [N] commits    +[X]k LOC    [solo/team]
║  [repo_name_full]        [N] commits    +[X]k LOC    [solo/team]
║  [repo_name_full]        [N] commits    +[X]k LOC    [solo/team]
║
║  SHIP OF THE WEEK
║  [PR title] — [LOC] lines across [N] files
║
║  TOP WORK
║  • [1-line description of biggest theme]
║  • [1-line description of second theme]
║  • [1-line description of third theme]
║
║  Powered by gstack
╚═══════════════════════════════════════════════════════════════
```

**개인 카드 규칙:**
- 사용자의 커밋이 있는 저장소만 표시합니다. 0개인 저장소는 건너뜁니다.
- 사용자의 커밋 수 내림차순으로 정렬합니다.
- **저장소 이름을 절대 잘라내지 마세요.** 전체 저장소 이름을 사용합니다 (예: `analyze_transcripts`
  가 아닌 `analyze_trans`). 모든 열이 정렬되도록 이름 열을 가장 긴 저장소 이름에
  패딩합니다. 이름이 길면 박스를 넓힙니다 — 박스 너비는 내용에 맞춰 적응합니다.
- LOC는 천 단위에 "k" 포맷을 사용합니다 (예: "+64.0k"가 아닌 "+64010").
- 역할: 사용자가 유일한 기여자이면 "solo", 다른 사람이 기여했으면 "team".
- 금주의 배포: 모든 저장소에 걸친 사용자의 단일 최고 LOC PR.
- 주요 작업: 커밋 메시지에서 추론한 사용자의 주요 테마를 요약하는 3개 항목.
  개별 커밋이 아닌 테마로 종합합니다.
  예: "Built /retro global — cross-project retrospective with AI session discovery"
  가 아닌 "feat: gstack-global-discover" + "feat: /retro global template".
- 카드는 자체 완결적이어야 합니다. 이 블록만 보는 사람도 주변 맥락 없이
  사용자의 한 주를 이해할 수 있어야 합니다.
- 팀원, 프로젝트 합계, 컨텍스트 스위칭 데이터를 여기에 포함하지 마세요.

**개인 스트릭:** 모든 저장소에 걸쳐 사용자 자신의 커밋 (`--author`로 필터)을 사용하여
팀 스트릭과 별도로 개인 스트릭을 산출합니다.

---

## 글로벌 엔지니어링 회고: [날짜 범위]

아래의 모든 내용은 전체 분석입니다 — 팀 데이터, 프로젝트 분석, 패턴.
공유 가능한 카드 다음에 오는 "심층 분석"입니다.

### 전체 프로젝트 개요
| 지표 | 값 |
|--------|-------|
| 활동 프로젝트 | N |
| 총 커밋 (모든 저장소, 모든 기여자) | N |
| 총 LOC | +N / -N |
| AI 코딩 세션 | N (CC: X, Codex: Y, Gemini: Z) |
| 활동 일수 | N |
| 글로벌 배포 스트릭 (모든 기여자, 모든 저장소) | N일 연속 |
| 일일 컨텍스트 스위치 | 평균 N (최대: M) |

### 프로젝트별 분석
각 저장소 (커밋 내림차순 정렬)에 대해:
- 저장소 이름 (총 커밋의 % 포함)
- 커밋, LOC, 병합된 PR, 상위 기여자
- 주요 작업 (커밋 메시지에서 추론)
- 도구별 AI 세션

**당신의 기여** (각 프로젝트 내 하위 섹션):
각 프로젝트에 대해, 해당 저장소 내 현재 사용자의 개인 통계를 보여주는
"당신의 기여" 블록을 추가합니다. `git config user.name`의 사용자 ID를 사용하여
필터합니다. 포함:
- 당신의 커밋 / 총 커밋 (% 포함)
- 당신의 LOC (+삽입 / -삭제)
- 당신의 주요 작업 (당신의 커밋 메시지에서만 추론)
- 당신의 커밋 유형 구성 (feat/fix/refactor/chore/docs 분류)
- 이 저장소에서 당신의 최대 배포 (최고 LOC 커밋 또는 PR)

사용자가 유일한 기여자이면 "솔로 프로젝트 — 모든 커밋이 당신의 것입니다."라고 합니다.
사용자가 저장소에 0개의 커밋이 있으면 (이번 기간 건드리지 않은 팀 프로젝트),
"이번 기간 커밋 없음 — AI 세션 [N]개만."이라고 하고 분석을 건너뜁니다.

형식:
```
**Your contributions:** 47/244 commits (19%), +4.2k/-0.3k LOC
  Key work: Writer Chat, email blocking, security hardening
  Biggest ship: PR #605 — Writer Chat eats the admin bar (2,457 ins, 46 files)
  Mix: feat(3) fix(2) chore(1)
```

### 크로스 프로젝트 패턴
- 프로젝트 간 시간 할당 (% 분석, 총계가 아닌 당신의 커밋 사용)
- 모든 저장소에 걸쳐 집계한 피크 생산성 시간대
- 집중한 날 vs 분산된 날
- 컨텍스트 스위칭 추세

### 도구 사용 분석
도구별 분석과 행동 패턴:
- Claude Code: M개 저장소에 걸쳐 N개 세션 — 관찰된 패턴
- Codex: M개 저장소에 걸쳐 N개 세션 — 관찰된 패턴
- Gemini: M개 저장소에 걸쳐 N개 세션 — 관찰된 패턴

### 금주의 배포 (글로벌)
모든 프로젝트에 걸친 가장 영향력 있는 PR. LOC와 커밋 메시지로 식별합니다.

### 크로스 프로젝트 인사이트 3가지
단일 저장소 회고로는 볼 수 없는, 글로벌 관점이 드러내는 것.

### 다음 주 3가지 습관
전체 크로스 프로젝트 그림을 고려하여.

---

### 글로벌 8단계: 히스토리 로드 및 비교

```bash
ls -t ~/.gstack/retros/global-*.json 2>/dev/null | head -5
```

**동일한 `window` 값을 가진 이전 회고와만 비교합니다** (예: 7d vs 7d). 가장 최근 이전 회고가 다른 범위를 가지면 비교를 건너뛰고 기록합니다: "이전 글로벌 회고는 다른 범위를 사용했습니다 — 비교를 건너뜁니다."

일치하는 이전 회고가 있으면 Read 도구로 로드합니다. 핵심 지표의 델타와 함께 **지난 글로벌 회고 대비 추세** 테이블을 표시합니다: 총 커밋, LOC, 세션, 스트릭, 일일 컨텍스트 스위치.

이전 글로벌 회고가 없으면 다음을 추가합니다: "첫 번째 글로벌 회고가 기록되었습니다 — 다음 주에 다시 실행하면 추세를 볼 수 있습니다."

### 글로벌 9단계: 스냅샷 저장

```bash
mkdir -p ~/.gstack/retros
```

오늘의 다음 시퀀스 번호를 결정합니다:
```bash
today=$(date +%Y-%m-%d)
existing=$(ls ~/.gstack/retros/global-${today}-*.json 2>/dev/null | wc -l | tr -d ' ')
next=$((existing + 1))
```

Write 도구를 사용하여 JSON을 `~/.gstack/retros/global-${today}-${next}.json`에 저장합니다:

```json
{
  "type": "global",
  "date": "2026-03-21",
  "window": "7d",
  "projects": [
    {
      "name": "gstack",
      "remote": "<detected from git remote get-url origin, normalized to HTTPS>",
      "commits": 47,
      "insertions": 3200,
      "deletions": 800,
      "sessions": { "claude_code": 15, "codex": 3, "gemini": 0 }
    }
  ],
  "totals": {
    "commits": 182,
    "insertions": 15300,
    "deletions": 4200,
    "projects": 5,
    "active_days": 6,
    "sessions": { "claude_code": 48, "codex": 8, "gemini": 3 },
    "global_streak_days": 52,
    "avg_context_switches_per_day": 2.1
  },
  "tweetable": "Week of Mar 14: 5 projects, 182 commits, 15.3k LOC | CC: 48, Codex: 8, Gemini: 3 | Focus: gstack (58%) | Streak: 52d"
}
```

---

## 비교 모드

사용자가 `/retro compare` (또는 `/retro compare 14d`)를 실행할 때:

1. 자정 정렬 시작 날짜를 사용하여 현재 범위 (기본 7d)의 지표를 산출합니다 (일반 회고와 동일한 로직 — 예: 오늘이 2026-03-18이고 범위가 7d이면, `--since="2026-03-11T00:00:00"` 사용)
2. 겹침을 피하기 위해 `--since`와 `--until` 모두 자정 정렬 날짜를 사용하여 직전 동일 길이 범위의 지표를 산출합니다 (예: 2026-03-11 시작 7d 범위: 이전 범위는 `--since="2026-03-04T00:00:00" --until="2026-03-11T00:00:00"`)
3. 델타와 화살표가 포함된 나란히 비교 테이블을 표시합니다
4. 가장 큰 개선과 회귀를 강조하는 간단한 내러티브를 작성합니다
5. 현재 범위 스냅샷만 `.context/retros/`에 저장합니다 (일반 회고 실행과 동일); 이전 범위 지표는 저장하지 **않습니다**.

## 어조

- 격려하되 솔직, 달래지 않음
- 구체적이고 명확 — 항상 실제 커밋/코드에 근거
- 일반적인 칭찬 ("잘했어요!") 건너뜀 — 무엇이 좋았고 왜 좋은지 정확히 말함
- 개선 사항은 레벨업으로 프레임, 비판이 아님
- **칭찬은 1:1에서 실제로 할 말처럼 느껴져야 합니다** — 구체적이고, 당연하며, 진정성 있게
- **성장 제안은 투자 조언처럼 느껴져야 합니다** — "이것은 당신의 시간을 투자할 가치가 있습니다 왜냐하면..." "...에 실패했습니다"가 아님
- 팀원을 서로 부정적으로 비교하지 마세요. 각 사람의 섹션은 독립적입니다.
- 총 출력 약 3000-4500단어 유지 (팀 섹션을 수용하기 위해 약간 더 길게)
- 데이터에는 마크다운 테이블과 코드 블록, 내러티브에는 산문 사용
- 대화에 직접 출력 — 파일 시스템에 작성하지 않음 (`.context/retros/` JSON 스냅샷 제외)

## 중요 규칙

- 모든 내러티브 출력은 대화에서 사용자에게 직접 전달됩니다. 작성되는 유일한 파일은 `.context/retros/` JSON 스냅샷입니다.
- 모든 git 쿼리에 `origin/<default>`를 사용합니다 (오래된 로컬 main이 아닌)
- 모든 타임스탬프는 사용자의 로컬 시간대로 표시합니다 (`TZ`를 오버라이드하지 마세요)
- 범위에 커밋이 없으면 그렇게 말하고 다른 범위를 제안합니다
- LOC/시간을 50 단위로 반올림합니다
- 머지 커밋을 PR 경계로 취급합니다
- CLAUDE.md나 다른 문서를 읽지 마세요 — 이 스킬은 자체 완결적입니다
- 첫 실행 시 (이전 회고 없음), 비교 섹션을 우아하게 건너뜁니다
- **글로벌 모드:** git 저장소 내에 있을 필요가 없습니다. 스냅샷을 `~/.gstack/retros/`에 저장합니다 (`.context/retros/`가 아닌). 설치되지 않은 AI 도구는 우아하게 건너뜁니다. 동일한 범위 값을 가진 이전 글로벌 회고와만 비교합니다. 스트릭이 365일 한도에 도달하면 "365+ days"로 표시합니다.
