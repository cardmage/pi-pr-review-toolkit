---
name: pr-review
description: Comprehensive PR and change review with the pr-review-toolkit subagents (code-reviewer, silent-failure-hunter, pr-test-analyzer, comment-analyzer, type-design-analyzer, code-simplifier), ported from Claude Code's /review-pr. Use when asked to review a pull request or recent changes, or to check test coverage, error handling, code comments, or type design before committing or opening a PR. Optional aspects are code, errors, tests, comments, types, simplify, and all. Reviewers run in parallel by default; add "sequential" to run them one at a time.
---

# PR Review (pr-review-toolkit for Pi)

Invoking this skill is the operator's authorization to launch the pr-review-toolkit subagents with the `subagent` tool for this review.

If only `subagents_enable` is visible, call it first; `subagent` becomes available on the next request.

Read [references/review-pr.md](references/review-pr.md), the upstream Claude Code `/review-pr` command, and follow it. `$ARGUMENTS` there is the user's request text that came with this skill. Only these Pi differences apply:

- **Launching (step 5):** use the `subagent` tool instead of Claude Code's Task tool.
  - Default is parallel: `subagent({ workflow: "parallel", args: { tasks: [{ agent: "<name>", task: "<brief>" }, ...] } })`, then wait for the completion notification.
  - If the user asks for `sequential`, launch one at a time: `subagent({ agent: "<name>", task: "<brief>", async: false })`.
  - `code-simplifier` edits files, so never put it in a parallel batch. Run it alone with `async: false`, after the reviewers finish.
- **Briefs:** each subagent starts with a fresh context. Every task must give the absolute repository path, the exact diff command for the scope, the changed files, and the aspect to review.
- **Claude Code-only names:** ignore `/pr-review-toolkit:review-pr`, `/agents`, and model notes in the reference.
