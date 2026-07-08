# Projects (프로젝트 컨텍스트)

이 디렉토리는 `/orchestrate <project-name> <task>` 명령에서 사용되는 **프로젝트별 컨텍스트** 파일을 담습니다.

## ⚠️ 커밋 주의

`projects/*.md` 는 서버 주소·계정·알려진 취약점 위치 등 **민감 정보**를 담을 수 있습니다.
저장소의 `.gitignore` 는 `README.md` 와 `example.md` 를 제외한 이 디렉토리의 모든 파일을 무시합니다.
**실제 프로젝트 파일(`myapp.md` 등)은 절대 커밋하지 마세요.**

## 파일 구조

각 프로젝트는 `<project-name>.md` 파일 하나로 표현됩니다.
**파일 이름이 곧 `/orchestrate` 명령어의 `<project-name>` 인자**입니다.

예:
- `myapp.md` → `/orchestrate myapp <task>`
- `blog-site.md` → `/orchestrate blog-site <task>`
- `data-pipeline.md` → `/orchestrate data-pipeline <task>`

## 새 프로젝트 추가

1. `example.md` 를 복사해 `<new-name>.md` 생성
2. 프로젝트 정보로 채움 (아래 템플릿 참고)
3. `/orchestrate <new-name> <task>` 로 즉시 사용 가능 (재시작 불필요)

## 템플릿

```markdown
# Project: <프로젝트 이름>

(한 줄 설명)

## 기본 정보
- **프로젝트 경로**: (절대 경로)
- **스택**: (프레임워크·언어·DB)
- **운영 서버**: (있으면)
- **배포 방식**: (수동/자동)

## 주요 모듈
- **<모듈명>** (`경로`): 설명

## 디렉토리 구조
(src 기준 트리)

## 코딩 컨벤션
- (언어·버전·스타일 제약)

## 금지/주의
- (Claude가 절대 하면 안 되는 것)

## 알려진 이슈
- (현재 파악된 P0/P1 이슈)

## 외부 리소스
- SSH / 로그 / 문서 경로

## Git
- 메인 브랜치: (main/master)
- 커밋 규칙: (규칙 요약)
```

## 작성 가이드

- **구체적 파일·경로**: 추상적 설명보다 "어느 파일을 읽어야 하는지" 명시
- **금지 사항**: Claude가 절대 하면 안 되는 것 (DB 쓰기, 직접 배포 등)
- **알려진 이슈**: 매번 재발견하지 않도록 기록
- **분량**: 100~300줄 권장 (너무 길면 Worker 프롬프트 비대화)

## 예시

- [example.md](./example.md) — 채워야 할 자리를 `<...>` 플레이스홀더로 표시한 시작용 템플릿
