# pi-pr-review-toolkit

Claude Code's [pr-review-toolkit](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/pr-review-toolkit) review agents and `/review-pr` workflow, packaged for [Pi](https://github.com/earendil-works/pi) and [pi-subagents](https://github.com/nicobailon/pi-subagents).

This package contains no code, only agent definitions, one skill, and documentation.

| Agent | Looks at | Output |
|---|---|---|
| `code-reviewer` | AGENTS.md/CLAUDE.md compliance, bugs, code quality | Issues with confidence ≥ 80 (Critical 90-100, Important 80-89) |
| `silent-failure-hunter` | Swallowed errors, broad catch blocks, unjustified fallbacks | CRITICAL / HIGH / MEDIUM with hidden errors and fixes |
| `pr-test-analyzer` | Behavioral test coverage and test quality | Gaps rated 1-10 |
| `comment-analyzer` | Comment accuracy and comment rot | Critical issues, improvements, removals |
| `type-design-analyzer` | Type invariants and encapsulation | Four ratings, 1-10 each |
| `code-simplifier` | Clarity and consistency, preserving behavior | **Edits files** in the given scope |

> **한국어 요약:** Claude Code의 pr-review-toolkit 리뷰 에이전트 6개와 `/review-pr` 흐름을 Pi(pi-subagents)에서 쓰게 해 주는 패키지입니다. pi-subagents를 먼저 설치한 뒤 `pi install npm:pi-pr-review-toolkit`으로 설치하고, "PR 리뷰해줘" 또는 `/skill:pr-review`로 실행합니다. 기본은 순차 실행이고, `parallel`을 붙이면 병렬로 실행합니다. 리뷰어 5개는 읽기 전용이며, `code-simplifier`는 `simplify`를 요청했을 때만 바뀐 파일 범위 안에서 코드를 직접 고칩니다. 모델은 패키지에 고정되어 있지 않으므로 아래 "Recommended model settings"처럼 `agentOverrides`로 지정하세요.

## Requirements

- Pi (tested with pi-coding-agent 1.4.0)
- pi-subagents 0.84.0 or later, installed first:

  ```bash
  pi install npm:pi-subagents
  ```

Pi packages cannot depend on each other through npm, so this package does not install pi-subagents for you.

## Install

```bash
pi install npm:pi-pr-review-toolkit
```

From a local checkout (Pi reads it in place, so edits apply after `/reload`):

```bash
pi install ./pi-pr-review-toolkit
```

Check that the agents are registered: `/subagents`, or ask the agent to run `subagent({ action: "list" })`. The six agents should appear with source `package`.

## Usage

The `pr-review` skill is the entry point. It reads the upstream `/review-pr` workflow and applies Pi-specific rules on top.

| You type | What runs |
|---|---|
| `Review this PR` / `PR 리뷰해줘` | Applicable reviewers, one at a time (if the skill loads automatically) |
| `/skill:pr-review` | Same, but forces the skill to load |
| `/skill:pr-review tests errors` | Only `pr-test-analyzer` and `silent-failure-hunter` |
| `/skill:pr-review all parallel` | All applicable reviewers at once |
| `/skill:pr-review simplify` | `code-simplifier` on the changed files (edits them) |
| `/skill:pr-review code against main` | `code-reviewer` on `git diff main...HEAD` |

Aspects: `code`, `errors`, `tests`, `comments`, `types`, `simplify`, `all` (default). Add `parallel` to run the selected reviewers concurrently.

**Scope.** In order of precedence: an explicit scope in your request (files, base ref, commit range, PR number); the current branch's PR when you ask about a PR and `gh` is available; otherwise uncommitted changes (`git diff` and `git diff --cached`).

**Result.** A `# PR Review Summary` with Critical, Important, Suggestions, Strengths, and Recommended Action. Each finding is tagged with the agent and `file:line`. The parent agent checks doubtful findings itself, merges duplicates, and asks before applying any fix.

You can also call a single agent directly, for example "run silent-failure-hunter on my uncommitted changes".

## Recommended model settings

The agents do not pin a model, so they use your subagent defaults (`subagents.defaultModel`, then the session model). Upstream uses Opus for `code-reviewer` and `code-simplifier` and inherits the main model for the others. To get something similar, set overrides in `~/.pi/agent/settings.json` (or the project's `.pi/settings.json`):

```json
{
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
}
```

Use model ids from your registry (`provider/id` or `id`). Claude Code aliases such as `opus` do not resolve in Pi. Check the result with `/subagents-models`. Every agent sets `thinking: high`; override `thinking` the same way if you want less.

Other useful overrides (`agentOverrides.<name>`):

- `"tools": ["read", "grep", "find", "ls"]` removes `bash` from a reviewer. It can then no longer run `git diff` itself, so the parent must pass the diff in the task.
- `"disabled": true` hides an agent.
- `"advertise": true` lists an agent in the parent's system prompt, so it can be picked without the skill.

## Customizing an agent

A user agent (`~/.pi/agent/agents/`) or project agent (`.pi/agents/`) with the same name replaces the packaged one. To start from the packaged prompt:

```text
subagent({ action: "eject", agent: "code-reviewer" })
```

## Differences from Claude Code

- **Models:** no model in frontmatter; Claude Code aliases (`opus`, `inherit`) are not used. Configure with `agentOverrides`.
- **Invocation:** the agents are not advertised and Pi does not launch them proactively. Use the `pr-review` skill or ask for an agent by name. pi-subagents only delegates when you ask, and invoking the skill counts as asking.
- **Project instructions:** agents receive the project's AGENTS.md/CLAUDE.md (`inheritProjectContext: true`), and "CLAUDE.md" in the prompts also means AGENTS.md.
- **Tools:** reviewers get `read, grep, find, ls, bash`; `code-simplifier` also gets `edit, write`. Reviewers are told to use bash read-only. This is an instruction, not a sandbox.
- **Workflow:** `code-simplifier` runs only when you ask for `simplify`, never as part of `all`. `pr-test-analyzer` also runs when logic changed without test changes. Staged changes are part of the default scope.
- **Prompt patches (P1-P6):** project-specific rules in `silent-failure-hunter` (Sentry, Statsig, `errorIds.ts`) and `code-simplifier` (hard-coded TypeScript/React style) are replaced with "follow the project's own conventions". See [NOTICE](NOTICE) for the full list.

## Troubleshooting

- **The skill does not load from a natural-language request.** Use `/skill:pr-review`.
- **The parent refuses to launch subagents.** Invoke the skill explicitly, or add a line to your AGENTS.md such as: "The pr-review skill may launch its review subagents when I ask for a review."
- **`Unknown agent` or the agents are missing.** Run `/reload`. `subagent({ action: "doctor" })` reports discovery problems.
- **Model resolution errors.** Use exact ids from `subagent({ action: "models" })` in `agentOverrides`.

## License and attribution

Apache License 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).

The agents and the `/review-pr` workflow are derived from Anthropic's pr-review-toolkit in [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/pr-review-toolkit) at commit `30202c7`. Each derived file has a change notice at the top. This is not an official Anthropic package.
