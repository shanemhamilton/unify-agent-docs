# Quality pass: make each canonical file actually good

Run this AFTER unification (symlinks + guard in place). Unification fixes *where* the rules
live; this pass fixes *how good* they are. Adapted from the CLAUDE.md-improver rubric for the
unified layout.

## The one rule that changes here

Edit the **canonical `AGENTS.md`** (or whatever `AGENT_DOCS_CANONICAL` is). `CLAUDE.md` and
`GEMINI.md` are symlinks — they reflect the edit automatically, and writing to them directly
would replace the symlink and trip the guard. If you change `docs/agents/shared-core.md`,
re-run `tools/agent-docs.sh --sync` so the embedded block (and its hash) update.

## Rubric — score each canonical file

| Criterion | Weight | Check |
|-----------|--------|-------|
| Commands / workflows | High | Are build, test, run, deploy, lint commands present and copy-paste ready? |
| Architecture clarity | High | Can an agent understand the directory's structure and entry points from this file? |
| Non-obvious patterns | Medium | Are gotchas, quirks, and "do X not Y" rules captured (the stuff not visible in code)? |
| Conciseness | Medium | No restating what the code says, no generic best-practice padding? |
| Currency | High | Do commands, paths, and facts match the current code? |
| Actionability | High | Are instructions executable and specific, not vague aspiration? |

Grades: **A** 90–100 · **B** 70–89 · **C** 50–69 · **D** 30–49 · **F** 0–29.

## Workflow

1. **Audit, don't edit yet.** Read each canonical file and skim the directory it governs.
   Score each file against the rubric.
2. **Report first.** Output a short per-file report (score + the 1–3 concrete gaps) BEFORE
   touching anything. Get the user's OK.
3. **Targeted, minimal additions only.** Good additions: a missing build/test command found in
   `package.json`/`Makefile`/CI, a gotcha discovered in the code, a corrected stale fact, an
   entry-point or config path that wasn't obvious. Bad additions: restating the code, generic
   advice, verbose prose, one-off fixes unlikely to recur. **Leanness is the feature** — every
   line competes for the agent's attention.
4. **Show diffs.** For each change: which canonical file, the added block as a diff, and a
   one-line "why this helps future sessions."
5. **Apply** to the canonical `AGENTS.md`. Re-run `tools/agent-docs.sh --check` afterward
   (content edits don't affect symlinks; shared-core edits need `--sync` first).

## Where a fact belongs (multi-root)

- A command/gotcha specific to one app/package → that directory's canonical `AGENTS.md`.
- An invariant every package must share (enum, constant, ownership, convention) →
  `docs/agents/shared-core.md`, then `--sync`. Don't paste a shared fact into five files.

## Using the claude-md-improver skill, if installed

If `claude-md-management:claude-md-improver` is available you can drive this pass with it, with
two adjustments: point it at the canonical `AGENTS.md` files (not the `CLAUDE.md` symlinks), and
ignore its assumption that it may write `CLAUDE.md` directly — here that file is a symlink.
