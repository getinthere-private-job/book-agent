# Book Agent - Claude Code Skills & Agents 가이드

> 책을 쓰는 AI Agent 시스템을 구축하기 위한 프로젝트
> Claude Code의 Skill과 Agent를 활용하여 책 집필 워크플로우를 자동화합니다.

---

## 목차

1. [Claude Code 기본 사용법](#1-claude-code-기본-사용법)
2. [CLAUDE.md - 프로젝트 메모리](#2-claudemd---프로젝트-메모리)
3. [Skill 만들기](#3-skill-만들기)
4. [Agent (Subagent) 만들기](#4-agent-subagent-만들기)
5. [Skill Creator 사용법](#5-skill-creator-사용법)
6. [Agent SDK](#6-agent-sdk)
7. [프로젝트 구조 설계](#7-프로젝트-구조-설계)

---

## 1. Claude Code 기본 사용법

### 설치 및 실행

```bash
# Claude Code 설치
npm install -g @anthropic-ai/claude-code

# 프로젝트 디렉토리에서 실행
cd your-project
claude
```

### 핵심 슬래시 명령어

| 명령어 | 설명 |
|--------|------|
| `/init` | CLAUDE.md 초기화 (프로젝트 메모리 생성) |
| `/help` | 도움말 |
| `/agents` | 에이전트 관리 (생성/조회) |
| `/skills` | 스킬 관리 |
| `/compact` | 컨텍스트 압축 |
| `/clear` | 대화 초기화 |
| `/cost` | 비용 확인 |
| `/background` | 백그라운드 실행 |

### 권한 모드

| 모드 | 설명 |
|------|------|
| `default` | 모든 작업에 사용자 승인 필요 |
| `acceptEdits` | 파일 편집 자동 승인, 나머지는 승인 필요 |
| `bypassPermissions` | 모든 작업 자동 승인 (CI/CD용) |
| `dontAsk` | 승인 필요 시 자동 거부 |
| `plan` | 읽기 전용 탐색 모드 |

### 키보드 단축키

| 단축키 | 기능 |
|--------|------|
| `Enter` | 메시지 전송 |
| `Ctrl+C` | 현재 작업 중단 |
| `Ctrl+B` | 백그라운드 실행 |
| `Esc` | 작업 취소 |

---

## 2. CLAUDE.md - 프로젝트 메모리

### 개요

CLAUDE.md는 **프로젝트별 지속 메모리 파일**입니다. Claude는 세션 시작 시 이 파일의 처음 200줄을 자동으로 로드합니다.

### 메모리 계층 구조 (우선순위 순)

| 위치 | 범위 | 설명 |
|------|------|------|
| `./CLAUDE.md` | 프로젝트 전체 | 팀 공유 규칙 |
| `./.claude/CLAUDE.md` | 프로젝트 전체 | 팀 공유 규칙 (대안 위치) |
| `./.claude/rules/*.md` | 모듈별 규칙 | 주제별 분리된 규칙 |
| `~/.claude/CLAUDE.md` | 개인 전역 | 모든 프로젝트에 적용 |
| `./CLAUDE.local.md` | 개인/프로젝트 | git에 포함하지 않는 개인 설정 |

### 초기화

```bash
claude
> /init
```

### CLAUDE.md 작성 예시

```markdown
# 프로젝트명

## 개요
프로젝트에 대한 간단한 설명

## 기술 스택
- 언어: TypeScript
- 프레임워크: Next.js
- 빌드: npm

## 명령어
- 빌드: `npm run build`
- 테스트: `npm test`
- 개발 서버: `npm run dev`

## 코드 스타일
- 2칸 들여쓰기 사용
- 함수명은 camelCase
- 컴포넌트명은 PascalCase

## 중요한 컨텍스트
- 인증 처리: `src/auth/`
- API 엔드포인트: `src/api/`
```

### 모듈별 규칙 파일 (.claude/rules/)

경로 기반으로 조건부 적용이 가능합니다:

```yaml
# .claude/rules/api-design.md
---
paths:
  - "src/api/**/*.ts"
---

# API 설계 규칙
- 모든 API 엔드포인트에 입력 검증 포함
- 표준 에러 응답 형식 사용
```

### 외부 파일 참조

```markdown
## Git 워크플로우
@docs/git-instructions.md

## 프로젝트 README
@README 참고
```

---

## 3. Skill 만들기

### 개요

**Skill = 재사용 가능한 지시사항 세트**

Skill은 Claude에게 특정 작업을 반복적으로 수행하는 방법을 가르칩니다. 최소 구성은 `SKILL.md` 파일 하나입니다.

### Skill의 핵심 특성

- **이식성**: Claude 생태계 전반에서 일관되게 작동
- **효율성**: 필요한 최소 컴포넌트만 로드
- **자동/수동 호출**: Claude가 자동으로 호출하거나 `/skill-name`으로 수동 호출

### Skill 저장 위치

| 위치 | 범위 | 우선순위 |
|------|------|----------|
| `~/.claude/skills/<skill-name>/` | 개인 (모든 프로젝트) | 높음 |
| `.claude/skills/<skill-name>/` | 프로젝트 전용 | 낮음 |

### Skill 생성 단계

#### Step 1: 디렉토리 생성

```bash
mkdir -p .claude/skills/my-skill
```

#### Step 2: SKILL.md 작성

```yaml
---
name: my-skill
description: 스킬의 용도를 설명. Claude가 이 설명을 보고 자동 호출 여부를 결정합니다.
---

스킬이 수행할 작업에 대한 상세 지시사항을 여기에 작성합니다.

1. **첫 번째 단계**: 무엇을 할지 설명
2. **두 번째 단계**: 어떻게 할지 설명
3. **세 번째 단계**: 결과물 형식
```

#### Step 3: 테스트

```bash
# 수동 호출
claude
> /my-skill 인자값

# 또는 자연어로 요청하면 Claude가 자동으로 스킬을 호출
```

### SKILL.md 프론트매터 필드

| 필드 | 필수 | 설명 |
|------|------|------|
| `name` | 아니오 | 표시 이름 (소문자, 하이픈만, 최대 64자) |
| `description` | 권장 | 스킬 사용 시점 설명. Claude가 자동 호출 판단에 사용 |
| `argument-hint` | 아니오 | 자동완성 힌트. 예: `[파일명]`, `[주제]` |
| `disable-model-invocation` | 아니오 | `true`로 설정 시 자동 호출 방지 (수동만 가능) |
| `user-invocable` | 아니오 | `false`로 설정 시 `/` 메뉴에서 숨김 |
| `allowed-tools` | 아니오 | 승인 없이 사용 가능한 도구 목록 |
| `model` | 아니오 | 사용할 모델: `sonnet`, `opus`, `haiku`, `inherit` |
| `context` | 아니오 | `fork`으로 설정 시 격리된 서브에이전트에서 실행 |
| `agent` | 아니오 | `context: fork` 시 사용할 서브에이전트 |

### 문자열 치환 변수

```yaml
---
name: session-logger
description: 세션 활동 로깅
---

$ARGUMENTS 내용을 logs/${CLAUDE_SESSION_ID}.log에 기록하세요.
```

| 변수 | 설명 |
|------|------|
| `$ARGUMENTS` | 스킬에 전달된 모든 인자 |
| `$ARGUMENTS[N]` 또는 `$N` | N번째 인자 |
| `${CLAUDE_SESSION_ID}` | 현재 세션 ID |

### 동적 컨텍스트 주입

셸 명령어 결과를 스킬에 주입할 수 있습니다:

```yaml
---
name: pr-summary
description: PR 변경사항 요약
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## PR 컨텍스트
- PR diff: !`gh pr diff`
- 변경된 파일: !`gh pr diff --name-only`

## 할 일
이 PR을 요약하세요...
```

### 복합 스킬 구조

```
my-skill/
├── SKILL.md          # 필수 - 개요 및 네비게이션
├── reference.md      # 상세 참조 문서
├── examples.md       # 사용 예시
└── templates/
    └── template.md   # 템플릿 파일
```

### Skill 실전 예시: 블로그 글쓰기

```yaml
---
name: write-blog
description: 블로그 글을 작성합니다. 주제를 받아 구조화된 블로그 포스트를 생성합니다.
argument-hint: [주제]
allowed-tools: Read, Write, WebSearch
---

# 블로그 글쓰기 스킬

$ARGUMENTS 주제로 블로그 글을 작성하세요.

## 작성 순서

1. **리서치**: 주제에 대해 웹 검색으로 최신 정보 수집
2. **개요 작성**: 다음 구조로 개요를 만듦
   - 제목 (눈길을 끄는)
   - 도입부 (왜 이 주제가 중요한지)
   - 본문 (3-5개 섹션)
   - 결론 (핵심 요약 + 행동 유도)
3. **초안 작성**: 각 섹션을 상세하게 작성
4. **검토**: 흐름, 명확성, 오류 확인

## 스타일 가이드
- 대화체 사용
- 한 문단은 3-4문장
- 각 섹션에 소제목 사용
- 코드 예시가 필요하면 포함
```

---

## 4. Agent (Subagent) 만들기

### 개요

**Agent = Skill을 조합하여 작업을 수행하는 전문 AI 어시스턴트**

각 Agent는 독립된 컨텍스트 윈도우에서 실행되며, 고유한 시스템 프롬프트, 도구 접근 권한, 권한 설정을 가집니다.

### 내장 서브에이전트

| 에이전트 | 모델 | 도구 | 용도 |
|----------|------|------|------|
| Explore | Haiku (빠름) | 읽기 전용 | 파일 탐색, 코드 이해 |
| Plan | 상속 | 읽기 전용 | 구현 계획 수립 |
| General-purpose | 상속 | 모든 도구 | 복합 멀티스텝 작업 |

### Agent 생성 방법

#### 방법 1: `/agents` 명령어 사용

```bash
claude
> /agents
# "Create new agent" 선택 → 범위 선택 (user/project)
```

#### 방법 2: 파일 기반 수동 생성

에이전트 파일 위치:
- **프로젝트**: `.claude/agents/<agent-name>/AGENT.md`
- **개인**: `~/.claude/agents/<agent-name>/AGENT.md`

#### AGENT.md 작성

```yaml
---
name: book-writer
description: 책 집필 전문가. 구조화된 책을 작성하고 편집합니다.
tools: Read, Write, Edit, Glob, Grep, WebSearch
model: opus
---

당신은 전문 작가입니다. 호출되면:

1. 주제를 분석하고 목차를 생성합니다
2. 각 장(chapter)을 순서대로 작성합니다
3. 일관된 톤과 스타일을 유지합니다
4. 완성된 원고를 검토합니다

## 작성 규칙
- 명확하고 읽기 쉬운 문체
- 각 장은 독립적으로 이해 가능해야 함
- 실용적인 예시 포함
- 전문 용어 사용 시 설명 추가
```

#### 방법 3: CLI 플래그 사용

```bash
claude --agents '{
  "book-writer": {
    "description": "책 집필 전문가",
    "prompt": "당신은 전문 작가입니다...",
    "tools": ["Read", "Write", "Edit", "WebSearch"],
    "model": "opus"
  }
}'
```

### Agent 설정 필드

| 필드 | 타입 | 설명 |
|------|------|------|
| `name` | string (필수) | 고유 식별자 (소문자, 하이픈) |
| `description` | string (필수) | Claude가 위임 판단에 사용하는 설명 |
| `tools` | array | 사용 가능한 도구 목록 |
| `disallowedTools` | array | 명시적으로 금지할 도구 |
| `model` | string | `sonnet`, `opus`, `haiku`, `inherit` |
| `permissionMode` | string | 권한 모드 |
| `maxTurns` | number | 최대 에이전트 턴 수 |
| `skills` | array | 미리 로드할 스킬 목록 |
| `memory` | string | `user`, `project`, `local` - 세션 간 지속 메모리 |
| `background` | boolean | 백그라운드 실행 여부 |
| `isolation` | string | `worktree`로 git 격리 실행 |

### Agent에 Skill 미리 로드하기

```yaml
---
name: book-writer
description: 책 집필 전문가
skills:
  - writing-style       # 글쓰기 스타일 스킬
  - chapter-structure    # 장 구조 스킬
  - research-method      # 리서치 방법 스킬
---

위 스킬들의 규칙을 따라 책을 집필하세요.
```

### Agent 지속 메모리

```yaml
---
name: book-writer
memory: project    # 프로젝트별로 학습 내용 저장
---
```

메모리 저장 위치:
- `user`: `~/.claude/agent-memory/<agent-name>/`
- `project`: `.claude/agent-memory/<agent-name>/`
- `local`: `.claude/agent-memory-local/<agent-name>/`

### 위임 패턴

#### 자동 위임

description에 "proactively" 같은 키워드를 포함하면 Claude가 자동으로 위임:

```yaml
description: "책 집필 전문가. 글쓰기 관련 요청 시 proactively 이 에이전트를 사용합니다."
```

#### 명시적 위임

```
book-writer 에이전트를 사용해서 3장을 작성해줘
```

#### 포그라운드 vs 백그라운드

- **포그라운드**: 메인 대화를 차단, 대화형 권한 요청
- **백그라운드**: 동시 실행, 권한 자동 승인

### Agent 실전 예시: 코드 리뷰어

```yaml
---
name: code-reviewer
description: 코드 리뷰 전문가. 코드 변경 후 proactively 리뷰합니다.
tools: Read, Grep, Glob, Bash
model: sonnet
---

당신은 시니어 코드 리뷰어입니다. 호출되면:

1. `git diff`로 최근 변경 확인
2. 수정된 파일에 집중
3. 즉시 리뷰 수행

## 리뷰 체크리스트
- 코드 가독성
- 보안 취약점
- 성능 이슈
- 테스트 커버리지

## 피드백 형식
- **Critical** (반드시 수정): ...
- **Warning** (수정 권장): ...
- **Suggestion** (개선 제안): ...
```

---

## 5. Skill Creator 사용법

### 개요

Skill Creator는 대화형으로 Skill을 쉽게 만들 수 있는 도구입니다.

### 사용 방법

#### 방법 1: Claude Code 내에서 직접 생성

```bash
claude
> 새로운 skill을 만들어줘. 이름은 "chapter-writer"이고,
  책의 한 챕터를 작성하는 스킬이야.
```

Claude가 자동으로 `.claude/skills/chapter-writer/SKILL.md`를 생성합니다.

#### 방법 2: /init 이후 수동 생성

```bash
# 1. 디렉토리 생성
mkdir -p .claude/skills/chapter-writer

# 2. SKILL.md 작성
cat > .claude/skills/chapter-writer/SKILL.md << 'EOF'
---
name: chapter-writer
description: 책의 한 챕터를 작성합니다
argument-hint: [챕터 제목] [주제]
allowed-tools: Read, Write, WebSearch
---

# 챕터 작성 스킬

...지시사항...
EOF
```

#### 방법 3: 기존 스킬 커뮤니티 활용

GitHub에서 공유된 스킬을 가져올 수 있습니다:
- [anthropics/skills](https://github.com/anthropics/skills)
- [awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills)

```bash
# 예: 커뮤니티 스킬 복사
cp -r community-skill/ .claude/skills/
```

### Skill 디버깅 팁

1. `/skills` 명령어로 등록된 스킬 확인
2. description이 명확한지 확인 - Claude가 이걸로 자동 호출 판단
3. `disable-model-invocation: true`로 설정 후 수동 테스트
4. 복잡한 스킬은 `context: fork`로 격리 실행

---

## 6. Agent SDK

### 개요

Agent SDK는 Claude Code의 에이전트 기능을 **프로그래밍 방식으로** 사용할 수 있게 해주는 라이브러리입니다. Python과 TypeScript를 지원합니다.

### 설치

```bash
# TypeScript
npm install @anthropic-ai/claude-agent-sdk

# Python (3.10+)
pip install claude-agent-sdk
```

### 인증

```bash
export ANTHROPIC_API_KEY=your-api-key
```

### 기본 사용법

#### Python

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="auth.py의 버그를 찾아서 수정해줘",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Bash"]
        ),
    ):
        print(message)

asyncio.run(main())
```

#### TypeScript

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "auth.py의 버그를 찾아서 수정해줘",
  options: { allowedTools: ["Read", "Edit", "Bash"] }
})) {
  console.log(message);
}
```

### SDK에서 사용 가능한 도구

| 도구 | 용도 |
|------|------|
| `Read` | 파일 읽기 |
| `Write` | 파일 생성 |
| `Edit` | 파일 수정 |
| `Bash` | 터미널 명령어 실행 |
| `Glob` | 파일 패턴 검색 |
| `Grep` | 파일 내용 검색 |
| `WebSearch` | 웹 검색 |
| `WebFetch` | 웹 페이지 가져오기 |
| `Task` | 서브에이전트 실행 |

### SDK에서 커스텀 에이전트 사용

```python
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async for message in query(
    prompt="book-writer 에이전트로 1장을 작성해줘",
    options=ClaudeAgentOptions(
        allowed_tools=["Read", "Write", "Task"],
        agents={
            "book-writer": AgentDefinition(
                description="책 집필 전문가",
                prompt="당신은 전문 작가입니다. 구조적이고 읽기 쉬운 글을 작성합니다.",
                tools=["Read", "Write", "WebSearch"],
            )
        },
    ),
):
    if hasattr(message, "result"):
        print(message.result)
```

### 세션 관리 (컨텍스트 유지)

```python
session_id = None

# 첫 번째 쿼리
async for message in query(
    prompt="인증 모듈을 읽어줘",
    options=ClaudeAgentOptions(allowed_tools=["Read", "Glob"]),
):
    if hasattr(message, "subtype") and message.subtype == "init":
        session_id = message.session_id

# 두 번째 쿼리 (컨텍스트 유지)
async for message in query(
    prompt="이제 호출하는 곳을 모두 찾아줘",
    options=ClaudeAgentOptions(resume=session_id),
):
    if hasattr(message, "result"):
        print(message.result)
```

### CLI vs SDK 비교

| 사용 사례 | 추천 |
|-----------|------|
| 대화형 개발 | CLI |
| CI/CD 파이프라인 | SDK |
| 커스텀 애플리케이션 | SDK |
| 일회성 작업 | CLI |
| 프로덕션 자동화 | SDK |

---

## 7. 프로젝트 구조 설계

### Book Agent 프로젝트 예시 구조

```
book-agent/
├── README.md                          # 이 문서
├── CLAUDE.md                          # 프로젝트 메모리/규칙
│
├── .claude/
│   ├── rules/                         # 모듈별 규칙
│   │   ├── writing-style.md           # 글쓰기 스타일 규칙
│   │   ├── chapter-format.md          # 챕터 형식 규칙
│   │   └── research-guidelines.md     # 리서치 가이드라인
│   │
│   ├── skills/                        # 스킬 정의
│   │   ├── outline-creator/
│   │   │   └── SKILL.md              # 목차/개요 생성 스킬
│   │   ├── chapter-writer/
│   │   │   ├── SKILL.md              # 챕터 작성 스킬
│   │   │   └── templates/
│   │   │       └── chapter-template.md
│   │   ├── research/
│   │   │   └── SKILL.md              # 리서치 수행 스킬
│   │   ├── editor/
│   │   │   └── SKILL.md              # 편집/교정 스킬
│   │   └── reviewer/
│   │       └── SKILL.md              # 리뷰/피드백 스킬
│   │
│   └── agents/                        # 에이전트 정의
│       ├── book-planner/
│       │   └── AGENT.md              # 책 기획 에이전트
│       ├── book-writer/
│       │   └── AGENT.md              # 책 집필 에이전트
│       └── book-editor/
│           └── AGENT.md              # 책 편집 에이전트
│
├── books/                             # 작성된 책 저장소
│   └── my-first-book/
│       ├── outline.md                 # 목차
│       ├── chapter-01.md
│       ├── chapter-02.md
│       └── ...
│
└── research/                          # 리서치 자료
    └── notes/
```

### Skill ↔ Agent 관계

```
┌─────────────────────────────────────────────────┐
│                   AGENTS                         │
│                                                  │
│  ┌──────────────┐  ┌──────────────┐             │
│  │ book-planner │  │ book-writer  │             │
│  │              │  │              │             │
│  │ Skills:      │  │ Skills:      │             │
│  │ - outline    │  │ - chapter    │             │
│  │ - research   │  │ - research   │             │
│  └──────────────┘  │ - editor     │             │
│                    └──────────────┘             │
│                                                  │
│  ┌──────────────┐                               │
│  │ book-editor  │                               │
│  │              │                               │
│  │ Skills:      │                               │
│  │ - editor     │                               │
│  │ - reviewer   │                               │
│  └──────────────┘                               │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│                   SKILLS (Rules)                 │
│                                                  │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐  │
│  │  outline   │ │  chapter   │ │  research  │  │
│  │  creator   │ │  writer    │ │            │  │
│  └────────────┘ └────────────┘ └────────────┘  │
│  ┌────────────┐ ┌────────────┐                  │
│  │  editor    │ │  reviewer  │                  │
│  └────────────┘ └────────────┘                  │
└─────────────────────────────────────────────────┘
```

> **핵심 정리**:
> - **Skill** = "어떻게 하는지"에 대한 규칙/지시사항 (재사용 가능한 레시피)
> - **Agent** = Skill을 조합하여 "무엇을 하는지" 담당하는 실행자
> - Agent는 여러 Skill을 `skills` 필드로 미리 로드하여 사용

---

## 참고 자료

### 공식 문서
- [Claude Code Skills](https://code.claude.com/docs/en/skills)
- [Claude Code Subagents](https://code.claude.com/docs/en/sub-agents)
- [CLAUDE.md & Memory](https://code.claude.com/docs/en/memory)
- [Agent SDK Overview](https://platform.claude.com/docs/en/agent-sdk/overview)

### GitHub
- [anthropics/claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)
- [anthropics/claude-agent-sdk-typescript](https://github.com/anthropics/claude-agent-sdk-typescript)
- [anthropics/skills](https://github.com/anthropics/skills)
- [awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills)
- [awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents)

### 커뮤니티
- [Subagents Directory](https://subagents.app/)
- [DataCamp - Claude Skills 튜토리얼](https://www.datacamp.com/tutorial/claude-skills)
