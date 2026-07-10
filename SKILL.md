---
name: skill-feedback
description: Capture structured feedback when a skill misfires, so it can be reported upstream to the skill's maintainer. Use this whenever a loaded skill's instructions turn out to be wrong, stale, incomplete, or ambiguous during a task; whenever you have to work around a skill's guidance to finish the job; whenever a script or reference bundled with a skill fails; or whenever the user says anything like "that skill didn't work," "report this," "the skill got that wrong," or "how do I give feedback on this skill." Also use it proactively at the end of a task if you noticed skill friction along the way, even if the user didn't mention it.
---

# Skill Feedback

Skills improve only when the friction observed during real use makes it back to the people who maintain them. Right now that loop is mostly broken: an agent hits a stale instruction, silently works around it, finishes the task, and the observation dies with the session. This skill closes the loop — it turns friction you actually experienced into a structured report a human can review and file upstream.

## What counts as friction

Report things you directly observed while following a skill:

- **Stale instructions** — the skill describes a workflow, API, flag, or UI that no longer matches reality.
- **Broken bundled resources** — a script in `scripts/` errors out, a reference doc contradicts the SKILL.md, an asset is malformed.
- **Gaps** — the skill is silent on a case the task required, and you had to improvise.
- **Ambiguity** — two readings of an instruction produce different results, and you had to guess.
- **Workarounds** — anything you did *instead of* what the skill said, in order to finish the job.
- **Triggering problems** — the skill loaded for a task it doesn't cover, or clearly should have covered the task but its instructions assume a different context.

Do **not** report: general dislike of a skill's style, hypothetical problems you didn't actually hit, or failures caused by the environment rather than the skill (missing network access, permissions). If unsure whether the skill or the environment is at fault, say so in the report — uncertainty stated honestly is still useful signal.

## The cardinal rule: human gate

**Never file, post, or submit a report anywhere yourself.** Your job ends at producing the draft report and telling the user where it could be filed. The human decides whether it's accurate, whether it leaks anything, and whether to send it. This is not a formality — an automated model-to-repo feedback channel is an injection vector, and maintainers need a accountable human on every submission. If the user asks you to auto-file, explain why the gate exists and offer the draft instead.

## Workflow

1. **Notice and remember.** When friction occurs mid-task, finish the task first — the user's goal comes before the meta-work. Keep a mental note of what happened: the exact instruction, what you expected, what actually occurred, what you did instead.

2. **Offer, don't interrupt.** At a natural end point, mention it in one sentence: "The X skill's instructions for Y were stale — want me to draft a friction report you could file upstream?" If the user declines, drop it.

3. **Draft the report.** Use the template in `assets/friction-report-template.md`. Fill every section you can from direct observation; write "not observed" rather than guessing. Quote the skill's actual instruction text verbatim (skills are meant to be quoted; this is not user data).

4. **Sanitize.** This is the step that matters most. The report describes the *skill's* behavior, not the user's business. Before showing the draft:
   - Replace the user's actual task content with a generic description ("a client outreach letter" not the client's name; "a financial spreadsheet" not the figures).
   - Strip names, emails, file contents, account details, paths containing usernames.
   - Keep only the minimum task context needed to reproduce the friction.
   Then show the user the draft and explicitly ask them to check it for anything they wouldn't want public.

5. **Point at the destination.** Identify where the report should go:
   - If the skill's SKILL.md or bundled files name a source repo or author, use that.
   - If it came from a known marketplace or plugin, the upstream repo's Issues page is the destination.
   - If the origin is unknown, say so and suggest the user check where they installed it from.
   Give the user the finished report as a copy-pasteable block (or a saved .md file if the task produced files anyway), plus the URL or location to file it. Then stop.

## Report quality bar

A maintainer reading the report cold should be able to answer three questions without follow-up: *what did the skill say, what actually happened, and what would have worked?* If your draft doesn't answer all three, it isn't done. Reproduction steps beat adjectives — "the `convert.py` script exits 1 on any filename containing a space" is filable; "the script is buggy" is noise.

One report per distinct issue. If a task surfaced three unrelated problems in one skill, that's three short reports, not one long one — maintainers triage per-issue.

## Example

**Situation:** While following a `pdf-forms` skill, its instructions said to use `fill_form.py --flatten`, but the bundled script has no `--flatten` flag; you used `--no-edit` after reading the script's argparse block.

**Report (abbreviated):**

> **Skill:** pdf-forms (installed from github.com/example/pdf-skills, commit unknown)
> **Friction type:** Stale instruction / instruction–script mismatch
> **Skill said:** "Run `python scripts/fill_form.py input.pdf --flatten` to lock the fields."
> **What happened:** `fill_form.py: error: unrecognized arguments: --flatten`
> **Workaround that succeeded:** The script's actual flag is `--no-edit` (see its argparse definitions). Ran `python scripts/fill_form.py input.pdf --no-edit`; output verified correct.
> **Suggested fix:** Update SKILL.md line referencing `--flatten` to `--no-edit`, or restore the `--flatten` alias in the script.
> **Environment:** Claude.ai container, Python 3.12, task was filling a standard government form (details withheld).
