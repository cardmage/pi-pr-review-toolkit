---
name: pr-review
description: Comprehensive PR and change review with the pr-review-toolkit subagents (code-reviewer, silent-failure-hunter, pr-test-analyzer, comment-analyzer, type-design-analyzer, code-simplifier), ported from Claude Code. Use when asked to review a pull request or recent changes, or to check test coverage, error handling, code comments, or type design before committing or opening a PR. Optional aspects are code, errors, tests, comments, types, simplify, and all. Reviewers run in parallel by default; add "sequential" to run them one at a time.
---

# PR Review (pr-review-toolkit for Pi)

Invoking this skill is the operator's authorization to launch the pr-review-toolkit subagents below with the `subagent` tool for this review. It does not authorize other subagents, and it does not authorize code changes except a `simplify` pass the user explicitly asked for.

If only `subagents_enable` is visible, call it first; `subagent` becomes available on the next request.

## Workflow

Read [references/review-pr.md](references/review-pr.md), the upstream Claude Code `/review-pr` command, and follow its workflow: determine the review scope, parse the requested aspects, identify changed files, choose applicable agents, launch them, aggregate the results, and present the action plan.

- Read `$ARGUMENTS` in that file as the user's request text that came with this skill, such as the text after `/skill:pr-review`, or the natural-language request that loaded it.
- The adaptation rules below take precedence over that file.

## Pi adaptation rules

### 1. Scope

Resolve the scope once, in the parent, before launching anything. Use the first rule that applies:

1. **Explicit scope in the request** wins: files or paths, a base ref (`git diff <base>...HEAD`), a commit range, or a PR number (`gh pr diff <number>`).
2. **The request is about a PR** and `gh pr view` finds one for the current branch: `gh pr diff`.
3. **Otherwise** (including when `gh` is missing or no PR exists): the working-tree changes, `git diff` and `git diff --cached`.

Then list the changed files (`git diff --name-only`, `git diff --cached --name-only`, `git diff --name-only <base>...HEAD`, or `gh pr diff --name-only`). If the diff is empty, say so and ask the user for a scope, such as a base branch, instead of launching agents.

### 2. Aspects and agent selection

Parse the request for the aspects `code`, `errors`, `tests`, `comments`, `types`, `simplify`, and `all`, and for the modifiers `sequential` and `parallel` (the default). Map equivalent natural-language requests, in any language, to the same aspects.

| Aspect | Agent |
|---|---|
| `code` | `code-reviewer` |
| `errors` | `silent-failure-hunter` |
| `tests` | `pr-test-analyzer` |
| `comments` | `comment-analyzer` |
| `types` | `type-design-analyzer` |
| `simplify` | `code-simplifier` |

- **Explicit aspects:** run exactly those agents.
- **No aspect, or `all`:** apply the "Determine Applicable Reviews" rules from the reference to the actual diff. `code-reviewer` always runs. Also run `pr-test-analyzer` when production logic changed (new branches, validation, parsing, or business logic), even if no test file changed, because missing tests are what it detects. Do not run `code-simplifier` under `all`; offer it after the review instead.
- Before launching, tell the user in one line which agents will run, which were skipped and why, and the scope.

### 3. Launching

Use the `subagent` tool where the reference says "Task".

- **Parallel (default):** launch all selected reviewers in one call:
  `subagent({ workflow: "parallel", args: { tasks: [{ agent: "<name>", task: "<brief>" }, ...] } })`.
  Results come back together. Wait for the completion notification; do not poll or sleep. If only one reviewer is selected, launch it directly with `subagent({ agent: "<name>", task: "<brief>" })`.
- **Sequential (`sequential` requested):** launch one agent at a time with `subagent({ agent: "<name>", task: "<brief>", async: false })` and wait for each result before launching the next.
- **`code-simplifier`** edits files, so it runs only when `simplify` was requested; alone, never inside a parallel batch; after any requested reviewers have finished; and only on the changed files in scope. Launch it with `async: false`. Afterwards, run `git status --short` and `git diff --stat` yourself, confirm that it changed only in-scope files, and report what changed.

### 4. Child briefs

Each child starts in a fresh context and sees nothing from this conversation. Every task must state:

- The repository's absolute working directory.
- The scope in one line, and the exact command(s) to see the diff.
- The changed files to focus on.
- The aspect to review, plus any concerns the user mentioned.
- That every finding needs `file:line` and evidence, reported in the agent's own output format.
- For reviewers: do not modify files, stage, commit, push, or comment on the PR. For `code-simplifier`: edit only the listed files and preserve behavior.

Template:

```text
Repository: <absolute cwd>
Scope: <e.g. "uncommitted changes (unstaged + staged)" or "PR #123 against main">
See the changes with: <exact command(s)>
Changed files:
- <path>
Focus: <aspect, and any user-specified concerns>
Report every finding with file:line and concrete evidence, in your standard output format.
<Reviewers: Read-only. Do not modify files, stage, commit, push, or post PR comments.>
<code-simplifier: Edit only the files listed above. Preserve behavior. List every file you changed.>
```

### 5. Aggregating results

Produce the "PR Review Summary" from step 7 of the reference, in the user's language.

- **Severity:** start from each agent's own top tier as Critical, its second tier as Important, and the rest as Suggestions. Put positive findings under Strengths. Re-rank after verification when the impact clearly differs.

  | Agent | Critical | Important | Suggestions |
  |---|---|---|---|
  | code-reviewer | Critical (90-100) | Important (80-89) | - |
  | silent-failure-hunter | CRITICAL | HIGH | MEDIUM |
  | pr-test-analyzer | Critical Gaps (8-10) | Important Improvements (5-7), Test Quality Issues | lower ratings |
  | comment-analyzer | Critical Issues | - | Improvement Opportunities, Recommended Removals |
  | type-design-analyzer | - | Concerns that allow invalid states | other Concerns, Recommended Improvements |

- **Verify:** before reporting a finding you doubt, read the cited code yourself. Drop findings that do not hold up, or mark them as unverified.
- **Deduplicate:** merge the same issue reported by several agents into one entry and list all of them, for example `[code-reviewer, silent-failure-hunter]`.
- **Failures:** if an agent fails (launch error, model resolution error, timeout), name it and the error in the summary, and keep the results of the agents that completed. In parallel mode a failed child fails the whole batch, so relaunch only the failed agents once; in sequential mode continue with the next agent. Do not silently substitute another agent.
- **No automatic fixes:** after the summary, ask whether to apply fixes and which. If `simplify` was not run, offer it.
