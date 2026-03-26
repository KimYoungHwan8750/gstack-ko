# QA 보고서: {APP_NAME}

| 필드 | 값 |
|------|-----|
| **날짜** | {DATE} |
| **URL** | {URL} |
| **브랜치** | {BRANCH} |
| **커밋** | {COMMIT_SHA} ({COMMIT_DATE}) |
| **PR** | {PR_NUMBER} ({PR_URL}) or "—" |
| **티어** | Quick / Standard / Exhaustive |
| **범위** | {SCOPE or "Full app"} |
| **소요 시간** | {DURATION} |
| **방문 페이지 수** | {COUNT} |
| **스크린샷 수** | {COUNT} |
| **프레임워크** | {DETECTED or "Unknown"} |
| **인덱스** | [전체 QA 실행 기록](./index.md) |

## 건강 점수(Health Score): {SCORE}/100

| 카테고리 | 점수 |
|----------|------|
| 콘솔 | {0-100} |
| 링크 | {0-100} |
| 시각 | {0-100} |
| 기능 | {0-100} |
| UX | {0-100} |
| 성능 | {0-100} |
| 접근성 | {0-100} |

## 우선 수정해야 할 상위 3개 항목

1. **{ISSUE-NNN}: {제목}** — {한 줄 설명}
2. **{ISSUE-NNN}: {제목}** — {한 줄 설명}
3. **{ISSUE-NNN}: {제목}** — {한 줄 설명}

## 콘솔 건강 상태

| 오류 | 횟수 | 최초 발견 |
|------|------|-----------|
| {오류 메시지} | {N} | {URL} |

## 요약

| 심각도 | 건수 |
|--------|------|
| Critical | 0 |
| High | 0 |
| Medium | 0 |
| Low | 0 |
| **합계** | **0** |

## 이슈 목록

### ISSUE-001: {간단한 제목}

| 필드 | 값 |
|------|-----|
| **심각도** | critical / high / medium / low |
| **카테고리** | visual / functional / ux / content / performance / console / accessibility |
| **URL** | {페이지 URL} |

**설명:** {무엇이 잘못되었는지, 기대 동작 vs 실제 동작.}

**재현 단계:**

1. {URL}로 이동
   ![Step 1](screenshots/issue-001-step-1.png)
2. {동작}
   ![Step 2](screenshots/issue-001-step-2.png)
3. **관찰:** {무엇이 잘못되는지}
   ![Result](screenshots/issue-001-result.png)

---

## 적용된 수정 (해당하는 경우)

| 이슈 | 수정 상태 | 커밋 | 변경된 파일 |
|------|----------|------|-------------|
| ISSUE-NNN | verified / best-effort / reverted / deferred | {SHA} | {files} |

### 수정 전/후 증거

#### ISSUE-NNN: {제목}
**수정 전:** ![Before](screenshots/issue-NNN-before.png)
**수정 후:** ![After](screenshots/issue-NNN-after.png)

---

## 회귀 테스트(Regression Tests)

| 이슈 | 테스트 파일 | 상태 | 설명 |
|------|------------|------|------|
| ISSUE-NNN | path/to/test | committed / deferred / skipped | 설명 |

### 지연된 테스트(Deferred Tests)

#### ISSUE-NNN: {제목}
**전제 조건:** {버그를 트리거하는 설정 상태}
**동작:** {사용자가 수행하는 것}
**기대 결과:** {올바른 동작}
**지연 사유:** {이유}

---

## 출시 준비 상태(Ship Readiness)

| 지표 | 값 |
|------|-----|
| 건강 점수 | {before} → {after} ({delta}) |
| 발견된 이슈 | N |
| 적용된 수정 | N (verified: X, best-effort: Y, reverted: Z) |
| 지연 | N |

**PR 요약:** "QA에서 N개 이슈 발견, M개 수정, 건강 점수 X → Y."

---

## 회귀 분석(Regression) (해당하는 경우)

| 지표 | 기준선(Baseline) | 현재 | 변동 |
|------|-----------------|------|------|
| 건강 점수 | {N} | {N} | {+/-N} |
| 이슈 | {N} | {N} | {+/-N} |

**기준선 이후 수정됨:** {목록}
**기준선 이후 신규:** {목록}
