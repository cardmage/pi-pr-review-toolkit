# 조사 노트: pi-pr-review-toolkit

- 작성일: 2026-11-01
- 목적: Claude Code `pr-review-toolkit` 플러그인을 **코드 없는 Pi 패키지**(pi-subagents 에이전트 + 스킬)로 옮기고, 나중에 npm으로 배포하기 위한 사전 조사
- 조사 환경
  - Windows, Git Bash
  - pi-coding-agent: `C:/Program Files/nodejs/node_modules/@earendil-works/pi-coding-agent`
  - pi-subagents 0.84.0: `~/.pi/agent/npm/node_modules/pi-subagents`
  - Claude Code CLI: `~/.local/bin/claude` (pr-review-toolkit 플러그인 설치되어 있음)

근거 표기:
- ✅ 파일, 코드, 실행 결과로 직접 확인
- 📄 공식 문서에 근거
- ⚠️ 추론이거나 아직 실행으로 검증하지 않음

---

## 0. 핵심 결론

1. **원본 에이전트 파일은 pi-subagents 파서로 그대로 읽힌다** ✅
   - 다만 그대로 쓰면 세 가지가 깨진다.
     - `model: opus`를 해석할 수 없다 ⚠️
     - `tools`가 없어서 쓰기 도구까지 포함된 기본 도구가 주어진다 📄
     - `inheritProjectContext` 기본값이 false라 AGENTS.md/CLAUDE.md가 전달되지 않는다 ✅
2. **원문에 Anthropic 사내 프로젝트 전제가 섞여 있다** ✅
   - 대상: silent-failure-hunter, code-simplifier
   - 다른 프로젝트에서 쓰려면 패치가 필요하다.
3. **라이선스 표기가 서로 다르다** ✅
   - upstream `LICENSE` 파일은 **Apache-2.0**인데, README에는 "MIT"로 적혀 있다.
   - 재배포는 Apache-2.0 조건을 기준으로 한다.
4. **코드 없는 Pi 패키지로 에이전트와 스킬을 함께 배포할 수 있다** ✅
   - pi-subagents는 패키지 `package.json`의 `pi.subagents.agents` 또는 `pi-subagents.agents`에서 에이전트 폴더를 읽는다(코드 확인).
5. **npm 이름 `pi-pr-review-toolkit`은 비어 있다** ✅
   - 2026-11-01 기준 `npm view` 결과 404.

---

## 1. Upstream: Claude Code pr-review-toolkit

### 1.1 소스와 버전

| 항목 | 내용 |
|---|---|
| 원본 저장소 | `anthropics/claude-plugins-official`의 `plugins/pr-review-toolkit` |
| 이 경로를 바꾼 최신 커밋 | `30202c767cfaf325212b3921ab484f83adaec850` (2026-10-26T22:42:29Z) ✅ GitHub API |
| 로컬 marketplace 체크아웃 | `~/.claude/plugins/marketplaces/claude-plugins-official/plugins/pr-review-toolkit` (LF) |
| 로컬 설치 캐시 | `~/.claude/plugins/cache/anthropic-plugin-directory/pr-review-toolkit/ea0a38e1d671-e40a2daa` (CRLF). 경로에 버전 해시가 들어 있어 업데이트하면 경로가 바뀜 |
| `installed_plugins.json` | gitCommitSha `ea0a38e1d671…`, 설치 시각 2026-10-30 |
| `.claude-plugin/plugin.json` | name `pr-review-toolkit`, version `1.0.1`(marketplace 쪽만 있음, 캐시 쪽에는 version 필드 없음), author Anthropic |

세 위치의 내용을 비교한 결과 ✅:
- **GitHub main과 marketplace 체크아웃:** 8개 파일(에이전트 6개, `commands/review-pr.md`, `LICENSE`)의 내용이 같다.
- **캐시와 marketplace:** 줄바꿈(CRLF와 LF)만 다르고 내용은 같다.

폴더 구성:
- `agents/`: 에이전트 6개
- `commands/review-pr.md`
- `README.md`, `LICENSE`, `.claude-plugin/plugin.json`

### 1.2 라이선스

- **`LICENSE` 파일:** Apache License 2.0 전문이다 ✅. `NOTICE` 파일은 없다(GitHub에서 404) ✅.
- **README:** 마지막 부분에 "License: MIT"라고 적혀 있다 ✅. 파일과 표기가 서로 다르다.
- **재배포할 때 지킬 것 (Apache-2.0 §4):**
  - 라이선스 사본을 함께 넣는다.
  - 수정한 파일에는 변경했다는 고지를 눈에 띄게 넣는다.
  - 원래의 저작권·귀속 고지를 유지한다.
  - upstream에 NOTICE가 없으므로 NOTICE를 그대로 넘겨줄 의무는 없다.
- ⚠️ 이 내용은 법률 자문이 아니다. npm에 배포하기 전에 다시 확인한다.

### 1.3 에이전트 6개

수치는 pi-subagents 파서로 읽은 결과다 ✅.

| 에이전트 | model | color | 설명 길이 | 본문 길이 | 초점 | 출력과 점수 |
|---|---|---|---|---|---|---|
| code-reviewer | `opus` | green | 972 | 2592 | 프로젝트 지침(CLAUDE.md) 준수, 버그, 품질 | 신뢰도 0–100, **80 이상만 보고**. Critical 90–100, Important 80–89 |
| code-simplifier | `opus` | (없음) | 2269 | 2899 | 동작을 유지한 채 단순화. **직접 수정함** | 수정된 코드 |
| comment-analyzer | `inherit` | green | 492 | 4358 | 주석의 정확성, 오래된 주석(comment rot) | Critical Issues / Improvement Opportunities / Recommended Removals / Positive Findings |
| pr-test-analyzer | `inherit` | cyan | 503 | 3983 | 동작 기준 커버리지, 테스트 품질 | 중요도 1–10 (Critical Gaps 8–10, Important 5–7) |
| silent-failure-hunter | `inherit` | yellow | 1428 | 6298 | 삼켜진 에러, 부적절한 fallback, 로그 누락 | CRITICAL/HIGH/MEDIUM + Hidden Errors, User Impact, 수정 예시 |
| type-design-analyzer | `inherit` | pink | 552 | 4369 | 타입 불변식, 캡슐화 | 캡슐화·불변식 표현·유용성·강제 각 1–10 |

- **도구:** 6개 모두 프론트매터에 `tools`가 없다. Claude Code에서는 이 경우 모든 도구를 물려받는다 📄.
- **설명문:** code-simplifier의 description은 `<example>` 블록이 들어 있는 YAML 블록 스칼라(`|`)다. pi-subagents 파서로도 정상적으로 읽힌다 ✅.

### 1.4 `/review-pr` 명령 (`commands/review-pr.md`) ✅

- **프론트매터:**
  - `allowed-tools: ["Bash","Glob","Grep","Read","Task"]`
  - `argument-hint: "[review-aspects]"`
- **진행 순서:**
  1. 리뷰 범위를 정한다(git status, 인자 해석).
  2. 리뷰 항목을 고른다. 선택지는 comments, tests, errors, types, code, simplify, all(기본값).
  3. 바뀐 파일을 확인한다(`git diff --name-only`, `gh pr view`).
  4. 바뀐 파일에 따라 실행할 에이전트를 정한다.
     - code-reviewer: 항상 실행
     - pr-test-analyzer: 테스트가 바뀌었을 때
     - comment-analyzer: 주석이나 문서가 바뀌었을 때
     - silent-failure-hunter: 에러 처리가 바뀌었을 때
     - type-design-analyzer: 타입이 추가되거나 바뀌었을 때
     - code-simplifier: 리뷰를 통과한 뒤
  5. 에이전트를 실행한다. **기본은 순차 실행**이고, `all parallel`을 붙이면 병렬로 실행한다.
  6. 결과를 Critical / Important / Suggestions / Positive로 묶는다.
  7. 요약을 출력한다. 형식은 `# PR Review Summary` 아래 Critical Issues, Important Issues, Suggestions, Strengths, Recommended Action이고, 지적마다 `[agent]: 설명 [file:line]`로 적는다.
- **팁:** PR을 만들기 전에 돌리고, 고친 뒤 다시 돌리고, 필요한 항목만 골라 쓰라고 권한다.

### 1.5 옮길 때 문제가 되는 upstream 내용 ✅

| 위치 | 내용 | 영향 |
|---|---|---|
| silent-failure-hunter: Logging Quality, §5 | `logError`, "error ID from constants/errorIds.ts for Sentry tracking", "Use proper error IDs for Sentry tracking" | 이런 것이 없는 프로젝트에서 "에러 ID 누락" 같은 오탐이 나온다 |
| silent-failure-hunter: Special Considerations | `logForDebugging`(사용자용), `logError`(Sentry), `logEvent`(Statsig), `constants/errorIds.ts` | Anthropic 사내 프로젝트 전용 규칙이다 |
| silent-failure-hunter: 어조 | "zero tolerance", "Call out every instance … no matter how minor" | 사소한 지적이 쏟아진다 |
| code-simplifier: §2 Apply Project Standards | ES modules(import 정렬과 확장자), `function` 키워드, 명시적 반환 타입, React Props 타입, try/catch 회피 | **직접 수정하는 에이전트**라서 원치 않는 스타일 변경이 실제로 생긴다 |
| code-simplifier: 작업 범위 | "operate autonomously and proactively", "recently modified … in the current session" | 새 컨텍스트로 시작하는 자식 세션에서는 범위가 불분명하다 |
| code-reviewer: Review Scope | "By default, review unstaged changes from `git diff`" | staged 변경과 커밋된 범위가 빠진다 |
| 공통 | 지침 파일을 "CLAUDE.md"로만 언급 | Pi에서는 AGENTS.md도 대상이다 |

에이전트 구성 자체의 한계:
- **영역이 겹친다.** 같은 문제가 여러 번 보고된다.
  - 테스트 안 된 에러 경로: pr-test-analyzer와 silent-failure-hunter
  - 에러 처리 누락, 테스트 부족: code-reviewer와 각 전문 에이전트
  - 불필요한 주석: comment-analyzer와 code-simplifier
- **판정 체계가 제각각이다.** 0–100, 1–10, CRITICAL/HIGH/MEDIUM이 섞여 있고 병합 판정이 없다.
- **빠진 관점이 있다.** 요구사항을 제대로 구현했는지 보는 에이전트가 없고, 보안·UX·성능 전문 에이전트도 없다.

---

## 2. 참고: Claude Code는 서브에이전트의 모델과 effort를 이렇게 고른다 📄

출처: https://code.claude.com/docs/en/sub-agents

- **호출할 때 모델 지정:** Agent 도구에 `model` 인자가 있다. 별칭(`sonnet`/`opus`/`haiku`/`fable`)만 받고, 정확한 ID나 `inherit`는 넘길 수 없다.
- **모델이 정해지는 순서:** 호출 인자 → 정의 파일의 `model`(`inherit`이면 메인 모델) → `CLAUDE_CODE_SUBAGENT_MODEL` → 메인 대화 모델. v2.1.251 이전에는 환경변수가 가장 먼저 적용됐다.
- **effort:**
  - 정의 파일의 `effort` 필드(low/medium/high/xhigh/max)로 정한다.
  - 호출할 때 넘기는 `effort` 인자는 v2.1.292부터 지원한다.
  - `CLAUDE_CODE_EFFORT_LEVEL` 환경변수가 가장 우선한다.
- **Explore 에이전트:**
  - 메인 모델을 쓴다. 메인이 Fable이면 연결 방식에 따라 opus를 쓴다.
  - 호출할 때 Claude가 탐색 깊이(quick/medium/very thorough)를 정해서 넘긴다.
- **플러그인 에이전트의 제한:** `hooks`, `mcpServers`, `permissionMode` 필드는 무시된다.

---

## 3. pi-subagents 0.84.0

### 3.1 에이전트를 찾는 방식

- **위치와 우선순위:** 낮은 것부터 builtin → package → user(`~/.pi/agent/agents/**`) → project(`.pi/agents/**`, 예전 방식인 `.agents/**` 포함) 📄
- **이름이 같을 때 고르는 순서:** builtin 0 < package 1 < user 2 < project 3 < runtime 4 ✅ (`src/agents/agents.js`의 `effectiveAgentMatch`)
- **패키지 에이전트 선언:** 둘 중 어느 형식이든 읽는다 ✅ (`extractSubagentPathsFromPackageRoot`)
  - `package.json`의 `"pi-subagents": { "agents": [...] }`
  - `package.json`의 `"pi": { "subagents": { "agents": [...] } }`
- **패키지를 찾는 위치** ✅ (`collectPackageSubagentPaths`)
  - 프로젝트 루트의 `package.json`
  - `<project>/.pi/npm/node_modules`, 프로젝트 settings의 `packages`
  - `~/.pi/agent/npm/node_modules`, 사용자 settings의 `packages`(로컬 경로 포함)
  - 전역 npm root
- **추가 스캔 경로:** settings의 `subagents.agentScanDirs`로 넣을 수 있다(`~` 확장, `*`는 한 단계) 📄. 패키지 안에서는 이 설정을 할 수 없다.
- **네임스페이스:** 프론트매터에 `package: x`를 넣으면 실행 이름이 `x.<name>`이 된다 📄.
- ⚠️ 등급이 같은 두 소스(예: 패키지 두 개)에 같은 이름의 에이전트가 있을 때 어느 쪽이 선택되는지는 확인하지 않았다.

### 3.2 원본 파일을 그대로 넣고 읽어 본 결과 ✅

방법: `PI_SUBAGENT_EXTRA_AGENT_DIRS`에 marketplace의 `agents` 폴더를 지정하고 `discoverAgents()`를 호출했다.

| 에이전트 | model | tools | systemPromptMode | inheritProjectContext | advertise |
|---|---|---|---|---|---|
| code-reviewer | opus | (없음) | replace | false | (없음) |
| code-simplifier | opus | (없음) | replace | false | (없음) |
| comment-analyzer | inherit | (없음) | replace | false | (없음) |
| pr-test-analyzer | inherit | (없음) | replace | false | (없음) |
| silent-failure-hunter | inherit | (없음) | replace | false | (없음) |
| type-design-analyzer | inherit | (없음) | replace | false | (없음) |

같은 방법으로, pi 프론트매터를 붙여 변환한 시험본(`/tmp/prt/out`)도 의도한 값(tools, acceptanceRole, inheritProjectContext, advertise)으로 읽히는 것을 확인했다 ✅. description에 콜론이 들어가도 정상적으로 읽힌다.

### 3.3 기본값과 해석 규칙

- **systemPromptMode:** delegate만 `append`이고 나머지는 `replace`다 ✅ (`defaultSystemPromptMode`)
- **inheritProjectContext:** delegate만 true다 ✅ (`defaultInheritProjectContext`). 그래서 일반 에이전트에는 AGENTS.md/CLAUDE.md가 전달되지 않는다.
- **inheritGlobalContext:** 기본 false다. 이 값은 inheritProjectContext가 true일 때만 의미가 있다 📄.
- **inheritSkills:** 기본 false다 ✅.
- **tools를 생략하면:** Pi 기본 도구가 주어진다. 백그라운드로 실행되면 주변 확장의 도구까지 포함된다 📄.
- **model 해석 순서** ✅ (`src/runs/shared/model-resolution.js`)
  1. `provider/id` 전체가 정확히 일치
  2. id만 정확히 일치
  3. 비슷한 이름으로 일치(구분자, 대소문자, 날짜 표기 차이만 허용)
  - `opus` 같은 별칭은 레지스트리에 있는 id가 아니므로 해석되지 않는다. 실행하면 오류가 날 것으로 예상하지만 실제로 돌려 보지는 않았다 ⚠️.
- **`inherit`:** 상위 세션의 모델을 쓴다 📄.
- **프론트매터 주석:** `#`으로 시작하는 YAML 주석 줄은 무시된다 ✅ (`src/agents/frontmatter.js`).
- **목록 필드:** 쉼표로 구분하거나 `- item` 블록 형식으로 쓸 수 있다 📄.

### 3.4 `agentOverrides` (settings.json)

- **지원 필드** 📄: `description`, `advertise`, `machine`, `output`, `outputMode`, `defaultReads`, `model`, `defaultProvider`, `thinking`, `systemPromptMode`, `inheritProjectContext`, `inheritGlobalContext`, `inheritSkills`, `defaultContext`, `acceptanceRole`, `disabled`, `skills`, `tools`, `systemPrompt`
- **적용 대상:** 패키지, 사용자, 프로젝트 에이전트 모두에 적용된다. 같은 필드가 있으면 프론트매터 값을 대체한다 📄.
- **`tools` 형식:** 문자열 배열, `"inherit"`, `false`만 받는다 ✅ (`parseToolsOverride`의 오류 메시지). 쉼표로 이은 문자열은 받지 않는다.
- **모델 우선순위** 📄: 호출할 때 넘긴 값 → 제공사별 override → `agentOverrides.<name>.model` → 프론트매터 `model` → `subagents.defaultModel` → 세션 모델
- **관련 설정:**
  - `subagents.defaultThinking`: 기본 생각 깊이
  - `subagents.maxThinking`: 생각 깊이 상한
  - `subagents.modelScope`: 허용 모델 목록
  - 프로필 명령(`/subagents-generate-profiles` 등)
  - 호출할 때 `model: "provider/id:level"` 형식으로 모델과 생각 깊이를 함께 지정할 수 있다.

### 3.5 위임 정책

- **실행 조건:** 함께 제공되는 pi-subagents 스킬에 따르면, 사용자가 현재 요청에서 시켰거나 적용되는 user/project 지침이 허락했을 때만 서브에이전트를 띄운다 📄. 따라서 스킬 본문에 **"이 스킬을 쓰는 것은 서브에이전트 실행을 허락한다는 뜻"이라는 문장이 필요하다**.
- **advertise:** 기본값 false다. false이면 에이전트 이름과 설명이 상위 에이전트의 시스템 프롬프트에 들어가지 않는다 📄.

### 3.6 비교 대상: 기본 역할과 번들 프롬프트

- **기본 역할 (Pi 안에서 실행):** scout(low), researcher(medium), evidence-auditor(high), worker(high, 쓰기), reviewer(high), oracle(high, 상위 대화 이어받음), delegate(세션 설정 따름) ✅
- **외부 CLI 역할:** `claude-code(-writer)`, `codex-exec(-writer)`, `cursor-agent(-writer)`. 이 PC에는 `claude`만 설치되어 있다 ✅.
- **`reviewer`:**
  - 도구: `read, grep, find, ls, watchdog_diff, contact_supervisor` ✅. **bash가 없다.**
  - `watchdog_diff`는 세션 시작 커밋 이후의 diff를 보여 준다. 그 뒤의 커밋과 추적되지 않는 파일도 포함한다 📄.
- **`/parallel-review`:**
  - 코드 처리가 없는 일반 프롬프트 템플릿이다 ✅. pi-subagents의 `package.json`에 `pi.prompts: ["./prompts"]`로 등록되어 있다.
  - 에이전트 이름을 지정하지 않는다. 함께 제공되는 스킬(`references/prompting-and-roles.md`)의 안내에 따라 상위 에이전트가 `reviewer`를 고른다.
  - 관점은 그때그때 3개를 정한다. 큰 변경이면 4번째를 추가한다.
  - 판정: P0/P1/P2와 `Merge verdict`를 쓰고, 근거가 있는 지적만 보고한다.
  - 리뷰 뒤 처리: `autofix`, 후속 확인 리뷰, 봇 코멘트 분류(VALID/STALE/INVALID/OUT-OF-POLICY).
- **`/parallel-cleanup`:** reviewer 2개(군더더기 제거, 장황함 제거)를 `output: false`로 띄운다. `autofix`를 지원한다.
- **`/review-loop`:** worker와 reviewer를 번갈아 돌린다. 최대 3번이다.

### 3.7 확장 API (코드 없는 패키지에서는 쓰지 않음, 참고용) 📄

- 런타임 에이전트 등록 이벤트 `pi-subagents:runtime-agent-register:v1`. 정의 필드로 문서에 나온 것은 description, systemPrompt, tools다.
- `registerWorkflowResource` (`pi-subagents/workflow-resources`). `resolve()`는 동기 함수여야 하고 입출력 작업을 할 수 없다.
- 이벤트 버스 RPC(`subagents:rpc:v1:*`)

---

## 4. Pi 코어 (pi-coding-agent)

### 4.1 패키지 📄 (`docs/packages.md`)

- **설치:** `pi install npm:<pkg>@<ver>`, `pi install git:<repo>@<ref>`, `pi install ./local`
  - 로컬 경로 패키지는 복사하지 않고 그 자리에서 읽는다.
  - 한 번만 시험하려면 `pi -e <source>`를 쓴다.
- **매니페스트:** `package.json`의 `pi.extensions/skills/prompts/themes`에 선언하거나, 관례 폴더(`extensions/`, `skills/`, `prompts/`, `themes/`)를 쓴다.
- **`pi-package` 키워드:** 넣으면 Pi 패키지 갤러리에 등록될 수 있다. 미리보기용 `pi.image`, `pi.video` 필드를 선택적으로 넣을 수 있다.
- **의존성:**
  - Pi가 제공하는 호스트 패키지는 `peerDependencies`에 `"*"`로 넣는다.
  - 다른 Pi 패키지는 서로를 Node 의존성으로 해석하지 못한다. 그래서 **pi-subagents는 "미리 설치되어 있어야 함"으로 문서화한다.**
- **리소스 골라 쓰기:** settings의 `packages`에 객체 형식으로 적으면 특정 리소스만 켜거나 끌 수 있다.
- **패키지 식별:** npm은 이름, git은 URL, 로컬은 절대경로로 구분한다.
- ✅ pi-coding-agent의 `schemas/`에는 패키지 매니페스트 스키마가 없다(settings, models, theme, keybindings뿐). ⚠️ 그래서 `pi` 키 안의 `subagents` 키를 Pi 코어가 경고 없이 무시하는지는 실제로 설치해서 확인해야 한다.

### 4.2 스킬 📄 (`docs/skills.md`)

- **찾는 위치:** `~/.pi/agent/skills`, `~/.agents/skills`, `.pi/skills`, `.agents/skills`(상위 폴더로 올라가며 탐색), 패키지
- **로드 방식:** 시작할 때는 이름, 설명, 경로만 시스템 프롬프트에 넣는다. 작업에 맞으면 모델이 `SKILL.md`를 읽는다. 모델이 안 읽을 수도 있으니 `/skill:name`으로 강제할 수 있다.
- **`/skill:name args`:** 뒤의 글은 사용자 요청으로 덧붙는다. `$ARGUMENTS`처럼 치환되는 방식이 아니다.
- **`disable-model-invocation: true`:** 명시적으로 호출할 때만 쓰이게 한다.
- **이름 규칙:** 소문자, 숫자, 하이픈. 64자 이하. description은 1024자 이하.
- **이름 충돌:** 처음 발견된 것을 쓰고 경고를 낸다.
- **상대 경로:** 스킬 폴더 기준으로 참조할 수 있다.

### 4.3 프롬프트 템플릿 📄 + ✅

- **기본:** 파일 이름이 명령 이름이 된다. 프론트매터에 `description`, `argument-hint`를 쓴다.
- **치환:** `$1`, `$@`/`$ARGUMENTS`, `${1:-기본값}`, `${@:N}`, `${@:N:L}`
- **이름이 같을 때 우선순위:** **프로젝트 > 사용자 > 패키지** ✅
  - 근거: `dist/core/package-manager.js`의 `resourcePrecedenceRank`
  - 근거: `dist/core/resource-loader.js`의 `dedupePrompts`. 처음 것을 쓰고 collision 진단을 남긴다.

### 4.4 컨텍스트 파일 📄 (`docs/configuration.md`)

- **읽는 파일:** `AGENTS.override.md`, `AGENTS.md`, `CLAUDE.md`(대문자 `.MD`도 포함)
- **끄는 방법:** `--no-context-files` 플래그(`docs/cli.md`)

---

## 5. 대안 비교 요약

| | `/parallel-review` (기본 제공) | pr-review-toolkit 이식 |
|---|---|---|
| 강점 | 요청 의도에 맞는지 검사함. 변경 종류에 맞춰 관점을 정함. P0/P1/P2와 Merge verdict로 통일. 근거 필터. autofix, 후속 리뷰, 봇 코멘트 분류. bash가 없는 진짜 읽기 전용. 비용 낮음 | 영역별 체크리스트가 깊음. 매번 같은 기준. 점수가 남음. 에이전트별 모델 지정. 어떤 diff 범위든 직접 조회. 이름으로 하나만 호출 가능. 수정 단계(simplifier) 포함 |
| 약점 | 결과가 상위 에이전트 실력에 좌우됨. 깊이가 얕음. 이전 세션의 커밋은 직접 못 봄 | 사내 규칙이 섞여 있음. 요구사항 충족 여부를 안 봄. 잡음과 중복. 판정 체계가 제각각. bash 읽기 전용이 지시로만 지켜짐. 비용 높음 |

통합할 만한 아이디어(나중 검토):
- parallel-review의 근거 필터, P0/P1/P2, Merge verdict, autofix를 toolkit 흐름에 넣는다.
- 요구사항 충족 여부를 보는 `reviewer` 1개를 추가한다.

---

## 6. 아직 검증하지 않은 것 ⚠️

1. 스킬을 통해 서브에이전트 실행이 실제로 허락되고 동작하는지 (위임 정책과의 상호작용)
2. 패키지 에이전트에 `agentOverrides`가 실제로 적용되는지 (문서상으로는 적용됨)
3. 원본의 `model: opus`가 실제로 실행에 실패하는지 (코드를 읽고 내린 추론)
4. Pi 코어가 `pi.subagents` 매니페스트 키를 경고 없이 무시하는지
5. 같은 이름의 에이전트를 가진 패키지가 두 개 있을 때 어느 쪽이 선택되는지
6. bash가 있는 리뷰어가 실제로 파일을 바꾸지 않는지 (지시문만으로 지켜지는지)

---

## 7. 출처

- upstream: https://github.com/anthropics/claude-plugins-official/tree/main/plugins/pr-review-toolkit
  - `agents/*.md`, `commands/review-pr.md`, `README.md`, `LICENSE`, `.claude-plugin/plugin.json`
- Claude Code 서브에이전트 문서: https://code.claude.com/docs/en/sub-agents
- pi-subagents 0.84.0 (로컬 설치본): `docs/{agents,models,workflows,configuration,extension-api}.md`, `agents/*.md`, `prompts/*.md`, `skills/pi-subagents/**`, `src/agents/{agents,frontmatter}.js`, `src/runs/shared/model-resolution.js`
  - 저장소: https://github.com/nicobailon/pi-subagents
- pi-coding-agent (로컬 설치본): `docs/{packages,skills,prompt-templates,extensions,configuration,cli}.md`, `dist/core/{package-manager,resource-loader}.js`
