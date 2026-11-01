# 구현 계획: pi-pr-review-toolkit v0.1

- 작성일: 2026-11-01
- 근거 자료: [research.md](./research.md)
- 형태: **코드 없는 Pi 패키지**. 들어가는 것은 에이전트 정의, 스킬, 문서뿐이고 실행되는 코드는 없다.
- 최종 목표: `npm publish` 후 `pi install npm:pi-pr-review-toolkit`로 설치할 수 있게 한다.

---

## 1. 목표

1. Claude Code pr-review-toolkit의 에이전트 6개와 `/review-pr` 작업 흐름을 Pi(pi-subagents)에서 Claude Code와 거의 같은 사용감으로 쓰게 한다.
2. 원문은 최대한 그대로 두고, Pi에서 동작하는 데 필요한 부분만 고친다. 사내 규칙 제거, 도구, 컨텍스트 설정이 그 대상이다. 이렇게 해야 upstream이 바뀌었을 때 따라가기 쉽다.
3. 로컬 설치로 검증한 다음 npm에 배포한다.

## 2. v0.1에서 하지 않는 것

- TypeScript 확장: 런타임 에이전트 등록, 워크플로 리소스, 슬래시 명령 코드
- upstream에 없는 새 에이전트: 요구사항 충족 확인, 보안 등
- `/parallel-review`의 판정 체계(P0/P1/P2, Merge verdict)를 합치는 것. v0.2 후보로 둔다.
- 패키지 안에서 모델을 고정하는 것. 모델은 사용자 설정에 맡긴다.
- 동기화 스크립트. 코드 없는 패키지를 유지하기 위해 수동 절차로 대신한다.

---

## 3. 결정 사항

| ID | 결정 | 이유 |
|---|---|---|
| D1 | 코드 없는 Pi 패키지로 만든다 (`agents/` + `skills/` + 문서) | 배포와 공유는 `pi install` 한 번으로 해결된다. 런타임 코드가 없어서 깨질 곳이 적다 |
| D2 | 에이전트는 upstream 본문을 복사하고, pi용 프론트매터를 붙이고, 최소한만 패치한다. 기준 커밋은 `30202c7`로 기록한다 | 패키지는 그 자체로 완결되어야 한다. 사용자 PC의 `~/.claude` 경로에 기대면 안 된다 |
| D3 | 매니페스트 키는 `pi.subagents.agents`를 쓴다 | pi-subagents가 읽는 두 형식 중 하나다. M3에서 Pi 코어가 경고를 내면 최상위 `pi-subagents` 키로 바꾼다 |
| D4 | 에이전트 이름은 upstream과 같게 둔다 (`code-reviewer` 등) | Claude Code와 같은 이름을 쓸 수 있다. 사용자나 프로젝트에 같은 이름의 에이전트가 있으면 그쪽이 이기므로, 사용자가 덮어쓸 수 있는 통로가 된다. 네임스페이스는 O1에서 정한다 |
| D5 | 도구: 리뷰어 5개는 `read, grep, find, ls, bash`, code-simplifier는 여기에 `edit, write`를 더한다 | `git diff base...HEAD`와 `gh pr diff`가 필요하다. bash 읽기 전용은 지시문으로 지키게 하고, M4에서 실제로 지켜지는지 확인한다 |
| D6 | `inheritProjectContext: true`, `inheritSkills: false`, `systemPromptMode: replace` | code-reviewer의 핵심 기능이 프로젝트 지침 준수 검사라서 AGENTS.md/CLAUDE.md가 필요하다 |
| D7 | 프론트매터에 `model`은 넣지 않고, `thinking: high`만 넣는다 | 모델은 사용자 설정(`agentOverrides`)에 맡긴다. README에 upstream과 비슷한 권장 설정(code-reviewer와 simplifier는 opus급)을 싣는다 |
| D8 | 진입점은 스킬 `pr-review` 하나다 | 자연어로도, `/skill:pr-review`로도 부를 수 있다. 프롬프트 템플릿은 O3에서 정한다 |
| D9 | upstream `commands/review-pr.md`를 스킬의 `references/` 폴더에 넣고, 스킬 본문에는 Pi에 맞게 바꾸는 규칙만 적는다. 기본은 순차 실행, `parallel`을 붙이면 병렬 실행 | upstream과 같은 동작이고, 동기화할 때 바꿀 부분이 줄어든다 |
| D10 | 패키지 라이선스는 Apache-2.0으로 한다. upstream LICENSE를 넣고, `NOTICE`에 출처·커밋·변경 요약을 적고, 수정한 파일마다 변경 고지를 넣는다 | Apache-2.0 §4를 지키기 위해서다. 파서가 YAML 주석을 무시하는 것은 확인했다(research §3.3) |
| D11 | `advertise`는 false(기본값)로 둔다 | 스킬이 에이전트 이름을 알려 주므로 상위 프롬프트에 설명을 넣을 필요가 없다. 바꿀지는 O4에서 정한다 |
| D12 | upstream 동기화는 수동 절차로 하고 `docs/SYNC.md`에 적는다 | 코드 없는 패키지를 유지하기 위해서다 |

---

## 4. 패키지 구조 (목표)

```
pi-pr-review-toolkit/
├── package.json
├── README.md
├── LICENSE                  # Apache-2.0 (upstream과 같은 전문)
├── NOTICE                   # 출처, 기준 커밋, 변경 요약
├── CHANGELOG.md
├── agents/
│   ├── code-reviewer.md
│   ├── code-simplifier.md
│   ├── comment-analyzer.md
│   ├── pr-test-analyzer.md
│   ├── silent-failure-hunter.md
│   └── type-design-analyzer.md
├── skills/
│   └── pr-review/
│       ├── SKILL.md
│       └── references/
│           └── review-pr.md # upstream 명령 사본 + 변경 고지
└── docs/                    # npm 배포에서는 뺌
    ├── SYNC.md
    └── poc/
        ├── research.md
        └── plan.md
```

## 5. package.json 초안

```json
{
  "name": "pi-pr-review-toolkit",
  "version": "0.1.0",
  "description": "Claude Code pr-review-toolkit review agents for Pi (pi-subagents): code-reviewer, silent-failure-hunter, pr-test-analyzer, comment-analyzer, type-design-analyzer, code-simplifier",
  "keywords": ["pi-package", "pi", "pi-subagents", "code-review", "pull-request", "claude-code"],
  "license": "Apache-2.0",
  "repository": { "type": "git", "url": "TBD" },
  "files": ["agents/", "skills/", "LICENSE", "NOTICE", "README.md", "CHANGELOG.md"],
  "pi": {
    "skills": ["./skills"],
    "subagents": { "agents": ["./agents"] }
  }
}
```
- 의존성은 없다. pi-subagents(0.84.0 이상)는 README에 "미리 설치되어 있어야 함"으로 적는다. 다른 Pi 패키지는 Node 의존성으로 해석되지 않기 때문이다(research §4.1).

---

## 6. 에이전트 변환 규칙

### 6.1 프론트매터 바꾸기

| upstream | pi 패키지 |
|---|---|
| `name` | 그대로 |
| `description` (길고 예시가 들어 있음) | 한 줄짜리 영어 설명(200자 이하)으로 바꾼다. advertise가 false라 어차피 노출되지 않는다 |
| `model: opus` / `inherit` | 지운다 (D7) |
| `color` | 지운다 (pi에서 쓰지 않음) |
| (없음) | `tools`, `thinking: high`, `systemPromptMode: replace`, `inheritProjectContext: true`, `inheritSkills: false`, `acceptanceRole: read-only` 또는 `writer`를 넣는다 |
| (없음) | 파일 맨 위에 YAML 주석으로 변경 고지를 넣는다. 예: `# Derived from anthropics/claude-plugins-official plugins/pr-review-toolkit/agents/<name>.md @30202c7 (Apache-2.0). Modified for Pi; see NOTICE.` |

프론트매터 예시 (리뷰어):
```yaml
---
# Derived from anthropics/claude-plugins-official plugins/pr-review-toolkit/agents/code-reviewer.md @30202c7 (Apache-2.0).
# Modified for Pi (pi-subagents): frontmatter replaced, Pi runtime notes appended. See NOTICE.
name: code-reviewer
description: PR review - AGENTS.md/CLAUDE.md compliance, bugs, and quality; reports only confidence >= 80
tools: read, grep, find, ls, bash
thinking: high
systemPromptMode: replace
inheritProjectContext: true
inheritSkills: false
acceptanceRole: read-only
---
```

### 6.2 본문 패치 목록

원문 문장은 최대한 살리고, 문장 단위로만 바꾼다.

| ID | 대상 | 바꾸는 내용 |
|---|---|---|
| P1 | 6개 공통 | 본문 끝에 `## Pi runtime notes`를 붙인다. 내용: 지침 파일은 AGENTS.md/CLAUDE.md다 / 리뷰 범위는 task에서 받고, 없으면 `git diff`와 `git diff --cached`를 본다 / 모든 지적에 file:line과 근거를 붙인다 / 리뷰어는 읽기 전용이다(bash는 조회만, 수정·커밋·푸시·PR 코멘트 금지) |
| P2 | silent-failure-hunter | `logError`, `logForDebugging`, `logEvent`, `constants/errorIds.ts`, Sentry, Statsig 언급을 "프로젝트의 로깅·에러 추적 규칙(AGENTS.md/CLAUDE.md에 있다면)"으로 바꾼다. `Special Considerations` 절은 특정 프로젝트에 묶이지 않는 문구로 바꾼다 |
| P3 | silent-failure-hunter | "Call out every instance … no matter how minor"를 "작업 범위 안에서 구체적인 근거가 있는 경우를 모두 지적하고, 추측성 지적은 하지 않는다"로 바꾼다 |
| P4 | code-simplifier | §2의 하드코딩된 스타일 목록을 "AGENTS.md/CLAUDE.md의 코딩 표준을 따르고, 없으면 주변 코드에 맞춘다"로 바꾼다. "autonomously and proactively"와 "current session"은 "task에서 지정한 범위만"으로 바꾼다 |
| P5 | code-reviewer | "By default, review unstaged changes from `git diff`"를 "unstaged와 staged 변경 모두(`git diff`, `git diff --cached`)"로 바꾼다 |
| P6 | `references/review-pr.md` | 원문은 그대로 두고, 맨 위에 변경 고지와 "Pi에서 읽을 때 SKILL.md의 적응 규칙이 우선한다"는 문구만 넣는다 |

`NOTICE`에는 P1부터 P6까지의 요약을 적는다.

---

## 7. 스킬 설계: `skills/pr-review/SKILL.md`

- **프론트매터**
  - `name: pr-review`
  - `description`(1024자 이하): "Comprehensive PR/change review with pr-review-toolkit agents (…). Use when asked to review a PR or recent changes, or to check test coverage, error handling, comments, or type design before commit/PR."
- **본문 구성** (모델이 읽으므로 영어로 쓴다)
  1. **허가 문장:** "Invoking this skill is the operator's authorization to launch the subagents below." 위임 정책 때문에 반드시 필요하다(research §3.5).
  2. **작업 흐름:** `references/review-pr.md`를 읽고 따르게 한다. 그 파일의 `$ARGUMENTS`는 사용자 요청 글로 해석하게 한다.
  3. **Pi에 맞게 바꾸는 규칙**
     - Task 도구 대신 `subagent`를 쓴다.
       - 순차: `subagent({ agent, task })`를 하나씩 호출한다.
       - 병렬: `subagent({ workflow: "parallel", args: { tasks } })`로 한 번에 호출한다.
     - 자식은 새 컨텍스트로 시작하므로, 지시문에 다음을 반드시 넣는다: cwd, 정확한 diff 명령, 바뀐 파일 목록, 리뷰 항목, file:line을 요구하는 문장.
     - 범위 정하기
       - `gh`가 없거나 PR이 없으면 `git diff`와 `git diff --cached`를 쓴다.
       - base를 지정하면 `git diff <base>...HEAD`를 쓴다.
     - code-simplifier는 쓰기 담당이므로 단독으로, 리뷰가 끝난 뒤, 요청했을 때만, 바뀐 파일 범위 안에서만 실행한다.
  4. **결과 종합:** upstream의 요약 형식을 따른다. 의심스러운 지적은 상위 에이전트가 직접 확인한다. 수정은 적용하기 전에 사용자에게 묻는다.
- **명시적으로만 부르게 할지:** 기본은 자연어 자동 로드를 허용한다. `disable-model-invocation`은 O3에서 함께 정한다.

---

## 8. README에 넣을 사용자 안내 (초안 항목)

1. **사전 요구사항:** pi-subagents 0.84.0 이상 (`pi install npm:pi-subagents`)
2. **설치:** `pi install npm:pi-pr-review-toolkit`, 개발 중에는 `pi install ./pi-pr-review-toolkit`
3. **권장 모델 설정** (`~/.pi/agent/settings.json`)
   ```json
   "subagents": {
     "agentOverrides": {
       "code-reviewer":         { "model": "<strong-model>" },
       "code-simplifier":       { "model": "<strong-model>" },
       "silent-failure-hunter": { "model": "<standard-model>" },
       "pr-test-analyzer":      { "model": "<standard-model>" },
       "comment-analyzer":      { "model": "<standard-model>" },
       "type-design-analyzer":  { "model": "<standard-model>" }
     }
   }
   ```
4. **사용법:** "PR 리뷰해줘", `/skill:pr-review`, `/skill:pr-review tests errors parallel`, `/skill:pr-review simplify`
5. **Claude Code와 다른 점:** 모델 별칭을 쓸 수 없음, 자동 호출 정책, 패치 목록(P1–P6)
6. **라이선스와 귀속:** Apache-2.0, upstream 출처와 커밋

---

## 9. 작업 단계

### M0. 뼈대 만들기
- [x] 저장소 위치: `C:/Users/hc2/orca/projects/pi-pr-review-toolkit`. git 저장소이고 `master` 브랜치에 빈 Initial commit이 있다. Orca worktree `workspaces/pi-pr-review-toolkit/dolphin`(브랜치 `dpdlf/dolphin`)도 있다. 원격 저장소는 아직 없다.
- [ ] `package.json`(§5), `LICENSE`(upstream 전문), `NOTICE`, `README.md`와 `CHANGELOG.md`의 빈 틀을 만든다.
- **완료 기준:** 파일 구조가 §4와 같다. `npm pack --dry-run` 결과에 `docs/`가 들어 있지 않다.

### M1. 에이전트 6개 변환
- [ ] upstream `30202c7` 기준으로 본문을 복사하고 §6.1 프론트매터를 붙인다.
- [ ] P1–P5 패치를 적용한다.
- [ ] 원본과의 diff를 검토한다. 의도한 변경만 들어갔는지 확인한다.
- **검증:** pi-subagents 로더(`discoverAgents`)로 6개 모두 읽히는지, 필드(tools, inheritProjectContext, acceptanceRole, thinking)가 의도한 값인지 확인한다. 검증용 일회성 스크립트는 패키지에 넣지 않는다.
- **완료 기준:** 6개 모두 파싱된다. 사내 규칙 키워드(`errorIds`, `Sentry`, `Statsig`, `logForDebugging`, `function keyword` 목록)가 본문에 남아 있지 않다. 남아 있다면 의도적으로 남긴 것이어야 한다.

### M2. 스킬
- [ ] `SKILL.md`(§7)를 쓴다.
- [ ] `references/review-pr.md`에 upstream 사본을 넣고 P6을 적용한다.
- **검증:** `/skill:pr-review`가 로드되는지, `references/` 상대 경로가 제대로 해석되는지 확인한다.
- **완료 기준:** 스킬 진단에 경고가 없다. description이 1024자 이하다.

### M3. 로컬 설치 통합
- [ ] `pi install ./pi-pr-review-toolkit`를 실행한다(사용자 settings `packages`에 추가됨).
- [ ] `/subagents` 목록이나 `subagent({ action: "list" })`에 6개가 source `package`로 보이는지 확인한다.
- [ ] Pi 시작 진단에 매니페스트(`pi.subagents`) 관련 경고가 없는지 확인한다. 경고가 있으면 D3의 대안을 적용한다.
- [ ] `agentOverrides`를 설정한 뒤 `/subagents-models`로 패키지 에이전트에 반영되는지 확인한다.
- **완료 기준:** 위 네 가지가 모두 통과한다.

### M4. 스모크 테스트
- [ ] 임시 git 저장소에 문제를 심어 둔다.
  - 빈 catch, 근거 없이 기본값을 돌려주는 fallback
  - 새 분기인데 테스트가 없는 코드
  - 동작과 맞지 않는 주석
  - 외부에서 상태를 바꿀 수 있는 타입
  - AGENTS.md 규칙 위반 하나
- [ ] 다음 경우를 실행해 본다: `/skill:pr-review`(전체, 순차), `… parallel`, `… tests errors`, `… simplify`, 자연어 "PR 리뷰해줘"
- **확인할 것:**
  - 각 에이전트가 심어 둔 문제를 찾는가
  - 리뷰어 실행 뒤 `git status`가 깨끗한가 (읽기 전용이 지켜지는가)
  - simplifier가 지정한 범위 밖 파일을 건드리지 않는가
  - 모델 해석 오류가 없는가
  - 위임이 거부되지 않는가
  - code-reviewer가 AGENTS.md 위반을 찾는가 (프로젝트 컨텍스트가 전달되는가)
- **완료 기준:** 모두 통과한다. 실패하면 원인과 대응을 `docs/test-notes.md`에 적는다.

### M5. 문서
- [ ] README(§8), NOTICE(P1–P6 요약), CHANGELOG(0.1.0)를 쓴다.
- [ ] `docs/SYNC.md`에 upstream 동기화 절차를 적는다.
  1. upstream 최신 커밋을 확인한다.
  2. `git diff <old>..<new> -- plugins/pr-review-toolkit`으로 바뀐 점을 본다.
  3. 바뀐 문단을 반영하고 P1–P6을 다시 적용한다.
  4. 기준 커밋 표기(파일 주석, NOTICE)를 갱신한다.
  5. M1과 M3의 검증을 다시 돌린다.
- **완료 기준:** README만 보고 처음 쓰는 사람이 설치부터 첫 리뷰까지 할 수 있다.

### M6. npm 배포
- [ ] 원격 저장소를 만들어 연결하고(O6), `repository` 필드를 채운다.
- [ ] 이름이 여전히 비어 있는지 `npm view pi-pr-review-toolkit`로 다시 확인한다. 스코프를 쓸지는 O6에서 정한다.
- [ ] `npm pack --dry-run`으로 들어가는 파일 목록을 확인한다.
- [ ] `npm login`, `npm publish --access public`, git 태그 `v0.1.0`
- [ ] 새 환경에서 설치를 확인한다: `pi install npm:pi-pr-review-toolkit`를 실행하고 M3의 확인 항목을 다시 본다.
- **완료 기준:** npm에서 설치해도 M3와 같은 결과가 나온다.

---

## 10. 검증 체크리스트 (요약)

- [ ] 에이전트 6개 파싱, 필드 값 확인 (M1)
- [ ] 사내 규칙 키워드 제거 (M1)
- [ ] 스킬 로드, references 경로 (M2)
- [ ] 패키지 에이전트 발견, 매니페스트 경고 없음, 덮어쓰기 반영 (M3)
- [ ] 심어 둔 문제 탐지, 읽기 전용 유지, simplifier 범위 준수, 위임 동작 (M4)
- [ ] 라이선스 고지: LICENSE, NOTICE, 파일별 주석 (M5)
- [ ] `npm pack` 내용, npm 설치 검증 (M6)

---

## 11. 리스크와 대응

| ID | 리스크 | 대응 |
|---|---|---|
| R1 | 스킬이 자연어 요청에 자동으로 로드되지 않을 수 있다 | `/skill:pr-review`를 안내하고, description을 다듬는다 |
| R2 | 위임 정책 때문에 상위 에이전트가 서브에이전트를 띄우지 않을 수 있다 | 스킬에 허가 문장을 넣고, M4에서 확인한다. 안 되면 README에 AGENTS.md 허가 문장 예시를 싣는다 |
| R3 | bash가 있는 리뷰어가 저장소를 바꿀 수 있다 | 지시문으로 막고 M4에서 `git status`로 확인한다. 실패하면 bash를 빼고 상위 에이전트가 diff를 넘기는 방식으로 바꾼다(O2) |
| R4 | upstream이 바뀔 수 있다 | SYNC 절차를 따르고 기준 커밋을 적어 둔다 |
| R5 | 다른 패키지나 사용자 에이전트와 이름이 겹칠 수 있다 | O1에서 정한다. README에 덮어쓰기 동작을 적는다 |
| R6 | 라이선스를 잘못 해석할 수 있다 (README의 MIT와 LICENSE의 Apache-2.0) | 더 엄격한 Apache-2.0 기준을 따르고, 배포 전에 다시 확인한다 |
| R7 | pi-subagents의 매니페스트나 동작이 바뀔 수 있다 | README에 검증한 버전(0.84.0)을 적고, 큰 버전이 바뀌면 M3를 다시 돌린다 |
| R8 | 최대 5개 리뷰어라 비용이 크다 | 기본을 순차 실행으로 하고, 항목을 골라 실행하도록 안내한다. 모델 권장 설정을 README에 싣는다 |

---

## 12. 아직 정하지 않은 것

| ID | 질문 | 선택지 | 현재 기울기 |
|---|---|---|---|
| O1 | 에이전트 이름에 네임스페이스를 붙일까 | (a) 붙이지 않음: `code-reviewer` (b) `package: pr-review` → `pr-review.code-reviewer` | v0.1은 (a). 공개 배포 전에 충돌 위험을 보고 다시 정한다 |
| O2 | 리뷰어에게 bash를 줄까 | (a) 준다. diff를 직접 조회 (b) 빼고 상위 에이전트가 diff를 넘김 | (a). M4 결과를 보고 정한다 |
| O3 | `/review-pr` 프롬프트 템플릿도 넣을까 | (a) 스킬만 (b) 템플릿을 함께 넣음 (명시적 진입점) | (a). 자동 로드가 불안정하면 (b) |
| O4 | `advertise: true`로 할까 | 켜면 "에러 처리만 봐줘" 같은 개별 호출이 쉬워지지만, 상위 프롬프트 비용이 늘고 다른 리뷰 흐름에 끼어들 수 있다 | false |
| O5 | README를 어떤 언어로 쓸까 | 영어만 / 영어+한국어 | 영어 본문 + 한국어 요약 |
| O6 | 원격 저장소와 npm 스코프 | 원격 저장소(GitHub 등) 생성 / unscoped와 `@<user>/` 중 선택 | 로컬 저장소는 이미 있음. 원격 저장소와 스코프는 사용자가 정한다 |
| O7 | P0/P1/P2와 Merge verdict를 합칠까 | v0.2에서 선택 옵션으로 | 보류 |
| O8 | 에이전트를 추가할까 (요구사항 충족 확인, 보안) | v0.2 이후 | 보류 |

---

## 13. 범위 밖 / 나중에

- **TypeScript 확장 버전:** 세션이 시작될 때 upstream을 읽어 자동으로 패치하고 런타임 등록한다. `registerWorkflowResource`로 에이전트 선택을 코드로 고정한다(research §3.7).
- **하이브리드 리뷰:** toolkit 에이전트에 parallel-review의 판정과 autofix를 결합한다.
- **Pi 패키지 갤러리 미리보기:** `pi.image`
