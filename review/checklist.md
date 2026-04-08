# 랜딩 전 리뷰 체크리스트

## 안내

`git diff origin/main` 출력을 아래 나열된 항목에 대해 리뷰합니다. 구체적으로 — `file:line`을 인용하고 수정 방안을 제안하세요. 문제없는 항목은 건너뛰세요. 실제 문제만 지적하세요.

**2단계 리뷰:**
- **1단계 (치명적):** SQL 및 데이터 안전성, 경쟁 조건, LLM 출력 신뢰 경계, Shell Injection, 열거형 완전성을 먼저 수행합니다. 가장 높은 심각도입니다.
- **2단계 (참고):** 아래 나머지 카테고리를 수행합니다. 심각도는 낮지만 여전히 조치 대상입니다.
- **전문가 카테고리(병렬 subagent가 처리하며, 이 체크리스트에서는 처리하지 않음):** 테스트 공백, 죽은 코드, 매직 넘버, 조건부 부수 효과, 성능 및 번들 영향, 암호화 및 엔트로피. 자세한 내용은 `review/specialists/`를 참고하세요.

모든 발견 사항은 선조치 리뷰(Fix-First Review)를 통해 처리됩니다: 명백한 기계적 수정은 자동 적용되고,
진정으로 모호한 문제는 하나의 사용자 질문으로 묶어 처리합니다.

**출력 형식:**

```
Pre-Landing Review: N issues (X critical, Y informational)

**AUTO-FIXED:**
- [file:line] 문제 → 적용된 수정

**NEEDS INPUT:**
- [file:line] 문제 설명
  Recommended fix: 제안된 수정
```

문제가 없으면: `Pre-Landing Review: No issues found.`

간결하게 작성하세요. 각 이슈마다: 문제 설명 한 줄, 수정 방안 한 줄. 서문, 요약, "전반적으로 괜찮아 보입니다" 같은 문구 없이.

---

## 리뷰 카테고리

### 1단계 — 치명적(CRITICAL)

#### SQL 및 데이터 안전성
- SQL에서의 문자열 보간(interpolation) (값이 `.to_i`/`.to_f`인 경우에도 — 매개변수화된 쿼리 사용 (Rails: sanitize_sql_array/Arel; Node: prepared statements; Python: parameterized queries))
- TOCTOU 경쟁 조건(race): 원자적(atomic) `WHERE` + `update_all`이어야 할 확인 후 설정 패턴
- 모델 유효성 검증을 우회하는 직접 DB 쓰기 (Rails: update_column; Django: QuerySet.update(); Prisma: raw queries)
- N+1 쿼리: 루프/뷰에서 사용되는 연관관계에 대한 즉시 로딩(eager loading) 누락 (Rails: .includes(); SQLAlchemy: joinedload(); Prisma: include)

#### 경쟁 조건(Race Conditions) 및 동시성
- 유일성 제약 조건(uniqueness constraint) 없이 또는 중복 키 오류 캐치 및 재시도 없이 읽기-확인-쓰기 (예: 동시 삽입 처리 없이 `where(hash:).first` 후 `save!`)
- 고유 DB 인덱스 없는 find-or-create — 동시 호출 시 중복 생성 가능
- 원자적 `WHERE old_status = ? UPDATE SET new_status`를 사용하지 않는 상태 전환 — 동시 업데이트가 전환을 건너뛰거나 이중 적용할 수 있음
- 사용자 제어 데이터에 대한 안전하지 않은 HTML 렌더링 (Rails: .html_safe/raw(); React: dangerouslySetInnerHTML; Vue: v-html; Django: |safe/mark_safe) (XSS)

#### LLM 출력 신뢰 경계(Trust Boundary)
- LLM이 생성한 값(이메일, URL, 이름)이 형식 검증 없이 DB에 쓰이거나 메일러에 전달됨. 저장 전에 경량 가드(`EMAIL_REGEXP`, `URI.parse`, `.strip`) 추가 필요.
- 구조화된 도구 출력(배열, 해시)이 데이터베이스 쓰기 전 타입/형태 검사 없이 수용됨.
- LLM이 생성한 URL을 허용 목록 없이 fetch함 — URL이 내부 네트워크를 가리킬 경우 SSRF 위험 (Python: `urllib.parse.urlparse` → `requests.get`/`httpx.get` 전에 hostname을 blocklist와 대조)
- LLM 출력을 sanitization 없이 지식 베이스 또는 vector DB에 저장함 — 저장형 prompt injection 위험

#### Shell Injection (Python-specific)
- 명령 문자열에 `shell=True`와 f-string/`.format()` 보간을 함께 사용하는 `subprocess.run()` / `subprocess.call()` / `subprocess.Popen()` — 대신 인자 배열 사용
- 변수 보간이 포함된 `os.system()` — 인자 배열을 사용하는 `subprocess.run()`으로 교체
- sandboxing 없이 LLM이 생성한 코드에 `eval()` / `exec()` 사용

#### 열거형(Enum) 및 값 완전성
열거형 값, 상태 문자열, 티어 이름 또는 타입 상수가 diff에 새로 추가될 때:
- **모든 소비자를 추적하세요.** 해당 값을 switch하거나, 필터링하거나, 표시하는 각 파일을 읽으세요(grep만 하지 말고 — 읽으세요). 소비자 중 새 값을 처리하지 않는 곳이 있으면 지적하세요. 흔한 누락: 프론트엔드 드롭다운에 값을 추가했지만 백엔드 모델/계산 메서드가 이를 저장하지 않는 경우.
- **허용 목록/필터 배열을 확인하세요.** 형제 값을 포함하는 배열이나 `%w[]` 목록을 검색하세요 (예: 티어에 "revise"를 추가하는 경우, 모든 `%w[quick lfg mega]`를 찾아 필요한 곳에 "revise"가 포함되었는지 확인).
- **`case`/`if-elsif` 체인을 확인하세요.** 기존 코드가 열거형에 따라 분기하는 경우, 새 값이 잘못된 기본값으로 빠지지 않는지 확인하세요.
이를 위해: Grep을 사용하여 형제 값의 모든 참조를 찾으세요 (예: "lfg" 또는 "mega"를 grep하여 모든 티어 소비자를 찾기). 각 매치를 읽으세요. 이 단계는 diff 외부의 코드를 읽어야 합니다.

### 2단계 — 참고(INFORMATIONAL)

#### Async/Sync 혼용(Python-specific)
- `async def` endpoint 안의 동기식 `subprocess.run()`, `open()`, `requests.get()` — event loop를 block합니다. 대신 `asyncio.to_thread()`, `aiofiles`, 또는 `httpx.AsyncClient`를 사용하세요.
- async 함수 안의 `time.sleep()` — `asyncio.sleep()` 사용
- `run_in_executor()` 래핑 없이 async context에서 sync DB 호출

#### 컬럼/필드 이름 안전성
- ORM 쿼리(`.select()`, `.eq()`, `.gte()`, `.order()`)의 컬럼 이름을 실제 DB schema와 대조하세요 — 잘못된 컬럼 이름은 조용히 빈 결과를 반환하거나 삼켜진 오류를 던질 수 있음
- 쿼리 결과의 `.get()` 호출이 실제로 select된 컬럼 이름을 사용하는지 확인
- 사용 가능한 경우 schema 문서와 교차 확인

#### 죽은 코드(Dead Code) 및 일관성(버전/changelog만 — 다른 항목은 maintainability specialist가 처리)
- PR 제목과 VERSION/CHANGELOG 파일 간 버전 불일치
- 변경사항을 부정확하게 설명하는 CHANGELOG 항목 (예: X가 존재한 적 없는데 "X에서 Y로 변경")

#### LLM 프롬프트 문제
- 프롬프트의 0-인덱스 목록 (LLM은 1-인덱스를 안정적으로 반환)
- `tool_classes`/`tools` 배열에 실제로 연결된 것과 일치하지 않는 가용 도구/기능을 나열하는 프롬프트 텍스트
- 여러 곳에 명시된 단어/토큰 제한이 서로 다를 수 있음

#### 완전성 공백(Completeness Gaps)
- 완전한 버전이 CC 시간 30분 미만으로 구현 가능한 축약 구현 (예: 부분적 열거형 처리, 불완전한 오류 경로, 추가가 간단한 누락된 엣지 케이스)
- 인력 투입 추정치만 제시된 옵션 — 인력 시간과 CC+gstack 시간 모두 표시해야 함
- 누락된 테스트 추가가 "호수" 규모이지 "바다" 규모가 아닌 테스트 커버리지 공백 (예: 누락된 부정 경로 테스트, 해피 경로 구조를 미러링하는 누락된 엣지 케이스 테스트)
- 적절한 추가 코드로 100% 달성 가능한데 80-90%로 구현된 기능

#### 시간 윈도우 안전성(Time Window Safety)
- "오늘"이 24시간을 커버한다고 가정하는 날짜 키 조회 — 오전 8시 PT 리포트는 오늘 키 아래 자정→오전 8시만 조회
- 관련 기능 간 불일치하는 시간 윈도우 — 하나는 시간별 버킷, 다른 하나는 같은 데이터에 대해 일별 키 사용

#### 경계에서의 타입 강제 변환(Type Coercion at Boundaries)
- Ruby→JSON→JS 경계를 넘는 값에서 타입이 변경될 수 있는 경우(숫자 vs 문자열) — 해시/다이제스트 입력은 타입을 정규화해야 함
- 직렬화 전 `.to_s` 또는 동등한 메서드를 호출하지 않는 해시/다이제스트 입력 — `{ cores: 8 }` vs `{ cores: "8" }`는 서로 다른 해시를 생성

#### 뷰/프론트엔드(View/Frontend)
- 파셜의 인라인 `<style>` 블록 (렌더링마다 재파싱)
- 뷰에서의 O(n*m) 조회 (루프 내 `Array#find` 대신 `index_by` 해시 사용)
- DB 결과에 대한 Ruby 측 `.select{}` 필터링으로 `WHERE` 절이 될 수 있는 것 (의도적으로 선행 와일드카드 `LIKE`를 피하는 경우 제외)

#### 배포 및 CI/CD 파이프라인(Distribution & CI/CD Pipeline)
- CI/CD 워크플로우 변경 (`.github/workflows/`): 빌드 도구 버전이 프로젝트 요구사항과 일치하는지, 아티팩트 이름/경로가 올바른지, 시크릿이 하드코딩된 값이 아닌 `${{ secrets.X }}`를 사용하는지 확인
- 새 아티팩트 유형 (CLI 바이너리, 라이브러리, 패키지): 게시/릴리스 워크플로우가 존재하고 올바른 플랫폼을 대상으로 하는지 확인
- 크로스 플랫폼 빌드: CI 매트릭스가 모든 대상 OS/아키텍처 조합을 커버하는지, 또는 미테스트 항목이 문서화되어 있는지 확인
- 버전 태그 형식 일관성: `v1.2.3` vs `1.2.3` — VERSION 파일, git 태그, 게시 스크립트 전체에서 일치해야 함
- 게시 단계 멱등성(idempotency): 게시 워크플로우 재실행 시 실패하지 않아야 함 (예: `gh release create` 전에 `gh release delete`)

**지적하지 마세요:**
- 기존 자동 배포 파이프라인이 있는 웹 서비스 (Docker 빌드 + K8s 배포)
- 팀 외부로 배포되지 않는 내부 도구
- 테스트 전용 CI 변경 (게시 단계가 아닌 테스트 단계 추가)

---

## 심각도 분류

```
치명적(CRITICAL, 가장 높은 심각도):   참고(INFORMATIONAL, main agent):   전문가(SPECIALIST, parallel subagents):
├─ SQL 및 데이터 안전성               ├─ Async/Sync 혼용                  ├─ Testing specialist
├─ 경쟁 조건 및 동시성                ├─ 컬럼/필드 이름 안전성             ├─ Maintainability specialist
├─ LLM 출력 신뢰 경계                ├─ 죽은 코드(version only)          ├─ Security specialist
├─ Shell Injection                    ├─ LLM 프롬프트 문제                ├─ Performance specialist
└─ 열거형 및 값 완전성                ├─ 완전성 공백                      ├─ Data Migration specialist
                                      ├─ 시간 윈도우 안전성               ├─ API Contract specialist
                                      ├─ 경계에서의 타입 강제 변환         └─ Red Team (conditional)
                                      ├─ 뷰/프론트엔드
                                      └─ 배포 및 CI/CD 파이프라인

모든 발견 사항은 선조치 리뷰(Fix-First Review)를 통해 처리됩니다.
심각도에 따라 표시 순서와 AUTO-FIX vs ASK 분류가 결정됩니다 —
치명적 발견 사항은 ASK 쪽으로 기울고(더 위험하므로),
참고 발견 사항은 AUTO-FIX 쪽으로 기웁니다(더 기계적이므로).
```

---

## 선조치(Fix-First) 휴리스틱

이 휴리스틱은 `/review`와 `/ship` 모두에서 참조됩니다. 에이전트가 발견 사항을
자동 수정할지 사용자에게 물을지를 결정합니다.

```
AUTO-FIX (에이전트가 묻지 않고 수정):       ASK (사람의 판단 필요):
├─ 죽은 코드 / 미사용 변수                 ├─ 보안 (인증, XSS, 인젝션)
├─ N+1 쿼리 (즉시 로딩 누락)               ├─ 경쟁 조건
├─ 코드와 모순되는 오래된 주석             ├─ 설계 결정
├─ 매직 넘버 → 명명된 상수                 ├─ 대규모 수정 (>20줄)
├─ 누락된 LLM 출력 유효성 검증            ├─ 열거형 완전성
├─ 버전/경로 불일치                        ├─ 기능 제거
├─ 할당되었지만 읽히지 않는 변수           └─ 사용자에게 보이는 동작을
└─ 인라인 스타일, O(n*m) 뷰 조회             변경하는 모든 것
```

**경험 법칙:** 수정이 기계적이고 시니어 엔지니어가 논의 없이 적용할 수준이면
AUTO-FIX입니다. 합리적인 엔지니어들이 수정에 대해 의견이 다를 수 있다면
ASK입니다.

**치명적 발견 사항은 기본적으로 ASK 쪽으로 기웁니다** (본질적으로 더 위험하므로).
**참고 발견 사항은 기본적으로 AUTO-FIX 쪽으로 기웁니다** (더 기계적이므로).

---

## 억제(Suppressions) — 지적하지 마세요

- 중복이 해가 없고 가독성에 도움이 되는 "X가 Y와 중복됩니다" (예: `present?`가 `length > 20`과 중복)
- "이 임계값/상수가 선택된 이유를 설명하는 주석을 추가하세요" — 임계값은 튜닝 중 변경되며, 주석은 부패함
- 어설션이 이미 동작을 커버하는데 "이 어설션이 더 엄격할 수 있습니다"
- 일관성만을 위한 변경 제안 (다른 상수가 보호되는 방식과 일치시키기 위해 값을 조건문으로 감싸기)
- 입력이 제한되어 있고 X가 실제로 발생하지 않는 경우 "정규식이 엣지 케이스 X를 처리하지 않습니다"
- "테스트가 여러 가드를 동시에 실행합니다" — 괜찮습니다, 테스트가 모든 가드를 격리할 필요는 없습니다
- 평가 임계값 변경 (max_actionable, 최소 점수) — 이들은 경험적으로 튜닝되며 지속적으로 변경됩니다
- 무해한 no-op (예: 배열에 절대 없는 요소에 대한 `.reject`)
- 리뷰 중인 diff에서 이미 해결된 모든 것 — 댓글 달기 전에 전체 diff를 읽으세요
