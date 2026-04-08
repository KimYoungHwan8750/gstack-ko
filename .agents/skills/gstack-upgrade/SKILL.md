---
name: gstack-upgrade
description: |
  gstack을 최신 버전으로 업그레이드합니다. 전역 또는 벤더 설치를 감지하고,
  업그레이드를 실행한 뒤 변경 사항을 표시합니다. "upgrade gstack",
  "update gstack", "get latest version" 요청 시 사용합니다.
  Voice triggers (speech-to-text aliases): "upgrade the tools", "update the tools", "gee stack upgrade", "g stack upgrade".
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

# /gstack-upgrade

gstack을 최신 버전으로 업그레이드하고 변경 사항을 표시합니다.

## 인라인 업그레이드 플로우

이 섹션은 모든 스킬 프리앰블이 `UPGRADE_AVAILABLE`을 감지했을 때 참조됩니다.

### Step 1: 사용자에게 묻기 (또는 자동 업그레이드)

먼저, 자동 업그레이드가 활성화되어 있는지 확인합니다:
```bash
_AUTO=""
[ "${GSTACK_AUTO_UPGRADE:-}" = "1" ] && _AUTO="true"
[ -z "$_AUTO" ] && _AUTO=$($GSTACK_ROOT/bin/gstack-config get auto_upgrade 2>/dev/null || true)
echo "AUTO_UPGRADE=$_AUTO"
```

**`AUTO_UPGRADE=true` 또는 `AUTO_UPGRADE=1`인 경우:** AskUserQuestion을 건너뜁니다. "Auto-upgrading gstack v{old} → v{new}..."를 로그에 기록하고 바로 Step 2로 진행합니다. 자동 업그레이드 중 `./setup`이 실패하면, 백업(`.bak` 디렉토리)에서 복원하고 사용자에게 경고합니다: "자동 업그레이드 실패 — 이전 버전을 복원했습니다. `/gstack-upgrade`를 수동으로 실행하여 재시도하세요."

**그 외의 경우**, AskUserQuestion을 사용합니다:
- 질문: "gstack **v{new}** 버전이 출시되었습니다 (현재 v{old}). 지금 업그레이드하시겠습니까?"
- 옵션: ["Yes, upgrade now", "Always keep me up to date", "Not now", "Never ask again"]

**"Yes, upgrade now"인 경우:** Step 2로 진행합니다.

**"Always keep me up to date"인 경우:**
```bash
$GSTACK_ROOT/bin/gstack-config set auto_upgrade true
```
사용자에게 전달합니다: "자동 업그레이드가 활성화되었습니다. 향후 업데이트는 자동으로 설치됩니다." 그런 다음 Step 2로 진행합니다.

**"Not now"인 경우:** 점진적 백오프로 다시 알림 상태를 기록합니다 (첫 번째 다시 알림 = 24시간, 두 번째 = 48시간, 세 번째 이상 = 1주일), 그런 다음 현재 스킬을 계속 진행합니다. 업그레이드에 대해 다시 언급하지 않습니다.
```bash
_SNOOZE_FILE=~/.gstack/update-snoozed
_REMOTE_VER="{new}"
_CUR_LEVEL=0
if [ -f "$_SNOOZE_FILE" ]; then
  _SNOOZED_VER=$(awk '{print $1}' "$_SNOOZE_FILE")
  if [ "$_SNOOZED_VER" = "$_REMOTE_VER" ]; then
    _CUR_LEVEL=$(awk '{print $2}' "$_SNOOZE_FILE")
    case "$_CUR_LEVEL" in *[!0-9]*) _CUR_LEVEL=0 ;; esac
  fi
fi
_NEW_LEVEL=$((_CUR_LEVEL + 1))
[ "$_NEW_LEVEL" -gt 3 ] && _NEW_LEVEL=3
echo "$_REMOTE_VER $_NEW_LEVEL $(date +%s)" > "$_SNOOZE_FILE"
```
참고: `{new}`는 `UPGRADE_AVAILABLE` 출력에서 가져온 원격 버전입니다 — 업데이트 확인 결과에서 대입하세요.

사용자에게 다시 알림 기간을 전달합니다: "다음 알림: 24시간 후" (또는 레벨에 따라 48시간 또는 1주일). 팁: "자동 업그레이드를 원하시면 `~/.gstack/config.yaml`에서 `auto_upgrade: true`로 설정하세요."

**"Never ask again"인 경우:**
```bash
$GSTACK_ROOT/bin/gstack-config set update_check false
```
사용자에게 전달합니다: "업데이트 확인이 비활성화되었습니다. 다시 활성화하려면 `$GSTACK_ROOT/bin/gstack-config set update_check true`를 실행하세요."
현재 스킬을 계속 진행합니다.

### Step 2: 설치 유형 감지

```bash
if [ -d "$HOME/.gstack/repos/gstack/.git" ]; then
  INSTALL_TYPE="ko-fork"
  INSTALL_DIR="$HOME/.gstack/repos/gstack"
elif [ -d "$HOME/.agents/skills/gstack/.git" ]; then
  INSTALL_TYPE="global-git"
  INSTALL_DIR="$HOME/.agents/skills/gstack"
elif [ -d ".agents/skills/gstack/.git" ]; then
  INSTALL_TYPE="local-git"
  INSTALL_DIR=".agents/skills/gstack"
elif [ -d ".agents/skills/gstack/.git" ]; then
  INSTALL_TYPE="local-git"
  INSTALL_DIR=".agents/skills/gstack"
elif [ -d ".agents/skills/gstack" ]; then
  INSTALL_TYPE="vendored"
  INSTALL_DIR=".agents/skills/gstack"
elif [ -d "$HOME/.agents/skills/gstack" ]; then
  INSTALL_TYPE="vendored-global"
  INSTALL_DIR="$HOME/.agents/skills/gstack"
else
  echo "ERROR: gstack not found"
  exit 1
fi
echo "Install type: $INSTALL_TYPE at $INSTALL_DIR"
```

위에서 출력된 설치 유형과 디렉토리 경로는 이후 모든 단계에서 사용됩니다.

### Step 3: 이전 버전 저장

아래에서 Step 2의 출력에서 얻은 설치 디렉토리를 사용합니다:

```bash
OLD_VERSION=$(cat "$INSTALL_DIR/VERSION" 2>/dev/null || echo "unknown")
```

### Step 4: 업그레이드

Step 2에서 감지된 설치 유형과 디렉토리를 사용합니다:

**한국어 fork 설치의 경우** (ko-fork) — **three-way merge 방식**:

이 방식은 한글화된 커스터마이징을 보존하면서 upstream 변경 사항을 병합합니다.

```bash
cd "$INSTALL_DIR"

# 1. 백업 브랜치 생성
git branch "backup-$(date +%s)" 2>/dev/null || true

# 2. upstream fetch
git fetch upstream 2>&1

# 3. base commit 확인
BASE_SHA=$(cat .ko-base-commit 2>/dev/null || git merge-base upstream/main HEAD)
UPSTREAM_SHA=$(git rev-parse upstream/main)

if [ "$BASE_SHA" = "$UPSTREAM_SHA" ]; then
  echo "NO_UPSTREAM_CHANGES"
else
  echo "UPSTREAM_CHANGES_AVAILABLE base=$BASE_SHA upstream=$UPSTREAM_SHA"
fi
```

**`NO_UPSTREAM_CHANGES`인 경우:** "upstream에 새로운 변경 사항이 없습니다."라고 전달하고 Step 6으로 건너뜁니다.

**`UPSTREAM_CHANGES_AVAILABLE`인 경우:** three-way merge를 시도합니다:
```bash
cd "$INSTALL_DIR"
git merge upstream/main --no-edit 2>&1
MERGE_STATUS=$?
echo "MERGE_STATUS=$MERGE_STATUS"
```

**`MERGE_STATUS=0` (성공)인 경우:**
```bash
cd "$INSTALL_DIR"
echo "$UPSTREAM_SHA" > .ko-base-commit
git add .ko-base-commit && git commit -m "chore: update ko-base-commit after upstream merge" 2>/dev/null || true
```

변경된 `.tmpl` 파일 목록을 확인합니다:
```bash
cd "$INSTALL_DIR"
git diff "$BASE_SHA".."$UPSTREAM_SHA" --name-only -- "*/SKILL.md.tmpl" "*.md"
```

변경된 파일이 있으면 AskUserQuestion으로 물어봅니다:
> upstream에서 N개 파일이 변경되었습니다. 새로 추가/수정된 영어 콘텐츠를 한글로 번역할까요?
> A) 지금 번역 (추천)
> B) 나중에 수동으로 번역
> C) 번역 없이 진행

A를 선택하면: 변경된 각 `.tmpl` 파일을 읽고, 영어로 남아있는 새 섹션을 찾아 한글로 번역합니다. 번역 후 `bun run gen:skill-docs`를 실행합니다.

**`MERGE_STATUS≠0` (충돌)인 경우:**
충돌 파일 목록을 출력합니다:
```bash
cd "$INSTALL_DIR"
git diff --name-only --diff-filter=U
```

사용자에게 전달합니다: "병합 충돌이 발생했습니다. 다음 파일에서 충돌을 해결해주세요:" 그리고 충돌 파일 목록을 표시합니다.

AskUserQuestion으로 물어봅니다:
> A) 충돌을 자동으로 해결 시도 (한글 버전 우선)
> B) 수동으로 해결할게요 — merge를 중단하고 대기
> C) 업그레이드 취소 — 이전 상태로 복원

A를 선택하면: 각 충돌 파일에서 한글 버전(ours)을 우선하되, upstream의 새로운 섹션은 추가합니다.
B를 선택하면: "충돌 해결 후 `cd $INSTALL_DIR && git merge --continue`를 실행한 뒤, `/gstack-upgrade`를 다시 호출하세요."
C를 선택하면:
```bash
cd "$INSTALL_DIR"
git merge --abort
```

모든 경우에 `./setup`을 실행하여 글로벌에 배포합니다.

**기존 git 설치의 경우** (global-git, local-git):
```bash
cd "$INSTALL_DIR"
STASH_OUTPUT=$(git stash 2>&1)
git fetch origin
git reset --hard origin/main
./setup
```
`$STASH_OUTPUT`에 "Saved working directory"가 포함되어 있으면, 사용자에게 경고합니다: "참고: 로컬 변경 사항이 stash되었습니다. 복원하려면 스킬 디렉토리에서 `git stash pop`을 실행하세요."

**벤더 설치의 경우** (vendored, vendored-global):
```bash
PARENT=$(dirname "$INSTALL_DIR")
TMP_DIR=$(mktemp -d)
git clone --depth 1 https://github.com/garrytan/gstack.git "$TMP_DIR/gstack"
mv "$INSTALL_DIR" "$INSTALL_DIR.bak"
mv "$TMP_DIR/gstack" "$INSTALL_DIR"
cd "$INSTALL_DIR" && ./setup
rm -rf "$INSTALL_DIR.bak" "$TMP_DIR"
```

### Step 4.5: 로컬 벤더 복사본 처리

Step 2의 설치 디렉토리를 사용합니다. 로컬 벤더 복사본도 있는지, 그리고 team mode가 활성화되어 있는지 확인합니다:

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
LOCAL_GSTACK=""
if [ -n "$_ROOT" ] && [ -d "$_ROOT/.agents/skills/gstack" ]; then
  _RESOLVED_LOCAL=$(cd "$_ROOT/.agents/skills/gstack" && pwd -P)
  _RESOLVED_PRIMARY=$(cd "$INSTALL_DIR" && pwd -P)
  if [ "$_RESOLVED_LOCAL" != "$_RESOLVED_PRIMARY" ]; then
    LOCAL_GSTACK="$_ROOT/.agents/skills/gstack"
  fi
fi
_TEAM_MODE=$($GSTACK_ROOT/bin/gstack-config get team_mode 2>/dev/null || echo "false")
echo "LOCAL_GSTACK=$LOCAL_GSTACK"
echo "TEAM_MODE=$_TEAM_MODE"
```

**`LOCAL_GSTACK`가 비어 있지 않고 `TEAM_MODE`가 `true`인 경우:** 벤더 복사본을 제거합니다. team mode는 전역 설치를 단일 진실 공급원으로 사용합니다.

```bash
cd "$_ROOT"
git rm -r --cached .agents/skills/gstack/ 2>/dev/null || true
if ! grep -qF '.agents/skills/gstack/' .gitignore 2>/dev/null; then
  echo '.agents/skills/gstack/' >> .gitignore
fi
rm -rf "$LOCAL_GSTACK"
```
사용자에게 전달합니다: "`$LOCAL_GSTACK`의 벤더 복사본을 제거했습니다 (team mode 활성화 — 전역 설치가 진실 공급원입니다). 준비되면 `.gitignore` 변경 사항을 커밋하세요."

**`LOCAL_GSTACK`가 비어 있지 않고 `TEAM_MODE`가 `true`가 아닌 경우:** 방금 업그레이드한 기본 설치에서 복사하여 업데이트합니다 (README 벤더 설치와 동일한 방식):
```bash
mv "$LOCAL_GSTACK" "$LOCAL_GSTACK.bak"
cp -Rf "$INSTALL_DIR" "$LOCAL_GSTACK"
rm -rf "$LOCAL_GSTACK/.git"
cd "$LOCAL_GSTACK" && ./setup
rm -rf "$LOCAL_GSTACK.bak"
```
사용자에게 전달합니다: "`$LOCAL_GSTACK`의 벤더 복사본도 업데이트되었습니다 — 준비되면 `.agents/skills/gstack/`을 커밋하세요."

`./setup`이 실패하면, 백업에서 복원하고 사용자에게 경고합니다:
```bash
rm -rf "$LOCAL_GSTACK"
mv "$LOCAL_GSTACK.bak" "$LOCAL_GSTACK"
```
사용자에게 전달합니다: "동기화 실패 — `$LOCAL_GSTACK`의 이전 버전을 복원했습니다. `/gstack-upgrade`를 수동으로 실행하여 재시도하세요."

### Step 4.75: 버전 마이그레이션 실행

`./setup`이 완료된 후, 이전 버전과 새 버전 사이의 마이그레이션 스크립트를 실행합니다. 마이그레이션은 `./setup`만으로 처리할 수 없는 상태 수정(오래된 config, 고아 파일, 디렉토리 구조 변경)을 처리합니다.

```bash
MIGRATIONS_DIR="$INSTALL_DIR/gstack-upgrade/migrations"
if [ -d "$MIGRATIONS_DIR" ]; then
  for migration in $(find "$MIGRATIONS_DIR" -maxdepth 1 -name 'v*.sh' -type f 2>/dev/null | sort -V); do
    # Extract version from filename: v0.15.2.0.sh → 0.15.2.0
    m_ver="$(basename "$migration" .sh | sed 's/^v//')"
    # Run if this migration version is newer than old version
    # (simple string compare works for dotted versions with same segment count)
    if [ "$OLD_VERSION" != "unknown" ] && [ "$(printf '%s\n%s' "$OLD_VERSION" "$m_ver" | sort -V | head -1)" = "$OLD_VERSION" ] && [ "$OLD_VERSION" != "$m_ver" ]; then
      echo "Running migration $m_ver..."
      bash "$migration" || echo "  Warning: migration $m_ver had errors (non-fatal)"
    fi
  done
fi
```

마이그레이션은 `gstack-upgrade/migrations/`에 있는 idempotent bash 스크립트입니다. 각 파일은 `v{VERSION}.sh`로 이름이 지정되며, 더 오래된 버전에서 업그레이드할 때만 실행됩니다. 새 마이그레이션을 추가하는 방법은 CONTRIBUTING.md를 참고하세요.

### Step 5: 마커 기록 + 캐시 삭제

```bash
mkdir -p ~/.gstack
echo "$OLD_VERSION" > ~/.gstack/just-upgraded-from
rm -f ~/.gstack/last-update-check
rm -f ~/.gstack/update-snoozed
```

### Step 6: 변경 사항 표시

`$INSTALL_DIR/CHANGELOG.md`를 읽습니다. 이전 버전과 새 버전 사이의 모든 버전 항목을 찾습니다. 테마별로 그룹화하여 5-7개의 핵심 사항으로 요약합니다. 너무 많은 정보를 나열하지 마세요 — 사용자 대면 변경 사항에 집중합니다. 중대한 변경이 아닌 한 내부 리팩토링은 건너뜁니다.

형식:
```
gstack v{new} — v{old}에서 업그레이드 완료!

변경 사항:
- [항목 1]
- [항목 2]
- ...

Happy shipping!
```

### Step 7: 계속 진행

변경 사항을 표시한 후, 사용자가 원래 호출한 스킬을 계속 진행합니다. 업그레이드가 완료되었으므로 추가 작업은 필요하지 않습니다.

---

## 독립 실행 사용법

프리앰블이 아닌 `/gstack-upgrade`로 직접 호출된 경우:

1. 강제로 최신 업데이트 확인을 실행합니다 (캐시 우회):
```bash
$GSTACK_ROOT/bin/gstack-update-check --force 2>/dev/null || \
.agents/skills/gstack/bin/gstack-update-check --force 2>/dev/null || true
```
출력을 사용하여 업그레이드 가능 여부를 판단합니다.

2. `UPGRADE_AVAILABLE <old> <new>`인 경우: 위의 Step 2-6을 따릅니다.

3. 출력이 없는 경우 (기본 설치가 최신인 경우): 오래된 로컬 벤더 복사본이 있는지 확인합니다.

위의 Step 2 bash 블록을 실행하여 기본 설치 유형과 디렉토리(`INSTALL_TYPE` 및 `INSTALL_DIR`)를 감지합니다. 그런 다음 위의 Step 4.5 감지 bash 블록을 실행하여 로컬 벤더 복사본(`LOCAL_GSTACK`)과 team mode 상태(`TEAM_MODE`)를 확인합니다.

**`LOCAL_GSTACK`가 비어 있는 경우** (로컬 벤더 복사본 없음): 사용자에게 "이미 최신 버전(v{version})입니다."라고 전달합니다.

**`LOCAL_GSTACK`가 비어 있지 않고 `TEAM_MODE`가 `true`인 경우:** 위의 Step 4.5 team-mode 제거 bash 블록을 사용하여 벤더 복사본을 제거합니다. 사용자에게 전달합니다: "전역 v{version}은 최신입니다. 오래된 벤더 복사본을 제거했습니다 (team mode 활성화). 준비되면 `.gitignore` 변경 사항을 커밋하세요."

**`LOCAL_GSTACK`가 비어 있지 않고 `TEAM_MODE`가 `true`가 아닌 경우**, 버전을 비교합니다:
```bash
PRIMARY_VER=$(cat "$INSTALL_DIR/VERSION" 2>/dev/null || echo "unknown")
LOCAL_VER=$(cat "$LOCAL_GSTACK/VERSION" 2>/dev/null || echo "unknown")
echo "PRIMARY=$PRIMARY_VER LOCAL=$LOCAL_VER"
```

**버전이 다른 경우:** 위의 Step 4.5 동기화 bash 블록을 따라 기본 설치에서 로컬 복사본을 업데이트합니다. 사용자에게 전달합니다: "전역 v{PRIMARY_VER}은 최신입니다. 로컬 벤더 복사본을 v{LOCAL_VER} → v{PRIMARY_VER}로 업데이트했습니다. 준비되면 `.agents/skills/gstack/`을 커밋하세요."

**버전이 같은 경우:** 사용자에게 "최신 버전(v{PRIMARY_VER})입니다. 전역 및 로컬 벤더 복사본 모두 최신 상태입니다."라고 전달합니다.
