# Upstream 동기화 절차

이 패키지의 에이전트 6개와 `skills/pr-review/references/review-pr.md`는 upstream
`anthropics/claude-plugins-official`의 `plugins/pr-review-toolkit`에서 가져왔다.
코드 없는 패키지를 유지하기 위해 동기화는 아래 절차를 손으로 따른다(plan D12).

- 현재 기준 커밋: `30202c767cfaf325212b3921ab484f83adaec850` (2026-10-26)
- 기준 커밋이 적힌 곳: 각 파생 파일 맨 위 YAML 주석(`@30202c7`), `NOTICE`, `README.md`, 이 문서

## 1. upstream 최신 커밋 확인

```bash
gh api "repos/anthropics/claude-plugins-official/commits?path=plugins/pr-review-toolkit&per_page=5" \
  --jq '.[] | "\(.sha[0:7]) \(.commit.committer.date) \(.commit.message | split("\n")[0])"'
```

맨 위 커밋이 기준 커밋과 같으면 할 일이 없다.

## 2. 바뀐 점 보기

```bash
git clone --filter=blob:none https://github.com/anthropics/claude-plugins-official /tmp/cpo
cd /tmp/cpo
git diff 30202c7..<new> --stat -- plugins/pr-review-toolkit
git diff 30202c7..<new> -- plugins/pr-review-toolkit/agents plugins/pr-review-toolkit/commands plugins/pr-review-toolkit/LICENSE
```

- 에이전트가 새로 생기거나 없어졌으면 이 패키지에 넣을지 따로 정한다.
- `LICENSE`가 바뀌었거나 `NOTICE` 파일이 생겼으면 재배포 조건을 다시 확인한다.
- 프론트매터(`description`, `model`, `color`) 변경은 반영하지 않는다. 우리 프론트매터는 따로 관리한다.

## 3. 바뀐 문단 반영과 패치 재적용

본문 변경은 문단 단위로 옮긴다. 옮긴 뒤 아래 패치가 그대로 살아 있는지 확인한다.
원문 문장이 바뀌어 패치 위치가 사라졌다면, 같은 의도로 새 문장에 다시 적용한다.

| ID | 파일 | 확인할 것 |
|---|---|---|
| P1 | 에이전트 6개 | 본문 끝 `## Pi runtime notes`가 있다. 리뷰어 5개는 Read-only 항목, code-simplifier는 Edit boundary 항목 |
| P2 | silent-failure-hunter | `logError`, `logForDebugging`, `logEvent`, `errorIds`, `Sentry`, `Statsig`가 없다. Special Considerations는 프로젝트 중립 문구 |
| P3 | silent-failure-hunter | Your Tone의 "Call out every instance …"가 "evidence-backed … no speculative issues" 문구 |
| P4 | code-simplifier | §2에 특정 스타일 목록(ES modules, `function` keyword, 반환 타입, React Props, try/catch 회피)이 없다. §5와 마지막 문단에 "current session", "autonomously and proactively"가 없다 |
| P5 | code-reviewer | Review Scope가 `git diff`와 `git diff --cached`를 모두 언급 |
| P6 | references/review-pr.md | 원문 그대로 + 맨 위 변경 고지 YAML 주석 + `> **Pi note:**` 한 줄 |

남은 사내 규칙 키워드 확인:

```bash
grep -nE 'errorIds|Sentry|Statsig|logForDebugging|logError|logEvent|function` keyword|arrow functions|ES modules|autonomously|current session' agents/*.md \
  || echo "none left"
```

원본과 비교해 의도한 변경만 있는지 본다(프론트매터, P1–P6만 달라야 한다):

```bash
for f in agents/*.md; do diff -u /tmp/cpo/plugins/pr-review-toolkit/$f $f; done
diff -u /tmp/cpo/plugins/pr-review-toolkit/commands/review-pr.md skills/pr-review/references/review-pr.md
```

## 4. 기준 커밋 표기 갱신

- 파생 파일 7개 맨 위 주석의 `@30202c7`
- `NOTICE`의 `Base commit`과 날짜
- `README.md`의 License and attribution 절
- 이 문서 맨 위
- `CHANGELOG.md`에 새 버전 항목

```bash
grep -rn '30202c7' agents skills NOTICE README.md docs/SYNC.md
```

## 5. 검증 다시 돌리기

M1(파싱·필드)과 M3(설치 통합)을 다시 확인한다. 아래 명령은 저장소 루트에서 실행하며, 저장소 루트의
`package.json`을 통해 에이전트를 `package` 소스로 찾는다.

```bash
node --input-type=module -e '
import { pathToFileURL } from "node:url"; import path from "node:path"; import os from "node:os";
const m = await import(pathToFileURL(path.join(os.homedir(), ".pi/agent/npm/node_modules/pi-subagents/src/agents/agents.js")).href);
const r = m.discoverAgents(process.cwd(), "both");
for (const a of r.agents.filter((a) => a.source === "package"))
  console.log(a.name, a.tools?.join(","), a.thinking, a.systemPromptMode, a.inheritProjectContext, a.inheritSkills, a.acceptanceRole, a.model ?? "-");
console.log("diagnostics:", r.agentDiagnostics.length);'
```

기대값: 6줄. 리뷰어는 `read,grep,find,ls,bash high replace true false read-only -`,
code-simplifier는 `...,edit,write high replace true false writer -`.

스킬은 Pi에서 `/reload` 뒤 `/skill:pr-review`가 경고 없이 로드되는지 본다.
