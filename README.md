# ai-office

Claude Code로 만든 AI 팀 프로젝트입니다. 메인 Claude가 PM 역할을 맡고, 두 명의 subagent가 구현과 코드 리뷰를 담당합니다.

## 팀 구성

| 역할 | 담당 | 정의 파일 |
|---|---|---|
| PM | 메인 Claude 세션. 요구사항 정리, 위임, 검수, 보고 | [CLAUDE.md](CLAUDE.md) |
| developer | 구현, 수정, pytest 테스트 작성과 실행 | [.claude/agents/developer.md](.claude/agents/developer.md) |
| code-reviewer | 읽기 전용 코드 검증 (문법, 버그, 오타, PEP 8, 타입 힌트, 테스트 결과) | [.claude/agents/code-reviewer.md](.claude/agents/code-reviewer.md) |

## 작업 흐름

![AI Office 팀 작업 흐름](docs/images/team-flow.svg)

요청 하나를 보내면 PM이 아래 순서로 진행합니다.

1. 요구사항 정리
2. developer가 구현하고 pytest 실행
3. code-reviewer가 검증 (필수)
4. 필수 지적이 있으면 developer가 수정하고 재검증 (최대 3회)
5. PM이 직접 확인하고 보고

## 사용 방법

1. 이 저장소를 clone하고 폴더를 VS Code로 엽니다.
2. 터미널에서 `claude`를 실행합니다.
3. 요청을 보냅니다. 예: `hello.py에 이름 "test" 인자로 받아 인사하는 기능을 추가해줘`

`CLAUDE.md`나 `.claude/agents/`를 수정한 뒤에는 `/exit` 후 `claude`로 새 세션을 시작해야 반영됩니다.

## 문서

처음부터 만드는 방법은 [AI 팀 만들기 가이드](docs/agent-team-guide.md)를 참고하세요.
