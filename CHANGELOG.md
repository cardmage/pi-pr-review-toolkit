# Changelog

## 0.1.0 - Unreleased

First release.

- Six review agents from Claude Code's pr-review-toolkit (upstream commit `30202c7`), adapted for pi-subagents: `code-reviewer`, `silent-failure-hunter`, `pr-test-analyzer`, `comment-analyzer`, `type-design-analyzer`, `code-simplifier`.
- `pr-review` skill: the upstream `/review-pr` workflow with Pi adaptation rules. Parallel by default, `sequential` on request.
- Patches P1-P6 (see NOTICE): AGENTS.md support, read-only reviewers, project-neutral error-handling and style rules, scope taken from the task.
- No model pinned in frontmatter. Set models with `subagents.agentOverrides`.
