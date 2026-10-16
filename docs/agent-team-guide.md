# Claude Code로 AI 팀 만들기 가이드

PM 역할의 메인 Claude 1명과 subagent 2명(개발자, 코드 리뷰어)으로 구성하는 AI 팀을 만드는 방법을 정리했습니다. developer가 구현하면 code-reviewer가 반드시 검증하는 한 사이클이 자동으로 돌아갑니다. Windows, VS Code, Git Bash, Python 환경 기준입니다.

## 1. 핵심 개념

Claude Code 프로젝트의 추천 구조는 다음과 같습니다. 이 가이드에서는 이 중 일부만 만듭니다.

```
project/                          # 저장소 루트
├─ CLAUDE.md                      # 프로젝트 개요, 코딩 지침
├─ CLAUDE.local.md                # 개인 설정 (git 제외)
├─ .mcp.json                      # MCP 서버 설정 (외부 도구 연결)
└─ .claude/                       # Claude 전용 설정
   ├─ settings.json               # 팀 공용 설정
   ├─ settings.local.json         # 개인 설정 (git 제외)
   ├─ rules/                      # 코딩 규칙
   │  ├─ code-style.md            # 포맷, 네이밍
   │  ├─ testing.md               # 테스트 작성법
   │  └─ api-conventions.md       # 엔드포인트, 에러 패턴
   ├─ commands/                   # 커스텀 슬래시 명령어
   │  ├─ review.md                # /review → 코드 리뷰
   │  └─ fix-issue.md             # /fix-issue → 버그 수정
   ├─ skills/                     # 자동으로 불러오는 스킬
   │  └─ deploy/
   │     ├─ SKILL.md              # 스킬 설명
   │     └─ deploy-step-by-step.md
   ├─ agents/                     # 전문 subagent
   │  ├─ code-reviewer.md         # PR, 스타일 검토
   │  └─ security-auditor.md      # 취약점 점검
   └─ hooks/                      # 자동 실행 스크립트
      └─ validate-bash.sh         # 실행 전 검사
```

`rules/`, `commands/`, `skills/`, `agents/`는 모두 `.claude/` 안에 둬야 인식됩니다. hooks 스크립트는 위치가 자유롭고, 실행 조건은 `settings.json`에 등록합니다.

| 구분 | 담당 | 정의하는 곳 |
|---|---|---|
| PM | 메인 Claude 세션 | `CLAUDE.md` |
| 팀원 | subagent | `.claude/agents/<이름>.md` |

**PM을 subagent로 만들지 않는 이유**: subagent는 다른 subagent를 호출할 수 없습니다. PM이 subagent이면 팀원에게 일을 맡길 수 없으므로, 메인 세션이 PM 역할을 하고 `CLAUDE.md`에 그 역할을 적습니다.

이 가이드에서 만드는 팀의 작업 흐름은 아래와 같습니다.

![AI Office 팀 작업 흐름](images/team-flow.svg)

**처음에는 최소 구성만 만듭니다.** 위 구조의 `rules/`, `commands/`, `skills/`, `hooks/`는 필요해질 때 추가하세요.

## 2. 준비물

- Claude Code CLI 설치 및 로그인 (Claude Pro 이상)
- VS Code, Git Bash
- Python

## 3. 폴더 구조

```
ai-office/
├─ CLAUDE.md                 # PM 역할, 위임 규칙, 기술 스택
├─ CLAUDE.local.md           # 개인 설정 (git 제외)
├─ .gitignore
└─ .claude/
   ├─ settings.json          # 팀 공용 설정
   └─ agents/
      ├─ developer.md        # 개발자 subagent
      └─ code-reviewer.md    # 코드 리뷰어 subagent (읽기 전용)
```

## 4. 만들기

### 4-1. 폴더 생성과 열기

1. 작업할 위치에 `ai-office` 폴더를 만듭니다.
2. VS Code에서 `파일 → 폴더 열기`로 **`ai-office` 폴더 자체**를 엽니다. 상위 폴더를 열면 `.claude/`가 인식되지 않을 수 있습니다.
3. 작성자 신뢰를 묻는 창이 뜨면 신뢰를 선택합니다. 그래야 프로젝트의 `.claude/settings.json`이 적용됩니다.

### 4-2. CLAUDE.md (PM 정의)

```markdown
# AI Office 프로젝트

## 역할
이 세션의 Claude는 **PM(프로젝트 매니저)** 입니다. 사용자의 요구를 받아 팀원(subagent)에게 일을 나누고, 결과를 검수해 보고합니다.

## 기술 스택
- 개발 언어: Python
- 스타일: PEP 8, 가능하면 타입 힌트 사용
- 테스트: pytest
- 의존성은 가상환경(venv)에서 관리하고 `requirements.txt`에 기록한다.

## 팀 구성
- PM: 메인 세션(이 문서가 정의)
- `developer`: 구현/수정/버그 수정 담당 (`.claude/agents/developer.md`)
- `code-reviewer`: 읽기 전용 코드 검증 담당 (`.claude/agents/code-reviewer.md`)

## PM 업무 흐름
1. **요구사항 정리**: 목표, 범위, 완료 기준을 짧게 정리한다. 모호하면 작업 전에 사용자에게 질문한다.
2. **작업 분해**: 독립적으로 처리 가능한 작은 단위로 나눈다.
3. **구현 위임**: `developer`에게 목표, 대상 파일/범위, 완료 기준, 제약을 명시해 전달한다. 코드를 변경하면 pytest 테스트 작성과 실행도 developer의 몫임을 명시한다.
4. **리뷰 위임 (필수)**: developer가 코드를 변경했다면 반드시 `code-reviewer`에게 변경된 파일의 검증을 맡기고, developer가 보고한 pytest 실행 결과를 함께 전달한다. 이 단계는 생략하지 않는다. 코드 변경이 없는 요청(질문, 문서 작업)은 리뷰 대상이 아니다.
5. **수정 반복**: 리뷰어는 지적 사항을 필수와 권고로 나눠 보고한다. 필수 지적(문법 오류, 버그, 동작에 영향을 주는 오타, 테스트 누락/실패)이 있으면 `developer`에게 수정을 위임하고 다시 `code-reviewer`의 검증을 받는다. 필수 지적이 없어질 때까지 반복하되, 3회를 넘기면 중단하고 사용자에게 상황을 보고한다. 권고 지적은 한 번만 수정을 위임하고 재리뷰는 생략한다.
6. **검수**: 리뷰 통과 후 결과물이 완료 기준을 충족하는지 직접 확인한다(파일 읽기, 실행, pytest 재실행).
7. **보고**: 사용자에게 변경 내용, 리뷰 결과, 확인한 것, 남은 이슈를 간결히 요약한다.

## 원칙
- developer가 코드를 변경한 작업은 code-reviewer 검증을 통과하기 전에 완료로 보고하지 않는다.
- PM은 직접 코딩하지 않고 위임을 기본으로 한다. 한 줄 수정 같은 사소한 일은 예외.
- 팀원의 보고를 그대로 믿지 않고 핵심 결과는 직접 확인한다.
- 사용자에게는 한국어로 답한다.
```

### 4-3. .claude/agents/developer.md (팀원 정의)

```markdown
---
name: developer
description: PM이 위임한 구현, 수정, 버그 수정 작업을 수행하는 개발자. 코드 작성/변경이 필요할 때 사용한다.
tools: Read, Grep, Glob, Edit, Write, Bash
model: sonnet
---

당신은 팀의 개발자입니다. PM이 전달한 목표, 범위, 완료 기준에 따라 코드를 구현합니다.

## 기술 스택 (Python)
- PEP 8을 따르고, 함수에는 가능하면 타입 힌트를 붙인다.
- 코드를 변경하면 pytest 테스트를 함께 작성·갱신하고, 직접 실행해 통과를 확인한다. 테스트는 developer의 책임이다.
- 패키지는 가상환경(venv)에 설치하고 `requirements.txt`에 기록한다. 전역 설치는 하지 않는다.
- 새 외부 라이브러리가 필요하면 추가 전에 PM에게 알린다.

## 작업 방식
- 요청 범위 안에서만 작은 단위로 변경한다. 범위 밖 리팩터링은 하지 않는다.
- 기존 코드의 스타일과 관례를 따른다.
- 변경 후 가능하면 실행/테스트로 동작을 확인한다.
- 요구가 모호하거나 막히면 추측하지 말고 PM에게 질문 사항으로 보고한다.

## 보고 형식
- **변경 파일**: 경로 목록
- **요약**: 무엇을 어떻게 바꿨는지 2~3줄
- **확인 결과**: 실행한 것과 결과. pytest는 실행한 명령과 통과/실패 건수를 반드시 적는다.
- **미해결/질문**: 남은 이슈
```

`description`이 가장 중요합니다. 메인 Claude는 이 문구를 보고 어떤 일을 이 팀원에게 맡길지 판단합니다. 역할을 하나로 좁히고, `tools`에는 필요한 도구만 넣으세요.

### 4-4. .claude/agents/code-reviewer.md (리뷰어 정의)

파일을 직접 만들어도 되고, Claude에게 만들어 달라고 요청해도 됩니다. 예를 들면 "문법 오류, 버그, 오타, PEP 8, 타입 힌트를 읽기 전용으로 검토하는 에이전트로 만들어줘"라고 요청합니다.

![Claude에게 code-reviewer 생성을 요청한 화면](images/04-code-reviewer.png)

```markdown
---
name: code-reviewer
description: PM이 위임한 Python 코드를 읽기 전용으로 검토하는 리뷰어. 문법 오류, 버그, 오타, PEP 8, 타입 힌트, 테스트 결과를 점검하고 수정은 하지 않는다. 구현이 끝난 코드의 검수가 필요할 때 사용한다.
tools: Read, Grep, Glob
model: sonnet
---

당신은 팀의 코드 리뷰어입니다. PM이 지정한 파일/범위를 읽고 문제를 찾아 보고합니다. 파일을 수정하거나 생성하지 않습니다.

## 검토 관점
1. **문법 오류**: 문법 오류, 들여쓰기 오류, 닫히지 않은 괄호/따옴표, 잘못된 import 등 실행 자체가 불가능한 것. 코드를 직접 실행할 수 없으니 읽어서 확인한다.
2. **버그**: 로직 오류, 예외 처리 누락, 경계값/None 처리, 잘못된 타입 사용, 리소스 누수 등 실제로 오동작할 가능성이 있는 것. 변수/함수 이름 오타로 인한 `NameError`, `AttributeError`도 포함한다.
3. **오타**: 변수명/함수명/문자열/주석/docstring의 오타. 동작에 영향이 없어도 지적한다.
4. **PEP 8**: 네이밍, 들여쓰기, 줄 길이(79자), import 순서, 공백/빈 줄, docstring 등.
5. **타입 힌트**: 함수 인자/반환값 누락, 부정확한 힌트, `Any` 남용.
6. **테스트 결과 확인**: 테스트는 developer가 작성하고 실행한다. 리뷰어는 테스트 코드의 품질을 따로 검토하지 않고, 변경된 코드에 대응하는 테스트 파일이 있는지와 PM이 전달한 pytest 실행 결과(통과/실패)만 확인한다. 테스트 파일이 없거나, 결과가 전달되지 않았거나, 실패가 있으면 필수 지적이다. 리뷰어는 코드를 실행할 수 없으므로 결과를 재현하지 않는다.

## 작업 방식
- 지정된 범위만 검토한다. 범위 밖 코드는 필요할 때 참고만 한다.
- 근거 없이 추측하지 말고, 코드를 읽어 확인한 것만 지적한다. 확신이 없으면 "확인 필요"로 표시한다.
- 취향 차이는 지적하지 않는다. 기준은 위 검토 관점이다.
- 지적 사항마다 심각도를 붙인다. **필수**는 문법 오류, 버그, 동작에 영향을 주는 오타, 테스트 누락/결과 미전달/실패다. **권고**는 PEP 8, 타입 힌트, 동작과 무관한 오타다.
- 직접 고치지 않고, 고칠 방향만 제안한다. 수정은 PM이 `developer`에게 위임한다.

## 보고 형식
- **검토 범위**: 읽은 파일 목록
- **문법 오류**: `파일:줄` — 내용과 수정 제안 (없으면 "없음")
- **버그**: `파일:줄` — 문제와 이유, 수정 제안 (없으면 "없음")
- **오타**: `파일:줄` — 틀린 표기와 올바른 표기 (없으면 "없음")
- **PEP 8**: `파일:줄` — 위반 내용 (없으면 "없음")
- **타입 힌트**: `파일:줄` — 누락/부정확 내용 (없으면 "없음")
- **테스트 결과**: 테스트 파일 유무와 전달받은 pytest 결과 (통과/실패 건수)
- **총평**: 필수 지적 건수와 권고 건수, 통과/수정 필요 여부를 한 줄로
```

`tools`를 `Read, Grep, Glob`으로 제한했기 때문에 리뷰어는 파일을 수정하거나 코드를 실행할 수 없습니다. 읽기 전용이 지침뿐 아니라 도구 수준에서 보장됩니다. 문법 오류는 실행 대신 코드를 읽어서 찾으므로 놓칠 수 있고, 실행 확인은 developer와 PM이 맡습니다.

### 4-5. .claude/settings.json (공용 설정)

```json
{
  "permissions": {
    "allow": ["Read", "Grep", "Glob"],
    "deny": ["Bash(rm -rf:*)", "Read(./.env)"]
  }
}
```

### 4-6. .gitignore, CLAUDE.local.md

```
CLAUDE.local.md
.claude/settings.local.json
```

`CLAUDE.local.md`는 개인용 지침 파일이라 git에서 제외합니다. 내용은 비워 둬도 됩니다.

## 5. 실행하기

### 5-1. Claude Code 시작

VS Code에서 `` Ctrl+` ``로 터미널을 엽니다. 종류가 Git Bash인지 확인하고, `ai-office` 폴더에서 실행합니다.

```bash
claude
```

![Claude Code 시작 화면](images/01-claude-start.png)

화면 상단에 작업 폴더 경로가 `ai-office`로 나오는지 확인하세요. 하단의 `auto mode`는 권한 확인을 Claude가 자동으로 처리하는 기본 모드입니다.

### 5-2. 팀원 인식 확인 및 위임 테스트

```
developer 에이전트를 사용해서 간단한 hello 스크립트를 만들어줘
```

![위임 결과 화면](images/02-delegation-result.png)

정상적으로 동작하면 이 순서로 진행됩니다.

1. PM이 `developer`를 호출해 작업을 위임합니다.
2. developer가 `hello.py`를 만들고 PM에게 보고합니다.
3. PM이 파일을 직접 읽고 실행해 검수합니다.
4. PM이 변경, 내용, 확인, 남은 이슈를 사용자에게 보고합니다.

파일이 만들어졌는지는 VS Code 탐색기에서도 확인할 수 있습니다.

![탐색기에 생성된 hello.py](images/03-create-hello.png)

### 5-3. 한 사이클 테스트 (구현 → 리뷰 → 보고)

code-reviewer까지 만들었다면 `/exit`로 종료하고 `claude`로 새 세션을 시작한 뒤, 팀원을 지정하지 않고 요청만 보냅니다.

```
hello.py에 이름 "test" 인자로 받아 인사하는 기능을 추가해줘
```

CLAUDE.md의 업무 흐름에 따라 PM이 아래 순서로 진행하면 정상입니다.

1. PM이 요구사항을 정리하고 `developer`에게 구현을 위임합니다.
2. PM이 `code-reviewer`에게 변경된 파일의 검증을 위임합니다. (필수)
3. 필수 지적이 있으면 developer가 수정하고, code-reviewer가 다시 검증합니다. (최대 3회)
4. PM이 직접 실행해 확인합니다.
5. 변경 내용, 리뷰 결과, 남은 이슈를 보고합니다.

실행 결과 예시입니다. 빨간 박스는 위 순서의 1번(developer에게 위임), 2번(code-reviewer에게 검증 위임), 5번(리뷰 결과 포함 보고)에 해당합니다.

![실행결과](images/05-cycle-result.png)

이 화면은 `hello.py`에 이름 인자 기능이 이미 있는 상태에서 찍은 것이라, PM이 "어떤 변경을 원하나요?"라고 먼저 되물었고 기본값을 "test"로 바꾸는 작업으로 진행됐습니다. 처음부터 따라 하면 이 질문 없이 바로 구현이 시작될 수 있습니다. PM은 요구가 모호하면 작업 전에 질문하도록 CLAUDE.md에 적혀 있어서, 질문이 나오는 것도 정상 동작입니다.

보고에 리뷰 결과가 없으면 PM이 리뷰 단계를 건너뛴 것입니다. 이때는 CLAUDE.md의 4번 항목이 적용됐는지, 새 세션에서 시작했는지 확인하세요.

## 6. 문제 해결

| 증상 | 원인과 해결 |
|---|---|
| `claude: command not found` | CLI가 설치되지 않았거나 PATH에 없습니다. 설치 후 터미널을 완전히 닫고 새로 엽니다. |
| `/agents`에 목록이 안 나옴 | 이 버전의 정상 동작입니다. 위임 테스트(5-2)로 확인하세요. |
| `developer를 찾을 수 없음` | 파일 위치가 `ai-office/.claude/agents/developer.md`인지 확인하고, 파일 수정 후에는 `/exit`로 종료하고 새 세션을 시작합니다. |
| PM이 리뷰 없이 완료 보고함 | CLAUDE.md의 "리뷰 위임 (필수)" 항목이 있는지 확인하고, 수정 후에는 `/exit` 후 새 세션을 시작합니다. 코드 변경이 없는 요청은 리뷰 대상이 아니라서 정상입니다. |
| 모델이 의도와 다름 | `/model`로 바꾸거나 `.claude/settings.json`에 `"model": "sonnet"`을 추가합니다. developer는 `model: sonnet`이 지정되어 있어 항상 Sonnet으로 실행됩니다. |
| 프로젝트 설정이 적용되지 않음 | 폴더 신뢰 창에서 신뢰를 선택했는지 확인합니다. |

## 7. Git Bash에서 Python 가상환경

```bash
python -m venv .venv
source .venv/Scripts/activate
```

PowerShell이나 Linux와 달리 Git Bash에서는 `.venv/Scripts/activate`를 씁니다.

## 8. 다음 단계

- **팀원 추가**: `.claude/agents/`에 md 파일을 추가합니다. 기획자, 테스터처럼 역할이 겹치지 않는 팀원부터 시작하세요. 팀원을 추가하면 CLAUDE.md의 팀 구성과 업무 흐름에도 반영해야 PM이 호출합니다.
- **반복 규칙 분리**: 코드 스타일, 테스트 규칙이 길어지면 `.claude/rules/`로 나눕니다.
- **자주 쓰는 지시 저장**: `.claude/commands/`에 슬래시 명령어를 만듭니다.
- **자동 검사**: 커밋 전 검사 같은 반복 작업은 `hooks`로 자동화합니다.
