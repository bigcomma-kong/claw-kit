# claw-kit

**[Claude Code](https://claude.com/claude-code)용 재사용 스킬 모음.**

🌐 **한국어** · [English](./README.md)

바로 꽂아 쓰는 스킬 컬렉션입니다. 각 스킬은 `skills/<name>/` 아래에 있고,
전역 `~/.claude/skills/` 디렉토리로 설치됩니다. 특정 회사·저장소에 종속된 내용은
없으며, 프로젝트 컨텍스트는 사용자가 직접 채웁니다.

> "배터리 포함 프리셋"(LazyVim 같은) 아이디어를 Claude Code 스킬 시스템에 적용한 것.
> 내 도구는 내가 소유하고, 비밀은 로컬에만 둡니다.

---

## 포함된 스킬

| 스킬 | 하는 일 | 역할 |
|------|---------|------|
| **orchestrate** | **비개발** 업무(조사·분석·문서·의사결정)를 위한 계층형 오케스트레이션. Planner → 병렬 Workers → Reviewer. | 4개 (research / analysis / writing / review) |
| **orchestrate-dev** | **개발** 업무(코드·디버그·테스트·리팩터·리뷰)를 위한 계층형 오케스트레이션. 동일 파이프라인, 개발 특화 역할. | 6개 (research / code / debug / test / review / refactor) |

두 스킬 모두 같은 구조를 공유합니다:

```
/orchestrate <project> <task>
        │
        ▼
   Planner (1개 에이전트)  ── 업무를 N개 서브태스크 + 병렬 그룹으로 분해
        │
        ▼
   Workers (N개 에이전트) ── 병렬 그룹 단위 실행. 각자 PROJECT_CONTEXT +
        │                     자신의 ROLE 정의를 주입받음
        ▼
   Reviewer (1개 에이전트) ── 통합·검증·채점 → 최종 마크다운 보고서
        │
        ▼
   ./orchestration-output/ 에 저장
```

### 설계 원칙

- **프로젝트 컨텍스트는 코드가 아니라 데이터.** 스킬은 엔진이고, 프로젝트 사실은
  `projects/<name>.md` 에 둡니다(기본 git-ignore).
- **역할은 단일 책임.** `research` 워커는 사실만 수집하고 수정 제안은 안 합니다
  (그건 `analysis`/`code` 담당). 출력이 깔끔하고 범위 침범이 없습니다.
- **거버넌스 내장.** 절대 규칙: 외부 AI-API fallback 금지, 운영 DB 쓰기 금지,
  명시 승인 없는 파일 수정·커밋·배포 금지. 각 `SKILL.md` 에서 조정하세요.
- **병합 말고 검증.** 선택적 반박 검증(Phase 2.5)이 Worker 주장을 Reviewer 병합
  전에 반증하고, `orchestrate-dev` 는 승인된 변경을 빌드/테스트 통과까지 반복하는
  **그린 루프**(Phase 5)를 제공합니다 — 무한 반복 방지 상한 있음.
- **기본이 병렬.** 독립 서브태스크는 동시 에이전트로 실행됩니다.

---

## 설치

스킬은 **전역** Claude Code 스킬 디렉토리로 설치됩니다
(`~/.claude/skills/` — Windows는 `C:\Users\<You>\.claude\skills\`).

```bash
git clone https://github.com/bigcomma-kong/claw-kit.git
cd claw-kit

# 원하는 스킬 복사
cp -r skills/orchestrate      ~/.claude/skills/orchestrate
cp -r skills/orchestrate-dev  ~/.claude/skills/orchestrate-dev
```

Windows (PowerShell):

```powershell
Copy-Item skills\orchestrate      "$env:USERPROFILE\.claude\skills\orchestrate"     -Recurse
Copy-Item skills\orchestrate-dev  "$env:USERPROFILE\.claude\skills\orchestrate-dev" -Recurse
```

재시작 불필요 — 다음 호출 때 Claude Code가 스킬을 인식합니다.

---

## 프로젝트 설정

각 스킬은 `projects/<project-name>.md` 에서 컨텍스트를 로드하며,
**파일 이름이 곧 `<project-name>` 인자**입니다.

```bash
cd ~/.claude/skills/orchestrate/projects
cp example.md myapp.md      # myapp.md 를 실제 프로젝트 정보로 채움
```

이제 실행:

```
/orchestrate myapp 최근 리서치 자료 정리해서 요약 보고서 만들어줘
/orchestrate-dev myapp UserService NPE 원인 추적하고 수정안 제시
```

> ⚠️ **실제 `projects/*.md` 파일은 절대 커밋하지 마세요.** 서버 주소·계정·알려진
> 이슈 위치 등이 들어갑니다. `.gitignore` 가 `projects/` 아래 `README.md`·`example.md`
> 를 제외한 모든 파일을 이미 제외합니다.

---

## 내 스킬 추가하기

`skills/` 아래에 같은 형식으로 폴더를 추가하면 됩니다:

```
skills/<your-skill>/
├── SKILL.md          # frontmatter: name, description, allowed-tools
├── roles/*.md        # 선택: 역할 정의
└── projects/         # 선택: 프로젝트별 컨텍스트 (git-ignore)
    ├── README.md
    └── example.md
```

---

## 라이선스

MIT — [LICENSE](./LICENSE) 참고.
