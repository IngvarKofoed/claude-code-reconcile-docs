# reconcile-docs

A Claude Code skill that brings a project's anchoring docs — `docs/CONCEPT.md` and/or `docs/ARCHITECTURE.md` — back into sync with the actual codebase when they've gone **thin** (vague, under-specified) or **drifted** (the code moved and the doc didn't).

It reads the docs and the code, diffs them into a gap/drift inventory, interviews you (via `AskUserQuestion`) only on what the code can't answer for itself, edits the docs in their existing voice, and produces a drift report that can surface latent bugs.

> Requires the docs to already exist — this skill reconciles existing docs, it does not create them from scratch.

## Install

Skills live in `~/.claude/skills/`. Clone this repo **into a folder named `reconcile-docs`** (the skill's name must match its folder, so don't use the default `claude-code-reconcile-docs` directory name):

```sh
git clone git@github.com:IngvarKofoed/claude-code-reconcile-docs.git ~/.claude/skills/reconcile-docs
```

Or over HTTPS:

```sh
git clone https://github.com/IngvarKofoed/claude-code-reconcile-docs.git ~/.claude/skills/reconcile-docs
```

That's it — Claude Code picks up skills from `~/.claude/skills/` automatically.

### Updating

```sh
cd ~/.claude/skills/reconcile-docs && git pull
```

## Usage

In a project whose `docs/CONCEPT.md` or `docs/ARCHITECTURE.md` has fallen out of sync with the code, invoke it:

```
/reconcile-docs
```

It will ask which doc to reconcile, build the inventory, lead with the question that frames the whole pass (is the doc a target-vision or a description of what's built?), then interview you only on the genuinely ambiguous findings before editing the doc and reporting any possible bugs.
