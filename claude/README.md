# Claude Code

Anthropic의 공식 터미널 기반 AI 코딩 에이전트. 세 도구 중 마크다운 기반 설정 생태계가 가장 풍부하다.

- 공식 문서: https://code.claude.com/docs
- 설치: `npm install -g @anthropic-ai/claude-code`
- 실행: `claude`

## 핵심 구조

```
지속 지침 (프로젝트 기억)    →  CLAUDE.md
모듈화된 규칙              →  .claude/rules/*.md
온디맨드 스킬              →  .claude/skills/*/SKILL.md
자동 기억                  →  MEMORY.md (auto memory)
출력 스타일                →  .claude/output-styles/*.md
서브에이전트               →  .claude/agents/*.md
라이프사이클 자동화         →  Hooks
외부 도구 연동             →  MCP 서버
```

---

## 1. CLAUDE.md — 지속 지침 ("프로젝트의 헌법")

매 세션 시작 시 자동으로 읽히는 마크다운 파일. 프로젝트의 규칙, 컨벤션, 워크플로우를 정의한다. `/init` 명령으로 프로젝트 분석 후 초안을 자동 생성할 수 있다.

### 위치 계층 (아래로 갈수록 우선순위 높음)

| 범위 | 경로 | 용도 | 공유 |
|------|------|------|------|
| 조직 관리 정책 | macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`<br>Linux: `/etc/claude-code/CLAUDE.md`<br>Windows: `C:\Program Files\ClaudeCode\CLAUDE.md` | IT/DevOps가 관리하는 조직 전체 정책 | 조직 전체 |
| 사용자 전역 | `~/.claude/CLAUDE.md` | 개인 선호 (모든 프로젝트 적용) | 본인만 |
| 프로젝트 | `./CLAUDE.md` 또는 `./.claude/CLAUDE.md` | 팀 공유 프로젝트 규칙 | 팀 (VCS) |
| 로컬 개인 | `./CLAUDE.local.md` | 개인 프로젝트 설정 (.gitignore) | 본인만 |
| 하위 디렉터리 | `./subdir/CLAUDE.md` | 해당 디렉터리 작업 시 지연 로드 | 팀 (VCS) |

### 로딩 방식

- 현재 작업 디렉터리에서 **상위로 올라가며** 모든 CLAUDE.md를 발견하여 로드
- 하위 디렉터리의 CLAUDE.md는 해당 디렉터리 파일 접근 시 **지연 로드(lazy load)**
- `@path/to/file` 문법으로 다른 파일을 임포트 가능 (최대 5단계 깊이)
- `claudeMdExcludes` 설정으로 모노레포에서 불필요한 CLAUDE.md 제외 가능

### 작성 원칙

- **200줄 이하** 유지 — 길수록 중요한 규칙이 묻혀 역효과
- **구체적으로** — "코드 잘 작성" (X) → "2칸 스페이스 들여쓰기" (O)
- **검증 가능하게** — "테스트해" (X) → "`npm test` 실행 후 커밋" (O)
- **Claude가 추측할 수 없는 것만** — 언어 표준 규칙은 불필요

### 포함해야 할 것 vs 제외해야 할 것

**포함:**
- 빌드/테스트/린트 명령어
- 기본값과 다른 코드 스타일
- 브랜치 네이밍, PR 컨벤션
- 프로젝트 아키텍처 결정
- 자주 만나는 gotcha

**제외:**
- 코드에서 이미 읽을 수 있는 것
- Claude가 이미 아는 언어 표준
- 상세 API 문서 (링크로 대체)
- "클린 코드 작성" 같은 자명한 지침

### 예시

```markdown
# 코드 스타일
- ES 모듈(import/export) 사용, CommonJS(require) 금지
- TypeScript strict 모드 사용

# 워크플로우
- 타입 체크: npm run typecheck (코드 변경 후 반드시 실행)
- 단일 테스트 실행: npm test -- path/to/test
- 중요: 항상 구현 전에 테스트 먼저 작성

# Git 컨벤션
- 브랜치: feature/[작업명], fix/[이슈번호]
- 커밋: Conventional Commits (feat:, fix:, docs:, refactor:)
- main 브랜치에 강제 푸시 금지

# 주의사항
- auth 미들웨어는 passport가 req.user를 설정해야 동작
- DB 마이그레이션은 반드시 npm run migrate로 실행
```

---

## 2. .claude/rules/ — 모듈화된 규칙

CLAUDE.md가 커질 때 주제별로 분리하는 방법. 경로 스코프로 특정 파일/폴더에서만 로드되게 할 수 있다.

### 구조

```
.claude/
├── CLAUDE.md              # 메인 프로젝트 지침
└── rules/
    ├── code-style.md      # 코드 스타일
    ├── testing.md         # 테스트 컨벤션
    ├── security.md        # 보안 요구사항
    └── frontend/
        └── react.md       # React 전용 규칙
```

### 경로 스코프 (Path-specific Rules)

YAML frontmatter의 `paths` 필드로 특정 파일 패턴에서만 적용:

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API 개발 규칙
- 모든 엔드포인트에 입력 검증 포함
- 표준 에러 응답 포맷 사용
```

| 패턴 | 매칭 대상 |
|------|-----------|
| `**/*.ts` | 모든 디렉터리의 TypeScript 파일 |
| `src/**/*` | src 하위 모든 파일 |
| `src/components/*.tsx` | 특정 디렉터리의 React 컴포넌트 |
| `src/**/*.{ts,tsx}` | 중괄호 확장으로 복수 확장자 |

### 사용자 전역 규칙

`~/.claude/rules/*.md`에 두면 모든 프로젝트에 적용. 개인 선호용.

---

## 3. Skills — 온디맨드 확장 능력

필요할 때만 불러오는 절차서/플레이북. Claude가 자동으로 사용하거나 `/skill-name`으로 직접 호출한다. [Agent Skills](https://agentskills.io) 오픈 표준을 따르며, Codex/Gemini와 공통이다.

> 이전의 `.claude/commands/*.md` 커스텀 명령은 Skills로 흡수되었다. 기존 commands 파일도 계속 동작한다.

### 디렉터리 구조

```
my-skill/
├── SKILL.md           # 메인 지침 (필수)
├── template.md        # Claude가 채울 템플릿
├── examples/
│   └── sample.md      # 예상 출력 형식
└── scripts/
    └── validate.sh    # 실행 가능한 스크립트
```

### 탐색 위치 (우선순위 순)

| 우선순위 | 경로 | 적용 범위 |
|----------|------|-----------|
| 1 (최고) | 엔터프라이즈 관리 설정 | 조직 전체 |
| 2 | `~/.claude/skills/<name>/SKILL.md` | 모든 프로젝트 (개인) |
| 3 | `.claude/skills/<name>/SKILL.md` | 현재 프로젝트만 |
| 4 | 플러그인 내부 `skills/` | 플러그인 활성화 시 |

### SKILL.md Frontmatter

```yaml
---
name: my-skill              # 슬래시 명령이 되는 이름
description: 이 스킬의 설명   # Claude가 자동 선택 시 참고
disable-model-invocation: true  # true면 사용자만 호출 가능
user-invocable: false        # false면 Claude만 호출 가능
allowed-tools: Read, Grep    # 허용 도구 제한
context: fork                # 서브에이전트에서 격리 실행
agent: Explore               # context: fork 시 에이전트 타입
model: opus                  # 사용할 모델
---
```

### 호출 제어

| 설정 | 사용자 호출 | Claude 자동 호출 | 용도 |
|------|------------|-----------------|------|
| (기본) | O | O | 일반 스킬 |
| `disable-model-invocation: true` | O | X | deploy, commit 등 부작용 있는 작업 |
| `user-invocable: false` | X | O | 배경 지식 (레거시 시스템 문맥 등) |

### 스킬 예시: 코드 설명

```yaml
---
name: explain-code
description: 코드를 시각적 다이어그램과 비유로 설명. "이거 어떻게 동작해?" 질문 시 사용.
---

코드를 설명할 때 항상 포함:

1. **비유로 시작**: 일상생활의 무언가에 비유
2. **다이어그램**: ASCII 아트로 흐름/구조 표현
3. **단계별 설명**: 코드가 하는 일을 순서대로
4. **주의점**: 흔한 실수나 오해
```

### 동적 컨텍스트 주입

`!`command`` 문법으로 셸 명령 결과를 스킬 콘텐츠에 삽입:

```yaml
---
name: pr-summary
description: PR 변경사항 요약
context: fork
agent: Explore
---

## PR 컨텍스트
- PR diff: !`gh pr diff`
- 변경 파일: !`gh pr diff --name-only`

이 PR을 요약해주세요...
```

### 문자열 치환

| 변수 | 설명 |
|------|------|
| `$ARGUMENTS` | 호출 시 전달된 전체 인자 |
| `$ARGUMENTS[N]` 또는 `$N` | N번째 인자 (0-based) |
| `${CLAUDE_SESSION_ID}` | 현재 세션 ID |
| `${CLAUDE_SKILL_DIR}` | SKILL.md가 있는 디렉터리 경로 |

### 번들 스킬 (내장)

| 스킬 | 기능 |
|------|------|
| `/simplify` | 최근 변경 파일의 코드 품질/효율성 리뷰 후 수정 |
| `/batch <instruction>` | 대규모 변경을 병렬 에이전트로 분해 실행 |
| `/debug [description]` | 현재 세션 디버그 로그로 문제 진단 |
| `/loop [interval] <prompt>` | 프롬프트를 주기적으로 반복 실행 |
| `/claude-api` | Claude API 레퍼런스 자료 로드 |

---

## 4. Auto Memory — 자동 기억

Claude가 세션 간에 자동으로 축적하는 기계 로컬 기억. 빌드 명령, 디버깅 인사이트, 코드 스타일 선호 등을 자동 학습한다.

### CLAUDE.md vs Auto Memory

| | CLAUDE.md | Auto Memory |
|---|-----------|-------------|
| **작성자** | 사용자 | Claude |
| **내용** | 지침과 규칙 | 학습한 패턴과 인사이트 |
| **범위** | 프로젝트/사용자/조직 | 작업 트리별 |
| **용도** | 코딩 표준, 워크플로우 | 빌드 명령, 디버깅 팁, 선호도 |

### 저장 위치

```
~/.claude/projects/<project>/memory/
├── MEMORY.md          # 인덱스 (매 세션 첫 200줄 로드)
├── debugging.md       # 디버깅 패턴 상세
├── api-conventions.md # API 설계 결정
└── ...
```

- `MEMORY.md` 첫 200줄만 세션 시작 시 로드 (나머지는 온디맨드)
- 같은 git 저장소의 모든 워크트리가 하나의 메모리 디렉터리를 공유
- 기계 로컬이므로 다른 머신과 공유되지 않음
- `/memory` 명령으로 확인/편집/토글 가능
- "이것 기억해" → auto memory에 저장, "CLAUDE.md에 추가해" → CLAUDE.md에 저장

### 활성화/비활성화

```json
// .claude/settings.json
{ "autoMemoryEnabled": false }
```

또는 환경변수: `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`

---

## 5. Hooks — 결정론적 자동화

Claude Code 라이프사이클 이벤트에서 자동 실행되는 핸들러. CLAUDE.md 규칙은 확률적(LLM이 따를 수도 안 따를 수도)이지만, Hooks는 **결정론적으로 강제**한다.

### 이벤트 종류

| 이벤트 | 트리거 시점 | 활용 예시 |
|--------|------------|-----------|
| `PreToolUse` | 도구 실행 전 | 보호 파일 편집 차단, 위험 명령 필터링 |
| `PostToolUse` | 도구 실행 후 | 코드 자동 포맷 (Prettier), 린트 실행 |
| `Notification` | Claude가 입력 대기 | 네이티브 알림 전송 |
| `SessionStart` | 세션 시작/재개 | 컨텍스트 재주입 |
| `Stop` | Claude 응답 완료 | 자동 커밋, 품질 체크 |
| `ConfigChange` | 설정 변경 | 감사 로그 기록 |
| `UserPromptSubmit` | 사용자 프롬프트 제출 | 프롬프트 전처리 |
| `InstructionsLoaded` | 지침 파일 로드 | 디버깅용 로드 추적 |

### 3가지 Handler 유형

| 유형 | 설명 |
|------|------|
| `command` | 셸 명령 실행 (알림, 포맷, 차단) |
| `prompt` | Claude 모델에 판단 위임 (Yes/No) |
| `agent` | 서브에이전트가 파일 읽기/명령 실행 후 판단 |

### Exit Code 의미

| 코드 | 의미 |
|------|------|
| 0 | 성공 — 계속 진행 |
| 2 | **차단** — 도구 실행 중단 |
| 기타 | 오류 — 경고 표시 후 계속 |

### 예시: 보호 파일 편집 차단

```bash
#!/bin/bash
INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
PROTECTED=("package-lock.json" ".env" "docker-compose.yml")
for p in "${PROTECTED[@]}"; do
  if [[ "$FILE" == *"$p"* ]]; then
    echo "BLOCKED: $FILE is protected"
    exit 2  # exit 2 = 차단
  fi
done
exit 0
```

### 설정 방법

`/hooks` 명령으로 대화형 설정 UI를 열거나, `.claude/settings.json` / `~/.claude/settings.json`에 직접 작성.

---

## 6. 서브에이전트 (Sub-agents)

메인 컨텍스트를 보호하면서 독립 작업을 위임하는 방식.

### 내장 에이전트 타입

| 에이전트 | 모델 | 역할 | 도구 접근 |
|----------|------|------|-----------|
| Explore | Haiku (빠름) | 코드베이스 탐색 전용 | Read-only |
| Plan | 상속 | Plan Mode에서 컨텍스트 수집 | Read-only |
| general-purpose | 상속 | 복합 작업 수행 | Full access |

### 커스텀 에이전트 정의

`.claude/agents/*.md` 또는 `~/.claude/agents/*.md`에 YAML frontmatter가 있는 마크다운 파일:

```yaml
---
name: security-reviewer
description: 코드 변경 후 보안 취약점 검토
tools: Read, Grep, Glob, Bash
model: opus
---

당신은 시니어 보안 엔지니어입니다. 다음 항목을 코드 리뷰하세요:
- 인젝션 취약점 (SQL, XSS, 커맨드 인젝션)
- 인증/인가 결함
- 코드 내 시크릿 노출
```

### Frontmatter 필드

| 필드 | 설명 |
|------|------|
| `model` | haiku / sonnet / opus |
| `tools` | 허용 도구 제한 |
| `isolation: worktree` | 격리된 git 워크트리에서 실행 |
| `memory: project` | 세션 간 지식 지속 |
| `background: true` | 항상 백그라운드 실행 |

### 병렬 워크트리 개발

```bash
# 터미널 탭 3개에서 동시에 독립 기능 개발
Tab 1: claude --worktree feature-auth
Tab 2: claude --worktree feature-db
Tab 3: claude --worktree feature-ui
# 각 에이전트가 독립 브랜치에서 충돌 없이 작업
```

---

## 7. MCP 서버 — 외부 도구 연동

MCP(Model Context Protocol)로 Claude의 능력을 외부 도구/서비스로 확장한다.

### 설정 위치

- 전역: `~/.claude/settings.json`
- 프로젝트: `.claude/settings.json`

### 설정 예시

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxx"
      }
    }
  }
}
```

### 추천 MCP 서버

| 서버 | 용도 |
|------|------|
| Context7 | NPM/PyPI 패키지 최신 공식 문서 조회 |
| Playwright | 브라우저 자동화, E2E 테스트 |
| GitHub | 이슈, PR, 코드 검색 |
| Supabase | DB 스키마/쿼리 관리 |
| PostgreSQL/SQLite | DB 직접 쿼리 |
| Figma | 디자인 → 코드 변환 |
| Slack | 팀 알림 자동화 |

### 레이지 로딩 (컨텍스트 절약)

```json
{
  "env": {
    "ENABLE_TOOL_SEARCH": "true"
  }
}
```

---

## 8. 기타 마크다운 설정

### 출력 스타일

`.claude/output-styles/*.md` 또는 `~/.claude/output-styles/*.md`로 Claude의 응답 스타일 전환.

### 설정 파일

| 범위 | 경로 | 용도 |
|------|------|------|
| 프로젝트 (팀 공유) | `.claude/settings.json` | 허용/거부 도구, MCP 서버 |
| 프로젝트 (개인) | `.claude/settings.local.json` | gitignore된 개인 설정 |
| 사용자 전역 | `~/.claude/settings.json` | 전역 개인 설정 |

---

## 9. 황금 워크플로우

### Explore → Plan → Implement → Commit

```
1. Explore  — "read /src/auth and understand how sessions work"
              (Shift+Tab x2로 Plan Mode 진입)
2. Plan     — "I want to add Google OAuth. What files need to change?"
3. Implement — Shift+Tab으로 Normal Mode 복귀 후 구현
4. Commit    — "commit with a descriptive message and open a PR"
```

### 컨텍스트 관리 명령어

| 명령어 | 기능 | 사용 시점 |
|--------|------|-----------|
| `/clear` | 컨텍스트 완전 초기화 | 새 작업 전, 성능 저하 시 |
| `/compact <hint>` | 지능형 압축 | `/compact Focus on API changes` |
| `/context` | 토큰 사용량 시각화 | 현재 상태 확인 |
| `/rewind` | 체크포인트 복원 | 잘못된 방향 되돌리기 |
| `--continue` | 최근 세션 이어서 시작 | 중단된 작업 재개 |
| `--resume` | 세션 목록에서 선택 | 특정 세션 복귀 |

---

## 10. 필수 단축키

| 키 | 기능 |
|----|------|
| `Esc` | 작업 중단 (컨텍스트 유지) |
| `Esc x 2` | 리와인드 메뉴 |
| `Shift+Tab` | 모드 순환 (Normal → Auto-accept → Plan) |
| `Ctrl+G` | 플랜을 외부 에디터에서 열기 |
| `Ctrl+B` | 현재 작업 백그라운드 전환 |
| `?` | 모든 단축키 표시 |

---

## 비교 요약

| 파일 | 역할 | 로딩 방식 | 특이점 |
|------|------|-----------|--------|
| `CLAUDE.md` | 지속 지침 | 세션 시작 시 자동 | 가장 풍부한 계층 구조 |
| `.claude/rules/` | 모듈화된 규칙 | 경로 스코프 지원 | 모노레포에 최적 |
| `SKILL.md` | 온디맨드 스킬 | 필요 시 / 슬래시 명령 | 3개 도구 공통 표준 |
| `MEMORY.md` | 자동 기억 | 세션 시작 (200줄) | Claude가 자동 작성 |
| `agents/*.md` | 서브에이전트 | 위임 시 로드 | 커스텀 전문 에이전트 |
| Hooks | 라이프사이클 자동화 | 이벤트 트리거 | 결정론적 강제 |
| MCP | 외부 도구 연동 | 설정 시 활성화 | 프로토콜 기반 확장 |
