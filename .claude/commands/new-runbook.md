# New Runbook

주어진 인수(`$ARGUMENTS`)를 기반으로 운영 런북을 생성합니다.

## 형식

`/new-runbook {topic}`

예시: `/new-runbook rolling-restart`

## 작업 순서

1. `docs/guides/operations/{topic}-runbook.md` 경로로 파일 생성
2. `docs/templates/runbook.md` 템플릿 기반으로 스캐폴딩
3. 사전 조건, 단계별 절차, 롤백, 검증 포함

## 규칙

- 각 단계는 실행 가능한 명령어 포함
- 예상 결과 및 이상 신호 명시
- 롤백 절차 필수
