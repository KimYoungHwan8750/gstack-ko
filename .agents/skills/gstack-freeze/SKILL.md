---
name: freeze
description: |
  세션 동안 파일 편집을 특정 디렉토리로 제한합니다. 허용된 경로 외부에서의 Edit 및
  Write를 차단합니다. 디버깅 시 관련 없는 코드를 실수로 "수정"하는 것을 방지하거나,
  변경 범위를 하나의 모듈로 한정하고 싶을 때 사용하세요. "freeze", "restrict edits",
  "only edit this folder", "lock down edits" 요청 시 사용합니다. (gstack)
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
> **Safety Advisory:** This skill includes safety checks that verify file edits are within the allowed scope boundary before applying, and verify file writes are within the allowed scope boundary before applying. When using this skill, always pause and verify before executing potentially destructive operations. If uncertain about a command's safety, ask the user for confirmation before proceeding.


# /freeze — 편집을 특정 디렉토리로 제한

파일 편집을 특정 디렉토리로 잠급니다. 허용된 경로 외부의 파일을 대상으로 하는
Edit 또는 Write 작업은 (경고가 아닌) **차단**됩니다.

```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"freeze","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 설정

사용자에게 편집을 제한할 디렉토리를 물어보세요. AskUserQuestion을 사용합니다:

- 질문: "편집을 제한할 디렉토리를 지정해주세요. 이 경로 외부의 파일은 편집이 차단됩니다."
- 텍스트 입력 (객관식 아님) — 사용자가 경로를 입력합니다.

사용자가 디렉토리 경로를 제공하면:

1. 절대 경로로 변환합니다:
```bash
FREEZE_DIR=$(cd "<user-provided-path>" 2>/dev/null && pwd)
echo "$FREEZE_DIR"
```

2. 후행 슬래시를 추가하고 freeze 상태 파일에 저장합니다:
```bash
FREEZE_DIR="${FREEZE_DIR%/}/"
STATE_DIR="${CLAUDE_PLUGIN_DATA:-$HOME/.gstack}"
mkdir -p "$STATE_DIR"
echo "$FREEZE_DIR" > "$STATE_DIR/freeze-dir.txt"
echo "Freeze boundary set: $FREEZE_DIR"
```

사용자에게 알려주세요: "편집이 `<path>/`로 제한되었습니다. 이 디렉토리 외부에서의
Edit 또는 Write는 차단됩니다. 경계를 변경하려면 `/freeze`를 다시 실행하세요.
제거하려면 `/unfreeze`를 실행하거나 세션을 종료하세요."

## 동작 방식

훅은 Edit/Write 도구 입력 JSON에서 `file_path`를 읽은 다음, 해당 경로가
freeze 디렉토리로 시작하는지 확인합니다. 그렇지 않으면
`permissionDecision: "deny"`를 반환하여 작업을 차단합니다.

freeze 경계는 상태 파일을 통해 세션 동안 유지됩니다. 훅 스크립트는
매 Edit/Write 호출 시 이를 읽습니다.

## 참고 사항

- freeze 디렉토리의 후행 `/`는 `/src`가 `/src-old`와 일치하는 것을 방지합니다
- Freeze는 Edit 및 Write 도구에만 적용됩니다 — Read, Bash, Glob, Grep은 영향을 받지 않습니다
- 실수로 인한 편집을 방지하는 것이지 보안 경계가 아닙니다 — `sed`와 같은 Bash 명령어는 여전히 경계 외부의 파일을 수정할 수 있습니다
- 비활성화하려면 `/unfreeze`를 실행하거나 대화를 종료하세요
