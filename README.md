# AI Agent Study

Claude Code, Codex, Gemini CLI 등 터미널 기반 AI 코딩 에이전트의 사용법을 연구하는 저장소.

## 구조

```
.
├── claude/    # Claude Code (Anthropic)
├── codex/     # Codex CLI (OpenAI)
└── gemini/    # Gemini CLI (Google)
```

## 3개 도구 비교 요약

### 공통점

세 도구 모두 **마크다운을 에이전트 설정 계층**으로 사용하며, 역할은 세 가지로 나뉜다:

1. **지속 지침** — 항상 붙는 상시 규칙
2. **온디맨드 스킬 (SKILL.md)** — 필요할 때만 불러오는 절차서 (3개 도구 공통 표준)
3. **고급 커스터마이징** — 시스템 프롬프트, 서브에이전트 등

### 핵심 차이

| 항목 | Codex (OpenAI) | Claude Code (Anthropic) | Gemini CLI (Google) |
|------|----------------|------------------------|---------------------|
| **지속 지침 파일** | `AGENTS.md` | `CLAUDE.md` | `GEMINI.md` |
| **온디맨드 스킬** | `SKILL.md` | `SKILL.md` | `SKILL.md` |
| **규칙 모듈화** | - | `.claude/rules/*.md` (경로 스코프) | - |
| **자동 기억** | - | `MEMORY.md` (auto memory) | `/memory` 명령 |
| **서브에이전트** | - | `.claude/agents/*.md` | `.gemini/agents/*.md` (실험적) |
| **시스템 프롬프트 오버라이드** | - | - | `system.md` (full replacement) |
| **파일명 커스터마이징** | - | - | `context.fileName` 설정 |
| **커스텀 명령** | - | Skills로 흡수 | `.gemini/commands/*.toml` |
| **출력 스타일** | - | `.claude/output-styles/*.md` | - |
| **라이프사이클 훅** | - | Hooks (14개 이벤트) | - |
| **외부 도구 연동** | - | MCP 서버 | MCP 서버 |

### 한 저장소에서 여러 CLI 함께 사용하기

- 공통 팀 규칙은 `AGENTS.md`에 두는 것이 이식성이 가장 좋음
- Codex는 본래 `AGENTS.md`를 읽음
- Gemini는 `context.fileName`으로 `AGENTS.md`를 읽게 설정 가능
- Claude는 `CLAUDE.md`가 중심이므로 별도 브리지 파일을 두거나, `@AGENTS.md`로 임포트

## 참고 자료

- [Claude Code 공식 문서](https://code.claude.com/docs)
- [Codex CLI GitHub](https://github.com/openai/codex)
- [Gemini CLI GitHub](https://github.com/google-gemini/gemini-cli)
- [Agent Skills 표준](https://agentskills.io)
