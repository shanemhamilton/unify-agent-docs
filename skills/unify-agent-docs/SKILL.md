---
name: unify-agent-docs
description: >-
  Consolidate a project's AI-agent instruction files (CLAUDE.md, AGENTS.md, GEMINI.md,
  .cursorrules, copilot-instructions) into ONE canonical file per directory, exposed under
  every tool's filename via symlink, plus a drift-proof guard. Use this whenever the user
  wants agent instructions to stay in sync across tools, mentions that CLAUDE.md and AGENTS.md
  have drifted or disagree, says switching between Claude Code and Codex (or Cursor/Gemini)
  changes the rules, wants a single source of truth for agent context, asks to dedupe or
  centralize CLAUDE.md/AGENTS.md across a repo or monorepo, or wants the same setup replicated
  in another project. Works for single-repo, monorepo, and multi-repo layouts on any stack.
---

# Unify Agent Docs

## What this does and why

Most projects accumulate several agent-instruction files — `CLAUDE.md` (Claude Code),
`AGENTS.md` (Codex/Cursor), `GEMINI.md` (Gemini CLI) — at multiple directory levels. They
drift: the same facts get restated and disagree, and the `CLAUDE.md` vs `AGENTS.md` in the
*same directory* carry different rules, so switching tools silently changes the agent's
instructions. That is the bug this skill removes.

The fix is structural, not vigilance-based: **one real file per directory, every other tool
name is a symlink to it.** All tools then read identical bytes — drift between names becomes
physically impossible. Cross-cutting facts that would otherwise be copy-pasted live in one
`shared-core` file. A pre-commit guard fails loudly if anything is tampered with. The result
survives sessions, tool switches, and new contributors.

Bundled with this skill:
- `scripts/agent-docs.sh` — generic sync/check engine. Auto-discovers every directory with a
  real canonical file; no per-project config. Copy it into the target repo's `tools/`.
- `scripts/pre-commit` — the guard hook.
- `references/multi-root.md` — the shared-core pattern and a `shared-core.md` template (read
  only when the project is a monorepo or has sibling repos).

## Phase 0 — Discover, then state the plan

Do this before changing anything. The point is to learn the project's actual shape rather
than assume one.

1. Find every `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `.cursorrules`, and
   `.github/copilot-instructions.md` at all levels. **Exclude** `node_modules`, `.git`,
   `vendor`, `build`, `dist`, `Pods`, and other vendored/generated dirs.
2. For each, note path, line count, and whether it's a thin pointer stub or a rich
   operational guide. `diff` any same-directory CLAUDE.md vs AGENTS.md pair to surface
   conflicts.
3. Classify the project:
   - **single-root** — one app/repo (most projects). Skip the shared-core machinery.
   - **multi-root** — a monorepo with packages/apps, or sibling repos that duplicate facts.
     Then read `references/multi-root.md`.
4. Choose the canonical filename. Default to **`AGENTS.md`** (broadest tool support) as the
   one real file; `CLAUDE.md`/`GEMINI.md` become symlinks. Only invert if the project clearly
   treats CLAUDE.md as primary — symlink either way. (The script's canonical name is set via
   `AGENT_DOCS_CANONICAL`, default `AGENTS.md`.)
5. Report the inventory, the conflicts, the chosen structure, and which directories will get a
   canonical file. Pause for confirmation only if something is genuinely ambiguous (single- vs
   multi-root, which file wins a real factual conflict, or a discovered secret). Otherwise
   proceed.

## Phase 1 — Merge, don't delete

This is the step that most needs judgment, so spend care here. Where a directory has both a
rich file and a thin one, **merge all operational detail into the single canonical file** —
never drop the richer file's content to "let the source of truth win." The thin file is
usually the *less* complete one even when it's nominally authoritative.

- De-duplicate repeated blocks to a single copy.
- When guidance is genuinely tool-specific (a `/foo` slash command for Claude vs a `$foo`
  invocation for Codex), keep ONE section with clearly labeled subsections
  (`### Claude Code`, `### Codex`, `### Gemini CLI`). Resolving these conflicts in plain sight
  is the whole point — it's what stops the drift from coming back.
- Add a short note at the top of each canonical file: "This is the canonical instruction file;
  CLAUDE.md and GEMINI.md are symlinks to it (managed by `tools/agent-docs.sh`). Edit this
  file, never the symlinks."

Don't invent project facts. Only consolidate what already exists; if two files state a fact
differently and you can't tell which is right, flag it for the user rather than guessing.

## Phase 2 — Shared core (multi-root only)

Skip entirely for single-root projects — put cross-cutting facts directly in the root
canonical file. For monorepos/sibling repos, follow `references/multi-root.md`: extract the
invariants (domain rules, schema/enum values, pricing constants, ownership tables, agent/skill
name tables, anti-hallucination rules) into `docs/agents/shared-core.md`, which the script
embeds into the root canonical file between `<!-- BEGIN SHARED-CORE hash:… -->` sentinels. For
sibling repos, pick one upstream and have the others pull it via `AGENT_DOCS_UPSTREAM`.

## Phase 3 — Install tooling + guard

```bash
mkdir -p <repo>/tools <repo>/.githooks
cp <skill>/scripts/agent-docs.sh <repo>/tools/agent-docs.sh && chmod +x <repo>/tools/agent-docs.sh
cp <skill>/scripts/pre-commit   <repo>/.githooks/pre-commit && chmod +x <repo>/.githooks/pre-commit
cd <repo> && git config core.hooksPath .githooks
tools/agent-docs.sh --sync     # creates the symlinks (+ embeds shared-core if present)
```

The script needs no editing — it discovers in-scope directories itself. Run `--sync` only
after the canonical `AGENTS.md` files from Phase 1 exist, because `--sync` replaces each
directory's `CLAUDE.md`/`GEMINI.md` with a symlink; running it before the merge would
overwrite a rich CLAUDE.md you hadn't folded in yet.

## Phase 4 — Security & legacy

- If you encounter any plaintext secret (API key, token, password) while merging, **stop and
  tell the user.** Remove it from the merged output, replace with an env-var reference, and
  warn that the secret remains in git history and must be **rotated** — deleting the file does
  not invalidate a leaked credential.
- Slim any stale/duplicate instruction files that are no longer authoritative into short
  redirect stubs pointing at the canonical files.

## Phase 5 — Quality pass (improve the canonical files)

Unification fixes *where* the rules live; this pass fixes *how good* they are. Now that each
directory has one canonical file, audit and tighten it so the agent actually has what it needs
— a structurally-perfect file full of vague or stale guidance still fails the agent.

- **Edit the canonical `AGENTS.md` only.** `CLAUDE.md`/`GEMINI.md` are symlinks and reflect the
  change automatically; writing to a symlink replaces it and trips the guard. If you change
  `docs/agents/shared-core.md`, re-run `tools/agent-docs.sh --sync` to re-embed it.
- Audit each canonical file against the rubric in
  [`references/quality-pass.md`](references/quality-pass.md) — commands present, architecture
  clear, non-obvious gotchas captured, concise, current, actionable. Score each, output a short
  report, and get the user's OK **before** writing.
- Make **targeted, minimal** additions only — leanness is the feature. Add a missing
  build/test/deploy command, a gotcha you found in the code, a corrected stale fact; don't
  restate the code or pad with generic advice. Show each change as a diff with a one-line "why."
- If the `claude-md-management:claude-md-improver` skill is installed you can drive this pass
  with it — but point it at the canonical `AGENTS.md` files (not the `CLAUDE.md` symlinks).

## Phase 6 — Verify, then commit

Verification is non-negotiable — the value of this skill is that drift *fails loudly*, so
prove the guard actually fires.

1. `tools/agent-docs.sh --check` passes; show the output.
2. Prove the guard: replace a symlinked CLAUDE.md with a real file, confirm `--check` **fails**
   (and the pre-commit hook would block), then restore it.
3. `ls -l` shows CLAUDE.md/GEMINI.md resolving to the canonical file in every in-scope dir.
4. (multi-root) `tools/agent-docs.sh --check-cross <other-repo>` shows zero drift.
5. Commit on a branch `chore/unify-agent-docs`, staging files by name (no `git add -A`). Do
   not push or open a PR unless the user asked — report what's staged and let them review.

## Constraints

- Documentation-only change: do not touch application code.
- Lose no operational detail — when in doubt, merge and label rather than drop.
- A fresh clone needs `git config core.hooksPath .githooks` once; git deliberately won't
  auto-activate a repo's own hooks on clone. Note this for the user.
- Symlinks are committed as git mode `120000`; confirm that after merging a PR so a squash
  didn't turn them back into regular files.
