---
name: orchestrate-code-review-dispatch
description: "Entry-point for Code Review Lead when a review task arrives and no specialist children exist yet. Use at the START of every review task heartbeat before creating any child issues, posting any comment, or reading any diff. Also invoke when resuming a partial dispatch (some children already exist). Do NOT use superpowers:dispatching-parallel-agents for code review — that skill lacks the domain-specific routing table, context templates, and wake protocol."
---

# Orchestrate Code Review Dispatch

This is the **mandatory entry-point** for every review task the Code Review Lead receives. It routes through the full dispatch pipeline in the correct order and prevents the most common failure modes: missing context in child issues, wrong or absent assignees, no blocker chain, and premature synthesis.

The Code Review Lead is an **orchestrator, not a reviewer.** You do not read diffs. You do not form findings. You set up the specialists and wait.

---

## When to use

Use this skill at the START of any heartbeat where:
- You are assigned or have checked out a review task
- No specialist child issues have been created yet

Also use when resuming a partial dispatch (some children already exist but the set is incomplete) — `review-dispatch-protocol` has a partial-dispatch recovery path.

---

## Step 1 — Scope triage

**REQUIRED SKILL:** Invoke `code-review-scope-triage` first.

This classifies the task as either:
- **STANDARD GATE** — all 5 reviewer dimensions dispatched
- **TARGETED VALIDATION** — approved subset (minimum 2; must include Correctness)

Do not proceed to Step 2 until you have a classification and a reviewable artifact (PR link or diff). If no PR link exists, comment on the parent asking for it and exit — do not create any child issues.

---

## Step 2 — Dispatch specialist child issues

**REQUIRED SKILL:** Invoke `review-dispatch-protocol` immediately after scope triage.

This skill handles all of the following — do not improvise any of it:
- Extracting PR/diff link, changed files, source issue, and scope constraints from the parent task
- Creating child issues with the standard title pattern, description template, and correct assignee
- Setting `blockedByIssueIds` on the parent to all dispatched child IDs and status to `blocked`
- Posting the dispatch comment (phase = dispatched, table of child links, expected next wake)

The specialist routing table (dimension → agent ID) lives in `review-dispatch-protocol`. Consult it every time — do not use memorized agent IDs.

---

## Step 3 — Wait for completion

After dispatch is complete, exit the heartbeat. You will be woken by:
- `issue_blockers_resolved` — all specialist children reached `done`
- `issue_children_completed` — all direct children reached a terminal state

**Do not poll children. Do not check in before the wake event.** Budget is finite.

---

## On wake — Synthesize or escalate

When woken after a dispatch:

**All children `done`?**
→ Invoke `review-synthesis-verdict`.

**One or more children unresponsive (no activity for 90+ minutes)?**
→ Invoke `stalled-reviewer-escalation` before attempting synthesis. Do not synthesize with an outstanding specialist issue.

---

## Non-negotiable rules

| Rule | Why |
|---|---|
| Always run `code-review-scope-triage` first | Determines whether all 5 dimensions apply. Skipping causes under-coverage. |
| Always use `review-dispatch-protocol` for child issue creation | Contains the routing table, description templates, and API call sequence. Improvising causes missing context and mis-routed issues. |
| Never read diffs or form findings yourself | That is the specialist reviewers' job. Opening a file here is a role violation. |
| Never synthesize with outstanding specialist issues | All dispatched children must be `done` before synthesis begins. |
| Never use `superpowers:dispatching-parallel-agents` for code review | That skill does not know the reviewer dimensions, agent IDs, or dispatch comment protocol. |
| Never create a partial dispatch and stop | Either the full classified set is dispatched, or none are. Partial dispatch with no recovery comment is worse than no dispatch. |
