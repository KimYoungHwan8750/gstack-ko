# gstack development

## Commands

```bash
bun install          # install dependencies
bun test             # run free tests (browse + snapshot + skill validation)
bun run test:evals   # run paid evals: LLM judge + E2E (diff-based, ~$4/run max)
bun run test:evals:all  # run ALL paid evals regardless of diff
bun run test:gate    # run gate-tier tests only (CI default, blocks merge)
bun run test:periodic  # run periodic-tier tests only (weekly cron / manual)
bun run test:e2e     # run E2E tests only (diff-based, ~$3.85/run max)
bun run test:e2e:all # run ALL E2E tests regardless of diff
bun run eval:select  # show which tests would run based on current diff
bun run dev <cmd>    # run CLI in dev mode, e.g. bun run dev goto https://example.com
bun run build        # gen docs + compile binaries
bun run gen:skill-docs  # regenerate SKILL.md files from templates
bun run skill:check  # health dashboard for all skills
bun run dev:skill    # watch mode: auto-regen + validate on change
bun run eval:list    # list all eval runs from ~/.gstack-dev/evals/
bun run eval:compare # compare two eval runs (auto-picks most recent)
bun run eval:summary # aggregate stats across all eval runs
```

`test:evals`에는 `ANTHROPIC_API_KEY`가 필요합니다. Codex E2E tests(`test/codex-e2e.test.ts`)는
`~/.codex/` config의 Codex 자체 auth를 사용합니다. `OPENAI_API_KEY` env var는 필요하지 않습니다.
E2E tests는 실시간으로 progress를 stream합니다(tool-by-tool via `--output-format stream-json
--verbose`). Results는 `~/.gstack-dev/evals/`에 저장되며 이전 run과 자동 비교됩니다.

**Diff-based test selection:** `test:evals`와 `test:e2e`는 base branch에 대한
`git diff`를 기준으로 tests를 자동 선택합니다. 각 test는 file dependencies를
`test/helpers/touchfiles.ts`에 선언합니다. global touchfiles(session-runner, eval-store,
touchfiles.ts 자체)를 변경하면 모든 tests가 trigger됩니다. 모든 tests를 강제로 실행하려면
`EVALS_ALL=1` 또는 `:all` script variants를 사용하세요. 어떤 tests가 실행될지 미리 보려면
`eval:select`를 실행하세요.

**Two-tier system:** Tests는 `E2E_TIERS`(`test/helpers/touchfiles.ts`)에서 `gate` 또는
`periodic`으로 분류됩니다. CI는 gate tests만 실행합니다(`EVALS_TIER=gate`).
periodic tests는 weekly cron 또는 manual로 실행됩니다. filter하려면 `EVALS_TIER=gate` 또는
`EVALS_TIER=periodic`을 사용하세요. 새 E2E tests를 추가할 때는 다음처럼 분류하세요:
1. Safety guardrail 또는 deterministic functional test? -> `gate`
2. Quality benchmark, Opus model test, 또는 non-deterministic? -> `periodic`
3. External service(Codex, Gemini)가 필요한가? -> `periodic`

## Testing

```bash
bun test             # run before every commit — free, <2s
bun run test:evals   # run before shipping — paid, diff-based (~$4/run max)
```

`bun test`는 skill validation, gen-skill-docs quality checks, browse integration tests를
실행합니다. `bun run test:evals`는 `claude -p`를 통해 LLM-judge quality evals와 E2E tests를
실행합니다. PR을 만들기 전에 둘 다 통과해야 합니다.

## Project structure

```
gstack/
├── browse/          # Headless browser CLI (Playwright)
│   ├── src/         # CLI + server + commands
│   │   ├── commands.ts  # Command registry (single source of truth)
│   │   └── snapshot.ts  # SNAPSHOT_FLAGS metadata array
│   ├── test/        # Integration tests + fixtures
│   └── dist/        # Compiled binary
├── hosts/           # Typed host configs (one per AI agent)
│   ├── claude.ts    # Primary host config
│   ├── codex.ts, factory.ts, kiro.ts  # Existing hosts
│   ├── opencode.ts, slate.ts, cursor.ts, openclaw.ts  # New hosts
│   └── index.ts     # Registry: exports all, derives Host type
├── scripts/         # Build + DX tooling
│   ├── gen-skill-docs.ts  # Template → SKILL.md generator (config-driven)
│   ├── host-config.ts     # HostConfig interface + validator
│   ├── host-config-export.ts  # Shell bridge for setup script
│   ├── host-adapters/     # Host-specific adapters (OpenClaw tool mapping)
│   ├── resolvers/   # Template resolver modules (preamble, design, review, etc.)
│   ├── skill-check.ts     # Health dashboard
│   └── dev-skill.ts       # Watch mode
├── test/            # Skill validation + eval tests
│   ├── helpers/     # skill-parser.ts, session-runner.ts, llm-judge.ts, eval-store.ts
│   ├── fixtures/    # Ground truth JSON, planted-bug fixtures, eval baselines
│   ├── skill-validation.test.ts  # Tier 1: static validation (free, <1s)
│   ├── gen-skill-docs.test.ts    # Tier 1: generator quality (free, <1s)
│   ├── skill-llm-eval.test.ts   # Tier 3: LLM-as-judge (~$0.15/run)
│   └── skill-e2e-*.test.ts       # Tier 2: E2E via claude -p (~$3.85/run, split by category)
├── qa-only/         # /qa-only skill (report-only QA, no fixes)
├── plan-design-review/  # /plan-design-review skill (report-only design audit)
├── design-review/    # /design-review skill (design audit + fix loop)
├── ship/            # Ship workflow skill
├── review/          # PR review skill
├── plan-ceo-review/ # /plan-ceo-review skill
├── plan-eng-review/ # /plan-eng-review skill
├── autoplan/        # /autoplan skill (auto-review pipeline: CEO → design → eng)
├── benchmark/       # /benchmark skill (performance regression detection)
├── canary/          # /canary skill (post-deploy monitoring loop)
├── codex/           # /codex skill (multi-AI second opinion via OpenAI Codex CLI)
├── land-and-deploy/ # /land-and-deploy skill (merge → deploy → canary verify)
├── office-hours/    # /office-hours skill (YC Office Hours — startup diagnostic + builder brainstorm)
├── investigate/     # /investigate skill (systematic root-cause debugging)
├── retro/           # Retrospective skill (includes /retro global cross-project mode)
├── bin/             # CLI utilities (gstack-repo-mode, gstack-slug, gstack-config, etc.)
├── document-release/ # /document-release skill (post-ship doc updates)
├── cso/             # /cso skill (OWASP Top 10 + STRIDE security audit)
├── design-consultation/ # /design-consultation skill (design system from scratch)
├── design-shotgun/  # /design-shotgun skill (visual design exploration)
├── open-gstack-browser/  # /open-gstack-browser skill (launch GStack Browser)
├── connect-chrome/  # symlink → open-gstack-browser (backwards compat)
├── design/          # Design binary CLI (GPT Image API)
│   ├── src/         # CLI + commands (generate, variants, compare, serve, etc.)
│   ├── test/        # Integration tests
│   └── dist/        # Compiled binary
├── extension/       # Chrome extension (side panel + activity feed + CSS inspector)
├── lib/             # Shared libraries (worktree.ts)
├── docs/designs/    # Design documents
├── setup-deploy/    # /setup-deploy skill (one-time deploy config)
├── .github/         # CI workflows + Docker image
│   ├── workflows/   # evals.yml (E2E on Ubicloud), skill-docs.yml, actionlint.yml
│   └── docker/      # Dockerfile.ci (pre-baked toolchain + Playwright/Chromium)
├── contrib/         # Contributor-only tools (never installed for users)
│   └── add-host/    # /gstack-contrib-add-host skill
├── setup            # One-time setup: build binary + symlink skills
├── SKILL.md         # Generated from SKILL.md.tmpl (don't edit directly)
├── SKILL.md.tmpl    # Template: edit this, run gen:skill-docs
├── ETHOS.md         # Builder philosophy (Boil the Lake, Search Before Building)
└── package.json     # Build scripts for browse
```

## SKILL.md workflow

SKILL.md files는 `.tmpl` templates에서 **generated**됩니다. docs를 update하려면:

1. `.tmpl` file을 edit합니다(e.g. `SKILL.md.tmpl` or `browse/SKILL.md.tmpl`)
2. `bun run gen:skill-docs`를 실행합니다(또는 자동으로 실행하는 `bun run build`)
3. `.tmpl`와 generated `.md` files를 모두 commit합니다

새 browse command를 추가하려면 `browse/src/commands.ts`에 추가하고 rebuild하세요.
snapshot flag를 추가하려면 `browse/src/snapshot.ts`의 `SNAPSHOT_FLAGS`에 추가하고 rebuild하세요.

**Merge conflicts on SKILL.md files:** generated SKILL.md files의 conflict는 절대 한쪽을
accept해서 해결하지 마세요. 대신 (1) source of truth인 `.tmpl` templates와
`scripts/gen-skill-docs.ts`의 conflicts를 resolve하고, (2) `bun run gen:skill-docs`를 실행해
모든 SKILL.md files를 regenerate하고, (3) regenerated files를 stage하세요. 한쪽 generated output을
accept하면 다른 쪽 template changes가 조용히 drop됩니다.

## Platform-agnostic design

Skills는 framework-specific commands, file patterns, directory structures를 절대 hardcode하면
안 됩니다. 대신:

1. project-specific config(test commands, eval commands, etc.)를 위해 **CLAUDE.md를 읽기**
2. **없으면 AskUserQuestion** — user가 알려주게 하거나 gstack이 repo를 search하게 하기
3. **답을 CLAUDE.md에 persist**해서 다시 묻지 않게 하기

이는 test commands, eval commands, deploy commands, 기타 project-specific behavior 모두에
적용됩니다. project가 config를 소유하고, gstack은 그것을 읽습니다.

## Writing SKILL templates

SKILL.md.tmpl files는 bash scripts가 아니라 **Claude가 읽는 prompt templates**입니다.
각 bash code block은 별도 shell에서 실행됩니다. variables는 blocks 사이에 persist되지 않습니다.

Rules:
- **logic과 state에는 natural language를 사용하세요.** code blocks 사이에 state를 넘기려고
  shell variables를 쓰지 마세요. 대신 Claude가 무엇을 기억해야 하는지 prose로 말하고,
  prose에서 참조하세요(e.g., "the base branch detected in Step 0").
- **branch names를 hardcode하지 마세요.** `gh pr view` 또는 `gh repo view`로 `main`/`master`/etc를
  dynamic하게 detect하세요. PR-targeting skills에는 `{{BASE_BRANCH_DETECT}}`를 사용하세요.
  prose에서는 "the base branch", code block placeholders에서는 `<base>`를 사용하세요.
- **bash blocks는 self-contained로 유지하세요.** 각 code block은 독립적으로 동작해야 합니다.
  block이 이전 step의 context를 필요로 하면 위 prose에서 다시 설명하세요.
- **conditionals는 English로 표현하세요.** bash에서 nested `if/elif/else`를 쓰는 대신,
  numbered decision steps로 작성하세요: "1. If X, do Y. 2. Otherwise, do Z."

## Browser interaction

browser와 interact해야 할 때(QA, dogfooding, cookie setup)는 `/browse` skill을 사용하거나
browse binary를 `$B <command>`로 직접 실행하세요. `mcp__claude-in-chrome__*` tools는 절대
사용하지 마세요. 이들은 느리고 unreliable하며 이 project가 사용하는 방식이 아닙니다.

**Sidebar architecture:** `sidepanel.js`, `background.js`, `content.js`, `sidebar-agent.ts`,
또는 sidebar-related server endpoints를 수정하기 전에 `docs/designs/SIDEBAR_MESSAGE_FLOW.md`를
읽으세요. 이 문서는 full initialization timeline, message flow, auth token chain, tab
concurrency model, known failure modes를 설명합니다. sidebar는 2개 codebases(extension + server)에
걸친 5개 files로 구성되어 있고, ordering dependencies가 non-obvious합니다. 이 문서는
cross-component flow를 이해하지 못해서 생기는 silent failures를 막기 위해 존재합니다.

## Dev symlink awareness

gstack을 development할 때 `.claude/skills/gstack`이 이 working directory(gitignored)로 돌아오는
symlink일 수 있습니다. 이는 skill changes가 **즉시 live**가 된다는 뜻입니다. rapid iteration에는
좋지만, half-written skills가 gstack을 동시에 사용하는 다른 Claude Code sessions를 깨뜨릴 수 있는
big refactors에서는 위험합니다.

**Check once per session:** symlink인지 real copy인지 확인하려면 `ls -la .claude/skills/gstack`를
실행하세요. working directory로 향하는 symlink라면 다음을 유의하세요:
- Template changes + `bun run gen:skill-docs`는 모든 gstack invocations에 즉시 영향을 줍니다
- SKILL.md.tmpl files의 breaking changes는 concurrent gstack sessions를 깨뜨릴 수 있습니다
- large refactors 중에는 symlink를 제거해(`rm .claude/skills/gstack`) 대신
  `~/.claude/skills/gstack/`의 global install을 사용하게 하세요

**Prefix setting:** Setup은 top level에 real directories(not symlinks)를 만들고 그 안에
SKILL.md symlink를 둡니다(e.g., `qa/SKILL.md -> gstack/qa/SKILL.md`). 이렇게 해야 Claude가
이들을 `gstack/` 아래 nested가 아니라 top-level skills로 discover합니다. 이름은 short(`qa`) 또는
namespaced(`gstack-qa`)이며, `~/.gstack/config.yaml`의 `skill_prefix`로 제어됩니다.
interactive prompt를 skip하려면 `--no-prefix` 또는 `--prefix`를 pass하세요.

**Note:** gstack을 project repo에 vendoring하는 것은 deprecated입니다. 대신 global install +
`./setup --team`을 사용하세요. team mode instructions는 README.md를 보세요.

**For plan reviews:** skill templates 또는 gen-skill-docs pipeline을 수정하는 plans를 review할 때는,
live로 가기 전에 changes를 isolation에서 test해야 하는지 고려하세요(특히 user가 다른 windows에서
gstack을 활발히 사용 중이라면).

**Upgrade migrations:** change가 existing user installs를 깨뜨릴 수 있는 방식으로 on-disk state
(directory structure, config format, stale files)를 수정한다면 `gstack-upgrade/migrations/`에
migration script를 추가하세요. format과 testing requirements는 CONTRIBUTING.md의
"Upgrade migrations" section을 읽으세요. upgrade skill은 `/gstack-upgrade` 중 `./setup` 후에
이를 자동 실행합니다.

## Compiled binaries — NEVER commit browse/dist/ or design/dist/

`browse/dist/`와 `design/dist/` directories에는 compiled Bun binaries(`browse`, `find-browse`,
`design`, ~58MB each)가 들어 있습니다. 이들은 Mach-O arm64 only입니다. Linux, Windows, Intel Macs에서
동작하지 않습니다. `./setup` script가 이미 모든 platform에서 source로 build하므로 checked-in binaries는
redundant합니다. historical mistake 때문에 git이 tracking하고 있으며, eventually `git rm --cached`로
제거되어야 합니다.

**NEVER stage or commit these files.** `.gitignore`에도 불구하고 tracked 상태라 `git status`에
modified로 나타납니다. 무시하세요. files를 staging할 때는 항상 specific filenames를 사용하세요
(`git add file1 file2`). binaries를 accidentally include하게 되는 `git add .` 또는 `git add -A`는
절대 사용하지 마세요.

## Commit style

**Always bisect commits.** 모든 commit은 single logical change여야 합니다. 여러 changes를 만들었다면
(e.g., rename + rewrite + new tests), push하기 전에 별도 commits로 split하세요. 각 commit은
independently understandable and revertable해야 합니다.

Examples of good bisection:
- behavior changes와 분리된 Rename/move
- test implementations와 분리된 Test infrastructure(touchfiles, helpers)
- generated file regeneration과 분리된 Template changes
- new features와 분리된 Mechanical refactors

user가 "bisect commit" 또는 "bisect and push"라고 말하면 staged/unstaged changes를 logical commits로
split하고 push하세요.

## Community PR guardrails

community PRs를 review하거나 merge할 때, 다음 commit을 accept하기 전에는 **항상 AskUserQuestion**을
사용하세요:

1. **ETHOS.md를 touch** — 이 file은 Garry의 personal builder philosophy입니다. external contributors
   또는 AI agents의 edits는 절대 허용하지 않습니다.
2. **promotional material을 제거하거나 soften** — YC references, founder perspective, product voice는
   intentional입니다. 이를 "unnecessary" 또는 "too promotional"로 frame하는 PRs는 reject해야 합니다.
3. **Garry's voice를 변경** — skill templates, CHANGELOG, docs의 tone, humor, directness, perspective는
   generic이 아닙니다. voice를 더 "neutral" 또는 "professional"하게 rewrite하는 PRs는 reject해야 합니다.

agent가 어떤 change가 project를 개선한다고 강하게 믿더라도, 이 세 category는 AskUserQuestion을 통한
explicit user approval이 필요합니다. No exceptions. No auto-merging. No "I'll just clean this up."

## CHANGELOG + VERSION style

**VERSION and CHANGELOG are branch-scoped.** shipping되는 모든 feature branch는 자체 version bump와
CHANGELOG entry를 갖습니다. entry는 THIS branch가 추가하는 것을 설명합니다. main에 이미 있던 것이
아닙니다.

**When to write the CHANGELOG entry:**
- development 또는 mid-branch가 아니라 `/ship` time(Step 5)에 작성합니다.
- entry는 base branch 대비 이 branch의 ALL commits를 다룹니다.
- 이미 main에 landed된 prior version의 existing CHANGELOG entry에 new work를 fold하지 마세요.
  main에 v0.10.0.0이 있고 branch가 features를 추가한다면, v0.10.1.0으로 bump하고 new entry를 만드세요.
  v0.10.0.0 entry를 edit하지 마세요.

**Key questions before writing:**
1. What branch am I on? What did THIS branch change?
2. Is the base branch version already released? (If yes, bump and create new entry.)
3. Does an existing entry on this branch already cover earlier work? (If yes, replace
   it with one unified entry for the final version.)

**Merging main does NOT mean adopting main's version.** origin/main을 feature branch에 merge하면
main이 new CHANGELOG entries와 higher VERSION을 가져올 수 있습니다. 그래도 branch는 그 위에 자체
version bump가 필요합니다. main이 v0.13.8.0이고 branch가 features를 추가한다면 v0.13.9.0으로 bump하고
new entry를 만드세요. 이미 main에 landed된 entry에 your changes를 jam하지 마세요. your entry는 branch가
다음에 land되므로 top에 들어갑니다.

**After merging main, always check:**
- CHANGELOG에 main entries와 분리된 branch 자체 entry가 있는가?
- VERSION이 main's VERSION보다 높은가?
- your entry가 CHANGELOG의 topmost entry인가(main's latest보다 위)?
하나라도 no라면 계속하기 전에 fix하세요.

**After any CHANGELOG edit that moves, adds, or removes entries,** 즉시
`grep "^## \[" CHANGELOG.md`를 실행하고 full version sequence가 gaps 또는 duplicates 없이 contiguous한지
verify하세요. version이 missing이라면 edit가 무언가를 깨뜨린 것입니다. 계속하기 전에 fix하세요.

CHANGELOG.md는 contributors가 아니라 **users**를 위한 것입니다. product release notes처럼 작성하세요:

- user가 이제 전에는 못 하던 무엇을 **할 수 있는지**로 시작하세요. feature를 sell하세요.
- implementation details가 아니라 plain language를 사용하세요. "Refactored the..."가 아니라 "You can now..."
- **TODOS.md, internal tracking, eval infrastructure, contributor-facing details는 절대 언급하지 마세요.**
  이는 users에게 보이지 않고 의미도 없습니다.
- contributor/internal changes는 bottom의 별도 "For contributors" section에 넣으세요.
- 모든 entry는 누군가가 "oh nice, I want to try that."라고 생각하게 해야 합니다.
- No jargon: "AskUserQuestion format standardized across skill templates via preamble resolver."가 아니라
  "every question now tells you which project and branch you're in"라고 쓰세요.

## AI effort compression

effort를 estimate하거나 discuss할 때는 항상 human-team time과 CC+gstack time을 모두 보여주세요:

| Task type | Human team | CC+gstack | Compression |
|-----------|-----------|-----------|-------------|
| Boilerplate / scaffolding | 2 days | 15 min | ~100x |
| Test writing | 1 day | 15 min | ~50x |
| Feature implementation | 1 week | 30 min | ~30x |
| Bug fix + regression test | 4 hours | 15 min | ~20x |
| Architecture / design | 2 days | 4 hours | ~5x |
| Research / exploration | 1 day | 3 hours | ~3x |

Completeness는 cheap합니다. complete implementation이 "ocean"(multi-quarter migration)이 아니라
"lake"(achievable)라면 shortcuts를 recommend하지 마세요. full philosophy는 skill preamble의
Completeness Principle을 보세요.

## Search before building

concurrency, unfamiliar patterns, infrastructure, 또는 runtime/framework에 built-in이 있을 수 있는
어떤 solution을 design하기 전에는:

1. Search for "{runtime} {thing} built-in"
2. Search for "{thing} best practice {current year}"
3. Check official runtime/framework docs

knowledge에는 세 layers가 있습니다: tried-and-true(Layer 1), new-and-popular(Layer 2),
first-principles(Layer 3). Layer 3를 무엇보다 높게 평가하세요. full builder philosophy는 ETHOS.md를
보세요.

## Local plans

Contributors는 long-range vision docs와 design documents를 `~/.gstack-dev/plans/`에 저장할 수 있습니다.
이들은 local-only입니다(not checked in). TODOS.md를 review할 때 TODOs로 promote하거나 implement할 준비가
되었을 수 있는 candidates를 `plans/`에서 확인하세요.

## E2E eval failure blame protocol

`/ship` 또는 다른 workflow 중 E2E eval이 fail하면, 증명 없이 **절대 "not related to our changes"라고
claim하지 마세요.** 이 systems에는 invisible couplings가 있습니다. preamble text change가 agent behavior에
영향을 주고, new helper가 timing을 바꾸며, regenerated SKILL.md가 prompt context를 shift합니다.

**"pre-existing"으로 failure를 attribute하기 전에 required:**
1. main(or base branch)에서 같은 eval을 실행하고 거기서도 fail한다는 것을 보여주세요
2. main에서는 pass하지만 branch에서는 fail한다면 — 그것은 your change입니다. blame을 trace하세요.
3. main에서 실행할 수 없다면 "unverified — may or may not be related"라고 말하고 PR body에 risk로 flag하세요

증거 없는 "Pre-existing"은 lazy claim입니다. 증명하거나, 말하지 마세요.

## Long-running tasks: don't give up

evals, E2E tests, 또는 long-running background task를 실행할 때는 **completion까지 poll**하세요.
3분마다 loop로 `sleep 180 && echo "ready"` + `TaskOutput`을 사용하세요. poll timeout이 났다고
blocking mode로 전환하고 포기하지 마세요. "I'll be notified when it completes"라고 말하고 checking을
멈추지 마세요. task가 끝나거나 user가 stop하라고 할 때까지 loop를 계속하세요.

full E2E suite는 30-45 minutes가 걸릴 수 있습니다. 이는 10-15 polling cycles입니다. 전부 하세요.
각 check마다 progress를 report하세요(which tests passed, which are running, any failures so far).
user가 원하는 것은 나중에 확인하겠다는 약속이 아니라 run이 complete되는 것을 보는 것입니다.

## E2E test fixtures: extract, don't copy

**NEVER copy a full SKILL.md file into an E2E test fixture.** SKILL.md files는
1500-2000 lines입니다. `claude -p`가 이렇게 큰 file을 읽으면 context bloat 때문에 timeouts, flaky turn
limits, 그리고 필요한 것보다 5-10x 오래 걸리는 tests가 생깁니다.

대신 test가 실제로 필요한 section만 extract하세요:

```typescript
// BAD — agent reads 1900 lines, burns tokens on irrelevant sections
fs.copyFileSync(path.join(ROOT, 'ship', 'SKILL.md'), path.join(dir, 'ship-SKILL.md'));

// GOOD — agent reads ~60 lines, finishes in 38s instead of timing out
const full = fs.readFileSync(path.join(ROOT, 'ship', 'SKILL.md'), 'utf-8');
const start = full.indexOf('## Review Readiness Dashboard');
const end = full.indexOf('\n---\n', start);
fs.writeFileSync(path.join(dir, 'ship-SKILL.md'), full.slice(start, end > start ? end : undefined));
```

targeted E2E tests를 실행해 failures를 debug할 때도:
- background with `&` and `tee`가 아니라 **foreground**에서 실행하세요(`bun test ...`)
- running eval processes를 `pkill`하고 restart하지 마세요. results를 잃고 money를 낭비합니다
- One clean run beats three killed-and-restarted runs

## Publishing native OpenClaw skills to ClawHub

Native OpenClaw skills는 `openclaw/skills/gstack-openclaw-*/SKILL.md`에 있습니다. 이들은 ClawHub에
publish되는 hand-crafted methodology skills(generated by the pipeline이 아님)이며, 모든 OpenClaw user가
install할 수 있습니다.

**Publishing:** command는 `clawhub publish`입니다(`clawhub skill publish`가 아님):

```bash
clawhub publish openclaw/skills/gstack-openclaw-office-hours \
  --slug gstack-openclaw-office-hours --name "gstack Office Hours" \
  --version 1.0.0 --changelog "description of changes"
```

각 skill에 대해 반복하세요: `gstack-openclaw-ceo-review`, `gstack-openclaw-investigate`,
`gstack-openclaw-retro`. update할 때마다 `--version`을 bump하세요.

**Auth:** `clawhub login`(GitHub auth를 위해 browser를 엽니다). verify하려면 `clawhub whoami`.

**Updating:** 더 높은 `--version`과 `--changelog`로 같은 `clawhub publish` command를 사용하세요.

**Verification:** live인지 confirm하려면 `clawhub search gstack`.

## Deploying to the active skill

active skill은 `~/.claude/skills/gstack/`에 있습니다. changes를 만든 후:

1. branch를 push하세요
2. skill directory에서 fetch and reset하세요: `cd ~/.claude/skills/gstack && git fetch origin && git reset --hard origin/main`
3. Rebuild하세요: `cd ~/.claude/skills/gstack && bun run build`

또는 binaries를 직접 copy하세요:
- `cp browse/dist/browse ~/.claude/skills/gstack/browse/dist/browse`
- `cp design/dist/design ~/.claude/skills/gstack/design/dist/design`
