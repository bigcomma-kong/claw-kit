---
name: orchestrate
description: 범용 오케스트레이션 스킬 (도메인 무관). "/orchestrate <project-name> <task>" 형식으로 호출. research/analysis/writing/review 4개 범용 역할만 사용합니다. 개발 작업(코드 수정·디버그·테스트)은 별도의 `orchestrate-dev` 스킬을 사용하세요.
allowed-tools: Agent, Read, Write, Edit, Bash, Grep, Glob, TaskCreate, TaskUpdate, TaskList
---

# Orchestrate (범용)

자연어 업무를 받아 계층형(Hierarchical) 오케스트레이션으로 처리합니다.

- **프로젝트 컨텍스트**: `projects/<project-name>.md` 에서 로드
- **역할 정의**: `roles/<role>.md` 에서 로드 (research / analysis / writing / review)
- **실행**: Planner → Workers(N개 병렬) → Reviewer → 파일 저장

## 도메인 분리

| 스킬 | 용도 | 역할 수 |
|------|------|---------|
| **orchestrate** (이 스킬) | 일반 정보 수집·분석·문서화 업무 | 4개 |
| **orchestrate-dev** | 개발 전담 (코드·디버그·테스트·리팩터링·리뷰) | 6개 |

업무 성격이 **코드 수정·디버깅·테스트**가 주면 `/orchestrate-dev` 를 사용하세요.
이 스킬은 **조사·분석·보고서·의사결정 보조** 등 비개발 업무에 최적화되어 있습니다.

---

## 입력 파싱

`$ARGUMENTS` 를 공백 기준으로 분리:
- **첫 단어** → `PROJECT_NAME`
- **나머지 전체** → `USER_TASK`

### 예외 처리

| 상황 | 동작 |
|------|------|
| `$ARGUMENTS` 빈 입력 | `projects/*.md` 목록 읽고 "어떤 프로젝트에서 어떤 업무?" 질문 후 중단 |
| 첫 단어에 해당하는 `projects/<name>.md` 없음 | projects 목록 제시 + "프로젝트 지정 필요" 후 중단 |
| 프로젝트만 있고 task 없음 | "프로젝트 `{name}` 에서 어떤 업무?" 질문 후 중단 |
| `PROJECT_NAME` 이 확실히 업무 문장처럼 보임 (한글 서술어 포함) | "프로젝트 이름을 먼저 지정해주세요" 후 중단 |

### 사용 예시
- `/orchestrate myapp 개선점 찾아줘` → PROJECT=myapp, TASK=개선점 찾아줘
- `/orchestrate myapp` → TASK 없음 → 질문
- `/orchestrate` → 전부 없음 → 프로젝트 목록 + 질문

---

## 절대 규칙 (위반 시 즉시 중단)

### NEVER
- 외부 AI API(예: Groq / Gemini / DeepSeek / OpenAI / Mistral / OpenRouter 등) **직접 호출 금지**
- `curl` 로 AI 엔드포인트(`api.groq.com`, `generativelanguage.googleapis.com` 등) 접근 금지
- 한도 초과 시 **우회·fallback 금지** — 현재까지 결과만 저장하고 중단
- 프로젝트 운영 DB에 **쓰기 쿼리 금지** (SELECT만 허용, 사용자 명시 승인 시 예외)

### ALWAYS
- 모든 AI 추론은 **Agent 도구로 subagent 소환** (현재 세션 모델만 사용)
- 독립 작업은 한 메시지에 Agent 여러 번 호출해서 **병렬 실행**
- 매 Phase 시작/종료 시 TaskUpdate로 상태 갱신
- 모든 subagent 프롬프트에 **PROJECT_CONTEXT + ROLE_DEF 동시 주입**

> 위 NEVER/ALWAYS 는 기본 거버넌스 예시입니다. 각자 환경에 맞게 조정하세요.

---

## Phase 0: 준비

> **경로 규칙**: 이 스킬은 글로벌 위치(`~/.claude/skills/orchestrate/`)에 설치됩니다.
> 스킬 호출 시 주입되는 `Base directory for this skill: <SKILL_BASE>` 값을 사용.
> 아래 `{SKILL_BASE}` 는 그 절대 경로로 치환합니다.
> (Windows 기준 보통: `C:\Users\<User>\.claude\skills\orchestrate`)

1. **프로젝트 컨텍스트 로드**
   - `Read` `{SKILL_BASE}/projects/{PROJECT_NAME}.md` → 변수 `PROJECT_CONTEXT`
   - 파일 없으면 `Glob` `{SKILL_BASE}/projects/*.md` 로 목록 나열 후 중단

2. **역할 정의 로드**
   - `Read` 4개 파일 (한 메시지에 4개 Read 병렬 호출):
     - `{SKILL_BASE}/roles/research.md`
     - `{SKILL_BASE}/roles/analysis.md`
     - `{SKILL_BASE}/roles/writing.md`
     - `{SKILL_BASE}/roles/review.md`
   - 각 파일 내용을 맵으로 저장: `ROLE_DEFS = {research: "...", analysis: "...", writing: "...", review: "..."}`

3. **TaskCreate × 3** (Phase 1/2/3)

4. **실행 메타데이터 생성**
   - 타임스탬프: `YYYYMMDD-HHMMSS`
   - 출력 경로: `./orchestration-output/{timestamp}-{PROJECT_NAME}-{slug}.md`
   - slug: USER_TASK 첫 40자에서 영숫자만 추출·소문자화

---

## Phase 1: 계획 수립 (Planner)

Agent 도구 1회 호출 (`subagent_type: general-purpose`).

### Planner 프롬프트 템플릿

```
당신은 {PROJECT_NAME} 프로젝트의 오케스트레이션 Planner입니다.
아래 업무를 서브태스크로 분해하세요.

[프로젝트 컨텍스트]
{PROJECT_CONTEXT 전체 내용}

[업무]
{USER_TASK}

[사용 가능한 역할 — 4개만 선택 가능]
- research: 문서·데이터·웹·파일 탐색 (사실 수집)
- analysis: 패턴·원인·이슈 도출 (해석)
- writing: 문서·보고서·요약 초안 작성
- review: 여러 결과 통합·검증·채점

(각 역할의 상세 정의는 Worker 실행 시 자동 주입됨)
(**주의**: "코드 수정·디버그·테스트·리팩터링" 업무면 이 스킬이 아닌 `orchestrate-dev` 를 사용해야 함. 해당 업무 감지 시 "이 업무는 orchestrate-dev 스킬로 실행 권장" 메시지 출력 후 중단)

[분해 기준 — 반드시 준수]

## Worker 수 결정 규칙
- **단일 파일/기능 한정 업무** → 1~2개
  예: "UserService.java의 NPE 재현 원인 찾기"
- **특정 서비스·모듈 내부 개선** → 2~3개
  예: "PaymentService 리팩터링 제안"
- **복수 도메인·전방위 진단** → 4~5개
  예: "프로젝트 전체 보안·성능 진단"
- **상한 5개 엄수** (초과 시 Worker 과잉 → 통합 품질 하락)

## parallel_group 결정 규칙
- **같은 그룹 (A)**: 입력이 독립적이고 서로의 결과를 참조하지 않는 경우
- **다른 그룹 (B, C)**: 선행 그룹의 산출을 입력으로 받아야 하는 경우
- **한 그룹 내 최대 4개** (동시 실행 안정성)
- 병렬 불가능한 작업을 억지로 같은 그룹에 넣지 말 것

## role 선택 가이드
- 파일·DB·문서·웹을 "읽기만" 함 → research
- 읽은 결과에서 "패턴·수치·원인"을 도출 → analysis
- 마크다운·보고서·제안서·요약 생성 → writing
- 여러 결과를 "통합·검증·채점" → review
- **코드 수정 제안(diff)·디버그·테스트 → 이 스킬이 아닌 orchestrate-dev 사용**

## 출력 전 자체 체크
1. 각 step의 task가 독립적으로 실행 가능한가?
2. 같은 그룹 step들이 같은 파일을 동시 수정하려 하지 않는가?
3. 업무 복잡도 대비 Worker 수가 과하거나 부족하지 않은가?
4. role 선택이 task 성격과 일치하는가?

[출력 형식 — JSON만 반환, 다른 설명 금지]
{
  "goal": "업무의 최종 목표 1문장",
  "worker_count_rationale": "왜 N개로 정했는지 1~2문장",
  "parallel_strategy": "그룹핑 근거 1~2문장",
  "steps": [
    {
      "step": 1,
      "role": "research|analysis|writing|review",
      "task": "구체적 작업 내용 (파일 경로/대상 명시)",
      "inputs": ["선행 단계 번호 또는 파일/DB 경로"],
      "parallel_group": "A|B|C"
    }
  ]
}

400 단어 이내. JSON 외 텍스트 금지.
```

### 후처리

- JSON 추출 (```json``` 블록 또는 첫 `{` ~ 마지막 `}`)
- 파싱 실패 시 1회 재시도 (프롬프트에 "유효한 JSON만" 강조 추가)
- 2회 실패 → 원본 응답을 `orchestration-output/{timestamp}-debug-plan.txt` 에 저장하고 중단
- 파싱된 `worker_count_rationale` 과 `parallel_strategy` 를 사용자에게 간결히 출력 (판단 근거 투명화)

---

## Phase 2: 작업 실행 (Workers)

`steps` 를 `parallel_group` 기준으로 묶어 순차 실행.
**같은 그룹 내 step들은 한 메시지에 Agent 여러 개로 병렬 호출**.

### 각 Worker 프롬프트 템플릿

```
당신은 {PROJECT_NAME} 프로젝트의 {ROLE} 전문가입니다.

[프로젝트 컨텍스트]
{PROJECT_CONTEXT 전체 내용}

[당신의 역할 정의]
{ROLE_DEFS[step.role] 전체 내용}

[업무 전체 목표]
{GOAL}

[이번 서브태스크]
{step.task}

[선행 결과]
{previous_outputs or "없음"}

[절대 규칙]
- 외부 AI API 직접 호출 금지
- 프로젝트 운영 DB 쓰기 금지
- curl로 AI 엔드포인트 호출 금지
- "역할 정의"에 명시된 책임 범위를 벗어나지 말 것
  (예: research 역할이 개선 제안까지 작성하면 analysis 침범)

500 단어 이내 결과 반환. "역할 정의"의 출력 포맷 준수.
```

### 실패 처리

- Worker 1개 실패 → 해당 step만 재시도 1회
- 재시도 실패 → 결과에 `[FAILED: <reason>]` 표시 후 다음 그룹 진행
- **과반 실패** → Phase 3 건너뛰고 부분 결과로 최종 보고

---

## Phase 3: 결과 검토 (Reviewer)

Agent 도구 1회 호출.

### Reviewer 프롬프트 템플릿

```
당신은 {PROJECT_NAME} 오케스트레이션 Reviewer입니다.

[프로젝트 컨텍스트]
{PROJECT_CONTEXT 전체 내용}

[당신의 역할 정의]
{ROLE_DEFS["review"] 전체 내용}

[업무]
{USER_TASK}

[목표]
{GOAL}

[Planner 판단 근거]
- Worker 수: {worker_count_rationale}
- 병렬 전략: {parallel_strategy}

[Worker 결과]
{WORKER_OUTPUTS — 각 step의 role/task/output 정리}

출력: 마크다운 최종 보고서.
섹션 구성은 역할 정의의 "출력 포맷" 준수.
1200 단어 이내.
```

---

## Phase 4: 최종 저장 및 보고

1. Reviewer 결과를 출력 경로에 저장 (`Write` 도구)
2. TaskUpdate로 모든 task completed 처리
3. 사용자에게 **간결한 요약** 출력:
   ```
   오케스트레이션 완료
   - 프로젝트: {PROJECT_NAME}
   - 업무: {USER_TASK 첫 80자}
   - Planner 판단: {worker_count_rationale 요약}
   - 실행 Worker: N개 (성공 N / 실패 N, 병렬 그룹 M개)
   - 결과 파일: {경로}
   - 소요: 약 {분}분

   [최종 결과 헤더 + 첫 섹션 미리보기]
   ```

---

## 디버깅

- Planner JSON 파싱 실패 → `orchestration-output/{timestamp}-debug-plan.txt` 에 원본 저장
- Worker 과반 실패 → 디버그 로그를 `{timestamp}-debug-workers.txt` 에 저장
- 한도 초과 감지 ("rate limit", "quota", "overloaded") → 즉시 중단 + 현재까지 결과 저장

---

## 확장

모든 확장 작업은 **글로벌 스킬 디렉토리** (`~/.claude/skills/orchestrate/`, Windows: `C:\Users\<User>\.claude\skills\orchestrate\`)에서 수행합니다.

### 새 프로젝트 추가
1. `{SKILL_BASE}/projects/<new-name>.md` 파일 생성
2. 템플릿은 `{SKILL_BASE}/projects/example.md` 참고
3. 이후 `/orchestrate <new-name> <task>` 로 즉시 사용 가능 (재시작 불필요)

### 새 역할 추가
1. `{SKILL_BASE}/roles/<new-role>.md` 파일 생성 (기존 역할 파일 포맷 준수)
2. SKILL.md Phase 0 step 2의 Read 목록에 추가
3. SKILL.md Phase 1 Planner 프롬프트의 "사용 가능한 역할" 에 추가
4. Phase 1 "role 선택 가이드" 에 선택 기준 추가

---

## 버전

- v2.1: 글로벌 위치로 이동 (`~/.claude/skills/orchestrate/`). 경로를 `{SKILL_BASE}` 기반으로 변경.
- v2.0: 프로젝트 독립화. projects/ 와 roles/ 분리. Planner 결정 규칙 및 rationale 출력 추가.
- v1.0: 초기 구현 (단일 프로젝트 하드코딩).
