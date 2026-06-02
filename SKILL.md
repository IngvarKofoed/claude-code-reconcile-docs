---
name: reconcile-docs
description: Use this skill to reconcile existing project docs — `docs/CONCEPT.md` and/or `docs/ARCHITECTURE.md` — against the actual codebase when they have gone thin or drifted out of sync with the code. The skill reads the docs and the code, diffs them into a gap/drift inventory, interviews the user (via AskUserQuestion) only on what the code can't answer, then edits the docs in their existing voice and produces a drift report that can surface latent bugs. Trigger this when the user says any of: "my CONCEPT/ARCHITECTURE doc is out of date", "the docs don't match the code anymore", "reconcile my docs with the codebase", "my architecture doc drifted", "audit/refresh my project docs against the code", "the docs are thin, fill the gaps", or any phrasing that implies the anchoring docs exist but no longer reflect reality. This skill REQUIRES the docs to already exist — if there are none, it's the wrong tool. Primarily user-invoked via /reconcile-docs; only auto-trigger on unambiguous requests to reconcile, refresh, or audit existing project docs against the code. Do NOT auto-trigger on "write a spec", "create the docs from scratch", or general "look at my docs".
---

# Reconcile docs

This skill brings a project's anchoring docs back into sync with the code, and enriches them with the intent that code alone can't express. It is for the case where `docs/CONCEPT.md` and/or `docs/ARCHITECTURE.md` already exist but have gone **thin** (vague, under-specified) or **drifted** (the code moved and the doc didn't).

These docs matter because a fresh agent loads them at session start to ground itself. A *true and rich* doc improves every later generation session; a *stale* doc actively misleads it — worse than no doc, because the agent trusts it. The whole job of this skill is to close the gap between **what the doc says** and **what a later LLM actually needs to know**.

## What this skill is not

- **Not a doc creator.** If `docs/CONCEPT.md` / `docs/ARCHITECTURE.md` don't exist yet, stop and say so — this is the wrong tool. Those are written by hand (or with the `spec`-style thinking phase), then scaffolded. This skill needs an existing doc to reconcile against.
- **Not a code fixer.** When the reconciliation reveals that the *code* drifted from the documented intent — a possible bug — this skill **reports** it. It does not fix it. Fixing is a separate, deliberate decision the user makes afterward.

## Hard rule: docs and the report only; code is read-only

The only files this skill may create or modify are:

- The doc(s) being reconciled (`docs/CONCEPT.md`, `docs/ARCHITECTURE.md`, or wherever the project keeps them).
- An optional drift report file, only if the user asks to persist it (see phase 6).

Everything else is read-only:

- Read, Grep, Glob, find, git log/blame/diff, ls, cat — all fine, and **essential** here. The quality of the whole skill rides on how well you read the code.
- Editing, creating, or deleting source code, config, tests — **not allowed**. If the code is wrong, that goes in the drift report, not into an edit.
- Commands with side effects (install, commit, migrate, generate) — not allowed.

This rule is what keeps the skill honest. A reconciliation pass that "just quickly fixes" a line of code has stopped being a doc pass and started being an unreviewed change.

## CONCEPT vs ARCHITECTURE — keep them distinct

They document different layers, and conflating them is the most common way these docs rot:

- **`CONCEPT.md`** — the *domain*: entities, lifecycles, vocabulary, user-facing behavior, the *why*. No class names, table names, or stack details. Mostly lives in the user's head — **interview-heavy**.
- **`ARCHITECTURE.md`** — the *how it's built*: stack, topology, internal layers/components, data-model topology, protocols, project structure. Largely **derivable from the repo** — so reconciling it is more *"confirm my inference"* than *"tell me from scratch."*

This difference shapes the interview: ARCHITECTURE questions mostly confirm what you read in the code; CONCEPT questions mostly extract what isn't anywhere in the code.

## Establish the doc's contract first — it decides what "drift" even means

You cannot label anything "drift" until you know what the doc is *promising to describe*. The same sentence — "the system evaluates calculated slots" — is drift in one doc and perfectly correct in another. A doc has one of three contracts:

- **As-built** — describes what exists *today*. Here, "doc says X, code doesn't do X" is genuine drift, and the remedy is to trim or correct the doc (or flag a code bug — see the fork).
- **Target vision** — describes the *intended whole product*, including parts not built yet. Here the same finding is **not drift at all** — it's an unbuilt-but-planned feature. The remedy is a **status marker** ("planned" / "phase 2" / a Built-vs-Planned note), so a fresh agent isn't fooled into thinking it exists today. You do *not* delete these sections.
- **Mixed** — the common, messy reality: mostly as-built, with some aspirational sections, and often *inconsistent* about marking which is which.

**This single decision usually governs the entire drift bucket**, so it must be settled *before* you present findings — not buried after a long list. And the doc often signals its own contract: if it already marks some sections "phase 2" but describes everything else in flat present tense, that inconsistency *is itself the finding* — it's a vision doc that forgot to mark most of its unbuilt parts. Read for those signals while building the inventory (phase 2), form a hypothesis about the contract, and put it to the user as the very first question (phase 3).

## The workflow

Six phases. The center of gravity is phase 2 — get that right and the rest is easy.

### 1. Scope which doc

Ask the user, in one question: are we reconciling **CONCEPT**, **ARCHITECTURE**, or **both**? This selects the question set and the reading focus. If they invoked the skill naming a specific doc, skip the question and confirm in a sentence.

Then locate the file(s). Default to `docs/CONCEPT.md` / `docs/ARCHITECTURE.md`; if the project's convention or a `CLAUDE.md` points elsewhere, follow that. If a named doc doesn't exist, stop — this is the wrong skill (see "What this skill is not").

### 2. Build the gap/drift inventory (the load-bearing step)

This is silent groundwork — **you** do the work, the user waits. Read the chosen doc(s) thoroughly first so you hold the full set of claims in mind. Then verify those claims against the code.

**Fan this out across parallel subagents — it's the default, not an optimization.** Building the inventory is structurally a set of *independent* checks: each section or feature the doc describes is a separate "does the code actually do this, and how?" question, and they don't depend on each other. That makes it a textbook fan-out. Slice the doc's claims into coherent areas (by section, feature, or subsystem) and spawn a read-only subagent per slice — the `Explore` or `general-purpose` agent type fits, since this phase touches no files. Give each agent the relevant doc excerpt and ask it to return structured findings — each tagged drift / gap / thin / missing-intent, each with concrete evidence (`file:line`, the contradicting identifier). Then you, the orchestrator, merge, dedup, and classify what comes back.

Two reasons this is the default, not a nicety. **Speed:** ten claims verified concurrently beats ten verified in series. **Context hygiene:** offloading the code-spelunking to subagents keeps *your* context clean and focused, so you arrive at the interview (phases 3–4) sharp rather than buried under file dumps — and the interview is where your judgment actually matters. The only time to skip the fan-out is a genuinely small, single-area doc where one pass covers everything; say so if you make that call, rather than silently defaulting to serial.

Sort every finding into one of four buckets:

- **Drift** — the doc makes a claim the code contradicts. *"ARCHITECTURE says single-table; the migrations show two."* This is the highest-value bucket and the trickiest. **Its dominant shape is "described in present tense, but not built,"** and whether that's actually drift depends entirely on the doc's contract (see above): under an as-built contract it's drift to trim or fix; under a target-vision contract it's a planned feature needing a status marker, not a deletion. Don't pre-judge — that's what the phase 3 framing question settles. Genuine drift (built differently than described) also has a fork — see phase 4.
- **Gap** — the code has a whole entity, subsystem, or behavior the doc never mentions. *"There's a `RetryQueue` module with no counterpart in ARCHITECTURE."*
- **Thin** — the doc names something but never pins it down: an undefined vocabulary word, an invariant stated nowhere, a lifecycle with missing states. *"CONCEPT uses `Carrier` throughout but never says what one is."*
- **Missing intent** — the *why* and especially the *why-not* that lives in neither the doc nor the code. *"Nothing records why caching was deliberately avoided."* This bucket comes from your judgment about what a later LLM would get wrong, not from a literal diff.

**The discipline that makes this skill worth using: only surface what the code can't answer for itself.** If an LLM could read the repo and get the right answer, it doesn't belong in the inventory — asking about it wastes the user's attention and persisting it creates a doc that drifts again tomorrow. Filter hard. A short inventory of genuinely non-derivable findings beats a long one padded with the obvious.

### 3. Lead with the governing question, then show the inventory

The failure mode here is dumping 20 raw findings as a wall of text and burying the decision that would have collapsed half of them. Avoid it in two moves.

**First, ask the governing question(s) via `AskUserQuestion` — before the full list.** If one decision reframes a whole bucket (almost always the doc's *contract* — as-built vs target-vision vs mixed, per the section above), that question comes first, as a real `AskUserQuestion` with grounded options:

> *"Your CONCEPT describes ~15 features in present tense that aren't in the code — but it already marks Stream/Pipeline as 'phase 2'. What is CONCEPT meant to be?"*
> - **Target product vision** → I'll keep those sections and add a clear built-vs-planned status signal so a fresh agent isn't misled.
> - **Description of what's built today** → I'll trim/move the unbuilt features out of present tense.
> - **Mixed** → *(Other)* — tell me which sections are which.

The answer often resolves a dozen findings at once, turning "interview me on 15 things" into "apply one uniform remedy to 15 things, then interview on the few that are genuinely ambiguous."

**Second, collapse repeated patterns instead of enumerating every instance.** If 15 findings are all the same shape ("present tense, not built"), present them as *one pattern with a count and a few representative examples*, not 15 numbered lines. Reserve the line-by-line list for the findings that are genuinely distinct (the gaps, the thin spots, the real built-differently drifts). A reader can prune "the unbuilt-features pattern (15 items, examples: calculated slots, audit trail, dry run)" far faster than fifteen separate bullets.

Then let the user **prune** — *"ignore the migrations one, that table's deprecated"* — before the detailed interview. The pruned, contract-framed inventory also doubles as a standalone drift report if they stop here. Don't machine-gun from reading straight into a long list of questions.

### 4. Interview — only on what survived

By now the governing question (phase 3) has usually done most of the work: a whole pattern of findings now has one agreed remedy and needs no per-item interview. What's left is the genuinely ambiguous remainder — the real built-differently drifts, the gaps worth documenting, the thin spots, the missing intent. Interview on *those*, not on the items the contract answer already settled. If the contract answer plus pruning left nothing ambiguous, say so and go straight to phase 5 — a short interview is a sign the skill worked, not that it failed.

Drive the surviving findings through `AskUserQuestion` — actually call the tool, don't ask in prose. Options must be **grounded in what you found**, not generic: a question built from the code (*"I see X in the code — intentional, deprecated, or a mistake?"*) is worth far more than a cold multiple-choice.

`AskUserQuestion` takes at most four questions per call and 2–4 options each, so this is naturally multi-round. Pace it like a conversation: a few related questions, integrate, continue. Always include the "Other" path — for CONCEPT especially, the real answer often isn't one of the options, and the options are there to prime thinking, not to constrain it.

**The drift fork — the move that earns this skill its keep.** A drift finding is a genuine fork that only the user can resolve, because the doc and the code can't both be right:

> *"ARCHITECTURE says single-table; the code uses two. Which is true?"*
> - **The doc is stale** → update the doc to match the code.
> - **The code drifted from the design** → the doc captured the intent; the code diverged. **This is a possible bug — flag it in the drift report, don't change the doc to bless it.**
> - **Both partially / it's nuanced** → *(Other)* — let me describe it.

That middle option is why reconciliation is more than doc-editing: it can surface latent bugs a pure doc-writer would silently paper over by "updating the doc to match the code."

For the other buckets the questions are simpler: **gap** → "is this worth documenting at this layer, and what's the intent behind it?"; **thin** → "define this term / state this invariant"; **missing intent** → "why this, and what was deliberately *not* done?"

### 5. Synthesize edits into the docs — in their voice

Edit the doc(s) directly. Two rules govern *how*:

- **Write prose, don't transcribe Q&A.** The answers are raw material; the doc is the product. A doc assembled from pasted multiple-choice answers reads like a form and nobody trusts it. Turn the answers into narrative that matches the doc's existing voice and structure — skim the surrounding sections first and match their density and tone.
- **Reconcile in the right direction — there are three, not one.** (a) *Write in*: gaps, thin spots, and stale-doc drifts get corrected into the doc. (b) *Mark, don't delete*: under a target-vision contract, unbuilt-but-planned features stay, but get a clear status signal — a per-section "planned / phase 2" marker or a top-level Built-vs-Planned note — so a fresh agent reads them as intent, not reality. Match whatever marking convention the doc already uses for its existing "phase 2" sections. (c) *Hands off*: code-drifted-from-design findings go to the report (phase 6) untouched — the doc already had the intent right.

Make surgical edits to the relevant sections rather than rewriting whole files — you're reconciling, not replacing.

### 6. Hand off: drift report + inline markup review

Two deliverables go back to the user.

**The drift report** (in chat). Summarize what changed and, separately and prominently, the **possible bugs** — the findings where the code drifted from documented intent. Each with file/identifier and a one-line "doc says X, code does Y." These are actionable items the user may want to investigate as bugs. If there are any such findings, offer to persist them so they survive the conversation — but only the code-drifted items; the doc edits speak for themselves.

When persisting, name the file so repeated runs never collide. Use:

```
docs/drift-reports/YYYY-MM-DD-<scope>-<n>.md
```

- `<scope>` is which doc was reconciled this run: `concept`, `architecture`, or `concept-architecture` if both. This is what keeps a same-day CONCEPT run from clobbering a same-day ARCHITECTURE run.
- `YYYY-MM-DD` is today's date (the system date in your context, or `date +%Y-%m-%d`).
- `<n>` is a counter that is **always present, starting at `1`** — so the first report of a given scope on a given day is `…-concept-1.md`, not bare `…-concept.md`. Keeping the counter even when there's only one means every report filename has the same shape, so they sort and glob predictably.
- **Before writing, check which `<n>` files already exist** for that date and scope, and use the next free number — a second same-day, same-scope run becomes `…-concept-2.md`, then `…-concept-3.md`. Never overwrite a prior report; it may hold bugs the user hasn't actioned yet.
- Create `docs/drift-reports/` if it doesn't exist. Keeping reports in their own subfolder stops them from cluttering `docs/` as they accumulate.

**The markup pass.** Tell the user the doc edits are in place and invite inline review:

> *"I've reconciled `docs/ARCHITECTURE.md`. If you want changes, add inline comments anywhere in the file by prefixing a line with `////` — e.g., `//// this oversimplifies the queue, it's actually two stages`. Reply when you've added comments, or just tell me you're happy as-is."*

When the user signals they've commented (or answers any remaining questions):

1. Re-read the doc from disk.
2. Find every line beginning with `////`, treat each as feedback on its surrounding context, integrate it, then **remove the `////` line itself** — the marker is review scaffolding, not part of the doc.
3. **If the user rewrote parts themselves, their words win** — make the rest of the doc consistent with their edit, don't revert it.
4. Surface any new question the changes expose.

Loop phases 5–6 until the user is satisfied. Then **stop** — don't drift into fixing the bugs the report surfaced. That's a separate decision.

## Style guidelines for the edits

- **Concrete over abstract.** "An `Order` moves `draft → placed → fulfilled` and never backwards" beats "orders have a lifecycle."
- **Reference real names** where they belong — but mind the layer: ARCHITECTURE names files and types; CONCEPT names domain entities and vocabulary, not classes.
- **Explain the why, especially the why-not.** The rejected alternative is the single most valuable thing to capture, because it's where a later LLM most confidently does the wrong thing.
- **Match the doc's existing voice.** Narrative if it's narrative, bullets if it's bullet-heavy.

## Common mistakes to avoid

- **Dumping the full inventory and burying the governing question.** The single worst experience this skill can produce: 20 raw findings as a wall of text, with the one decision that collapses half of them tacked on at the bottom. Settle the doc's contract *first*, via `AskUserQuestion`, and collapse repeated patterns into a count. A reconciliation that opens with a 20-item list and no question has the order exactly backwards.
- **Calling unbuilt-but-planned features "drift" before settling the contract.** "Described in present tense, not built" is only drift if the doc is supposed to be as-built. In a vision doc it's a missing status marker. Decide the contract before you decide the remedy.
- **Asking what the code already answers.** "What test framework?" when `package.json` says so. The entire value proposition is the non-derivable; derivable questions burn trust and create tomorrow's drift.
- **Skipping the inventory hand-off.** Jumping from reading straight into questions denies the user the chance to prune, and hides the drift report.
- **Editing the doc to "bless" a code bug.** When the code contradicts a deliberate design intent, the answer may be that the *code* is wrong. Reconciling the doc to match it silently buries a bug. That's the drift fork — don't default to "doc is stale."
- **Transcribing the interview.** Pasted Q&A is not a doc. Synthesize prose in the doc's voice.
- **Touching source code.** Read-only on everything but the doc and the optional report. A bug goes in the report, not into an edit.
- **Blurring CONCEPT and ARCHITECTURE.** Class names creeping into CONCEPT, or domain rationale crowding out topology in ARCHITECTURE. Keep the layers clean.
- **Padding the inventory.** A long list of obvious findings looks thorough and helps little. Filter to the genuinely non-derivable.
