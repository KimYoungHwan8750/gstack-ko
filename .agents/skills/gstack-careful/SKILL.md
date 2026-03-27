---
name: careful
version: 0.1.0
description: |
  위험한 명령어에 대한 안전 가드레일. rm -rf, DROP TABLE, force-push, git reset --hard,
  kubectl delete 등 파괴적인 명령어 실행 전 경고를 표시합니다. 사용자가 각 경고를
  재정의할 수 있습니다. 프로덕션 환경 작업, 라이브 시스템 디버깅, 공유 환경에서
  작업할 때 사용하세요. "be careful", "safety mode", "prod mode", "careful mode" 요청 시
  사용합니다.
allowed-tools:
  - Bash
  - Read
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "bash ${CLAUDE_SKILL_DIR}/bin/check-careful.sh"
          statusMessage: "Checking for destructive commands..."
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->
> **Safety Advisory:** This skill includes safety checks that check bash commands for destructive operations (rm -rf, DROP TABLE, force-push, git reset --hard, etc.) before execution. When using this skill, always pause and verify before executing potentially destructive operations. If uncertain about a command's safety, ask the user for confirmation before proceeding.


# /careful — 파괴적 명령어 가드레일

안전 모드가 **활성화**되었습니다. 모든 bash 명령어가 실행 전에 파괴적 패턴이
있는지 검사됩니다. 파괴적 명령어가 감지되면 경고가 표시되며, 계속 진행하거나
취소할 수 있습니다.

```bash
mkdir -p ~/.gstack/analytics
echo '{"skill":"careful","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

## 보호 대상

| 패턴 | 예시 | 위험 |
|---------|---------|------|
| `rm -rf` / `rm -r` / `rm --recursive` | `rm -rf /var/data` | 재귀적 삭제 |
| `DROP TABLE` / `DROP DATABASE` | `DROP TABLE users;` | 데이터 손실 |
| `TRUNCATE` | `TRUNCATE orders;` | 데이터 손실 |
| `git push --force` / `-f` | `git push -f origin main` | 히스토리 재작성 |
| `git reset --hard` | `git reset --hard HEAD~3` | 커밋되지 않은 작업 손실 |
| `git checkout .` / `git restore .` | `git checkout .` | 커밋되지 않은 작업 손실 |
| `kubectl delete` | `kubectl delete pod` | 프로덕션 영향 |
| `docker rm -f` / `docker system prune` | `docker system prune -a` | 컨테이너/이미지 손실 |

## 안전 예외

다음 패턴은 경고 없이 허용됩니다:
- `rm -rf node_modules` / `.next` / `dist` / `__pycache__` / `.cache` / `build` / `.turbo` / `coverage`

## 동작 방식

훅은 도구 입력 JSON에서 명령어를 읽고, 위의 패턴과 대조하여 일치하는 항목이
발견되면 경고 메시지와 함께 `permissionDecision: "ask"`를 반환합니다. 경고를
재정의하고 계속 진행할 수 있습니다.

비활성화하려면 대화를 종료하거나 새 대화를 시작하세요. 훅은 세션 범위입니다.
