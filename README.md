# unify-agent-docs

**One canonical instruction file per directory for Claude Code, Codex, Cursor, and Gemini — so your agent docs can never drift apart again.**

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code) (and a standalone shell script) that consolidates a project's `CLAUDE.md`, `AGENTS.md`, and `GEMINI.md` files behind a single source of truth, then guards against drift with a pre-commit hook.

---

## The problem

Modern projects collect a pile of AI-agent instruction files — `CLAUDE.md` for Claude Code, `AGENTS.md` for Codex and Cursor, `GEMINI.md` for Gemini CLI — at multiple directory levels. They drift:

- The same facts (build commands, schema enums, pricing constants, conventions) get restated in several files and quietly disagree.
- The `CLAUDE.md` and `AGENTS.md` in the **same directory** end up with different rules, so switching tools silently changes the instructions your agent follows.
- Nobody notices until an agent acts on a stale fact.

## The fix

Structural, not vigilance-based:

- **One real file per directory.** `AGENTS.md` holds the content; `CLAUDE.md` and `GEMINI.md` become **symlinks** to it. Every tool reads identical bytes — drift between the names is impossible by construction.
- **One home for cross-cutting facts.** In a monorepo or multi-repo setup, invariants (enums, constants, ownership tables, agent/skill names, anti-hallucination rules) live once in `docs/agents/shared-core.md` and are embedded into the root file between hash-stamped sentinels.
- **A guard that fails loudly.** A `pre-commit` hook runs a `--check` that fails the commit if a symlink was replaced by a real file (e.g. a tool overwrote `CLAUDE.md`) or the embedded shared block drifted from its source.

```
your-project/
├── AGENTS.md            # the one real file (canonical)
├── CLAUDE.md  → AGENTS.md   (symlink)
├── GEMINI.md  → AGENTS.md   (symlink)
└── packages/api/
    ├── AGENTS.md        # canonical for this package
    ├── CLAUDE.md → AGENTS.md
    └── GEMINI.md → AGENTS.md
```

---

## Install

### As a Claude Code skill (recommended)

```bash
git clone https://github.com/shanemhamilton/unify-agent-docs.git \
  ~/.claude/skills/unify-agent-docs
```

Then in any project, ask Claude to *"unify the agent docs in this project"* (or invoke `/unify-agent-docs`). The skill walks the full workflow: inventory the files, merge their content losslessly, set up the symlinks, install the guard, and verify.

### Just the script (any agent, no skill runtime)

The engine is a single dependency-free Bash script — copy it into your repo:

```bash
curl -fsSL https://raw.githubusercontent.com/shanemhamilton/unify-agent-docs/main/scripts/agent-docs.sh \
  -o tools/agent-docs.sh && chmod +x tools/agent-docs.sh
```

---

## Usage (the script)

```bash
tools/agent-docs.sh --sync                 # create the symlinks (+ embed shared-core if present)
tools/agent-docs.sh --check                # verify everything is in sync (use in pre-commit)
tools/agent-docs.sh --check-cross ../other # compare shared-core against a sibling repo
```

`--sync` and `--check` are **config-free**: the script auto-discovers every directory that
contains a real canonical file and manages the symlinks there. It works on single-repo,
monorepo, and multi-repo layouts on any stack.

Override behavior with environment variables when you need to:

| Variable | Default | Purpose |
|----------|---------|---------|
| `AGENT_DOCS_CANONICAL` | `AGENTS.md` | Which filename is the real file (the others symlink to it). |
| `AGENT_DOCS_LINKS` | `CLAUDE.md GEMINI.md` | Which names become symlinks. |
| `AGENT_DOCS_SHARED` | `docs/agents/shared-core.md` | Path to the shared-core file (multi-root). |
| `AGENT_DOCS_UPSTREAM` | _(unset)_ | A sibling repo to pull shared-core from during `--sync`. |

### Enable the guard

```bash
cp scripts/pre-commit .githooks/pre-commit && chmod +x .githooks/pre-commit
git config core.hooksPath .githooks
```

> Git deliberately won't auto-activate a repo's own hooks on clone (a security boundary), so each fresh clone runs `git config core.hooksPath .githooks` once.

---

## Multi-root / monorepo

For monorepos or sibling repos that share invariants, put the cross-cutting facts in
`docs/agents/shared-core.md`. The script embeds it into the root canonical file between
sentinels and stamps a content hash:

```
<!-- BEGIN SHARED-CORE hash:XXXX src:docs/agents/shared-core.md ... -->
...generated copy...
<!-- END SHARED-CORE -->
```

`--check` recomputes the hash and fails if the embed drifted from the source. For separate
repos, designate one as upstream and have the others pull it via `AGENT_DOCS_UPSTREAM`, then
verify with `--check-cross`. See [`references/multi-root.md`](references/multi-root.md) for the
full pattern and a `shared-core.md` starter template.

---

## How merging works

The skill never deletes the richer file to "let the source of truth win." It **merges** all
operational detail into the single canonical file, de-duplicates repeated blocks, and where
guidance is genuinely tool-specific (a `/command` for Claude vs a `$command` for Codex) keeps
one section with clearly labeled `### Claude Code` / `### Codex` / `### Gemini CLI`
subsections. Resolving those conflicts in plain sight is what stops the drift from returning.

It also stops and flags any plaintext secret it finds while merging — removing it from the
output and reminding you the leaked credential must be rotated (deleting the file doesn't
invalidate it).

---

## Compatibility

| Tool | Reads | Covered by |
|------|-------|------------|
| Claude Code | `CLAUDE.md` | symlink → canonical |
| Codex / Cursor | `AGENTS.md` | canonical (default) |
| Gemini CLI | `GEMINI.md` | symlink → canonical |

Symlinks are committed as git mode `120000` and resolve transparently on every platform git
supports. After merging a PR, confirm a squash didn't turn a symlink back into a regular file
(the `--check` guard catches this).

---

## License

[MIT](LICENSE) © Shane Hamilton
