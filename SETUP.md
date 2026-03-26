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
# 1. clone
mkdir -p ~/.gstack/repos
git clone https://github.com/KimYoungHwan8750/gstack-ko.git ~/.gstack/repos/gstack
cd ~/.gstack/repos/gstack
git checkout ko

# 2. upstream 등록 (업그레이드용)
git remote add upstream https://github.com/garrytan/gstack.git

# 3. 빌드
bun install
bun run build
./setup

# 4. 스킬 배포
GSTACK=~/.gstack/repos/gstack
for d in "$GSTACK"/*/; do
  [ -f "$d/SKILL.md" ] && cp -r "$d" ~/.claude/skills/"$(basename "$d")"
done
```

Claude Code를 재시작하면 `/office-hours`, `/qa`, `/ship` 등 한글 스킬이 활성화됩니다.

## 업그레이드

Claude Code에서 `/gstack-upgrade` 실행. three-way merge로 한글화가 보존됩니다.

수동:
```bash
cd ~/.gstack/repos/gstack
git fetch upstream
git merge upstream/main
bun run gen:skill-docs && ./setup
# 위 Step 4 스킬 배포 재실행
```

## 문제 해결

| 증상 | 해결 |
|------|------|
| 스킬 안 뜸 | Claude Code 재시작 |
| browse 에러 | `cd ~/.gstack/repos/gstack && bun install && bun run build` |
| Chromium 에러 | `cd ~/.gstack/repos/gstack && bunx playwright install chromium` |
