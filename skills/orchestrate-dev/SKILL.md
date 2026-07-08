---
name: orchestrate-dev
description: 개발 전담 오케스트레이션 스킬. "/orchestrate-dev <project-name> <task>" 형식으로 호출. 코드 수정·디버그·테스트·리팩터링·코드리뷰 같은 개발 업무에 최적화된 6개 역할(research, code, debug, test, review, refactor)을 사용합니다. 비개발 업무는 `orchestrate` 스킬을 사용하세요.
allowed-tools: Agent, Read, Write, Edit, Bash, Grep, Glob, TaskCreate, TaskUpdate, TaskList
---

# Orchestrate-Dev (개발 전담)

자연어로 개발 업무를 받아 계층형(Hierarchical) 오케스트레이션으로 처리합니다.

- **프로젝트 컨텍스트**: `projects/<project-name>.md` 에서 로드
- **역할 정의**: `roles/<role>.md` 에서 로드 (research / code / debug / test / review / refactor)
- **실행**: Planner → Workers(N개 병렬) → Reviewer → 파일 저장

## 이 스킬을 쓰는 경우

✅ 코드 수정·신규 기능 개발
✅ 버그 원인 추적·디버그
✅ 테스트 작성·실행
✅ 리팩터링·구조 개선
✅ 코드 리뷰 (보안·성능·품질)
✅ 빌드·배포 관련 진단

❌ 위 업무가 아니면 → 범용 `/orchestrate` 스킬 사용

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
| 업무가 비개발 성격으로 보임 | "이 업무는 `/orchestrate` 로 실행 권장" 안내 후 중단 |

### 사용 예시
- `/orchestrate-dev myapp UserService NPE 원인 찾고 수정안 제시`
- `/orchestrate-dev myapp ApiClient 타임아웃 추가`
- `/orchestrate-dev myapp 최근 변경 코드 리뷰`

---

## 절대 규칙 (위반 시 즉시 중단)

### NEVER
- 외부 AI API(예: Groq / Gemini / DeepSeek / OpenAI / Mistral / OpenRouter 등) **직접 호출 금지**
- `curl` 로 AI 엔드포인트 접근 금지
- 한도 초과 시 **우회·fallback 금지**
- **사용자 명시 승인 없이 파일 수정·커밋·빌드·배포 금지**
- 프로젝트 운영 DB 쓰기 금지

### ALWAYS
- 모든 AI 추론은 **Agent 도구로 subagent 소환**
- 독립 작업은 한 메시지에 Agent 여러 번 호출해서 **병렬 실행**
- `code` 역할은 diff/변경 제안만, 실제 수정은 별도 승인 단계
- 매 Phase 시작/종료 시 TaskUpdate 상태 갱신
- 모든 subagent 프롬프트에 **PROJECT_CONTEXT + ROLE_DEF 동시 주입**

> 위 NEVER/ALWAYS 는 기본 거버넌스 예시입니다. 각자 환경에 맞게 조정하세요.

---

## Phase 0: 준비

> **경로 규칙**: 이 스킬은 글로벌 위치(`~/.claude/skills/orchestrate-dev/`)에 설치.
> 스킬 호출 시 주입되는 `Base directory for this skill: <SKILL_BASE>` 값 사용.

1. **프로젝트 컨텍스트 로드**
   - `Read` `{SKILL_BASE}/projects/{PROJECT_NAME}.md` → 변수 `PROJECT_CONTEXT`
   - 파일 없으면 `Glob` `{SKILL_BASE}/projects/*.md` 로 목록 나열 후 중단

2. **역할 정의 로드**
   - `Read` 6개 파일 (한 메시지에 6개 Read 병렬 호출):
     - `{SKILL_BASE}/roles/research.md`
     - `{SKILL_BASE}/roles/code.md`
     - `{SKILL_BASE}/roles/debug.md`
     - `{SKILL_BASE}/roles/test.md`
     - `{SKILL_BASE}/roles/review.md`
     - `{SKILL_BASE}/roles/refactor.md`
   - 각 파일 내용을 맵으로 저장

3. **TaskCreate × 3** (Phase 1/2/3)

4. **실행 메타데이터**
   - 타임스탬프: `YYYYMMDD-HHMMSS`
   - 출력 경로: `./orchestration-output/{timestamp}-dev-{PROJECT_NAME}-{slug}.md`

---

## Phase 1: 계획 수립 (Planner)

Agent 도구 1회 호출 (`subagent_type: general-purpose`).

### Planner 프롬프트 템플릿

```
당신은 {PROJECT_NAME} 프로젝트의 개발 오케스트레이션 Planner입니다.
아래 개발 업무를 서브태스크로 분해하세요.

[프로젝트 컨텍스트]
{PROJECT_CONTEXT 전체 내용}

[업무]
{USER_TASK}

[사용 가능한 역할 — 6개만 선택 가능]
- research: 코드베이스·문서·커밋 탐색 (Grep/Glob/Read 중심)
- code: 코드 수정 제안 (diff/before-after, 실제 수정은 사용자 승인 후)
- debug: 버그 재현·원인 추적·스택트레이스 분석
- test: 테스트 작성·실행·커버리지 확인
- review: 코드 리뷰 (보안·성능·스타일·아키텍처)
- refactor: 구조 개선 제안 (파일 분할·패턴 적용·중복 제거)

[분해 기준 — 반드시 준수]

## Worker 수 결정 규칙
- **단일 파일/기능 한정** → 1~2개
  예: "TokenUtil의 하드코딩된 키 분리 수정안"
- **특정 서비스·모듈 개선** → 2~3개
  예: "OrderService 분할 리팩터링"
- **복수 도메인·전방위 진단/개편** → 4~6개
  예: "프로젝트 전체 보안·성능·품질 리뷰 + 우선순위 개선안"
- **상한 6개 엄수**

## parallel_group 결정 규칙
- **같은 그룹**: 입력 독립적, 서로 결과 참조 안함
- **다른 그룹**: 선행 결과가 입력이어야 함
- **한 그룹 내 최대 4개**
- 같은 파일 동시 수정 제안 금지

## role 선택 가이드
- 코드·문서·커밋을 "읽어서 파악" → research
- diff/patch 형식 변경 제안 → code
- 실패 원인·스택 추적 → debug
- 테스트 작성/실행/커버리지 확인 → test
- 보안·성능·스타일 점검 → review
- 구조 개선·패턴 적용 제안 → refactor

## 출력 전 자체 체크
1. 각 step이 독립 실행 가능한가?
2. 같은 그룹 step들이 같은 파일을 동시 수정 시도하지 않는가?
3. 업무 복잡도 대비 Worker 수 과부족 없는가?
4. role 선택이 적절한가?

[출력 형식 — JSON만 반환, 다른 설명 금지]
{
  "goal": "업무의 최종 목표 1문장",
  "worker_count_rationale": "왜 N개로 정했는지 1~2문장",
  "parallel_strategy": "그룹핑 근거 1~2문장",
  "steps": [
    {
      "step": 1,
      "role": "research|code|debug|test|review|refactor",
      "task": "구체적 작업 내용 (파일 경로/대상 명시)",
      "inputs": ["선행 단계 번호 또는 파일 경로"],
      "parallel_group": "A|B|C"
    }
  ]
}

400 단어 이내. JSON 외 텍스트 금지.
```

### 후처리

- JSON 추출 (```json``` 블록 또는 첫 `{` ~ 마지막 `}`)
- 파싱 실패 시 1회 재시도 ("유효한 JSON만" 강조 추가)
- 2회 실패 → `orchestration-output/{timestamp}-debug-plan.txt` 저장 후 중단
- `worker_count_rationale` 과 `parallel_strategy` 를 사용자에게 간결히 출력

---

## Phase 2: 작업 실행 (Workers)

`steps` 를 `parallel_group` 기준으로 묶어 순차 실행.
**같은 그룹 step들은 한 메시지에 Agent 여러 개로 병렬 호출**.

### 각 Worker 프롬프트 템플릿

```
당신은 {PROJECT_NAME} 프로젝트의 {ROLE} 전문가입니다.

[프로젝트 컨텍스트]
{PROJECT_CONTEXT 전체}

[당신의 역할 정의]
{ROLE_DEFS[step.role] 전체}

[업무 전체 목표]
{GOAL}

[이번 서브태스크]
{step.task}

[선행 결과]
{previous_outputs or "없음"}

[절대 규칙]
- 외부 AI API 직접 호출 금지
- 사용자 명시 승인 없이 파일 수정·커밋·빌드 금지
- 프로젝트 운영 DB 쓰기 금지
- 역할 정의에 명시된 책임 범위를 벗어나지 말 것

500 단어 이내 결과 반환. 역할 정의의 "출력 포맷" 준수.
```

### 실패 처리

- Worker 1개 실패 → 해당 step만 재시도 1회
- 재시도 실패 → 결과에 `[FAILED: <reason>]` 표시 후 다음 그룹 진행
- **과반 실패** → Phase 3 건너뛰고 부분 결과로 최종 보고

---

## Phase 3: 결과 검토 (Reviewer)

Agent 도구 1회 호출. 프롬프트 템플릿은 `orchestrate` 와 동일하되 프로젝트 컨텍스트·review 역할 정의 주입.

### 출력 포맷 (Reviewer가 작성할 최종 문서)

```
# {업무 제목}

## 실행 개요
- 업무: ...
- 목표: ...
- 실행 단계: N개

## 결과
(Worker별 결과 통합·재구성)

## 코드 변경 제안 (있을 경우)
파일별 diff 목록. 실제 적용은 사용자 승인 필요.

## 품질 평가
- 요구사항 충족도: X/10
- 근거 충실도: X/10
- 개선 제안 (최대 3개)

## 다음 단계
- 사용자 승인 후 실행할 명령 (있으면)
- 추가 검토 필요 영역
```

---

## Phase 4: 최종 저장 및 보고

1. 결과를 `./orchestration-output/{timestamp}-dev-{PROJECT_NAME}-{slug}.md` 에 저장
2. TaskUpdate 전체 completed
3. 요약 출력:
   ```
   개발 오케스트레이션 완료
   - 프로젝트: {PROJECT_NAME}
   - 업무: {USER_TASK 첫 80자}
   - Planner 판단: {worker_count_rationale 요약}
   - 실행 Worker: N개 (성공 N / 실패 N, 병렬 그룹 M개)
   - 결과 파일: {경로}
   - 소요: 약 {분}분

   [최종 결과 헤더 + 첫 섹션 미리보기]

   [코드 변경 제안 있을 경우]
   → 사용자 승인 후 별도 세션에서 적용 필요
   ```

---

## 디버깅

- Planner JSON 파싱 실패 → `{timestamp}-debug-plan.txt`
- Worker 과반 실패 → `{timestamp}-debug-workers.txt`
- 한도 초과 감지 → 즉시 중단 + 부분 결과 저장

---

## 확장

### 새 프로젝트 추가
`{SKILL_BASE}/projects/<new-name>.md` 파일 생성. 템플릿은 `projects/example.md` 참고.

### 새 역할 추가
1. `{SKILL_BASE}/roles/<new-role>.md` 생성
2. SKILL.md Phase 0 step 2 Read 목록에 추가
3. Planner 프롬프트 "사용 가능한 역할" 에 추가
4. "role 선택 가이드" 에 기준 추가

### 다른 도메인 업무라면
- 비개발(조사·문서) → `/orchestrate`
- 콘텐츠·비즈니스 → 추후 `/orchestrate-content`, `/orchestrate-business` (미구현)

---

## 버전

- v1.0: 초기 구현. 6개 개발 전담 역할 (research, code, debug, test, review, refactor).
