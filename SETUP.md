# gstack-ko 셋업 가이드

gstack 한글화 fork를 새 PC에 설치하는 방법.

## 사전 요구 사항

```bash
git --version    # 2.x 이상
node --version   # 18 이상
bun --version    # 1.x 이상
```

없으면 설치:
- Git: https://git-scm.com
- Node.js: https://nodejs.org
- Bun: `curl -fsSL https://bun.sh/install | bash` (macOS/Linux) / `powershell -c "irm bun.sh/install.ps1 | iex"` (Windows)

> **Windows:** Bun + Node.js 둘 다 필요합니다 (browse 데몬의 Windows 파이프 버그 우회).

## 설치

```bash
# 1. clone (반드시 이 경로에)
git clone -b ko https://github.com/KimYoungHwan8750/gstack-ko.git ~/.claude/skills/gstack

# 2. upstream 등록 (업그레이드용)
cd ~/.claude/skills/gstack
git remote add upstream https://github.com/garrytan/gstack.git

# 3. 빌드 + 전역 설치
bun install
bun run build
./setup
```

`./setup`이 `~/.claude/skills/` 안에서 실행되면 자동으로 각 스킬의 심볼릭 링크를 전역에 생성합니다.

Claude Code를 재시작하면 `/office-hours`, `/qa`, `/ship` 등 한글 스킬이 활성화됩니다.

## 업그레이드

Claude Code에서 `/gstack-upgrade` 실행. three-way merge로 한글화가 보존됩니다.

수동:
```bash
cd ~/.claude/skills/gstack
git fetch upstream
git merge upstream/main
bun run gen:skill-docs
./setup
```

## 문제 해결

| 증상 | 해결 |
|------|------|
| 스킬 안 뜸 | Claude Code 재시작 |
| browse 에러 | `cd ~/.claude/skills/gstack && bun install && bun run build` |
| Chromium 에러 | `cd ~/.claude/skills/gstack && bunx playwright install chromium` |
