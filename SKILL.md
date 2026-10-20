---
name: pr-code-review
description: >
  Review a GitHub pull request with three parallel subagents (general
  correctness review, Bugbot, security-review), then merge into a
  blocking / non-blocking findings list. Use when the user asks to
  "review this PR", "code review", or wants a structured PR findings
  report without praise or spec restatement.
---

# PR Code Review

Review a PR by launching three subagents in parallel, then combining
their findings into a short structured report.

## Inputs

1. **PR URL or number** (required) — e.g. `https://github.com/org/repo/pull/123`
   or `123` when the repo is already clear from the workspace.
2. Optionally **base branch** — default is the PR's base.
3. Optionally **custom instructions** forwarded to Bugbot and
   security-review (focus areas, ignore paths, etc.).

## Preconditions

- Check out the PR head branch (or ensure the working tree matches the
  PR head) before launching Bugbot / security-review so `Diff: branch
  changes` covers the PR.
- Resolve the absolute repository root (git toplevel) for Bugbot and
  security-review prompts.

## Steps

### 1. Gather PR context (parent agent)

Fetch title, body, base/head, changed files, and the full diff
(`gh pr view`, `gh pr diff`). Skim enough to brief the general-review
subagent; do not paste the PR description into the final report.

### 2. Launch three subagents in parallel

Use the Task tool with `run_in_background: false` for all three in one
message. Do not serialize them.

#### A. General review (`subagent_type: generalPurpose`)

Prompt it to:

- Review the PR diff for correctness, API/contract misuse, missing
  edge cases, inconsistent error reporting, test gaps that matter for
  the change, and local-style mismatches that could cause real bugs.
- Read surrounding call sites / existing patterns when needed.
- Return **only findings**: each with severity `blocking` or
  `non-blocking`, file path, and a one- or two-sentence rationale plus
  a concrete fix direction when obvious.
- **Do not** restate what the PR claims to do.
- **Do not** list strengths, praise, or "looks good" summary.
- If there are no findings, return an empty list.

#### B. Bugbot (`subagent_type: bugbot`)

- Set Task `description` to exactly `Bugbot`.
- Launch exactly one Bugbot subagent; do not resume a prior one.
- Prompt (fixed form):

```
Full Repository Path: <git toplevel>
Diff: branch changes
Custom Instructions: <user custom instructions, or omit this line>
```

#### C. Security review (`subagent_type: security-review`)

- Set Task `description` to exactly `Security Review`.
- Launch exactly one security-review subagent; do not resume a prior one.
- Prompt (fixed form):

```
Full Repository Path: <git toplevel>
Diff: branch changes
Custom Instructions: <user custom instructions, or omit this line>
```

### 3. Merge findings

Deduplicate overlapping items (same root cause → one finding; prefer
the sharper wording). Classify each remaining finding:

| Level | Use when |
| --- | --- |
| **blocking** | Correctness bug, security issue, broken contract, or likely production/test failure if merged as-is |
| **non-blocking** | Nit, clarity, missing hardening, style-with-teeth, or follow-up that should not hold the merge |

When Bugbot or security-review marks something critical/high and it is
real for this diff, put it under **blocking**. Downgrade false
positives or out-of-scope items; drop them if they do not apply.

### 4. Final response format

Reply with **only** this structure (no PR summary, no "what works
well", no restating the ticket/spec):

```markdown
## Blocking
- `path/to/file.py`: <finding>

## Non-blocking
- `path/to/file.py`: <finding>
```

Rules for the report:

- If a section has no items, write `None.` under that heading.
- One bullet per finding; path first when a path exists.
- Keep each bullet to one or two sentences.
- Name the source in parentheses only when useful to disambiguate
  duplicates, e.g. `(bugbot)` — do not dump raw subagent transcripts.
- Do not post GitHub review comments unless the user asks.

## Guardrails

- Never invent findings that are not grounded in the diff or subagent
  output.
- Do not block on pre-existing issues unrelated to the PR unless the
  change makes them worse.
- Prefer fewer, sharper findings over exhaustive nits.
