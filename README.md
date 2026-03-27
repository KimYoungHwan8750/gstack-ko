# gstack-ko

[garrytan/gstack](https://github.com/garrytan/gstack)의 한국어 fork.

gstack은 Claude Code를 가상 엔지니어링 팀으로 만들어주는 스킬 모음입니다. CEO가 제품을 재정의하고, 엔지니어링 매니저가 아키텍처를 확정하고, 디자이너가 AI slop을 잡아내고, 리뷰어가 프로덕션 버그를 찾고, QA가 실제 브라우저를 열고, 보안 책임자가 OWASP + STRIDE 감사를 돌리고, 릴리스 엔지니어가 PR을 배포합니다. 전부 슬래시 커맨드, 전부 Markdown, 전부 무료, MIT 라이선스.

## 원본과 다른 점

| 항목 | 내용 |
|------|------|
| **한글화** | 28개 스킬 전체 + 지원 파일 6개 한글 번역. 고유명사/개발용어/코드블록은 원문 유지. |
| **yhlib 통합** | yhlib 모노레포 자동 감지 시 프레임워크 선택 건너뛰기, 확정 스택 참조, `apps/` 앱 선택 안내. 미감지 시 기존 동작 유지. |
| **한국 규제/문화** | CSO에 개인정보 보호법/정보통신망법 점검, benchmark에 한국 기업 성능 기준, office-hours/plan-ceo-review에 한국 스타트업 생태계 컨텍스트 추가. |
| **업그레이드** | `git reset --hard` 대신 three-way merge로 한글화 보존하며 upstream 병합. |

## 빠른 시작

```bash
# clone
git clone -b ko https://github.com/KimYoungHwan8750/gstack-ko.git ~/.claude/skills/gstack

# upstream 등록 (업그레이드용)
cd ~/.claude/skills/gstack
git remote add upstream https://github.com/garrytan/gstack.git
```

Claude Code 재시작하면 스킬이 활성화됩니다. 상세 설치 가이드는 [SETUP.md](SETUP.md)를 참조하세요.

### browse 스킬 활성화 (선택)

`/browse`, `/qa`, `/design-review` 등 브라우저 기반 스킬을 쓰려면:

```bash
cd ~/.claude/skills/gstack
bun install && bun run build && ./setup
```

## 스킬 목록

### 스프린트 워크플로우

**생각 → 계획 → 구현 → 리뷰 → 테스트 → 배포 → 회고**

| 스킬 | 역할 | 설명 |
|------|------|------|
| `/office-hours` | YC 오피스 아워 | 아이디어 브레인스토밍 + 디자인 문서 작성. 스타트업/빌더 모드. |
| `/plan-ceo-review` | CEO / 창업자 | 문제 재정의, 10-star 제품 찾기. 범위 확장/유지/축소 모드. |
| `/plan-eng-review` | 엔지니어링 매니저 | 아키텍처, 데이터 흐름, 다이어그램, 엣지 케이스, 테스트 커버리지. |
| `/plan-design-review` | 시니어 디자이너 | 디자인 차원별 0-10 평가 + 개선. AI Slop 감지. |
| `/design-consultation` | 디자인 파트너 | 디자인 시스템 구축, DESIGN.md 생성. |
| `/review` | 스태프 엔지니어 | PR diff 분석, SQL 안전성, LLM 신뢰 경계, 조건부 부작용 검출. |
| `/investigate` | 디버거 | 근본 원인 조사. 철칙: 근본 원인 없이 수정하지 않는다. |
| `/design-review` | 코딩하는 디자이너 | 시각적 QA + 소스 코드 수정. 원자적 커밋 + 전후 스크린샷. |
| `/qa` | QA 리드 | 실제 브라우저로 테스트, 버그 찾기, 수정, 재검증. |
| `/qa-only` | QA 리포터 | 테스트만, 수정 없음. 버그 리포트 생성. |
| `/cso` | 보안 책임자 | OWASP Top 10 + STRIDE + 한국 개인정보 보호법 점검. |
| `/ship` | 릴리스 엔지니어 | 테스트 → 리뷰 → VERSION 범프 → CHANGELOG → PR 생성. |
| `/land-and-deploy` | 릴리스 엔지니어 | PR merge → CI 대기 → 프로덕션 검증. |
| `/canary` | SRE | 배포 후 모니터링. 콘솔 에러, 성능 회귀, 페이지 장애 감시. |
| `/benchmark` | 성능 엔지니어 | Core Web Vitals 기준선 + PR별 전후 비교. 한국 기업 벤치마크 포함. |
| `/document-release` | 테크니컬 라이터 | 배포 후 README/CHANGELOG/CLAUDE.md 자동 업데이트. |
| `/retro` | 엔지니어링 매니저 | 주간 회고. 커밋 분석, 코드 품질 추세, 개인별 기여도. |
| `/browse` | QA 엔지니어 | 헤드리스 Chromium. 명령당 ~100ms. |
| `/autoplan` | 리뷰 파이프라인 | CEO → 디자인 → 엔지니어링 리뷰 자동 실행. |

### 유틸리티

| 스킬 | 설명 |
|------|------|
| `/codex` | Codex 세컨드 오피니언 — 독립적 코드 리뷰, 적대적 챌린지, 상담 모드. |
| `/careful` | 파괴적 명령어 경고 (rm -rf, DROP TABLE, force-push 등). |
| `/freeze` | 편집을 특정 디렉토리로 제한. |
| `/guard` | `/careful` + `/freeze` 결합. 프로덕션 작업용 최대 안전 모드. |
| `/unfreeze` | freeze 해제. |
| `/setup-deploy` | `/land-and-deploy`용 배포 설정 (Vercel, Fly.io, Render 등). |
| `/setup-browser-cookies` | 실제 브라우저 쿠키를 헤드리스 세션으로 가져오기. |
| `/gstack-upgrade` | gstack 업그레이드. three-way merge로 한글화 보존. |

## 업그레이드

Claude Code에서 `/gstack-upgrade` 실행. 또는 수동:

```bash
cd ~/.claude/skills/gstack
git fetch upstream
git merge upstream/main
bun run gen:skill-docs && ./setup
```

## 라이선스

MIT. 원본: [garrytan/gstack](https://github.com/garrytan/gstack).
