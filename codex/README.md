# OpenAI Codex CLI

터미널에서 동작하는 OpenAI의 오픈소스 코딩 에이전트. OpenAI 모델(o4-mini 기본, o3, o4-mini-high 등)을 기반으로 동작한다. 원래 TypeScript로 작성되었으나 현재는 Rust 구현(`codex-rs/`)으로 전환되었다.

- GitHub: https://github.com/openai/codex
- 공식 문서: https://developers.openai.com/codex
- 설치: `npm install -g @openai/codex` 또는 `brew install --cask codex`
- 실행: `codex`

## 핵심 구조

```
지속 지침 (상시 규칙)      →  AGENTS.md
온디맨드 스킬 (절차서)     →  SKILL.md
자동 기억                  →  Memories 시스템
설정                       →  ~/.codex/config.toml
```

---

## 1. AGENTS.md — 지속 지침

Codex가 작업 시작 전 자동으로 읽는 "항상 지켜야 하는 운영 규칙" 파일이다.

### 탐색 순서 (위에서 아래로 합쳐짐)

| 범위 | 경로 | 용도 |
|------|------|------|
| 사용자 전역 | `~/.codex/AGENTS.md` | 개인 전역 지침 |
| 프로젝트 루트 | `./AGENTS.md` | 팀 공유 프로젝트 규칙 |
| 현재 작업 디렉터리 | `<cwd>/AGENTS.md` | 하위 폴더/기능 특화 규칙 |

### 계층적 스코프 규칙

공식 문서에서 정의한 핵심 규칙:

> 각 AGENTS.md는 자신이 있는 디렉터리와 그 아래 모든 하위 디렉터리를 관장한다. 파일을 변경할 때는 해당 파일을 스코프로 포함하는 모든 AGENTS.md를 따라야 한다.
>
> **두 AGENTS.md가 충돌하면 디렉터리 구조에서 더 깊은 위치의 파일이 상위 파일을 오버라이드하며, 시스템/개발자/사용자가 프롬프트로 직접 준 지침이 모든 AGENTS.md보다 우선한다.**

### 비활성화

- CLI 플래그: `--no-project-doc`
- 환경변수: `CODEX_DISABLE_PROJECT_DOC=1`
- 기능 플래그: `config.toml`의 `[features]` 섹션에서 `child_agents_md`로 계층적 스코프 메시지 활성화/비활성화

### 주로 담는 내용

- 빌드/테스트/린트 명령어
- 코드 스타일 가이드라인
- 저장소 규칙 및 컨벤션
- 디렉터리별 예외 규칙
- PR 설명에 포함해야 할 문구

### 예시

```markdown
# 프로젝트 규칙

## 빌드
- `npm run build`로 빌드
- `npm test`로 테스트 실행

## 코드 스타일
- TypeScript strict mode 사용
- 함수는 화살표 함수 선호
- 들여쓰기는 2칸 스페이스

## 커밋
- Conventional Commits 형식 사용
- PR은 반드시 리뷰 후 머지
```

---

## 2. SKILL.md — 온디맨드 스킬

"필요할 때만 불러오는 절차서/플레이북"으로, AGENTS.md가 상시 규칙이라면 SKILL.md는 특정 작업을 위한 전문 지침이다.

### 디렉터리 구조

```
my-skill/
├── SKILL.md              # 메인 지침 (필수, name + description 필요)
├── agents/
│   └── openai.yaml       # UI 메타데이터 (권장)
├── scripts/              # 실행 가능한 스크립트
├── references/           # 참조 문서 (필요 시 컨텍스트에 로드)
└── assets/               # 출력용 파일 (컨텍스트에 로드되지 않음)
```

### SKILL.md 구조

두 부분으로 구성:

```yaml
---
name: my-skill
description: >
  이 스킬이 하는 일과 언제 트리거되어야 하는지 설명.
  description이 PRIMARY 트리거 메커니즘이므로
  "언제 사용하는지"를 반드시 여기에 포함해야 한다.
metadata:
  short-description: UI 표시용 짧은 설명
---

스킬 본문 (상세 지침) — 트리거 후에만 로드된다.
```

> **핵심**: `description`에 "무엇을 하는지"와 "언제 사용하는지"를 모두 포함해야 한다. 본문은 트리거 후에만 로드되므로 트리거 조건은 반드시 description에 넣어야 한다.

### 탐색 위치

| 범위 | 경로 |
|------|------|
| 사용자 전역 | `~/.codex/skills/<skill-name>/SKILL.md` |
| 프로젝트 | `.codex/skills/<skill-name>/SKILL.md` |
| 시스템 내장 | `github.com/openai/skills/tree/main/skills/.system` |
| 큐레이티드 | `github.com/openai/skills/tree/main/skills/.curated` |
| 실험적 | `github.com/openai/skills/tree/main/skills/.experimental` |

### Progressive Disclosure (3단계 로딩)

| 단계 | 로드 시점 | 크기 제한 |
|------|-----------|-----------|
| 1. 메타데이터 (name + description) | 항상 컨텍스트에 존재 | ~100단어 |
| 2. SKILL.md 본문 | 스킬 트리거 시 | <5,000단어 |
| 3. 번들 리소스 (scripts, references) | 필요 시 | 제한 없음 (스크립트는 실행만, 컨텍스트에 읽지 않음) |

### 설계 원칙

- **간결함이 핵심** — 컨텍스트 윈도우는 공유 자원. Codex가 이미 아는 것은 추가하지 않는다
- **SKILL.md는 500줄 이하** — 길어지면 reference 파일로 분리
- **README.md, CHANGELOG.md 등 보조 문서 금지** — AI 에이전트에 필요한 것만 포함
- **참조 중첩 최소화** — references는 SKILL.md에서 1단계 깊이만

### 스킬 설치

```bash
# 큐레이티드 스킬 목록 조회
scripts/list-skills.py

# GitHub에서 설치
scripts/install-skill-from-github.py --repo openai/skills --path skills/.curated/<skill-name>
```

설치 위치: `$CODEX_HOME/skills/<skill-name>` (기본: `~/.codex/skills/`). 설치 후 Codex 재시작 필요.

---

## 3. 실행 모드 (Sandbox / Approval)

| 모드 | 파일 수정 | 명령 실행 | 설명 |
|------|-----------|-----------|------|
| `suggest` (기본) | X | X | 읽기 전용, 제안만 |
| `auto-edit` | O | 승인 필요 | 파일 수정 가능, 명령은 승인 |
| `full-auto` | O | O | 모두 자동 (네트워크 비활성화 샌드박스) |

```bash
codex                                    # 대화형 REPL
codex "explain this codebase"            # 초기 프롬프트 포함
codex --approval-mode full-auto "..."    # 풀 오토 모드
codex -q "..."                           # 비대화형 조용한 모드
```

---

## 4. 설정 파일

`~/.codex/config.toml`이 기본 설정 파일이다 (레거시 TS CLI는 `.yaml`/`.json`도 지원).

### 설정 우선순위 (위가 최우선)

| 순위 | 출처 |
|------|------|
| 1 | MDM 관리 설정 (macOS only) |
| 2 | 시스템 관리 설정 (`managed_config.toml`) |
| 3 | CLI 세션 플래그 |
| 4 | 사용자 설정 (`config.toml`) |

### 주요 설정

| 파라미터 | 기본값 | 설명 |
|----------|--------|------|
| `model` | `o4-mini` | AI 모델 |
| `approvalMode` | `suggest` | 승인 모드 |
| `fullAutoErrorMode` | `ask-user` | full-auto 에러 처리 |
| `notify` | `true` | 데스크톱 알림 |

### 지원 프로바이더

OpenAI, Azure, OpenRouter, Gemini, Ollama, Mistral, DeepSeek, xAI, Groq, ArceeAI 및 모든 OpenAI 호환 API.

---

## 5. Memories 시스템

Codex도 자동 기억 시스템을 갖추고 있다. 과거 세션에서 학습한 내용을 추출하고 통합한다.

- 저장 위치: `~/.codex/` 하위에 `raw_memories.md`와 `rollout_summaries/`
- 세션 간 패턴과 인사이트를 축적

---

## 6. 협업 모드 (Collaboration Modes)

`codex-rs/core/templates/collaboration_mode/`에 정의된 템플릿 기반 모드:

| 모드 | 설명 |
|------|------|
| `default` | 기본 협업 모드 |
| `execute` | 실행 중심 |
| `pair_programming` | 페어 프로그래밍 |
| `plan` | 계획 수립 중심 |

---

## 7. MCP 서버 지원

`~/.codex/config.toml`에서 MCP 서버를 설정하여 외부 도구를 연동할 수 있다.

---

## 비교 요약

| 파일 | 역할 | 로딩 방식 | 공유 |
|------|------|-----------|------|
| `AGENTS.md` | 상시 운영 규칙 | 세션 시작 시 자동 (깊은 곳이 우선) | 저장소에 커밋 |
| `SKILL.md` | 온디맨드 절차서 | 3단계 progressive disclosure | 저장소/전역/GitHub |
| `Memories` | 자동 기억 | 세션 간 자동 축적 | 개인만 |
| `config.toml` | 설정 | CLI 시작 시 | 개인만 |
