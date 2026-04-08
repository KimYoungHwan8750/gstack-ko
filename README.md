# gstack

> "I don't think I've typed like a line of code probably since December, basically, which is an extremely large change." — [Andrej Karpathy](https://fortune.com/2026/03/21/andrej-karpathy-openai-cofounder-ai-agents-coding-state-of-psychosis-openclaw/), No Priors podcast, March 2026

Karpathy가 이렇게 말하는 걸 듣고, 저는 어떻게 가능한지 알고 싶었습니다. 한 사람이 어떻게 스무 명짜리 팀처럼 ship할 수 있을까? Peter Steinberger는 [OpenClaw](https://github.com/openclaw/openclaw)를 만들었습니다. GitHub star 247K개짜리 프로젝트를 AI agents와 함께 사실상 혼자서요. 혁명은 이미 와 있습니다. 올바른 tooling을 가진 한 명의 builder는 전통적인 팀보다 더 빠르게 움직일 수 있습니다.

저는 [Y Combinator](https://www.ycombinator.com/)의 President & CEO인 [Garry Tan](https://x.com/garrytan)입니다. 저는 Coinbase, Instacart, Rippling 같은 수천 개의 startup과 함께 일했습니다. 그들이 차고에서 한두 명으로 시작했을 때부터요. YC 전에는 Palantir의 초기 eng/PM/designer 중 한 명이었고, Posterous를 공동 창업해 Twitter에 매각했으며, YC의 내부 소셜 네트워크인 Bookface를 만들었습니다.

**gstack is my answer.** 저는 20년 동안 제품을 만들어 왔고, 지금은 그 어느 때보다 많은 code를 ship하고 있습니다. 지난 60일 동안: **production code 600,000+ lines** (35% tests), **하루 10,000-20,000 lines**, YC를 full-time으로 운영하면서 part-time으로요. 3개 프로젝트에 걸친 제 최근 `/retro`는 이랬습니다: 일주일에 **140,751 lines added, 362 commits, ~115k net LOC**.

**2026 — 1,237 contributions and counting:**

![GitHub contributions 2026 — 1,237 contributions, massive acceleration in Jan-Mar](docs/images/github-2026.png)

**2013 — when I built Bookface at YC (772 contributions):**

![GitHub contributions 2013 — 772 contributions building Bookface at YC](docs/images/github-2013.png)

같은 사람. 다른 시대. 차이는 tooling입니다.

**gstack is how I do it.** gstack은 Claude Code를 virtual engineering team으로 바꿉니다. 제품을 다시 생각하는 CEO, architecture를 고정하는 eng manager, AI slop을 잡아내는 designer, production bug를 찾는 reviewer, 실제 browser를 여는 QA lead, OWASP + STRIDE audit을 실행하는 security officer, PR을 ship하는 release engineer까지. 스물세 명의 specialists와 여덟 개의 power tools, 모두 slash commands, 모두 Markdown, 모두 free, MIT license입니다.

이것이 제 open source software factory입니다. 저는 매일 사용합니다. 이런 tools는 모두가 사용할 수 있어야 한다고 생각해서 공유합니다.

Fork하세요. 개선하세요. 여러분의 것으로 만드세요. 그리고 free open source software를 비난하고 싶다면 그렇게 해도 됩니다. 하지만 먼저 한번 써보는 편이 더 좋겠습니다.

**Who this is for:**
- **Founders and CEOs** — 특히 여전히 직접 ship하고 싶은 technical founders
- **First-time Claude Code users** — 빈 prompt 대신 구조화된 roles
- **Tech leads and staff engineers** — 모든 PR에 rigorous review, QA, release automation

## Quick start

1. gstack 설치 (30초 — 아래 참고)
2. `/office-hours` 실행 — 만들고 있는 것을 설명하세요
3. 어떤 feature idea든 `/plan-ceo-review` 실행
4. 변경 사항이 있는 branch에서 `/review` 실행
5. staging URL에서 `/qa` 실행
6. 여기서 멈추세요. 이게 여러분에게 맞는지 알게 됩니다.

## Install — 30 seconds

**Requirements:** [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Git](https://git-scm.com/), [Bun](https://bun.sh/) v1.0+, [Node.js](https://nodejs.org/) (Windows only)

### Step 1: Install on your machine

Claude Code를 열고 이것을 붙여넣으세요. 나머지는 Claude가 처리합니다.

> Install gstack: run **`git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup`** then add a "gstack" section to CLAUDE.md that says to use the /browse skill from gstack for all web browsing, never use mcp\_\_claude-in-chrome\_\_\* tools, and lists the available skills: /office-hours, /plan-ceo-review, /plan-eng-review, /plan-design-review, /design-consultation, /design-shotgun, /design-html, /review, /ship, /land-and-deploy, /canary, /benchmark, /browse, /connect-chrome, /qa, /qa-only, /design-review, /setup-browser-cookies, /setup-deploy, /retro, /investigate, /document-release, /codex, /cso, /autoplan, /plan-devex-review, /devex-review, /careful, /freeze, /guard, /unfreeze, /gstack-upgrade, /learn. Then ask the user if they also want to add gstack to the current project so teammates get it.

### Step 2: Team mode — shared repos를 위한 auto-update (권장)

모든 developer가 global로 설치하고, update는 자동으로 이루어집니다:

```bash
cd ~/.claude/skills/gstack && ./setup --team
```

그다음 repo를 bootstrap해서 teammates도 사용하게 하세요:

```bash
cd <your-repo>
~/.claude/skills/gstack/bin/gstack-team-init required  # or: optional
git add .claude/ CLAUDE.md && git commit -m "require gstack for AI-assisted work"
```

repo에 vendored files가 들어가지 않고, version drift도 없고, manual upgrade도 없습니다. 모든 Claude Code session은 빠른 auto-update check로 시작합니다. 이 check는 한 시간에 한 번으로 throttle되고, network failure에도 안전하며, 완전히 silent합니다.

> **Contributing하거나 full history가 필요하신가요?** 위 commands는 빠른 install을 위해 `--depth 1`을 사용합니다. contribute할 계획이 있거나 full git history가 필요하다면 full clone을 하세요:
> ```bash
> git clone https://github.com/garrytan/gstack.git ~/.claude/skills/gstack
> ```

### OpenClaw

OpenClaw는 ACP를 통해 Claude Code sessions를 spawn하므로, Claude Code에 gstack이 설치되어 있으면 모든 gstack skill이 그대로 작동합니다. 이것을 OpenClaw agent에 붙여넣으세요:

> Install gstack: run `git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup` to install gstack for Claude Code. Then add a "Coding Tasks" section to AGENTS.md that says: when spawning Claude Code sessions for coding work, tell the session to use gstack skills. Include these examples — security audit: "Load gstack. Run /cso", code review: "Load gstack. Run /review", QA test a URL: "Load gstack. Run /qa https://...", build a feature end-to-end: "Load gstack. Run /autoplan, implement the plan, then run /ship", plan before building: "Load gstack. Run /office-hours then /autoplan. Save the plan, don't implement."

**Setup 후에는 OpenClaw agent와 자연스럽게 대화하면 됩니다:**

| You say | What happens |
|---------|-------------|
| "Fix the typo in README" | 단순 작업 — Claude Code session, gstack 필요 없음 |
| "Run a security audit on this repo" | `Run /cso`로 Claude Code를 spawn |
| "Build me a notifications feature" | /autoplan → implement → /ship으로 Claude Code를 spawn |
| "Help me plan the v2 API redesign" | /office-hours → /autoplan으로 Claude Code를 spawn하고 plan 저장 |

Advanced dispatch routing과 gstack-lite/gstack-full prompt templates는 [docs/OPENCLAW.md](docs/OPENCLAW.md)를 참고하세요.

### Native OpenClaw Skills (via ClawHub)

Claude Code session 없이 OpenClaw agent에서 직접 작동하는 네 가지 methodology skills입니다. ClawHub에서 설치하세요:

```
clawhub install gstack-openclaw-office-hours gstack-openclaw-ceo-review gstack-openclaw-investigate gstack-openclaw-retro
```

| Skill | What it does |
|-------|-------------|
| `gstack-openclaw-office-hours` | 6가지 forcing questions로 product interrogation |
| `gstack-openclaw-ceo-review` | 4가지 scope modes로 strategic challenge |
| `gstack-openclaw-investigate` | Root cause debugging methodology |
| `gstack-openclaw-retro` | Weekly engineering retrospective |

이것들은 conversational skills입니다. OpenClaw agent가 chat을 통해 직접 실행합니다.

### Other AI Agents

gstack은 Claude뿐 아니라 8개의 AI coding agents에서 작동합니다. Setup은 설치된 agent를 자동 감지합니다:

```bash
git clone --single-branch --depth 1 https://github.com/garrytan/gstack.git ~/gstack
cd ~/gstack && ./setup
```

또는 `./setup --host <name>`으로 특정 agent를 지정하세요:

| Agent | Flag | Skills install to |
|-------|------|-------------------|
| OpenAI Codex CLI | `--host codex` | `~/.codex/skills/gstack-*/` |
| OpenCode | `--host opencode` | `~/.config/opencode/skills/gstack-*/` |
| Cursor | `--host cursor` | `~/.cursor/skills/gstack-*/` |
| Factory Droid | `--host factory` | `~/.factory/skills/gstack-*/` |
| Slate | `--host slate` | `~/.slate/skills/gstack-*/` |
| Kiro | `--host kiro` | `~/.kiro/skills/gstack-*/` |

**다른 agent support를 추가하고 싶으신가요?** [docs/ADDING_A_HOST.md](docs/ADDING_A_HOST.md)를 참고하세요. TypeScript config file 하나면 되고, code changes는 없습니다.

## See it work

```
You:    I want to build a daily briefing app for my calendar.
You:    /office-hours
Claude: [asks about the pain — specific examples, not hypotheticals]

You:    Multiple Google calendars, events with stale info, wrong locations.
        Prep takes forever and the results aren't good enough...

Claude: I'm going to push back on the framing. You said "daily briefing
        app." But what you actually described is a personal chief of
        staff AI.
        [extracts 5 capabilities you didn't realize you were describing]
        [challenges 4 premises — you agree, disagree, or adjust]
        [generates 3 implementation approaches with effort estimates]
        RECOMMENDATION: Ship the narrowest wedge tomorrow, learn from
        real usage. The full vision is a 3-month project — start with
        the daily briefing that actually works.
        [writes design doc → feeds into downstream skills automatically]

You:    /plan-ceo-review
        [reads the design doc, challenges scope, runs 10-section review]

You:    /plan-eng-review
        [ASCII diagrams for data flow, state machines, error paths]
        [test matrix, failure modes, security concerns]

You:    Approve plan. Exit plan mode.
        [writes 2,400 lines across 11 files. ~8 minutes.]

You:    /review
        [AUTO-FIXED] 2 issues. [ASK] Race condition → you approve fix.

You:    /qa https://staging.myapp.com
        [opens real browser, clicks through flows, finds and fixes a bug]

You:    /ship
        Tests: 42 → 51 (+9 new). PR: github.com/you/app/pull/42
```

당신은 "daily briefing app"이라고 말했습니다. Agent는 "you're building a chief of staff AI"라고 말했습니다. feature request가 아니라 pain을 들었기 때문입니다. 여덟 개의 commands, end to end. 이것은 copilot이 아닙니다. 팀입니다.

## The sprint

gstack은 tools 모음이 아니라 process입니다. Skills는 sprint가 진행되는 순서대로 실행됩니다:

**Think → Plan → Build → Review → Test → Ship → Reflect**

각 skill은 다음 skill로 이어집니다. `/office-hours`는 `/plan-ceo-review`가 읽는 design doc을 씁니다. `/plan-eng-review`는 `/qa`가 집어 드는 test plan을 씁니다. `/review`는 `/ship`이 fix 여부를 verify하는 bug를 잡습니다. 모든 단계가 이전에 무엇이 있었는지 알고 있으므로 아무것도 틈새로 빠지지 않습니다.

| Skill | Your specialist | What they do |
|-------|----------------|--------------|
| `/office-hours` | **YC Office Hours** | 여기서 시작하세요. code를 쓰기 전에 product를 재구성하는 6가지 forcing questions. framing에 pushback하고, premises에 challenge하며, implementation alternatives를 생성합니다. Design doc은 모든 downstream skill로 이어집니다. |
| `/plan-ceo-review` | **CEO / Founder** | 문제를 다시 생각합니다. request 안에 숨어 있는 10-star product를 찾습니다. 네 가지 modes: Expansion, Selective Expansion, Hold Scope, Reduction. |
| `/plan-eng-review` | **Eng Manager** | architecture, data flow, diagrams, edge cases, tests를 고정합니다. 숨은 assumptions를 밖으로 끌어냅니다. |
| `/plan-design-review` | **Senior Designer** | 각 design dimension을 0-10으로 평가하고, 10이 어떤 모습인지 설명한 뒤, 거기에 도달하도록 plan을 수정합니다. AI Slop detection. Interactive — design choice마다 AskUserQuestion 하나. |
| `/plan-devex-review` | **Developer Experience Lead** | Interactive DX review: developer personas를 탐색하고, competitors의 TTHW와 benchmark하며, magical moment를 설계하고, friction points를 단계별로 추적합니다. 세 가지 modes: DX EXPANSION, DX POLISH, DX TRIAGE. 20-45 forcing questions. |
| `/design-consultation` | **Design Partner** | 완전한 design system을 처음부터 만듭니다. landscape를 research하고, creative risks를 제안하며, realistic product mockups를 생성합니다. |
| `/review` | **Staff Engineer** | CI는 통과하지만 production에서 터지는 bugs를 찾습니다. obvious한 것들은 auto-fix합니다. completeness gaps를 flag합니다. |
| `/investigate` | **Debugger** | 체계적인 root-cause debugging. Iron Law: investigation 없는 fix는 없습니다. data flow를 추적하고, hypotheses를 test하며, 3번 fix에 실패하면 멈춥니다. |
| `/design-review` | **Designer Who Codes** | /plan-design-review와 같은 audit을 한 뒤, 발견한 문제를 수정합니다. Atomic commits, before/after screenshots. |
| `/devex-review` | **DX Tester** | Live developer experience audit. onboarding을 실제로 test합니다: docs를 탐색하고, getting started flow를 시도하고, TTHW를 측정하고, errors를 screenshot합니다. `/plan-devex-review` scores와 비교합니다 — plan이 reality와 맞았는지 보여주는 boomerang입니다. |
| `/design-shotgun` | **Design Explorer** | "Show me options." 4-6개의 AI mockup variants를 생성하고, browser에서 comparison board를 열고, feedback을 수집한 뒤 iterate합니다. Taste memory가 당신이 좋아하는 것을 학습합니다. 마음에 드는 것이 나올 때까지 반복한 뒤 `/design-html`로 넘깁니다. |
| `/design-html` | **Design Engineer** | mockup을 실제로 작동하는 production HTML로 바꿉니다. Pretext computed layout: text가 reflow되고, heights가 조정되며, layouts가 dynamic합니다. 30KB, zero deps. React/Svelte/Vue를 감지합니다. design type(landing page vs dashboard vs form)에 따라 Smart API routing. output은 demo가 아니라 shippable입니다. |
| `/qa` | **QA Lead** | app을 test하고, bugs를 찾고, atomic commits로 고친 뒤 re-verify합니다. 모든 fix에 대해 regression tests를 auto-generate합니다. |
| `/qa-only` | **QA Reporter** | /qa와 같은 methodology지만 report only입니다. code changes 없는 순수 bug report입니다. |
| `/pair-agent` | **Multi-Agent Coordinator** | browser를 어떤 AI agent와도 공유합니다. 한 command, 한 번 paste, connected. OpenClaw, Hermes, Codex, Cursor, 또는 curl할 수 있는 무엇이든 작동합니다. 각 agent는 자기 tab을 얻습니다. headed mode를 auto-launch해서 모든 것을 볼 수 있습니다. remote agents를 위해 ngrok tunnel을 auto-start합니다. Scoped tokens, tab isolation, rate limiting, activity attribution. |
| `/cso` | **Chief Security Officer** | OWASP Top 10 + STRIDE threat model. Zero-noise: 17 false positive exclusions, 8/10+ confidence gate, independent finding verification. 각 finding에는 concrete exploit scenario가 포함됩니다. |
| `/ship` | **Release Engineer** | main sync, tests 실행, coverage audit, push, PR open. test framework가 없으면 bootstrap합니다. |
| `/land-and-deploy` | **Release Engineer** | PR을 merge하고, CI와 deploy를 기다리고, production health를 verify합니다. "approved"에서 "verified in production"까지 한 command. |
| `/canary` | **SRE** | Post-deploy monitoring loop. console errors, performance regressions, page failures를 감시합니다. |
| `/benchmark` | **Performance Engineer** | page load times, Core Web Vitals, resource sizes를 baseline합니다. 모든 PR에서 before/after를 비교합니다. |
| `/document-release` | **Technical Writer** | 방금 ship한 내용에 맞게 모든 project docs를 update합니다. stale READMEs를 자동으로 잡습니다. |
| `/retro` | **Eng Manager** | Team-aware weekly retro. Per-person breakdowns, shipping streaks, test health trends, growth opportunities. `/retro global`은 모든 projects와 AI tools(Claude Code, Codex, Gemini)에 걸쳐 실행됩니다. |
| `/browse` | **QA Engineer** | agent에게 눈을 줍니다. Real Chromium browser, real clicks, real screenshots. command당 ~100ms. `/open-gstack-browser`는 sidebar, anti-bot stealth, auto model routing이 있는 GStack Browser를 launch합니다. |
| `/setup-browser-cookies` | **Session Manager** | 실제 browser(Chrome, Arc, Brave, Edge)의 cookies를 headless session으로 import합니다. authenticated pages를 test합니다. |
| `/autoplan` | **Review Pipeline** | 한 command로 완전히 review된 plan. encoded decision principles로 CEO → design → eng review를 자동 실행합니다. approval이 필요한 taste decisions만 surface합니다. |
| `/learn` | **Memory** | session을 넘어 gstack이 학습한 것을 관리합니다. project-specific patterns, pitfalls, preferences를 review, search, prune, export합니다. Learnings는 session을 거듭할수록 쌓이므로 gstack은 codebase에 대해 점점 더 똑똑해집니다. |

### Which review should I use?

| Building for... | Plan stage (before code) | Live audit (after shipping) |
|-----------------|--------------------------|----------------------------|
| **End users** (UI, web app, mobile) | `/plan-design-review` | `/design-review` |
| **Developers** (API, CLI, SDK, docs) | `/plan-devex-review` | `/devex-review` |
| **Architecture** (data flow, perf, tests) | `/plan-eng-review` | `/review` |
| **All of the above** | `/autoplan` (CEO → design → eng → DX를 실행하고, 적용할 것을 auto-detect) | — |

### Power tools

| Skill | What it does |
|-------|-------------|
| `/codex` | **Second Opinion** — OpenAI Codex CLI의 independent code review. 세 가지 modes: review(pass/fail gate), adversarial challenge, open consultation. `/review`와 `/codex`가 둘 다 실행된 경우 cross-model analysis. |
| `/careful` | **Safety Guardrails** — destructive commands(rm -rf, DROP TABLE, force-push) 전에 warn합니다. activate하려면 "be careful"이라고 말하세요. 어떤 warning이든 override할 수 있습니다. |
| `/freeze` | **Edit Lock** — file edits를 하나의 directory로 제한합니다. debugging 중 scope 밖 accidental changes를 막습니다. |
| `/guard` | **Full Safety** — `/careful` + `/freeze`를 한 command로. prod work를 위한 maximum safety. |
| `/unfreeze` | **Unlock** — `/freeze` boundary를 제거합니다. |
| `/open-gstack-browser` | **GStack Browser** — sidebar, anti-bot stealth, auto model routing(actions에는 Sonnet, analysis에는 Opus), one-click cookie import, Claude Code integration이 있는 GStack Browser를 launch합니다. pages를 clean up하고, smart screenshots를 찍고, CSS를 edit하고, 정보를 terminal로 다시 넘깁니다. |
| `/setup-deploy` | **Deploy Configurator** — `/land-and-deploy`를 위한 one-time setup. platform, production URL, deploy commands를 감지합니다. |
| `/gstack-upgrade` | **Self-Updater** — gstack을 latest로 upgrade합니다. global vs vendored install을 감지하고, 둘 다 sync하며, 변경 사항을 보여줍니다. |

**[Deep dives with examples and philosophy for every skill →](docs/skills.md)**

## Parallel sprints

gstack은 하나의 sprint에서도 잘 작동합니다. 동시에 열 개가 돌아가면 흥미로워집니다.

**Design is at the heart.** `/design-consultation`은 design system을 처음부터 만들고, 무엇이 이미 있는지 research하며, creative risks를 제안하고, `DESIGN.md`를 씁니다. 하지만 진짜 magic은 shotgun-to-HTML pipeline입니다.

**`/design-shotgun` is how you explore.** 원하는 것을 설명하세요. GPT Image를 사용해 4-6개의 AI mockup variants를 생성합니다. 그런 다음 browser에서 모든 variants를 나란히 보여주는 comparison board를 엽니다. favorites를 고르고, feedback("more whitespace", "bolder headline", "lose the gradient")을 남기면 새 round를 생성합니다. 마음에 드는 것이 나올 때까지 반복하세요. 몇 round가 지나면 taste memory가 작동해서 실제로 당신이 좋아하는 쪽으로 bias하기 시작합니다. 더 이상 vision을 말로 설명하고 AI가 알아듣기를 바라지 않아도 됩니다. options를 보고, 좋은 것을 고르고, 시각적으로 iterate합니다.

**`/design-html` makes it real.** 승인된 mockup(`/design-shotgun`, CEO plan, design review, 또는 단순 description에서 온 것)을 production-quality HTML/CSS로 바꿉니다. 한 viewport width에서는 괜찮아 보이지만 다른 곳에서는 깨지는 그런 AI HTML이 아닙니다. 이것은 computed text layout을 위해 Pretext를 사용합니다: text가 resize에서 실제로 reflow되고, heights가 content에 맞게 조정되며, layouts가 dynamic합니다. 30KB overhead, zero dependencies. framework(React, Svelte, Vue)를 감지하고 올바른 format으로 output합니다. Smart API routing은 landing page, dashboard, form, card layout 중 무엇인지에 따라 다른 Pretext patterns를 고릅니다. output은 demo가 아니라 실제로 ship할 수 있는 것입니다.

**`/qa` was a massive unlock.** 이를 통해 저는 parallel workers를 6개에서 12개로 늘릴 수 있었습니다. Claude Code가 *"I SEE THE ISSUE"*라고 말하고 실제로 고치고, regression test를 생성하고, fix를 verify하는 것 — 이것이 제 작업 방식을 바꿨습니다. 이제 agent에게 눈이 생겼습니다.

**Smart review routing.** 잘 운영되는 startup과 같습니다. CEO가 infra bug fixes를 볼 필요는 없고, backend changes에는 design review가 필요 없습니다. gstack은 어떤 reviews가 실행됐는지 추적하고, 무엇이 적절한지 파악한 뒤, 똑똑하게 처리합니다. Review Readiness Dashboard는 ship하기 전에 현재 상태를 알려줍니다.

**Test everything.** `/ship`은 project에 test framework가 없으면 처음부터 bootstrap합니다. 모든 `/ship` 실행은 coverage audit을 생성합니다. 모든 `/qa` bug fix는 regression test를 생성합니다. 목표는 100% test coverage입니다. tests는 vibe coding을 yolo coding이 아니라 safe하게 만듭니다.

**`/document-release` is the engineer you never had.** project의 모든 doc file을 읽고, diff와 cross-reference하며, drift된 모든 것을 update합니다. README, ARCHITECTURE, CONTRIBUTING, CLAUDE.md, TODOS — 모두 자동으로 최신 상태를 유지합니다. 이제 `/ship`이 이를 auto-invoke합니다. 추가 command 없이 docs가 최신으로 유지됩니다.

**Real browser mode.** `/open-gstack-browser`는 anti-bot stealth, custom branding, sidebar extension이 내장된 AI-controlled Chromium인 GStack Browser를 launch합니다. Google과 NYTimes 같은 site도 captchas 없이 작동합니다. menu bar에는 "Chrome for Testing" 대신 "GStack Browser"라고 표시됩니다. 일반 Chrome은 건드리지 않습니다. 기존 browse commands는 모두 변경 없이 작동합니다. `$B disconnect`는 headless로 돌아갑니다. browser는 window가 열려 있는 동안 살아 있습니다. 작업 중 idle timeout으로 죽지 않습니다.

**Sidebar agent — your AI browser assistant.** Chrome side panel에 natural language로 입력하면 child Claude instance가 실행합니다. "Navigate to the settings page and screenshot it." "Fill out this form with test data." "Go through every item in this list and extract the prices." sidebar는 적절한 model로 auto-route합니다: 빠른 actions(click, navigate, screenshot)에는 Sonnet, reading과 analysis에는 Opus. 각 task는 최대 5분을 받습니다. sidebar agent는 isolated session에서 실행되므로 main Claude Code window를 방해하지 않습니다. sidebar footer에서 one-click cookie import가 가능합니다.

**Personal automation.** sidebar agent는 dev workflows만을 위한 것이 아닙니다. 예: "Browse my kid's school parent portal and add all the other parents' names, phone numbers, and photos to my Google Contacts." authenticated되는 방법은 두 가지입니다: (1) headed browser에서 한 번 login하면 session이 유지됩니다. 또는 (2) sidebar footer의 "cookies" button을 클릭해 실제 Chrome에서 cookies를 import합니다. authenticated된 후 Claude는 directory를 navigate하고, data를 extract하고, contacts를 생성합니다.

**Browser handoff when the AI gets stuck.** CAPTCHA, auth wall, MFA prompt에 막혔나요? `$B handoff`는 모든 cookies와 tabs를 그대로 유지한 채 정확히 같은 page에서 visible Chrome을 엽니다. 문제를 해결하고 Claude에게 끝났다고 말하면, `$B resume`이 바로 이어서 진행합니다. agent는 연속 3번 실패하면 이를 자동으로 제안하기도 합니다.

**`/pair-agent` is cross-agent coordination.** 당신은 Claude Code에 있습니다. OpenClaw도 실행 중입니다. 또는 Hermes. 또는 Codex. 둘 다 같은 website를 보게 하고 싶습니다. `/pair-agent`를 입력하고 agent를 고르면, 볼 수 있도록 GStack Browser window가 열립니다. skill은 instructions block을 출력합니다. 그 block을 다른 agent의 chat에 붙여넣으세요. one-time setup key를 session token으로 교환하고, 자기 tab을 만들고, browsing을 시작합니다. 두 agent가 같은 browser에서 각자 자기 tab으로 작업하는 것을 볼 수 있고, 서로 방해할 수 없습니다. ngrok이 설치되어 있으면 tunnel이 자동으로 시작되어 다른 agent가 완전히 다른 machine에 있어도 됩니다. 같은 machine의 agents는 credentials를 직접 쓰는 zero-friction shortcut을 얻습니다. scoped tokens, tab isolation, rate limiting, domain restrictions, activity attribution을 갖춘 real security로 서로 다른 vendor의 AI agents가 shared browser를 통해 coordinate할 수 있게 된 첫 사례입니다.

**Multi-AI second opinion.** `/codex`는 OpenAI의 Codex CLI에서 independent review를 받습니다. 같은 diff를 보는 완전히 다른 AI입니다. 세 가지 modes: pass/fail gate가 있는 code review, code를 적극적으로 깨뜨리려는 adversarial challenge, session continuity가 있는 open consultation. `/review`(Claude)와 `/codex`(OpenAI)가 같은 branch를 모두 review하면, 어떤 findings가 겹치고 어떤 것이 각자 unique한지 보여주는 cross-model analysis를 얻습니다.

**Safety guardrails on demand.** "be careful"이라고 말하면 `/careful`이 destructive command — rm -rf, DROP TABLE, force-push, git reset --hard — 전에 warn합니다. `/freeze`는 debugging 중 edits를 하나의 directory로 lock해서 Claude가 관련 없는 code를 실수로 "fix"하지 못하게 합니다. `/guard`는 둘 다 activate합니다. `/investigate`는 조사 중인 module로 auto-freeze합니다.

**Proactive skill suggestions.** gstack은 지금 단계가 brainstorming, reviewing, debugging, testing 중 어디인지 알아차리고 적절한 skill을 제안합니다. 마음에 들지 않나요? "stop suggesting"이라고 말하면 session을 넘어 기억합니다.

## 10-15 parallel sprints

gstack은 하나의 sprint에서도 강력합니다. 동시에 열 개가 돌아가면 transformative합니다.

[Conductor](https://conductor.build)는 여러 Claude Code sessions를 parallel로 실행합니다. 각 session은 자기 isolated workspace 안에서요. 한 session은 새로운 idea에 `/office-hours`를 실행하고, 다른 하나는 PR에 `/review`를 하고, 세 번째는 feature를 구현하고, 네 번째는 staging에서 `/qa`를 실행하고, 나머지 여섯 개는 다른 branches에서 일합니다. 모두 동시에요. 저는 정기적으로 10-15개의 parallel sprints를 돌립니다. 지금 practical max는 그 정도입니다.

sprint structure가 parallelism을 가능하게 합니다. process가 없으면 열 개의 agents는 열 개의 chaos source입니다. process — think, plan, build, review, test, ship — 가 있으면 각 agent는 무엇을 해야 하고 언제 멈춰야 하는지 정확히 압니다. CEO가 팀을 관리하듯이 관리하면 됩니다. 중요한 decisions만 확인하고 나머지는 실행되게 두세요.

### Voice input (AquaVoice, Whisper, etc.)

gstack skills에는 voice-friendly trigger phrases가 있습니다. 원하는 것을 자연스럽게 말하세요 — "run a security check", "test the website", "do an engineering review" — 그러면 적절한 skill이 activate됩니다. slash command names나 acronyms를 외울 필요가 없습니다.

## Uninstall

### Option 1: Run the uninstall script

gstack이 machine에 설치되어 있다면:

```bash
~/.claude/skills/gstack/bin/gstack-uninstall
```

이 script는 skills, symlinks, global state(`~/.gstack/`), project-local state, browse daemons, temp files를 처리합니다. config와 analytics를 보존하려면 `--keep-state`를 사용하세요. confirmation을 건너뛰려면 `--force`를 사용하세요.

### Option 2: Manual removal (no local repo)

repo clone이 없는 경우(예: Claude Code paste로 설치한 뒤 나중에 clone을 삭제한 경우):

```bash
# 1. Stop browse daemons
pkill -f "gstack.*browse" 2>/dev/null || true

# 2. Remove per-skill symlinks pointing into gstack/
find ~/.claude/skills -maxdepth 1 -type l 2>/dev/null | while read -r link; do
  case "$(readlink "$link" 2>/dev/null)" in gstack/*|*/gstack/*) rm -f "$link" ;; esac
done

# 3. Remove gstack
rm -rf ~/.claude/skills/gstack

# 4. Remove global state
rm -rf ~/.gstack

# 5. Remove integrations (skip any you never installed)
rm -rf ~/.codex/skills/gstack* 2>/dev/null
rm -rf ~/.factory/skills/gstack* 2>/dev/null
rm -rf ~/.kiro/skills/gstack* 2>/dev/null
rm -rf ~/.openclaw/skills/gstack* 2>/dev/null

# 6. Remove temp files
rm -f /tmp/gstack-* 2>/dev/null

# 7. Per-project cleanup (run from each project root)
rm -rf .gstack .gstack-worktrees .claude/skills/gstack 2>/dev/null
rm -rf .agents/skills/gstack* .factory/skills/gstack* 2>/dev/null
```

### Clean up CLAUDE.md

uninstall script는 CLAUDE.md를 edit하지 않습니다. gstack을 추가했던 각 project에서 `## gstack`과 `## Skill routing` sections를 제거하세요.

### Playwright

`~/Library/Caches/ms-playwright/` (macOS)는 다른 tools와 공유될 수 있어서 그대로 둡니다. 다른 어떤 것도 필요로 하지 않는다면 제거하세요.

---

Free, MIT licensed, open source. No premium tier, no waitlist.

저는 제가 software를 만드는 방식을 open source로 공개했습니다. fork해서 여러분의 것으로 만드세요.

> **We're hiring.** Want to ship 10K+ LOC/day and help harden gstack?
> Come work at YC — [ycombinator.com/software](https://ycombinator.com/software)
> Extremely competitive salary and equity. San Francisco, Dogpatch District.

## Docs

| Doc | What it covers |
|-----|---------------|
| [Skill Deep Dives](docs/skills.md) | 모든 skill의 philosophy, examples, workflow (Greptile integration 포함) |
| [Builder Ethos](ETHOS.md) | Builder philosophy: Boil the Lake, Search Before Building, three layers of knowledge |
| [Architecture](ARCHITECTURE.md) | Design decisions와 system internals |
| [Browser Reference](BROWSER.md) | `/browse`의 full command reference |
| [Contributing](CONTRIBUTING.md) | Dev setup, testing, contributor mode, dev mode |
| [Changelog](CHANGELOG.md) | 모든 version의 변경 사항 |

## Privacy & Telemetry

gstack에는 project 개선을 돕기 위한 **opt-in** usage telemetry가 포함되어 있습니다. 정확히 이렇게 작동합니다:

- **Default is off.** 명시적으로 yes라고 하지 않는 한 아무것도 전송되지 않습니다.
- **On first run,** gstack은 anonymous usage data를 share할지 묻습니다. no라고 답할 수 있습니다.
- **What's sent (if you opt in):** skill name, duration, success/fail, gstack version, OS. 그게 전부입니다.
- **What's never sent:** code, file paths, repo names, branch names, prompts, 또는 어떤 user-generated content도 전송되지 않습니다.
- **Change anytime:** `gstack-config set telemetry off`는 즉시 모든 것을 disable합니다.

Data는 [Supabase](https://supabase.com)(open source Firebase alternative)에 저장됩니다. schema는 [`supabase/migrations/`](supabase/migrations/)에 있습니다. 정확히 무엇이 수집되는지 verify할 수 있습니다. repo의 Supabase publishable key는 public key(Firebase API key와 같은 것)입니다. row-level security policies가 모든 direct access를 deny합니다. Telemetry는 schema checks, event type allowlists, field length limits를 enforce하는 validated edge functions를 통해 흐릅니다.

**Local analytics are always available.** `gstack-analytics`를 실행하면 local JSONL file에서 personal usage dashboard를 볼 수 있습니다. remote data는 필요 없습니다.

## Troubleshooting

**Skill not showing up?** `cd ~/.claude/skills/gstack && ./setup`

**`/browse` fails?** `cd ~/.claude/skills/gstack && bun install && bun run build`

**Stale install?** `/gstack-upgrade`를 실행하세요 — 또는 `~/.gstack/config.yaml`에서 `auto_upgrade: true`를 설정하세요

**Want shorter commands?** `cd ~/.claude/skills/gstack && ./setup --no-prefix` — `/gstack-qa`에서 `/qa`로 전환합니다. 이 선택은 future upgrades에서도 기억됩니다.

**Want namespaced commands?** `cd ~/.claude/skills/gstack && ./setup --prefix` — `/qa`에서 `/gstack-qa`로 전환합니다. gstack과 함께 다른 skill packs를 실행할 때 유용합니다.

**Codex says "Skipped loading skill(s) due to invalid SKILL.md"?** Codex skill descriptions가 stale입니다. Fix: `cd ~/.codex/skills/gstack && git pull && ./setup --host codex` — 또는 repo-local installs의 경우: `cd "$(readlink -f .agents/skills/gstack)" && git pull && ./setup --host codex`

**Windows users:** gstack은 Git Bash 또는 WSL을 통해 Windows 11에서 작동합니다. Bun 외에도 Node.js가 필요합니다. Bun에는 Windows에서 Playwright의 pipe transport와 관련된 알려진 bug가 있습니다([bun#4253](https://github.com/oven-sh/bun/issues/4253)). browse server는 자동으로 Node.js로 fallback합니다. `bun`과 `node`가 모두 PATH에 있는지 확인하세요.

**Claude says it can't see the skills?** project의 `CLAUDE.md`에 gstack section이 있는지 확인하세요. 이것을 추가하세요:

```
## gstack
Use /browse from gstack for all web browsing. Never use mcp__claude-in-chrome__* tools.
Available skills: /office-hours, /plan-ceo-review, /plan-eng-review, /plan-design-review,
/design-consultation, /design-shotgun, /design-html, /review, /ship, /land-and-deploy,
/canary, /benchmark, /browse, /open-gstack-browser, /qa, /qa-only, /design-review,
/setup-browser-cookies, /setup-deploy, /retro, /investigate, /document-release, /codex,
/cso, /autoplan, /pair-agent, /careful, /freeze, /guard, /unfreeze, /gstack-upgrade, /learn.
```

## License

MIT. Free forever. Go build something.
