---
name: planforge
description: Use when someone wants a detailed technical plan for an application before writing code — "plan out this app", "create a technical plan for X", "/planforge new", or wants an existing plan/spec evaluated and gap-filled — "evaluate this plan", "critique this plan doc", "grill this plan for gaps", "import this spec and check it for holes", "/planforge import". Produces a 16-section technical plan via a multi-agent critique loop (a 5-persona adversarial council reviews each round, you answer the open questions, the plan gets refined) — repeating until the council finds no high-severity gaps. Runs entirely through Claude Code (this session's subscription), not a separate API-billed tool.
argument-hint: "new \"App Name\" | import <path to existing plan doc> | resume <slug> | show <slug> | finalize <slug> | list"
---

## What this does

PlanForge takes an application description — or an existing plan document that's incomplete or informal — and produces a detailed 16-section technical plan before any code gets written. A 5-persona Critic Council (see `references/council.md`) adversarially reviews each draft, you answer the open questions it surfaces, and the plan gets refined. Repeat until the council finds nothing HIGH severity left, or you say stop.

It runs entirely inside Claude Code, so it uses your existing Claude subscription and needs no separate API key.

Read `references/sections.md` before drafting or refining any plan — it has the exact schema and the required `implementation` fields for all 16 sections. Read `references/council.md` before running any critique round — it has the five persona mandates and the exact output contract for compiling their findings.

## Storage

One JSON file per plan at `~/planforge-plans/<slug>.json`, plus a rendered `<slug>.md` written when a plan is finalized. Create the directory if it doesn't exist. Slug = the project name, lowercased, non-alphanumeric runs collapsed to single hyphens, trimmed of leading/trailing hyphens.

Plan file shape:

```json
{
  "slug": "my-app",
  "project_name": "My App",
  "description": "the original description, or the full text of an imported source doc",
  "created_at": "ISO timestamp",
  "updated_at": "ISO timestamp",
  "status": "in_progress | finalized",
  "rounds": [
    {
      "number": 1,
      "timestamp": "ISO timestamp",
      "plan": { "sections": [ /* 16 section objects, see references/sections.md */ ] },
      "critique": { /* see references/council.md compiled shape, or null if not yet critiqued */ },
      "qa_log": [
        { "question_id": "q1", "section_id": "...", "question": "...", "why_it_matters": "...", "severity": "...", "answer": "...", "discussion": [{ "role": "critic|user", "content": "..." }] }
      ]
    }
  ],
  "final_plan": null
}
```

Write the JSON file after every meaningful step (round generated, each Q&A answer recorded) so a plan is always resumable even if the conversation is interrupted.

`MAX_ROUNDS = 6`. Auto-finalize at that limit regardless of convergence.

## Step 0: Parse the command

`$ARGUMENTS` starts with a subcommand: `new`, `import`, `resume`, `show`, `finalize`, or `list`. If it's ambiguous or empty, ask the user what they want to do rather than guessing.

## `new "<project name>"`

1. If no description was given, ask for one: purpose, target users, key features, known constraints.
2. Compute the slug. If `~/planforge-plans/<slug>.json` already exists, tell the user to use `resume <slug>` instead and stop.
3. Create the plan file (`status: in_progress`, `rounds: []`, `final_plan: null`).
4. Run **Round 1** using the **Planner** procedure (below).

## `import <path>`

1. Read the full file at `<path>`. If it doesn't exist or is empty, say so and stop. Don't summarize or compress it — the council needs the actual detail.
2. Infer a project name from the document's title/filename if the user didn't give one; confirm with them if it's not obvious.
3. Store the full source document text as `description` in the plan file (the council re-reads it every round for context, same as the original description field).
4. Create the plan file, then run **Round 1** using the **Importer** procedure (below) instead of the Planner.

### Importer procedure (round 1 of `import` only)

Map the source document into the 16-section schema:
- For any section (or field) the source actually specifies, use its real content faithfully — preserve the actual decision and rationale, don't rephrase away specifics.
- For any section the source doesn't address, or addresses too thinly to extract a real decision, draft a best-effort section using sound architectural judgment for the described app, and prefix every `agent_notes` entry in that section with `[GAP-FILLED]`.
- Still satisfies every rule in `references/sections.md` (non-empty overview, decisions, risks, agent_notes — even gap-filled sections need all of these).

### Planner procedure (round 1 of `new` only)

Draft all 16 sections directly from the description (and any accumulated Q&A if this is a later round after skipped questions), following `references/sections.md` exactly. No section may be thin or say "TBD."

## `resume <slug>`

1. Load the plan file. If not found, say so and list available slugs.
2. If `status: finalized`, tell the user it's done and give the file paths; stop.
3. Look at the last round. If it has a critique with unanswered `open_questions` (not present in that round's `qa_log`), go straight to **Q&A phase** for those. Otherwise, run the next round using the **Synthesizer procedure**.

## The critique round (runs after every plan draft — Planner, Importer, or Synthesizer output)

1. Save the round's `plan` to the JSON file immediately (`critique: null` for now).
2. Spawn the 5-persona council **in parallel, one Agent call each** — see `references/council.md` for mandates and the exact output contract each persona must follow.
3. Once all five return, act as **Judge**: compile their findings into the single critique object shape in `references/council.md`. Save it into the round and write the file.
4. Print a compact summary to the user: the `summary`, `overall_severity`, `convergence_ready`, the issues list (grouped by severity), the open question count, and the five council scores on one line.
5. If `convergence_ready` is true, or this round number `>= MAX_ROUNDS`, go to **Finalize** with this round's plan and stop.
6. Otherwise, go to **Q&A phase**.

## Q&A phase

Present the round's `open_questions` one at a time, in severity order (HIGH first). For each:
- If it has `suggested_options`, use `AskUserQuestion` with those as options (plus whatever custom-answer affordance the tool gives) — include `why_it_matters` in the option/question description.
- If it doesn't, just ask it as a normal chat question.
- Support these typed responses at any point: `/skip` (drop this question, move on), `/finalize` (stop answering, go straight to Synthesizer with answers collected so far), `/discuss` (see below).
- Record every answered (or explicitly skipped) question into that round's `qa_log`, writing the file after each one.

**`/discuss`**: identify which council persona(s) raised this question (`raised_by`). Spawn a single quick `Agent` call playing that persona (or the first-listed one if several), giving it: the original question, the plan sections it concerns, and the user's opening reply. Relay its response, let the user keep replying, until they type `/done` — append the whole exchange to that question's `discussion` array, then record the user's original reply as the `answer` and move on.

Once all open questions are resolved (answered/skipped) or the user typed `/finalize`, run the next round using the **Synthesizer procedure**.

### Synthesizer procedure (round 2+)

Take the previous round's `plan`, its `critique.issues` (apply every recommended fix directly — these don't need re-asking), and this round's `qa_log` answers (incorporate exactly what the user decided). Produce a refined 16-section plan. Carry forward every section, even ones nothing changed about. Then go straight to **The critique round** again on this new plan.

## `finalize <slug>`

Force-finalize at the current state regardless of remaining issues: take the last round's `plan`, set `status: finalized`, `final_plan` = that plan, write the JSON file, then render and write `<slug>.md` (all 16 sections, human-readable — mirror the JSON structure: title, overview, decisions with rationale, implementation fields, risks with mitigations, agent notes). Tell the user both file paths and that the plan is ready to hand to build-phase agents.

## `show <slug>`

Load the plan (`final_plan` if finalized, else the latest round's `plan`). Print each section's id, title, and overview.

## `list`

List every `*.json` file in `~/planforge-plans/`, printing slug, project name, status, round count, and last-updated time.
