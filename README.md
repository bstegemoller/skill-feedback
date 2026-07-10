# skill-feedback

**A skill that closes the missing feedback loop in the skills ecosystem.**

Skills are how agents learn workflows — but the observations agents make *while using* skills mostly die with the session. An agent hits a stale instruction, quietly works around it, finishes the task, and the maintainer never hears about it. Feedback only arrives when a human happens to notice and bothers to file an issue.

This skill fixes that with the smallest possible mechanism: when an agent experiences friction with a loaded skill (stale instructions, broken bundled scripts, gaps, ambiguity), it drafts a structured, sanitized friction report at the end of the task. **A human reviews it and files it upstream.** The agent never submits anything itself.

## Why the human gate is the design, not a limitation

An automated model-to-repo feedback pipeline would be a self-modifying instruction store — a prompt-injection vector and an accountability gap. The git PR/issue model already solves attribution, review, and rollback. What's missing isn't a new channel; it's the *draft*. This skill produces the draft; humans keep the send button.

## Install

Drop the `skill-feedback/` folder into your skills directory:

- **Claude Code:** `.claude/skills/skill-feedback/` (project) or `~/.claude/skills/skill-feedback/` (personal)
- **Claude.ai:** upload via Settings → Capabilities → Skills
- **Other agents supporting the open skill format:** per your tool's skill directory

## What a report looks like

See [`assets/friction-report-template.md`](assets/friction-report-template.md). Every report answers three questions a maintainer needs: what did the skill say, what actually happened, and what would have worked. Reports are sanitized — they describe the skill's behavior, never the user's data.

## For skill maintainers: adopt the convention

If you maintain a skill repo, you can make these reports land cleanly:

1. Add a `friction-report.md` issue template (copy ours from `assets/`) to `.github/ISSUE_TEMPLATE/`.
2. Optionally add a `FEEDBACK.md` at your repo root stating where friction reports should go.
3. Label incoming reports `skill-friction` so agent-observed issues are triageable separately from feature requests.

The value compounds with adoption: consistent report structure means maintainers can triage in seconds, and patterns across reports (many agents hitting the same line) become visible.

## Contributing

Issues and PRs welcome — including friction reports about this skill itself, which would be pleasingly recursive.

## License

Apache 2.0
