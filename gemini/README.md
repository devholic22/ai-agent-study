# Google Gemini CLI

터미널에서 동작하는 Google의 오픈소스 AI 코딩 에이전트. Gemini 모델을 기반으로 동작하며, 세 도구 중 커스터마이징 유연성이 가장 높다.

- GitHub: https://github.com/google-gemini/gemini-cli
- 설치: `npm install -g @google/gemini-cli`
- 실행: `gemini`

## 핵심 구조

```
지속 지침 (프로젝트 문맥)   →  GEMINI.md
온디맨드 스킬 (절차서)      →  SKILL.md
시스템 프롬프트 오버라이드   →  system.md (GEMINI_SYSTEM_MD)
커스텀 서브에이전트          →  .gemini/agents/*.md
커스텀 명령                  →  .gemini/commands/*.toml
```

---

## 1. GEMINI.md — 지속 지침

프로젝트와 도메인 문맥을 담는 파일로, Claude의 CLAUDE.md와 유사한 역할이다.

### 탐색 순서

| 범위 | 경로 |
|------|------|
| 사용자 전역 | `~/.gemini/GEMINI.md` |
| 프로젝트 | `./GEMINI.md` |
| 상위 디렉터리 | 디렉터리 트리를 올라가며 탐색 |
| JIT 로딩 | 파일 접근 시 해당 디렉터리와 조상 디렉터리 추가 탐색 |

- 여러 GEMINI.md 파일은 합쳐져서 매 프롬프트마다 모델에 전달된다
- `@file.md` 문법으로 다른 마크다운 파일을 임포트할 수 있다

### 메모리 명령

```
/memory show    # 현재 로드된 컨텍스트 확인
/memory reload  # GEMINI.md 다시 로드
/memory add     # 메모리에 동적으로 추가
```

### 파일명 커스터마이징 (핵심 차별점)

`settings.json`의 `context.fileName`으로 읽을 파일명 자체를 바꿀 수 있다:

```json
{
  "context": {
    "fileName": ["AGENTS.md", "CONTEXT.md", "GEMINI.md"]
  }
}
```

이를 통해 Codex의 `AGENTS.md` 생태계와 호환이 가능하다. 한 저장소에서 여러 CLI를 함께 사용할 때 가장 유연한 접근이 된다.

---

## 2. SKILL.md — 온디맨드 스킬

Codex, Claude와 동일한 SKILL.md 표준을 따른다.

### 디렉터리 구조

```
my-skill/
├── SKILL.md        # 메인 지침 (필수)
├── scripts/        # 실행 스크립트
├── references/     # 참조 문서
└── assets/         # 리소스 파일
```

### 탐색 위치 (우선순위 순)

| 우선순위 | 경로 | 설명 |
|----------|------|------|
| 1 (최고) | `.gemini/skills/` 또는 `.agents/skills/` | 워크스페이스 스킬 |
| 2 | `~/.gemini/skills/` 또는 `~/.agents/skills/` | 사용자 전역 스킬 |
| 3 | Extension 내부 `skills/` | 확장 프로그램 스킬 |

- 같은 계층에서는 `.agents/skills/`가 `.gemini/skills/`보다 우선한다
- Progressive disclosure 방식: 처음에 name/description만 읽고, 필요 시 전체 로드

---

## 3. system.md — 시스템 프롬프트 오버라이드

Gemini만의 고급 기능으로, 내장 시스템 프롬프트를 완전히 대체할 수 있다.

### 활성화

`GEMINI_SYSTEM_MD` 환경변수를 설정하면 `./.gemini/system.md`(또는 지정 파일)을 시스템 프롬프트로 사용한다.

### 핵심 특징

- **단순 추가가 아닌 Full Replacement** — 내장 시스템 프롬프트를 완전히 갈아치운다
- 템플릿 변수를 통해 현재 환경 요소를 주입할 수 있다:

| 변수 | 설명 |
|------|------|
| `${AgentSkills}` | 사용 가능한 스킬 목록 |
| `${SubAgents}` | 서브에이전트 정의 |
| `${AvailableTools}` | 현재 사용 가능한 도구 목록 |

### SYSTEM.md vs GEMINI.md 역할 구분

| 파일 | 역할 | 비유 |
|------|------|------|
| `system.md` | 안전/도구 사용 규칙 | Firmware |
| `GEMINI.md` | 프로젝트 전략/문맥 | Strategy |

---

## 4. Custom Agents — 서브에이전트

`.gemini/agents/*.md` 또는 `~/.gemini/agents/*.md`에 YAML frontmatter가 있는 마크다운 파일을 둔다.

```yaml
---
name: code-reviewer
model: gemini-2.5-pro
---

당신은 코드 리뷰 전문가입니다.
코드의 버그, 보안 취약점, 성능 이슈를 찾아주세요.
```

- 본문이 해당 에이전트의 시스템 프롬프트가 된다
- **실험적 기능**: `experimental.enableAgents`를 설정에서 켜야 한다

---

## 5. Custom Commands

Claude와 달리 마크다운이 아닌 `.gemini/commands/*.toml` 형식을 사용한다.

---

## 6. 설정 파일

| 범위 | 경로 |
|------|------|
| 사용자 전역 | `~/.gemini/settings.json` |
| 프로젝트 | `.gemini/settings.json` |

주요 설정:
- `context.fileName` — 컨텍스트 파일명 커스터마이징
- `experimental.enableAgents` — 에이전트 기능 활성화
- 모델 설정, 테마, 디스플레이 등

---

## 7. 인증

| 방식 | 설명 |
|------|------|
| Google 계정 | 무료 티어 (Gemini API) |
| API 키 | Gemini API 키 직접 사용 |
| Google Cloud | Vertex AI 연동 |

---

## 비교 요약

| 파일 | 역할 | 로딩 방식 | 특이점 |
|------|------|-----------|--------|
| `GEMINI.md` | 프로젝트 문맥 | 매 프롬프트 자동 | `@file.md` 임포트, 파일명 변경 가능 |
| `SKILL.md` | 온디맨드 전문성 | 필요 시 로드 | 3개 도구 공통 표준 |
| `system.md` | 시스템 프롬프트 | 전체 대체 | 다른 도구에 없는 고유 기능 |
| `agents/*.md` | 서브에이전트 | 실험적 | `enableAgents` 필요 |
| `commands/*.toml` | 커스텀 명령 | 명시 호출 | TOML 형식 (마크다운 아님) |
