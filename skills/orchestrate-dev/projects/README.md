# Projects (프로젝트 컨텍스트)

이 디렉토리는 `/orchestrate-dev <project-name> <task>` 명령에서 사용되는 **프로젝트별 컨텍스트** 파일을 담습니다.

## ⚠️ 커밋 주의

`projects/*.md` 는 서버 주소·계정·알려진 취약점 위치 등 **민감 정보**를 담을 수 있습니다.
저장소의 `.gitignore` 는 `README.md` 와 `example.md` 를 제외한 이 디렉토리의 모든 파일을 무시합니다.
**실제 프로젝트 파일(`myapp.md` 등)은 절대 커밋하지 마세요.**

## 파일 구조

각 프로젝트는 `<project-name>.md` 파일 하나로 표현됩니다.
**파일 이름이 곧 `/orchestrate-dev` 명령어의 `<project-name>` 인자**입니다.

예:
- `myapp.md` → `/orchestrate-dev myapp <task>`
- `api-server.md` → `/orchestrate-dev api-server <task>`

`orchestrate` 와 `orchestrate-dev` 는 서로 다른 스킬이므로 프로젝트 파일도 **각각** 둡니다.
동일 프로젝트를 두 스킬에서 쓰려면 같은 내용을 양쪽 `projects/` 에 두거나, 심볼릭 링크를 사용하세요.

## 새 프로젝트 추가

1. `example.md` 를 복사해 `<new-name>.md` 생성
2. 프로젝트 정보로 채움
3. `/orchestrate-dev <new-name> <task>` 로 즉시 사용 가능 (재시작 불필요)

## 작성 가이드

- **구체적 파일·경로**: 추상적 설명보다 "어느 파일을 읽어야 하는지" 명시
- **금지 사항**: 승인 없이 파일 수정·커밋·빌드·배포 등
- **알려진 이슈**: 매번 재발견하지 않도록 기록
- **분량**: 100~300줄 권장 (너무 길면 Worker 프롬프트 비대화)

## 예시

- [example.md](./example.md) — 채워야 할 자리를 `<...>` 플레이스홀더로 표시한 시작용 템플릿
